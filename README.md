# narmada

**Daily Monsoon Situation Report — Narmada**

A single-file, fully offline, self-contained web tool for the Office of the
District Collector, Narmada (Rajpipla) to compile and print the daily monsoon
situation report. Every figure is editable; KPI tiles, rainfall bars and
reservoir gauges recompute live; data carries day-to-day via JSON save/load;
and the report prints straight to a PDF for the Collector.

**Live site:** https://pancham97.github.io/narmada/

## Features

- **Live dashboard** — rainfall, reservoirs/river, damage & compensation,
  roads, rescue & relief and 14 narrative sections, all editable in-browser.
- **Comparison mode** — toggle on to compare against any arbitrary past date
  (load a saved report or type figures by hand) and see the change per line as
  absolute + % with ▲/▼ indicators, across every section and KPI.
- **Reservoir alert bands** — ≥98% red (ALERT), 75–97% amber (WATCH),
  <75% green (NORMAL), with the danger marker at the 98% line.
- **Media clippings** — attach newspaper images; they auto-arrange in a masonry
  layout by aspect ratio, embed into the report, and print with it.
- **PDF-ready** — empty free-text sections drop out of the printed PDF.

## Usage

Open `Narmada_Daily_Monsoon_Report.html` (or the site root) in any modern
browser — no server or internet required once loaded.

1. Edit any figure; everything recomputes instantly.
2. **Save data (JSON)** at the end of the day; **Load yesterday** next morning
   to carry figures forward.
3. **Compare** → optionally **Load comparison report** to diff against a past
   day, then tweak.
4. **Print / Save PDF** to hand to the Collector.
