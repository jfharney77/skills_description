---
name: create-run-scripts
description: Creates executable start scripts for a React frontend and FastAPI backend
trigger: /create-run-scripts
---

# /create-run-scripts

Create executable shell scripts that start the React frontend and FastAPI backend for this project.

## What to do

1. **Detect paths** — find the frontend directory (look for `package.json` with a `dev` script) and the backend directory (look for `main.py` or `app.py` with FastAPI). Default to `frontend/` and `backend/` if present.

2. **Create `scripts/start_backend.sh`** — executable script that:
   - `cd`s into the backend directory
   - Creates a Python virtual environment at `backend/.venv` if one does not exist
   - Activates it
   - Runs `pip install -r requirements.txt` quietly if `requirements.txt` is present, or `pip install -e .` if `pyproject.toml` is present
   - Copies `.env.example` to `.env` if `.env` does not exist yet (and prints a reminder to review it)
   - Starts uvicorn: `uvicorn main:app --reload --host 0.0.0.0 --port 8000`

3. **Create `scripts/start_frontend.sh`** — executable script that:
   - `cd`s into the frontend directory
   - Runs `npm install` if `node_modules/` does not exist
   - Runs `npm run dev`

4. **Create `scripts/start_all.sh`** — convenience script that:
   - Starts the backend in the background (with its output piped to `logs/backend.log`)
   - Creates the `logs/` directory if it doesn't exist
   - Starts the frontend in the foreground so Ctrl-C stops everything cleanly
   - Traps SIGINT/SIGTERM to kill the background backend process on exit

5. **Make all three scripts executable** with `chmod +x`.

6. **Print a short summary** of what was created and how to use it:
   ```
   scripts/start_backend.sh   — start FastAPI only
   scripts/start_frontend.sh  — start React dev server only
   scripts/start_all.sh       — start both (backend in background, frontend in foreground)
   ```

## Rules

- Use `#!/usr/bin/env bash` and `set -euo pipefail` in every script.
- All paths in scripts must be relative to the project root (each script resolves the root via `SCRIPT_DIR` + `..`).
- Do not hardcode virtualenv activation paths — detect OS (check for `.venv/Scripts/activate` on Windows, `.venv/bin/activate` elsewhere).
- If scripts already exist, overwrite them (they are generated artifacts).
- Do not modify any source files — only create files under `scripts/` and `logs/` (add `logs/` to `.gitignore` if a `.gitignore` exists).
