# Phase 7: Conservation & Energy Trackers - Research

**Researched:** 2026-02-23
**Domain:** Wildlife conservation data panels + renewable energy visualization panels
**Confidence:** MEDIUM (species data sourcing has constraints; energy data via existing EIA + World Bank is HIGH confidence)

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SPECIES-01 | Wildlife conservation wins displayed as cards with species photo, population trend sparkline, and recovery status badge | IUCN Red List API v4 provides `/api/v4/population_trends/increasing` to list species with increasing populations. Individual assessments via `/api/v4/assessment/{id}` provide narrative text and Red List category. Species photos require a separate source (curated static dataset or Wikimedia Commons API). D3 sparklines use the same pattern as ProgressChartsPanel. |
| SPECIES-02 | Data sourced from IUCN Red List data and conservation reports | IUCN Red List API v4 requires a free token (apply at api.iucnredlist.org). API provides species data, population trends, and conservation status. However, the API does NOT provide longitudinal population count time-series data -- only trend direction (increasing/decreasing/stable/unknown). Historical population numbers must come from a curated static dataset compiled from published conservation reports (NOAA, USFWS, IUCN case studies). |
| SPECIES-03 | Monthly update cadence with historical population trend data | IUCN Red List updates when new versions publish (2025-1 was March 2025). Monthly cadence is achievable via a curated JSON data file refreshed monthly rather than live API polling. The IUCN API itself updates infrequently (1-2x/year). Historical population trend data (actual numbers over years) is NOT available from the IUCN API and must be curated from published conservation literature. |
| ENERGY-01 | Renewable energy capacity visualization showing solar/wind installations growing, coal plants closing | EIA API v2 `operating-generator-capacity` endpoint provides monthly generator inventory data since 2008, filterable by `energy_source_code` facet (SUN for solar, WND for wind, coal codes). Includes nameplate capacity (MW), operating dates, and planned retirement dates. This is US-only data. For global data, World Bank indicators `EG.ELC.RNEW.ZS` (renewable % of electricity) and `EG.ELC.RNWX.ZS` (renewable excluding hydro %) provide country-level time-series from 1990-2021. |
| ENERGY-02 | Animated gauge showing global renewable percentage climbing plus regional breakdown | World Bank indicator `EG.ELC.RNEW.ZS` provides global aggregate via country code `1W` (World). Regional breakdown possible using WB region codes (EAS, ECS, LCN, MEA, NAC, SAS, SSF). D3 arc/donut gauge with CSS transition animation for the climbing effect. The existing `getIndicatorData()` RPC already supports this exact World Bank query pattern (used by progress-data.ts). |
| ENERGY-03 | Data from IEA Renewable Energy Progress Tracker and existing EIA API integration | IEA does NOT provide a free public API -- their data is behind paywall/dashboard downloads only. However, the World Bank's SE4ALL indicators (EG.ELC.RNEW.ZS, EG.FEC.RNEW.ZS) are sourced FROM IEA data (IEA Energy Statistics Data Browser). So we can use World Bank as the free API proxy for IEA-sourced renewable data. The existing EIA integration (`server/worldmonitor/economic/v1/get-energy-prices.ts`) already fetches from `api.eia.gov` with an API key. Extending it to query `operating-generator-capacity` is straightforward. |
</phase_requirements>

---

## Summary

Phase 7 adds two new panels to the happy variant: a **Species Comeback Panel** (conservation wins) and a **Renewable Energy Panel** (capacity growth visualization). Both follow the established `Panel` base class pattern used by Phase 5-6 panels.

**Species data** is the more complex domain. The IUCN Red List API v4 (free, token-required) provides species lists by population trend and conservation status, but critically does NOT provide historical population count time-series. The API returns trend direction (increasing/decreasing/stable/unknown) and Red List category (LC/NT/VU/EN/CR/EW/EX), but not "population was X in 1970, Y in 2000, Z in 2024." For sparklines showing population recovery curves, the data must be curated from published conservation reports (USFWS, NOAA, IUCN case studies) into a static JSON dataset. This is the primary challenge of the species panel. Species photos must also be sourced separately -- either curated image URLs or Wikimedia Commons. The recommended approach is a curated `conservation-wins.json` static data file containing 15-25 species with: name, scientific name, population timeline data, photo URL, recovery status, source citation, and IUCN category. The IUCN API can supplement this with fresh Red List category checks, but the core sparkline data must be pre-compiled.

**Energy data** is well-supported by existing infrastructure. World Bank indicators for renewable energy percentage (EG.ELC.RNEW.ZS, EG.FEC.RNEW.ZS, EG.ELC.RNWX.ZS) are already queryable via the existing `getIndicatorData()` RPC in the economic service. Global and regional aggregates are available via country code `1W` and WB region codes. The EIA API v2 `operating-generator-capacity` endpoint can provide US-specific capacity data by fuel type (solar/wind/coal) including retirement dates. D3.js is already in the project for the progress charts and can be reused for the energy gauge and capacity bar charts.

**Primary recommendation:** Build the species panel around a curated static JSON dataset for population sparkline data, supplemented by optional IUCN API calls for live Red List status. Build the energy panel entirely from existing World Bank RPC + an optional EIA capacity extension. Both panels follow the established Panel class lifecycle and D3.js charting patterns.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Vanilla TypeScript | Project standard | Panel classes | No framework (project decision) |
| D3.js | Already in project | Sparklines, area charts, arc gauge | Used by ProgressChartsPanel -- same pattern |
| `Panel` base class | `src/components/Panel.ts` | Lifecycle, resize, header | Mandatory for all happy variant panels |
| `getIndicatorData()` | `src/services/economic/index.ts` | World Bank renewable energy queries | Already proven for progress-data.ts, same pattern |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `getCSSColor()` | `src/utils/index.ts` | Theme-aware colors for D3 charts | Used by ProgressChartsPanel for chart colors |
| `replaceChildren()` | `src/utils/dom-utils.ts` | Safe DOM replacement | Used by all Panel subclasses |
| `createCircuitBreaker()` | `src/utils/index.ts` | Resilient API calls | If adding IUCN API or EIA capacity calls |
| `dataFreshness` | `src/services/data-freshness.ts` | Track data update timestamps | For recording when energy/species data updates |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Curated JSON for species data | Live IUCN API only | IUCN API lacks population count time-series -- no sparklines possible with API alone |
| World Bank for global renewable % | IEA API directly | IEA has no free public API; World Bank sources its data FROM IEA anyway |
| D3 arc gauge | CSS conic-gradient gauge | D3 gives smoother animation, tooltip interaction, and consistency with existing D3 charts |
| EIA for US capacity details | IRENA for global capacity | IRENA has no API; data is PDF/spreadsheet only. EIA is already integrated. |

**Installation:** No new npm packages required.

---

## Architecture Patterns

### Recommended Project Structure

```
src/
  services/
    conservation-data.ts        # Species data service: curated JSON + optional IUCN API
    renewable-energy-data.ts    # Renewable energy service: World Bank + EIA capacity queries
  components/
    SpeciesComebackPanel.ts     # SPECIES-01, SPECIES-02, SPECIES-03
    RenewableEnergyPanel.ts     # ENERGY-01, ENERGY-02, ENERGY-03
  data/
    conservation-wins.json      # Curated species recovery dataset (static)
  config/
    variants/happy.ts           # Add 'species' and 'renewable' panel keys
    panels.ts                   # Add to HAPPY_PANELS
  styles/
    happy-theme.css             # Panel CSS (species cards, energy gauge, capacity bars)
```

### Pattern 1: Curated Static Data + Optional API Enrichment

**What:** Load a curated JSON file at build time or via dynamic import for the species panel's core sparkline data. Optionally call IUCN API to verify/update Red List categories.

**When to use:** When the primary API (IUCN) lacks the specific data format needed (time-series population counts) but can still provide supplementary live data (current Red List status).

**Example:**
```typescript
// conservation-data.ts
import type { ProgressDataPoint } from '@/services/progress-data';

export interface SpeciesRecovery {
  id: string;
  commonName: string;
  scientificName: string;
  photoUrl: string;           // Curated photo URL (Wikimedia Commons or similar)
  iucnCategory: string;       // LC, NT, VU, EN, CR, EW, EX
  populationTrend: 'increasing' | 'stable' | 'decreasing';
  recoveryStatus: 'recovered' | 'recovering' | 'stabilized';
  populationData: ProgressDataPoint[];  // [{year: 1970, value: 417}, {year: 2024, value: 5800}]
  summaryText: string;        // "Population grew from X to Y thanks to..."
  source: string;             // "USFWS 2024 Survey"
  region: string;             // "North America", "Global", etc.
  lastUpdated: string;        // ISO date of data compilation
}

// Load from curated static JSON
export async function fetchConservationWins(): Promise<SpeciesRecovery[]> {
  const { default: data } = await import('@/data/conservation-wins.json');
  return data as SpeciesRecovery[];
}
```

### Pattern 2: Reuse Existing World Bank RPC for Energy Data

**What:** Use the already-proven `getIndicatorData()` from `src/services/economic/index.ts` to fetch renewable energy World Bank indicators, exactly like `progress-data.ts` does for life expectancy, literacy, etc.

**When to use:** For global/regional renewable energy percentage data.

**Example:**
```typescript
// renewable-energy-data.ts
import { getIndicatorData } from '@/services/economic';

export interface RenewableEnergyDataSet {
  indicator: { id: string; label: string; color: string };
  data: Array<{ year: number; value: number }>;
  latestValue: number;
  latestYear: number;
}

const RENEWABLE_INDICATORS = [
  { id: 'renewableElectricity', code: 'EG.ELC.RNEW.ZS', label: 'Renewable Electricity Output (%)', color: '#6B8F5E' },
  { id: 'renewableExHydro', code: 'EG.ELC.RNWX.ZS', label: 'Renewable (ex. Hydro) (%)', color: '#C4A35A' },
  { id: 'renewableConsumption', code: 'EG.FEC.RNEW.ZS', label: 'Renewable Energy Consumption (%)', color: '#7BA5C4' },
];

const REGIONS = ['1W', 'EAS', 'ECS', 'LCN', 'MEA', 'NAC', 'SAS', 'SSF'];

export async function fetchGlobalRenewableData(): Promise<RenewableEnergyDataSet[]> {
  // Same pattern as progress-data.ts -- fetch World Bank indicators for World aggregate
  const results = await Promise.all(
    RENEWABLE_INDICATORS.map(async (indicator) => {
      const response = await getIndicatorData(indicator.code, { countries: ['1W'], years: 35 });
      const countryData = response.byCountry['1W'];
      // ... extract and return (same as progress-data.ts pattern)
    }),
  );
  return results;
}
```

### Pattern 3: EIA Capacity Extension (US-specific)

**What:** Extend the existing EIA handler with a new RPC for `operating-generator-capacity` data, or add it as a new endpoint alongside `getEnergyPrices`.

**When to use:** For US-specific solar/wind installation growth and coal retirement visualization.

**Example EIA API v2 URL:**
```
https://api.eia.gov/v2/electricity/operating-generator-capacity/data/
  ?api_key=XXX
  &frequency=monthly
  &data[0]=nameplate-capacity-mw
  &facets[energy_source_code][]=SUN
  &facets[energy_source_code][]=WND
  &sort[0][column]=period
  &sort[0][direction]=desc
  &length=100
```

### Pattern 4: D3 Arc Gauge for Renewable Percentage

**What:** Build an animated SVG arc/donut gauge showing global renewable energy percentage, using D3's `d3.arc()` generator with CSS transitions for the climbing animation effect.

**When to use:** For ENERGY-02's animated gauge showing renewable percentage climbing.

**Example:**
```typescript
// Inside RenewableEnergyPanel.renderGauge()
const arc = d3.arc()
  .innerRadius(radius * 0.7)
  .outerRadius(radius)
  .startAngle(0)
  .cornerRadius(4);

// Background arc (full circle)
g.append('path')
  .attr('d', arc({ endAngle: Math.PI * 2 }) as string)
  .attr('fill', 'var(--border)');

// Foreground arc (renewable percentage) with transition
const foreground = g.append('path')
  .attr('fill', '#6B8F5E')
  .attr('d', arc({ endAngle: 0 }) as string);

// Animate to target percentage
foreground.transition()
  .duration(1500)
  .ease(d3.easeCubicOut)
  .attrTween('d', () => {
    const interpolate = d3.interpolate(0, (percentage / 100) * Math.PI * 2);
    return (t) => arc({ endAngle: interpolate(t) }) as string;
  });
```

### Anti-Patterns to Avoid

- **Fetching IUCN API on every page load:** IUCN data changes 1-2x/year. The curated JSON file is the right primary source. At most, use IUCN API as a monthly background enrichment to verify Red List categories haven't changed.
- **Depending on IEA API directly:** IEA has no free public API. World Bank acts as the free proxy for IEA-sourced data.
- **Building a new RPC service for static data:** The species curated JSON should be imported directly (bundled or fetched as static asset), not wrapped in a sebuf service. Only the World Bank and EIA calls need server-side proxying (for API key protection).
- **Adding IUCN_API_KEY to runtime-config:** The species panel can work entirely from static data. If IUCN API enrichment is added later, it should be optional and not block the panel from rendering.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Population sparklines | Custom canvas renderer | D3.js `d3.line()` + `d3.area()` | Already used by ProgressChartsPanel; proven pattern with tooltips |
| Animated gauge | CSS conic-gradient with JS timer | D3 `d3.arc()` with `.transition()` | Smoother animation, easier to add tooltip and regional breakdown |
| World Bank data fetching | Direct `fetch()` to World Bank API | `getIndicatorData()` from economic service | Already has circuit breaker, pagination, response parsing |
| Species photo loading | Raw `<img>` with no error handling | `<img>` with `onerror` fallback to placeholder SVG | Curated URLs may go stale; always need a fallback |
| Theme-aware chart colors | Hardcoded hex values | `getCSSColor('--green')` etc. | Dark mode support is mandatory |

**Key insight:** The project already has D3.js, World Bank API integration, and a proven panel lifecycle. Phase 7 should reuse all of these rather than introducing new charting libraries or data pipelines.

---

## Common Pitfalls

### Pitfall 1: IUCN API Does Not Have Population Count Time-Series

**What goes wrong:** Assuming the IUCN Red List API provides historical population numbers (e.g., "bald eagle had 417 pairs in 1963, 9,789 in 2006, 71,400 in 2020"). It does not. The API only provides the population trend direction (increasing/decreasing/stable/unknown) and the current Red List category.

**Why it happens:** The requirement says "population trend sparkline" and "IUCN Red List data," which sounds like the API should provide the sparkline data. But the IUCN API is an assessment database, not a population census database.

**How to avoid:** Use a curated static JSON file for sparkline population numbers, compiled from published conservation reports (USFWS surveys, NOAA assessments, peer-reviewed papers). The IUCN API can supplement with live Red List status checks.

**Warning signs:** Panel shows flat sparklines or only two data points (latest assessment value).

### Pitfall 2: World Bank Renewable Data Lag

**What goes wrong:** World Bank indicator `EG.ELC.RNEW.ZS` data stops at 2021-2022. Users expect to see 2024-2025 data.

**Why it happens:** World Bank aggregates from IEA statistics, which have a 1-2 year publication lag.

**How to avoid:** Display the data range clearly in the UI ("1990-2022"). Optionally supplement with the most recent year from EIA data (which updates monthly for US data). Consider displaying a "latest available" badge with the actual year.

**Warning signs:** Users report "old data" or "not updating."

### Pitfall 3: EIA Energy Source Codes

**What goes wrong:** Using wrong facet values for filtering solar/wind/coal generators in the EIA API.

**Why it happens:** EIA energy source codes are abbreviated (e.g., SUN for solar, WND for wind). Coal has multiple codes (BIT for bituminous, SUB for subbituminous, LIG for lignite, RC for refined coal). Missing codes means missing data.

**How to avoid:** Query the EIA API facet metadata endpoint first to discover available codes, or use the documented codes: SUN (solar), WND (wind), BIT/SUB/LIG/RC (coal variants). Test with `?api_key=DEMO_KEY` first.

**Warning signs:** Coal retirement numbers seem too low (forgot a coal sub-type).

### Pitfall 4: Species Photo URL Stability

**What goes wrong:** Curated photo URLs in the static JSON break over time as external sources reorganize or remove images.

**Why it happens:** External image hosts (Wikimedia, government sites) change URL structures.

**How to avoid:** Use Wikimedia Commons URLs (most stable) and always include an `onerror` fallback to a generic species silhouette SVG. Consider hosting the images in the project's own static assets if licensing permits.

**Warning signs:** Broken image icons in the species cards.

### Pitfall 5: Panel Grid Overflow with Two New Panels

**What goes wrong:** Adding two more panels to the happy variant grid (now 9 total) causes layout overflow on smaller screens.

**Why it happens:** The happy variant already has 7 panels (map, positive-feed, progress, counters, spotlight, breakthroughs, digest). Two more pushes the count to 9.

**How to avoid:** Consider the panel span classes. Species and energy panels may need `priority: 2` in panel config so they appear after the core panels. Test at viewport widths 768px and 1024px.

**Warning signs:** Panels appear too small or the grid becomes scrolly on typical laptop screens.

---

## Code Examples

Verified patterns from the existing codebase:

### Panel Class Lifecycle (from CountersPanel)

```typescript
// Source: src/components/CountersPanel.ts
export class SpeciesComebackPanel extends Panel {
  constructor() {
    super({ id: 'species', title: 'Conservation Wins', trackActivity: false });
  }

  public setData(species: SpeciesRecovery[]): void {
    replaceChildren(this.content);
    // ... render species cards with D3 sparklines
  }

  public destroy(): void {
    // cleanup
    super.destroy();
  }
}
```

### D3 Sparkline (from ProgressChartsPanel)

```typescript
// Source: src/components/ProgressChartsPanel.ts - line 184-209
const area = d3.area<DataPoint>()
  .x(d => x(d.year))
  .y0(height)
  .y1(d => y(d.value))
  .curve(d3.curveMonotoneX);

const line = d3.line<DataPoint>()
  .x(d => x(d.year))
  .y(d => y(d.value))
  .curve(d3.curveMonotoneX);

g.append('path').datum(data).attr('d', area).attr('fill', color).attr('opacity', 0.2);
g.append('path').datum(data).attr('d', line).attr('fill', 'none').attr('stroke', color).attr('stroke-width', 2);
```

### World Bank Data Fetch (from progress-data.ts)

```typescript
// Source: src/services/progress-data.ts - line 103-165
const response = await getIndicatorData(indicator.code, {
  countries: ['1W'],  // World aggregate
  years: indicator.years,
});

const countryData = response.byCountry['1W'];
const data = countryData.values
  .filter(v => v.value != null && Number.isFinite(v.value))
  .map(v => ({ year: parseInt(v.year, 10), value: v.value }))
  .sort((a, b) => a.year - b.year);
```

### App.ts Wiring Pattern (from Phase 6 panels)

```typescript
// Source: src/App.ts - line 2435-2461
// Panel creation:
if (SITE_VARIANT === 'happy') {
  this.speciesPanel = new SpeciesComebackPanel();
  this.panels['species'] = this.speciesPanel;
}

// Data loading in initial tasks:
if (SITE_VARIANT === 'happy') {
  tasks.push({
    name: 'species',
    task: runGuarded('species', () => this.loadSpeciesData()),
  });
}

// Load method:
private async loadSpeciesData(): Promise<void> {
  const species = await fetchConservationWins();
  this.speciesPanel?.setData(species);
}
```

### Happy Variant Panel Config (from variants/happy.ts)

```typescript
// Source: src/config/variants/happy.ts
export const DEFAULT_PANELS: Record<string, PanelConfig> = {
  // ... existing panels ...
  species: { name: 'Conservation Wins', enabled: true, priority: 1 },
  renewable: { name: 'Renewable Energy', enabled: true, priority: 1 },
};
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| IUCN Red List API v3 | IUCN Red List API v4 | 2024 | v4 has OpenAPI spec, better pagination, assessment search endpoint |
| EIA API v1 (deprecated) | EIA API v2 | 2023 | v2 has faceted querying, RESTful hierarchy, better filtering |
| Hardcoded annual reports | World Bank API + SE4ALL indicators | Ongoing | Standardized indicator codes with global + regional aggregates |
| Chart.js | D3.js (project standard) | Project decision | D3 provides lower-level control for custom visualizations like arc gauges |

**Deprecated/outdated:**
- IUCN API v3: Still functional but v4 is the documented version with better tooling
- EIA API v1: Deprecated; project already uses v2 paths in `get-energy-prices.ts`
- IEA free data API: Never existed as a free public API; always required commercial license

---

## Data Architecture Decision: Static vs. Live for Species Data

### Option A: Fully Curated Static JSON (Recommended)

A `conservation-wins.json` file containing 15-25 well-documented conservation success stories with population timeline data compiled from published sources. Updated monthly via a manual or scripted refresh process.

**Pros:**
- No API key required
- No API rate limits or downtime
- Sparkline data guaranteed (actual population numbers over decades)
- High-quality curated content (only genuine success stories)
- Fast loading (bundled with app)

**Cons:**
- Requires manual curation effort
- Data can become stale if not refreshed
- Limited to pre-selected species

### Option B: IUCN API-Primary with Curated Fallback

Use IUCN API `/api/v4/population_trends/increasing` to dynamically list improving species, then look up assessments. Fall back to curated JSON when API is unavailable.

**Pros:**
- Dynamic, fresh data from authoritative source
- Covers all species, not just curated ones

**Cons:**
- No population count time-series from API (sparklines would be flat/estimated)
- Requires IUCN_API_KEY token (free but requires registration)
- API updates infrequently (1-2x/year)
- Photo URLs not available from IUCN API
- Recovery "stories" not available from IUCN API

### Recommendation: Option A (Static JSON) with optional IUCN enrichment

The sparkline requirement (SPECIES-01) makes curated data essential. The IUCN API is best used as a secondary enrichment to verify Red List categories are current, not as the primary data source.

---

## Curated Conservation Wins Dataset -- Suggested Species

Based on well-documented conservation successes with available population data:

1. **Bald Eagle** (Haliaeetus leucocephalus) -- 417 pairs (1963) to 71,400 pairs (2020). Source: USFWS
2. **Humpback Whale** (Megaptera novaeangliae) -- ~5,000 (1966) to ~80,000 (2024). Source: NOAA/IWC
3. **Giant Panda** (Ailuropoda melanoleuca) -- 1,114 (1988) to 1,864 (2014). Source: WWF/Chinese Government
4. **Southern White Rhino** (Ceratotherium simum simum) -- ~50 (1900) to ~16,800 (2024). Source: IUCN
5. **Gray Wolf** (Canis lupus) -- 1,000 (1974) to 6,100 (2023) in US lower 48. Source: USFWS
6. **Peregrine Falcon** (Falco peregrinus) -- 324 pairs (1975) to 3,000+ pairs (2015). Source: TPC
7. **American Alligator** (Alligator mississippiensis) -- Endangered (1967) to 5 million+ (2024). Source: FWS
8. **Arabian Oryx** (Oryx leucoryx) -- 0 wild (1972) to 1,220 (2023). Source: EAD/IUCN
9. **California Condor** (Gymnogyps californianus) -- 22 (1982) to 561 (2024). Source: USFWS
10. **Mountain Gorilla** (Gorilla beringei beringei) -- 254 (1981) to 1,063 (2021). Source: GVTC

Each entry needs: common name, scientific name, population timeline (array of {year, value}), photo URL, IUCN category, recovery status, summary text, and source citation.

---

## Renewable Energy Data Sources Summary

### World Bank Indicators (Global + Regional -- Existing RPC)

| Indicator Code | Name | Coverage | Data Range |
|----------------|------|----------|------------|
| `EG.ELC.RNEW.ZS` | Renewable electricity output (% of total) | Global + 200+ countries | 1990-2022 |
| `EG.ELC.RNWX.ZS` | Renewable (excluding hydro) (% of total) | Global + 200+ countries | 1990-2022 |
| `EG.FEC.RNEW.ZS` | Renewable energy consumption (% of total) | Global + 200+ countries | 1990-2022 |

**Access:** Already works via `getIndicatorData()` from `src/services/economic/index.ts`. Tested pattern from `progress-data.ts`.

**Regional breakdown country codes for World Bank:**
- `1W` = World, `EAS` = East Asia & Pacific, `ECS` = Europe & Central Asia, `LCN` = Latin America & Caribbean, `MEA` = Middle East & North Africa, `NAC` = North America, `SAS` = South Asia, `SSF` = Sub-Saharan Africa

### EIA API v2 (US-specific -- Existing Integration)

| Endpoint | Path | Data | Frequency |
|----------|------|------|-----------|
| Operating Generator Capacity | `/v2/electricity/operating-generator-capacity/data/` | Nameplate capacity (MW), operating dates, retirement dates | Monthly |
| Electric Power Operational Data | `/v2/electricity/electric-power-operational-data/data/` | Generation by fuel type (MWh) | Monthly |

**Facet values for fuel type filtering:**
- Solar: `SUN`
- Wind: `WND`
- Coal: `BIT` (bituminous), `SUB` (subbituminous), `LIG` (lignite), `RC` (refined coal)

**Access:** Requires `EIA_API_KEY` (already in project env). Server-side handler at `server/worldmonitor/economic/v1/get-energy-prices.ts` provides the pattern for adding new EIA endpoints.

### IRENA (Not accessible via API)

IRENA publishes global renewable capacity statistics annually (Renewable Capacity Statistics 2025, released March 2025) but has no public API. Data available only as PDF/Excel downloads. The humanity-counters service already references IRENA data (510,000 MW annual additions). Hardcoded annual totals from IRENA publications can supplement World Bank data for the most recent year.

---

## Open Questions

1. **IUCN API Token Registration**
   - What we know: Token is free, obtained by applying at api.iucnredlist.org. Required for API access. Commercial use is forbidden.
   - What's unclear: How long token provisioning takes. Whether the happy.worldmonitor.app use case qualifies as non-commercial.
   - Recommendation: Build species panel to work entirely from curated static data. Add IUCN API enrichment as an optional enhancement in a later phase.

2. **Species Photo Licensing**
   - What we know: Wikimedia Commons has CC-licensed photos for most conservation success species. Government sources (USFWS, NOAA) have public domain photos.
   - What's unclear: Whether specific photo URLs are stable long-term. Whether any photos require attribution display.
   - Recommendation: Use Wikimedia Commons URLs with `onerror` fallback to a nature-themed SVG placeholder. Include attribution in the curated JSON.

3. **EIA Capacity Data Granularity**
   - What we know: EIA `operating-generator-capacity` has monthly data since 2008. Can filter by energy source code.
   - What's unclear: Exact facet values for energy_source_code (SUN, WND confirmed; coal variants need validation). Whether aggregated capacity totals are available or if we must sum individual generator records.
   - Recommendation: Test with DEMO_KEY first. Build the panel to work from World Bank data alone, with EIA as an optional US-specific enhancement.

4. **Proto Extension vs. Static Import for Species Data**
   - What we know: All existing happy variant data services use either static data (humanity-counters.ts) or RPC calls (progress-data.ts via economic service).
   - What's unclear: Whether to create a new sebuf service for conservation data or just import static JSON.
   - Recommendation: Use static JSON import for species data (like humanity-counters.ts uses hardcoded data). No new proto/service needed. Reserve RPC services for data that requires server-side API key protection.

---

## Sources

### Primary (HIGH confidence)
- Existing codebase: `src/services/economic/index.ts`, `src/services/progress-data.ts`, `src/components/ProgressChartsPanel.ts` -- verified patterns for World Bank data + D3 charting
- Existing codebase: `server/worldmonitor/economic/v1/get-energy-prices.ts` -- verified EIA API v2 integration pattern
- Existing codebase: `src/services/humanity-counters.ts` -- verified pattern for static data service
- EIA Open Data documentation: https://www.eia.gov/opendata/documentation.php -- API v2 structure, electricity endpoints
- EIA API endpoint discovery: `https://api.eia.gov/v2/electricity` -- confirmed `operating-generator-capacity` route with monthly frequency, MW data, energy_source_code facet

### Secondary (MEDIUM confidence)
- World Bank indicator pages: https://data.worldbank.org/indicator/EG.ELC.RNEW.ZS -- confirmed global aggregate data availability via `1W` code, 1990-2022 range, sourced from IEA
- IUCN Red List API v4 OpenAPI spec: https://api.iucnredlist.org/api-docs/v4/openapi.yaml -- confirmed endpoints for population_trends, red_list_categories, assessment, taxa
- Conservation success population data: Multiple sources (USFWS, NOAA, WWF, IUCN case studies) -- documented species recovery numbers cross-verified across sources

### Tertiary (LOW confidence)
- EIA energy_source_code facet values: SUN, WND, BIT, SUB, LIG, RC -- inferred from EIA form documentation and search results; needs validation against live API
- IRENA 2024 capacity additions (510,000 MW): Referenced in humanity-counters.ts -- specific figure needs verification against IRENA Renewable Capacity Statistics 2025

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- All libraries already in the project; D3, World Bank RPC, Panel class are proven
- Architecture: HIGH -- Follows exact same patterns as Phase 5 (progress-data.ts + ProgressChartsPanel) and Phase 6 (GoodThingsDigestPanel)
- Data availability (energy): HIGH -- World Bank indicators confirmed with global aggregates; EIA already integrated
- Data availability (species): MEDIUM -- IUCN API confirmed but lacks time-series population data; curated JSON approach is sound but requires manual compilation
- Pitfalls: MEDIUM -- Based on API documentation review and codebase analysis; EIA facet codes need live validation

**Research date:** 2026-02-23
**Valid until:** 2026-03-23 (30 days -- stable domain, no fast-moving dependencies)
