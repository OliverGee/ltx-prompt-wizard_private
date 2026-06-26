# LTX Prompt Studio — Planning Document

> Working title — rename once a final name is chosen. This document is the technical and product plan for an open-source app that generates LTX2 video-generation prompts from an input image, with local-first inference and optional third-party providers.

**Status:** Draft v1
**Last updated:** 2026-06-26

---

## Table of Contents

1. [Goals & Non-Goals](#1-goals--non-goals)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Tech Stack](#3-tech-stack)
4. [Repository Layout](#4-repository-layout)
5. [Data Model](#5-data-model)
6. [Provider Abstraction](#6-provider-abstraction)
7. [Local Inference Strategy](#7-local-inference-strategy)
8. [API Contract](#8-api-contract)
9. [Page-by-Page Design](#9-page-by-page-design)
10. [Security Considerations](#10-security-considerations)
11. [Deployment Modes](#11-deployment-modes)
12. [Build Phases / Milestones](#12-build-phases--milestones)
13. [Open Questions](#13-open-questions)
14. [Risks](#14-risks)
15. [Future Work (Post-v1)](#15-future-work-post-v1)

---

## 1. Goals & Non-Goals

### Goals

- Turn an image into one or more high-quality LTX2-style prompts with minimal friction.
- Work entirely offline/local by default — no third-party account required to use the core pipeline.
- Allow power users to bolt on cloud providers (Ollama, LM Studio, Anthropic, Z.AI, Grok, ChatGPT) per-step, via their own API keys.
- Ship as both a single-user desktop app (Tauri/AppImage) and a self-hosted server (Docker Compose), from one shared codebase.
- Keep a permanent, searchable record of every image and every generated prompt.

### Non-Goals (v1)

- No built-in video generation — this app only produces *prompts*, the output is consumed downstream (e.g. ComfyUI).
- No multi-user/auth system in v1 — single-user/local-trust model only (see [§13](#13-open-questions) for whether Docker mode needs this sooner).
- No Windows/macOS packaging in v1 (Linux AppImage + Docker only; architecture should not preclude it later).
- No mobile app.

---

## 2. High-Level Architecture

The single biggest architectural decision: **one backend, two supervisors.**

```
                     ┌─────────────────────────┐
                     │   Frontend (React SPA)  │
                     │  identical build output │
                     └─────────────┬────────────┘
                                   │ HTTP/SSE/WebSocket
                                   │ (localhost or LAN)
                     ┌─────────────▼────────────┐
                     │   Backend (Node/Fastify) │
                     │  - REST API              │
                     │  - SQLite                │
                     │  - Provider adapters     │
                     │  - File storage          │
                     │  - HF download manager   │
                     └──────┬─────────────┬─────┘
                             │             │
                  ┌──────────▼───┐   ┌─────▼──────────┐
                  │ Tauri sidecar │   │ Docker container│
                  │ (desktop)     │   │ (server)        │
                  └───────────────┘   └─────────────────┘
```

- The **frontend** never knows or cares which mode it's running in. It only talks to the backend's HTTP API.
- The **backend** is a single Node service, compiled to a standalone binary for the Tauri case (e.g. via `bun build --compile`) and run as a normal container for Docker.
- **Tauri** is reduced to a thin Rust shell: it launches the backend binary as a sidecar process, waits for it to report ready, then points its webview at `http://localhost:<port>`. Tauri does not reimplement business logic.
- This keeps ~95% of the code (everything except process supervision and storage-path resolution) shared between deployment modes.

### Why not a Rust backend?

A "pure" Tauri app would implement backend logic as Rust `#[tauri::command]`s. We're explicitly avoiding this:
- It would require a second backend implementation for Docker/server mode (Rust doesn't run "as a container business-logic service" as conveniently as it runs as a sidecar).
- The team's existing tooling fluency is Node/TypeScript/SQLite (per prior projects: Nodex, BrickVault, Lightframe), so this minimizes new-tool risk.
- Tauri's job here is window chrome + native packaging, not application logic.

---

## 3. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Frontend framework | React + Vite + TypeScript | Matches existing project conventions |
| Styling | Tailwind CSS | Dark-theme-first, consistent with prior projects |
| Backend framework | Node.js + Fastify | Lightweight, good plugin ecosystem, fast |
| Database | SQLite (via `better-sqlite3` or `drizzle-orm`) | Single-file, zero-ops, works identically in both deployment modes |
| Schema validation | Zod | Shared between frontend and backend via a `/shared` package |
| Local LLM/VLM runtime | `llama.cpp` (`llama-server`) | OpenAI-compatible HTTP API; see [§7](#7-local-inference-strategy) |
| Process management (Tauri) | Tauri sidecar API | Backend binary bundled as a sidecar executable |
| Containerization | Docker Compose | Backend + optional GPU-enabled inference service |
| Packaging (desktop) | Tauri v2 (AppImage target) | Rust shell only |
| Background jobs | In-process queue (e.g. `p-queue`) or lightweight worker thread pool | For batch generation & HF downloads |
| Realtime updates | Server-Sent Events (SSE) | Simpler than WebSockets for one-directional progress streams (download progress, generation streaming) |

---

## 4. Repository Layout

A single monorepo using pnpm workspaces:

```
ltx-prompt-studio/
├── frontend/              # React/Vite SPA
│   ├── src/
│   │   ├── pages/         # Settings, Models, History, Favourites, Generate
│   │   ├── components/
│   │   ├── api/           # typed API client (generated or hand-written)
│   │   └── store/         # client state (Zustand/Jotai)
│   └── vite.config.ts
│
├── backend/                # Node/Fastify API
│   ├── src/
│   │   ├── routes/
│   │   ├── providers/      # provider adapters (see §6)
│   │   ├── db/             # schema, migrations, queries
│   │   ├── storage/        # image/file storage abstraction
│   │   ├── jobs/           # batch generation, HF downloads
│   │   └── server.ts
│   └── package.json
│
├── tauri/                  # Rust shell, sidecar wiring, tauri.conf.json
│
├── shared/                 # Zod schemas & TS types shared front/back
│
├── presets/                # Built-in preset definitions (JSON), versioned
│
├── docker/
│   ├── docker-compose.yml
│   ├── Dockerfile.backend
│   └── Dockerfile.frontend  (or served by backend directly)
│
├── docs/
│   ├── features.md
│   └── PLANNING.md          (this file)
│
└── pnpm-workspace.yaml
```

---

## 5. Data Model

SQLite schema (illustrative — finalize via migration tool of choice, e.g. Drizzle or raw SQL migrations).

```sql
-- Uploaded source images
CREATE TABLE images (
  id            TEXT PRIMARY KEY,         -- uuid
  stored_path   TEXT NOT NULL,            -- relative path in app storage dir
  thumbnail_path TEXT,
  width         INTEGER,
  height        INTEGER,
  original_name TEXT,
  created_at    TEXT NOT NULL             -- ISO 8601
);

-- Captions generated from images (one image can have many, across models/time)
CREATE TABLE captions (
  id            TEXT PRIMARY KEY,
  image_id      TEXT NOT NULL REFERENCES images(id) ON DELETE CASCADE,
  provider_id   TEXT NOT NULL REFERENCES providers(id),
  model_name    TEXT NOT NULL,
  raw_text      TEXT NOT NULL,
  edited_text   TEXT,                     -- user manual edit, if any
  created_at    TEXT NOT NULL
);

-- Preset library
CREATE TABLE presets (
  id            TEXT PRIMARY KEY,
  name          TEXT NOT NULL,
  category      TEXT,                     -- e.g. 'camera', 'lighting', 'motion', 'style'
  description   TEXT,
  template      TEXT NOT NULL,             -- prompt template / instruction fragment
  tags          TEXT,                     -- JSON array, stored as text
  is_builtin    INTEGER NOT NULL DEFAULT 0,
  created_at    TEXT NOT NULL
);

-- Final generated prompts
CREATE TABLE prompts (
  id            TEXT PRIMARY KEY,
  image_id      TEXT REFERENCES images(id) ON DELETE SET NULL,
  caption_id    TEXT REFERENCES captions(id) ON DELETE SET NULL,
  preset_id     TEXT REFERENCES presets(id) ON DELETE SET NULL,
  provider_id   TEXT NOT NULL REFERENCES providers(id),
  model_name    TEXT NOT NULL,
  final_text    TEXT NOT NULL,
  batch_id      TEXT,                      -- groups prompts generated together
  is_favorite   INTEGER NOT NULL DEFAULT 0,
  created_at    TEXT NOT NULL
);

-- Configured providers (local + third-party)
CREATE TABLE providers (
  id                TEXT PRIMARY KEY,
  type              TEXT NOT NULL,         -- 'llama_cpp' | 'ollama' | 'lm_studio' | 'anthropic' | 'zai' | 'grok' | 'openai'
  display_name      TEXT NOT NULL,
  base_url          TEXT,
  api_key_encrypted TEXT,
  capabilities      TEXT NOT NULL,         -- JSON array: ['caption','enhance']
  default_model     TEXT,
  enabled           INTEGER NOT NULL DEFAULT 1,
  created_at        TEXT NOT NULL
);

-- Local model registry
CREATE TABLE local_models (
  id            TEXT PRIMARY KEY,
  path          TEXT NOT NULL,
  format        TEXT,                      -- 'gguf', etc.
  role          TEXT,                      -- 'caption' | 'enhance' | 'both'
  size_bytes    INTEGER,
  hf_repo_id    TEXT,                       -- if downloaded from HF
  added_at      TEXT NOT NULL
);

-- Generic key/value settings
CREATE TABLE settings (
  key           TEXT PRIMARY KEY,
  value         TEXT
);
```

**Notes:**
- `captions` is separate from `prompts` so the same image can be re-captioned by different VLMs over time without losing history.
- `batch_id` lets the History/Favourites UI group "8 prompts generated together from this image" as one unit.
- `api_key_encrypted` — see [§10](#10-security-considerations).

---

## 6. Provider Abstraction

One interface, implemented by every backend (local or remote):

```ts
type Capability = 'caption' | 'enhance';

interface ProviderAdapter {
  id: string;
  type: string;
  capabilities: Capability[];

  testConnection(): Promise<{ ok: boolean; message?: string }>;
  listModels(): Promise<{ name: string; capabilities: Capability[] }[]>;

  caption(input: { imagePath: string; modelName: string }): Promise<string>;

  enhance(input: {
    text: string;
    modelName: string;
    presetContext?: string;
  }): Promise<string>;
}
```

### Adapter groupings

| Adapter | Backed by | Notes |
|---|---|---|
| `LlamaCppAdapter` | local `llama-server` | Default, no-network. OpenAI-compatible endpoint. |
| `OllamaAdapter` | user's local Ollama instance | OpenAI-compatible (or native Ollama API) |
| `LmStudioAdapter` | user's local LM Studio instance | OpenAI-compatible endpoint |
| `OpenAICompatibleAdapter` | ChatGPT, Z.AI, Grok | One shared adapter parameterized by base URL + key, since all three speak the OpenAI chat-completions format |
| `AnthropicAdapter` | Anthropic API | Distinct message/content-block format, needs its own adapter |

This means adding a new OpenAI-compatible provider in the future is a config change, not new code — important since "Z.AI" and "Grok" style providers tend to multiply over time.

The UI filters model selection dropdowns by `capabilities`, so a text-only model never shows up as a captioning option.

---

## 7. Local Inference Strategy

**Default engine: `llama.cpp`'s `llama-server`.**

Rationale:
- Exposes an **OpenAI-compatible HTTP API** — local inference becomes "just another provider" using the same adapter code path as cloud OpenAI-compatible providers, rather than a special-cased integration.
- GGUF-quantized vision-capable models (LLaVA, Qwen2-VL, MiniCPM-V, Moondream, etc.) cover captioning needs on consumer GPUs without a Python/CUDA-toolkit dependency chain.
- Single static binary, easy to bundle as a Tauri sidecar and as a slim layer in the Docker image.

### Binary distribution approach

- Do **not** bundle a CUDA build of `llama-server` inside the AppImage — this bloats the package and forces a CPU/CUDA/Vulkan build matrix baked into the installer.
- On first run, detect the GPU (vendor + driver) and download the matching prebuilt `llama-server` release from llama.cpp's GitHub releases, storing it in the app-data directory — the same pattern Ollama uses.
- Docker mode: ship a CUDA-enabled image variant (and a CPU-only fallback variant) rather than runtime detection, since the container's GPU access is already explicit via `docker-compose.yml` device reservations.

### Process lifecycle

- Backend manages the `llama-server` child process: start on first model load, health-check via its `/health` endpoint, restart on crash, shut down cleanly on app exit.
- Model swapping (different GGUF for caption vs. enhance) means either running multiple `llama-server` instances on different ports, or unloading/reloading — multiple instances is simpler operationally and avoids reload latency when switching between caption/enhance steps.

---

## 8. API Contract

Representative REST endpoints (Fastify). Final shape should be captured as an OpenAPI spec once stabilized.

```
POST   /api/images                     — upload image, returns image record
GET    /api/images/:id                 — fetch image metadata
GET    /api/images                     — list (paginated, filterable)

POST   /api/images/:id/caption         — { providerId, modelName } -> caption record
PATCH  /api/captions/:id               — edit caption text

POST   /api/prompts/generate           — { captionId, presetId, providerId, modelName, count }
                                          -> returns batch_id + prompt[] (SSE stream of progress for count > 1)
GET    /api/prompts                    — list/filter history
PATCH  /api/prompts/:id/favorite       — toggle favourite
GET    /api/prompts/favorites          — list favourites only

GET    /api/presets                    — list
POST   /api/presets                    — create custom preset
PUT    /api/presets/:id
DELETE /api/presets/:id
POST   /api/presets/import             — import JSON
GET    /api/presets/:id/export

GET    /api/providers                  — list configured providers (keys redacted)
POST   /api/providers                  — add provider config
PUT    /api/providers/:id
DELETE /api/providers/:id
POST   /api/providers/:id/test         — test connection

GET    /api/models/local               — scan configured folders, list local models
POST   /api/models/folders             — add a watched folder
GET    /api/models/huggingface/search  — proxy HF search
POST   /api/models/huggingface/download — start download (SSE progress)
GET    /api/models/huggingface/downloads/:id  — progress/status

GET    /api/settings
PUT    /api/settings
```

Batch generation (`count > N`) and HF downloads are the two operations long enough to warrant SSE progress streams rather than a single blocking response.

---

## 9. Page-by-Page Design

### 9.1 Generate (primary workflow page)

1. Upload/select image → thumbnail preview.
2. Caption step: provider + model dropdown (filtered to `caption` capability) → "Caption" button → editable text area with result.
3. Enhance step: provider + model dropdown (filtered to `enhance` capability) → optional preset selection → "Enhance" button → editable result.
4. Preset application: choose one or more presets/axes (see open question on flat vs. composable below).
5. Generate: "Generate 1" or "Generate batch of N" → results list, each with copy button and "+ Favourite".

### 9.2 Settings

- Provider list (cards): type, display name, base URL (if applicable), masked API key, capability badges, enabled toggle, "Test connection" button.
- Add Provider modal: select type → relevant fields appear (URL for self-hosted, key field for cloud).
- General settings: storage location, theme, default provider per pipeline step.

### 9.3 Models

- Tab 1 — **Local**: list of watched folders (add/remove), scanned models with size/format/role tags.
- Tab 2 — **Hugging Face**: search box, result cards (name, size, downloads, license), download button → progress bar (SSE), HF token field (reuses the same encrypted-storage mechanism as provider keys).

### 9.4 History

- Paginated/searchable table or card grid of all generated prompts.
- Filters: date range, source image, provider/model, preset.
- Click-through to the originating image + caption + preset chain.
- Bulk export (JSON/CSV) of filtered results.

### 9.5 Favourites

- Same visual treatment as History, but pre-filtered to `is_favorite = 1`.
- Unfavourite action.
- Export.

---

## 10. Security Considerations

- **API key encryption at rest**: generate a local app encryption key on first run, stored outside the database:
  - Tauri: OS-appropriate secure location (or app-data dir with restrictive file permissions if a full keyring integration is deferred).
  - Docker: provided via environment variable or a mounted secrets file, **not** baked into the image.
  - All `api_key` columns are encrypted with this key before being written to SQLite; never returned unredacted via the API.
- **Docker mode network exposure**: since this can be deployed as a server, the default Compose file should bind to `127.0.0.1` rather than `0.0.0.0` unless the user explicitly opts into LAN exposure — avoids accidentally exposing an unauthenticated API with stored third-party keys to the network. See [§13](#13-open-questions) re: whether auth is needed for v1 server mode.
- **Hugging Face token** scope: document that a read-only HF token is sufficient; never request write scope.
- **Image storage**: stored images may contain sensitive content (NSFW, copyrighted source material, personal photos) — storage directory should not be web-served directly without access control, even in Docker mode.

---

## 11. Deployment Modes

### Tauri / AppImage (desktop)

- Single AppImage containing: Tauri shell + compiled Node backend binary + frontend static assets.
- First-run flow: detect GPU → offer to download appropriate `llama-server` build → set default storage paths under `~/.local/share/ltx-prompt-studio/`.
- Backend binds to a locally-chosen free port; not exposed beyond localhost.

### Docker Compose (server)

```yaml
services:
  backend:
    image: ltx-prompt-studio/backend:latest
    ports:
      - "127.0.0.1:8420:8420"
    volumes:
      - ./data:/app/data            # SQLite + settings
      - ./images:/app/storage/images
      - ./models:/app/storage/models
    environment:
      - APP_ENCRYPTION_KEY_FILE=/run/secrets/app_key
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
  # llama-server can run as a sidecar service or be spawned by backend directly
```

- A CPU-only Compose variant should be provided for users without an NVIDIA GPU (local inference will simply be slower; cloud providers remain unaffected).
- Frontend can be served by the backend itself (static file serving) to keep the Compose file minimal, with a separate Nginx-fronted variant offered for users who want TLS termination/reverse proxy in front.

---

## 12. Build Phases / Milestones

| Phase | Scope |
|---|---|
| **1. Core pipeline** | Backend skeleton, SQLite schema, image upload, `LlamaCppAdapter` + one cloud adapter (e.g. Anthropic), bare-bones frontend: upload → caption → enhance → single prompt out. No presets, no history UI yet (just DB writes). |
| **2. Docker packaging** | Dockerfile(s), Compose file, GPU passthrough, CPU-only fallback. Validate the full pipeline runs server-side. |
| **3. Settings & Providers** | Full Settings page, encrypted key storage, remaining provider adapters (Ollama, LM Studio, OpenAI-compatible group), connection testing. |
| **4. Models page** | Local folder scanning, HF search/download manager with progress streaming, HF token handling. |
| **5. Presets & batch generation** | Decide flat vs. composable preset model (see open questions), build preset CRUD + import/export, wire batch generation with `batch_id` grouping. |
| **6. History & Favourites** | Full filter/search UI, favouriting, export. |
| **7. Tauri packaging** | Compile backend to standalone binary, wire as sidecar, build AppImage, GPU auto-detection + `llama-server` binary fetch flow. |
| **8 (post-v1).** | Additional platforms, additional local-inference backends (e.g. diffusers/Python sidecar), preset marketplace/sharing, multi-user/auth for server mode. |

---

## 13. Open Questions

These should be resolved before (or early in) the corresponding build phase:

1. **Flat presets vs. composable axes** — a single template per preset (simple, but combinations require duplicate presets) vs. independent axes (camera movement / lighting / motion intensity / visual style) that combine into the final instruction (more natural fit for LTX2's descriptive prompt style, but more upfront schema/UI work). Affects the `presets` table shape directly — needs deciding before Phase 5.
2. **Server-mode auth** — v1 assumes single-user/local-trust. If Docker mode is expected to run on a shared network or VPS (matching prior self-hosting patterns), even a minimal shared-secret or basic-auth layer may be worth pulling into Phase 2 rather than deferring indefinitely, given that stored API keys make this a higher-value target than a typical hobby app.
3. **Local inference scope beyond llama.cpp** — is GGUF-based captioning/enhancement sufficient long-term, or is a Python/diffusers-class sidecar (for non-GGUF model families) wanted eventually? Fine to defer, but worth flagging since it would introduce a second local-inference runtime type.
4. **License** — MIT (maximally permissive, consistent with most personal OSS projects) vs. AGPL (if there's concern about a hosted-SaaS fork). No technical impact, but should be settled before the first public push.
5. **Preset sharing format** — if export/import is JSON now, is a future "preset pack" marketplace/registry in scope, and does that change the export schema needed today (e.g. versioning, author metadata)?

---

## 14. Risks

- **GPU/driver fragmentation** for the auto-detect-and-download `llama-server` flow — Linux GPU/driver combinations are diverse; the detection logic needs solid fallback (manual binary selection) rather than assuming success.
- **Vision-model quality ceiling** — GGUF-quantized VLMs are good but generally behind their full-precision/cloud counterparts for nuanced image description; users chasing the best captions will lean on cloud providers, which is fine, but the local-only experience should still be "good enough" to be the credible default.
- **HF download reliability** — multi-GB files over flaky connections need real resumability, not just a progress bar; underestimate this and it becomes the most support-heavy feature in the app.
- **Scope creep via presets** — "a wide range of presets" can expand indefinitely; worth capping the built-in set for v1 and leaning on import/export + community contribution rather than trying to ship an exhaustive library at launch.

---

## 15. Future Work (Post-v1)

- Windows/macOS Tauri builds.
- Multi-user accounts + auth for server mode.
- Preset marketplace/community sharing.
- Additional local-inference backend (Python/diffusers sidecar) for model families outside GGUF.
- Direct ComfyUI integration (push prompt + image straight into a running ComfyUI instance via its API) rather than copy/paste export.
- Prompt versioning/diffing (track edits to a favourited prompt over time).
