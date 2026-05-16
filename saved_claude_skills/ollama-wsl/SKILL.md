---
name: ollama-wsl
description: Configure a WSL application to reach an Ollama instance running on the Windows host
trigger: /ollama-wsl
---

# /ollama-wsl

Make an Ollama instance running on Windows accessible from a WSL application.

## Background

In WSL2, `localhost` refers to the WSL VM, not the Windows host. Ollama running on Windows
binds to `127.0.0.1` by default, so WSL cannot reach it via `localhost:11434`. Two things
must be true for this to work:

1. **Windows side**: Ollama must listen on `0.0.0.0` (not just `127.0.0.1`), and Windows
   Firewall must allow inbound TCP on port 11434.
2. **WSL side**: the app must use the Windows host IP (the WSL gateway), not `localhost`.

## What to do

### Step 1 — Detect the Windows host IP

Run this from WSL and capture the output:
```bash
ip route show default | awk '{print $3; exit}'
```
This is the WSL gateway, which is the Windows host IP. Call it `$HOST_IP`.

### Step 2 — Test Ollama connectivity

```bash
curl -s --max-time 3 http://$HOST_IP:11434/api/version
```

**If the request succeeds** (returns JSON), skip to Step 4.

**If the request fails**, proceed to Step 3 to fix the Windows configuration.

### Step 3 — Fix the Windows configuration (if unreachable)

Run these from WSL using `powershell.exe`. Tell the user what each command does before running it.

**a) Set `OLLAMA_HOST=0.0.0.0` as a Windows system environment variable:**
```bash
powershell.exe -Command "[System.Environment]::SetEnvironmentVariable('OLLAMA_HOST', '0.0.0.0', 'Machine')"
```

**b) Add a Windows Firewall inbound rule for port 11434:**
```bash
powershell.exe -Command "if (-not (Get-NetFirewallRule -DisplayName 'Ollama WSL' -ErrorAction SilentlyContinue)) { New-NetFirewallRule -DisplayName 'Ollama WSL' -Direction Inbound -Action Allow -Protocol TCP -LocalPort 11434 | Out-Null; Write-Output 'Firewall rule created.' } else { Write-Output 'Firewall rule already exists.' }"
```

**c) Restart Ollama on Windows** so it picks up the new `OLLAMA_HOST` value.
   - Tell the user: "Please restart Ollama on Windows now (quit and relaunch it), then press Enter to continue."
   - Use `read -p` to pause and wait.

**d) Re-test connectivity** with the same curl command from Step 2. If it still fails, print a diagnostic checklist:
   - Ollama is running on Windows
   - Ollama was fully restarted *after* setting `OLLAMA_HOST`
   - No VPN or third-party firewall is blocking port 11434
   - The `OLLAMA_HOST` env var is set in the *system* scope (not just user scope) — verify with:
     ```bash
     powershell.exe -Command "[System.Environment]::GetEnvironmentVariable('OLLAMA_HOST', 'Machine')"
     ```
   Stop here and tell the user to resolve the connectivity issue before continuing.

### Step 4 — Update the project to use the Windows host IP

The goal is to use an env var (`OLLAMA_BASE_URL`) rather than hardcoding the IP, so the
value can be changed without touching source code.

**a) Find hardcoded Ollama URLs in the project:**
```bash
grep -rn "localhost:11434\|127\.0\.0\.1:11434" --include="*.py" --include="*.ts" --include="*.js" --include="*.env*" .
```

**b) Update or create `.env` in the backend directory** (or project root if no backend dir):
   - Add or update the line: `OLLAMA_BASE_URL=http://$HOST_IP:11434`
   - If a `.env.example` exists, add the same line there (with a placeholder comment):
     `OLLAMA_BASE_URL=http://<WINDOWS_HOST_IP>:11434  # get IP via: ip route show default | awk '{print $3; exit}'`

**c) Update source files** that hardcode `http://localhost:11434` or `http://127.0.0.1:11434`:

   For **Python** files using a hardcoded constant like `OLLAMA_URL = "http://localhost:11434/..."`:
   - Add `import os` if not present
   - Replace the hardcoded base with:
     ```python
     OLLAMA_BASE_URL = os.getenv("OLLAMA_BASE_URL", "http://localhost:11434")
     OLLAMA_URL = f"{OLLAMA_BASE_URL}/api/generate"
     ```
   - Load the `.env` file if `python-dotenv` is available (check `requirements.txt`):
     ```python
     from dotenv import load_dotenv
     load_dotenv()
     ```
     If `python-dotenv` is not in `requirements.txt`, add it.

   For **TypeScript/JavaScript** files:
   - Replace hardcoded URLs with `process.env.OLLAMA_BASE_URL ?? 'http://localhost:11434'`

### Step 5 — Print a summary

Print what was changed:
```
Windows host IP: <HOST_IP>
Ollama reachable: yes

Changes made:
  backend/.env              — added OLLAMA_BASE_URL=http://<HOST_IP>:11434
  backend/.env.example      — added OLLAMA_BASE_URL placeholder
  backend/main.py           — OLLAMA_URL now reads from OLLAMA_BASE_URL env var

To update the IP in the future (WSL gateway can change on restart):
  ip route show default | awk '{print $3; exit}'
  # then update OLLAMA_BASE_URL in backend/.env
```

## Rules

- Always detect the host IP dynamically — never ask the user to supply it.
- Never hardcode the detected IP in source files; always use an env var.
- Run `powershell.exe` commands one at a time and check output before proceeding.
- If `.env` already contains `OLLAMA_BASE_URL`, update it in place rather than appending a duplicate.
- Do not modify `.env` files that contain secrets beyond adding/updating `OLLAMA_BASE_URL`.
