# Narmada Daily Monsoon Report — Feature Additions

**Date:** 2026-07-10
**Target file:** `Narmada_Daily_Monsoon_Report.html` (single self-contained HTML file)
**Status:** Approved design → ready for implementation plan

## Context

The District Collector's office, Narmada (Rajpipla) publishes a *Daily Monsoon
Situation Report*. `Narmada_Daily_Monsoon_Report.html` is a single, fully
offline, self-contained tool used to produce it: every figure is an editable
input; KPI tiles, rainfall bars, and reservoir gauges recompute live via
`recompute()`; data is carried day-to-day by exporting/importing a JSON
snapshot; and the report is finalised by printing to PDF for the Collector.

This spec adds four capabilities without changing that architecture. When the
new features are not in use, the plain daily report and its PDF must look
exactly as they do today.

### Existing architecture (do not break)

- **State lives in DOM inputs**, each with a stable `id` (e.g. `rf0p`, `rf0t`,
  `rf0l`, `dgD0`, `dgC0`, `dm0_0`, `rd0`, `kEvac`, narrative `n7`…`n14`).
- Seed arrays `RF`, `DAMS`, `DMG`, `ROADS` build the editable tables once on load.
- `recompute()` runs on every edit: derives cumulatives, totals, bar widths,
  gauge fills/colours, KPI tiles, status band, and the auto-drafted executive
  summary (unless the user has edited it — tracked by `execSummary.dataset.touched`).
- `snapshot()` serialises every `input/textarea/select` by `id`;
  `exportJSON()` / `importJSON()` save & restore a day.
- Print CSS (`@media print`) strips the toolbar/hints and flattens inputs.

## Requirements

### 1. Comparison mode — compare an arbitrary past date against the current report

**Scope:** all data sections — rainfall, reservoirs/river, damage & compensation,
roads, and the KPI tiles.

**Activation & data source:**
- A **`Compare` toggle** in the toolbar adds/removes the class `comparing` on
  `<body>`. All comparison UI is styled `display:none` unless `body.comparing`,
  so it appears on screen **and** in the printed PDF only while the toggle is on.
- A **`⭱ Load comparison report`** button (file input, accepts the tool's own
  saved JSON) reads a previously-saved report into a JS object `CMP` with the
  **same schema as a normal snapshot**. This makes any day ever saved a valid
  comparison baseline.
- A **Comparison-date field** (`cmpDate`) shown in a slim compare banner under
  the masthead. Auto-filled from the loaded file's `dateTo`; also editable by
  hand for last-year figures that were never saved digitally.
- Comparison values are **editable** in every section (prefilled when a file is
  loaded, or typed manually).

**Comparison metric & change display per section:**

| Section | Baseline metric (comparison value) | Change shown |
|---|---|---|
| Rainfall | Cumulative (mm) per taluka | ▲/▼ +abs mm (+/−%) |
| Reservoirs | Fill % (level ÷ danger) per dam | ▲/▼ +abs pts |
| Damage & compensation | Total cases per category | ▲/▼ +abs (+/−%) |
| Roads | Roads blocked per category | ▲/▼ +abs |
| KPI tiles | The derived KPI value | small ▲/▼ line under the tile |

- Change is rendered as **absolute value + percentage** (percentage omitted
  where it is meaningless, e.g. baseline = 0, or where a points delta is more
  natural such as reservoir %). Direction shown with **▲ / ▼**, coloured
  green (down/safer or neutral-good) / red (up/worse) per section semantics —
  see "Delta colour semantics" below.
- Tables gain two `cmp-only` columns: an **editable comparison input** and a
  computed **Change** cell.
- Reservoir gauges gain a comparison-level input + a delta line under the tank.
- The rainfall bar's existing "last year" tick is repurposed to mark the
  **comparison cumulative** on the same scale.
- KPI tiles gain a small delta line under the sub-text.

**Delta colour semantics:** rising water/damage/roads = red (worse), falling =
green (better). This matches the report's existing alert palette
(`--alert` red, `--normal` green). A zero change is muted/neutral.

**Data model:**
- Rainfall comparison baseline per taluka = **cumulative season total**
  (confirmed default, not the single-day value). Comparison input id: `cmpRf<i>`.
  - When a file is loaded, prefilled as `CMP['rf'+i+'p'] + CMP['rf'+i+'t']`.
- Reservoir comparison baseline per dam = **current level (m)** at the
  comparison date. Input id `cmpDg<i>`; % derived vs the live danger level;
  delta shown in percentage points (and metres in the gauge line).
  - Prefilled from `CMP['dgC'+i]`.
- Damage comparison per category = **total cases**. Input id `cmpDm<i>`.
  Prefilled from `CMP['dm'+i+'_0']`.
- Roads comparison per category = **blocked count**. Input id `cmpRd<i>`.
  Prefilled from `CMP['rd'+i]`.
- **KPI deltas** are computed by running the KPI derivation over the comparison
  dataset. When a full report file is loaded, `CMP` contains every raw field so
  KPI deltas are exact. With only hand-typed table figures, KPI deltas are
  computed from the comparison values that exist and omitted where the
  underlying comparison figures are absent (documented limitation, accepted).

**Toggle behaviour:** toggling triggers `recompute()` so deltas render/clear.
When off, comparison inputs retain their values (hidden) so re-enabling is
lossless.

### 2. Reservoir thresholds → 98 / 75

- Recolour logic (replaces current 90/75 split):
  - **≥ 98 %** → red, status word **ALERT** (`--alert`)
  - **75 – 97 %** → amber, status word **WATCH** (`--watch`)
  - **< 75 %** → green, status word **NORMAL** (`--normal`)
  - Implemented by editing `gaugeColor(pct)` and `gaugeWord(pct)`.
- The dashed **red danger marker (`.danger`) moves to the 98 % line** of each
  tank (currently pinned to the top / 100 %). A fill crossing that dashed line
  visually reads red, consistent with the status word.
- The "Reservoirs on Watch (≥75 %)" KPI tile is unchanged (75 % remains the
  amber entry point).

### 3. Empty free-text fields drop out of the PDF

- Applies to every **free-text (narrative) field**: Executive Summary (`§1`,
  `execSummary`), Rescue & Relief (`§6`, `reliefNote`), and `§§7–14`
  (`n7`…`n14`). Numeric/table inputs are **not** affected (they default to 0).
- On `window.onbeforeprint`: any narrative textarea whose trimmed value is empty
  is hidden **together with its section heading**, so no orphaned heading prints.
  `window.onafterprint` reverses it, leaving the on-screen editor intact.
  - Full-width sections (`§1`, `§6`): hide the whole `.section`.
  - Grid items (`§§7–14`): hide the individual block (heading + textarea); hide
    the whole grid section only if **all** eight are empty.
- Mechanism: JS toggles a `print-hidden` class (CSS `display:none`) rather than
  relying on CSS alone, because a textarea's live value is not reflected to any
  attribute CSS can match.

### 4. Media clippings — attach multiple images, auto-arranged by dimensions

**Placement:** a **new full-width "Media Clippings" section** appended after
`§14`, before the footer. No per-image captions.

**Adding images:**
- File picker (`<input type="file" accept="image/*" multiple>`) **and**
  drag-and-drop onto the section.
- Each image is read as a **base64 data-URI** (`FileReader.readAsDataURL`) so
  the report stays fully offline and self-contained.
- Images held in a JS array `CLIPS` (array of data-URI strings). Each rendered
  image has a **screen-only `×` remove button** (`no-print`).

**Auto-layout (adjusts to each image's width/height):**
- **CSS masonry columns**: a container with `column-width: ~220px`,
  `column-gap`, and each image at `width:100%; height:auto; display:block`.
  Every clipping keeps its true aspect ratio, so tall portraits stay tall and
  wide ones stay short — placement follows each image's real dimensions with no
  cropping and no distortion. Text in clippings stays readable.
- Each image wrapped in a figure with `break-inside: avoid` for clean pagination.

**Print:** images print (data-URIs need no network). The section is **hidden
from the PDF when `CLIPS` is empty** (same empty-hiding principle as #3).

**Persistence:**
- `snapshot()` / `exportJSON()` include `CLIPS` (as its own key, not via input
  ids). `importJSON()` ("Load yesterday") restores `CLIPS`.
- **Loading a *comparison* report must not overwrite the current `CLIPS`** —
  comparison load only populates `CMP` and comparison inputs, never the live
  clippings.
- Data-URIs inflate the JSON; acceptable given the offline, self-contained
  requirement. (No hard size cap in v1.)

## Non-goals / out of scope

- No server, database, or network calls — the tool stays 100 % offline.
- No image editing/cropping/rotation; no captions (per decision).
- No automatic fetch of last-year data — comparison is load-a-file or hand-key.
- No change to the report's sections, wording, or bilingual labels beyond the
  additions above.
- No change to the "Reservoirs on Watch (≥75 %)" KPI definition.

## Confirmed defaults

- Rainfall comparison metric = **cumulative** season total per taluka.
- KPI comparison deltas are exact **only when a full saved report is loaded**;
  partial with hand-typed data. Accepted limitation.

## Acceptance criteria

1. With `Compare` off, the report and its PDF are byte-for-behaviour identical
   to today.
2. Turning `Compare` on and loading a saved JSON shows correct ▲/▼ absolute + %
   changes across rainfall, reservoirs, damage, roads, and KPI tiles, labelled
   with the comparison date; comparison values remain hand-editable.
3. A reservoir at ≥98 % shows red/ALERT with the dashed marker at 98 %;
   75–97 % amber/WATCH; <75 % green/NORMAL.
4. Clearing any narrative textarea removes that block (and its heading) from the
   printed PDF while leaving it editable on screen.
5. Attaching several mixed-orientation images arranges them in masonry columns
   with correct aspect ratios, no cropping; they print; they survive
   export → import; and the section disappears from the PDF when none are
   attached.
6. Loading a comparison report never disturbs the current day's figures or
   attached clippings.
