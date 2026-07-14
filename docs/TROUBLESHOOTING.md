# Troubleshooting

Common issues, root causes, and fixes. Most are environment/port related.

## The blank page / port 8080 conflict

**Symptom:** the app shows nothing; DevTools console shows
`Failed to load resource: 404 (edblogo.png)` and `404 (:8080/favicon.ico)`.

**Root cause:** Vite's bundled config defaults to **host `::`, port 8080**, so Vite binds IPv6
`[::]:8080`. But this machine already runs an **EnterpriseDB Apache `httpd`** on IPv4
`0.0.0.0:8080`. When the browser opens `http://localhost:8080`, Windows resolves `localhost` to
**IPv4 first** and hits the EDB "Server is up and running" page (which references `edblogo.png`),
not the Vite app.

**Fix - run the frontend on 5173** (free, and already in both backends' CORS allow-list):

```powershell
cd "space-weaver-main"
npm run dev -- --port 5173 --strictPort
# then open http://localhost:5173/  (hard-refresh; don't reuse the :8080 tab)
```

Verify you're hitting Vite, not EDB - the served HTML title must be
`Packaging Analysis - MG2 Engineering`:

```powershell
(Invoke-WebRequest http://localhost:5173/ -UseBasicParsing).Content.Substring(0,300)
```

Diagnose port ownership (two rows on 8080 = the conflict):

```powershell
Get-NetTCPConnection -LocalPort 8080 -State Listen |
  ForEach-Object { "{0} PID {1} {2}" -f $_.LocalAddress,$_.OwningProcess,(Get-Process -Id $_.OwningProcess).ProcessName }
# expect:  ::       node   (Vite)
#          0.0.0.0  httpd  (EnterpriseDB - leave it alone)
```

Do **not** stop the EDB `httpd`/`postgres` services - they are unrelated database services.

## Port already in use

```powershell
# find the owner
Get-NetTCPConnection -LocalPort 8000 -State Listen | Select OwningProcess
# free it (only if it's a stale dev process you own)
Stop-Process -Id <pid> -Force
```
Or run on a different port: backend `$env:BACKEND_PORT="8010"`, MG2 `$env:MG2_PORT="8011"`.
If you change MG2's port, update `MG2_API_BASE` in `space-weaver-main/src/lib/api.ts`.

## Orphaned node process won't die ("Access denied")

A previous `npm run dev` can leave a `node` process holding a port that resists `Stop-Process`
/ `taskkill /F`. It's harmless if you've moved to another port. To clear it, end it from Task
Manager, or just close the terminal/session that spawned it.

## "Cannot reach MG2 service" / Analyze fails

1. Confirm MG2 is up: `Invoke-WebRequest http://127.0.0.1:8001/health`.
2. The **Analyze** button requires the 3D viewer to be **open in ENV mode and ready** (the button
   is gated on `captureReady`). Open with **3D‑ENV** and wait for meshes to load.
3. Check CORS: the frontend origin (`http://localhost:5173`) must be in MG2's `allow_origins`
   (it is, by default).
4. Check the MG2 terminal for LLM errors (see below).

## Empty clearance table / "NO_STL" / viewer loads nothing

The zone you selected has **no STL meshes on disk**. Only **Electrical + Suspension** ships with
meshes (229 files). See [DATA_AND_PREPROCESSING.md](DATA_AND_PREPROCESSING.md#4-data-availability).
Use that zone, or add the missing `.stl` files to the matching STL subfolder.

## Zones missing after backend restart

`MOCK_ZONES` is in-memory. Only the **7 predefined zones** auto-reload; zones added via the UI's
custom-zone loader are lost on restart. Re-load them, or add them to
`auto_load_predefined_zones()` in `space-weaver-main/backend.py`.

## LLM / API-key errors during analysis

- The MG2 **server** uses the **Capgemini Generative Engine** (`GENERATIVE_ENGINE_API_KEY`,
  `GENERATIVE_ENGINE_BASE_URL`). These keys are present in
  `MG2_NOK_Correction_System_fixed/.env`. If analysis fails, verify they're valid and reachable.
- `ANTHROPIC_API_KEY` is only needed for the MG2 **CLI** `main.py --api anthropic`, not the app.
- Image generation (`image_generator.py`) needs `CAPGEMINI_ASSETS_API_KEY` (Assets API - a
  different service). Not required for the core Analyze flow.

## `peugeot_preprocessor2/venv2` is broken

Its `pyvenv.cfg` points to a nonexistent `C:\Program Files\Python311`. Use the **global Python**
for the preprocessor/engine CLIs, or recreate the venv:

```powershell
cd "peugeot_preprocessor2"
python -m venv venv2
venv2\Scripts\activate
pip install -r requirements.txt
```

## `trimesh` / STL loading errors

- `/preprocess/*` return HTTP 400 on a bad mesh, HTTP 500 if `trimesh` isn't installed.
- Ensure STL filenames exactly match `part_name` + `.stl` (the resolver is case-insensitive on
  the extension but matches the stem).

## Native file picker (`/browse/*`) doesn't appear

These open a **tkinter** dialog on the **server host** (where `backend.py` runs), not in the
browser. On a headless/remote host they won't show. Use the web-based `FileExplorerModal`
(`/fs/list`) instead. Cancelling a picker returns **HTTP 204** (treated as an error by the client).

## Quick health sweep

```powershell
Invoke-WebRequest http://127.0.0.1:8000/health -UseBasicParsing | Select -Expand Content
Invoke-WebRequest http://127.0.0.1:8001/health -UseBasicParsing | Select -Expand Content
Invoke-WebRequest http://127.0.0.1:8000/zones  -UseBasicParsing | Select -Expand Content
Invoke-WebRequest http://localhost:5173/        -UseBasicParsing | Select -Expand StatusCode
```
</content>
