# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

`AGENTS.md` in the repo root has the full code-style + dependency reference. Read it for naming, formatting, and copyright-header rules. This file focuses on architecture and the day-to-day commands.

## Commands

Dependency manager is **uv**. Tests live under `tests/` (not yet present in the tree — create it before running pytest).

```bash
# Setup
uv sync
uv pip install -e ".[dev]"

# Run services
uv run streamlit run web/app.py                              # Streamlit UI on :8501
uv run python api/app.py                                     # FastAPI on :8000
uv run python api/app.py --host 0.0.0.0 --port 8080 --reload

# Lint (ruff: rules E,F,I; line-length 100; E501 ignored)
uv run ruff check .
uv run ruff check --fix .
uv run ruff check pixelle_video/services/llm_service.py     # single file

# Test (pytest-asyncio in auto mode — no @pytest.mark.asyncio needed)
uv run pytest -v
uv run pytest tests/test_foo.py::test_bar -v
uv run pytest -k "test_llm" -v

# Build / Docker
uv build
docker-compose up -d
```

CLI entry points (defined in `pyproject.toml`): `pixelle-video` and `pvideo` → `pixelle_video.cli:main`.

## Architecture

The system is a 4-layer monorepo: **CLI/UI → API → Core service → ComfyUI workflows**. Understanding the seams between these layers matters more than any individual file.

### Core service layer (`pixelle_video/`)

`PixelleVideoCore` (`pixelle_video/service.py`) is the singleton hub everything else talks to. It is exposed as `from pixelle_video import pixelle_video`. After `await pixelle_video.initialize()` it owns:

- **Services** (`pixelle_video/services/`): `llm` (OpenAI-compatible SDK), `tts` (Edge-TTS or ComfyUI workflow), `media` (image+video, ComfyUI), `image_analysis`, `video_analysis`, `video` (ffmpeg/moviepy compositor), `frame_processor`, `persistence`, `history`. `media` is also aliased as `image` for backward compat.
- **Pipelines** (`pixelle_video/pipelines/`): `standard`, `custom`, `asset_based`, `linear`. All inherit `BasePipeline` and implement `async def __call__(text, progress_callback, **kwargs) -> VideoGenerationResult`. Pipelines are registered in `service.py::initialize()` — adding a pipeline means importing it and adding to the `self.pipelines` dict there.
- **ComfyKit** is **lazily** initialized and recreated when its config hash changes (see `_get_or_create_comfykit` and `_compute_comfykit_config_hash`). Don't construct ComfyKit directly elsewhere — go through the core so config hot-reload works.

Config flows through `pixelle_video/config/` (Pydantic schema → YAML loader → `config_manager` singleton). Services read via `config_manager.config.to_dict()` on each access; do not cache config locally if you want hot reload to work.

### API layer (`api/`)

FastAPI app (`api/app.py`) with routers under `api/routers/` — one per capability: `llm`, `tts`, `image`, `content`, `video`, `tasks`, `files`, `resources`, `frame`, plus `health`. All API routes are mounted under `api_config.api_prefix` (typically `/api`) except `/health`.

- Long-running video generation has both **sync** (`/api/video/generate/sync`) and **async** (`/api/video/generate/async` + `/api/tasks/{task_id}` polling) flavors.
- Background work goes through `api/tasks/task_manager`, started/stopped in `lifespan()`.
- All routers depend on the same `pixelle_video` core via `api/dependencies.py` (`shutdown_pixelle_video()` is the cleanup hook).

### Web layer (`web/`)

Streamlit multi-page app with `web/app.py` as the entry, pages declared via `st.navigation` in `pages/`. There is a parallel `web/pipelines/` (`standard`, `custom`, `asset_based`, `digital_human`, `i2v`, `action_transfer`) — these are **UI-side pipeline wrappers** for the Home page, not the same objects as `pixelle_video/pipelines/`. The UI calls into the core, but each UI pipeline file owns the form/state for one mode of generation. When adding a new generation mode end-to-end you typically need a core pipeline AND a UI pipeline file.

### Workflows (`workflows/`)

ComfyUI workflow JSON files split by execution backend:
- `selfhost/` — your own ComfyUI server (uses `comfyui_url` from config)
- `runninghub/` — RunningHub cloud service (uses `runninghub_api_key`)

Config keys `comfyui.image.default_workflow`, `comfyui.video.default_workflow`, `comfyui.tts.default_workflow` reference these by relative path (e.g. `runninghub/image_flux.json`). When adding a workflow, drop the JSON in the right directory — no code change needed for the loader to find it.

### Templates (`templates/`)

HTML frame templates organized by resolution (`1080x1080`, `1080x1920`, `1920x1080`). Naming is **load-bearing**:
- `static_*.html` — no AI-generated media
- `image_*.html` — requires AI-generated images
- `video_*.html` — requires AI-generated videos

`frame_processor` and `frame_html` services consume these and Playwright renders them to frames. Default template comes from `template.default_template` in config.

## Things to know before changing code

- `edge-tts` is **pinned to 7.2.7** — past upgrades broke TTS. Don't bump without testing.
- ComfyKit is recreated on config change via hash comparison. If you cache anything derived from the ComfyKit instance, invalidate it the same way.
- Both `api/app.py` and `web/app.py` manually prepend the project root to `sys.path` so `pixelle_video` resolves when run as a script. Keep that pattern if you add another entry point.
- Project is bilingual: README is Chinese-primary, README_EN.md is the English mirror, log messages and docstrings are mixed. Don't assume English-only text in user-facing strings.
- There is no `tests/` directory yet — create it (and `tests/__init__.py` if needed) when adding the first test. `pyproject.toml` already declares `testpaths = ["tests"]`.
