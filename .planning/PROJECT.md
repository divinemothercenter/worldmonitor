# HappyMonitor (happy.worldmonitor.app)

## What This Is

The optimistic counterpart to WorldMonitor — a live, real-time dashboard that shows only good news and uplifting data about humanity. Same addictive "stare at it all day" quality, but everything on screen makes you feel better about the world. A new subdomain variant within the existing WorldMonitor codebase, following the same pattern as finance.worldmonitor.app and tech.worldmonitor.app.

## Core Value

Every piece of content on the dashboard makes the viewer feel genuinely better about humanity — curated positive news, upward-trending metrics, and real-time acts of kindness happening worldwide.

## Requirements

### Validated

- ✓ Domain-driven micro-services architecture with Sebuf — existing
- ✓ Variant architecture supporting multiple subdomains (full/tech/finance) — existing
- ✓ Real-time data ingestion pipeline (RSS, APIs, WebSockets) — existing
- ✓ Interactive map with MapLibre GL and Deck.gl layers — existing
- ✓ Panel-based dashboard UI with React/TypeScript — existing
- ✓ Proto-generated typed clients and handlers — existing
- ✓ PWA with offline support — existing
- ✓ Internationalization (14 languages) — existing
- ✓ Desktop app via Tauri — existing
- ✓ ML/NLP pipeline with Transformers.js and ONNX — existing

### Active

- [ ] AI-powered positive sentiment filtering on existing news feeds
- [ ] Dedicated positive news source integration (Good News Network, Positive.News, Reasons to be Cheerful, etc.)
- [ ] Positive news feed panel — scrolling real-time curated uplifting stories
- [ ] Humanity metrics panel — charts showing long-term human progress (poverty declining, literacy rising, life expectancy up, child mortality down)
- [ ] Good deeds tracker panel — community kindness, donations, volunteer events worldwide
- [ ] Nature & wildlife panel — environmental recovery, species rebounds, conservation wins
- [ ] Warm & bright visual theme — departure from dark military aesthetic
- [ ] "happy" variant configuration in the variant architecture
- [ ] New subdomain routing for happy.worldmonitor.app
- [ ] Additional viral-worthy panels (scientific breakthroughs ticker, renewable energy growth tracker, "today's best human" spotlight, live kindness map)

### Out of Scope

- User-generated content / submissions — complexity, moderation burden
- Social features (comments, sharing within app) — keep it a dashboard, not a social platform
- Paid/premium tier — this should be freely accessible
- Native mobile app — web-first, PWA covers mobile

## Context

WorldMonitor is an established real-time global intelligence dashboard that went viral. It tracks 18 intelligence domains (aviation, military, seismology, climate, conflict, etc.) with a dark, command-center aesthetic. The existing variant architecture already supports multiple subdomain flavors (full, tech, finance) via environment configuration. The codebase has a mature Sebuf-based service layer with proto-generated clients, making it straightforward to add new domain handlers.

The existing ML pipeline (Transformers.js, ONNX Runtime) already runs in-browser, which can be leveraged for sentiment classification on incoming news feeds. The same WorldMonitor audience is the target — people who already watch the world's events and would appreciate a positive counterbalance.

The key creative challenge: making it feel as compelling and "watchable" as WorldMonitor's threat-tracking aesthetic, but for positive events. The dark military command center vibe needs to flip to warm, bright, and hopeful without feeling cheesy or naive.

## Constraints

- **Architecture**: Must follow existing variant pattern — same codebase, env-based configuration, no separate repo
- **Data sources**: Reuse existing feed infrastructure where possible, add positive-focused feeds
- **Performance**: Same real-time responsiveness as WorldMonitor — data should feel live
- **Visual identity**: Warm & bright theme, but must still feel like a serious dashboard (not a greeting card)
- **Deployment**: Same Vercel deployment pipeline, new subdomain config

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Same codebase, new variant | Follows established finance/tech pattern, shared infrastructure | — Pending |
| AI sentiment filtering | Leverage existing ML pipeline for positive story detection | — Pending |
| Warm & bright theme | Visual departure needed to signal different intent than WorldMonitor | — Pending |
| Reuse + extend data sources | Existing feeds provide volume, new positive sources provide curation quality | — Pending |

---
*Last updated: 2026-02-22 after initialization*
