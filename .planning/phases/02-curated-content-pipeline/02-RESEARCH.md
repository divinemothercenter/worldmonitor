# Phase 2: Curated Content Pipeline - Research

**Researched:** 2026-02-22
**Domain:** RSS feed integration, GDELT API tone filtering, content categorization
**Confidence:** HIGH

## Summary

Phase 2 adds the data backbone for HappyMonitor: dedicated positive news RSS feeds, GDELT positive tone filtering, and a content category tagging system. The existing codebase already has all the infrastructure needed -- the RSS proxy pipeline (`/api/rss-proxy`), the GDELT Sebuf handler (`server/worldmonitor/intelligence/v1/`), and the variant configuration system (`src/config/variants/happy.ts`). No new external dependencies are required. The work is configuration and integration, not greenfield development.

Three requirements drive this phase: FEED-01 (positive RSS feeds), FEED-03 (GDELT tone filter), and FEED-04 (content categories). The feeds populate the `FEEDS` record in `src/config/variants/happy.ts` (currently an empty placeholder). The GDELT tone filter extends the existing `SearchGdeltDocumentsRequest` proto with a `tone_filter` field. Content categories are a new type system (`HappyContentCategory`) with keyword-based classification, following the same pattern as the existing `threat-classifier.ts` but inverted for positive content.

**Primary recommendation:** Populate the happy variant's `FEEDS` config with verified positive RSS feeds, add a `tone_filter` string field to the GDELT proto, and implement a `positive-classifier.ts` service module that maps stories to the six defined content categories via keyword matching.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| FEED-01 | Dedicated positive news RSS feeds integrated (Good News Network, Positive.News, Reasons to be Cheerful, Optimist Daily, SunnySkyz, The Better India, Future Crunch) | 5 of 7 feeds verified with live RSS. Feed URLs documented. GNN offers 8+ category-specific feeds. Existing RSS proxy pipeline handles all of them with zero code changes. |
| FEED-03 | GDELT positive tone filter -- extend existing GDELT integration with `tone>5` parameter for positive global news | GDELT DOC 2.0 API supports `tone>5` in query string and `sort=ToneDesc`. Existing proto already has `tone` field on `GdeltArticle`. Need to add `tone_filter` to request proto and pass it through the handler to the GDELT URL. |
| FEED-04 | Content categories defined and mapped: Science & Health, Nature & Wildlife, Humanity & Kindness, Innovation & Tech, Climate Wins, Culture & Community | Implement as keyword-based classifier following `threat-classifier.ts` pattern. Each category gets a keyword map. Stories classified at fetch time and tagged on the `NewsItem` type. |
</phase_requirements>

## Standard Stack

### Core

No new libraries. Everything uses existing infrastructure.

| Library/System | Version | Purpose | Why Standard |
|----------------|---------|---------|--------------|
| Existing RSS proxy (`/api/rss-proxy`) | N/A | Fetch RSS feeds through Vercel proxy to avoid CORS | Already handles 150+ feeds across all variants. Proven reliable. |
| Existing `rss.ts` service | N/A | Parse RSS/Atom XML, cache results, circuit-break failures | Handles DOMParser, Atom/RSS detection, per-feed cooldowns. Zero changes needed for new feeds. |
| Existing GDELT Sebuf handler | N/A | Server-side GDELT DOC API queries | `server/worldmonitor/intelligence/v1/search-gdelt-documents.ts` already builds GDELT URLs and returns typed articles. |
| Existing variant config system | N/A | Per-variant feed definitions | `src/config/variants/happy.ts` already exports `FEEDS` record (currently empty placeholder). |
| `buf` + `protoc-gen-ts` | N/A | Proto codegen for GDELT request extension | Standard proto workflow: edit `.proto`, run `cd proto && buf generate`. |

### Supporting

| Library/System | Version | Purpose | When to Use |
|----------------|---------|---------|-------------|
| Railway RSS proxy | N/A | Fallback for feeds blocked by Vercel IPs | If any positive news feed blocks Vercel's IP range. Pattern already established in `src/config/feeds.ts` with `railwayRss()` helper. |
| `persistent-cache.ts` | N/A | IndexedDB caching for feed results | Already wired into `rss.ts` -- cached automatically for offline/stale-while-revalidate. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Client-side RSS parsing (DOMParser) | Server-side RSS parsing in Sebuf handler | Server-side would centralize parsing but requires new handler, increases serverless function complexity. Current client-side approach works and is already battle-tested for 150+ feeds. |
| Keyword-based content classification | ML-based topic classification | ML adds complexity and is Phase 3 scope (FEED-02/FEED-05). Keyword classification is fast, deterministic, and sufficient for curated sources where content is already positive. |
| Individual GDELT queries per topic | Single broad GDELT query with client-side filtering | Per-topic queries give better relevance but multiply API calls. Start with per-topic (matching existing `INTEL_TOPICS` pattern) since GDELT has no rate limit. |

**Installation:**
```bash
# No new packages needed. Zero npm install.
```

## Architecture Patterns

### Recommended Project Structure

```
src/
  config/
    variants/
      happy.ts          # MODIFY: populate FEEDS record with positive RSS feeds
  services/
    positive-classifier.ts  # NEW: keyword-based content category classifier
    gdelt-intel.ts          # MODIFY: add positive-tone query support
  types/
    index.ts            # MODIFY: extend NewsItem and add HappyContentCategory

proto/
  worldmonitor/intelligence/v1/
    search_gdelt_documents.proto  # MODIFY: add tone_filter field to request

server/
  worldmonitor/intelligence/v1/
    search-gdelt-documents.ts     # MODIFY: pass tone_filter to GDELT URL
```

### Pattern 1: Positive RSS Feed Configuration

**What:** Define positive news feeds in the happy variant's `FEEDS` record, following the exact same structure as other variants.

**When to use:** For all curated positive news sources (FEED-01).

**Example:**
```typescript
// src/config/variants/happy.ts
import type { Feed } from '@/types';

const rss = (url: string) => `/api/rss-proxy?url=${encodeURIComponent(url)}`;

export const FEEDS: Record<string, Feed[]> = {
  positive: [
    { name: 'Good News Network', url: rss('https://www.goodnewsnetwork.org/feed/') },
    { name: 'Positive.News', url: rss('https://www.positive.news/feed/') },
    { name: 'Reasons to be Cheerful', url: rss('https://reasonstobecheerful.world/feed/') },
    { name: 'Optimist Daily', url: rss('https://www.optimistdaily.com/feed/') },
    // SunnySkyz, The Better India, Future Crunch -- see Open Questions
  ],
  science: [
    { name: 'GNN Science', url: rss('https://www.goodnewsnetwork.org/category/news/science/feed/') },
  ],
  nature: [
    { name: 'GNN Animals', url: rss('https://www.goodnewsnetwork.org/category/news/animals/feed/') },
  ],
  health: [
    { name: 'GNN Health', url: rss('https://www.goodnewsnetwork.org/category/news/health/feed/') },
  ],
  inspiring: [
    { name: 'GNN Heroes', url: rss('https://www.goodnewsnetwork.org/category/news/inspiring/feed/') },
  ],
};
```

**Source:** Existing pattern from `src/config/variants/full.ts` which re-exports from `../feeds`. Happy variant defines feeds inline because its sources are fundamentally different from the geopolitical feeds.

### Pattern 2: GDELT Tone Filter Extension

**What:** Add a `tone_filter` field to the `SearchGdeltDocumentsRequest` proto and pass it through the handler to the GDELT API URL.

**When to use:** For FEED-03 (GDELT positive tone filtering).

**Example (proto change):**
```protobuf
// proto/worldmonitor/intelligence/v1/search_gdelt_documents.proto
message SearchGdeltDocumentsRequest {
  string query = 1 [(buf.validate.field).required = true, (buf.validate.field).string.min_len = 1];
  int32 max_records = 2 [(buf.validate.field).int32.gte = 1, (buf.validate.field).int32.lte = 250];
  string timespan = 3;
  // Optional tone filter appended to GDELT query (e.g., "tone>5" for positive articles).
  string tone_filter = 4;
  // Optional sort mode (e.g., "ToneDesc" for most positive first). Defaults to "DateDesc".
  string sort = 5;
}
```

**Example (handler change):**
```typescript
// server/worldmonitor/intelligence/v1/search-gdelt-documents.ts
export async function searchGdeltDocuments(
  _ctx: ServerContext,
  req: SearchGdeltDocumentsRequest,
): Promise<SearchGdeltDocumentsResponse> {
  // ... existing validation ...

  let query = req.query;
  // Append tone filter if provided (e.g., "tone>5")
  if (req.toneFilter) {
    query = `${query} ${req.toneFilter}`;
  }

  const gdeltUrl = new URL(GDELT_DOC_API);
  gdeltUrl.searchParams.set('query', query);
  gdeltUrl.searchParams.set('mode', 'artlist');
  gdeltUrl.searchParams.set('maxrecords', maxRecords.toString());
  gdeltUrl.searchParams.set('format', 'json');
  gdeltUrl.searchParams.set('sort', req.sort || 'DateDesc');
  gdeltUrl.searchParams.set('timespan', timespan);
  // ... rest unchanged ...
}
```

**Example (client-side usage):**
```typescript
// src/services/gdelt-intel.ts -- new function for happy variant
export async function fetchPositiveGdeltArticles(
  query: string,
  maxrecords = 15,
  timespan = '72h',
): Promise<GdeltArticle[]> {
  return fetchGdeltArticlesWithTone(query, 'tone>5', 'ToneDesc', maxrecords, timespan);
}
```

**Source:** [GDELT DOC 2.0 API documentation](https://blog.gdeltproject.org/gdelt-doc-2-0-api-debuts/) -- tone filter syntax verified. [GDELT positive news filtering blog post](https://blog.gdeltproject.org/finding-good-news-in-the-midst-of-the-pandemic-using-tone-filtering/) -- official example of `tone>5` usage.

### Pattern 3: Content Category Classifier

**What:** A keyword-based classifier that tags each story with one of the six defined content categories. Follows the exact same pattern as `threat-classifier.ts` (keyword maps -> category assignment) but for positive content.

**When to use:** For FEED-04 (content category tagging).

**Example:**
```typescript
// src/services/positive-classifier.ts

export type HappyContentCategory =
  | 'science-health'
  | 'nature-wildlife'
  | 'humanity-kindness'
  | 'innovation-tech'
  | 'climate-wins'
  | 'culture-community';

export const HAPPY_CATEGORY_LABELS: Record<HappyContentCategory, string> = {
  'science-health': 'Science & Health',
  'nature-wildlife': 'Nature & Wildlife',
  'humanity-kindness': 'Humanity & Kindness',
  'innovation-tech': 'Innovation & Tech',
  'climate-wins': 'Climate Wins',
  'culture-community': 'Culture & Community',
};

type CategoryKeywordMap = Record<string, HappyContentCategory>;

const CATEGORY_KEYWORDS: CategoryKeywordMap = {
  // Science & Health
  'breakthrough': 'science-health',
  'discovery': 'science-health',
  'researchers': 'science-health',
  'scientists': 'science-health',
  'study finds': 'science-health',
  'clinical trial': 'science-health',
  'cure': 'science-health',
  'vaccine': 'science-health',
  'treatment': 'science-health',
  'medical': 'science-health',
  'disease': 'science-health',
  'cancer': 'science-health',
  'therapy': 'science-health',
  // Nature & Wildlife
  'species': 'nature-wildlife',
  'wildlife': 'nature-wildlife',
  'conservation': 'nature-wildlife',
  'endangered': 'nature-wildlife',
  'animal': 'nature-wildlife',
  'marine': 'nature-wildlife',
  'reef': 'nature-wildlife',
  'forest': 'nature-wildlife',
  'whale': 'nature-wildlife',
  'bird': 'nature-wildlife',
  // Humanity & Kindness
  'volunteer': 'humanity-kindness',
  'donated': 'humanity-kindness',
  'charity': 'humanity-kindness',
  'rescued': 'humanity-kindness',
  'hero': 'humanity-kindness',
  'community': 'humanity-kindness',
  'kindness': 'humanity-kindness',
  'helping': 'humanity-kindness',
  // Innovation & Tech
  'ai': 'innovation-tech',
  'robot': 'innovation-tech',
  'technology': 'innovation-tech',
  'startup': 'innovation-tech',
  'invention': 'innovation-tech',
  'innovation': 'innovation-tech',
  'engineering': 'innovation-tech',
  '3d print': 'innovation-tech',
  // Climate Wins
  'renewable': 'climate-wins',
  'solar': 'climate-wins',
  'wind energy': 'climate-wins',
  'electric vehicle': 'climate-wins',
  'emissions': 'climate-wins',
  'carbon': 'climate-wins',
  'clean energy': 'climate-wins',
  'climate': 'climate-wins',
  'green hydrogen': 'climate-wins',
  // Culture & Community
  'art': 'culture-community',
  'music': 'culture-community',
  'festival': 'culture-community',
  'cultural': 'culture-community',
  'education': 'culture-community',
  'school': 'culture-community',
  'library': 'culture-community',
  'museum': 'culture-community',
};

export function classifyPositiveContent(title: string): HappyContentCategory {
  const lower = title.toLowerCase();
  for (const [keyword, category] of Object.entries(CATEGORY_KEYWORDS)) {
    if (lower.includes(keyword)) return category;
  }
  return 'humanity-kindness'; // default for curated positive sources
}
```

**Source:** Pattern derived from `src/services/threat-classifier.ts` lines 60-200. Same approach: keyword map -> iterate -> first match wins. Simple, fast, deterministic.

### Pattern 4: Feed-to-Category Source Mapping

**What:** Curated sources can be pre-mapped to categories. Stories from GNN Science always get `science-health`. Stories from GNN Animals always get `nature-wildlife`. Only general feeds need keyword classification.

**When to use:** As a fast path before keyword classification.

**Example:**
```typescript
// Source-based pre-classification (faster than keyword scanning)
const SOURCE_CATEGORY_MAP: Record<string, HappyContentCategory> = {
  'GNN Science': 'science-health',
  'GNN Health': 'science-health',
  'GNN Animals': 'nature-wildlife',
  'GNN Heroes': 'humanity-kindness',
  'GNN Inspiring': 'humanity-kindness',
};

export function classifyNewsItem(item: NewsItem): HappyContentCategory {
  // Fast path: source already maps to a category
  const sourceCategory = SOURCE_CATEGORY_MAP[item.source];
  if (sourceCategory) return sourceCategory;

  // Slow path: keyword classification
  return classifyPositiveContent(item.title);
}
```

### Anti-Patterns to Avoid

- **ML-based topic classification in Phase 2:** Keyword classification is the right tool for curated sources where content is already positive. Save ML classification for Phase 3 (FEED-02/FEED-05) when filtering mixed-source mainstream feeds.

- **Modifying the shared `fetchFeed()` function for happy-specific behavior:** The existing `rss.ts` pipeline is shared across all variants. Do not add `if (SITE_VARIANT === 'happy')` branches inside `fetchFeed()`. Instead, classification happens after fetch, in the caller or in a wrapper function.

- **Hardcoding GDELT tone threshold:** Make the tone threshold configurable (pass as parameter, not embedded in handler). Different queries may need different thresholds, and Phase 3 may want to adjust based on ML feedback.

- **Creating a new Sebuf domain for positive feeds:** The positive feeds use the exact same RSS proxy and fetch pipeline as other variants. No new backend handler is needed for RSS. Only the GDELT handler needs modification.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| RSS parsing | Custom XML parser | Existing `rss.ts` DOMParser pipeline | Handles RSS 2.0, Atom, parse errors, date fallbacks, circuit breakers. Battle-tested with 150+ feeds. |
| RSS CORS proxy | Direct browser fetch | Existing `/api/rss-proxy` Vercel function | CORS blocks direct browser-to-RSS-server requests. The proxy is already deployed and handles all feeds. |
| Feed caching | Custom cache layer | Existing `feedCache` + `persistent-cache.ts` | Memory cache (10min TTL) + IndexedDB persistence already wired into `fetchFeed()`. |
| Feed error handling | Custom retry logic | Existing circuit breaker in `rss.ts` | Per-feed failure tracking, cooldown (5min after 2 failures), automatic recovery. |
| GDELT API client | Direct fetch in components | Existing `IntelligenceServiceClient` + Sebuf handler | Type-safe proto client, circuit breaker, caching, error handling already in place. |

**Key insight:** Phase 2 is primarily a configuration phase. The infrastructure exists. The work is (1) defining feeds, (2) extending one proto field, (3) writing a lightweight classifier. Resist the urge to refactor the feed pipeline.

## Common Pitfalls

### Pitfall 1: RSS Feed URLs Not Verified Programmatically

**What goes wrong:** A feed URL from a blog post or aggregator listing is outdated, returns 404, or redirects to an HTML page instead of XML. The dashboard shows empty panels on launch.

**Why it happens:** Small positive news outlets change their CMS, switch domains, or deprecate FeedBurner URLs without notice. URLs from secondary sources (Feedspot, Reddit lists) are often stale.

**How to avoid:** Test every feed URL with a direct HTTP request before committing it to the config. Verified feeds in this research:
- Good News Network (`goodnewsnetwork.org/feed/`) -- **VERIFIED, valid RSS 2.0**
- Positive.News (`positive.news/feed/`) -- **VERIFIED, valid RSS 2.0**
- Reasons to be Cheerful (`reasonstobecheerful.world/feed/`) -- **VERIFIED, valid RSS 2.0**
- Optimist Daily (`optimistdaily.com/feed/`) -- **VERIFIED, valid RSS 2.0**
- GNN category feeds (`/category/news/{cat}/feed/`) -- **VERIFIED for science, animals, health**
- SunnySkyz -- **NOT VERIFIED, no standard RSS URL found** (see Open Questions)
- The Better India -- **NOT VERIFIED, `thebetterindia.com/rss` returns 403** (may need Railway proxy)
- Future Crunch -- **NOT VERIFIED, `futurecrunch.com/feed/` returns 404** (newsletter-only?)

**Warning signs:** Feed fetch failures in browser console immediately after adding a new source.

### Pitfall 2: GDELT Tone Scores Are Coarse-Grained

**What goes wrong:** Setting `tone>5` returns fewer articles than expected because GDELT's tone scoring assigns scores to entire articles, not headlines. Many genuinely positive articles get tone scores between 2-5 (mildly positive) and are filtered out. The result is a thin trickle of GDELT content.

**Why it happens:** GDELT's tone is computed from the full article text using the Hu and Liu 2004 opinion lexicon. An article about a scientific breakthrough that also discusses challenges in the field gets a moderate tone score (e.g., 3.5), not a strongly positive one (>5). The threshold `tone>5` is conservative.

**How to avoid:** Start with `tone>3` as an initial threshold. Test volume at different thresholds (3, 4, 5) during implementation. Make the threshold configurable in the client-side call so it can be tuned without a proto change. Use `sort=ToneDesc` to get the most positive articles first regardless of threshold.

**Warning signs:** GDELT positive queries returning fewer than 5 articles per 24-hour window.

### Pitfall 3: Content Category Keyword Overlap

**What goes wrong:** A story about "solar-powered robots saving endangered turtles" matches keywords for climate-wins, innovation-tech, AND nature-wildlife. The classifier picks whichever keyword appears first in the iteration order, producing inconsistent categorization.

**Why it happens:** Positive news stories are often cross-cutting. The six categories are not mutually exclusive in practice. The keyword-first-match approach from `threat-classifier.ts` was designed for threat severity levels (hierarchical), not topic categories (orthogonal).

**How to avoid:** Use a priority-ordered keyword scan where primary keywords (most specific) are checked first. For example, "endangered species" should match nature-wildlife before "robot" matches innovation-tech. Alternatively, implement multi-category tagging (a story can have primary + secondary categories). For Phase 2, single-category assignment with ordered keywords is sufficient -- Phase 3 can add multi-category support.

**Warning signs:** Browsing a single category shows stories that obviously belong elsewhere.

### Pitfall 4: Empty Happy Variant After Phase 2

**What goes wrong:** Phase 2 adds feeds and categories to the config, but the happy variant dashboard still shows empty panels because no panel is wired to consume the feeds yet (panel wiring is Phase 3).

**Why it happens:** Phase 2 is a data pipeline phase. It defines the feeds and classification but does not create the UI to display them. The `live-news` panel in the happy variant needs to be wired to consume the positive feeds, but that wiring may not exist until Phase 3.

**How to avoid:** Ensure the existing `LiveNewsPanel` (which is already enabled in the happy variant config as `'live-news': { name: 'Good News', enabled: true }`) can consume the positive feeds. The existing panel reads from `fetchCategoryFeeds()` which takes a `Feed[]` array. If the happy variant's data loading code passes the positive feeds to this function, the panel will render stories immediately after Phase 2. This wiring should be included in Phase 2, not deferred to Phase 3.

**Warning signs:** Deploying Phase 2 to happy.worldmonitor.app and seeing empty news panels.

## Code Examples

### Example 1: Extending the GDELT Proto

```protobuf
// proto/worldmonitor/intelligence/v1/search_gdelt_documents.proto
// Add two new fields to the request message:

message SearchGdeltDocumentsRequest {
  string query = 1 [
    (buf.validate.field).required = true,
    (buf.validate.field).string.min_len = 1
  ];
  int32 max_records = 2 [
    (buf.validate.field).int32.gte = 1,
    (buf.validate.field).int32.lte = 250
  ];
  string timespan = 3;
  // Tone filter appended to query (e.g., "tone>5" for positive, "tone<-5" for negative).
  // Left empty to skip tone filtering.
  string tone_filter = 4;
  // Sort mode: "DateDesc" (default), "ToneDesc", "ToneAsc", "HybridRel".
  string sort = 5;
}
```

Source: Existing proto at `proto/worldmonitor/intelligence/v1/search_gdelt_documents.proto`. New fields are additive (backward compatible).

### Example 2: Handler Modification for Tone Filter

```typescript
// server/worldmonitor/intelligence/v1/search-gdelt-documents.ts
// Inside searchGdeltDocuments():

let query = req.query;
if (req.toneFilter) {
  query = `${query} ${req.toneFilter}`;
}

const gdeltUrl = new URL(GDELT_DOC_API);
gdeltUrl.searchParams.set('query', query);
gdeltUrl.searchParams.set('mode', 'artlist');
gdeltUrl.searchParams.set('maxrecords', maxRecords.toString());
gdeltUrl.searchParams.set('format', 'json');
gdeltUrl.searchParams.set('sort', req.sort || 'date');
gdeltUrl.searchParams.set('timespan', timespan);
```

Source: Existing handler at `server/worldmonitor/intelligence/v1/search-gdelt-documents.ts`.

### Example 3: Positive GDELT Topics for Happy Variant

```typescript
// src/services/gdelt-intel.ts -- new constant for happy variant topics

export const POSITIVE_GDELT_TOPICS: IntelTopic[] = [
  {
    id: 'science-breakthroughs',
    name: 'Science Breakthroughs',
    query: '(breakthrough OR discovery OR "new treatment" OR "clinical trial success") sourcelang:eng',
    icon: '', // happy variant uses warm category badges, not emojis
    description: 'Scientific discoveries and medical advances',
  },
  {
    id: 'climate-progress',
    name: 'Climate Progress',
    query: '(renewable energy record OR "solar installation" OR "wind farm" OR "emissions decline" OR "green hydrogen") sourcelang:eng',
    icon: '',
    description: 'Renewable energy milestones and climate wins',
  },
  {
    id: 'conservation-wins',
    name: 'Conservation Wins',
    query: '(species recovery OR "population rebound" OR "conservation success" OR "habitat restored" OR "marine sanctuary") sourcelang:eng',
    icon: '',
    description: 'Wildlife recovery and habitat restoration',
  },
  {
    id: 'humanitarian-progress',
    name: 'Humanitarian Progress',
    query: '(poverty decline OR "literacy rate" OR "vaccination campaign" OR "peace agreement" OR "humanitarian aid") sourcelang:eng',
    icon: '',
    description: 'Poverty reduction, education, and peace',
  },
  {
    id: 'innovation',
    name: 'Innovation',
    query: '("clean technology" OR "AI healthcare" OR "3D printing" OR "electric vehicle" OR "fusion energy") sourcelang:eng',
    icon: '',
    description: 'Technology for good and clean innovation',
  },
];
```

### Example 4: Extending NewsItem Type for Content Category

```typescript
// src/types/index.ts -- extend NewsItem with optional category field

export interface NewsItem {
  source: string;
  title: string;
  link: string;
  pubDate: Date;
  isAlert: boolean;
  monitorColor?: string;
  tier?: number;
  threat?: import('@/services/threat-classifier').ThreatClassification;
  lat?: number;
  lon?: number;
  locationName?: string;
  lang?: string;
  // Happy variant: positive content category
  happyCategory?: import('@/services/positive-classifier').HappyContentCategory;
}
```

Source: Existing type at `src/types/index.ts` line 15-28.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| FeedBurner for RSS distribution | Direct WordPress/CMS RSS feeds | Google deprecated FeedBurner features 2021-2023 | SunnySkyz FeedBurner URL (`feeds.feedburner.com/SunnySkyz`) may be unreliable. Prefer direct site feeds. |
| GDELT v1 API | GDELT DOC 2.0 API | 2017+ | v2 supports tone filtering, full-text search, richer metadata. Already used in the codebase. |
| `@xenova/transformers` | `@huggingface/transformers` v3 | Late 2024 | Not relevant for Phase 2 (no ML needed), but noteworthy for Phase 3 ML sentiment filtering. |

**Deprecated/outdated:**
- FeedBurner feed URLs: May still work but Google has progressively deprecated features. Prefer direct RSS URLs from the source site.

## Open Questions

1. **SunnySkyz RSS feed URL**
   - What we know: SunnySkyz.com exists and publishes positive news. Historical references mention `feeds.feedburner.com/SunnySkyz` but FeedBurner is unreliable.
   - What's unclear: Whether SunnySkyz still offers a working RSS feed. No standard feed URL (`/feed/`, `/rss/`, `/feed.xml`) was found.
   - Recommendation: Try `https://www.sunnyskyz.com/good-news/rss` and `https://www.sunnyskyz.com/feed` at implementation time. If neither works, drop SunnySkyz from Phase 2 -- the other verified feeds provide sufficient volume. We already have 4 verified general sources + 4 GNN category feeds = 8 feeds, exceeding the "at least 5" success criterion.

2. **The Better India RSS access**
   - What we know: The site references `thebetterindia.com/rss` in its HTML metadata and `feeds.feedburner.com/TheBetterIndia` in older pages.
   - What's unclear: Direct RSS requests return 403, suggesting they may require a referrer header or the server blocks automated requests.
   - Recommendation: Try fetching via the Railway RSS proxy (which has a different IP and user-agent). If that fails, try `thebetterindia.com/feed/` (WordPress convention). If all fail, drop from Phase 2 -- the India-specific positive news angle is nice but not required for success criteria.

3. **Future Crunch RSS feed**
   - What we know: Future Crunch operates primarily as an email newsletter (Substack/Medium based). `futurecrunch.com/feed/` returns 404.
   - What's unclear: Whether they offer any RSS/Atom endpoint.
   - Recommendation: Check if they have a Substack at `futurecrunch.substack.com/feed`. Many newsletters publish via Substack which auto-generates RSS. If not available, drop from Phase 2.

4. **Optimal GDELT tone threshold**
   - What we know: `tone>5` is the commonly cited threshold for "fairly positive" content per GDELT's own blog post. The practical tone range is -10 to +10.
   - What's unclear: How many articles per day pass `tone>5` vs `tone>3` for general positive news queries. Volume may be too low at >5.
   - Recommendation: Implement with a configurable threshold passed as a parameter. Test at >3, >4, >5 during development. Default to >5 per the roadmap specification but allow easy adjustment.

5. **Happy variant feed wiring in App.ts**
   - What we know: The happy variant config defines `'live-news': { name: 'Good News', enabled: true, priority: 1 }`. The existing `LiveNewsPanel` displays feeds fetched by `fetchCategoryFeeds()`.
   - What's unclear: Whether the data loading path in `App.ts` already passes the happy variant's feeds to the panel, or if this wiring is missing.
   - Recommendation: Investigate the data loading code path in `App.ts` at implementation time. If the wiring is missing, Phase 2 should include it so the pipeline is end-to-end verifiable. An invisible pipeline is an unverifiable pipeline.

## Sources

### Primary (HIGH confidence)
- [GDELT DOC 2.0 API documentation](https://blog.gdeltproject.org/gdelt-doc-2-0-api-debuts/) -- tone filter syntax, sort parameters, query format
- [GDELT positive news blog post](https://blog.gdeltproject.org/finding-good-news-in-the-midst-of-the-pandemic-using-tone-filtering/) -- `tone>5` usage pattern
- Good News Network RSS (`goodnewsnetwork.org/feed/`) -- verified live, valid RSS 2.0 with category sub-feeds
- Positive.News RSS (`positive.news/feed/`) -- verified live, valid RSS 2.0, "Good journalism about good things"
- Reasons to be Cheerful RSS (`reasonstobecheerful.world/feed/`) -- verified live, valid RSS 2.0
- Optimist Daily RSS (`optimistdaily.com/feed/`) -- verified live, valid RSS 2.0, "Making Solutions the News"
- Existing codebase: `src/services/rss.ts` -- RSS proxy pipeline, DOMParser, caching, circuit breakers
- Existing codebase: `src/services/gdelt-intel.ts` -- GDELT Sebuf client, article mapping, caching
- Existing codebase: `src/services/threat-classifier.ts` -- keyword classification pattern (model for positive-classifier)
- Existing codebase: `src/config/variants/happy.ts` -- variant config with empty `FEEDS` placeholder
- Existing codebase: `proto/worldmonitor/intelligence/v1/search_gdelt_documents.proto` -- current request/response definitions
- Existing codebase: `server/worldmonitor/intelligence/v1/search-gdelt-documents.ts` -- GDELT handler implementation

### Secondary (MEDIUM confidence)
- [Good News Network RSS feeds page](https://www.goodnewsnetwork.org/more/rss-feeds/) -- lists category feeds
- [Feedspot Good News RSS Feeds list](https://rss.feedspot.com/good_news_rss_feeds/) -- 35 curated positive feed URLs
- The Better India -- site references RSS at `thebetterindia.com/rss` but returns 403 on direct access

### Tertiary (LOW confidence)
- SunnySkyz RSS -- no verified URL found. FeedBurner reference (`feeds.feedburner.com/SunnySkyz`) is from secondary sources and may be deprecated.
- Future Crunch RSS -- `futurecrunch.com/feed/` returns 404. May be newsletter-only.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all infrastructure exists, zero new dependencies
- Architecture: HIGH -- follows established patterns in the codebase (variant config, RSS pipeline, Sebuf handler, keyword classifier)
- Pitfalls: HIGH -- pitfalls are well-bounded (feed URL validation, tone threshold tuning, keyword overlap)
- Feed URLs: MEDIUM -- 5 of 7 required feeds verified; 2 need runtime validation; GNN category feeds provide backup

**Research date:** 2026-02-22
**Valid until:** 2026-04-22 (feeds and APIs are stable; RSS URLs may change)
