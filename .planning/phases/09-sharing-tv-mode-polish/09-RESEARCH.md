# Phase 9: Sharing, TV Mode & Polish - Research

**Researched:** 2026-02-23
**Domain:** Share card generation (Canvas 2D), TV/ambient mode (Fullscreen API + panel cycling), celebration animations (canvas-confetti)
**Confidence:** HIGH

## Summary

Phase 9 has three distinct sub-domains: (1) branded share cards for positive stories, (2) a fullscreen TV/ambient mode with auto-cycling panels, and (3) celebration animations for milestone moments. All three are well-served by existing browser APIs and the single new dependency `canvas-confetti` that was already approved in the project roadmap.

The codebase already has a mature story-card rendering pipeline (`src/services/story-renderer.ts`) that uses Canvas 2D to generate PNG images for the geopolitical variant's country intelligence stories. The share infrastructure (`src/services/story-share.ts`, `src/components/StoryModal.ts`) already implements Web Share API with file sharing, clipboard fallback, and social media deep links. Phase 9 needs a **new happy-variant renderer** that produces warm, branded image cards from `NewsItem` data rather than the existing `StoryData` (which is CII/geopolitical focused). The existing fullscreen toggle in `App.ts` provides a starting point for TV mode.

**Primary recommendation:** Build the happy share card renderer as a new Canvas 2D function following the existing `renderStoryToCanvas` pattern. TV mode should be a new controller class that manages panel cycling using `setInterval`, fullscreen via the existing toggle, and CSS class-based typography/interactivity overrides. Canvas-confetti integrates as a single `npm install` with a thin wrapper service.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SHARE-01 | One-tap generation of branded image cards for sharing positive stories on social media | Canvas 2D renderer for `NewsItem` with warm gradient, branding. Add share button to `PositiveNewsFeedPanel` card markup. Use existing Web Share API pattern from `StoryModal.ts`. |
| SHARE-02 | Canvas/SVG rendering with headline, category badge, warm gradient background, HappyMonitor branding | Canvas 2D API (already proven in codebase). Warm gradient from happy-theme CSS vars. Category badge colors from `HAPPY_CATEGORY_LABELS` + existing CSS class mapping. |
| SHARE-03 | Export as PNG with watermark | `canvas.toDataURL('image/png')` / `canvas.toBlob()` already used in `story-renderer.ts`. Watermark = HappyMonitor logo + URL drawn on canvas. |
| TV-01 | Full-screen lean-back mode designed for TV/second monitor with auto-cycling between panels | Fullscreen API already wired in `App.ts`. New `TvModeController` manages panel cycling via `setInterval`. CSS class `[data-tv-mode]` on root element controls layout. |
| TV-02 | Configurable panel rotation interval (30s-2min), suppressed interactive elements, larger typography | `[data-tv-mode]` CSS overrides: scale up font sizes, hide interactive elements (resize handles, filter bars, buttons). Interval stored in `localStorage` with UI control. |
| TV-03 | Subtle ambient animations (floating particles, gentle transitions) for warm background feel | CSS-only floating particles via `@keyframes` + pseudo-elements. Panel transitions via CSS `opacity`/`transform` with `transition`. Keep lightweight per "warm, not birthday party" constraint. |
| THEME-06 | Celebration animations via canvas-confetti for milestone moments | `canvas-confetti@1.9.x` -- approved dependency. Thin wrapper service detects milestones from panel data updates and fires confetti. Respects `prefers-reduced-motion`. |
</phase_requirements>

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Canvas 2D API | Browser built-in | Share card image generation | Already proven in `story-renderer.ts`. No extra dependency. Full control over layout, gradients, typography. |
| Fullscreen API | Browser built-in | TV mode fullscreen | Already wired in `App.ts` via `toggleFullscreen()`. Cross-browser with `webkitRequestFullscreen` fallback. |
| Web Share API | Browser built-in | Native share sheet on mobile | Already implemented in `StoryModal.ts` with `navigator.share()` + `navigator.canShare()` + clipboard fallback. |
| `canvas-confetti` | `1.9.x` | Celebration animations | ~6KB gzipped. Framework-agnostic (no React). Web worker support via `useWorker: true`. Respects `prefers-reduced-motion`. Already approved in roadmap. |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| D3 (existing) | `7.9.0` | Color interpolation for share card gradients | Use `d3.interpolateWarm` or `d3.interpolateYlGn` for canvas gradient stops if needed. Already bundled. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Canvas 2D for share cards | `html-to-image` / `html2canvas` | DOM-to-image libraries can capture existing HTML but are fragile with CSS variables, cross-origin images, and pseudo-elements. Canvas 2D gives pixel-perfect control and the codebase already uses this pattern. |
| CSS-only ambient particles | `tsparticles` / `particles.js` | Heavy JS particle libraries are overkill. "Warm, not birthday party" constraint means CSS `@keyframes` pseudo-elements are sufficient. Keeps bundle size zero for this feature. |
| Canvas-confetti | `js-confetti` (emoji-based) | `js-confetti` uses emoji rendering which looks informal. `canvas-confetti` uses physics-based particle shapes that are more elegant. Already approved in roadmap. |

**Installation:**
```bash
npm install canvas-confetti
npm install -D @types/canvas-confetti
```

Note: `@types/canvas-confetti` provides TypeScript type definitions. The package itself ships as ESM + UMD.

## Architecture Patterns

### Recommended Project Structure
```
src/
  services/
    happy-share-renderer.ts   # Canvas 2D renderer for happy story cards
    celebration.ts             # canvas-confetti wrapper + milestone detection
    tv-mode.ts                 # TvModeController class
  styles/
    happy-theme.css            # TV mode CSS overrides appended here
    tv-mode.css                # (optional) separate file if CSS grows large
```

### Pattern 1: Happy Share Card Renderer
**What:** A Canvas 2D function that generates branded PNG cards from `NewsItem` data with warm gradients, category badges, and HappyMonitor branding.
**When to use:** When user taps share button on a positive news card.
**Example:**
```typescript
// src/services/happy-share-renderer.ts
import type { NewsItem } from '@/types';
import type { HappyContentCategory } from '@/services/positive-classifier';
import { HAPPY_CATEGORY_LABELS } from '@/services/positive-classifier';

const W = 1080;
const H = 1080; // Square for social media (Instagram-friendly)

const CATEGORY_COLORS: Record<HappyContentCategory, string> = {
  'science-health': '#7BA5C4',
  'nature-wildlife': '#6B8F5E',
  'humanity-kindness': '#C48B9F',
  'innovation-tech': '#C4A35A',
  'climate-wins': '#2d9a4e',
  'culture-community': '#8b5cf6',
};

// Warm gradient backgrounds per category
const GRADIENT_STOPS: Record<HappyContentCategory, [string, string]> = {
  'science-health': ['#E8F4FD', '#C5DFF8'],
  'nature-wildlife': ['#E8F5E4', '#C5E8BE'],
  'humanity-kindness': ['#FDE8EE', '#F5C5D5'],
  'innovation-tech': ['#FDF5E8', '#F5E2C0'],
  'climate-wins': ['#E4F5E8', '#BEE8C5'],
  'culture-community': ['#F0E8FD', '#D8C5F5'],
};

export async function renderHappyShareCard(item: NewsItem): Promise<HTMLCanvasElement> {
  const canvas = document.createElement('canvas');
  canvas.width = W;
  canvas.height = H;
  const ctx = canvas.getContext('2d')!;
  const category = item.happyCategory || 'humanity-kindness';
  const [gradTop, gradBottom] = GRADIENT_STOPS[category];

  // Warm gradient background
  const grad = ctx.createLinearGradient(0, 0, 0, H);
  grad.addColorStop(0, gradTop);
  grad.addColorStop(1, gradBottom);
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, W, H);

  // ... headline, category badge, branding, watermark
  return canvas;
}
```

### Pattern 2: TV Mode Controller
**What:** A class that manages fullscreen mode, panel cycling, and ambient animations. Attaches `data-tv-mode` to `<html>` for CSS targeting.
**When to use:** User clicks "TV Mode" button or uses keyboard shortcut.
**Example:**
```typescript
// src/services/tv-mode.ts
export class TvModeController {
  private intervalId: ReturnType<typeof setInterval> | null = null;
  private currentPanelIndex = 0;
  private panels: string[];
  private intervalMs: number;

  constructor(panels: string[], intervalMs = 60_000) {
    this.panels = panels;
    this.intervalMs = intervalMs;
  }

  enter(): void {
    document.documentElement.dataset.tvMode = 'true';
    document.documentElement.requestFullscreen?.().catch(() => {});
    this.showPanel(0);
    this.intervalId = setInterval(() => this.nextPanel(), this.intervalMs);
  }

  exit(): void {
    delete document.documentElement.dataset.tvMode;
    if (document.fullscreenElement) document.exitFullscreen?.().catch(() => {});
    if (this.intervalId) clearInterval(this.intervalId);
    this.intervalId = null;
    // Restore normal panel visibility
    this.showAllPanels();
  }

  setInterval(ms: number): void {
    this.intervalMs = ms;
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = setInterval(() => this.nextPanel(), this.intervalMs);
    }
  }

  private nextPanel(): void {
    this.currentPanelIndex = (this.currentPanelIndex + 1) % this.panels.length;
    this.showPanel(this.currentPanelIndex);
  }

  private showPanel(index: number): void {
    // Hide all panels, show only the current one with fade transition
    const grid = document.getElementById('panelsGrid');
    if (!grid) return;
    const panelEls = grid.querySelectorAll<HTMLElement>('.panel');
    panelEls.forEach((el, i) => {
      el.classList.toggle('tv-hidden', i !== index);
      el.classList.toggle('tv-active', i === index);
    });
  }

  private showAllPanels(): void {
    const grid = document.getElementById('panelsGrid');
    if (!grid) return;
    grid.querySelectorAll<HTMLElement>('.panel').forEach(el => {
      el.classList.remove('tv-hidden', 'tv-active');
    });
  }

  destroy(): void {
    this.exit();
  }
}
```

### Pattern 3: Celebration Service
**What:** A thin wrapper around `canvas-confetti` that fires on milestone data events with appropriate reduced-motion checks.
**When to use:** When panel data indicates a milestone (species recovery, renewable record, etc.).
**Example:**
```typescript
// src/services/celebration.ts
import confetti from 'canvas-confetti';

const REDUCED_MOTION = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

export function celebrate(type: 'milestone' | 'record' = 'milestone'): void {
  if (REDUCED_MOTION) return;

  const defaults = {
    particleCount: type === 'record' ? 80 : 40,
    spread: type === 'record' ? 90 : 60,
    origin: { y: 0.7 },
    colors: ['#6B8F5E', '#C4A35A', '#7BA5C4', '#8BAF7A'],
    disableForReducedMotion: true,
  };

  void confetti(defaults);
}
```

### Pattern 4: Share Button Integration in Panel Cards
**What:** Add a share icon to each positive news card that triggers the share card renderer.
**When to use:** On every `.positive-card` in `PositiveNewsFeedPanel`.
**Example:**
```typescript
// In renderCard() method of PositiveNewsFeedPanel
const shareBtn = `<button class="positive-card-share"
  data-title="${escapeHtml(item.title)}"
  data-source="${escapeHtml(item.source)}"
  data-category="${escapeHtml(item.happyCategory || '')}"
  aria-label="Share this story">
  <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M4 12v8a2 2 0 002 2h12a2 2 0 002-2v-8M16 6l-4-4-4 4M12 2v13"/>
  </svg>
</button>`;
```

### Anti-Patterns to Avoid
- **DOM-to-image for share cards:** Do NOT use `html2canvas` or `html-to-image`. They fail with CSS variables, cross-origin images, and pseudo-elements. The codebase already proves Canvas 2D works reliably for this purpose.
- **Heavy particle libraries for ambient mode:** Do NOT add `tsparticles` or `particles.js`. CSS `@keyframes` with pseudo-elements achieves the "subtle floating particles" requirement without any JS overhead.
- **Polling for milestone detection:** Do NOT poll panel data for milestones. Instead, fire celebration events from the data update paths in `App.ts` where panel data is already being processed (e.g., in `loadConservationData()`, `loadRenewableData()`).
- **Separate fullscreen implementation for TV mode:** Do NOT duplicate the fullscreen API code. Reuse the existing `toggleFullscreen()` method in App.ts or extract it to a shared utility.
- **Inline styles for TV mode:** Do NOT use JavaScript to set individual element styles for TV mode. Use a single `data-tv-mode` attribute on `<html>` and let CSS cascade handle all visual changes.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Confetti/celebration animations | Custom canvas particle system | `canvas-confetti` | Physics, performance, reduced-motion support, web worker offloading |
| PNG export from canvas | Manual blob conversion | `canvas.toBlob()` callback or `canvas.toDataURL('image/png')` | Browser-native, handles encoding correctly |
| Social share URLs | Custom URL construction per platform | Extend existing `getShareUrls()` in `story-share.ts` | Platform URL schemas change; existing code already handles Twitter, LinkedIn, WhatsApp, etc. |
| Fullscreen management | Custom fullscreen wrapper | Existing `toggleFullscreen()` in App.ts | Already handles vendor prefixes, error suppression |
| Floating particle animation | Canvas/WebGL particle system | CSS `@keyframes` + pseudo-elements | Zero JS overhead, GPU-accelerated, trivially pausable |

**Key insight:** Every sub-feature in Phase 9 has existing codebase precedent or a battle-tested tiny library. No greenfield architecture is needed.

## Common Pitfalls

### Pitfall 1: Canvas Font Loading Race
**What goes wrong:** Canvas 2D `fillText()` renders system font fallback if the web font (Nunito) hasn't loaded yet, producing ugly share cards.
**Why it happens:** `document.fonts.ready` may resolve before all font weights load. Canvas doesn't automatically wait for fonts.
**How to avoid:** Before rendering the share card canvas, explicitly await the Nunito font:
```typescript
await document.fonts.load('700 40px Nunito');
await document.fonts.load('400 24px Nunito');
```
**Warning signs:** Share cards using serif/system font instead of Nunito.

### Pitfall 2: Cross-Origin Image Tainting
**What goes wrong:** If the share card includes news item images (from RSS feeds), `canvas.toDataURL()` throws a security error because the canvas is "tainted" by cross-origin images.
**Why it happens:** RSS feed images come from third-party domains without CORS headers.
**How to avoid:** Either (a) don't include the news image in the share card (safest -- use gradient + text only), or (b) proxy images through the app's own API to add CORS headers, or (c) catch the error and generate a text-only card fallback.
**Warning signs:** `SecurityError: The operation is insecure` in console.

### Pitfall 3: Fullscreen Denied on Non-User-Gesture
**What goes wrong:** `requestFullscreen()` throws `NotAllowedError` if called without a user gesture (e.g., on page load or in a timer callback).
**Why it happens:** Browsers require fullscreen to be initiated from a trusted user event.
**How to avoid:** TV mode entry must always be triggered by a button click. The `setInterval` panel cycling runs after fullscreen is already entered. Already handled in Sentry ignore list.
**Warning signs:** `NotAllowedError: Fullscreen request denied` (already in Sentry ignoreErrors).

### Pitfall 4: TV Mode Panel Cycling Memory Leak
**What goes wrong:** If TV mode cycles panels but doesn't properly clean up hidden panel states, D3 charts and map layers accumulate memory.
**Why it happens:** Panels are hidden via CSS (`display: none` or `opacity: 0`) but their timers, observers, and animation frames continue running.
**How to avoid:** TV mode should NOT destroy/recreate panels. Instead, use CSS `visibility: hidden` + `opacity: 0` (keeps layout, prevents reflows). Panel timers already gate on `isAbortError()` checks via the base `Panel` class.
**Warning signs:** Gradually increasing memory in DevTools during TV mode.

### Pitfall 5: Canvas-Confetti Blocking Main Thread
**What goes wrong:** Without `useWorker: true`, confetti animation blocks main thread causing frame drops on the dashboard.
**Why it happens:** Default confetti uses requestAnimationFrame on main thread.
**How to avoid:** Always pass `useWorker: true` in the confetti options when calling from the dashboard context. Note: `useWorker` requires the confetti instance be created with `confetti.create()` and a specific canvas element.
**Warning signs:** Janky scrolling or panel updates when confetti fires.

### Pitfall 6: Share Card Size for Social Platforms
**What goes wrong:** Wrong aspect ratio causes poor rendering on Twitter, Instagram, or WhatsApp previews.
**Why it happens:** Each platform has different optimal image sizes.
**How to avoid:** Use 1080x1080 (square) as the default -- works well on Instagram, WhatsApp, and LinkedIn. Twitter prefers 1200x675 but gracefully handles square. The existing `story-renderer.ts` uses 1080x1920 (portrait) for its country stories.
**Warning signs:** Images cropped or letterboxed on social media previews.

## Code Examples

Verified patterns from existing codebase:

### Canvas-to-PNG Export (from `story-renderer.ts`)
```typescript
// Source: src/services/story-renderer.ts + src/components/StoryModal.ts
const canvas = await renderStoryToCanvas(data);
const dataUrl = canvas.toDataURL('image/png');

// Convert to Blob for Web Share API
const binStr = atob(dataUrl.split(',')[1] ?? '');
const bytes = new Uint8Array(binStr.length);
for (let i = 0; i < binStr.length; i++) bytes[i] = binStr.charCodeAt(i);
const blob = new Blob([bytes], { type: 'image/png' });
const file = new File([blob], 'happymonitor-story.png', { type: 'image/png' });
```

### Web Share API with Fallback (from `StoryModal.ts`)
```typescript
// Source: src/components/StoryModal.ts lines 128-147
if (navigator.share && navigator.canShare?.({ files: [file] })) {
  try {
    await navigator.share({
      text: 'Check out this positive story from HappyMonitor!',
      files: [file]
    });
    return;
  } catch { /* user cancelled */ }
}

// Fallback: copy to clipboard
try {
  await navigator.clipboard.write([
    new ClipboardItem({ 'image/png': blob }),
  ]);
} catch {
  // Final fallback: trigger download
  const a = document.createElement('a');
  a.href = dataUrl;
  a.download = 'happymonitor-story.png';
  a.click();
}
```

### Fullscreen Toggle (from `App.ts`)
```typescript
// Source: src/App.ts lines 2954-2962
private toggleFullscreen(): void {
  if (document.fullscreenElement) {
    try { void document.exitFullscreen()?.catch(() => {}); } catch {}
  } else {
    const el = document.documentElement as HTMLElement & { webkitRequestFullscreen?: () => void };
    if (el.requestFullscreen) {
      void el.requestFullscreen().catch(() => {});
    } else if (el.webkitRequestFullscreen) {
      el.webkitRequestFullscreen();
    }
  }
}
```

### Panel Visibility via CSS Class (existing pattern)
```typescript
// Source: src/components/Panel.ts lines 377-388
public show(): void {
  this.element.classList.remove('hidden');
}
public hide(): void {
  this.element.classList.add('hidden');
}
```

### Canvas 2D Warm Gradient (new, based on theme colors)
```typescript
// Derived from src/styles/happy-theme.css color variables
function drawWarmGradient(ctx: CanvasRenderingContext2D, w: number, h: number, category: HappyContentCategory): void {
  const grad = ctx.createLinearGradient(0, 0, w * 0.3, h);
  // Map category to warm tones from happy-theme.css
  const colorMap: Record<string, [string, string, string]> = {
    'science-health': ['#E8F4FD', '#C5DFF8', '#7BA5C4'],
    'nature-wildlife': ['#E8F5E4', '#C5E8BE', '#6B8F5E'],
    'humanity-kindness': ['#FDE8EE', '#F5C5D5', '#C48B9F'],
    'innovation-tech': ['#FDF5E8', '#F5E2C0', '#C4A35A'],
    'climate-wins': ['#E4F5E8', '#BEE8C5', '#2d9a4e'],
    'culture-community': ['#F0E8FD', '#D8C5F5', '#8b5cf6'],
  };
  const [top, bottom] = colorMap[category] || colorMap['humanity-kindness'];
  grad.addColorStop(0, top);
  grad.addColorStop(1, bottom);
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, w, h);
}
```

### Canvas-Confetti Basic Usage
```typescript
// Source: canvas-confetti npm documentation
import confetti from 'canvas-confetti';

// Simple burst
confetti({
  particleCount: 50,
  spread: 70,
  origin: { y: 0.6 },
  colors: ['#6B8F5E', '#C4A35A', '#7BA5C4'],
  disableForReducedMotion: true,
});

// Custom canvas (for TV mode overlay)
const myCanvas = document.createElement('canvas');
document.body.appendChild(myCanvas);
const myConfetti = confetti.create(myCanvas, {
  resize: true,
  useWorker: true, // Off main thread
});
myConfetti({ particleCount: 80, spread: 90 });
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `html2canvas` for DOM screenshots | Canvas 2D API direct drawing | Codebase convention | More reliable, no cross-origin issues, pixel-perfect control |
| `dom-to-image` | `html-to-image` (maintained fork) | 2023 | Better TypeScript support, but still not needed here |
| Custom particle systems | `canvas-confetti` | Stable since 2019 | 6KB, battle-tested, reduced-motion aware |
| Manual fullscreen vendor prefixes | Fullscreen API standard | Wide support since 2022 | Only `webkitRequestFullscreen` needed as fallback for Safari |

**Deprecated/outdated:**
- `dom-to-image`: Unmaintained since 2020. Use `html-to-image` if DOM capture is ever needed (not for this phase).
- Fullscreen API vendor prefixes: `mozRequestFullScreen` no longer needed. Only `webkitRequestFullscreen` for Safari fallback.

## Open Questions

1. **Share card image inclusion**
   - What we know: RSS news images come from third-party domains and will taint the canvas (cross-origin restriction). The existing `story-renderer.ts` does not include images.
   - What's unclear: Whether the user wants news article images in the share card, or if a text-only branded card is sufficient.
   - Recommendation: Start with gradient + text-only share cards (reliable, no CORS issues). If images are desired later, proxy them through the app's API.

2. **TV mode panel set**
   - What we know: The happy variant has 9 panels (map + 8 content panels). The map panel uses a different rendering system (MapLibre + Deck.gl).
   - What's unclear: Whether TV mode should include the map panel in its rotation, or only cycle through the 8 content panels.
   - Recommendation: Include the map as one of the cycling "slides" -- it provides visual variety. When the map is the active slide, center it on a region and let positive event markers be visible.

3. **Milestone data detection**
   - What we know: Species comeback data includes `status` badges (recovered, recovering, stabilized). Renewable energy data includes percentage values.
   - What's unclear: What specific thresholds constitute a "milestone" worth celebrating with confetti.
   - Recommendation: Define milestone events as: (a) species status changed to "recovered", (b) renewable energy percentage crosses a round number (30%, 35%, etc.), (c) a new species is added to the comeback list. Store "last celebrated" milestones in `sessionStorage` to avoid repeat celebrations on data refresh.

4. **TV mode entry point**
   - What we know: The header already has a fullscreen button (`#fullscreenBtn`). Settings modal has panel toggles.
   - What's unclear: Whether TV mode should be a separate button in the header, a settings modal option, or both.
   - Recommendation: Add a TV icon button next to the existing fullscreen button in the header (happy variant only). Also support keyboard shortcut (e.g., `Shift+T`).

## Sources

### Primary (HIGH confidence)
- `src/services/story-renderer.ts` -- Existing Canvas 2D PNG renderer pattern (487 lines)
- `src/components/StoryModal.ts` -- Existing Web Share API + clipboard + download fallback
- `src/services/story-share.ts` -- Social media share URL construction
- `src/components/Panel.ts` -- Panel base class with show/hide/destroy lifecycle
- `src/App.ts` -- Fullscreen toggle, panel creation, happy variant branching
- `src/styles/happy-theme.css` -- Complete happy theme CSS variables and component styles
- `src/config/panels.ts` -- HAPPY_PANELS configuration (9 panels)
- `src/services/positive-classifier.ts` -- Category labels and colors for share cards

### Secondary (MEDIUM confidence)
- [canvas-confetti npm](https://www.npmjs.com/package/canvas-confetti) -- API documentation, ~6KB gzipped, `useWorker` option
- [MDN Fullscreen API](https://developer.mozilla.org/en-US/docs/Web/API/Fullscreen_API) -- Standard reference
- [MDN Navigator.share()](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/share) -- Web Share API with file support
- [MDN Navigator.canShare()](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/canShare) -- Feature detection
- [canvas-confetti GitHub](https://github.com/catdad/canvas-confetti) -- Source, examples, `disableForReducedMotion` option

### Tertiary (LOW confidence)
- [Best HTML to Canvas Solutions in 2025](https://portalzine.de/best-html-to-canvas-solutions-in-2025/) -- Confirmed Canvas 2D is superior to html2canvas for controlled rendering
- [Dash0 TV Mode](https://www.dash0.com/changelog/tv-mode-for-dashboard-panels) -- Reference implementation of dashboard TV mode (auto-rotation + fullscreen)
- [Screenful TV Mode](https://screenful.com/guide/tv-mode) -- Dashboard TV mode with configurable rotation interval

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- Only one new dependency (`canvas-confetti`), everything else is browser-native or existing codebase patterns
- Architecture: HIGH -- All three sub-features have direct precedent in the existing codebase (`story-renderer.ts`, `toggleFullscreen()`, Panel lifecycle)
- Pitfalls: HIGH -- Based on known issues in existing code (canvas font loading, cross-origin tainting, fullscreen gesture requirement)

**Research date:** 2026-02-23
**Valid until:** 2026-03-23 (stable domain -- browser APIs and canvas-confetti rarely change)
