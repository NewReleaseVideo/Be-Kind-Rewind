# New Release Wall — Conversation Context & Decision Log

This file documents the full build history of this project so a new Claude Code session can pick up exactly where the chat left off.

---

## Origin
Built entirely in a Claude.ai chat session. The user wanted a Blockbuster-themed "new release wall" PWA showing live streaming and rental movies via the TMDB API.

---

## Build History (Chronological)

### v1 — Initial Build
- Single `index.html` PWA with Blockbuster aesthetic (navy/amber/yellow, VHS stripe)
- Three shelves: New to Streaming, Top Rentals & VOD, Coming Soon
- TMDB API integration (discover + now_playing + upcoming endpoints)
- Basic Blockbuster Score algorithm for rental shelf ranking
- Provider dots on hover via JustWatch/TMDB watch providers
- Click → modal with tagline, runtime, providers, score breakdown
- Loading screen with animated progress bar ("Stocking the shelves...")
- Skeleton cards during load
- `manifest.json` + `sw.js` for PWA

### v2 — Genre Filtering
- Added genre filter bar below the sign (Action, Comedy, Drama, Horror, Thriller, Sci-Fi, Animation, Romance, Adventure)
- **Decision:** Hybrid cache + live fetch approach chosen by user
  - Boot fetches 3 pages per shelf endpoint (parallel) to build large local cache
  - On genre click: filter local cache first; if result count < 10 across all shelves, fire live TMDB `/discover/movie?with_genres={id}` call
  - `genreCache` object prevents re-fetching a genre already live-fetched
  - Inline loading indicator (spinning dot + "Loading more...") on genre fetch, not full-screen

### v3 — Star Rating Filter
- Added a second filter row: `★ Min Rating:` slider
- 10 stepped positions: Any / 6+ / 6.5+ / 7+ / 7.5+ / 8+ / 8.5+ / 9+ / 9.5+ / 10
- Yellow track fill updates as slider moves
- Display label updates live (e.g. "★ 7.5+ / 10")
- ✕ Clear button, highlights yellow when active
- Stacks with genre filter

### v4 — MPAA Rating Filter
- Added third filter row: `🎬 Max Rating:` with G / PG / PG-13 / R / NC-17 buttons
- Acts as a **ceiling** (max allowed rating, not minimum)
- Colour-coded buttons: G=green, PG=blue, PG-13=yellow, R=orange, NC-17=red
- "In range" buttons (below selected) show lighter highlight
- Description text updates (e.g. "— Parents Strongly Cautioned — may be inappropriate under 13")
- Certifications fetched lazily from `/movie/{id}/release_dates` (US only)
- Small MPAA badge appears on each card once cert is known

### v4.1 — MPAA Bug Fix
**Bug:** Clicking MPAA buttons had no visible effect.
**Root cause (two issues):**
1. Certifications are fetched asynchronously — most movies had `_mpaa = undefined` when the filter was first clicked, and the old `mpaaAllowed()` passed undefined through with `return true`
2. Only the first 16 movies on each shelf had certs pre-fetched at boot

**Fix:**
- `mpaaAllowed()` now returns `false` for movies without a fetched cert when a filter is active (strict, not permissive)
- Added `ensureCertsForVisible()` — fires background batch cert fetch for all uncertified pool movies on filter activation, then re-renders
- Clicking a rating button now immediately filters with current data, then re-renders again once background certs arrive

### v5 — 3D Shelf Visual Upgrade
**User request:** Make the UI more like a physical video store shelf.
**Chosen upgrades:** 3D perspective tilt + physical wood shelf with shadows & depth.

**Changes:**
- `.shelf-row` now has `perspective: 900px; perspective-origin: 50% 60%`
- `.shelf-stage` wraps the movie grid, applies `rotateX(4deg)` to tilt shelf surface back
- `.shelf-plank` upgraded: 28px tall wood fascia, multi-stop gradient (bright top edge → deep shadow), wood grain stripes via repeating gradients, left-edge depth shadow
- `.shelf-underlight` — warm amber radial glow below the plank on the floor
- Cards now lift **up and forward** on hover: `translateY(-22px) translateZ(40px) scale(1.06)` with springy cubic-bezier
- Resting card shadow is short/tight (tape sitting on wood); hover shadow stretches tall/soft
- Card sleeve gets left-edge spine highlight (plastic cassette effect)
- Animation changed from `cardDrop` (fall in) to `cardSlotIn` (slide down into slot)
- HTML structure updated: `movie-grid` now nested inside `shelf-stage`

### v6 — Cross-Pool Deduplication & Date Guard Fixes
**Bug 1:** Same movie appearing on multiple shelves (e.g. in both Rental and Coming Soon)
**Bug 2:** Unreleased titles appearing on the Streaming shelf
**Bug 3:** Released movies appearing on the Coming Soon shelf

**Root causes:**
- No cross-pool deduplication — pools were built independently with no awareness of each other
- Upcoming pool had no date guard (TMDB `/movie/upcoming` sometimes includes recently released films)
- Streaming discover query used `release_date.lte=today` but some TMDB records have incorrect digital release dates

**Fixes in `loadAll()`:**
- Streaming pool: added strict `daysOld >= 0` guard (release_date must be past or today)
- Upcoming pool: added strict `daysUntil > 0` guard (release_date must be strictly future)
- Post-build cross-pool dedup with priority: **rental > streaming > upcoming**
  ```js
  const rentalIds = new Set(pools.rental.map(m => m.id));
  const streamIds = new Set(pools.streaming.map(m => m.id));
  pools.streaming = pools.streaming.filter(m => !rentalIds.has(m.id));
  pools.upcoming  = pools.upcoming.filter(m => !rentalIds.has(m.id) && !streamIds.has(m.id));
  ```
- Same date guards and cross-pool awareness applied to `fetchForGenre()` merge logic

---

## Key Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Framework | None (vanilla JS) | Simplicity, single-file deployment |
| Genre filter strategy | Hybrid cache+live | Fast on common genres, correct on niche ones |
| MPAA filter behaviour | Ceiling (max), not floor | Parental control use case — "show me nothing above PG-13" |
| Unknown MPAA certs | Hidden when filter active | Permissive pass-through made filter useless |
| Rental ranking | Composite score vs. simple popularity | Prevents one viral title from dominating; rewards audience quality |
| Pool dedup priority | Rental > Streaming > Upcoming | Rental is most specific/recent; upcoming is least specific |
| 3D shelf | CSS perspective + rotateX | Pure CSS, no Three.js overhead |

---

## Potential Next Tasks (User Has Not Requested Yet)
- Trailer playback (YouTube embed in modal)
- User watchlist / saved titles (localStorage)
- Multi-region support (currently US providers only)
- Pagination or "load more" for shelves
- Optimised PNG PWA icons (currently SVG data URIs)
- Dark/light mode toggle
- Search bar across all pools
- Sort options for shelves (by date, rating, score)
- Move API key to environment variable for production deploy
- Replace hardcoded TMDB key with a proxy/serverless function

---

## How to Run Locally
```bash
# Any static file server works. Examples:
npx serve .
python3 -m http.server 8080
# Then open http://localhost:8080
```
> ⚠️ Must be served over HTTP/S (not file://) for the service worker to register.

---

## Repository Notes
The user has a GitHub repository for this project. Files are deployed as a static site. No build process — edit `index.html` directly.
