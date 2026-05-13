# skills_description

This repository stores reusable skill definitions for Claude Code agent workflows. Each skill lives under `saved_skills/<name>/SKILL.md` and can be invoked via a slash command.

## Skills

---

### cerebras-setup
**Trigger:** `/cerebras-setup`

Adds [Cerebras AI](https://cloud.cerebras.ai) API support alongside existing Ollama or OpenAI config in any LangChain/LangGraph Python project. Because Cerebras is OpenAI API-compatible, it uses `langchain-openai`'s `ChatOpenAI` with a custom `base_url` — no new SDK required.

**What it does:**
- Detects the agent directory by looking for `ChatOllama`, `ChatOpenAI`, `ChatAnthropic`, or `create_react_agent`
- Adds `LLM_PROVIDER`, `CEREBRAS_API_KEY`, and `CEREBRAS_MODEL` to `.env` and `.env.example` without overwriting existing values
- Creates or updates `llm.py` with a provider switch so the project can toggle between `ollama` and `cerebras` via a single env var
- Adds `langchain-openai` to `requirements.txt` and installs it

**Default model:** `gpt-oss-120b` (128K context window)

[→ Full skill](saved_skills/cerebras-setup/SKILL.md)

---

### composio-cli
**Trigger:** automatic (context-based)

Guides use of the [Composio CLI](https://composio.dev) to discover, connect, and execute third-party tools from the terminal.

**What it covers:**
- `composio execute <slug>` — run a known tool with JSON args; use `--get-schema` or `--dry-run` to inspect inputs first
- `composio search "<task>"` — discover tool slugs when the name is unknown
- `composio link <toolkit>` — connect an account when `execute` reports the toolkit is not connected
- `composio execute --parallel` — run multiple independent tool calls in one command
- `composio listen` — subscribe to trigger events
- `composio run` — script multi-step workflows
- `composio proxy` — call authenticated app APIs directly

[→ Full skill](saved_skills/composio-cli/SKILL.md)

---

### create-run-scripts
**Trigger:** `/create-run-scripts`

Creates executable shell scripts to start a FastAPI backend and React/Vite frontend for a monorepo project.

**What it creates:**
- `scripts/start_backend.sh` — creates a Python venv if needed, installs dependencies from `requirements.txt` or `pyproject.toml`, copies `.env.example` to `.env` if missing, then starts uvicorn on port 8000 with `--reload`
- `scripts/start_frontend.sh` — runs `npm install` if `node_modules/` is absent, then `npm run dev`
- `scripts/start_all.sh` — launches both in parallel, piping backend output to `logs/backend.log`

Auto-detects frontend and backend directories; defaults to `frontend/` and `backend/`.

[→ Full skill](saved_skills/create-run-scripts/SKILL.md)

---

### create-stop-script
**Trigger:** `/create-stop-script`

Creates `scripts/stop_all.sh` — a script that cleanly stops a running FastAPI backend and Vite dev server.

**How it works:**
- Reads PIDs written by `start_all.sh` from `logs/pids` and kills each if still running
- Falls back to port-based killing via `lsof` if the PID file is missing or stale (port 8000 → uvicorn, port 5173 → Vite)
- Prints what it stopped, or "No running processes found." if nothing was running
- Updates `start_all.sh` to write a PID file if it doesn't already

[→ Full skill](saved_skills/create-stop-script/SKILL.md)

---

### explain-code
**Trigger:** automatic (context-based)

Explains code in a way that builds genuine understanding rather than just describing what each line does.

**Always includes:**
1. **An analogy** — compares the code to something from everyday life
2. **An ASCII diagram** — shows flow, structure, or relationships visually
3. **A step-by-step walkthrough** — explains what actually happens at runtime
4. **A gotcha** — highlights a common mistake or non-obvious behavior

[→ Full skill](saved_skills/explain-code/SKILL.md)

---

### github-railway
**Trigger:** `/github-railway`

Sets up or repairs a GitHub Actions CI/CD pipeline that deploys a FastAPI backend and React/Vite frontend monorepo to [Railway](https://railway.app).

**Key hard-won lessons encoded in the skill:**
- Railway CLI v4 rejects UUID tokens — the only working auth method is writing `~/.railway/config.json` directly using `accessToken` + `refreshToken` stored as GitHub secrets
- `RAILWAY_TOKEN` env var must be deleted entirely — if set, the CLI reads it first and fails before checking `config.json`
- `railway up` must run from the repo root, not a subdirectory (double-nesting breaks builds)
- Use `ubuntu-latest` runners — Alpine/musl crashes the Railway CLI binary (requires glibc)
- OAuth tokens expire roughly daily — update `RAILWAY_ACCESS_TOKEN` and `RAILWAY_REFRESH_TOKEN` secrets after `railway login`
- `VITE_API_URL` is baked at build time — changing it requires a full rebuild, not just a restart

[→ Full skill](saved_skills/github-railway/SKILL.md)

---

### gitlab-aws-ecs
**Trigger:** `/gitlab-aws-ecs`

Sets up or repairs a GitLab CI/CD pipeline that deploys a FastAPI backend and React/Vite frontend monorepo to AWS ECS Express Mode via ECR.

**Key hard-won lessons encoded in the skill:**
- AWS App Runner is deprecated (April 30, 2026) — ECS Express Mode is the replacement
- Alpine musl (`docker:24`, `docker:27`) breaks Python 3.12 `pyexpat` on all pip operations — use `python:3.12-slim` (Debian) instead
- `amazon/aws-cli` Docker image requires `entrypoint: [""]` override to run GitLab CI job scripts
- Non-docker base images need explicit `DOCKER_HOST: tcp://docker:2376` when using dind
- Covers required IAM roles, ECR setup, ECS service config, and first-deploy ordering for `VITE_API_URL`

[→ Full skill](saved_skills/gitlab-aws-ecs/SKILL.md)

---

### gitlab-railway
**Trigger:** `/gitlab-railway`

Sets up or repairs a GitLab CI/CD pipeline that deploys a FastAPI backend and React/Vite frontend monorepo to [Railway](https://railway.app). The GitLab counterpart to `github-railway`, with the same core lessons adapted for GitLab CI/CD variables and runner syntax.

**Key hard-won lessons encoded in the skill:**
- Railway CLI v4 OAuth auth requires writing `~/.railway/config.json` directly — store `accessToken`/`refreshToken` as GitLab CI/CD variables
- `railway up` must run from the repo root — subdirectory execution double-nests and breaks builds
- Use `node:20`, not `node:20-alpine` — Alpine/musl crashes the Railway CLI binary
- Railway service `Root Directory` setting handles subdirectory navigation; don't `cd` into it before `railway up`
- `VITE_API_URL` and `CORS_ORIGINS` require a full redeploy to take effect — not just a restart

[→ Full skill](saved_skills/gitlab-railway/SKILL.md)

---

### graphify
**Trigger:** `/graphify`

Converts any folder of files (code, docs, papers, images) into a navigable knowledge graph with community detection, an honest audit trail, and multiple output formats.

**What it produces:**
- **Interactive HTML** — navigable graph visualization
- **GraphRAG-ready JSON** — for use with downstream AI retrieval pipelines
- **GRAPH_REPORT.md** — plain-language audit of what was extracted and inferred

**Notable flags:**
- `--mode deep` — thorough extraction with richer inferred edges
- `--update` — incremental rebuild (re-extracts only new/changed files)
- `--directed` — preserves edge direction
- `--neo4j` / `--neo4j-push` — export Cypher or push directly to Neo4j
- `--obsidian` — write output to an Obsidian vault
- `--wiki` — build an agent-crawlable wiki (index + one article per community)
- `--watch` — auto-rebuild on file changes
- `--mcp` — start an MCP stdio server for agent access

[→ Full skill](saved_skills/graphify/SKILL.md)

---

### langgraph-cv
**Trigger:** `/langgraph-cv`

Instruments any LangGraph agent to export its full message state as `state.json`, readable by the [Context Visualizer](http://localhost:5173) — a tool for inspecting every system prompt, human turn, AI reasoning step, tool call, and tool result in a LangGraph run.

**What it does:**
- Finds the agent file (looks for `from langgraph` / `import langgraph`)
- Identifies the model name, agent variable, invoke call, and messages variable
- Drops a `cv_export.py` helper into the agent's directory
- Adds two lines to the agent file: one import, one call after `agent.invoke()`

[→ Full skill](saved_skills/langgraph-cv/SKILL.md)

---

### ollama-wsl
**Trigger:** `/ollama-wsl`

Configures a WSL2 application to reach an [Ollama](https://ollama.ai) instance running on the Windows host. In WSL2, `localhost` resolves to the Linux VM — not Windows — so direct `localhost:11434` connections fail without this configuration.

**What it does:**
- Detects the Windows host IP via `ip route show default`
- Configures Windows Firewall to allow inbound TCP on port 11434
- Sets `OLLAMA_HOST=0.0.0.0` on the Windows side so Ollama listens on all interfaces
- Updates the app's `.env` to set `OLLAMA_HOST` to the gateway IP
- Sets `OLLAMA_ORIGINS` for CORS if the app calls Ollama from a browser

[→ Full skill](saved_skills/ollama-wsl/SKILL.md)

---

### security-https-bruteforce
**Trigger:** `/security-https-bruteforce`

Audits and fixes two security vulnerabilities in a local FastAPI + vanilla JS application: unencrypted HTTP transport and an unprotected login endpoint.

**Fix 1 — HTTPS:**
- Creates `scripts/generate_cert.py` to generate a self-signed P-256 TLS certificate for localhost
- Updates `scripts/run.sh` to pass `--ssl-keyfile` and `--ssl-certfile` to uvicorn

**Fix 2 — Brute-force protection:**
- Adds `slowapi` rate limiting (5 attempts/minute) to the unlock/login endpoint
- Adds a server-side math CAPTCHA (`backend/captcha.py`) with `generate()` and `verify()` functions
- Updates the frontend to fetch and display the CAPTCHA challenge before submitting credentials

Audits current state first and reports what is already fixed before making changes.

[→ Full skill](saved_skills/security-https-bruteforce/SKILL.md)

---

### security-web-vulns
**Trigger:** `/security-web-vulns`

Audits and fixes five common web vulnerabilities in a FastAPI + vanilla JS application.

**The five fixes:**
1. **Session token in server logs** — moves the token from a `?token=` URL query param to an `Authorization: Bearer` header so it doesn't appear in access logs
2. **Stored XSS** — adds an `esc()` HTML-encoder and passes all user-supplied strings through it before assigning to `innerHTML`
3. **`javascript:` URL injection** — validates URL fields against an `http://` / `https://` allowlist before rendering as `<a href>`
4. **Mass-assignment in PUT** — adds an explicit `_PLAINTEXT_FIELDS` allowlist in `crud.py` that maps field names to encrypted columns; silently ignores `*_enc` columns and non-updatable fields
5. **Secrets in DOM attributes** — replaces `data-secret="actualValue"` with `data-field="fieldName"` so secret values live only in JS state, not in the DOM where DevTools can read them

Audits current state first and reports what is already fixed before making changes.

[→ Full skill](saved_skills/security-web-vulns/SKILL.md)

---

### simple-auth
**Trigger:** `/simple-auth`

Adds JWT-based username/password authentication with an admin role to a FastAPI + React (Vite) monorepo. All existing data and wikis remain shared across authenticated users.

**Backend additions:**
- `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/auth/me`
- `GET/POST /api/admin/users`, `PATCH /api/admin/users/{id}/role`, `DELETE /api/admin/users/{id}`
- `get_current_user` and `require_admin` FastAPI dependencies applied to all existing endpoints
- First admin bootstrapped from env vars (`ADMIN_USERNAME` / `ADMIN_PASSWORD`) on startup if no users exist

**Frontend additions:**
- Login/register page that gates the entire app
- Admin tab (visible to admins only) for user management
- Logout button in the top bar
- `api.js` module with `getToken`, `setToken`, `apiFetch`, and `streamUrl` helpers

Audits current state first and reports what is already in place before making changes.

[→ Full skill](saved_skills/simple-auth/SKILL.md)

---

---

## Windsurf Skills

Windsurf skills and workflows found across local projects and global config. Sourced from `~/.codeium/windsurf/windsurf/workflows/` and project-level `.windsurf/` directories.

---

### review
**Source:** `~/.codeium/windsurf/windsurf/workflows/review.md` (global)

A code review workflow for identifying bugs, security issues, and improvements in code changes.

**Focus areas:**
- Logic errors, edge cases, null/undefined references
- Race conditions and concurrency issues
- Security vulnerabilities
- Resource leaks and improper resource management
- API contract violations and incorrect caching behavior
- Violations of existing code patterns or conventions

Only reports high-confidence findings based on complete codebase understanding — no speculative issues.

[→ Full skill](saved_windsurf_skills/review/SKILL.md)

---

### railway-deployment
**Source:** `~/github/windsurf_tutorials/.windsurf/workflows/railway-deployment.md`

Step-by-step guide for deploying a FastAPI backend + React/Vite frontend monorepo to Railway using Dockerfiles.

**What it covers:**
- Service configuration: backend on port 8080 (Python 3.12 + uvicorn), frontend on port 3000 (Node 22 + Vite + serve)
- `VITE_API_BASE_URL` must include `https://` prefix and `/api` suffix, and requires a manual redeploy after being set (baked at build time)
- Common issues: Node.js version errors (use `node:22-alpine`), frontend receiving HTML instead of JSON (missing `/api` suffix), 404s on API endpoints
- Ordered deployment steps: backend first → get URL → set frontend env var → deploy frontend

[→ Full skill](saved_windsurf_skills/railway-deployment/SKILL.md)

---

### fix-cors-errors
**Source:** `~/github/windsurf_tutorials/.windsurf/skills/fix-cors-errors.md`

How to diagnose and fix CORS (Cross-Origin Resource Sharing) errors in web applications.

**Covers:**
- **FastAPI**: `CORSMiddleware` with `allow_origins`, `allow_credentials`, `allow_methods`, `allow_headers`
- **Express (Node.js)**: `cors` npm package configuration
- **Flask**: `flask-cors` configuration
- Common pitfalls: browser preview ports requiring `allow_origins=["*"]` in dev, credentials requiring explicit origins (no wildcards), backend restart required after config changes
- Debugging checklist: console errors, backend accessibility, API URL match, Vite proxy config

[→ Full skill](saved_windsurf_skills/fix-cors-errors/SKILL.md)

---

### gitlab-aws-ecs
**Source:** `~/github/project_index/.windsurf/workflows/gitlab-aws-ecs.md`

Sets up a GitLab CI/CD pipeline to deploy a FastAPI backend + React/Vite frontend monorepo to AWS ECS Express Mode via ECR. More detailed than the Claude equivalent — includes full `.gitlab-ci.yml` templates and recovery procedures.

**Key hard-won lessons (15 total):**
- App Runner deprecated April 2026 → use ECS Express Mode
- Alpine musl breaks Python 3.12 pyexpat — use `python:3.12-slim`
- `amazon/aws-cli` image requires `entrypoint: [""]`
- Never mix standard ECS API with Express Mode API — causes undrainable zombie deployments; only fix is delete + recreate
- `serve` requires `tcp://0.0.0.0:PORT` form and explicit `WORKDIR`
- Supabase direct connection is IPv6-only — use session pooler URL for ECS
- Stuck deployment recovery procedure included

[→ Full skill](saved_windsurf_skills/gitlab-aws-ecs/SKILL.md)

---

### cerebras-setup
**Source:** `~/github/project_index/.windsurf/workflows/cerebras-setup.md`

Adds Cerebras AI API support alongside existing Ollama/OpenAI config in any LangChain/LangGraph Python project. Windsurf-flavored version of the Claude `cerebras-setup` skill.

**What it does:**
- Adds `LLM_PROVIDER`, `CEREBRAS_API_KEY`, `CEREBRAS_MODEL` to `.env` and `.env.example` without overwriting existing values
- Creates or updates `llm.py` with a `build_llm()` provider switch (ollama ↔ cerebras)
- Updates agent files to use `build_llm()` instead of direct instantiation
- Adds `langchain-openai` to `requirements.txt`
- Prints a summary with activation instructions

[→ Full skill](saved_windsurf_skills/cerebras-setup/SKILL.md)

---

### github-railway
**Source:** `~/github/project_index/.windsurf/workflows/github-railway.md`

Sets up a GitHub Actions CI/CD pipeline to deploy a FastAPI backend + React/Vite frontend monorepo to Railway. Windsurf-flavored version of the Claude `github-railway` skill, with a full workflow YAML template included.

**Key hard-won lessons (11 total):**
- `RAILWAY_TOKEN` does not work with CLI v4 — write `~/.railway/config.json` directly
- `projects` section must be populated with real UUIDs (empty `{}` → "No linked project")
- OAuth tokens expire ~daily — refresh via `railway login` and update GitHub secrets
- `railway up` must run from repo root
- Use `ubuntu-latest` — Alpine/musl crashes the CLI
- `VITE_API_URL` baked at build time — requires full rebuild after change
- Deploy jobs gated on `github.event_name == 'push'` so PRs don't deploy

[→ Full skill](saved_windsurf_skills/github-railway/SKILL.md)

---

---

## Windsurf Workflows

Windsurf workflows found across local projects and global config. Sourced from `~/.codeium/windsurf/windsurf/workflows/` and project-level `.windsurf/workflows/` directories. Stored as `WORKFLOW.md` to distinguish from skills.

---

### review
**Source:** `~/.codeium/windsurf/windsurf/workflows/review.md` (global)

Senior software engineer code review workflow. Identical in scope to the `review` entry in Windsurf Skills — finds bugs, security issues, and code improvements in changed code. Reports only high-confidence findings.

[→ Full workflow](saved_windsurf_workflows/review/WORKFLOW.md)

---

### railway-deployment
**Source:** `~/github/windsurf_tutorials/.windsurf/workflows/railway-deployment.md`

Deployment guide for a FastAPI + React/Vite monorepo to Railway using Dockerfiles. Covers service configuration, `VITE_API_BASE_URL` formatting requirements, common error patterns, and ordered deployment steps. See the [Windsurf Skills entry](#railway-deployment) for the full summary.

[→ Full workflow](saved_windsurf_workflows/railway-deployment/WORKFLOW.md)

---

### gitlab-aws-ecs
**Source:** `~/github/project_index/.windsurf/workflows/gitlab-aws-ecs.md`

GitLab CI/CD → AWS ECS Express Mode deployment workflow. 15 hard-won lessons, full `.gitlab-ci.yml` template, IAM setup, and stuck-deployment recovery procedure. See the [Windsurf Skills entry](#gitlab-aws-ecs-1) for the full summary.

[→ Full workflow](saved_windsurf_workflows/gitlab-aws-ecs/WORKFLOW.md)

---

### cerebras-setup
**Source:** `~/github/project_index/.windsurf/workflows/cerebras-setup.md`

Adds Cerebras AI API support to any LangChain/LangGraph Python project. Provider switch via `LLM_PROVIDER` env var, `build_llm()` helper, no overwriting of existing config. See the [Windsurf Skills entry](#cerebras-setup-1) for the full summary.

[→ Full workflow](saved_windsurf_workflows/cerebras-setup/WORKFLOW.md)

---

### github-railway
**Source:** `~/github/project_index/.windsurf/workflows/github-railway.md`

GitHub Actions → Railway deployment workflow with full YAML template. 11 hard-won lessons including Railway CLI v4 OAuth auth, token expiry, and `VITE_API_URL` build-time baking. See the [Windsurf Skills entry](#github-railway-1) for the full summary.

[→ Full workflow](saved_windsurf_workflows/github-railway/WORKFLOW.md)

---

### cerebras-setup-skill
**Source:** `~/github/project_index/.windsurf/workflows/cerebras-setup/SKILL.md`

A Claude-style `SKILL.md` version of the Cerebras setup workflow, stored inside the Windsurf workflows directory. Content is equivalent to the Claude [`cerebras-setup`](#cerebras-setup) skill.

[→ Full workflow](saved_windsurf_workflows/cerebras-setup-skill/WORKFLOW.md)

---

## Windsurf Rules

Windsurf rules found across local projects and global config. Sourced from `~/.codeium/windsurf/memories/` (global) and project-level `.windsurf/rules/` directories. Stored as `RULE.md`.

---

### global_rules
**Source:** `~/.codeium/windsurf/memories/global_rules.md` (global)

Global rules file — currently empty.

[→ Full rule](saved_windsurf_rules/global_rules/RULE.md)

---

### backend_logging
**Source:** `~/github/token_center/.windsurf/rules/backend_logging.md`

Enforces file-based logging for all backend application logs.

**Rules:**
- All logs must be written to text files (not stdout only)
- Include ISO-format timestamps in every log entry
- Create a dedicated `logs/` directory; create it if it doesn't exist
- Use descriptive file names (`app.log`, `error.log`, etc.)
- Append to log files — never overwrite
- Include relevant context in messages (request ID, user ID, etc.)

[→ Full rule](saved_windsurf_rules/backend_logging/RULE.md)

---

### planning
**Source:** `~/github/token_center/.windsurf/rules/planning.md`  
**Trigger:** `always_on`

When asked to create a plan, write it to a markdown file in the project directory (e.g. `PLAN.md` at the root) rather than outputting it only in the Cascade panel.

[→ Full rule](saved_windsurf_rules/planning/RULE.md)

---

### tech_stack
**Source:** `~/github/token_center/.windsurf/rules/tech_stack.md`  
**Trigger:** `always_on`

Enforces a consistent web application tech stack. When building a web app, always use:
- **Backend:** FastAPI (Python)
- **Frontend:** React (TypeScript)

Do not suggest Flask, Django, Express, Vue, or other alternatives unless explicitly asked.

[→ Full rule](saved_windsurf_rules/tech_stack/RULE.md)

---

## Windsurf Plans

App development plans written by Windsurf Cascade, stored in `~/.windsurf/plans/`. Stored as `PLAN.md`.

---

### echo-app
**Source:** `~/.windsurf/plans/echo-app-railway-ready-3f28c1.md`

Plan for a simple FastAPI + React (TypeScript) echo web app with file-based logging and Railway deployment preparation.

**What it builds:**
- FastAPI backend with a `POST /echo` endpoint that logs each request to `logs/echo.log` (ISO timestamps, per `backend_logging` rule)
- React frontend with a text input, submit button, and response display area
- Executable `run-backend.sh` and `run-frontend.sh` scripts (from project skills)
- Railway-ready backend: `railway.toml`, `requirements.txt`, `.env.example`, `/health` endpoint, CORS via env var
- Backend on port 8000, frontend on port 5173

[→ Full plan](saved_windsurf_plans/echo-app/PLAN.md)

---

### influencer-tracking-app
**Source:** `~/.windsurf/plans/influencer-tracking-app-88baf9.md`

Plan for a FastAPI + React app that displays news summaries and tweets for three influencers (Obama, Musk, Cuban) with sentiment analysis badges.

**What it builds:**
- SQLite database (SQLAlchemy) with `influencers`, `news`, and `tweets` tables
- Sentiment analysis via TextBlob (positive/neutral/negative, scored and labeled)
- REST API: `GET /api/influencers`, `/api/influencers/{id}/news`, `/api/influencers/{id}/tweets`, `POST /api/refresh-data`
- React frontend with an influencer dropdown, two-column layout (news panel + Twitter feed), color-coded sentiment badges
- Mock data: 10–15 news articles and 20–30 tweets per influencer

[→ Full plan](saved_windsurf_plans/influencer-tracking-app/PLAN.md)

---

### project-index-app
**Source:** `~/.windsurf/plans/project-index-app-0b176e.md`

Plan for a React + FastAPI dashboard that scans local git repos, generates AI summaries via Ollama, and persists results in a JSON file.

**What it builds:**
- FastAPI backend with `POST /projects`, `GET /projects`, `GET /projects/{id}`, `DELETE /projects/{id}`, `POST /projects/{id}/refresh`
- Repo analyzer: validates git repo, counts LOC by language, extracts metadata (last commit, branch, size), reads README
- Ollama integration: sends repo metadata + README to local model, returns summary and run instructions
- JSON persistence at `data/projects.json`
- React (Vite + TypeScript) frontend with TailwindCSS + shadcn/ui: dashboard grid, add-project dialog, project detail view with language breakdown

[→ Full plan](saved_windsurf_plans/project-index-app/PLAN.md)

---

### word-to-latex
**Source:** `~/.windsurf/plans/word-to-latex-plan-8411e2.md`

Plan for a docx-to-LaTeX CLI pipeline plus a FastAPI + React document viewer, built on top of an existing `research_papers` repository.

**What it builds:**
- CLI (`typer`/`click`) that accepts `--docx` and `--template` args; parses Word headings/figures/references; renders LaTeX via Jinja templates per venue (IEEE, ACM, NeurIPS, AAAI); optionally compiles to PDF
- FastAPI backend (`src/docserve`): upload endpoint, document CRUD, section/PDF/Word streaming endpoints, background compilation tasks
- React frontend (Vite + TypeScript): library view, document workspace with PDF preview (`react-pdf`), LaTeX viewer/editor, Word preview, drag-and-drop upload flow
- JSON manifest per document for section tree, assets, and build status

[→ Full plan](saved_windsurf_plans/word-to-latex/PLAN.md)

---

## Project Structure

```text
saved_skills/
  cerebras-setup/SKILL.md
  composio-cli/SKILL.md
  create-run-scripts/SKILL.md
  create-stop-script/SKILL.md
  explain-code/SKILL.md
  github-railway/SKILL.md
  gitlab-aws-ecs/SKILL.md
  gitlab-railway/SKILL.md
  graphify/SKILL.md
  langgraph-cv/SKILL.md
  ollama-wsl/SKILL.md
  security-https-bruteforce/SKILL.md
  security-web-vulns/SKILL.md
  simple-auth/SKILL.md

saved_windsurf_skills/
  review/SKILL.md
  railway-deployment/SKILL.md
  fix-cors-errors/SKILL.md
  gitlab-aws-ecs/SKILL.md
  cerebras-setup/SKILL.md
  github-railway/SKILL.md

saved_windsurf_workflows/
  review/WORKFLOW.md
  railway-deployment/WORKFLOW.md
  gitlab-aws-ecs/WORKFLOW.md
  cerebras-setup/WORKFLOW.md
  github-railway/WORKFLOW.md
  cerebras-setup-skill/WORKFLOW.md

saved_windsurf_rules/
  global_rules/RULE.md
  backend_logging/RULE.md
  planning/RULE.md
  tech_stack/RULE.md

saved_windsurf_plans/
  echo-app/PLAN.md
  influencer-tracking-app/PLAN.md
  project-index-app/PLAN.md
  word-to-latex/PLAN.md
```

## How To Use This Repository

1. Each skill is self-contained: frontmatter (`name`, `description`, `trigger`) at the top, followed by action steps.
2. Skills with a `/trigger` are invoked explicitly; skills without one are applied automatically by Claude based on context.
3. Hard-won lessons are captured in each skill so future runs avoid known pitfalls.
