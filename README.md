# Packaging Analysis Platform — Peugeot 3008 (MG2 Engineering)

An AI-assisted **automotive packaging clearance analysis** platform. It automates what the
Stellantis / PSA *Synthèse Implantation* team traditionally does by hand in CATIA: measuring
3D gaps between vehicle parts, checking them against packaging clearance rules, and proposing
corrective actions for violations.

The platform lets an engineer:

1. Load a **zone** of the vehicle (a group of parts, e.g. *Electrical + Suspension*).
2. Pick a **study part** and automatically detect its **neighbours** in 3D space.
3. Compute **real mesh-to-mesh clearance** (minimum distance / interference / penetration depth).
4. Inspect everything in an interactive **3D viewer** (Three.js).
5. Run **AI analysis** on a clearance result to get a verdict (`OK / WARNING / NOK / CLASH`),
   a plain-language explanation, and concrete **corrective scenarios** (which part to move,
   along which axis, by how many mm).
6. Generate **HTML engineering reports**.

---

## 1. System at a glance

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Frontend — space-weaver  (React 19 + TanStack Start + Three.js + Vite)    │
│  Zone/part selection · neighbour detection · 3D viewer · clearance table   │
│  DEV PORT: 5173  (see note on 8080 below)                                  │
└───────────────┬──────────────────────────────────┬───────────────────────┘
                │ HTTP (JSON / NDJSON)              │ HTTP (JSON, base64 PNG)
                ▼                                    ▼
┌───────────────────────────────────┐   ┌───────────────────────────────────┐
│  space-weaver Backend (backend.py) │   │  MG2 NOK Correction System         │
│  FastAPI · PORT 8000               │   │  (mg2_api_server.py) FastAPI · 8001 │
│  • zones / parts                   │   │  • classify clearance images        │
│  • real clearance (trimesh)        │──▶│  • AI corrective analysis (LLM)     │
│  • STL serving                     │   │  • HTML report generation           │
│  • proxies AI analysis to MG2 ─────┼──▶│                                     │
└───────────────┬───────────────────┘   └───────────────┬───────────────────┘
                │ reads                                   │ calls
                ▼                                         ▼
┌───────────────────────────────────┐   ┌───────────────────────────────────┐
│  peugeot_preprocessor2/            │   │  Capgemini Generative Engine (LLM)  │
│  • data/*.json  (zone part index)  │   │  and/or Anthropic Claude            │
│  • stl_files/   (3D meshes)        │   │  Capgemini Assets API (image gen)   │
│  • preprocess.py (CAD → JSON)      │   └───────────────────────────────────┘
└───────────────────────────────────┘
```

### The four components

| # | Component | Folder | Language / stack | Role |
|---|-----------|--------|------------------|------|
| 1 | **Frontend** (space-weaver) | `space-weaver-main/` | React 19, TanStack Start/Router, Three.js, Vite, Tailwind v4 | The UI: zone/part selection, 3D viewer, clearance table, report generation |
| 2 | **Main Backend** | `space-weaver-main/backend.py` | Python, FastAPI, trimesh, numpy | Serves zones/parts, computes real clearance, serves STL, proxies AI analysis |
| 3 | **MG2 NOK Correction System** | `MG2_NOK_Correction_System_fixed/` | Python, FastAPI, LLM SDKs | AI analysis of clearance images → verdicts, corrective scenarios, HTML reports |
| 4 | **Preprocessor + data** | `peugeot_preprocessor2/` | Python, trimesh | The CAD→JSON tool + the zone data (JSON part index) and STL meshes consumed by the backend |

> **This is the documentation repository.** The application **source code is kept in a private
> repository**; only the documentation is published here (and on Read the Docs). Paths like
> `space-weaver-main/backend.py` below refer to files in that private repo.

> **Note on the two Python backends.** The *integrated* application uses
> `space-weaver-main/backend.py` (port 8000) as its data/clearance backend.
> `peugeot_preprocessor2/main.py` is a **separate, earlier standalone FastAPI backend** ("Week 3")
> with its own engine and AI path. You do **not** run it for the integrated app — it is kept for
> reference and for the preprocessing/engine CLIs.
> See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md#6-the-two-backends).

---

## 2. Quick start (Windows)

Three services must run at once. Full details in [docs/SETUP_AND_RUN.md](docs/SETUP_AND_RUN.md).

```powershell
# Terminal 1 — Main backend (port 8000)
cd "space-weaver-main"
$env:BACKEND_HOST="127.0.0.1"; $env:BACKEND_PORT="8000"
python backend.py

# Terminal 2 — MG2 AI server (port 8001)
cd "MG2_NOK_Correction_System_fixed"
venv_mg2\Scripts\activate
python mg2_api_server.py

# Terminal 3 — Frontend (port 5173)
cd "space-weaver-main"
npm install          # first time only
npm run dev -- --port 5173 --strictPort
```

Then open **http://localhost:5173/**.

> ⚠️ **Do not use port 8080.** Vite's default is 8080, but on this machine an
> **EnterpriseDB Apache `httpd`** already listens on IPv4 `0.0.0.0:8080`. Because `localhost`
> resolves to IPv4 first, `http://localhost:8080` hits the EDB page ("Server is up and running",
> with a broken `edblogo.png`) instead of the app. Always run the frontend on **5173**
> (it is already in every backend's CORS allow-list). See
> [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md#the-blank-page--port-8080-conflict).

### Prerequisites

- **Python 3.10** (verified) — global install has all packages; `venv_mg2` is the MG2 virtualenv.
- **Node.js 18+** (v26 verified) and **npm**.
- LLM API credentials in the MG2 `.env` (Capgemini Generative Engine keys are already present).

---

## 3. End-to-end workflow

1. **Load a zone** — pick a predefined Peugeot 3008 zone (or load a custom STL folder + JSON).
2. **Search / select a study part** in that zone.
3. Adjust **expansion params** (Per % / MinDis / MaxDis) → neighbours are detected automatically.
4. Select which **neighbours** to analyze, and tag each part `Fixed` / `Moving` / `Connected`.
5. Click **3D‑ENV** to open the viewer with the study part + neighbours.
6. Click **CC** to compute clearance → results stream into the **Clearance Table**.
7. For any row, click **Analyze** → the viewer captures a 2400×1600 multi-view AI composite,
   sends it to MG2 (:8001), and shows a verdict + corrective scenarios.
8. Approve scenarios and **Generate Report** → an HTML engineering report is produced.

> **Data availability caveat.** Only the **Electrical + Suspension** zone currently has STL
> meshes on disk (229 files). The other six zones have part metadata (JSON) but **empty STL
> folders**, so clearance/viewer/AI steps only work end-to-end for *Electrical + Suspension*.
> See [docs/DATA_AND_PREPROCESSING.md](docs/DATA_AND_PREPROCESSING.md#4-data-availability).

---

## 4. Documentation index

| Document | What's inside |
|----------|---------------|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Full system architecture, data flow, the two backends, AI providers |
| [docs/SETUP_AND_RUN.md](docs/SETUP_AND_RUN.md) | Prerequisites, install, environment variables, running all services, verification |
| [docs/API_REFERENCE.md](docs/API_REFERENCE.md) | Every endpoint of the backend (8000) and MG2 (8001), with request/response shapes |
| [docs/FRONTEND.md](docs/FRONTEND.md) | Frontend structure, components, 3D viewer, `api.ts`, state & workflow |
| [docs/DATA_AND_PREPROCESSING.md](docs/DATA_AND_PREPROCESSING.md) | Zone JSON schema, the CAD→JSON preprocessor, engine.py, data availability |
| [docs/CLEARANCE_RULES.md](docs/CLEARANCE_RULES.md) | Packaging clearance rules — SCR structural rules, CCP circuit rules, verdict thresholds |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Port conflicts, blank page, CORS, empty STL zones, LLM/API key issues |

### Legacy / integration notes (pre-existing)

- [QUICK_START.md](QUICK_START.md) — original 5-minute integration guide
- [INTEGRATION_GUIDE.md](INTEGRATION_GUIDE.md) — original MG2↔space-weaver integration guide
- [INTEGRATION_SUMMARY.md](INTEGRATION_SUMMARY.md) — integration completion summary
- Capgemini image-generation API setup — see `CAPGEMINI_SETUP_GUIDE.md` in the private application repo

---

## 5. Security note

Live-looking API keys are committed in `MG2_NOK_Correction_System_fixed/.env` and
`peugeot_preprocessor2/.env`. Treat these as secrets: rotate them if this repository is shared,
and consider moving them out of version control. See
[docs/SETUP_AND_RUN.md](docs/SETUP_AND_RUN.md#environment-variables).
</content>
</invoke>
