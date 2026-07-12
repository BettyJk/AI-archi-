# Project Report & Technology Rationale

A detailed explanation of the **Packaging Analysis Platform** — what it does, how it is built, and
**why each technology was chosen** over the alternatives. This document is the "reasoning behind
the architecture" companion to the reference docs ([Architecture](ARCHITECTURE.md),
[API Reference](API_REFERENCE.md), etc.).

---

## 1. Executive summary

The platform digitises and automates the **automotive packaging clearance study** ("Synthèse
Implantation") for the Peugeot 3008. In classic OEM engineering this is done by hand in CATIA: an
engineer visually measures the 3D gap between a part and its neighbours, cross-references a paper
"abacus" of clearance rules (fuel lines need X mm, harnesses need Y mm, moving parts need more…),
decides whether each gap is acceptable, and writes up findings in Excel/PowerPoint for weekly
review meetings.

Our platform turns that into an interactive web tool that:

1. Loads a **zone** of the vehicle (a set of parts) from preprocessed CAD data.
2. Automatically finds a part's spatial **neighbours**.
3. Computes **exact mesh-to-mesh clearance** (minimum distance, interference, penetration depth).
4. Renders everything in a **browser 3D viewer**.
5. Uses a **vision LLM** to classify each clearance, explain the risk, and propose concrete
   **corrective actions** (which part to move, along which axis, by how many mm).
6. Produces shareable **HTML engineering reports**.

The guiding design principle is **separation by rate-of-change and by workload**: heavy one-off CAD
processing, interactive geometry maths, AI reasoning, and the UI each have very different runtime
profiles, so each lives in its own component with a narrow contract between them.

---

## 2. Why a multi-service architecture?

```
Frontend (browser)  ──►  Main backend (:8000)  ──►  MG2 AI system (:8001)
   React + Three.js         FastAPI + trimesh          FastAPI + vision LLM
        ▲                        │
        └── preprocessed data ◄──┘  peugeot_preprocessor2 (offline CAD → JSON)
```

We deliberately split the system into four parts rather than one monolith:

| Concern | Runtime profile | Why it must be isolated |
|---------|-----------------|-------------------------|
| **CAD preprocessing** | Minutes, offline, CPU-heavy | Runs rarely; must not block the interactive app. Output is static JSON. |
| **Clearance geometry** | 100 ms–seconds per request | Needs Python's mesh libraries and CPU; benefits from an in-process mesh cache. |
| **AI analysis** | 2–30 s per image, network-bound, costs money | Optional, slow, external-API dependent — isolating it keeps the core app responsive and lets it fail independently. |
| **UI + 3D rendering** | Real-time, per-frame | Must run in the browser on the user's GPU; a totally different language/runtime. |

Benefits of the split: each piece can be developed, deployed, scaled and **fail independently**;
the expensive/optional AI service never slows down basic clearance work; and the browser does the
GPU rendering while Python does the geometry — each on the runtime that is best at it.

The trade-off is operational: three processes to start and CORS to configure. For a single-user
engineering tool that is an acceptable cost (see [Setup & Run](SETUP_AND_RUN.md)).

---

## 3. Component 1 — Frontend (space-weaver)

**Purpose:** the entire user experience — zone/part selection, neighbour detection, the 3D viewer,
the clearance table, AI-analysis triggering, and report generation.

### Stack

| Technology | Version | Role |
|------------|---------|------|
| **React** | 19 | UI component model |
| **TanStack Start** | 1.167 | Full-stack React framework (SSR-capable, Vite-native) |
| **TanStack Router** | 1.168 | Type-safe file-based routing |
| **Vite** | 7 | Dev server + build tool |
| **TypeScript** | 5.8 | Static typing across the app and API layer |
| **Three.js** | 0.184 | WebGL 3D rendering of STL meshes |
| **Tailwind CSS** | 4 | Utility-first styling + CSS-variable theming |
| **Radix UI / shadcn** | — | Accessible headless UI primitives |
| **Zod + react-hook-form** | 3.24 / 7.71 | Schema validation and forms |
| **JSZip** | 3.10 | Client-side bundling of captured images |
| **lucide-react, sonner, recharts** | — | Icons, toasts, charts |
| **Cloudflare Workers** (wrangler) | — | Deployment target (edge) |

### Why these choices

- **React 19** — the dominant component library; the largest ecosystem of 3D, form, and UI
  packages; and the concurrent-rendering model keeps a data-dense UI (large part lists, live
  clearance updates) responsive. Choosing the mainstream option maximises hiring and library
  availability. *Alternatives:* Vue/Svelte are excellent but have thinner 3D/enterprise-component
  ecosystems; Angular is heavier than this single-page tool needs.
- **TanStack Start + Router** — gives **type-safe routing** (routes, params and loaders are checked
  by TypeScript) and file-based organisation, with SSR available if we later need it. It is
  Vite-native, so it composes with the rest of the toolchain instead of fighting it.
  *Alternative:* Next.js is more popular but is opinionated toward its own server/runtime; TanStack
  is lighter and keeps us on plain Vite + Cloudflare.
- **Vite** — near-instant cold start and hot-module reload make iterating on a 3D UI practical.
  Its dev server is what we run locally (see the [port note](TROUBLESHOOTING.md#the-blank-page--port-8080-conflict)).
  *Alternative:* Webpack/CRA are markedly slower for large TS+asset projects.
- **Three.js (used raw, not via react-three-fiber)** — the only mature, general-purpose WebGL
  engine capable of loading arbitrary STL meshes and doing custom camera/render work in-browser.
  We use it **imperatively** (not through the react-three-fiber abstraction) because the killer
  feature — `captureSegmentAI`, which renders a **4-panel 2400×1600 composite** with CAD-style
  dimension lines for the LLM — needs precise, frame-exact control over multiple offscreen
  renderers, camera framing, and 2D canvas overlays. A declarative wrapper would fight that.
  *Alternative:* Babylon.js is comparable; Three.js has the larger community and lighter footprint.
- **Tailwind v4 + Radix/shadcn** — Tailwind gives fast, consistent styling with a CSS-variable
  theme (light/dark); Radix supplies **accessible** primitives (focus management, ARIA) so we don't
  reinvent dropdowns/dialogs. Together they let a small team ship a polished, accessible UI quickly.
- **TypeScript + Zod** — the frontend talks to two APIs; static types plus runtime schema
  validation catch contract mismatches (e.g. a clearance row missing `point_a`) before they become
  runtime bugs in the 3D viewer.
- **Cloudflare Workers** — an inexpensive, globally-distributed static/SSR host that pairs naturally
  with TanStack Start's server-entry, giving low-latency delivery without managing servers.

---

## 4. Component 2 — Main backend (`backend.py`)

**Purpose:** the data + geometry engine. Serves zones/parts, computes real clearance, streams STL
meshes to the viewer, and proxies batch AI requests to the MG2 service.

### Stack

| Technology | Role | Why |
|------------|------|-----|
| **Python 3.10** | Language | The geometry and AI ecosystems are Python-first; nothing else has `trimesh`. |
| **FastAPI** | Web framework | Async, automatic request validation via **Pydantic**, built-in OpenAPI docs, minimal boilerplate. |
| **Uvicorn** | ASGI server | High-performance async server; supports the streaming responses we need. |
| **trimesh** | Mesh maths | Loads STL, and — critically — provides `proximity.signed_distance` and `closest_point` for interference/penetration. |
| **NumPy** | Numerics | Vectorised pairwise vertex-distance matrix; orders of magnitude faster than Python loops. |
| **Pydantic** | Models | Declarative request/response schemas, validated automatically by FastAPI. |

### Why these choices

- **Python** was effectively mandatory: the entire problem hinges on **`trimesh`**, whose
  `signed_distance`/`closest_point` proximity queries are what let us detect not just "do these
  parts touch" but "by how many mm do they interpenetrate, and where". No JavaScript/Go/Rust
  library offers this maturity of mesh proximity out of the box. Python also hosts the AI SDKs, so
  keeping the backend in Python avoids a second runtime.
- **FastAPI over Flask/Django** — we needed three things Flask doesn't give for free and Django is
  too heavy for: (1) **async** request handling, (2) **Pydantic** request/response validation with
  zero boilerplate, and (3) **streaming responses**. FastAPI delivers all three and generates
  interactive API docs automatically.
- **NDJSON streaming for `/clearance`** — a zone can have hundreds of neighbours; computing them all
  before replying would make the UI feel frozen. We stream one JSON line per neighbour
  (`application/x-ndjson`) so the frontend can render a **live progress bar**. (Implementation note:
  the rows are currently computed then streamed, so it reports progress rather than truly computing
  incrementally — a documented, deliberate simplification.)
- **Vertex-sampling clearance (≤1000 points) + seeded RNG** — exact mesh-to-mesh minimum distance on
  full meshes is expensive. We sample a bounded number of vertices and compute a NumPy distance
  matrix, then refine only genuine interferences with `signed_distance`. This is the classic
  **accuracy/latency trade-off**: sampling makes an interactive tool possible, and the separate
  `/clearance/refine-penetration` endpoint offers a higher-fidelity pass on demand. A fixed seed
  keeps results reproducible.
- **In-memory zone store + `(path, mtime)` mesh cache** — the zone data is **read-only reference
  data** loaded from JSON at startup, so a database would add operational weight for no benefit.
  The mesh cache (FIFO, capped) avoids re-loading the same STL across repeated clearance calls,
  which is the real bottleneck. *Trade-off:* zones added at runtime aren't persisted — acceptable
  for a reference tool, documented in [Architecture](ARCHITECTURE.md).

---

## 5. Component 3 — MG2 NOK Correction System

**Purpose:** the AI brain. Takes a rendered clearance image, classifies the verdict, explains the
engineering risk, and proposes corrective scenarios; generates HTML reports.

### Stack

| Technology | Role | Why |
|------------|------|-----|
| **FastAPI + Uvicorn** | API server | Same reasons as the main backend; consistency across the two Python services. |
| **Pillow (PIL)** | Image processing | Decodes base64 PNG, converts PNG→JPEG for the vision API, handles report thumbnails. |
| **Capgemini Generative Engine** (via OpenAI SDK) | LLM (analysis) | Enterprise gateway to frontier **Claude** models; used for the vision analysis. |
| **Anthropic SDK** | LLM (alt/CLI) | Direct Claude access for the standalone CLI path. |
| **Capgemini Assets API** | Image generation | Generates before/after correction visuals (`gemini-2.5-flash-image`). |

### The AI design and why it works this way

- **Why an LLM at all, and why a *vision* LLM?** The hard part of a packaging study is judgement:
  "this fuel line is 12 mm from a moving suspension arm — is that acceptable, and if not, what's the
  cheapest fix?" That is reasoning over rules plus spatial context. A vision LLM can read the
  rendered geometry *and* apply the rule set *and* explain the fix in engineer-readable language — a
  combination no deterministic checker provides. The deterministic classifier still assigns the
  hard verdict; the LLM adds the explanation and corrective scenarios.
- **Why the 4-panel composite image (the `captureSegmentAI` output)?** A single 3D screenshot is
  ambiguous — depth and the true gap direction are unclear. We feed the model a **2×2 view**
  (perpendicular-to-gap, lateral, along-axis cross-section, and context) with an explicit
  **green dimension line** annotating the measured gap. This gives the model unambiguous spatial
  evidence, which measurably improves verdict quality. It is the same set of views a human engineer
  would rotate to in CATIA.
- **Model choice — Claude Sonnet 4.5 (vision) via Capgemini** — Sonnet 4.5 combines strong visual
  understanding with the reasoning needed to apply layered clearance rules, at a latency/cost that
  suits interactive use. We reach it through the **Capgemini Generative Engine** because the project
  runs in a Capgemini enterprise context: the Generative Engine is an **OpenAI-compatible gateway**,
  so we use the widely-supported `openai` SDK pointed at Capgemini's base URL — giving governed,
  billed, corporate-approved access to frontier models without a direct vendor contract. The direct
  **Anthropic** path (Claude Opus 4.5) remains for the CLI/offline use.
- **Why OpenAI SDK against a Claude model?** Because Capgemini's gateway speaks the OpenAI wire
  format. Using the `openai` client with a custom `base_url` is the least-code way to consume it and
  keeps us portable to any OpenAI-compatible endpoint.
- **Separate Assets API for image generation** — producing a *rendered* before/after correction
  illustration is a different modality (text→image, `gemini-2.5-flash-image`) with its own key and
  endpoint; keeping it separate from the chat/vision path keeps each integration simple, and results
  are cached on disk to avoid paying for regeneration.
- **Rules encoded in a system prompt + a deterministic classifier** — the SCR structural rules and
  CCP circuit rules (see [Clearance Rules](CLEARANCE_RULES.md)) live both in code (for a fast,
  reproducible verdict) and in the LLM prompt (so the model's reasoning stays anchored to the real
  Stellantis thresholds). This dual encoding trades some duplication for reliability: the number
  never depends solely on the model.

---

## 6. Component 4 — Preprocessor & data

**Purpose:** convert raw CAD/STL exports into the compact JSON "part index" the app consumes, and
provide the standalone geometry/AI CLIs.

### Stack & rationale

| Technology | Role | Why |
|------------|------|-----|
| **Python + trimesh** | CAD → metadata | Same mesh library as the backend, so geometry (bbox, centroid, volume, surface area) is computed identically on both sides. |
| **JSON** | Intermediate data format | Human-readable, diff-able, language-neutral, trivially served by the backend. |

- **Why a separate offline preprocessing step at all?** Loading and analysing hundreds of STL
  meshes is slow and memory-heavy. Doing it **once, offline**, and caching the results as JSON means
  the interactive app never pays that cost: at startup the backend just reads small JSON files.
  This cleanly **decouples heavy CAD work from the real-time app** — the single most important data
  decision in the project.
- **Why JSON as the contract?** It is the lowest-friction interchange format: readable in a text
  editor, versionable in git, and served directly over HTTP with no serialization layer. The
  per-part schema (bounding box, centroid, volume, `part_type`, `circuit_type`) is exactly what
  neighbour detection and rule selection need — nothing more.
- **The adaptive neighbour-expansion algorithm** — `expansion = clamp(dim × Per%, MinDis, MaxDis)`
  scales the search box to each part's size (a tiny clip and a long beam need very different search
  radii) while bounding it so huge parts don't select the whole car. This heuristic is why neighbour
  detection is both relevant and fast. See [Data & Preprocessing](DATA_AND_PREPROCESSING.md).

> **Note:** there are two Python backends — the integrated `space-weaver` `backend.py` and a
> legacy standalone `main.py` in the preprocessor folder. Only the former runs in the app; the
> latter is kept for its CLIs. This is explained in [Architecture](ARCHITECTURE.md#6-the-two-backends).

---

## 7. Documentation & hosting stack

| Technology | Role | Why |
|------------|------|-----|
| **MkDocs** | Static docs generator | Plain-Markdown source, trivial config, huge plugin ecosystem — far lower friction than Sphinx/reStructuredText for a Markdown-first team. |
| **Material for MkDocs** | Theme | Search, dark mode, responsive nav, code copy — a professional docs site with zero custom front-end work. |
| **Read the Docs** | Hosting | Free hosted builds triggered on every `git push`, versioned docs, a public `readthedocs.io` URL. |

- **MkDocs over Sphinx** — our docs are already Markdown; MkDocs consumes Markdown natively, whereas
  Sphinx centres on reStructuredText. MkDocs' config is a single readable YAML file.
- **Documentation-only public repo** — the application source is proprietary, so the public docs
  repo contains **only** the Markdown, `mkdocs.yml`, and `.readthedocs.yaml` (history rewritten to
  guarantee no source or secrets are present). Read the Docs only needs those files to build.

---

## 8. Cross-cutting decisions & trade-offs

| Decision | Rationale | Trade-off accepted |
|----------|-----------|--------------------|
| Split into 4 services | Isolate workloads that differ in speed, cost and runtime | More processes to run + CORS setup |
| Vertex sampling for clearance | Interactive latency on large meshes | Approximate gap unless refined |
| NDJSON streaming | Live progress feedback | Slightly more complex client parsing |
| In-memory data, no DB | Read-only reference data; zero ops | Runtime-added zones not persisted |
| Rules in code **and** in the LLM prompt | Reproducible verdict + anchored reasoning | Duplicated thresholds to keep in sync |
| Enterprise LLM gateway (Capgemini) | Governed, billed, approved model access | Indirection vs. calling Anthropic directly |
| Multi-view composite for the LLM | Removes 3D ambiguity, better verdicts | Extra client-side rendering work |
| Offline preprocessing to JSON | Keeps the interactive app fast | Data can go stale vs. the CAD source |

### Known limitations (honest accounting)

- Only the **Electrical + Suspension** zone currently ships with STL meshes; the other six have
  metadata only, so clearance/viewer/AI only work fully for that zone
  ([details](DATA_AND_PREPROCESSING.md#4-data-availability)).
- The clearance stream reports progress but computes rows up front.
- `packaging_rules.json` is a **reference spec**, not loaded at runtime — the operative rules are the
  code + prompt copies, which must be kept in sync manually.
- Live API keys were present in local `.env` files (never committed) — rotate them before sharing.

---

## 9. Full stack at a glance

| Layer | Technology | Chosen because |
|-------|-----------|----------------|
| UI framework | React 19 | Ecosystem, ecosystem, concurrent rendering |
| Framework/routing | TanStack Start + Router | Type-safe, Vite-native, SSR-capable |
| Build/dev | Vite 7 | Instant HMR, modern bundling |
| 3D | Three.js (imperative) | Only mature in-browser mesh engine; needed for the AI composite |
| Styling | Tailwind v4 + Radix/shadcn | Fast + accessible |
| Types/validation | TypeScript + Zod | Contract safety across two APIs |
| Edge host | Cloudflare Workers | Cheap, global, pairs with TanStack |
| Backend framework | FastAPI + Uvicorn | Async, Pydantic validation, streaming, auto-docs |
| Geometry | trimesh + NumPy | Signed-distance proximity + vectorised maths |
| Language (services) | Python 3.10 | Only ecosystem with the mesh + AI libraries |
| AI (analysis) | Claude Sonnet 4.5 via Capgemini Generative Engine (OpenAI SDK) | Vision + reasoning through a governed enterprise gateway |
| AI (images) | Capgemini Assets API — gemini-2.5-flash-image | Text→image correction visuals |
| Data format | JSON | Portable, readable, decouples CAD from app |
| Image lib | Pillow | Base64/PNG/JPEG handling for the vision pipeline |
| Docs | MkDocs + Material + Read the Docs | Markdown-native, professional, auto-published |

---

*This report explains the reasoning behind the choices; for the concrete "how", see
[Architecture](ARCHITECTURE.md), [Setup & Run](SETUP_AND_RUN.md), and [API Reference](API_REFERENCE.md).*
</content>
