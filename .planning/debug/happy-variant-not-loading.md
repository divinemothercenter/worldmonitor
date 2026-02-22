---
status: diagnosed
trigger: "Happy variant not loading -- user navigates to what should be the happy variant but sees the default WorldMonitor"
created: 2026-02-23T00:00:00Z
updated: 2026-02-23T00:00:00Z
---

## Current Focus

hypothesis: Multiple gaps across the stack prevent the happy variant from presenting a distinct experience
test: Traced variant detection -> config -> panels -> rendering pipeline
expecting: Several missing artifacts and wiring issues
next_action: Return structured diagnosis

## Symptoms

expected: Happy variant with warm theme, curated positive news, unique panels (progress, counters, spotlight, breakthroughs)
actual: Default WorldMonitor with military layers, DEFCON indicator, Bloomberg live news
errors: N/A (no crash, just wrong variant experience)
reproduction: Set VITE_VARIANT=happy in .env.local, run dev server, visit localhost:3000
started: Always — happy variant UI was never fully wired

## Eliminated

(none needed — root cause identified on first pass)

## Evidence

- timestamp: 2026-02-23
  checked: .env.local line 115
  found: VITE_VARIANT=happy is correctly set
  implication: Variant detection works at config level

- timestamp: 2026-02-23
  checked: src/config/variant.ts
  found: SITE_VARIANT correctly reads VITE_VARIANT, 'happy' is in allowed list
  implication: SITE_VARIANT resolves to 'happy' in dev

- timestamp: 2026-02-23
  checked: src/config/panels.ts lines 374-470
  found: HAPPY_PANELS, HAPPY_MAP_LAYERS, HAPPY_MOBILE_MAP_LAYERS all defined and wired into exports
  implication: Panel config is properly variant-aware

- timestamp: 2026-02-23
  checked: src/config/feeds.ts lines 945-973
  found: HAPPY_FEEDS defined with 5 categories (positive, science, nature, health, inspiring), FEEDS export switches correctly
  implication: Feed config is properly variant-aware

- timestamp: 2026-02-23
  checked: src/main.ts lines 131-133
  found: data-variant='happy' is set on documentElement when SITE_VARIANT !== 'full'
  implication: CSS variant attribute is applied

- timestamp: 2026-02-23
  checked: src/styles/happy-theme.css
  found: Exists and is imported in main.ts line 2
  implication: CSS theme loads for all builds

- timestamp: 2026-02-23
  checked: index.html line 95
  found: Inline script detects happy. from hostname and also reads localStorage; sets data-variant and forces light theme
  implication: Pre-paint variant detection works

- timestamp: 2026-02-23
  checked: public/map-styles/happy-light.json, happy-dark.json
  found: Both exist
  implication: Map basemap styles are ready

- timestamp: 2026-02-23
  checked: src/components/DeckGLMap.ts lines 140-144
  found: Happy variant map styles are wired in
  implication: Map renders correct basemap

- timestamp: 2026-02-23
  checked: src/App.ts line 696
  found: DEFCON/PizzInt indicator NOT excluded for happy variant (only excludes tech/finance)
  implication: BUG — DEFCON indicator shows on happy variant

- timestamp: 2026-02-23
  checked: src/App.ts lines 1835-1862
  found: Variant switcher in header has NO happy option (only full/tech/finance)
  implication: GAP — no UI way to identify or switch to happy variant

- timestamp: 2026-02-23
  checked: src/components/LiveNewsPanel.ts lines 52-74
  found: LIVE_CHANNELS only handles 'tech' vs default (full). No happy-specific channels. Happy gets Bloomberg/SkyNews/military news streams
  implication: BUG — Live news shows Bloomberg and war news channels instead of positive content

- timestamp: 2026-02-23
  checked: src/App.ts panel creation (lines 2160-2410)
  found: No happy-specific panel components exist. The HAPPY_PANELS config defines panels like 'progress', 'counters', 'spotlight', 'breakthroughs' but NO component classes exist for them
  implication: CRITICAL BUG — happy-specific panels are configured but never created. Only 'live-news' and generic FEEDS-derived panels would render

- timestamp: 2026-02-23
  checked: Glob for src/components/*Progress*, *Counter*, *Spotlight*, *Breakthrough*
  found: Zero files. No component implementations
  implication: CRITICAL GAP — The panel components needed for the happy variant were never built

- timestamp: 2026-02-23
  checked: src/App.ts lines 2273-2287 (dynamic panel creation from FEEDS keys)
  found: App dynamically creates NewsPanel instances for each FEEDS category key. HAPPY_FEEDS has: positive, science, nature, health, inspiring. These will get NewsPanel instances.
  implication: The RSS-based news panels DO work — positive/science/nature/health/inspiring will render as generic NewsPanel instances

- timestamp: 2026-02-23
  checked: src/App.ts loadAllData (lines 3105-3159)
  found: loadAllData always loads markets, predictions, pizzint, fred, oil, spending for ALL variants. No happy-specific data loading exclusions.
  implication: GAP — Happy variant loads unnecessary geopolitical/financial data

- timestamp: 2026-02-23
  checked: src/services/positive-classifier.ts
  found: Fully implemented classifier with 6 categories and keyword matching
  implication: Content classification is ready

- timestamp: 2026-02-23
  checked: src/App.ts line 3384
  found: classifyNewsItem IS called for happy variant items
  implication: Items get tagged with happyCategory but nothing uses the tags in rendering

## Resolution

root_cause: |
  The happy variant has a SPLIT implementation state:

  WORKING (Phase 1 visual + Phase 2 pipeline):
  - Variant detection (SITE_VARIANT, data-variant attribute, hostname detection)
  - CSS theme (happy-theme.css loaded, skeleton styles, warm palette)
  - Map basemap styles (happy-light.json, happy-dark.json wired in DeckGLMap)
  - Panel/map-layer config (HAPPY_PANELS, HAPPY_MAP_LAYERS in panels.ts)
  - Feed config (HAPPY_FEEDS with positive-news RSS sources in feeds.ts)
  - Content classifier (positive-classifier.ts classifies items into 6 categories)
  - Dynamic news panels (positive, science, nature, health, inspiring categories render as generic NewsPanels via the FEEDS dynamic loop)

  NOT WORKING (Phase 3 integration gaps):
  1. DEFCON/PizzInt indicator NOT excluded for happy (App.ts:696 only checks tech/finance)
  2. LiveNewsPanel shows Bloomberg/war channels — no happy-specific live channels
  3. Variant switcher header has no happy option
  4. Happy-specific panels (progress, counters, spotlight, breakthroughs) have NO component implementations
  5. classifyNewsItem tags items but rendering never uses happyCategory for display
  6. loadAllData loads geopolitical data (markets, predictions, PizzInt, FRED, oil) even for happy variant
  7. The happy variant falls through to default behavior in most App.ts conditionals

fix: (not applied — research only)
verification: (not applied — research only)
files_changed: []
