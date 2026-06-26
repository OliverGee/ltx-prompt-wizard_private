# Features

> Working title: **LTX Prompt Studio** — rename freely, used throughout for readability.

A self-hostable / desktop app that turns an input image into one or more polished LTX2 video-generation prompts, with local-first inference and optional third-party LLM/VLM providers.

---

## 1. Image → Caption

- [ ] Upload/drag-drop a single image
- [ ] Batch upload (multiple images queued for processing)
- [ ] Captioning via selectable VLM (local or remote)
- [ ] Manual edit of generated caption before continuing
- [ ] Re-caption an existing image with a different model

## 2. Prompt Enhancement

- [ ] Enhance/rewrite caption using a selectable model
- [ ] Choice of **local model** (runs on user's GPU) or **third-party API**
- [ ] Side-by-side view of raw caption vs. enhanced version
- [ ] Re-run enhancement with a different model without re-captioning

## 3. Presets

- [ ] Library of built-in presets tailored to LTX2 prompt structure
- [ ] User-created custom presets
- [ ] Preset categories/tags (e.g. camera, motion, lighting, style)
- [ ] Import/export presets as shareable JSON files
- [ ] Preview of how a preset modifies the final prompt

## 4. Prompt Generation

- [ ] Generate a single final prompt
- [ ] Generate a batch (N variations) from one image/caption
- [ ] Copy-to-clipboard and "send to ComfyUI"–style export
- [ ] Export batch as plain text / JSON / CSV

## 5. Providers & Models

- [ ] Local inference on user's own GPU (no internet required by default)
- [ ] Pluggable third-party providers: Ollama, LM Studio, Anthropic, Z.AI, Grok, ChatGPT
- [ ] Per-provider capability flags (captioning vs. text enhancement vs. both)
- [ ] Connection test button per configured provider

## 6. Settings Page

- [ ] Add/edit/remove API keys per provider
- [ ] Encrypted storage of API keys at rest
- [ ] Set default provider/model per pipeline step (caption / enhance)
- [ ] General app settings (storage paths, theme, etc.)

## 7. Models Page

- [ ] List local models found in user-configured filesystem folders
- [ ] Add/remove watched model folders
- [ ] Search and download models from Hugging Face
- [ ] Hugging Face API key/token management
- [ ] Download progress + resumable downloads
- [ ] Tag models by role (captioning / text enhancement / both)

## 8. History

- [ ] Persistent storage of every uploaded image
- [ ] Persistent storage of every generated prompt (with the image/caption/preset/provider that produced it)
- [ ] Filter/search history (by date, image, provider, preset)
- [ ] Re-open a history entry to re-run or tweak

## 9. Favourites

- [ ] Mark any generated prompt as a favourite
- [ ] Dedicated favourites page, independent of full history
- [ ] Remove from favourites
- [ ] Export favourites

## 10. Platform & Deployment

- [ ] Linux Tauri AppImage build
- [ ] Docker Compose stack
- [ ] Shared core app logic between both deployment modes
- [ ] Future: Windows/macOS Tauri builds (not in scope for v1)

## 11. Cross-cutting

- [ ] No required third-party dependency for core functionality
- [ ] Dark-themed UI by default
- [ ] Local SQLite-backed persistence
- [ ] Open source license (TBD — e.g. MIT/AGPL)
