---
name: gitlab-railway
description: Set up or repair GitLab CI/CD pipeline for deploying a monorepo to Railway (FastAPI backend + React/Vite frontend)
trigger: /gitlab-railway
---

# /gitlab-railway

Set up or audit a GitLab CI/CD → Railway deployment for a project with a FastAPI backend and React/Vite frontend.

## Background — hard-won lessons

These are non-obvious issues discovered in practice. Apply them proactively:

1. **Railway CLI v4 OAuth auth** — `RAILWAY_TOKEN` env var does NOT work with Railway CLI v4's OAuth tokens. The UUID tokens from `railway.app/account/tokens` are for the REST API, not the CLI. The only working approach for CI is to write `~/.railway/config.json` directly using the `accessToken` and `refreshToken` from the developer's local `~/.railway/config.json`. Store these as `RAILWAY_ACCESS_TOKEN` and `RAILWAY_REFRESH_TOKEN` in GitLab CI/CD variables.

2. **`railway up` must run from repo root** — Running `cd backend && railway up` uploads just the subdirectory. If the Railway service also has Root Directory set to `backend`, it double-nests and fails with "No such file or directory". Always run `railway up` from the repo root. Railway's Root Directory service setting then correctly navigates into the subdirectory.

3. **Use `node:20`, not `node:20-alpine`** — The Railway CLI binary is compiled against glibc. Alpine uses musl and will crash the CLI with exit code 1 and no error message.

4. **`VITE_API_URL` is baked at build time** — Set this in the Railway frontend service variables *before* triggering a deploy. Changing it after and just restarting the service has no effect — a full rebuild is required.

5. **CORS_ORIGINS requires a backend redeploy** — Setting or changing this variable in Railway doesn't take effect until the backend service is redeployed.

6. **Railway environment** — Default environment name is `production`. Always pass `--environment production` to `railway up`.

7. **Root Directory in Railway service settings** — Set `backend` for the backend service, `frontend` for the frontend service. This setting is what tells Railway which subdirectory of the uploaded repo to build.

## What to do

### Step 1 — Detect project structure

Check for:
- `frontend/package.json` (React/Vite)
- `backend/main.py` or `backend/app.py` (FastAPI)
- Existing `.gitlab-ci.yml`
- Local Railway auth at `~/.railway/config.json`

### Step 2 — Read local Railway credentials

Read `~/.railway/config.json` and extract:
- `user.accessToken` — needed for `RAILWAY_ACCESS_TOKEN` GitLab variable
- `user.refreshToken` — needed for `RAILWAY_REFRESH_TOKEN` GitLab variable

Do NOT print these values. Tell the user where to find them.

### Step 3 — Ask for Railway service names

Ask the user for the exact names of their Railway services (e.g. `clockgame-backend`, `clockgame-frontend`). These must match exactly what's in the Railway dashboard.

Also ask for the Railway project ID (UUID from the Railway project URL).

### Step 4 — Generate or update `.gitlab-ci.yml`

Create or overwrite `.gitlab-ci.yml` at the project root using this exact template (substituting SERVICE_BACKEND, SERVICE_FRONTEND with the real names):

```yaml
stages:
  - validate
  - deploy

validate-frontend:
  stage: validate
  image: node:20
  script:
    - cd frontend
    - npm ci
    - npm run build
  artifacts:
    paths:
      - frontend/dist/
    expire_in: 1 hour

validate-backend:
  stage: validate
  image: python:3.12-slim
  script:
    - cd backend
    - pip install -r requirements.txt -q
    - python -c "import main; print('Backend import OK')"

.railway-auth: &railway-auth
  - npm install -g @railway/cli
  - mkdir -p ~/.railway
  - |
    cat > ~/.railway/config.json <<EOF
    {
      "projects": {},
      "user": {
        "token": null,
        "accessToken": "$RAILWAY_ACCESS_TOKEN",
        "refreshToken": "$RAILWAY_REFRESH_TOKEN",
        "tokenExpiresAt": 9999999999
      },
      "linkedFunctions": null
    }
    EOF
  - railway whoami

deploy-backend:
  stage: deploy
  image: node:20
  needs: [validate-backend]
  script:
    - *railway-auth
    - railway up --service SERVICE_BACKEND --environment production
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-frontend:
  stage: deploy
  image: node:20
  needs: [validate-frontend]
  script:
    - *railway-auth
    - railway up --service SERVICE_FRONTEND --environment production
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### Step 5 — Print setup checklist

After writing the file, print this checklist for the user:

```
GitLab CI/CD Variables (Settings → CI/CD → Variables — uncheck "Protect variable"):
  ☐ RAILWAY_ACCESS_TOKEN   — user.accessToken from ~/.railway/config.json
  ☐ RAILWAY_REFRESH_TOKEN  — user.refreshToken from ~/.railway/config.json
  ☐ RAILWAY_PROJECT_ID     — UUID from Railway project URL (before the ?)

Railway service settings (for each service → Settings):
  ☐ clockgame-backend  → Root Directory: backend
  ☐ clockgame-frontend → Root Directory: frontend

Railway environment variables (set before first deploy):
  ☐ clockgame-backend  → CORS_ORIGINS = https://<frontend-domain>.up.railway.app
  ☐ clockgame-frontend → VITE_API_URL = https://<backend-domain>.up.railway.app
      ↑ VITE_API_URL must be set BEFORE the frontend build runs (baked at compile time)

After deploying:
  ☐ Generate public domains for both services in Railway → Settings → Networking
  ☐ Fill in CORS_ORIGINS and VITE_API_URL with the real domains
  ☐ Redeploy backend after setting CORS_ORIGINS
  ☐ Re-run full pipeline after setting VITE_API_URL (rebuild required)
```

### Step 6 — Commit the file

Stage and commit `.gitlab-ci.yml` with message:
`ci: add GitLab CI/CD pipeline for Railway deployment`

## Rules

- Always use `node:20`, never `node:20-alpine`
- Always run `railway up` from the repo root (no `cd backend` before it)
- Always pass `--environment production`
- Never print the actual token values — tell the user where to find them
- If `.gitlab-ci.yml` already exists with Railway jobs, audit it against the lessons above and fix any issues found
