# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This repo contains **three independent apps** that share a git history but are otherwise unrelated:

1. **`web/` — AI Storyboard Pro v2.2** (Python, Gradio + FastAPI): the main product. Turns scripts into professional storyboards (characters/scenes/shots → AI-generated images). This is where most active work happens.
2. **Repo root — `director_ai` "AI漫导"** (Flutter/Dart): a mobile AI-agent chat app that turns text ideas into videos via a ReAct loop. Separate codebase, not part of the storyboard platform.
3. **`web/mobile/` — `ai_storyboard_pro`** (Flutter): an empty scaffold (no `lib/` yet) intended as the mobile client for the storyboard platform.

Note: the root `README.md` is a leftover Vite/React template — ignore it. The real documentation lives in `web/README.md` and the Chinese design docs under `docs/`.

## web/ — AI Storyboard Pro (Python)

### Commands

Run from `web/` (dependencies live in the repo-root `.venv`, Python 3.13):

```bash
pip install -r requirements.txt          # install deps
python setup_wizard.py                    # first-time interactive setup (writes .env)
python app.py                             # Gradio UI at http://localhost:7861
python api_server.py                      # FastAPI REST API at http://localhost:8000 (docs at /docs)
# or use the launcher scripts:
./start.sh / start.bat                    # UI, kills whatever is on port 7861 first
./start_api.sh / start_api.bat            # API, kills whatever is on port 8000 first
```

There is no meaningful automated test suite (no tests exist; `test_api.py` at root is empty). `pytest`/`black`/`flake8` are mentioned in README but commented out of `requirements.txt`.

### Configuration

- **`settings.py` is the single source of config**: a `Settings` dataclass loaded from `web/.env` at import time (it parses `.env` itself, no python-dotenv needed). Global singleton `settings` + `get_settings()`, `reload_settings()`, `needs_setup()`.
- **`config.py` is a legacy compatibility shim** that re-exports values from `settings.py` under old names (e.g. `NANA_BANANA_API_KEY` → `settings.api_key`, `CANGHE_API_KEY`). New code imports from `settings`; `config.py` exists so old modules still work.
- `web/.env` is gitignored (template: `.env.example`). Copy it and set `CANGHE_API_KEY`.
- The platform supports **two image-generation backends**, selected by `IMAGE_BACKEND`:
  - `canghe` (default) — cloud 苍何 API, needs `CANGHE_API_KEY`.
  - `comfyui` — local ComfyUI server (`COMFYUI_ENABLED=true`, host/port in `.env`).
- Important settings: `GRADIO_PORT` (7861), `API_PORT` (8000), `CORS_ORIGINS`, `MAX_UPLOAD_SIZE_MB`, `ALLOWED_EXTENSIONS`.

### Architecture

Layering, from UI down to image generation:

- **`app.py`** — the Gradio UI. Huge (~12.6k lines). `create_ui()` builds the interface; the `if __name__ == "__main__"` block (line ~12616) validates config, runs the setup wizard if needed, connects ComfyUI, and launches. `app_v2.1_backup.py` is a full backup of the previous version — treat as dead code.
- **`gradio_adapter.py`** — thin adapter between the Gradio UI and the services layer (project/character/scene/shot CRUD + generation). The UI is meant to be able to migrate to another frontend by rewriting this adapter.
- **`api_server.py`** — FastAPI REST server exposing the same features over HTTP (projects, characters, scenes, shots, generation, smart import, export). Uses Pydantic schemas from `schemas.py`.
- **`services.py`** — business logic. Exposes a global `services` singleton (built from `ProjectService`, `CharacterService`, `SceneService`, `ShotService`). State is a **single in-memory `StoryboardProject`** (`services.project.current_project`) — no persistence, so a restart loses the working project. `EXAMPLE_STORIES` (内置故事范例) are defined here.
- **`models.py`** — domain models: `StoryboardProject`, `Character`, `Scene`, `Prop`, `Shot`, `StyleConfig`, `ShotTemplate` (enum: 全景/中景/特写/过肩/低角度/跟随), `SlotWeights`, camera/composition settings. Each entity has `ref_images` and a `consistency_weight` used for visual consistency across shots.
- **`templates.py`** — predefined shot templates (camera angle, composition) keyed by `ShotTemplate`.
- **`prompt_generator.py` / `shot_prompt_generator.py`** — build the text-to-image prompt for a shot from the shot description + template + project consistency context.
- **`image_generator.py`** — `create_generator()` factory + Canghe image client. `generate_shot()` collects reference images (character/scene/prop/style refs + slot weights) into the prompt. Model fallback chain: nano-banana (fal-ai) → DALL-E 3 → jimeng (and notes jimeng is video-only).
- **`canghe_api.py`** — `CangheAPIClient`: text-to-image (nano-banana), image edit, and video generation endpoints (v1/video + jimeng), with poll-until-done loops.
- **`comfyui_client.py`** — `ComfyUIClient` for local ComfyUI generation (used when backend is comfyui).
- **`smart_import.py`** — script import from PDF/DOCX/Markdown/HTML/text.
- **`schemas.py`** — Pydantic request/response models for the FastAPI server.
- Supporting/experimental modules: `video_analyzer.py` (86k), `ai_creative_generator.py`, `storyboard_ai_generator.py`, `export_storyboard_pdf.py`/`export_storyboard_ep1_full.py`/`export_storyboard_part1.py` (PDF/JSON export scripts), `setup_wizard.py`.

Runtime output directories (all auto-created by `settings.ensure_directories()`): `outputs/`, `exports/`, `projects/`, `uploads/`, `assets/{characters,scenes,props,styles}` — all gitignored except `.gitkeep`.

## Root Flutter app — director_ai ("AI漫导")

An AI-agent mobile app that converts text ideas into videos. All code is under `lib/`; UI strings and comments are in Chinese.

### Commands

```bash
flutter pub get
flutter run
flutter build apk
dart run build_runner build        # regenerate Hive .g.dart files after editing models
flutter test                        # no test files exist yet
```

### Architecture

- **`main.dart`** — app entry. `DirectorAIApp` wires three `ChangeNotifierProvider`s (`ChatProvider`, `ConversationProvider`, `VideoMergeProvider`) into a Material 3 purple-anime theme. Routes: `/settings`, `/screenplay-review`, `/logs`.
- **`controllers/agent_controller.dart`** — the core: a **ReAct loop**. It streams messages to 智谱 GLM, parses the response into an `AgentCommand` (models/agent_command.dart), executes the tool, and repeats (max 10 iterations) until the LLM emits a `complete` command. Tracks thinking/video/image/error message types and builds a synthetic tool-history for the next GLM call.
- **`providers/chat_provider.dart`** — `ChatProvider` orchestrates the controller and screen-draft generation; `conversation_provider.dart` manages persisted conversation history.
- **`services/api_service.dart`** — `ApiConfig` holds the three AI providers:
  - 智谱 GLM (`open.bigmodel.cn`) — the LLM driving the agent.
  - 词元 API (`ciyuan.today`) — video & image generation.
  - 豆包 ARK (`ark.cn-beijing.volces.com`) — image generation.
  - `ApiConfig` exposes **mock switches** (`USE_MOCK_VIDEO_API`, `USE_MOCK_IMAGE_API`, `USE_MOCK_CHARACTER_SHEET_API`) and `USE_THINKING_MODE`, plus `sceneCount` / `concurrentScenes` tuning knobs.
- **`services/api_config_service.dart`** — reads/writes API keys (shared_preferences) initialized before `runApp`.
- **`services/screenplay_parser.dart`** — parses & validates the screenplay JSON the LLM returns.
- **`models/`** — Hive-serialized models (`.g.dart` via build_runner), chat messages, conversations, screenplays/drafts, character sheets.
- **`cache/`** — Hive + media cache managers for offline reuse.
- **`utils/app_logger.dart`** — structured logging (initialized at startup, disposed on app detach).

The video-generation workflow (script draft → character three-view sheets → preview/confirm → per-scene image+video generation, scenes processed in parallel) is documented in `docs/视频生成完整流程.md`.

## Conventions

- **Comments and user-facing strings are in Chinese** — match this when editing (e.g. `# 添加角色`, `添加角色`).
- **Python**: `web/` files are flat (no packages), imports rely on `cwd = web/`. Avoid breaking this flat layout.
- **web/ has two "app" entry points** that both talk to the same services layer: `app.py` (Gradio) and `api_server.py` (FastAPI). When adding a feature, keep the adapter layering (`app.py` → `gradio_adapter.py` → `services.py`; `api_server.py` → `services.py`) rather than inlining logic into the UI.
- **Do not commit `.env` files** (gitignored) or real API keys.
- Design docs for consistency (人物/场景一致性) and image-gen API details are under `docs/` — read them before touching character/scene consistency or image generation.
