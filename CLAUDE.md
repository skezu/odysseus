# Odysseus — Project Instructions

Self-hosted, local-first AI workspace (a ChatGPT/Claude-style UI you run on your own
hardware): chat + agents, deep research, model Cookbook, memory/skills, email,
calendar, notes/tasks, image editor. FastAPI backend, vanilla-JS frontend.

## Tech Stack
- **Language:** Python 3.11+ (backend), vanilla JS ES modules (frontend, **no build step**)
- **Web:** FastAPI + Uvicorn (ASGI), Starlette middleware
- **DB:** SQLite via SQLAlchemy (`DATABASE_URL`, default `sqlite:///./data/app.db`)
- **Vectors/RAG:** ChromaDB (HTTP client) + fastembed (local ONNX embeddings)
- **Models:** OpenAI-compatible endpoints — vLLM, llama.cpp, Ollama, OpenRouter, OpenAI
- **Bundled services (Docker):** ChromaDB, SearXNG (search), ntfy (push)
- **Tests:** pytest + pytest-asyncio (`asyncio_mode=auto`); some JS tests (`.mjs`/`.ts`)

## Architecture
- `app.py` — **slim orchestrator** and only entry point. Builds the FastAPI app,
  installs middleware (CORS → SecurityHeaders → RequestTimeout → Auth), then wires
  ~48 routers via `app.include_router(setup_X_routes(deps))`. Component singletons
  come from `src/app_initializer.py::initialize_managers`.
- `core/` — cross-cutting primitives: `auth`, `database` (SQLAlchemy models +
  `SessionLocal`), `middleware`, `constants`, `models`, `exceptions`,
  `session_manager`, `atomic_io`, `platform_compat`.
- `routes/` — HTTP surface, one module per feature. Each exposes a
  `setup_<feature>_routes(...) -> APIRouter` factory (dependencies injected as args,
  **not** module globals).
- `src/` — business logic (~80 modules): `agent_loop`, `agent_tools`, `llm_core`,
  `chat_processor`/`chat_handler`, `tool_*` (execution/parsing/schemas/security/index),
  `config`, `task_scheduler`, `mcp_manager`, `webhook_manager`.
- `services/` — integrations: `memory`, `research`, `search`, `hwfit` (Cookbook),
  `stt`, `tts`, `youtube`, `shell`, `docs`.
- `static/` — frontend: `index.html` + `app.js` + `js/` modules + `style.css`.
  Raw ES modules served with `Cache-Control: no-cache` (no bundler/versioned URLs).
- `mcp_servers/`, `companion/`, `integrations/` — built-in MCP servers, companion
  pairing, and Claude/Codex plugin bridges.

## Request Lifecycle
Request → CORS → `SecurityHeadersMiddleware` → `_RequestTimeoutMiddleware` (504 after
`REQUEST_HARD_TIMEOUT`, streaming/research paths exempt) → `AuthMiddleware`
(session cookie, `Bearer ody_…` API token, or loopback internal-tool token) →
feature router → handler → `core.database.SessionLocal()` (open/use/`close()` per call).

## Code Style & Conventions
- **Files:** `snake_case.py`. Route modules are `*_routes.py` with a
  `setup_*_routes()` factory returning an `APIRouter`. Match this when adding routes.
- **DI:** pass managers/handlers into route factories; avoid new module-level globals.
- **Errors:** raise domain exceptions from `core/exceptions.py`
  (`SessionNotFoundError`, `InvalidFileUploadError`, `LLMServiceError`,
  `WebSearchError`); they map to JSON responses via handlers in `app.py`.
- **Auth/ownership:** most data is owner-scoped. Check `request.state.current_user`
  / `api_token_owner`; honor admin-gating for shell, MCP, tokens, webhooks, settings.
- **No Unicode emoji in UI or code** — use inline monochrome SVG or plain text.
- **Frontend visual rules are strict:** reuse existing CSS vars (`--fg`, `--bg`,
  `--card`, `--border`, `--red`…), existing button/input/card classes, `Fira Code`
  mono, dark-theme-default. Don't add parallel components. See `CONTRIBUTING.md`.
- Comments explain **why** (platform gotchas, security rationale), matching the
  dense existing style in `app.py`.

## Build & Run
- **Dev (native):** `python -m uvicorn app:app --host 127.0.0.1 --port 7000`
  (first run: `python setup.py`). Windows: `.\launch-windows.ps1`.
- **Docker:** `docker compose up -d --build` → http://localhost:7000
- First boot creates an `admin` account and prints a temp password to the log.

## Testing / Checks (run the smallest relevant set)
- `python -m pytest`  (tests in `tests/`, ~370 files)
- `python -m py_compile app.py routes/*.py src/*.py`
- `node --check static/js/<file-you-changed>.js`  (frontend changes)
- For UI changes, **run the app and attach a screenshot** — type-checks aren't enough.

## Git / PR Workflow
- Branch model: **open PRs against `dev`** (not `main`; `main` is the curated release
  branch). You are currently on `dev`.
- Conventional-commit style: `fix:`, `feat:`, `chore:`, `refactor(scope):`, often with
  a trailing `(#PR)`. One focused change per PR; no broad rewrites/format-only churn.
- ⚠️ `CONTRIBUTING.md` asks LLM-agent contributors to **open an issue before opening a
  PR**, and auto-closes off-style bulk agent PRs. Surface this before pushing agent PRs.

## Where to Look
| Task | Location |
|------|----------|
| Add an API endpoint | new `routes/<feature>_routes.py` + `include_router` in `app.py` |
| Add a DB table/model | `core/database.py` |
| Add an agent tool | `src/tool_*.py` (schemas/execution/security) + `src/agent_tools.py` |
| Change chat/streaming | `routes/chat_routes.py`, `src/chat_processor.py`, `src/llm_core.py` |
| Add frontend UI | `static/js/` (extend existing module) + `static/index.html` |
| Configure deployment | `.env` (see `.env.example`), `docker-compose.yml` |
