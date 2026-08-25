# Fork Improvement Harvest — tiny-world-builder
_Scope: forks pushed in the last week (since 2026-05-23). Only forks genuinely AHEAD of upstream were analyzed: limudim972/main (168 commits ahead, merge-base 2026-05-14) and yuxiaoli/develop (5 commits ahead, merge-base 2026-05-28). Other 10 recently-touched forks had zero original commits._

## TL;DR

| Fix | Fork | Status | Effort | Current module |
|---|---|---|---|---|
| Validate cell extras/transform + landscape/planetLandscape fields in `validateWorld()` | yuxiaoli | liftable | small | engine/world/26-ai-generation.js |
| Load world from `?world=` URL param (inline JSON + remote fetch) | yuxiaoli | liftable | medium | engine/world/29-persistence-api.js + 30-ui-boot-wiring.js |
| `touch-action: none` on `.minimap-wrap` (fixes touch-drag on mobile) | limudim972 | liftable | trivial | styles/tiny-world.css (~2814) |
| `publish.sh` copy `data/` into `dist/data` | yuxiaoli | liftable | trivial | publish.sh (root) |
| Crowd walk-trail stroke renderer + visibility toggle | limudim972 | needs-investigation | medium | 17-tile-renderers.js, 25-animation-loop-schema.js, 11-vehicle-crowd.js |
| Crop-duster/banner camera-relative flight refactor | limudim972 | needs-investigation | medium | engine/world/24-crop-duster-banners.js |
| Ambient route anti-repeat/anti-loop (concept only) | limudim972 | needs-investigation | medium | engine/world/11-vehicle-crowd.js |
| House-edit long-press (vs repeat-click floor removal) | limudim972 | needs-investigation | small | engine/world/20-input-place-erase.js |
| Center modals vertically (shared `.modal` rule) | limudim972 | needs-investigation | trivial | styles/tiny-world.css (~2402) |
| CRLF normalize in tools/check.js | yuxiaoli | needs-investigation | trivial | tools/check.js |
| Pinch-to-zoom on touch | limudim972 | already-done | — | engine/world/20-input-place-erase.js |
| Draggable minimap on touch (JS) | limudim972 | already-done | — | engine/world/30-ui-boot-wiring.js |
| Multi-file schema split | yuxiaoli | already-done | — | world.schema.json |
| Toolbar-zoom-panel override removal | limudim972 | already-done | — | styles/tiny-world.css |
| Mobile minimap position override removal | limudim972 | already-done | — | engine/world/30-ui-boot-wiring.js |
| **Remove model-stamp from schema + okKind** | yuxiaoli | **DO NOT LIFT** | — | world.schema.json + 26-ai-generation.js |
| House-aware crowd routing subsystem (~1900 lines) | limudim972 | do-not-lift* | large | engine/world/11-vehicle-crowd.js |
| Rain house-shelter routing | limudim972 | do-not-lift* | large | engine/world/11-vehicle-crowd.js |
| Per-person crowd click-vs-drag | limudim972 | do-not-lift* | medium | 20-input-place-erase.js / 11-vehicle-crowd.js |
| Mobile toolbar redesign (hamburger/grid) | limudim972 | do-not-lift | large | 19-tools-toolbar.js + CSS |
| Banner flight camera-relative + Hebrew edge banners | limudim972 | do-not-lift | medium | 24-crop-duster-banners.js / 13-distant-dressing-ghost.js |
| Modal z-index 50→200; Hebrew welcome modal copy | limudim972 | do-not-lift | trivial | styles/tiny-world.css |
| Fork stacked-map "numbers" minimap fixes | limudim972 | do-not-lift | medium | (no counterpart) |
| publish.sh → scripts/ rename + ROOT fix (mojibake) | yuxiaoli | do-not-lift | small | publish.sh, package.json, tools/check.js |
| Deploy /crowd caching + check.js monolithic-HTML markers | limudim972 | do-not-lift | small | netlify.toml, publish.sh, tools/check.js |

_*marked do-not-lift in practice: architecturally incompatible with this repo's path-cell-only ambient crowd; would require porting the whole free-space/envelope model first._

## ✅ Recommended to lift (status: liftable)

### Schema validation (yuxiaoli)
**Enhance `validateWorld()` with per-cell extras/transform + landscape checks** — `cfa5165`. Edit `engine/world/26-ai-generation.js` (`validateWorld`, ~line 301-392). Effort: small. Net-new defensive validation for fields the schema already documents (extras as `{kind|k}` in fence/tuft; transform as len-3/4 array or `{rotationY,offsetX}`; top-level `landscape*`/`planetLandscape.*`). Porting notes: current code destructures cells WITHOUT extras/transform (line 365 `[...fenceSide, , , appearance]`, line 367 omits them) — you must add extras/transform to both the tuple and object destructuring. **Keep this repo's existing `okKind` that INCLUDES `model-stamp` and `blank-island`; do NOT take the fork's trimmed okKind** (see Do-Not-Lift). Low coupling, contained to one function.

### URL loading + publish (yuxiaoli)
**Load world from `?world=` URL param** — `60d6e89` (inline-JSON branch) + `b8c3364` (remote-fetch branch). Edit `engine/world/29-persistence-api.js` (`loadState`, ~line 882) and the boot caller in `30-ui-boot-wiring.js` (~line 593). Effort: medium. Confirmed ABSENT (repo only has `?planet=`/`?perf=`). Two real hazards to adapt rather than copy verbatim: (1) the fork's fetch branch calls `loadInitialScene()` directly, but that lives in `30-ui-boot-wiring.js`, a different module from `loadState` — not reachable as written; (2) the async fetch returns `true` synchronously while data still loads, and the boot caller treats `loadState()===true` as "skip initial scene", so restructure the async/fallback (callback or sentinel return). `applyState`/`resetCameraDefaults` are in scope and graft cleanly. **Security: `fetch()` of an arbitrary query-param URL is an SSRF/exfiltration surface — sanity-check/allowlist before shipping.**

**publish.sh copy `data/` into `dist/data`** — `b8c3364`. Edit root `publish.sh`. Effort: trivial. Self-contained `mkdir + find/cp` block, appends cleanly. Only meaningful if URL-loading is lifted AND a `data/` folder of world JSON is added (none exists today) — couple it to the URL feature.

### Touch input (limudim972)
**`touch-action: none` on `.minimap-wrap`** — `d0e1464`. Edit `styles/tiny-world.css` (~line 2814-2825). Effort: trivial. The only genuinely missing piece across both touch commits: `.minimap-wrap` has `cursor:grab` but no `touch-action:none`, and no wildcard/parent rule covers it, so mobile touch-drag may be hijacked as browser scroll before the pointer handlers run. One line, zero coupling, no conflict with current uncommitted changes. Optionally also add the `e.button !== 0` pointerdown guard (cheap correctness).

## 🔍 Needs investigation

### Crowd walk-trail rendering (limudim972) — vendor-coupling is the key question
**Persistent red walk-trail stroke + "Show red trail" toggle** — lift `052e399` (canonical stroke version; supersedes the earlier `ec769a4` box-marks) and `228b4db` (toggle). Touches `17-tile-renderers.js` (segment mesh + `renderCellTile` refresh + `disposeGroup`), `25-animation-loop-schema.js` (`syncCrowdTrailSegments`), `11-vehicle-crowd.js` (person walking iteration). Effort: medium. **Critical coupling to verify:** the fork's walking-guard reads person fields that DO NOT exist in this repo's `vendor/tiny-crowd-layer.js` (`falling`, `hoverPaused`, `routeHold`, `draggedCrowdPersonId`) — this repo's person objects expose only `route/speed/speedMul/x/z/id`. Rewrite the guard as `person.route.length>1 && person.speed` (vendor layer ~line 621). Also: `crowdClamp` helper does not exist here (inline it); wire new clear-trail calls into `loadInitialScene` (18-scene-pick-xr.js) and the persistence load path (29-persistence-api.js); reconcile the fork's `RENDER_SETTINGS_VERSION` bump (fork used 18) with this repo's own version; verify visual layering against `TOP_H`/`terrainRiseAt`. Toggle is meaningless without the meshes — lift together or not at all.

### Crop-duster / banner flight (limudim972) — design decision first
**Make flyover camera-relative** — `ee9c2fb` (+ `e2dfd94` as a follow-on). Edit `24-crop-duster-banners.js`. Effort: medium. Current module still has the OLD axis-locked code (`planDustingCurve`, `cropDusterPassAxis`, lines 586-641), so NOT already-done — but this repo deliberately added `planBannerCurve()` (lines 656-668) flying banners ~2 grid widths behind the island to keep them off the build area. The fork's refactor collapses `planFlyoverCurve` into `planViewPassCurve` and **drops banner-behind-island**. Decide whether dropping that intentional behavior is acceptable before porting; `e2dfd94` only patches the fork's new `planViewPassCurve` and is meaningless standalone.

### Ambient route quality (limudim972) — port the concept, not the code
**Anti-repeat / anti-loop route dedup** — concepts from `1486ee3`/`92719cb`/`09c80d6` et al. Edit `11-vehicle-crowd.js` (`crowdRouteAround`). Effort: medium. The fork's code is envelope-dependent (`crowdConstrainRoute`/`crowdFreeSpaceRoute`/`crowdRandomHomePoint`) and will not drop in. But current `crowdRouteAround` uses `sort(()=>Math.random()-0.5)` which can repeat/loop — a standalone anti-repeat/anti-loop guard against the existing path-cell router is a worthwhile small follow-up. Reimplement the idea.

### House-edit long-press (limudim972)
**Long-press to remove house floor (vs accidental repeat-click)** — `c8d55b9`. Edit `20-input-place-erase.js` (house edit/erase). Effort: small. Least-coupled candidate in the crowd theme — this is house-EDITING UX, not crowd routing. No `longPress`/`removeHouseFloor`/`queueHouseClick` exists here. Verify this repo's current house-removal interaction model before porting; themed only by the "long-press" keyword.

### Center modals vertically (limudim972)
**`align-items: center` + `padding: 20px` on `.modal`** — `c752186`. Edit `styles/tiny-world.css` (~line 2402). Effort: trivial. Applies to the SHARED `.modal` rule (no per-modal overrides exist), so it re-centers every modal. This repo's `flex-start` + 80px-top is an intentional layout choice — treat as a design preference, lift only if a designer wants centered modals.

### CRLF normalize in check.js (yuxiaoli)
**`.replace(/\r\n/g,'\n')` when reading HTML** — `60d6e89`. Edit `tools/check.js`. Effort: trivial. No evidence this repo has a CRLF problem and check.js was substantially refactored. Speculative — skip unless a real CRLF check failure surfaces.

## ⏭️ Already done on our side (skip)

- **Pinch-to-zoom on touch** (`b2be22f`) — this repo already has `activePointers`/`pinchPrevDist` plus a more advanced `dragMode='pinch'` state machine + `clampViewSize`; fork is a strict subset.
- **Draggable minimap on touch, JS side** (`d0e1464`) — `30-ui-boot-wiring.js` (lines 932-1020) has no `innerWidth<=700` guards, uses unified `MINIMAP_POS_KEY`, and is more advanced (relative pct storage). Only the CSS one-liner is missing (lifted above).
- **Multi-file schema split** (`60d6e89`) — this repo's single `world.schema.json` is a strict superset (islands/moorings, blank-island, model-stamp). The split is a stale subset.
- **`.toolbar-zoom-panel` override removal** (`fbbabe4`) — class doesn't exist here; end-state already holds.
- **Mobile minimap position override removal** (`cdb1930`) — no mobile/desktop split here already; cleaner.

## ⛔ Do NOT lift (would regress / conflict)

- **yuxiaoli — Remove `modelStampId` / `model-stamp` from schemas and `okKind`** (`b851211`). This is the clearest trap. The fork removed it because their build doesn't use model stamps. THIS repo has `09-model-stamp-loader.js` and uses `model-stamp`/`modelStampId` across 10+ modules (04, 17, 19, 20, 21, 26, 28, 29, 32). Lifting would regress a live feature and break round-tripping of saved worlds. When porting the GOOD validation commit `cfa5165`, explicitly keep the existing `okKind`.
- **yuxiaoli — publish.sh → scripts/ rename + `ROOT='${dir}/..'` fix** (`60d6e89`, `8d34f9a`). This repo keeps `publish.sh` at root (package.json `build='./publish.sh'`, netlify `command='./publish.sh'`). The ROOT fix is only correct after the move; lifting breaks the layout. These commits also introduce mojibake (corrupted emoji glyphs).
- **limudim972 — Mobile toolbar redesign** (`104d1bd`/`ae81bad`/`6d6052a`/`8ba8cc9`, `04d885d`). Self-reverting fork churn against a monolithic Hebrew `.toolbar` with classes this repo doesn't have. This repo ships a more mature programmatic toolbar (`tool-group-btn`, `brand-title-btn`, `toolbar-world-slot`). Lifting would regress it.
- **limudim972 — House-aware crowd routing + rain shelter + per-person drag** (`423c3d8`…`2e9a030`, `12ed403`, `67209f0`…). Architecturally incompatible: this repo routes only over `path` cells so houses are never approached. Adopting requires porting the entire free-space + envelope model (`crowdHouseEnvelope`, `crowdConstrainRoute`, `crowdDoorPoint`, etc.) — not isolated patches. Corner/doorway bugfixes (`6a068c2`/`eb5a176`) are meaningless without it.
- **limudim972 — Banner camera-relative refactor + Hebrew edge banners** (`f1e91b4`, banner half of the theme). `f1e91b4` is a divergent Hebrew new-feature (home-edge labels), not a fix. The camera-relative refactor conflicts with this repo's `planBannerCurve` design (see Needs-Investigation).
- **limudim972 — Modal z-index 50→200** (`d76833d`). This repo's toolbar is z-index:12; the modal already wins at 50. The fork bumped because its own toolbar was ≥50. No benefit here.
- **limudim972 — Welcome modal copy / RTL tweaks** (`41eb58d` et al.). Divergent Hebrew build; this repo has a richer English launcher with `.welcome-options`; backdrop values don't even map.
- **limudim972 — Stacked-map "numbers" minimap fixes** (`6ed6f83`, `db86441`). This repo replaced the DOM-grid minimap with a canvas redraw; there is no "numbers" view mode to fix.
- **limudim972 — Deploy /crowd caching + check.js monolithic markers** (`06762f9`, `204f152`, `b1db1ef`). Coupled to the fork's `/crowd` sprite runtime and old single-file WORLD_SCHEMA parser this repo refactored into modules. README deploy note (`0a20166`) is a fork-local operational instruction.

## Notes on architecture mismatch

The forks' fixes live in a monolithic `tiny-world-builder.html` (limudim972 also added a `/crowd` sprite runtime and Hebrew/RTL UI), whereas this repo has been refactored into modular `engine/world/*.js` with a different, generally more mature implementation of the same features. Lifting is therefore translation, not copy-paste: every candidate must be re-expressed in the right module, and several "fixes" target code paths this repo no longer has (path-cell-only ambient crowd vs. house-envelope routing; canvas minimap vs. DOM-grid; programmatic toolbar vs. monolithic `.toolbar`). The highest-value lifts are the ones that are genuinely additive and low-coupling — schema validation, `?world=` loading, and the minimap `touch-action` one-liner.
