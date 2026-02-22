---
phase: 01-variant-shell-visual-foundation
plan: 03
subsystem: ui
tags: [maplibre, basemap, css, panels, empty-states, loading, variant-theme]

# Dependency graph
requires:
  - phase: 01-variant-shell-visual-foundation/01-01
    provides: "Variant registration, SITE_VARIANT config, build scripts"
  - phase: 01-variant-shell-visual-foundation/01-02
    provides: "Happy CSS theme with warm palette, semantic color variables, Nunito typography"
provides:
  - "Warm MapLibre basemap style JSONs (light + dark) for happy variant"
  - "Variant-aware basemap selection in DeckGLMap.ts"
  - "Panel chrome CSS overrides (rounded corners, shadows, empty states, loading)"
  - "Complete happy variant visual shell verified by user"
affects: [phase-2, phase-4, phase-5, phase-6, phase-7]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Self-hosted MapLibre style JSON with CARTO CDN tile sources"
    - "CSS @layer for theme cascade control (base-layer.css)"
    - "data-variant attribute bridging from SITE_VARIANT for CSS scoping"

key-files:
  created:
    - "public/map-styles/happy-light.json"
    - "public/map-styles/happy-dark.json"
    - "src/styles/base-layer.css"
  modified:
    - "src/components/DeckGLMap.ts"
    - "src/styles/happy-theme.css"
    - "src/config/variant.ts"
    - "src/main.ts"
    - "src/utils/theme-manager.ts"
    - "index.html"
    - "vite.config.ts"

key-decisions:
  - "CSS @layer base introduced to control cascade — happy-theme overrides sit outside @layer so they always win over :root defaults"
  - "data-variant attribute set on <html> element from SITE_VARIANT to enable pure-CSS variant scoping without JS"
  - "Theme toggle wired to swap DeckGLMap basemap style URL in addition to CSS theme switch"

patterns-established:
  - "MapLibre basemap per-variant: self-host style JSON in public/map-styles/, keep CARTO CDN for tile/sprite/glyph sources"
  - "CSS specificity: [data-variant='happy'] selectors for variant-scoped overrides"
  - "@layer base wrapping for main.css defaults to guarantee theme overrides win"

requirements-completed: [THEME-02, THEME-05]

# Metrics
duration: 113min
completed: 2026-02-22
---

# Phase 01 Plan 03: Map Basemap & Panel Chrome Summary

**Warm MapLibre basemap styles (sage land, light blue ocean) with rounded panel chrome, nature-themed empty states, and user-verified complete happy variant shell**

## Performance

- **Duration:** 113 min (includes CSS cascade debugging and human verification)
- **Started:** 2026-02-22T14:07:51Z
- **Completed:** 2026-02-22T16:01:19Z
- **Tasks:** 3
- **Files modified:** 11

## Accomplishments
- Created two self-hosted MapLibre style JSONs (light: sage land + cream bg + light blue ocean; dark: dark sage land + navy bg + dark navy ocean) preserving CARTO CDN tile sources
- Wired variant-aware basemap selection in DeckGLMap.ts so happy variant loads warm map styles
- Styled panel chrome with 14px rounded corners, subtle warm shadows, nature-themed sprout icon empty states, and gentle sage pulse loading animation
- Resolved CSS cascade issues: introduced @layer base and data-variant attribute bridging to ensure happy theme always overrides default :root variables
- User verified complete happy variant shell in both light and dark modes -- approved

## Task Commits

Each task was committed atomically:

1. **Task 1: Create warm basemap style JSONs and wire into DeckGLMap** - `a22a0d4` (feat)
2. **Task 2: Style panel chrome, empty states, and loading indicators** - `e9dce79` (feat)
   - Fix: `a86a902` - bridge SITE_VARIANT to data-variant attribute on html
   - Fix: `7b1f39e` - boost CSS specificity so happy theme wins over :root
   - Fix: `37a2627` - fix CSS cascade (import happy-theme after main.css)
   - Fix: `ce24790` - fix CSS cascade with @layer base and theme toggle
3. **Task 3: Visual verification of complete happy variant shell** - user-approved checkpoint (no code commit)

## Files Created/Modified
- `public/map-styles/happy-light.json` - MapLibre style JSON with sage land, cream background, light blue ocean
- `public/map-styles/happy-dark.json` - MapLibre style JSON with dark sage land, navy background, dark navy ocean
- `src/components/DeckGLMap.ts` - Variant-aware basemap URL selection (happy vs default)
- `src/styles/happy-theme.css` - Panel chrome overrides, empty states, loading animations for happy variant
- `src/styles/base-layer.css` - @layer base wrapper for CSS cascade control
- `src/config/variant.ts` - Extended variant bridging to data-variant attribute
- `src/main.ts` - Import order fix (happy-theme after main.css)
- `src/utils/theme-manager.ts` - Theme toggle wired to swap basemap style
- `index.html` - data-variant attribute support in inline script
- `vite.config.ts` - base-layer.css registration

## Decisions Made
- **CSS @layer base for cascade control:** Wrapping main.css defaults in @layer base ensures happy-theme.css overrides always win without !important escalation. This is a clean CSS architecture pattern.
- **data-variant attribute on html:** Bridging the JS SITE_VARIANT value to a DOM attribute enables pure-CSS variant scoping with [data-variant="happy"] selectors, avoiding runtime JS for styling.
- **Theme toggle basemap swap:** The theme manager now dispatches basemap URL changes to DeckGLMap when toggling light/dark, so the map colors match the mode.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] CSS cascade: happy theme variables not overriding :root defaults**
- **Found during:** Task 2 (panel chrome styling)
- **Issue:** Happy theme CSS custom properties set on [data-variant="happy"] were not overriding the :root defaults in main.css due to CSS specificity rules
- **Fix:** Introduced @layer base wrapping (base-layer.css) for main.css defaults, ensuring happy-theme.css (outside @layer) always wins. Also bridged SITE_VARIANT to data-variant HTML attribute for CSS scoping.
- **Files modified:** src/styles/base-layer.css (created), src/styles/main.css, src/main.ts, src/config/variant.ts, index.html, vite.config.ts
- **Verification:** Happy variant renders warm cream background and sage colors; default variant unchanged
- **Committed across:** a86a902, 7b1f39e, 37a2627, ce24790

**2. [Rule 3 - Blocking] Theme toggle not updating map basemap style**
- **Found during:** Task 2 verification
- **Issue:** Toggling light/dark mode changed CSS but the MapLibre map retained the previous basemap style
- **Fix:** Wired theme-manager.ts to dispatch basemap style URL changes when theme toggles
- **Files modified:** src/utils/theme-manager.ts
- **Committed in:** ce24790

---

**Total deviations:** 2 auto-fixed (both Rule 3 - blocking issues)
**Impact on plan:** Both fixes were essential for the happy variant to render correctly. CSS cascade fix was a structural issue that would have affected all future variant styling. No scope creep.

## Issues Encountered
- CSS specificity war between :root variables and [data-variant] selectors required architectural solution (@layer base) rather than simple specificity bumps. The initial attempt at boosting specificity (7b1f39e) was insufficient; the @layer approach (ce24790) was the definitive fix.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- Phase 1 is now COMPLETE: variant registration, warm theme, basemap, panel chrome all verified
- The happy variant shell is ready for content panels in Phase 2+
- All CSS variable infrastructure established for future panel styling
- MapLibre basemap pattern can be extended with additional overlay layers in Phase 4/8
- No blockers for Phase 2 (Curated Content Pipeline)

## Self-Check: PASSED

All 10 key files verified present on disk. All 6 task/fix commits verified in git history.

---
*Phase: 01-variant-shell-visual-foundation*
*Completed: 2026-02-22*
