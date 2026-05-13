---
description: Set up or repair GitHub Actions CI/CD pipeline for deploying a monorepo to Railway (FastAPI backend + React/Vite frontend)
---

## Background — hard-won lessons

These are non-obvious issues discovered in practice. Apply them proactively:

1. **RAILWAY_TOKEN does NOT work** — UUID tokens from Railway dashboard (both project tokens and account tokens) are rejected by Railway CLI v4 with "Invalid RAILWAY_TOKEN". The only working approach is to write `~/.railway/config.json` directly using the `accessToken` and `refreshToken` from the developer's local `~/.railway/config.json`. Store these as `RAILWAY_ACCESS_TOKEN` and `RAILWAY_REFRESH_TOKEN` in GitHub Actions secrets.

2. **Delete RAILWAY_TOKEN if it exists** — If `RAILWAY_TOKEN` is set as a GitHub secret or environment variable, the CLI reads it first and fails before even checking `~/.railway/config.json`. Remove it entirely.

3. **The projects section in config.json must be populated** — An empty `"projects": {}` causes "No linked project found." Use `$(pwd)` as the key so it matches the GitHub Actions runner checkout path (`/home/runner/work/REPO/REPO`). The project UUID and environment UUID come from running `railway link` locally and reading `~/.railway/config.json`.

4. **Tokens expire roughly daily** — Railway OAuth tokens in `~/.railway/config.json` have a `tokenExpiresAt` of ~24 hours. When deployment starts failing with "Unauthorized", run `railway login` locally to get fresh tokens and update both `RAILWAY_ACCESS_TOKEN` and `RAILWAY_REFRESH_TOKEN` in GitHub secrets.

5. **`railway up` must run from repo root** — Running `railway up` from a subdirectory uploads only that subdirectory. If the Railway service has Root Directory set, it double-nests and fails. Always run `railway up` from the repo root (use `working-directory` only for non-railway steps).

6. **Use `ubuntu-latest`, not Alpine-based runners** — The Railway CLI binary requires glibc. Alpine-based runners use musl and crash with exit code 1 and no error message.

7. **`railway whoami` succeeds with project tokens but "No linked project" still appears** — Auth working does not mean the project is linked. Both the auth section AND the projects section of config.json must be correct.

8. **`VITE_API_URL` is baked at build time** — Set this in Railway frontend service variables before deploying. Changing it after and restarting has no effect — a full rebuild via `railway up` is required.

9. **CORS_ORIGINS requires a backend redeploy** — Changing this Railway variable doesn't take effect until the backend service is redeployed.

10. **Deploy jobs only run on push, not pull_request** — Use `if: github.event_name == 'push'` on deploy jobs so PRs only run lint/validate/test/security without deploying.

11. **`npm audit --audit-level=high`** — Use high threshold, not the default. npm audit is noisy with moderate findings that are rarely actionable. High/critical findings are the only ones worth blocking a deploy over.

## Step 1 — Detect project structure

Check for:
- `frontend/package.json` (React/Vite)
- `backend/main.py` or `backend/app.py` (FastAPI)
- Existing `.github/workflows/` directory
- Local Railway auth at `~/.railway/config.json`

## Step 2 — Read local Railway credentials

Read `~/.railway/config.json` and extract:
- `user.accessToken` — needed for `RAILWAY_ACCESS_TOKEN` GitHub secret
- `user.refreshToken` — needed for `RAILWAY_REFRESH_TOKEN` GitHub secret

Do NOT print these values. Tell the user where to find them.

Also check `tokenExpiresAt` — if it's in the past, tell the user to run `railway login` first to get fresh tokens.

## Step 3 — Ask for Railway project details

Ask the user for:
- Exact Railway service names (e.g. `myapp-backend`, `myapp-frontend`) — must match Railway dashboard exactly
- Railway project UUID and environment UUID — obtained by running `railway link` locally in the project directory, then reading the new entry added to `~/.railway/config.json`

## Step 4 — Generate or update `.github/workflows/deploy.yml`

Create or overwrite `.github/workflows/deploy.yml` using this exact template (substituting SERVICE_BACKEND, SERVICE_FRONTEND, PROJECT_UUID, ENVIRONMENT_UUID):

```yaml
name: Lint, Validate, Test, Security, and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
        working-directory: frontend
      - run: npm run lint
        working-directory: frontend

  lint-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install ruff -q
      - run: ruff check .
        working-directory: backend

  validate-frontend:
    runs-on: ubuntu-latest
    needs: lint-frontend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
        working-directory: frontend
      - run: npm run build
        working-directory: frontend
      - uses: actions/upload-artifact@v4
        with:
          name: frontend-dist
          path: frontend/dist/
          retention-days: 1

  validate-backend:
    runs-on: ubuntu-latest
    needs: lint-backend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt -q
        working-directory: backend
      - run: python -c "import main; print('Backend import OK')"
        working-directory: backend

  test-backend:
    runs-on: ubuntu-latest
    needs: validate-backend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt -q
        working-directory: backend
      - run: pip install -r requirements-test.txt -q
        working-directory: backend
      - run: python -m pytest tests/ -v
        working-directory: backend

  security-frontend:
    runs-on: ubuntu-latest
    needs: validate-frontend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
        working-directory: frontend
      - run: npm audit --audit-level=high
        working-directory: frontend

  security-backend:
    runs-on: ubuntu-latest
    needs: test-backend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install pip-audit -q
      - run: pip-audit -r requirements.txt
        working-directory: backend

  deploy-backend:
    runs-on: ubuntu-latest
    needs: security-backend
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm install -g @railway/cli
      - run: mkdir -p ~/.railway
      - name: Write Railway config
        run: |
          WORKDIR=$(pwd)
          cat > ~/.railway/config.json <<EOF
          {
            "projects": {
              "$WORKDIR": {
                "projectPath": "$WORKDIR",
                "name": "PROJECT_NAME",
                "project": "PROJECT_UUID",
                "environment": "ENVIRONMENT_UUID",
                "environmentName": "production",
                "service": null
              }
            },
            "user": {
              "token": null,
              "accessToken": "${{ secrets.RAILWAY_ACCESS_TOKEN }}",
              "refreshToken": "${{ secrets.RAILWAY_REFRESH_TOKEN }}",
              "tokenExpiresAt": 9999999999
            },
            "linkedFunctions": null
          }
          EOF
      - run: railway whoami
      - run: railway up --service SERVICE_BACKEND --environment production

  deploy-frontend:
    runs-on: ubuntu-latest
    needs: [security-frontend, validate-frontend]
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm install -g @railway/cli
      - run: mkdir -p ~/.railway
      - name: Write Railway config
        run: |
          WORKDIR=$(pwd)
          cat > ~/.railway/config.json <<EOF
          {
            "projects": {
              "$WORKDIR": {
                "projectPath": "$WORKDIR",
                "name": "PROJECT_NAME",
                "project": "PROJECT_UUID",
                "environment": "ENVIRONMENT_UUID",
                "environmentName": "production",
                "service": null
              }
            },
            "user": {
              "token": null,
              "accessToken": "${{ secrets.RAILWAY_ACCESS_TOKEN }}",
              "refreshToken": "${{ secrets.RAILWAY_REFRESH_TOKEN }}",
              "tokenExpiresAt": 9999999999
            },
            "linkedFunctions": null
          }
          EOF
      - run: railway whoami
      - run: railway up --service SERVICE_FRONTEND --environment production
```

## Step 5 — Check frontend is configured for ESLint

The `npm run lint` step requires ESLint to be set up. Check `frontend/package.json` for:
- A `"lint"` script calling `eslint`
- ESLint devDependencies: `eslint`, `@eslint/js`, `eslint-plugin-react-hooks`, `globals`
- A `frontend/eslint.config.mjs` file

If missing, set them up. The working eslint.config.mjs:

```js
import js from "@eslint/js";
import globals from "globals";
import reactHooks from "eslint-plugin-react-hooks";

export default [
  js.configs.recommended,
  {
    files: ["src/**/*.{js,jsx}"],
    plugins: { "react-hooks": reactHooks },
    languageOptions: {
      globals: { ...globals.browser },
      parserOptions: { ecmaFeatures: { jsx: true } },
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      "react-hooks/set-state-in-effect": "off",
    },
  },
];
```

## Step 6 — Print setup checklist

```
GitHub Actions Secrets (Settings → Secrets and variables → Actions):
  ☐ RAILWAY_ACCESS_TOKEN   — user.accessToken from ~/.railway/config.json
  ☐ RAILWAY_REFRESH_TOKEN  — user.refreshToken from ~/.railway/config.json
  ☐ Do NOT set RAILWAY_TOKEN — it overrides config.json and breaks auth

Railway project link (run locally first):
  ☐ cd <project-root> && railway link
  ☐ Copy project UUID and environment UUID from ~/.railway/config.json
  ☐ Paste into the "projects" section of the workflow config.json

Railway service settings (for each service → Settings):
  ☐ backend service  → Root Directory: backend
  ☐ frontend service → Root Directory: frontend

Railway environment variables (set before first deploy):
  ☐ backend service  → CORS_ORIGINS = https://<frontend-domain>.up.railway.app
  ☐ frontend service → VITE_API_URL = https://<backend-domain>.up.railway.app
      ↑ VITE_API_URL must be set BEFORE railway up runs (baked at compile time)

After deploying:
  ☐ Generate public domains in Railway → Settings → Networking
  ☐ Fill in CORS_ORIGINS and VITE_API_URL with real domains
  ☐ Redeploy backend after setting CORS_ORIGINS
  ☐ Re-run full pipeline after setting VITE_API_URL (full rebuild required)

If deploy fails with "Unauthorized":
  ☐ Run `railway login` locally to refresh tokens
  ☐ Update RAILWAY_ACCESS_TOKEN and RAILWAY_REFRESH_TOKEN in GitHub secrets
```

## Step 7 — Commit the file

Stage and commit with message:
`ci: add GitHub Actions pipeline for Railway deployment`

## Rules

- Never set RAILWAY_TOKEN — it breaks auth
- Always use `ubuntu-latest`, never Alpine
- Always run `railway up` from repo root (no `working-directory` on that step)
- Always pass `--environment production`
- Always populate the `projects` section with real UUIDs — empty `{}` causes "No linked project"
- Never print actual token values — tell the user where to find them
- Deploy jobs must have `if: github.event_name == 'push'`
- If workflow already exists, audit it against all lessons above and fix any issues
