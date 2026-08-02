# AGENTS.md — KrishiSaarthi AI

## Project Structure

Two independent apps (not a monorepo). No shared package manager or build system.

```
be-MVP5/   → FastAPI backend (Python)
fe-MVP5/   → React + Vite frontend (JS)
```

Convenience start scripts exist: `be-MVP5/start_backend.sh`, `fe-MVP5/start_frontend.sh`.

## Backend (`be-MVP5/`)

**Stack:** FastAPI, SQLAlchemy 2.0, SQLite, LangChain, ChromaDB, scikit-learn, Pillow.

**Start:**
```bash
cd be-MVP5
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
[ -f .env ] || cp .env.example .env
python seed.py
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

**Test:**
```bash
cd be-MVP5
MOCK_LLM=true pytest tests/
```

**Key facts:**
- `MOCK_LLM=true` skips all LLM calls (deterministic fallbacks run instead). Set to `false` for live model calls.
- `API_KEY` is required for live LLM mode. The `.env.example` contains a sample key.
- Database is SQLite at `be-MVP5/krishisaarthi.db`. Tables auto-created on startup; `seed.py` populates defaults.
- Deletes `krishisaarthi.db` to reset the database; re-run `python seed.py` or restart the server.
- ChromaDB vector index lives in `be-MVP5/chroma_index/`. Regenerate via the `/api/v1/knowledge/reindex` endpoint (admin-only).
- Uploaded files go to `be-MVP5/uploads/` (images, soil reports, knowledge docs). These are gitignored.
- `TIKTOKEN_CACHE_DIR` defaults to `tiktoken_cache/` relative to the backend dir. The root-level `tiktoken_cache/` is a pre-populated cache for offline installs.
- Env loading order: project-root `.env` first, then `be-MVP5/.env` (overrides).

**API prefix:** All routes are under `/api/v1`.

**Auth:** Header-based role system (no JWT). Send `X-App-Role` and `X-App-User` headers. Legacy headers `X-Demo-Role`/`X-Demo-User` are also accepted. Roles: `farmer`, `officer`, `admin`. Demo users: `farmer/farmer123`, `officer/officer123`, `admin/admin123`.

**Agent pipeline** (`POST /api/v1/advisories/{id}/analyze`):
1. Supervisor Agent plans tool sequence (deterministic fallback on failure)
2. Image Analysis Agent (vision model, optional)
3. Disease Rule Agent (keyword matching + LLM selection)
4. Weather, Soil, RAG retrieval
5. Advisory generation + guardrail validation
6. Translation (if non-English)

**Key gotchas:**
- `FRONTEND_ORIGINS` must match the frontend URL or CORS blocks requests.
- Image uploads require `content_type` starting with `image/` and minimum 160×160 px.
- `MOCK_LLM=false` with missing `API_KEY` causes runtime errors in the analysis pipeline.
- The seed database auto-runs on server startup (`main.py:26`), so manual `seed.py` is only needed for a clean DB.

## Frontend (`fe-MVP5/`)

**Stack:** React 18, Vite, React Router, Axios, Recharts.

**Start:**
```bash
cd fe-MVP5
[ -f .env ] || cp .env.example .env
npm install
npm run dev
```

**Build:** `npm run build` (output to `dist/`).

**Key facts:**
- Dev server runs on port 5173. Backend must be on port 8000.
- `.env` sets `VITE_API_URL=http://localhost:8000/api/v1` and `VITE_BACKEND_URL=http://localhost:8000`.
- Routes are role-gated in `App.jsx` via `<Protected roles={[...]}>`. Changing role access requires editing both frontend routes and backend `require_role()` calls.

**No lint/typecheck/test scripts defined.** The frontend has no `lint`, `typecheck`, or `test` scripts in `package.json`.

## Common Mistakes

- Starting the frontend without the backend → API calls fail silently or show fallback data.
- Forgetting `python seed.py` on a fresh DB → missing prompts, model settings, and seed knowledge docs.
- Editing routes in `main.py` without checking `App.jsx` role guards → access control breaks.
- Changing env vars without restarting the server → old values cached in memory.
- Running `pytest` without `MOCK_LLM=true` → tests require live API key and will fail.

## Reference

- `CHANGES.md` — detailed before/after changelog for past code modifications.
