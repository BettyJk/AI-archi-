# Packaging Analysis Platform 

An AI-assisted **automotive packaging clearance analysis** platform. It automates what the
MG2 engineering  *Analyse d'implantation* team traditionally does by hand in CATIA: measuring
3D gaps between vehicle parts, checking them against packaging clearance rules, and proposing
corrective actions for violations.

The platform lets an engineer:

1. Load a **zone** of the vehicle (a group of parts, e.g. *Electrical + Suspension*).
2. Pick a **study part** and automatically detect its **neighbours** in 3D space.
3. Compute **real mesh-to-mesh clearance** (minimum distance / interference / penetration depth).
4. Inspect everything in an interactive **3D viewer** (Three.js).
5. Run **AI analysis** on a clearance result to get a verdict (`OK / WARNING / NOK / CLASH`),
   a plain-language explanation, and concrete **corrective scenarios**.
6. Generate **HTML engineering reports**.

## System at a glance

```
Frontend (React + Three.js, :5173)
   │  HTTP (JSON / NDJSON)          │  HTTP (base64 PNG)
   ▼                                 ▼
Main backend backend.py (:8000) ──▶ MG2 NOK Correction System (:8001)
   │  reads                          │  calls
   ▼                                 ▼
peugeot_preprocessor2/            Capgemini Generative Engine / Anthropic / Assets API
  data/*.json  +  stl_files/
```

### The four components

| # | Component | Stack | Role |
|---|-----------|-------|------|
| 1 | **Frontend** (space-weaver) | React 19, TanStack Start, Three.js, Vite | UI: zone/part selection, 3D viewer, clearance table, reports |
| 2 | **Main Backend** (`backend.py`) | FastAPI, trimesh, numpy | Zones/parts, real clearance, STL serving, AI proxy |
| 3 | **MG2 NOK Correction System** | FastAPI, LLM SDKs | AI analysis → verdicts, corrective scenarios, HTML reports |
| 4 | **Preprocessor + data** | Python, trimesh | CAD→JSON tool + the zone data and STL meshes |

## Quick start (Windows)

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

!!! warning "Do not use port 8080"
    Vite's default is 8080, but on the reference machine an **EnterpriseDB Apache `httpd`** already
    listens on IPv4 `0.0.0.0:8080`. Because `localhost` resolves to IPv4 first, `http://localhost:8080`
    hits the EDB page instead of the app. Always run the frontend on **5173**.
    See [Troubleshooting](TROUBLESHOOTING.md#the-blank-page--port-8080-conflict).

!!! note "Data availability"
    Only the **Electrical + Suspension** zone currently has STL meshes on disk (229 files). The
    other six zones have part metadata (JSON) but empty STL folders, so clearance/viewer/AI steps
    only work end-to-end for that zone. See
    [Data & Preprocessing](DATA_AND_PREPROCESSING.md#4-data-availability).

## Documentation

| Page | Contents |
|------|----------|
| [Project Report & Rationale](PROJECT_REPORT.md) | The project explained + why each technology was chosen over the alternatives |
| [Architecture](ARCHITECTURE.md) | System architecture, data flow, the two backends, AI providers |
| [Setup & Run](SETUP_AND_RUN.md) | Prerequisites, install, env vars, running all services, verification |
| [API Reference](API_REFERENCE.md) | Every endpoint of the backend (8000) and MG2 (8001) |
| [Frontend](FRONTEND.md) | Frontend structure, components, 3D viewer, `api.ts`, workflow |
| [Data & Preprocessing](DATA_AND_PREPROCESSING.md) | Zone JSON schema, CAD→JSON preprocessor, engine, data availability |
| [Clearance Rules](CLEARANCE_RULES.md) | SCR structural rules, CCP circuit rules, verdict thresholds |
| [Troubleshooting](TROUBLESHOOTING.md) | Port conflicts, blank page, CORS, empty STL zones, LLM/API keys |

## Security note

Live-looking API keys are committed in `MG2_NOK_Correction_System_fixed/.env` and
`peugeot_preprocessor2/.env`. These are **excluded from git** via `.gitignore`. Rotate them if they
were ever shared, and keep secrets out of version control.
</content>
