# Requirements: HappyMonitor

**Defined:** 2026-02-22
**Core Value:** Every piece of content on the dashboard makes the viewer feel genuinely better about humanity

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Variant Infrastructure

- [ ] **INFRA-01**: Happy variant config (`VITE_VARIANT=happy`) registered in variant architecture with panel/feed/layer definitions
- [ ] **INFRA-02**: Subdomain routing for happy.worldmonitor.app via Vercel config
- [ ] **INFRA-03**: Variant-specific metadata (title, description, OG tags, favicon) for happy subdomain

### Visual Theme

- [ ] **THEME-01**: Warm & bright CSS theme with custom color palette (warm greens, golds, soft blues), replacing dark military aesthetic
- [ ] **THEME-02**: Warm map basemap style (green landmass, light ocean, gold accents) replacing dark military map
- [ ] **THEME-03**: Typography and spacing adjustments for welcoming, readable feel (consider sans-serif for happy variant)
- [ ] **THEME-04**: Positive-semantic color system replacing threat/severity colors (celebration gold, growth green, hope blue, kindness pink)
- [ ] **THEME-05**: Happy counterparts for all existing dark-variant UI elements (panel headers, status indicators, loading states, empty states)
- [ ] **THEME-06**: Celebration animations via canvas-confetti for milestone moments (species recovery announced, renewable energy record, etc.)

### Content Pipeline

- [ ] **FEED-01**: Dedicated positive news RSS feeds integrated (Good News Network, Positive.News, Reasons to be Cheerful, Optimist Daily, SunnySkyz, The Better India, Future Crunch)
- [ ] **FEED-02**: AI sentiment filter using existing DistilBERT-SST2 to gate mixed-source feeds, only passing positive stories (score >= 0.85 threshold)
- [ ] **FEED-03**: GDELT positive tone filter — extend existing GDELT integration with `tone>5` parameter for positive global news
- [ ] **FEED-04**: Content categories defined and mapped: Science & Health, Nature & Wildlife, Humanity & Kindness, Innovation & Tech, Climate Wins, Culture & Community
- [ ] **FEED-05**: Multi-stage content quality pipeline: curated sources as primary, sentiment-filtered mainstream as supplement, confidence threshold to minimize false positives

### Positive News Feed Panel

- [ ] **NEWS-01**: Scrolling real-time positive news feed panel showing curated uplifting stories with title, source, image, and category
- [ ] **NEWS-02**: Category filtering within the news feed (user can filter by Science, Nature, Humanity, etc.)
- [ ] **NEWS-03**: Auto-refresh on configurable interval (same real-time feel as WorldMonitor)

### Global Map & Positive Events

- [ ] **MAP-01**: Interactive map displaying geocoded positive events with warm-colored markers (green/gold pulses instead of red threat markers)
- [ ] **MAP-02**: Positive event layer showing news stories geolocated on the map
- [ ] **MAP-03**: World happiness heatmap choropleth layer (World Happiness Report data, green = happy)
- [ ] **MAP-04**: Species recovery zones as map overlay (IUCN data, wildlife comeback locations)
- [ ] **MAP-05**: Renewable energy installation locations as map layer

### Live Humanity Counters Panel

- [ ] **COUNT-01**: Worldometer-style ticking counters for positive metrics: babies born today, trees planted, vaccines administered, students graduated, books published, renewable MW installed
- [ ] **COUNT-02**: Per-second rate calculation from latest annual UN/WHO/World Bank data (no live API needed)
- [ ] **COUNT-03**: Smooth animated number transitions (always-moving, hypnotic feel)

### Humanity Progress Charts Panel

- [ ] **PROG-01**: Long-term trend charts showing human progress: extreme poverty declining, literacy rising, child mortality dropping, life expectancy increasing
- [ ] **PROG-02**: Data sourced from Our World in Data REST API and World Bank Indicators API
- [ ] **PROG-03**: D3.js sparkline/area chart visualizations with warm color palette

### Scientific Breakthroughs Ticker

- [ ] **SCI-01**: Horizontal scrolling ticker of recent scientific discoveries and medical advances
- [ ] **SCI-02**: RSS feeds from ScienceDaily, Nature, Science, Live Science filtered for breakthrough/positive content

### Today's Best Human Spotlight

- [ ] **HERO-01**: Daily featured story card of an individual doing extraordinary good (hero, inventor, rescuer, volunteer)
- [ ] **HERO-02**: Full-width card with photo, story excerpt, and location shown on map
- [ ] **HERO-03**: Curated from GNN Heroes feed and editorial picks

### Daily Digest / "5 Good Things"

- [ ] **DIGEST-01**: Summary panel showing 5 curated top positive stories of the day in 50 words or less each
- [ ] **DIGEST-02**: AI summarization via existing Flan-T5 model in ML worker

### Species Comeback Tracker Panel

- [ ] **SPECIES-01**: Wildlife conservation wins displayed as cards with species photo, population trend sparkline, and recovery status badge
- [ ] **SPECIES-02**: Data sourced from IUCN Red List data and conservation reports
- [ ] **SPECIES-03**: Monthly update cadence with historical population trend data

### Renewable Energy Growth Tracker Panel

- [ ] **ENERGY-01**: Renewable energy capacity visualization showing solar/wind installations growing, coal plants closing
- [ ] **ENERGY-02**: Animated gauge showing global renewable percentage climbing plus regional breakdown
- [ ] **ENERGY-03**: Data from IEA Renewable Energy Progress Tracker and existing EIA API integration

### Live Kindness Map

- [ ] **KIND-01**: Animated map layer showing acts of kindness, donations, volunteer events geolocated worldwide with green pulses
- [ ] **KIND-02**: Hybrid data approach: estimated baseline pulses weighted by population density, overlaid with real events from GoFundMe trending, Change.org wins, etc.

### Shareable Story Cards

- [ ] **SHARE-01**: One-tap generation of branded image cards for sharing positive stories on social media
- [ ] **SHARE-02**: Canvas/SVG rendering with headline, category badge, warm gradient background, HappyMonitor branding
- [ ] **SHARE-03**: Export as PNG with watermark

### Ambient / TV Mode

- [ ] **TV-01**: Full-screen lean-back mode designed for TV/second monitor with auto-cycling between panels
- [ ] **TV-02**: Configurable panel rotation interval (30s-2min), suppressed interactive elements, larger typography
- [ ] **TV-03**: Subtle ambient animations (floating particles, gentle transitions) for warm background feel

## v2 Requirements

### Engagement & Retention

- **ENGAGE-01**: Weekly email digest of "5 Good Things" (requires email infrastructure)
- **ENGAGE-02**: User-customizable panel layout (choose which panels to show/hide)

### Data Expansion

- **DATA-01**: Multi-language positive news filtering (beyond English)
- **DATA-02**: Real-time social media positivity tracker
- **DATA-03**: Country-specific positive news deep-dives

## Out of Scope

| Feature | Reason |
|---------|--------|
| User-submitted good news | Moderation nightmare, quality collapse, spam risk |
| Comments / social features | Destroys dashboard experience, moderation burden |
| Gamification / streaks / points | Undermines authenticity, creates anxiety |
| Push notifications | Notification fatigue, interrupts rather than soothes |
| Personalized / algorithmic feed | Filter bubbles even in positive content, added complexity |
| Premium / paywall | Contradicts freely-accessible mission |
| Excessive animation / particle effects | Kills dashboard credibility (warm, not birthday party) |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| INFRA-01 | TBD | Pending |
| INFRA-02 | TBD | Pending |
| INFRA-03 | TBD | Pending |
| THEME-01 | TBD | Pending |
| THEME-02 | TBD | Pending |
| THEME-03 | TBD | Pending |
| THEME-04 | TBD | Pending |
| THEME-05 | TBD | Pending |
| THEME-06 | TBD | Pending |
| FEED-01 | TBD | Pending |
| FEED-02 | TBD | Pending |
| FEED-03 | TBD | Pending |
| FEED-04 | TBD | Pending |
| FEED-05 | TBD | Pending |
| NEWS-01 | TBD | Pending |
| NEWS-02 | TBD | Pending |
| NEWS-03 | TBD | Pending |
| MAP-01 | TBD | Pending |
| MAP-02 | TBD | Pending |
| MAP-03 | TBD | Pending |
| MAP-04 | TBD | Pending |
| MAP-05 | TBD | Pending |
| COUNT-01 | TBD | Pending |
| COUNT-02 | TBD | Pending |
| COUNT-03 | TBD | Pending |
| PROG-01 | TBD | Pending |
| PROG-02 | TBD | Pending |
| PROG-03 | TBD | Pending |
| SCI-01 | TBD | Pending |
| SCI-02 | TBD | Pending |
| HERO-01 | TBD | Pending |
| HERO-02 | TBD | Pending |
| HERO-03 | TBD | Pending |
| DIGEST-01 | TBD | Pending |
| DIGEST-02 | TBD | Pending |
| SPECIES-01 | TBD | Pending |
| SPECIES-02 | TBD | Pending |
| SPECIES-03 | TBD | Pending |
| ENERGY-01 | TBD | Pending |
| ENERGY-02 | TBD | Pending |
| ENERGY-03 | TBD | Pending |
| KIND-01 | TBD | Pending |
| KIND-02 | TBD | Pending |
| SHARE-01 | TBD | Pending |
| SHARE-02 | TBD | Pending |
| SHARE-03 | TBD | Pending |
| TV-01 | TBD | Pending |
| TV-02 | TBD | Pending |
| TV-03 | TBD | Pending |

**Coverage:**
- v1 requirements: 48 total
- Mapped to phases: 0
- Unmapped: 48

---
*Requirements defined: 2026-02-22*
*Last updated: 2026-02-22 after initial definition*
