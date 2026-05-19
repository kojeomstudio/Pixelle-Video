# AGENTS.md — Pixelle-Video

AI-powered automated short video generation engine. Python 3.11+, FastAPI backend, Streamlit UI, OpenAI SDK for LLM, ComfyKit for image/video generation.

## Project Structure

```
pixelle_video/    # Core Python package (services, models, pipelines, prompts, utils)
api/               # FastAPI REST API (routers, schemas, tasks, dependencies)
web/               # Streamlit multi-page web UI
templates/         # HTML video frame templates organized by resolution
workflows/         # ComfyUI workflow JSON files (selfhost/, runninghub/)
tests/             # Test directory (pytest)
```

## Commands

### Setup

```bash
uv sync                          # Install all dependencies
uv pip install -e ".[dev]"       # Install with dev dependencies (pytest, ruff)
```

### Run

```bash
uv run streamlit run web/app.py             # Streamlit Web UI (port 8501)
uv run python api/app.py                    # FastAPI API (port 8000)
uv run python api/app.py --host 0.0.0.0 --port 8080 --reload  # Custom API
```

### Lint

```bash
uv run ruff check .                # Lint all files (rules: E, F, I)
uv run ruff check pixelle_video/services/llm_service.py   # Lint single file
uv run ruff check --fix .          # Auto-fix lint issues
```

### Test

```bash
uv run pytest -v                                   # Run all tests
uv run pytest tests/test_foo.py -v                 # Run single test file
uv run pytest tests/test_foo.py::test_bar -v       # Run single test function
uv run pytest -k "test_llm" -v                     # Run tests matching pattern
```

- Test framework: **pytest** with **pytest-asyncio** (asyncio_mode = "auto")
- Test directory: `tests/`
- Async tests work automatically; no decorator needed

### Build

```bash
uv build                         # Build package (hatchling)
docker-compose up -d             # Docker (API + Web UI)
```

## Code Style

### File Header

Every `.py` file must start with the Apache 2.0 copyright header:

```python
# Copyright (C) 2025 AIDC-AI
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#     http://www.apache.org/licenses/LICENSE-2.0
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
```

### Module Docstrings

Immediately after the header, every module has a triple-quoted docstring describing its purpose.

### Imports

Order: stdlib → third-party → local (enforced by Ruff `I` rule). One blank line between groups.

```python
import json
from typing import Optional, Type, TypeVar, Union

from openai import AsyncOpenAI
from pydantic import BaseModel
from loguru import logger

from pixelle_video.services.llm_service import LLMService
```

### Formatting

- **Line length:** 100 characters (E501 is ignored in ruff)
- **Indentation:** 4 spaces
- **String quotes:** Double quotes preferred
- **Trailing commas:** Used in multi-line collections and function signatures

### Type Annotations

Use type annotations consistently on all function signatures and return types.

```python
from typing import Optional, List, Dict, Any, Literal, TypeVar

T = TypeVar("T", bound=BaseModel)

def build_prompt(self, topic: str, n_frames: int = 5) -> str:
async def generate(self, prompt: str, **kwargs) -> Optional[bytes]:
```

### Naming Conventions

| Element | Style | Example |
|---|---|---|
| Classes | PascalCase | `PixelleVideoCore`, `LLMService` |
| Functions/methods | snake_case | `build_image_prompt`, `_get_config_value` |
| Private methods | _prefix | `_concat_demuxer`, `_report_progress` |
| Constants | UPPER_SNAKE_CASE | `DEFAULT_MODEL` |
| Variables | snake_case | `api_key`, `base_url` |

### Data Models

- **Config / API schemas:** Pydantic `BaseModel` with `Field()` descriptors and descriptions
- **Domain models:** Python `@dataclass` with inline comments for each field

```python
@dataclass
class StoryboardFrame:
    """Single storyboard frame"""
    index: int
    narration: str
    audio_path: Optional[str] = None
```

```python
class LLMConfig(BaseModel):
    """LLM configuration"""
    api_key: str = Field(default="", description="LLM API Key")
    model: str = Field(default="", description="LLM Model Name")
```

### Docstrings

Google-style docstrings with `Args`, `Returns`, `Raises`, `Examples`, `Note` sections:

```python
def _get_config_value(self, key: str, default=None):
    """
    Get config value dynamically from config_manager (supports hot reload)

    Args:
        key: Config key name
        default: Default value if not found

    Returns:
        Config value
    """
```

### Error Handling

- Catch exceptions and re-raise with context: `RuntimeError(f"Failed to ...: {error_msg}")`
- Use `logger.error()`, `logger.warning()` for error logging
- Prefer graceful degradation over hard failures where possible

### Async Patterns

- Heavy use of `async/await` throughout services
- `AsyncOpenAI` for LLM calls
- Async context managers (`__aenter__`/`__aexit__`) for service lifecycle
- `asyncio_mode = "auto"` in pytest config — no `@pytest.mark.asyncio` needed

### Logging

Use `loguru` logger throughout. Emoji prefixes are used in log messages:

```python
from loguru import logger
logger.info("🚀 Starting video generation...")
logger.success("✅ Video generation complete")
logger.error("❌ Failed to generate video")
```

### Path Handling

Always use `pathlib.Path` for path manipulation. Never concatenate paths with string operations.

### Configuration

- Config is managed via `config_manager` singleton from `pixelle_video.config`
- Schema defined in `pixelle_video/config/schema.py` using Pydantic models
- Config values are read dynamically (not cached) to support hot reload
- YAML file (`config.yaml`) validated against Pydantic schema

### Architecture Patterns

- **Service layer:** `PixelleVideoCore` in `service.py` provides unified access to all capabilities
- **Pipeline pattern:** Abstract `BasePipeline` with `__call__`; concrete pipelines in `pipelines/`
- **Progress callbacks:** Pipelines accept callback functions for real-time progress reporting
- **Global singletons:** `pixelle_video` and `config_manager` are module-level singletons

## Ruff Configuration

```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I"]   # pycodestyle errors, pyflakes, isort
ignore = ["E501"]           # line too long
```

## Key Dependencies

| Package | Purpose |
|---|---|
| fastapi + uvicorn | REST API |
| streamlit | Web UI |
| openai | LLM integration (OpenAI-compatible SDK) |
| comfykit | ComfyUI workflow integration |
| pydantic | Config and schema validation |
| loguru | Logging |
| edge-tts (pinned 7.2.7) | Text-to-speech |
| ffmpeg-python + moviepy | Video processing |
| playwright | HTML template rendering |
| httpx | Async HTTP client |

## Important Notes

- `edge-tts` version is pinned to `==7.2.7` for stability — do not upgrade without testing
- When adding new ComfyUI workflows, place JSON files in `workflows/selfhost/` or `workflows/runninghub/`
- Template files follow naming convention: `static_*.html` (static scenes), `image_*.html` (image scenes), `video_*.html` (video scenes)
- The project is bilingual (Chinese primary, English secondary)
