# Data & Preprocessing

How vehicle CAD data becomes the zone files the app consumes, the exact schemas, the clearance
algorithm, and — importantly — **what data is actually present on disk**.

## 1. The data pipeline

```
STL meshes (per zone)  ──preprocess.py──▶  data/*.json (part index)  ──backend.py──▶  app
      │                                          │
      │                                          └─ bounding_box, centroid, volume, surface_area
      └─ one .stl per part                          (+ part_type / circuit_type added downstream)
```

1. **CAD → JSON.** `peugeot_preprocessor2/stl_files/peugeot_preprocessor/preprocess.py`
   walks each zone's STL folder, loads every `.stl` with `trimesh.load(..., force="mesh")`, and
   extracts spatial metadata per part. It writes per-folder JSON to `output/by_folder/<key>.json`
   and a combined `output/parts_index.json`.
2. **Enrichment.** A downstream step (not in `preprocess.py`) adds `part_type` (`fixed`/`moving`)
   and `circuit_type` (`null`, or `cooling`/`harness`/`cable`/…) to each part.
3. **Consumption.** `space-weaver-main/backend.py` loads
   `peugeot_preprocessor2/data/*.json` at startup into the in-memory `MOCK_ZONES` and serves them.

## 2. Zone JSON schema

Each `data/<Zone>.json` is a JSON **array of part objects**:

```json
{
  "part_name":    "IB_6-4122-1114_Plugs_Support",
  "folder_key":   "Peugeot 3008 [Accessories + Seat]",
  "folder_label": "Peugeot 3008 [Accessories + Seat]",
  "file_path":    "C:\\...\\IB_6-4122-1114_Plugs_Support.stl",
  "bounding_box": { "min": [x, y, z], "max": [x, y, z] },
  "centroid":     [x, y, z],
  "volume":       60972.90,
  "surface_area": 30982.20,
  "part_type":    "fixed",     // or "moving"  (enrichment step)
  "circuit_type": null          // or "cooling" / "harness" / "cable" / ...  (enrichment step)
}
```

`part_type` (fixed/moving) is a **different taxonomy** than `engine.py`'s keyword-based
`detect_part_type()` (hv_cable/pipe/harness/…). The backend also injects `zone`, `folder_label`,
and `folder_key` into each part at load time.

## 3. Zones on disk

**`peugeot_preprocessor2/data/` — 7 zone files, ~1,709 parts total:**

| Zone JSON | Parts |
|-----------|-------|
| `AC_Heating_Brakes.json` | 624 |
| `Interior.json` | 230 |
| `Electrical_Suspension.json` | 229 |
| `IB_Body2.json` | 198 |
| `IB_Body3.json` | 168 |
| `Accessories_Seat.json` | 158 |
| `Pedals_Safety_Steering.json` | 102 |

**`peugeot_preprocessor2/stl_files/` — 8 zone subfolders + the nested preprocessor.**

## 4. Data availability

> **Only one zone has STL meshes on disk.**
>
> | STL folder | Meshes |
> |------------|--------|
> | `Peugeot 3008(Electrical + Suspension)` | **229 `.stl` + `hierarchy.json`** ✅ |
> | `Peugeot 3008 [Accessories + Seat]` | empty |
> | `Peugeot 3008 [Air conditioning + Heating + Brakes Mechanism]` | empty |
> | `IB - Peugeot 3008 [BODY 2]` | empty |
> | `IB - Peugeot 3008 [Body 3]` | empty |
> | `Peugeot 3008 [Interior]` | empty |
> | `Peugeot 3008 [Pedals + Safty +Steering]` | empty |

Consequences:
- **Zone/part lists work for all 7 zones** (they come from the JSON metadata).
- **Clearance, the 3D viewer, and AI analysis only work end-to-end for *Electrical + Suspension*.**
  For the other six zones the backend can't find STL files, so neighbours resolve to errors /
  `NO_STL` and the viewer has nothing to load.

To enable the other zones, place their `.stl` files into the matching STL subfolders (filenames
must equal `part_name` + `.stl`) — or regenerate with the preprocessor if you have the CAD exports.

## 5. The clearance engine

Two implementations share the same algorithm:

- **Production path:** `backend.py` `POST /clearance` (see [API_REFERENCE.md](API_REFERENCE.md)).
- **Standalone CLI:** `peugeot_preprocessor2/engine.py`.

**Algorithm (engine.py):**
1. Load all `data/*.json` part metadata.
2. Load the study STL; compute bounds, per-axis dimensions, centroid.
3. **Adaptive expansion** per axis: `expansion(dim) = min(max(dim × Per/100, MinDis), MaxDis)`
   (defaults Per=10 %, MinDis=20 mm, MaxDis=150 mm). Small parts floor at MinDis, large parts cap at MaxDis.
4. **Neighbour detection:** expanded study AABB must overlap a part's stored AABB on all 3 axes;
   neighbours sorted by centroid distance.
5. **Exact clearance:** sample up to **300 vertices** per mesh (engine.py) / **1000** (backend.py),
   compute the full pairwise distance matrix, take the minimum (mm). ≤0 = clash.
6. **Rule + verdict:** map the part-type pair to `(min_mm, target_mm, criticality)` and classify
   `CLASH (≤0) / NOK (<min) / WARNING (<target) / OK`. See [CLEARANCE_RULES.md](CLEARANCE_RULES.md).

**engine.py output** (`analysis_<study>.json`): `study_part, study_type, bounds, centroid,
dimensions, params, expansions, summary{clashes,noks,warnings,oks}, results[{neighbour_name,
neighbour_type, clearance_mm, min_mm, target_mm, criticality, verdict, verdict_reason,
centroid_distance_mm, ...}]`. Neighbours without an STL get `verdict:"NO_STL"`.

## 6. Preprocessor CLIs

Run with the global Python (the `venv2` in this folder is broken). Requirements:
`numpy, trimesh, anthropic, fastapi, uvicorn[standard], python-dotenv, pydantic`.

```powershell
cd "peugeot_preprocessor2"

# CAD → JSON (regenerate zone metadata from STL folders)
python "stl_files\peugeot_preprocessor\preprocess.py"

# Compute clearance for one part
python engine.py --stl "stl_files\Peugeot 3008(Electrical + Suspension)\<part>.stl" \
                 --per 10 --min_dis 20 --max_dis 150

# LLM analysis of an engine output (writes ai_report_<part>.json)
python "Ai analyst.py" --analysis analysis_<part>.json
```

- `Ai analyst.py` uses the **Capgemini Generative Engine** (OpenAI SDK, `openai.gpt-4o`,
  key `LLM_OPENAI_KEY`) — despite the function being named `call_claude`.
- The legacy `main.py` backend's `call_claude_ai` uses **Anthropic** (`claude-sonnet-4-20250514`,
  `ANTHROPIC_API_KEY`).

## 7. `history/`

Empty. The legacy `main.py` backend persists each analysis (8-char UUID) here; the integrated app
does not use it.
</content>
