# Word-to-LaTeX System Plan
This plan outlines how we will add a docx-to-LaTeX CLI plus a FastAPI + React document viewer on top of the existing research_papers repository.

## 1. Repository understanding
- Documented the `latex/*` templates (IEEE, ACM, NeurIPS, AAAI) with sectionized inputs and ready-made build scripts under `script/latex/*`.
- Noted the existing FastAPI simulation app under `src/apip/backend`, which we can reuse for project scaffolding and deployment patterns but will keep separate from the new services.
- Confirmed utilities such as `pyproject.toml` (FastAPI + uvicorn only) so we’ll expand dependencies (e.g., `python-docx`, `pypandoc`, `jinja2`) as needed for parsing and templating.

## 2. CLI conversion pipeline
1. Add a `cli/` package exposing a `papers` command (`typer` or `click`) that accepts `--docx <path>` and `--template <name>`.
2. Implement ingestion: validate template directory in `/latex/<name>`, ensure docx exists, derive a snake_case workspace name, and create `documents/<title>/` with the original docx plus `<template>/sections|references|output` subfolders (matching the user’s structure).
3. Parsing & decomposition:
   - Use `python-docx` to scan headings, build a hierarchical outline, and split content into sections/subsections.
   - Detect figures (inline images/captions) and export them into template-specific asset folders, inserting `\includegraphics` blocks referencing the saved files.
   - Detect bibliography cues (e.g., “References” heading) and convert entries into BibTeX via heuristics or `citeproc-py` fallback, writing `references/references.bib`.
4. Rendering: feed section content through lightweight templates (Jinja) that map Word styles to LaTeX macros per venue (e.g., IEEE adds `\IEEEPARstart`, NeurIPS uses standard `\section`).
5. Compilation hooks: optionally invoke the existing `script/latex/<template>/build.sh` inside the new workspace (copying or symlinking `main.tex` + style files) to populate `output/<title>.pdf`.
6. Persist metadata (JSON manifest) describing sections, assets, and compilation status so the backend/UI can enumerate conversions without reparsing docx files.

## 3. FastAPI backend service
1. Stand up a new `src/docserve` package (separate from APIP) with Pydantic models covering Document, Section, Asset, and BuildJob.
2. Core storage layer reads the `documents/` directory, loads manifests, and provides CRUD helpers for sections and generated artifacts.
3. Endpoints:
   - `POST /documents` accepts docx upload + template choice, queues conversion by invoking the CLI module (in-process task) and returns job status.
   - `GET /documents` / `GET /documents/{id}` list metadata, section tree, and compilation results.
   - `GET /documents/{id}/sections/{section_id}` returns LaTeX source; `GET /documents/{id}/pdf` streams the compiled PDF; `GET /documents/{id}/word` streams the original docx.
   - `POST /documents/{id}/compile` re-runs the LaTeX build pipeline (helpful after manual edits).
4. Add background tasks (FastAPI `BackgroundTasks` or `asyncio.create_task`) for long-running conversions/compiles, with progress persisted in the manifest.
5. Provide basic observability: structured logging around conversions and error propagation so the frontend can surface failures (missing template, parse error, LaTeX compile error).

## 4. React frontend viewer
1. Create a `web/` directory (Vite + React + TypeScript) that talks to the new FastAPI endpoints.
2. Screens:
   - **Library view**: grid/table of converted documents showing title, template, last modified, compile status, and quick actions (view PDF, sections, original docx).
   - **Document workspace**: split layout with sidebar section tree and main content area offering three tabs—PDF preview (using `react-pdf` / PDF.js), LaTeX section viewer/editor (syntax highlighting), and Word preview/download (render plaintext summary plus download link; optionally embed via `mammoth` HTML view).
   - **Upload flow**: drag-and-drop docx chooser with template selector that calls `POST /documents` and shows conversion progress.
3. State management via React Query or SWR for caching document metadata and streaming build progress via polling or WebSocket (optional stretch goal).
4. Styling: align with project branding (light neutral palette) and ensure responsive layout so sections/PDFs are usable on tablets.

## 5. Validation & developer experience
- Unit-test the parser (heading extraction, figure export, reference block detection) using fixtures in `tests/cli/`.
- Add integration tests that run the CLI against a sample docx and assert that the manifest, section files, and PDF exist.
- Provide `make` or `uv` scripts to run the backend and frontend together, plus documentation updates in `README.md` covering CLI usage and the new web experience.
