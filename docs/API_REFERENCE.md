# API Reference

Two FastAPI services expose HTTP JSON APIs. All examples assume `localhost`.

- **Main backend** - `http://127.0.0.1:8000` - `space-weaver-main/backend.py`
- **MG2 AI server** - `http://127.0.0.1:8001` - `MG2_NOK_Correction_System_fixed/mg2_api_server.py`

CORS on both allows `localhost`/`127.0.0.1` on ports 3000, 5173, 8080 (+ a few more on MG2).

---

## Part 1 - Main backend (port 8000)

### `GET /health`
Health check. → `{ "status": "ok", "service": "space-weaver Backend", "timestamp": "<iso>" }`

### `GET /zones`
List all loaded zones.
```json
{ "zones": [ { "zone": "Electrical_Suspension", "folder_label": "<abs path>", "part_count": 229 }, ... ] }
```

### `GET /zones/{zone_name}/parts`
Parts for one zone (404 if unknown).
```json
{
  "zone": "Electrical_Suspension",
  "folder_label": "<abs path>",
  "total": 229,
  "parts": [
    { "part_name": "...", "part_id": null, "folder_label": "...", "folder_key": "...",
      "bounding_box": {"min":[x,y,z],"max":[x,y,z]}, "volume": 1234.5, "centroid":[x,y,z] }
  ]
}
```

### `POST /clearance`  → **NDJSON stream** (`application/x-ndjson`)
Compute real mesh-to-mesh clearance for a study part against neighbours.

Request (`ClearanceRequest`):
```json
{ "study_part": "PartA", "neighbour_names": ["PartB","PartC"], "search_folder": "<optional abs path>" }
```
Each streamed line (one per neighbour):
```json
{ "type":"progress", "neighbour_name":"PartB", "clearance_mm":5.234, "interference":false,
  "penetration_mm":null, "method":"real_engine_sampled",
  "point_a":[x,y,z], "point_b":[x,y,z], "done":1, "total":2, "error":null }
```
- `method` is `real_engine_sampled` for a gap, or `real_engine_collision_signed_distance` for interference.
- On interference: `interference:true`, `clearance_mm:null`, `penetration_mm:<depth>`, and `point_a`/`point_b` are the contact points.
- Uses seeded vertex sampling (≤1000 verts/mesh) + `trimesh.proximity.signed_distance`.

### `POST /clearance/refine-penetration`
Higher-fidelity penetration depth for one interfering pair.

Request (`RefinePenetrationRequest`): `{ study_part, neighbour_name, search_folder?, sample_count?=500 }`
→ `{ study_part, neighbour_name, interference, penetration_mm, samples_used, elapsed_sec }`
(`sample_count` is echoed as `samples_used` but the code samples a hardcoded ≤1000.)

### `GET /stl-proxy?folder=<abs>&name=<part>`
Streams an STL file (`model/stl`) resolved from `folder` (+ immediate subfolders). 404 if not found.
Used by the 3D viewer for zones whose `folder_label` is an absolute path.

### `POST /load-zone`
Load a custom zone from a JSON file + STL folder; background-caches meshes.
Request (`LoadZoneRequest`): `{ stl_folder, json_path }` → `{ zone, json_file, part_count, parts }`.

### `POST /preprocess/single`
Read one STL and return geometry metadata.
Request: `{ stl_path, part_name?, folder_key?="custom", folder_label?="" }`
→ `{ part_name, folder_key, folder_label, bounding_box, centroid, dimensions, part_type }`.

### `POST /preprocess/folder`
Read every `*.stl` in a folder.
Request: `{ folder_path, folder_key?, folder_label? }`
→ `{ folder_key, folder_label, total_found, total_processed, errors, parts }`.

### `GET /fs/list?path=<dir>&ext=<csv>`
Web filesystem browser: `{ current_path, parent_path, drives, items:[{name,path,is_dir,size?}] }`
(folders first; optional extension filter).

### `GET /browse/folder` · `GET /browse/file` · `GET /browse/stl-file`
Open a **native tkinter picker** on the server host. → `{ "path": "<selected>" }`.
Returns **HTTP 204 as an error** when the dialog is cancelled.

### `POST /ai-analysis`  (proxy to MG2)
Request (`AiAnalysisRequest`): `{ study_part, images: [ {neighbour_name, image_b64, clearance_mm, interference}, ... ] }`
Proxies to `POST http://{MG2_HOST}:{MG2_PORT}/analyze-batch` (`api_type:"capgemini", include_ok:false`, 180 s).
→ `{ success:true, report_url, overall_summary }` or `{ success:false, error }`.

---

## Part 2 - MG2 AI server (port 8001)

### `GET /health`
→ `{ "status":"ok", "service":"MG2 NOK Correction System", "timestamp":"<iso>" }`

### `GET /reports/{path}`
Serves generated report files from `output/` (`FileResponse`, 404 if missing).

### `POST /analyze`  - single image
Request (`AnalysisRequest`):
```json
{ "study_part":"PartA", "neighbour_part":"PartB", "image_base64":"<png b64>",
  "gap_mm":5.23, "interference":false,
  "study_part_type":"Fixed", "neighbour_part_type":"Moving" }
```
Response (`AnalysisResponse`):
```json
{ "study_part":"PartA", "neighbour_part":"PartB", "gap_mm":5.23,
  "verdict":"WARNING",
  "analysis": { "visual_observations":[...], "failure_mode_analysis":{...},
                "problem_explanation":"...", "scenarios":[ {...}, {...}, {...} ],
                "recommended_scenario_id":2, "confidence_score":0.8, "ccp_report_entry":"..." },
  "error": null, "report_url":"http://127.0.0.1:8001/reports/NOK_Report_<ts>.html" }
```
`verdict` ∈ `CLASH | NOK | WARNING | OK | ERROR | UNKNOWN`.
Pipeline: base64 → temp PNG (structured filename) → `classify_images` → `analyse_nok_images(api_type="capgemini")` → `generate_html_report`.

### `POST /analyze-batch`  - many images
Request (`BatchAnalysisRequest`):
```json
{ "images":[ { "study_part":"A","neighbour_part":"B","image_base64":"...",
               "gap_mm":5.2,"interference":false,
               "study_part_type":"Fixed","neighbour_part_type":"Moving" }, ... ],
  "api_type":"anthropic", "include_ok":false }
```
Response (`BatchAnalysisResponse`):
```json
{ "analysis_id":"20260712_143000", "timestamp":"<iso>", "total_images":2,
  "results":[ <AnalysisResponse>, ... ], "report_url":"..." }
```
> `api_type` is **ignored** - the server always uses Capgemini. `include_ok:false` skips OK images.

### `POST /analyze-file`  - multipart upload
Form field `file` (an image). Reads `AI_API` env (default `anthropic`). → `AnalysisResponse` (HTTP 500 on error).

### `POST /generate-final-report`  - validated engineering report
Request (`FinalReportRequest`):
```json
{ "study_part":"A", "study_part_type":"Moving", "zone":"Electrical_Suspension",
  "clearances":[ {...} ], "validated_scenarios":[ {...} ] }
```
→ `{ "report_url":"http://127.0.0.1:8001/reports/Final_Report_<ts>.html" }`
Produces a 3-part human-validated HTML report (environment verification, clearance verdicts,
approved corrective scenarios).

---

## Output artifacts

Written to `MG2_NOK_Correction_System_fixed/output/` and served under `:8001/reports/`:

- `NOK_Report_<timestamp>.html` - full analysis dashboard
- `NOK_Results_<timestamp>.json` - raw results dump
- `Final_Report_<timestamp>.html` - validated engineering report

## curl examples

```bash
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/zones

curl -X POST http://127.0.0.1:8000/clearance \
  -H "Content-Type: application/json" \
  -d '{"study_part":"PartA","neighbour_names":["PartB"]}'

curl -X POST http://127.0.0.1:8001/analyze \
  -H "Content-Type: application/json" \
  -d '{"study_part":"A","neighbour_part":"B","image_base64":"iVBORw0KGgo...","gap_mm":5.2}'
```
</content>
