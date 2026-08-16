# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

Voicebox — a local-first AI voice studio (TTS + STT + local LLM) shipped as a Tauri
desktop app. A Python FastAPI backend does all inference; a React frontend is shared
between the Tauri shell and a browser build.

## Layout

```
app/       Shared React frontend (components, hooks, stores, lib/api)
tauri/     Desktop shell — Vite frontend wrapper + src-tauri/ (Rust)
web/       Browser build of the same frontend (thin platform shim)
landing/   Marketing site
docs/      Fumadocs site — content lives in docs/content/docs/
backend/   FastAPI server (see below)
scripts/   Build & release scripts (PyInstaller, icons, asset conversion)
```

`backend/` internals:

- `main.py` — uvicorn entry point; `app.py` — FastAPI factory (CORS, lifespan)
- `server.py` — Tauri sidecar launcher with a parent-pid watchdog
- `routes/` — HTTP layer, one module per resource. Validate input only.
- `services/` — business logic (`generation.py`, `task_queue.py`, `tts.py`, …)
- `backends/` — TTS/STT/LLM engine implementations behind a common protocol
  (`base.py`); every model is declared as a `ModelConfig` in `backends/__init__.py`
- `database/` — SQLAlchemy + SQLite; `utils/` — audio processing helpers

Request flow: **routes → services → backends → utils**. Keep it in that direction —
routes should not import from `backends/`, and `backends/` should not know about HTTP.

## Commands

Everything goes through `just` (see `justfile`, `just --list`).

| Task | Command |
| --- | --- |
| Full setup (venv + JS deps + dev sidecar) | `just setup` |
| Backend + desktop app | `just dev` |
| Backend + browser app (no Rust build) | `just dev-web` |
| Backend only | `just dev-backend` |
| Stop everything | `just kill` |
| Lint + format + typecheck | `just check` |
| Auto-fix | `just fix` |
| Python tests | `just test` |
| Regenerate the TS API client | `just generate-api` (backend must be running) |
| Tail backend logs | `just logs` |

The backend always listens on **127.0.0.1:17493**. API docs at `/docs`, MCP server
mounted at `/mcp`.

Prefer `just dev-web` over `just dev` when the change is frontend- or backend-only —
it skips the Rust compile entirely.

## Conventions

**Python** (`backend/STYLE_GUIDE.md` is authoritative — read it before writing backend code):

- Target 3.12+. Ruff for lint + format, config in `backend/pyproject.toml`. 120-char
  lines, 4-space indent, double quotes.
- Built-in generics and `X | None` — never `typing.List` / `Optional`, and no
  `from __future__ import annotations`.
- Relative imports inside the `backend` package (`from .database import get_db`).
- Google-style docstrings on public functions, classes, and modules.
- Heavy imports (torch, transformers, mlx) may be lazy-imported inside functions;
  mark them `# lazy: heavy import`.
- ORM models are aliased with a `DB` prefix on import
  (`from .database import VoiceProfile as DBVoiceProfile`); Pydantic models use
  `…Create` / `…Response` / `…Request` suffixes.

**TypeScript/React** — Biome (`biome.json`), 2-space indent, 100-char lines, single
quotes, semicolons, trailing commas. `noUnusedImports` and `useHookAtTopLevel` are
errors. State is Zustand + React Query. The API client in `app/src/lib/api/` is
**generated** — change the backend and run `just generate-api`, don't hand-edit it.

**Adding a TTS engine** — follow `docs/content/docs/developer/tts-engines.mdx`; there is
an agent skill at `.agents/skills/add-tts-engine/SKILL.md` that automates the whole
integration.

## Things that bite

- **Dependency pins are deliberate.** `chatterbox-tts` and `hume-tada` are installed
  `--no-deps` because their pins (numpy<1.26 / torch==2.6 / torch<2.8) are incompatible
  with the rest of the stack; their real sub-dependencies are listed explicitly in
  `backend/requirements.txt`. Don't "fix" this by dropping `--no-deps`.
- **`numpy>=1.24,<2.0` and `transformers<=4.57.6` are hard caps.** Several engines break
  above them.
- **Frozen-build constraints.** `en_core_web_sm` and `unidic-lite` are pinned as wheels
  because runtime downloads (`spacy.cli.download`, `python -m unidic download`) crash
  PyInstaller builds. Keep any new model asset resolvable at install time.
- **Models are large.** Sizes range from 350 MB (Kokoro) to 8 GB (TADA 3B) and download
  from HuggingFace on first use. `VOICEBOX_MODELS_DIR` overrides the cache location.
- The root `requirements.txt` is a stub — `backend/requirements.txt` is the real one.
- `data/` is gitignored and holds the SQLite DB, profiles, and generations.

## Local dev on this machine

See [`docs/SETUP_THIS_LAPTOP.md`](docs/SETUP_THIS_LAPTOP.md) for the Ubuntu 24.04 /
RTX 3050 Ti setup walkthrough, including which models actually fit in 4 GB of VRAM.
