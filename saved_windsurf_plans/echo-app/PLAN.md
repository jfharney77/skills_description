# Echo Web Application Plan with Railway Deployment

Build a simple web application with a React frontend and FastAPI backend that echoes user input with file-based logging, using project skills for executable scripts, following the backend logging rule, and preparing the backend for Railway deployment without automatic deployment.

## Project Structure
- `/backend/` - FastAPI Python application
- `/frontend/` - React TypeScript application
- `logs/` - Directory for timestamped log files
- `run-backend.sh` - Executable script to start backend (from skill)
- `run-frontend.sh` - Executable script to start frontend (from skill)

## Use Project Skills

1. **run-backend skill**: Generate `run-backend.sh` script
   - Uses `uv` for Python dependency management
   - Starts FastAPI backend with `uv run uvicorn`
   - Defaults to port 8000
   - Mark executable with chmod +x

2. **run-frontend skill**: Generate `run-frontend.sh` script
   - Navigates to frontend directory
   - Starts React dev server
   - Defaults to port 3000
   - Mark executable with chmod +x

3. **prepare-for-railway skill**: Prepare backend for Railway deployment
   - Create `backend/railway.toml` configuration file
   - Create `backend/requirements.txt` for Python dependencies
   - Create `backend/.env.example` for environment variables template
   - Update CORS configuration to use environment variables
   - Add `/health` endpoint for Railway health checks

## Follow Backend Logging Rule

Per the `backend_logging.md` rule, all backend logs must:
- Be written to text files
- Include timestamps (ISO format)
- Use a dedicated logs directory
- Use descriptive log file names
- Append to log files rather than overwriting
- Include relevant context in log messages

## Follow No-Internal-Deploy Skill

Per the `no-internal-deploy.md` skill:
- Do NOT start the backend server after implementation
- Do NOT start the frontend server after implementation
- Do NOT provide browser previews
- Update README with manual startup instructions

## Backend Implementation (FastAPI)
1. Create FastAPI application with POST endpoint `/echo`
2. Endpoint accepts JSON payload with `text` field
3. Log each request with timestamp to `logs/echo.log` (following backend_logging rule)
4. Return the echoed text in JSON response
5. Add CORS middleware to allow frontend requests (with environment variable support)
6. Add `/health` endpoint for Railway health checks
7. Use uv for dependency management
8. Prepare for Railway deployment (per prepare-for-railway skill)

## Frontend Implementation (React + TypeScript)
1. Create React app with TypeScript using Vite
2. Build simple UI with:
   - Text input field
   - Submit button
   - Display area for echoed response (most recent only)
3. Use modern styling with Tailwind CSS
4. Implement fetch call to backend `/echo` endpoint on button click
5. Display most recent echo response in the UI
6. Handle loading states and errors

## File Layout
```
token_center/
├── backend/
│   ├── main.py
│   ├── pyproject.toml
│   ├── railway.toml        # Railway configuration
│   ├── requirements.txt     # Python dependencies for Railway
│   ├── .env.example         # Environment variables template
│   └── logs/
│       └── echo.log
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   └── ...
│   ├── package.json
│   └── ...
├── run-backend.sh          # Generated from skill
├── run-frontend.sh         # Generated from skill
└── README.md               # Manual startup instructions
```

## Manual Startup Instructions (per no-internal-deploy)

After implementation, the user can start the servers manually using:

**Backend:**
```bash
./run-backend.sh
# or
cd backend && uv run uvicorn main:app --reload --port 8000
```

**Frontend:**
```bash
./run-frontend.sh
# or
cd frontend && npm run dev
```

Backend runs on http://localhost:8000
Frontend runs on http://localhost:5173 (or next available port)

## Railway Deployment Instructions

After implementation, the backend is ready for Railway deployment:

1. Push changes to GitHub
2. Deploy via Railway
3. Set backend directory as root in Railway settings
4. Configure CORS_ORIGINS environment variable
5. Railway will use `/health` endpoint for health checks
