# Project Index — Local Git Repo Dashboard

Build a React + FastAPI app that scans local git repos and produces an indexed dashboard with AI-generated summaries (via Ollama), run instructions, and code metrics, persisted in a JSON file.

---

## Architecture

- **Frontend**: React (Vite + TypeScript) with TailwindCSS and shadcn/ui
- **Backend**: FastAPI (Python) with uvicorn
- **Storage**: Single JSON file (`data/projects.json`)
- **AI**: Ollama (local) for generating natural-language summaries
- **Repo Analysis**: Python `git` + file-system walking for metrics

## Backend (`/backend`)

1. **FastAPI app** with endpoints:
   - `POST /projects` — accepts a local repo path, analyzes it, calls Ollama, stores result
   - `GET /projects` — returns all indexed projects
   - `GET /projects/{id}` — single project detail
   - `DELETE /projects/{id}` — remove from index
   - `POST /projects/{id}/refresh` — re-analyze a project
2. **Repo analyzer module** — given a path:
   - Validate it's a git repo
   - Count lines of code per language (simple heuristic via file extensions)
   - Detect predominant languages
   - Measure repo size (files, total bytes)
   - Extract last commit date, branch info
   - Read README if present
3. **Ollama integration** — send a prompt with repo metadata + README + sample file list to a local Ollama model and get back:
   - Summary of what the project does
   - How to run it
4. **JSON persistence** — read/write `data/projects.json`

## Frontend (`/frontend`)

1. **Dashboard page** — grid/list of all indexed projects with key stats (language badges, lines of code, last updated)
2. **Add Project** — input field/dialog to paste a local path; triggers analysis
3. **Project Detail view** — shows full summary, run instructions, language breakdown, size metrics
4. Modern styling with TailwindCSS + shadcn/ui components

## File Structure

```
project_index/
├── backend/
│   ├── main.py
│   ├── analyzer.py
│   ├── llm.py
│   ├── models.py
│   ├── requirements.txt
│   └── data/
│       └── projects.json
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   ├── tailwind.config.js
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── components/
│   │   └── pages/
│   └── ...
└── README.md
```

## Key Assumptions

- Ollama is already running locally (default `http://localhost:11434`)
- Model to use: `llama3` (configurable via env var `OLLAMA_MODEL`)
- Frontend runs on port 5173, backend on port 8000
- CORS configured for local dev

## Implementation Steps

1. Scaffold backend: FastAPI app, models, JSON storage helpers
2. Implement repo analyzer (git validation, LOC counting, metadata extraction)
3. Implement Ollama integration (prompt engineering, HTTP call)
4. Wire up API endpoints
5. Scaffold frontend (Vite + React + Tailwind + shadcn/ui)
6. Build dashboard UI (project list, add-project dialog, detail view)
7. Connect frontend to backend API
8. Add README with run instructions
