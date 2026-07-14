# Frontend

The frontend lives in `space-weaver-main/` - a **React 19 + TanStack Start
(SSR-capable) + TanStack Router + Vite + Three.js** single-page tool for automotive packaging
clearance analysis. `package.json` name: `tanstack_start_ts`.

## 1. Tech stack

- **React 19**, **TanStack Start** (`@tanstack/react-start`) + **TanStack Router** (file-based
  routes in `src/routes/`, generated `routeTree.gen.ts`).
- **Vite 7** via the bundled config plugin `@lovable.dev/vite-tanstack-config` - it already
  includes `tanstackStart`, `viteReact`, `tailwindcss`, `tsConfigPaths`, Cloudflare (build-only),
  the dev `componentTagger`, the `@` alias, and **port/host/strictPort** handling. Do **not** add
  those plugins manually.
- **Three.js** (raw, no react-three-fiber) for the 3D viewer.
- **Tailwind CSS v4** (theme via CSS custom properties in `src/styles.css`), **Radix UI** deps +
  shadcn conventions (only `components/ui/input.tsx` is actually vendored), **lucide-react** icons,
  **JSZip** (zip export), **sonner** (toasts).
- **Cloudflare Workers** target: `wrangler.jsonc` (`main: "@tanstack/react-start/server-entry"`,
  `nodejs_compat`). The Cloudflare plugin is build-only.

## 2. Dev server port

`vite.config.ts` is just `export default defineConfig()` from the lovable plugin. That plugin
defaults the dev server to **host `::`, port 8080** (with `strictPort` in sandbox mode). Because
port 8080 collides with an EnterpriseDB `httpd` on this machine, **run the frontend on 5173**:

```powershell
npm run dev -- --port 5173 --strictPort
```

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#the-blank-page--port-8080-conflict).

## 3. Directory structure (`src/`)

```
src/
  router.tsx              getRouter() factory
  routeTree.gen.ts        generated route tree
  styles.css              Tailwind v4 + CSS-variable theme (--accent #00bfa5, --panel, ...)
  routes/
    __root.tsx            SSR HTML shell, <head>, 404 component
    index.tsx             the entire app UI + workflow (single Index component)
  components/
    AppHeader.tsx         logo, light/dark toggle, "STL Preprocessor" button
    AppFooter.tsx
    ZoneSelector.tsx      predefined zone buttons + custom zone loader (STL folder + JSON)
    FileExplorerModal.tsx backend-driven filesystem browser (drives/dirs/files)
    PreprocessorModal.tsx STL → JSON preprocessing (single file + folder)
    PartTaggerModal.tsx   browse/visualize/tag all zone parts (FIXED/MOBILE/FLEXIBLE)
    PartSummary.tsx       study-part metadata + AP type (Fixed/Moving)
    ExpansionPanel.tsx    Per% / MinDis / MaxDis steppers → ΔX/ΔY/ΔZ
    NeighbourList.tsx     neighbour checklist + per-neighbour type select + filter
    ActionRow.tsx         3D-AP / 3D-SP / 3D-ENV / CC buttons
    ClearanceTable.tsx    results table + capture / analyze / report controls
    Viewer3D.tsx          Three.js viewer + AI multi-view capture (imperative handle)
    Panel.tsx, TypeBadge.tsx, SectionLabel.tsx, ui/input.tsx
  hooks/
    use-mobile.tsx        useIsMobile() (matchMedia < 768px)
  lib/
    api.ts                all backend/MG2 fetch functions
    types.ts              Part, Zone, ClearanceRow, ...
    neighbours.ts         client-side AABB neighbour detection (findNeighbours)
    stl.ts                binary + ASCII STL parser → THREE.BufferGeometry
    export.ts             client-side HTML/JSON report builder (buildReportHtml)
    utils.ts              cn() (clsx + tailwind-merge)
  assets/
    mg2-logo-placeholder.png
```

State management is entirely local `useState`/`useMemo`/`useRef` inside `Index` - no Redux/Zustand;
`@tanstack/react-query` is a dependency but not used.

## 4. The main page (`routes/index.tsx`)

Two-column layout (`420px | 1fr`, min width 1280) under `AppHeader` / `AppFooter`.

- **Left column:** Working Zone (`ZoneSelector` + Tag Parts), part search (client-side filter over
  `zoneParts`, top 20), then `PartSummary` + `ExpansionPanel` once a study part is chosen.
- **Right column:** Detected Neighbours (`NeighbourList`), `ActionRow`, CC progress bar,
  Clearance Results (`ClearanceTable`), the `Viewer3D` panel, and a transient toast (6 s).

Key state: `zone`, `zoneParts`, `studyPart`, `studyPartType` (Fixed/Moving), `neighbourTypes`
(Record → Fixed/Moving/Connected), `params` ({per:10,minDis:20,maxDis:150}),
`neighbours` (memoized `findNeighbours`), `selected` (Set), `clearanceRows`, `analysisResults`,
`validatedScenarios`, `capturedImages`, viewer state (`viewerOpen`, `viewerMode`, `viewerRef`).

### Handlers

- **`handleCalculateClearance`** → `calculateClearance()` streams NDJSON, updates `ccProgress`, sets `clearanceRows`.
- **`handleAnalyzeImage(neighbourName)`** (the **Analyze** button):
  1. Requires a study part + viewer open in **ENV** mode.
  2. `viewerRef.current.captureSegmentAI(name, 2400, 1600)` → base64 composite.
  3. `analyzeClearanceImage({study_part, neighbour_part, image_base64, gap_mm, interference, study_part_type, neighbour_part_type})` → MG2 :8001.
  4. Stores result in `analysisResults[name]`, toasts the verdict, downloads the `report_url` HTML if present.
- **`handleAiAnalysis`** (batch) → captures all rows, `analyzeClearanceBatch({api_type:"capgemini", include_ok:false})`.
- **`handleVerifyPenetration`** → `refinePenetration()`, updates the row in place.
- **`handleGenerateReport`** → builds `clearances` + `validated_scenarios`, calls `generateFinalReport()`, downloads HTML.
- **View modes:** `openViewerAP` (study only), `openViewerSP` (neighbours only), `openViewerENV` (both + clearance rows).
- **Capture / export:** `handleCaptureRow` (one PNG), `handleCaptureAll` (JSZip; all or clashes-only).

## 5. `lib/api.ts`

Base URL resolution (SSR-safe): if hostname is `localhost`/`127.0.0.1` → `http://127.0.0.1:<port>`,
else `http://<current-hostname>:<port>` (API assumed on the same host as the frontend).

- `API_BASE` → port **8000** (main backend, exported).
- `MG2_API_BASE` → port **8001** (module-private).

| Backend (8000) | MG2 (8001) |
|----------------|------------|
| `stlUrl`, `stlProxyUrl`, `isAbsoluteFolderPath` | `analyzeClearanceImage` → `/analyze` |
| `browseFolder` / `browseFile` / `browseStlFile` | `analyzeClearanceBatch` → `/analyze-batch` |
| `loadCustomZone` → `/load-zone` | `checkMG2Service` → `/health` |
| `getZones`, `getZoneParts` | `generateFinalReport` → `/generate-final-report` |
| `calculateClearance` (streaming NDJSON) | |
| `preprocessSingle`, `preprocessFolder` | |
| `refinePenetration`, `performAiAnalysis`, `listDirectory` | |

`analyzeClearanceImage` swallows errors and returns a `verdict:"UNKNOWN"` object rather than throwing.

## 6. The 3D viewer (`components/Viewer3D.tsx`)

A `forwardRef` component exposing `Viewer3DHandle` (`captureSegmentAI`, `isReady`). Raw Three.js.

- **Scene:** grid, ambient + key/fill/rim lights, `PerspectiveCamera(45°)`, `WebGLRenderer` with PCF
  shadows, `OrbitControls`, a corner XYZ gizmo, theme-reactive background (`MutationObserver` on `data-theme`).
- **STL loading:** `resolveStlUrl` uses `stlProxyUrl` for absolute folder paths, else `stlUrl`;
  fetches + parses with `lib/stl.ts`; caches parsed geometries in a module-level Map. Optimized mode
  caps at 50 parts. Study part = `#4f7cff`; neighbours cycle 5 colors.
- **Interaction:** raycast hover tooltip + click-select; Focus / Isolate / Hide / Show / Clear;
  "Find part…" search (pulses magenta, focuses camera).
- **Clearance visualization (ENV):** a `segGroup` draws gap lines (red ≤0 / orange <5 / green else)
  with endpoint spheres, and for interference red contact spheres + a penetration `ArrowHelper`.
  Clicking a table row highlights its segment and dims the rest.
- **Toolbar:** reset, plan views (XY/YZ/XZ), ±90° rotations, grid toggle, screenshot-to-clipboard, close.

### `captureSegmentAI(name, width=2400, height=1600)` - the AI composite

Isolates study + one neighbour, then builds a **4-panel 2400×1600 canvas** (header ~7% + 2×2 panels
+ footer ~7%): (1) view ⊥ to the gap/penetration axis, (2) lateral ⊥ view, (3) along-axis
cross-section, (4) context view in the user's orientation. Panels 1-2 project `point_a`/`point_b`
to 2D and draw a **green CAD dimension line** with arrowheads + a "Distance minimale ~X.Xmm" (or
"Pénétr. ~X.Xmm") label. Header shows part names + mm value (or a red INTERFERENCE banner); footer
shows world coords + direction. Returns a PNG data URL; visibility is restored in `finally`.

The resulting composite that is sent to the vision model looks like this:

![The 2400x1600 multi-view composite image sent to the vision LLM](image_sent_to_llm.png)

## 7. The clearance table (`components/ClearanceTable.tsx`)

- Rows sorted: interfering first (deepest penetration), then ascending clearance, errors last.
- Per row: neighbour name, `AI: <verdict>` badge (if analyzed), type badge, rule label + evaluation,
  and the value (`−X.XXX mm INTERFERENCE` in red / `X.XXX mm` / `NOT ATTACHED` / error).
  Row click highlights the 3D segment.
- Header buttons: `ZIP All (AI)`, `ZIP Clashes (n)`, `Analyze Selected` (batch, Capgemini),
  `Generate Report (n)`, `Copy as CSV`.
- Per-row buttons (`CamBtn`): `CamViewAI` / `CamClash`, `Verify` (interference only), `Analyze`
  (→ `handleAnalyzeImage`). All gated on `captureReady` (viewer open in ENV + ready) and `hasSegment`.
- AI expansion panel: verdict, problem explanation, visual observations, and a grid of corrective
  **scenarios** (name, action, movement, timeline, cost, est. new clearance, complexity), each with
  an "Approve Scenario" toggle.

> **Note:** the **CC (compute)** button is in `ActionRow.tsx`, not the table.

## 8. Client-side report builder (`lib/export.ts`)

`buildReportHtml` / `downloadHtmlReport` produce a self-contained printable HTML report from an
`AnalysisReport` (verdict banner, engine summary, study-part table, clearance table, AI diagnosis).
This is a *client-side* path distinct from the MG2 server-generated reports; the primary flow uses
the server reports (`generateFinalReport`).
</content>
