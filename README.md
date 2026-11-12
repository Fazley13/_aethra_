# AETHRA — Astronaut Health Monitoring & Decision Support System
*Beyond Earth, Beyond Limits.*

> **Research prototype.** AETHRA is not a certified medical device, is **not endorsed by or affiliated with NASA** or any space agency, does not diagnose, and does not replace a flight surgeon. General adult screening guidance and NASA OSD source intervals are not mission-approved spaceflight thresholds. All demo data is simulated.

AETHRA (formerly "ASTRA MED") is an offline-first Next.js app that compares an astronaut's readings with their **own baseline**, explains every flag, tracks the five NASA spaceflight hazards, and guides a local emergency report. No API keys or cloud services are needed. The supplied NASA OSD Inspiration4 files are bundled as a read-only application data source and loaded by the shared app provider.

## What it does
- **Status model:** GREEN, AMBER, RED and **UNKNOWN** (missing, stale or insufficient data — never treated as healthy). Hybrid status combines the existing vital-sign rules with recent, compatible laboratory findings; laboratory out-of-range values can raise AMBER but do not independently create RED/critical status.
- **Explainable assessment:** every finding shows the value, personal baseline, deviation, timestamp, source and rule that fired; combined patterns are reported once, not double-counted. "Data completeness" (not "AI confidence") describes how much recent data exists. Resting HR, BP and SpO₂ use general adult screening references alongside personal baselines; these are not validated for microgravity or a specific mission.
- **Pages:** Mission Overview · Health Dashboard · Crew Overview · Health Check · Health Assessment · Health History · Body Explorer · Space Hazards · Mental Wellness · Emergency Center · Medical Reports · Scientific Resources · Settings.
- **Space Hazards:** radiation, isolation, distance (communication delay), gravity (exercise) and cabin environment, each labelled SIMULATED / MANUAL / NO DATA through pluggable adapters (`lib/adapters.ts`).
- **Emergency Center:** reports are built only from the affected person's own data (otherwise `NOT AVAILABLE`). Nothing is transmitted.
- **Reports:** PDF, JSON, CSV (RFC 4180 escaping, formula-injection guard).
- **Offline/PWA:** IndexedDB persistence with migration from the old `astra-med` database, service worker (versioned cache), backup/restore.

## Develop
```bash
npm install
npm run dev          # http://localhost:3000
npm run typecheck && npm run lint && npm test
npm run build && npm start
```
`npm test` runs `scripts/test-engine.ts` (existing engine, persistence and migration checks plus hybrid lab interval/unit/staleness checks).

## Deploy to Vercel
Import the repo (framework: Next.js, build `npm run build`). No environment variables. Open the deployed site once online so the service worker can cache all pages.

## Structure
```
app/         routes (one folder per page)         components/  Shell, ui kit, charts, forms, LabPanel
lib/         assess.ts rules.ts labs.ts validate.ts csv.ts persist.ts store.tsx
             adapters.ts isolation.ts emergency.ts export.ts access.ts content.ts demo.ts
public/      sw.js, manifest, icon                 scripts/     test-engine.ts
```

## Data & privacy
Data lives in this browser's IndexedDB, **unencrypted**, and is not synced anywhere. "Online" does not mean saved to a server. `lib/access.ts` only hides data in the UI; real deployments need server-side auth, RBAC and encryption.

## Known limitations
No real sensors, NASA feeds or ground communication. Mission-specific thresholds, HR urgent cut-offs, temperature/glucose and wellbeing heuristics, trend rules, the isolation indicator and emergency wording are unvalidated prototype choices. Offline behaviour and UI rendering have not been verified in a real browser by the authors of this build. `npm audit` reports advisories in Next's bundled PostCSS that only a breaking Next 16 upgrade resolves.

## Hybrid assessment reference notes
- Blood pressure categories: American Heart Association / ACC 2025 adult guidance (normal <120/<80; elevated 120–129 and <80; stage 1 130–139 or 80–89; stage 2 ≥140 or ≥90). A single reading is not a diagnosis; guidance recommends averaging careful measurements on separate occasions. https://www.heart.org/en/health-topics/high-blood-pressure/understanding-blood-pressure-readings
- Resting adult pulse and vital-sign context: MedlinePlus. https://medlineplus.gov/ency/article/002341.htm
- Pulse oximeter context and limitations: US FDA. https://www.fda.gov/consumers/consumer-updates/pulse-oximeter-basics
- The vital-sign thresholds remain general adult screening guidance only. Spaceflight-specific thresholds and alerting logic require mission flight-surgeon approval.

## References
See the in-app **Scientific Resources** page (DOIs for Patel et al. 2020, Stepanek et al. 2019, Afshinnekoo et al. 2020, Zeitlin et al. 2013, plus NASA HRP pages) with verification notes.

## UI design — "Flight Deck" theme
The interface uses a single dark, low-glare palette suited to dim cabin lighting. All colours are CSS variables in `app/globals.css` (exposed to Tailwind in `tailwind.config.ts`), so the whole theme can be re-tuned in one place; printing automatically switches to a light, ink-friendly palette. Status is never conveyed by colour alone (icon + text labels are unchanged). No features or logic were changed by the redesign.

## Relax & recharge
- **Earth Games** (`/relax`): Earth Memory Match, Home Planet Quiz, Tiny Garden and an Earth Soundscape mixer (rain, ocean, wind, birdsong — synthesised in-browser, works offline). No timers or failure states.
- **Canvas Studio** (`/studio`): a whiteboard with pen, marker, eraser, shapes, text, fill bucket, undo/redo, colouring pages, PNG download and a local gallery. Drawings stay on the device (localStorage) and are never added to reports.


## NASA OSD backend data integration
- Original OSD-569, OSD-571, OSD-575 and OSD-656 CSV/XLSX files plus their investigation/sample metadata are bundled under `public/data/osd/`.
- `public/data/osd/records.json` is the normalized, browser-ready data layer loaded once by `Provider` in `lib/store.tsx`. Existing application screens consume it through the shared `osdRecords` context; there is no standalone NASA research page.
- Mission Overview, Dashboard, Crew Overview, Health Assessment, Body Explorer, Space Hazards, Health History and Medical Reports use the relevant participant-level lab records. History exports include the selected OSD participant data; generated reports include a clearly separated OSD research context snapshot.
- The shared participant selector uses the dataset's anonymized IDs C001–C004. These IDs are not mapped to AETHRA's fictional demo astronaut profiles.
- OSD-571 workbook sheets are group-level differential analyses; they are explicitly labelled cohort findings and are never attributed to an individual participant.
- The source files do not provide continuous heart rate, blood pressure, SpO₂, body temperature or spacecraft sensor telemetry. The integration does not fabricate these fields or use laboratory values as substitutes. `reference-ranges.json` extracts 27 source-supplied CBC/CMP intervals from OSD-569 and OSD-575; the app compares astronaut-entered lab results only when analyte and unit match exactly. These intervals are study-file reference intervals, not astronaut-specific or mission-approved thresholds. Hemoglobin is excluded because the supplied transformed file labels its unit as percent despite a conventional-looking numeric interval; it requires source clarification before use.
- The service worker caches `records.json` and `reference-ranges.json` for offline use. New lab results are stored per AETHRA profile in the existing browser IndexedDB and included in history/report exports.
- **Hybrid workflow:** enter vital signs in Health Check as before, then add CBC/CMP laboratory results in the new laboratory-result panel on the same page. Each profile stores its own results. The latest result for each analyte is assessed; results older than 90 days are marked stale and excluded from the current combined status. A matching low/high result raises overall status to AMBER; NASA reference intervals alone never create RED/critical status.
- The 27 enabled intervals comprise 17 consistent OSD-569 CBC ranges and 10 consistent OSD-575 CMP ranges. Sex-varying/ambiguous intervals and the hemoglobin field with a source unit-label ambiguity are intentionally excluded. OSD-571 cohort statistics and OSD-575/656 protein-abundance assays without clinical intervals remain research context only.
- **Architecture limitation:** this is still a static JSON data source served from `public/` plus per-browser IndexedDB, not a server-side database/API backend. Manual entries recalculate immediately after save; no live sensor stream is connected.


## UI usability update
- Reduced the astronaut/Earth background image visibility with a stronger overlay so page content is easier to read.
- Increased card and top-bar opacity and softened visual decoration.
- Simplified the sidebar into three clear groups: Mission, Health, and Support & Tools. All existing pages and features remain accessible.
- Tightened page-header, card-heading, and navigation spacing to reduce visual clutter without hiding health data or safety information.

## Run locally
```bash
npm ci
npm run dev
```

## UI simplification update (November 2026)

- Long measurement instructions, health explainers, body-system education, hazard details, and privacy/deployment notes are now tucked into clearly labelled expandable sections so the primary action and current status are easier to find.
- Repeated NASA OSD research and laboratory panels are now optional expandable sections across dashboard, crew, health check, assessment, history, body explorer, hazards, and reports. The underlying data and tools remain available.
- Relax & Recharge now includes two additional offline mini-games: Reaction Test and Pattern Recall.
- Existing health monitoring, records, reporting, emergency guidance, NASA OSD data and other app functionality are retained.

This remains a research/demo prototype. Simulated values are not live spacecraft telemetry and the interface is not a substitute for mission medical procedures.
