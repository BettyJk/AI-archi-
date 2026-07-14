# Clearance Rules Reference

The packaging clearance rules used to classify a measured gap into
`OK / WARNING / NOK (VIOLATION) / CLASH (COLLISION)`. Based on the automotive packaging
"CCP Routing" clearance standard (ref. IFIM_SV_IMN_0070).

> **Where the rules actually live.** There is a human-readable reference spec at
> `peugeot_preprocessor2/packaging_rules.json`, but
> **no code imports it**. The operative thresholds are hard-coded in two places:
> 1. `MG2_NOK_Correction_System_fixed/pipeline/classifier.py` → `CLEARANCE_RULES` (deterministic verdict).
> 2. `MG2_NOK_Correction_System_fixed/pipeline/ai_analyst.py` → `CCP_SYSTEM_PROMPT` (the rules the LLM applies).
> `peugeot_preprocessor2/engine.py` has its own copy of `CLEARANCE_RULES` for the CLI.
> The tables below reconcile all of them.

## 1. Verdict / status thresholds

Given a measured clearance `x` and a required minimum `min`:

| Verdict (code) | Status (spec) | Condition | Color |
|----------------|---------------|-----------|-------|
| **CLASH** | COLLISION | `x ≤ 0` (interference) | red |
| **NOK** | VIOLATION | `0 < x < min` | red |
| **WARNING** | WARNING | `min ≤ x < min + 3` (code uses `x < target`) | orange |
| **OK** | OK | `x ≥ min + 3` (code: `x ≥ target`) | green |

For **Connected** parts (attached by design): `|x| < 0.01` → OK, `x < 0` → CLASH, else NOK.
Missing gap → **UNKNOWN**.

## 2. Structural clearance rules (SCR) - by part motion

Selected when explicit part types are provided (Fixed / Moving / Connected):

| Rule | Pair | Minimum |
|------|------|---------|
| SCR-01 | Fixed ↔ Fixed | **5 mm** |
| SCR-02 | Fixed ↔ Moving | **10 mm** |
| SCR-03 | Moving ↔ Moving | **15 mm** |
| SCR-04 | Installed **with** guide | 10 mm |
| SCR-05 | Installed **without** guide | 15 mm |
| SCR-06 | Connected (attached by design) | **0 mm** |

These match the frontend's `getRowEvaluation` limits (Fixed/Fixed 5, Moving/Moving 15, mixed 10,
Connected 0).

## 3. CCP circuit routing rules - by circuit type

Applied when a part is a routed circuit (fuel line, harness, etc.). The governing minimum is the
**highest** triggered value among base + special conditions.

| CCP | Circuit | Base min | Special conditions |
|-----|---------|----------|--------------------|
| CCP-01 | Fuel (Carburant) | **25 mm** | near heat 50; near moving 30 |
| CCP-02 | Cooling (Refroidissement) | **15 mm** | near electrical 20 |
| CCP-03 | Air / Pneumatic | **10 mm** | flexible duct near moving 15 |
| CCP-04 | Air Conditioning (Climatisation) | **15 mm** | HP line near structure 20; near heat 30 |
| CCP-05 | Hydraulic / Brake | **15 mm** | near heat 25; near moving suspension/steering 20 (safety-critical) |
| CCP-06 | Cable Controls (Commandes par câbles) | **10 mm** | under tension near sharp edge 15; near moving 15 |
| CCP-07 | Wiring Harness (Faisceau) | **6 mm** | near heat 20; near moving 12; near sharp edge/bracket 10 |

High-voltage (HV) cables/supports → treated as CCP-07 with a high-voltage flag.

## 4. Code `CLEARANCE_RULES` (keyword-based, deterministic)

Used by `classifier.py` / `engine.py` when explicit part types aren't given. The part name is
keyword-matched to a type:

| Type | min / target (mm) | Criticality |
|------|-------------------|-------------|
| `hv_cable` | 5 / 10 | CRITICAL |
| `pipe` | 3 / 5 | MEDIUM |
| `harness` | 3 / 5 | MEDIUM |
| `moving` | 10 / 15 | HIGH |
| `bracket` | 1 / 3 | LOW |
| `default` | 5 / 10 | STANDARD |

Type detection uses `PART_TYPE_KEYWORDS` (mixed French/English, e.g. `porte`, `capot`, `faisceau`,
`pipe`, `bracket`) and `HV_KEYWORDS` in `classifier.py`.

## 5. LLM output gating (from `CCP_SYSTEM_PROMPT`)

The AI analyst is instructed to:
- Extract data from the 2×2 CATIA viewport composite image.
- Classify circuit type (CCP-01…07) and motion (Fixed/Moving/Connected).
- Apply the governing rule (highest triggered minimum across SCR + CCP conditions).
- **Gate the output:** `OK` / `FASTENER` (contact by design) → a one-line entry; `WARNING`,
  `VIOLATION`, `COLLISION` → a full report with 3 corrective scenarios.
- Return a `ccp_report_entry` plus structured JSON (verdict, failure-mode analysis, scenarios with
  which part to move / axis / direction / mm, cost, timeline, complexity, recommended scenario).

## 6. Process context (from the spec)

`packaging_rules.json` also documents the manual process this platform replaces: weekly Hot Housing
(HH) and Zone (RZ) meetings, roles (TI = Technicien Implantation, SCIL/SIDL teams), and the tools
being automated (manual CATIA gap measurement, Excel/PowerPoint reporting, manual CCP abacus
cross-referencing).

> **Consistency note.** The reference spec uses COLLISION/VIOLATION/WARNING/OK; the code uses
> CLASH/NOK/WARNING/OK. They map 1:1 as in §1. Keep both in sync manually - the JSON is not loaded
> at runtime.
</content>
