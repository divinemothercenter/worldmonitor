# App.ts Decomposition Plan — Phase 1 Analysis

## File Stats
- **File**: `src/App.ts`
- **Lines**: 4,597
- **Single class**: `App` with ~60 private methods, ~60 private fields
- **Imports**: 107 lines (lines 1–110)

---

## 1. Dependency Graph of Major Logical Components

```
App (God class)
├── [A] Initialization & Lifecycle
│   ├── constructor()          → reads localStorage, URL state, variant config
│   ├── init()                 → orchestrates startup sequence
│   ├── destroy()              → teardown all intervals, listeners, streams
│   └── setupEventListeners()  → global DOM events, keyboard, visibility, idle
│
├── [B] Layout & Rendering
│   ├── renderLayout()         → generates full HTML template string (~90 lines of HTML)
│   ├── createPanels()         → instantiates ~40 panel components, orders them in grid
│   ├── applyPanelSettings()   → shows/hides panels based on config
│   ├── applyInitialUrlState() → applies URL-parsed view/zoom/layers
│   └── renderCriticalBanner() → military posture warning banner
│
├── [C] Panel Management
│   ├── makeDraggable()        → mouse-based panel reordering
│   ├── handlePanelDragMove()  → DOM reorder during drag
│   ├── savePanelOrder()       → persist to localStorage
│   ├── getSavedPanelOrder()   → read from localStorage
│   ├── scrollToPanel()        → scroll + flash highlight
│   └── attachRelatedAssetHandlers() → news→map asset linking
│
├── [D] Data Loading (largest section, ~1,800 lines)
│   ├── loadAllData()          → orchestrates parallel initial fetch
│   ├── loadDataForLayer()     → on-demand layer activation
│   ├── loadNews()             → RSS feeds → clustering → map hotspots
│   ├── loadNewsCategory()     → single feed category with progressive render
│   ├── loadMarkets()          → stocks, sectors, commodities, crypto
│   ├── loadPredictions()      → Polymarket
│   ├── loadIntelligenceSignals() → protests, military, conflicts, UCDP, displacement, climate
│   ├── loadNatural()          → earthquakes + EONET
│   ├── loadWeatherAlerts()
│   ├── loadAisSignals()       → maritime AIS
│   ├── loadCableActivity/Health() → undersea cables
│   ├── loadProtests()         → ACLED/GDELT (standalone, uses cache)
│   ├── loadMilitary()         → OpenSky flights + vessels (standalone, uses cache)
│   ├── loadFlightDelays()     → FAA
│   ├── loadCyberThreats()
│   ├── loadFirmsData()        → NASA FIRMS satellite fires
│   ├── loadTechEvents()       → conference events
│   ├── loadFredData()         → FRED economic data (with retry logic)
│   ├── loadOilAnalytics()     → EIA oil data
│   ├── loadGovernmentSpending() → USASpending
│   ├── loadPizzInt()          → DEFCON indicator
│   └── loadCachedPosturesForBanner()
│
├── [E] Refresh Scheduling
│   ├── scheduleRefresh()      → generic timer with jitter, visibility-aware
│   ├── setupRefreshIntervals() → registers all refresh schedules
│   ├── flushStaleRefreshes()  → re-trigger after tab return
│   └── (managed via refreshTimeoutIds, refreshRunners maps)
│
├── [F] Country Intelligence / Brief
│   ├── openCountryBrief()     → reverse geocode → brief
│   ├── openCountryBriefByCode() → CII, signals, stock, news, AI brief (~150 lines)
│   ├── mountCountryTimeline() → build timeline events from cached data
│   ├── getCountrySignals()    → aggregate signals by country
│   ├── openCountryStory()     → story modal with share data
│   ├── isInCountry()          → geo bounds check
│   └── static: resolveCountryName, getCountrySearchTerms, getOtherCountryTerms,
│       firstMentionPosition, toFlagEmoji, COUNTRY_BOUNDS, COUNTRY_ALIASES
│
├── [G] Search
│   ├── setupSearchModal()     → register sources by variant (~160 lines)
│   ├── handleSearchResult()   → dispatch result type → map/panel action
│   ├── updateSearchIndex()    → refresh dynamic sources
│   └── buildCountrySearchItems()
│
├── [H] Desktop / Update System
│   ├── setupUpdateChecks()    → periodic version polling
│   ├── checkForUpdate()       → fetch remote version, compare
│   ├── showUpdateBadge()      → DOM badge creation
│   ├── resolveUpdateDownloadUrl() → platform-specific download URL
│   ├── isNewerVersion()       → semver comparison
│   ├── mapDesktopDownloadPlatform()
│   └── getDesktopBuildVariant()
│
├── [I] Deep Links
│   ├── handleDeepLinks()      → /story?c=XX, ?country=XX
│   └── (retry-polling until data ready)
│
├── [J] URL State Sync
│   ├── setupUrlStateSync()    → debounced history.replaceState
│   ├── getShareUrl()          → build shareable URL
│   └── copyToClipboard()
│
├── [K] UI Utilities
│   ├── showToast()
│   ├── setCopyLinkFeedback()
│   ├── toggleFullscreen()
│   ├── setupMapResize()       → drag-to-resize map section
│   ├── setupMapPin()          → sticky map toggle
│   ├── startHeaderClock()     → UTC clock in header
│   ├── updateHeaderThemeIcon()
│   └── setupIdleDetection()   → pause animations after 2min idle
│
├── [L] Settings / Configuration
│   ├── setupUnifiedSettings() → settings panel callbacks
│   ├── setupMobileWarning()
│   ├── setupStatusPanel()
│   ├── setupPizzIntIndicator()
│   ├── setupExportPanel()
│   ├── setupPlaybackControl()
│   ├── setupSnapshotSaving()  → periodic IndexedDB snapshots
│   ├── restoreSnapshot()
│   ├── getAllSourceNames()
│   └── getLocalizedPanelName()
│
├── [M] Correlation & Analysis
│   ├── runCorrelationAnalysis() → web worker clustering + geo convergence
│   ├── flashMapForNews()      → map flash on new news
│   ├── findFlashLocation()    → keyword→hotspot geolocation
│   └── updateMonitorResults()
│
└── [N] News Filtering
    ├── filterItemsByTimeRange()
    ├── getTimeRangeWindowMs()
    ├── getTimeRangeLabel()
    ├── renderNewsForCategory()
    └── applyTimeRangeFilterToNewsPanels()
```

### Cross-component data flows:

```
loadNews ──→ allNews ──→ updateMonitorResults, updateSearchIndex, flashMapForNews
           ──→ latestClusters ──→ runCorrelationAnalysis, InsightsPanel, map
loadMarkets ──→ latestMarkets ──→ runCorrelationAnalysis, updateSearchIndex
loadPredictions ──→ latestPredictions ──→ runCorrelationAnalysis, updateSearchIndex
loadIntelligenceSignals ──→ intelligenceCache ──→ getCountrySignals, mountCountryTimeline, loadProtests(cached), loadMilitary(cached), loadOutages(cached)
```

---

## 2. Proposed Module Structure

| # | File | Responsibility |
|---|------|---------------|
| 1 | `src/app/data-loaders.ts` | All `load*()` methods: fetching, caching, status/freshness reporting, map/panel updates |
| 2 | `src/app/refresh-scheduler.ts` | `scheduleRefresh()`, `flushStaleRefreshes()`, `setupRefreshIntervals()` — timer management with jitter and visibility awareness |
| 3 | `src/app/country-intel.ts` | Country brief, story, timeline, signals aggregation, country bounds/aliases, and all static country helper methods |
| 4 | `src/app/search-controller.ts` | `setupSearchModal()`, `handleSearchResult()`, `updateSearchIndex()`, `buildCountrySearchItems()` — search source registration and result dispatch |
| 5 | `src/app/desktop-updater.ts` | Desktop update checks, version comparison, badge rendering, download URL resolution |
| 6 | `src/app/panel-manager.ts` | Panel creation, ordering, drag-reorder, settings application, and layout migration logic |
| 7 | `src/app/event-handlers.ts` | `setupEventListeners()`, idle detection, visibility handling, fullscreen, map resize/pin, theme toggle |
| 8 | `src/app/deep-links.ts` | `handleDeepLinks()` — URL-based navigation for stories and country briefs |
| 9 | `src/app/url-state.ts` | URL state sync, share URL generation, clipboard helpers |
| 10 | `src/app/news-filter.ts` | Time-range filtering, category rendering, time range label/window helpers |
| 11 | `src/app/layout-renderer.ts` | `renderLayout()` HTML template, `renderCriticalBanner()`, header clock, theme icon |
| 12 | `src/app/correlation-engine.ts` | `runCorrelationAnalysis()`, `flashMapForNews()`, `findFlashLocation()`, monitor updates |
| 13 | `src/App.ts` | Slim orchestrator: constructor, `init()`, `destroy()`, delegates to modules. Holds shared state (`map`, `panels`, `allNews`, `latestClusters`, etc.) via a shared context object |

---

## 3. Circular Dependencies & Coupling Problems

### Problem 1: Shared Mutable State (Critical)
Almost every method reads/writes a shared set of instance fields:
- `this.map`, `this.panels`, `this.allNews`, `this.latestClusters`, `this.latestMarkets`, `this.latestPredictions`
- `this.intelligenceCache`, `this.statusPanel`, `this.signalModal`
- `this.mapLayers`, `this.panelSettings`, `this.disabledSources`

**Resolution**: Define an `AppContext` interface that exposes these as getters/setters. Each module receives the context object, not the full App class. This breaks the circular "module needs App, App needs module" pattern.

### Problem 2: Data Loaders ↔ Panels Coupling
`loadMilitary()`, `loadIntelligenceSignals()`, etc. directly cast `this.panels['cii']` to `CIIPanel` and call `.refresh()`. This makes data loading tightly coupled to specific panel implementations.

**Resolution**: Use an event/callback pattern. Data loaders emit data events; the panel manager subscribes to them and dispatches to the appropriate panel. Or, more pragmatically, data loaders return their results and App.ts orchestrates the panel updates.

### Problem 3: Country Intel ↔ Data Loaders
`openCountryBriefByCode()` reads `this.intelligenceCache`, `this.allNews`, `this.latestClusters` — data produced by loaders. And `getCountrySignals()` filters cached intelligence data.

**Resolution**: Country intel module receives the context object (read-only access to caches). No circular dependency since it only reads, never writes to loader state.

### Problem 4: Search ↔ Everything
`setupSearchModal()` registers data from config imports (`INTEL_HOTSPOTS`, `MILITARY_BASES`, etc.) and reads `this.allNews`, `this.latestPredictions`, `this.latestMarkets`.

**Resolution**: Same context pattern. Search module registers static sources at setup time and gets dynamic data from context on `updateSearchIndex()`.

### Problem 5: Refresh Scheduler ↔ Data Loaders
`setupRefreshIntervals()` references `this.loadNews()`, `this.loadMarkets()`, etc. by method reference.

**Resolution**: Pass loader functions as a registry object to the scheduler module. No direct import of the loader module.

### Verdict: No true circular dependencies exist
All flows are unidirectional: init → setup → load → update. The coupling is through shared mutable state, which the `AppContext` pattern resolves cleanly.

---

## 4. Recommended Split Order

Extract in this order to minimize breakage at each step:

### Wave 1 — Pure utilities, no shared state dependency
1. **`news-filter.ts`** — Pure functions (`filterItemsByTimeRange`, `getTimeRangeWindowMs`, `getTimeRangeLabel`). Zero coupling to App state. Extract and test immediately.
2. **`desktop-updater.ts`** — Self-contained subsystem (version check, badge DOM, platform mapping). Only needs `isDesktopApp` and container element.
3. **`deep-links.ts`** — Isolated URL parsing and retry-polling logic. Needs only a few callbacks.
4. **`url-state.ts`** — Pure URL building and clipboard utilities.

### Wave 2 — Define shared context, extract state-dependent modules
5. **Define `AppContext` interface** in `src/app/types.ts` — the shared state contract.
6. **`refresh-scheduler.ts`** — Generic scheduling engine, receives function registry.
7. **`correlation-engine.ts`** — Reads clusters/predictions/markets from context, writes signals.
8. **`country-intel.ts`** — Large self-contained subsystem (~400 lines). Reads intelligence cache and news from context.

### Wave 3 — Large modules with panel dependencies
9. **`search-controller.ts`** — Depends on config imports + context for dynamic data.
10. **`panel-manager.ts`** — Panel creation, ordering, drag logic. The biggest behavioral change since it currently lives in `createPanels()`.
11. **`layout-renderer.ts`** — HTML template generation. Depends on variant, theme, i18n.
12. **`event-handlers.ts`** — DOM event wiring. Depends on map, search, theme.

### Wave 4 — The heavyweight
13. **`data-loaders.ts`** — Extract last because it's the most coupled module (~1,800 lines). By this point, all its downstream consumers are already modularized, making the extraction cleaner.
14. **Slim down `App.ts`** — Should be ~200-300 lines: constructor, init orchestration, destroy, and the AppContext implementation.
