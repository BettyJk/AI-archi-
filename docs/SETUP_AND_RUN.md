# Setup & Run

How to install dependencies, configure credentials, and run all three services of the
integrated application. Commands are for **Windows PowerShell** (the verified environment).

## 1. Prerequisites

| Tool | Version (verified) | Notes |
|------|--------------------|-------|
| Python | 3.10 | The **global** Python has every backend dependency installed |
| Node.js | 18+ (v26 verified) | Ships with npm |
| Git Bash / POSIX shell | optional | Not required; PowerShell is sufficient |

Python packages required by the backends: `fastapi`, `uvicorn`, `trimesh`, `numpy`, `pydantic`,
`Pillow (PIL)`, `anthropic`, `openai`, `python-dotenv`, `requests`. These are present in the
global interpreter and in `MG2_NOK_Correction_System_fixed/venv_mg2`.

### Virtual environments present

- `MG2_NOK_Correction_System_fixed/venv_mg2` — the MG2 server's venv (working).
- `peugeot_preprocessor2/venv2` — **broken** (its `pyvenv.cfg` points to a nonexistent
  `C:\Program Files\Python311`). Use the global Python for the preprocessor CLIs, or recreate it.

## 2. Install

```powershell
# Frontend deps (first run only)
cd "space-weaver-main"
npm install

# MG2 Python deps (into the existing venv)
cd "..\MG2_NOK_Correction_System_fixed"
venv_mg2\Scripts\activate
pip install -r requirements.txt
deactivate

# (Backend runs on global Python which already has fastapi/trimesh/numpy/etc.)
```

## 3. Environment variables

### `MG2_NOK_Correction_System_fixed/.env`

Present keys (used by the running server's Capgemini path):

```env
CAPGEMINI_ASSETS_API_KEY=...        # image generation (Assets API)
GENERATIVE_ENGINE_API_KEY=...       # LLM chat/vision (Generative Engine)
GENERATIVE_ENGINE_BASE_URL=https://openai.generative.engine.capgemini.com/v1
```

Optional / for the Anthropic path (MG2 CLI only):

```env
ANTHROPIC_API_KEY=sk-ant-...        # only needed if you run main.py --api anthropic
MG2_HOST=0.0.0.0
MG2_PORT=8001
DEBUG=false
AI_API=anthropic                    # default provider for /analyze-file (not the batch routes)
```

> The MG2 **server** always uses the Capgemini Generative Engine for `/analyze` and
> `/analyze-batch` (see [ARCHITECTURE.md](ARCHITECTURE.md#4-data-flow-ai-analysis-of-a-clearance)),
> so `ANTHROPIC_API_KEY` is not required to run the app as integrated.

### `peugeot_preprocessor2/.env`

```env
LLM_OPENAI_KEY=...                  # Capgemini Generative Engine key for Ai analyst.py
```

### Backend (`backend.py`) — no `.env`, only process env vars

```env
BACKEND_HOST=0.0.0.0   # default
BACKEND_PORT=8000      # default
DEBUG=false            # true → uvicorn auto-reload
MG2_HOST=127.0.0.1     # used only to build the proxy URL for /ai-analysis
MG2_PORT=8001
```

> **Security:** both `.env` files contain live-looking keys. Rotate them if the repo is shared
> and prefer keeping secrets out of version control.

## 4. Run all three services

Open three terminals (or run each in the background).

### Terminal 1 — Main backend (port 8000)

```powershell
cd "space-weaver-main"
$env:BACKEND_HOST="127.0.0.1"; $env:BACKEND_PORT="8000"
python backend.py
```

Expect: `Uvicorn running on http://127.0.0.1:8000` and lines like
`Auto-loaded predefined zone: Electrical_Suspension (229 parts)`.

### Terminal 2 — MG2 AI server (port 8001)

```powershell
cd "MG2_NOK_Correction_System_fixed"
venv_mg2\Scripts\activate
$env:MG2_HOST="127.0.0.1"; $env:MG2_PORT="8001"
python mg2_api_server.py
```

Expect: `Uvicorn running on http://127.0.0.1:8001`.

### Terminal 3 — Frontend (port 5173)

```powershell
cd "space-weaver-main"
npm run dev -- --port 5173 --strictPort
```

Expect: `VITE vX ready` and `Local: http://localhost:5173/`.
Open **http://localhost:5173/**.

> **Why 5173 and not 8080?** Vite's bundled config defaults to port 8080, but on this machine an
> EnterpriseDB Apache `httpd` occupies IPv4 `0.0.0.0:8080`. `localhost` resolves to IPv4 first, so
> `http://localhost:8080` serves the EDB placeholder page instead of the app. Port 5173 is free
> and already whitelisted in both backends' CORS config. Details:
> [TROUBLESHOOTING.md](TROUBLESHOOTING.md#the-blank-page--port-8080-conflict).

## 5. Verify everything is up

```powershell
# Backends healthy
Invoke-WebRequest http://127.0.0.1:8000/health -UseBasicParsing | Select -Expand Content
Invoke-WebRequest http://127.0.0.1:8001/health -UseBasicParsing | Select -Expand Content

# Real zones loaded (should list 7 zones with part counts)
Invoke-WebRequest http://127.0.0.1:8000/zones -UseBasicParsing | Select -Expand Content

# Frontend serving the app (title should be "Packaging Analysis — MG2 Engineering")
Invoke-WebRequest http://localhost:5173/ -UseBasicParsing | Select -Expand Content

# Confirm which process owns each port
foreach ($p in 8000,8001,5173) {
  Get-NetTCPConnection -LocalPort $p -State Listen -ErrorAction SilentlyContinue |
    ForEach-Object { "{0}: PID {1} {2}" -f $p,$_.OwningProcess,(Get-Process -Id $_.OwningProcess).ProcessName }
}
```

## 6. Optional: the preprocessor & engine CLIs

These are standalone tools (not needed to run the app). Use the global Python.

```powershell
cd "peugeot_preprocessor2"

# Regenerate zone JSON from STL folders (CAD → JSON)
python "stl_files\peugeot_preprocessor\preprocess.py"

# Compute clearance for a single STL (writes analysis_<name>.json)
python engine.py --stl "stl_files\Peugeot 3008(Electrical + Suspension)\<part>.stl"

# LLM analysis of an engine output file (writes ai_report_<name>.json)
python "Ai analyst.py" --analysis analysis_<part>.json
```

## 7. Stopping / restarting

- Ctrl+C in each terminal, or stop the background tasks.
- Zones added at runtime via the UI's "custom zone" loader are **not persisted** — only the 7
  predefined zones reload automatically on backend restart.
</content>
