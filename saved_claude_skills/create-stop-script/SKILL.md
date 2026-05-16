---
name: create-stop-script
description: Creates an executable stop script that kills a running React frontend and FastAPI backend
trigger: /create-stop-script
---

# /create-stop-script

Create `scripts/stop_all.sh` — an executable script that stops the React dev server and FastAPI backend for this project.

## What to do

1. **Check for `scripts/start_all.sh`** — if it doesn't write a PID file to `logs/pids`, update it to do so (write backend PID on line 1, frontend PID on line 2).

2. **Create `scripts/stop_all.sh`** that:
   - Reads PIDs from `logs/pids` and kills each process if it is still running
   - Removes `logs/pids` after use
   - Falls back to port-based killing via `lsof` if the PID file is missing or stale:
     - Port `8000` → FastAPI/uvicorn
     - Port `5173` → Vite dev server
   - Prints what it stopped, or "No running processes found." if nothing was running

3. **Make the script executable** with `chmod +x`.

4. **Print a short summary** confirming the script was created and how to use it:
   ```
   scripts/stop_all.sh  — stop both backend and frontend
   ```

## Rules

- Use `#!/usr/bin/env bash` and `set -euo pipefail`.
- All paths must be resolved relative to the project root via `SCRIPT_DIR` + `..`.
- Never use `kill -9` — use plain `kill` (SIGTERM) to allow graceful shutdown.
- If scripts already exist, overwrite them (they are generated artifacts).
