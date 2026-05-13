---
description: Set up or repair GitLab CI/CD pipeline for deploying a monorepo (FastAPI backend + React/Vite frontend) to AWS ECS Express Mode via ECR
---

## Background — hard-won lessons

These are non-obvious issues discovered in practice. Apply them proactively:

1. **App Runner is deprecated (April 30, 2026)** — AWS no longer accepts new App Runner customers. Use ECS Express Mode instead. It works similarly: give it a container image URI, execution role, and infrastructure role, and AWS provisions the load balancer, scaling, and HTTPS automatically. URL format: `https://cl-<hash>.ecs.<region>.on.aws`

2. **Alpine musl breaks Python 3.12 pyexpat** — Both `apk add aws-cli` and `pip install awscli` fail on Alpine 3.20 and 3.21 (docker:24, docker:27) with `ImportError: Error relocating pyexpat... XML_SetAllocTrackerTrackerActivationThreshold: symbol not found`. This affects ALL pip operations, not just awscli. The fix: use `python:3.12-slim` (Debian) as the job image instead of any `docker:XX` Alpine image.

3. **amazon/aws-cli image needs entrypoint override** — The `amazon/aws-cli` Docker image uses `aws` as its entrypoint, which prevents GitLab CI from running `sh` to execute the job script. Always override it:
   ```yaml
   image:
     name: amazon/aws-cli
     entrypoint: [""]
   ```

4. **dind requires explicit DOCKER_HOST when using non-docker base image** — When the job image is not `docker:XX` (e.g. `python:3.12-slim`), the Docker CLI doesn't automatically know where the dind daemon is. Set these variables:
   ```yaml
   DOCKER_TLS_CERTDIR: "/certs"
   DOCKER_HOST: tcp://docker:2376
   DOCKER_TLS_VERIFY: 1
   DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"
   ```

5. **VITE_API_URL chicken-and-egg** — This must be baked into the JS bundle at Vite compile time. You must deploy the backend first, get its ECS URL, then set `BACKEND_URL` as a GitLab CI variable, then deploy the frontend. Changing it after the fact requires a full rebuild.

6. **IAM PassRole is required** — When the CI user calls `ecs:CreateExpressGatewayService` or `ecs:UpdateExpressGatewayService`, it must also have `iam:PassRole` permission for both the execution role and infrastructure role. Missing this causes a confusing AccessDeniedException.

7. **ECS Express Mode needs two IAM roles** — Unlike App Runner (one role), ECS Express Mode requires:
   - **Task Execution Role**: trusted by `ecs-tasks.amazonaws.com`, attach `AmazonECSTaskExecutionRolePolicy`. Lets ECS pull images and write logs.
   - **Infrastructure Role**: trusted by `ecs.amazonaws.com`, attach `AmazonECSInfrastructureRoleForExpressGatewayServices`. Lets ECS provision the ALB, security groups, and networking.

8. **Port hardcoding** — ECS Express Mode does not inject a `$PORT` variable like Railway does. Hardcode port 8080 in the Dockerfile CMD and set the `containerPort` to 8080 in the service config.

9. **ECR region must match** — ECR repos are region-specific. If the repo is in `us-east-2`, authenticate to `us-east-2` and push to `us-east-2`. Mismatches cause "repository not found" errors even if the repo name is correct.

10. **AmazonECS_FullAccess avoids permission whack-a-mole** — ECS Express Mode triggers several internal ECS API calls (`ecs:CreateCluster`, etc.) beyond just the ones you call explicitly. For the CI user, attaching the `AmazonECS_FullAccess` managed policy saves repeated debugging cycles.

11. **NEVER mix the standard ECS API with the Express Mode API** — Calling `aws ecs update-service` or `aws ecs update-service --force-new-deployment` on an Express Mode service corrupts its deployment state. It creates zombie ACTIVE deployments with `desired=1` that Express Mode will never drain, no matter how many times you stop the tasks. The only fix is to delete and recreate the service with `delete-express-gateway-service` + `create-express-gateway-service`. Only ever use `update-express-gateway-service` for config changes.

12. **`serve` (npm static server) requires `tcp://0.0.0.0:PORT` and `WORKDIR`** — In the frontend Dockerfile final stage, two things are required:
    - Set `WORKDIR /app` so `serve` finds the `dist/` directory relative to the correct path.
    - Use `CMD ["serve", "-s", "dist", "-l", "tcp://0.0.0.0:8080"]` — the plain `-l 8080` flag is silently ignored in newer versions of `serve`, causing it to fall back to port 3000. The `tcp://0.0.0.0:PORT` form is always explicit.
    ```dockerfile
    FROM node:20-slim
    RUN npm install -g serve
    WORKDIR /app
    COPY --from=builder /app/dist /app/dist
    EXPOSE 8080
    CMD ["serve", "-s", "dist", "-l", "tcp://0.0.0.0:8080"]
    ```

13. **Supabase direct connection is IPv6-only** — The default Supabase connection string (`db.<ref>.supabase.co`) resolves to an IPv6 address. ECS tasks on default VPC public subnets do not have IPv6. Use the **session pooler** URL instead: `aws-1-us-east-1.pooler.supabase.com:5432` with username `postgres.<ref>`. This resolves to IPv4 and works without a NAT gateway.

14. **asyncpg SSL for Supabase** — Pass `ssl="require"` (a string, not an `ssl.SSLContext`) to `asyncpg.create_pool()` for non-local connections. asyncpg accepts this string shorthand directly.

15. **Stuck deployment recovery** — If `describe-express-gateway-service` shows multiple ACTIVE deployments that won't drain, don't try to fight them with `stop-task` loops or `force-new-deployment`. Delete the service and recreate it:
    ```bash
    aws ecs delete-express-gateway-service --service-arn <ARN> --region <REGION>
    # Wait for drain (~3-5 min) — poll until create stops returning "still draining"
    aws ecs create-express-gateway-service --service-name <NAME> ... --region <REGION>
    # Note the new ingressPaths[0].endpoint — update BACKEND_URL CI variable
    # Rebuild frontend (VITE_API_URL is baked in at build time)
    ```

## Step 1 — Detect project structure

Check for:
- `frontend/package.json` (React/Vite)
- `backend/main.py` or `backend/app.py` (FastAPI)
- `backend/Dockerfile` and `frontend/Dockerfile` (create if missing)
- Existing `.gitlab-ci.yml`

## Step 2 — Create Dockerfiles if missing

**backend/Dockerfile** (port hardcoded to 8080, not $PORT):
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**frontend/Dockerfile** (multi-stage; VITE_API_URL baked at build time):
```dockerfile
FROM node:20-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
COPY . .
RUN npm run build

FROM node:20-slim
WORKDIR /app
RUN npm install -g serve
COPY --from=builder /app/dist ./dist
EXPOSE 8080
CMD ["serve", "-s", "dist", "-l", "8080"]
```

## Step 3 — Explain AWS prerequisites (one-time setup)

Tell the user to complete these in the AWS Console before the pipeline can run:

### ECR Repositories
Create two private repositories in ECR (match the region you'll use for everything):
- `<project>-backend`
- `<project>-frontend`

### IAM User for GitLab CI
Create a user (e.g. `gitlab-ci-<project>`) with no console access. Attach:
- `AmazonECS_FullAccess` managed policy
- This inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
        "arn:aws:iam::ACCOUNT_ID:role/ecsInfrastructureRoleForExpressServices"
      ]
    }
  ]
}
```

Generate an Access Key for this user — save the key ID and secret.

### Task Execution Role
IAM → Roles → Create role → AWS service → Elastic Container Service Task → attach `AmazonECSTaskExecutionRolePolicy` → name: `ecsTaskExecutionRole`

### Infrastructure Role
IAM → Roles → Create role → Custom trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ecs.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
Attach `AmazonECSInfrastructureRoleForExpressGatewayServices` → name: `ecsInfrastructureRoleForExpressServices`

### Create ECS Express Mode Services (after first image push)
Backend first (to get its URL for VITE_API_URL):
```bash
aws ecs create-express-gateway-service \
  --service-name <project>-backend \
  --primary-container '{"image":"ACCOUNT.dkr.ecr.REGION.amazonaws.com/<project>-backend:latest","containerPort":8080,"environment":[{"name":"CORS_ORIGINS","value":"*"}]}' \
  --execution-role-arn arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole \
  --infrastructure-role-arn arn:aws:iam::ACCOUNT:role/ecsInfrastructureRoleForExpressServices \
  --health-check-path /health \
  --region REGION
```
Note the `ingressPaths[0].endpoint` URL from the response — that's the backend URL.

Then frontend:
```bash
aws ecs create-express-gateway-service \
  --service-name <project>-frontend \
  --primary-container '{"image":"ACCOUNT.dkr.ecr.REGION.amazonaws.com/<project>-frontend:latest","containerPort":8080}' \
  --execution-role-arn arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole \
  --infrastructure-role-arn arn:aws:iam::ACCOUNT:role/ecsInfrastructureRoleForExpressServices \
  --region REGION
```

After both are created, update the backend's CORS_ORIGINS to the frontend's URL:
```bash
aws ecs update-express-gateway-service \
  --service-arn BACKEND_ARN \
  --primary-container '{"image":"...backend:latest","containerPort":8080,"environment":[{"name":"CORS_ORIGINS","value":"https://FRONTEND_URL"}]}' \
  --region REGION
```

## Step 4 — Generate .gitlab-ci.yml

Add a `build` stage between `security` and `deploy`. Use `python:3.12-slim` + `docker:27-dind` for image builds to avoid the Alpine pyexpat bug. Use `amazon/aws-cli` with entrypoint override for deploy jobs.

```yaml
stages:
  - lint
  - validate
  - test
  - security
  - build
  - deploy

# ... existing lint/validate/test/security jobs unchanged ...

# ── Build Docker Images for AWS ECR ───────────────────────────────────────────
.docker-ecr-setup:
  image: python:3.12-slim
  services:
    - docker:27-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_VERIFY: 1
    DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"
  before_script:
    - apt-get update -qq && apt-get install -y -qq docker.io
    - pip install awscli -q
    - aws ecr get-login-password --region $AWS_REGION
      | docker login --username AWS --password-stdin
        $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

build-backend-image:
  extends: .docker-ecr-setup
  stage: build
  needs: [security-backend]
  script:
    - REPO=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-backend
    - docker build -t $REPO:$CI_COMMIT_SHA -t $REPO:latest ./backend
    - docker push $REPO:$CI_COMMIT_SHA
    - docker push $REPO:latest
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

build-frontend-image:
  extends: .docker-ecr-setup
  stage: build
  needs: [security-frontend]
  script:
    - REPO=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-frontend
    - docker build --build-arg VITE_API_URL=$BACKEND_URL
        -t $REPO:$CI_COMMIT_SHA -t $REPO:latest ./frontend
    - docker push $REPO:$CI_COMMIT_SHA
    - docker push $REPO:latest
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# ── Deploy to AWS ECS Express Mode ────────────────────────────────────────────
deploy-backend-aws:
  stage: deploy
  image:
    name: amazon/aws-cli
    entrypoint: [""]
  needs: [build-backend-image]
  script:
    - aws ecs update-express-gateway-service
        --service-arn $BACKEND_SERVICE_ARN
        --primary-container "{\"image\":\"$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-backend:$CI_COMMIT_SHA\"}"
        --region $AWS_REGION
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-frontend-aws:
  stage: deploy
  image:
    name: amazon/aws-cli
    entrypoint: [""]
  needs: [build-frontend-image]
  script:
    - aws ecs update-express-gateway-service
        --service-arn $FRONTEND_SERVICE_ARN
        --primary-container "{\"image\":\"$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME-frontend:$CI_COMMIT_SHA\"}"
        --region $AWS_REGION
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

## Step 5 — Print setup checklist

```
GitLab CI/CD Variables (Settings → CI/CD → Variables):
  ☐ AWS_ACCESS_KEY_ID        — IAM user access key
  ☐ AWS_SECRET_ACCESS_KEY    — IAM user secret (mask this)
  ☐ AWS_REGION               — e.g. us-east-2
  ☐ AWS_ACCOUNT_ID           — 12-digit AWS account number
  ☐ BACKEND_URL              — ECS backend URL (set after first deploy)
  ☐ BACKEND_SERVICE_ARN      — ECS backend service ARN
  ☐ FRONTEND_SERVICE_ARN     — ECS frontend service ARN

First-deploy order (chicken-and-egg for VITE_API_URL):
  1. Push backend image to ECR manually (podman or docker)
  2. Create backend ECS service → note its URL
  3. Set BACKEND_URL in GitLab variables
  4. Push frontend image to ECR manually (with --build-arg VITE_API_URL=...)
  5. Create frontend ECS service → note its URL
  6. Update backend CORS_ORIGINS to frontend URL
  7. Set BACKEND_SERVICE_ARN and FRONTEND_SERVICE_ARN in GitLab variables
  8. Push branch → pipeline runs fully automated from here
```

## Step 6 — Commit

Stage and commit with message:
`feat: add AWS ECS Express Mode deployment via GitLab CI`

## Rules

- Never use `docker:XX` (Alpine) for jobs that need Python or pip — use `python:3.12-slim` instead
- Always override entrypoint when using `amazon/aws-cli` as the job image
- Always tag images with both `$CI_COMMIT_SHA` (traceable) and `latest` (for ECS to pull)
- The deploy command must include `--primary-container` with the image URI — ECS Express Mode doesn't have a separate "trigger deployment" command; you update the service config directly
- Scope AWS pipeline jobs to the feature branch during development; update rules to `$CI_DEFAULT_BRANCH` when ready to make AWS the primary deployment target
