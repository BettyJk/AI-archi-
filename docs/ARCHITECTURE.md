# Architecture

This document describes how the four components fit together, the request/data flow, and the
key design details engineers need to know.

## 1. Component topology

```
                         Browser (http://localhost:5173)
                                    │
      ┌─────────────────────────────┴──────────────────────────────┐
      │                    Frontend (Vite dev server)               │
      │           space-weaver-main/src  (React + Three.js)         │
      └───────┬──────────────────────────────────────────┬─────────┘
              │ api.ts → API_BASE                          │ api.ts → MG2_API_BASE
              │ http://127.0.0.1:8000                      │ http://127.0.0.1:8001
              ▼                                            ▼
   ┌──────────────────────────┐               ┌──────────────────────────────┐
   │ space-weaver backend      │  /ai-analysis │ MG2 NOK Correction System     │
   │ backend.py  :8000         │──────────────▶│ mg2_api_server.py  :8001      │
   │                           │  (proxy to     │                              │
   │  trimesh / numpy          │   analyze-batch)│  pipeline/classifier.py      │
   │  in-memory MOCK_ZONES     │               │  pipeline/ai_analyst.py       │
   └──────────┬───────────────┘               │  pipeline/report.py           │
              │ reads files                    │  pipeline/image_generator.py  │
              ▼                                └───────────┬──────────────────┘
   ┌──────────────────────────┐                           │ LLM / image APIs
   │ peugeot_preprocessor2/    │                           ▼
   │  data/*.json  (metadata)  │           ┌───────────────────────────────────┐
   │  stl_files/   (meshes)    │           │ Capgemini Generative Engine (LLM)   │
   └──────────────────────────┘           │ Anthropic Claude (LLM)              │
                                           │ Capgemini Assets API (image gen)    │
                                           └───────────────────────────────────┘
```

## 2. Ports

| Service | Default port | Bind host | Notes |
|---------|-------------|-----------|-------|
| Frontend (Vite) | **8080** (config) → **use 5173** | `::` (IPv6, dual-stack) | 8080 collides with EnterpriseDB `httpd` on this machine; run on 5173 |
| Main backend `backend.py` | **8000** | `0.0.0.0` (`BACKEND_HOST`) | Env: `BACKEND_PORT`, `DEBUG` |
| MG2 server `mg2_api_server.py` | **8001** | `0.0.0.0` (`MG2_HOST`) | Env: `MG2_PORT`, `DEBUG`, `AI_API` |
| (legacy) `peugeot_preprocessor2/main.py` | 8000 (uvicorn default) | - | Not used by the integrated app |

## 3. Data flow: computing a clearance

1. On startup, `backend.py` runs `auto_load_predefined_zones()` - it scans
   `peugeot_preprocessor2/data/*.json`, maps each JSON file to its STL subfolder, injects
   `zone` / `folder_label` / `folder_key` into every part, and stores them in the in-memory
   global `MOCK_ZONES`. (Despite the name, this is **real data**, not mocks.)
2. Frontend calls `GET /zones` and `GET /zones/{zone}/parts` to populate the UI.
3. Neighbour detection runs **client-side** (`src/lib/neighbours.ts`) using each part's stored
   bounding box and the user's expansion params (Per % / MinDis / MaxDis).
4. Frontend calls `POST /clearance` with the study part + selected neighbours. The backend:
   - resolves each STL via a filename cache (`{stem_lower: path}`),
   - loads meshes through `load_mesh_cached` (keyed by `(path, mtime)`, FIFO eviction > 200),
   - does an **AABB overlap** pre-check per axis,
   - samples up to **1000 vertices** per mesh (seeded RNG) and computes the full pairwise
     distance matrix → minimum distance = the gap, with the closest vertex pair as `point_a`/`point_b`,
   - if boxes overlap, runs `trimesh.proximity.signed_distance` on the closest vertices to detect
     **interference / penetration depth** and the real contact points.
   - Returns results as **NDJSON** (`application/x-ndjson`), one `{"type":"progress", ...}` row
     per neighbour. (Note: rows are all computed first, then streamed - the stream reports
     progress but does not compute incrementally.)
5. `POST /clearance/refine-penetration` re-runs the signed-distance step at higher fidelity
   (20 closest vertices per side) for a single interfering pair.

See [API_REFERENCE.md](API_REFERENCE.md) for exact request/response shapes and
[DATA_AND_PREPROCESSING.md](DATA_AND_PREPROCESSING.md) for the clearance algorithm and rules.

## 4. Data flow: AI analysis of a clearance

There are **two** AI paths, both ending at the MG2 server:

**A. Single-image (the "Analyze" button)** - frontend → MG2 directly:
1. `Viewer3D.captureSegmentAI(neighbour, 2400, 1600)` renders a 4-panel composite image
   (two perpendicular gap views, an along-axis cross-section, and a context view) with a
   green CAD-style dimension line and header/footer bars.
2. Frontend `analyzeClearanceImage()` → `POST http://127.0.0.1:8001/analyze` with the base64 PNG.
3. MG2 decodes the image to a structured temp filename → `classify_images` (parses the gap and
   part names from the filename, applies rules, assigns a verdict) → `analyse_nok_images`
   (LLM call for NOK/WARNING/CLASH) → `generate_html_report`.
4. Response includes `verdict`, an `analysis` object (visual observations, failure mode, 3
   corrective scenarios), and a `report_url`.

**B. Batch (the "Analyze Selected" button)** - can go through either the backend proxy or MG2:
- Frontend `analyzeClearanceBatch()` → `POST :8001/analyze-batch`, **or**
- Backend `POST :8000/ai-analysis` proxies to `POST :8001/analyze-batch`
  (`{images, api_type:"capgemini", include_ok:false}`, 180 s timeout).

> **`api_type` is ignored by the MG2 server.** Both `/analyze` and `/analyze-batch` hard-code
> `api_type="capgemini"` when calling `analyse_nok_images`, so server-side analysis always uses
> the Capgemini Generative Engine regardless of what the client sends. Only the MG2 **CLI**
> (`main.py --api`) honors the provider choice.

## 5. AI providers (there are several)

| Where | SDK | Provider | Model | Env var |
|-------|-----|----------|-------|---------|
| MG2 `ai_analyst.py` (Capgemini path - used by the server) | `openai` | Capgemini Generative Engine | `us.anthropic.claude-sonnet-4-5-20250929-v1:0` | `GENERATIVE_ENGINE_API_KEY`, `GENERATIVE_ENGINE_BASE_URL` |
| MG2 `ai_analyst.py` (Anthropic path - CLI only) | `anthropic` | Anthropic | `claude-opus-4-5` | `ANTHROPIC_API_KEY` |
| MG2 `image_generator.py` (correction visuals) | `requests` | Capgemini **Assets** API | `gemini-2.5-flash-image` | `CAPGEMINI_ASSETS_API_KEY` |
| Preprocessor `Ai analyst.py` | `openai` | Capgemini Generative Engine | `openai.gpt-4o` | `LLM_OPENAI_KEY` |
| Preprocessor `main.py` `call_claude_ai` | `anthropic` | Anthropic | `claude-sonnet-4-20250514` | `ANTHROPIC_API_KEY` |

The Generative Engine (chat/vision, OpenAI-compatible) and the Assets API (image generation) are
two **different** Capgemini services with different keys and base URLs.

## 6. The two backends

The repository contains two independent FastAPI backends. Only the first is part of the running app.

### `space-weaver-main/backend.py` - the one you run (port 8000)
- The integrated app's data + clearance backend, called by the frontend's `api.ts`.
- Loads the 7 predefined zones from `peugeot_preprocessor2/data` + `stl_files` on startup.
- Endpoints: `/zones`, `/zones/{zone}/parts`, `/clearance` (NDJSON), `/clearance/refine-penetration`,
  `/stl-proxy`, `/load-zone`, `/preprocess/single`, `/preprocess/folder`, `/fs/list`,
  `/browse/*` (native tkinter pickers), `/ai-analysis` (proxy to MG2), `/health`.
- In-memory only: zones added via `/load-zone` are lost on restart; the 7 predefined ones reload.

### `peugeot_preprocessor2/main.py` - legacy standalone ("Week 3")
- A **separate** FastAPI backend with its own embedded clearance engine and AI path
  (`call_claude_ai` → Anthropic `claude-sonnet-4-20250514`), analysis history persisted to
  `history/`, endpoints like `/analyse/by-name`, `/analyse/upload`, `/parts/search`, `/history`.
- Not wired to the frontend; kept for reference. Its sibling CLIs are the useful parts:
  - `engine.py` - standalone clearance computation for one STL.
  - `Ai analyst.py` - standalone LLM analysis of an engine output file.
  - `stl_files/peugeot_preprocessor/preprocess.py` - the actual **CAD→JSON** tool that generated
    `data/*.json`.

## 7. Key implementation notes / gotchas

- **In-memory zone store** (`MOCK_ZONES`) - no database; restart reloads only the 7 predefined zones.
- **Hardcoded fallback paths** - `backend.py` falls back to
  `C:\Users\jbouthay\Downloads\pfe implantation\peugeot_preprocessor\...` if the relative
  `peugeot_preprocessor2` folders are missing. Machine-specific; flag when relocating.
- **Rules live in code, not `packaging_rules.json`** - the classifier's `CLEARANCE_RULES` and the
  LLM system prompt in `ai_analyst.py` are the operative thresholds. `packaging_rules.json` in the
  preprocessor is an authoritative **reference spec** but is not imported by any code. See
  [CLEARANCE_RULES.md](CLEARANCE_RULES.md).
- **Picker endpoints return HTTP 204 as an error** on cancel (`/browse/*`) - unusual but intentional.
- **Report artifacts** are written to `MG2_NOK_Correction_System_fixed/output/` and served at
  `:8001/reports/<file>`.
</content>
