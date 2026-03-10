# New Release Wall — Claude Code Project Brief

## What This Is
A single-page Progressive Web App (PWA) that mimics the Blockbuster "new release wall" — a live, browsable shelf of popular streaming movies, top rentals, and coming-soon titles. No framework, no build step. Pure vanilla HTML/CSS/JS in a single `index.html` file.

---

## File Structure
```
index.html      — entire app (HTML + CSS + JS, all inline)
sw.js           — service worker (cache-first for static, network-first for TMDB images)
manifest.json   — PWA manifest (standalone display, SVG icons)
CLAUDE.md       — this file
CONTEXT.md      — full conversation history and decision log
```

---

## Tech Stack
- **Vanilla HTML/CSS/JS** — no framework, no bundler
- **Fonts:** Bebas Neue + Barlow Condensed + Barlow (Google Fonts)
- **Data:** TMDB API v3 (api_key param style)
- **Provider data:** JustWatch via TMDB `/movie/{id}/watch/providers` (US region)
- **MPAA certs:** TMDB `/movie/{id}/release_dates` (US certifications)

---

## API Key
```
TMDB API key: 5446e0b1e589ec5e4feacc4e9dcdbd20
Base URL: https://api.themoviedb.org/3
Image base (w342): https://image.tmdb.org/t/p/w342
Image base (w500): https://image.tmdb.org/t/p/w500
```

---

## Design System
```css
--bb-yellow: #FFD700   /* primary accent, Blockbuster gold */
--bb-blue:   #003087   /* Blockbuster navy */
--bb-dark:   #0d0500   /* page background */
--bb-shelf:  #2a1500   /* filter button background */
--bb-text:   #f5e6c8   /* body text */
--bb-muted:  #a08060   /* secondary text */
--bb-accent: #e8b400   /* warm amber for shelf labels */
--bb-neon-r: #ff2244   /* LIVE sign / STREAM badge */
```

**Visual theme:** Blockbuster Video store. Navy/amber/yellow. VHS tape aesthetic. 3D perspective shelves with wood grain planks and ambient underlighting. Cards lift off the shelf on hover like pulling a tape.

---

## Three Shelves

### 1. 📺 New to Streaming (`pools.streaming`)
- Source: `/discover/movie` with `with_release_type=4|5|6` (digital releases), last 45 days
- Fetches 3 pages on boot
- Strict date guard: `release_date <= today` (no future titles)
- Cross-pool deduplicated: movies in `rental` are excluded

### 2. 🎟 Top Rentals & VOD (`pools.rental`)
- Source: `/movie/now_playing` (3 pages) + `/movie/popular` (3 pages)
- Window: last 120 days
- Ranked by **Blockbuster Score** (see below)
- Cross-pool dedup priority: rental wins over all other shelves

### 3. 📅 Coming Soon (`pools.upcoming`)
- Source: `/movie/upcoming` (2 pages)
- Strict date guard: `release_date > today` only (no released titles)
- Cross-pool deduplicated: excluded if in rental or streaming

---

## Blockbuster Score Algorithm
Composite score 0–100 for the rental shelf ranking:

```js
const WEIGHTS = { theater: 0.40, trending: 0.35, audience: 0.25 };

// Theater Heat: 1.0 if in now_playing, decays linearly to 0 over 90 days after leaving
theaterHeat = inTheaters ? 1.0 : Math.max(0, 1 - daysOld / 90)

// Trending: TMDB popularity, log-scaled against corpus max
trendScore = Math.log1p(movie.popularity) / Math.log1p(maxPop)

// Audience: Bayesian-weighted vote_average (m=1500 min votes threshold)
bayesRating = (votes/(votes+1500)) * voteAvg + (1500/(votes+1500)) * corpusMean
audienceScore = bayesRating / 10

score = (theaterHeat*0.40 + trendScore*0.35 + audienceScore*0.25) * 100
```

Top 3 get gold/silver/bronze rank badges. All rental cards show a score bar. The modal shows the full signal breakdown.

---

## Genre Filtering — Hybrid Cache + Live Fetch
- On boot: 3 pages fetched per shelf → large local cache (~60–80 movies per shelf)
- On genre click: filters local cache first
- If total results across all three shelves < 10: fires live TMDB `/discover/movie?with_genres={id}` calls, merges new results into pools, re-renders
- Genre fetch cache: `genreCache[genreId] = true` prevents re-fetching

---

## Star Rating Filter
- Slider with 10 steps: Any, 6, 6.5, 7, 7.5, 8, 8.5, 9, 9.5, 10
- Filters on `movie.vote_average >= minRating`
- `RATING_STEPS = [0, 6, 6.5, 7, 7.5, 8, 8.5, 9, 9.5, 10]`
- Stacks with genre and MPAA filters

---

## MPAA Rating Filter
- Buttons: G → PG → PG-13 → R → NC-17 (max rating selector)
- Acts as a ceiling: selecting PG-13 shows G, PG, and PG-13
- Certifications fetched lazily from `/movie/{id}/release_dates` (US only)
- Movies without a fetched cert are **hidden** when filter is active (not shown)
- On filter activation, `ensureCertsForVisible()` fires background fetch for all uncertified movies in pool, then re-renders
- `MPAA_ORDER = ['G','PG','PG-13','R','NC-17']`
- Stacks with genre and star rating filters

---

## Provider Data
TMDB provider IDs mapped to display names and CSS classes:
```js
8:'Netflix', 9:'Prime', 337:'Disney+', 384:'Max', 15:'Hulu',
386:'Peacock', 2:'Apple TV+', 531:'Paramount+',
3:'iTunes'(rental), 10:'Amazon'(rental), 68:'Vudu'(rental),
7:'Vudu'(rental), 192:'YouTube'(rental), 257:'Google Play'(rental)
```
Flatrate providers = streaming badges. Rent/buy providers = "(Rent)" suffix in amber.

---

## Key State Variables
```js
const pools = { streaming: [], rental: [], upcoming: [] }  // master data pools
const genreCache = {}          // genreId → true (live fetch done)
let theaterIds = new Set()     // TMDB movie IDs currently in theaters
let scoringMeta = null         // { maxPop, meanRating } for scoring
let genreFilter = 'all'        // current genre filter ('all' or TMDB genre ID string)
let minRating = 0              // minimum vote_average (0 = off)
let maxMpaa = null             // max MPAA rating string or null
let detailCache = {}           // movie id → full TMDB detail response
let fetchingGenre = false      // debounce flag during live genre fetch
const RATING_STEPS = [0, 6, 6.5, 7, 7.5, 8, 8.5, 9, 9.5, 10]
const MPAA_ORDER = ['G','PG','PG-13','R','NC-17']
```

---

## Key Functions
| Function | Purpose |
|---|---|
| `loadAll()` | Boot data fetch — 3 pages per shelf, build pools, score, dedup |
| `fetchForGenre(genreId)` | Live TMDB fetch when genre cache misses, merges into pools |
| `fetchProviders(movie)` | Lazy-loads JustWatch data onto a movie object |
| `fetchCertification(movie)` | Lazy-loads MPAA cert from `/release_dates` |
| `ensureCertsForVisible()` | Batch-fetches certs for all uncertified pool movies, re-renders |
| `applyFilter(movies)` | Applies genre + star rating + MPAA filter to a movie array |
| `mpaaAllowed(cert, max)` | Returns false if no cert or cert exceeds max |
| `scoreMovie(movie)` | Calculates and attaches `movie._score` object |
| `buildScoringMeta(pool)` | Computes maxPop and meanRating for Bayesian scoring |
| `renderAll()` | Calls renderShelf for all three shelves with filtered/sliced pools |
| `renderShelf(key, movies, badge, showRank)` | Renders cards into a grid, wires click handlers |
| `movieCard(m, badge, i, showRank)` | Returns HTML string for a single movie card |
| `openModal(id)` | Opens detail modal, triggers lazy provider/cert fetch |
| `updateSliderUI(stepIdx)` | Updates star rating slider track fill and display label |
| `updateMpaaUI()` | Updates MPAA button selected/in-range states |
| `boot()` | Entry point — shows loading screen, calls loadAll, renders |

---

## 3D Shelf CSS Architecture
```
.shelf-row              — perspective container (perspective: 900px)
  .shelf-row-label      — section title above shelf
  .shelf-stage          — rotateX(4deg) tilted surface, wood texture
    .movie-grid         — CSS grid, cards aligned to bottom
      .movie-card       — lifts up+forward on hover (translateY + translateZ)
        .card-sleeve    — the tape case (aspect-ratio 2/3, Blockbuster stripe)
  .shelf-plank          — front wooden fascia, 28px tall, multi-stop gradient
  .shelf-underlight     — warm glow cast on floor below plank
```

---

## Known Limitations / Potential Next Tasks
- TMDB cert data is incomplete — many movies have no US certification on record
- No user accounts or saved lists
- No trailer playback
- Provider data is US-only (region hardcoded)
- No pagination UI — shelves cap at 20 (streaming/rental) or 16 (upcoming) cards
- PWA icons are SVG data URIs, not optimised PNGs
- The TMDB API key is hardcoded and public — fine for a personal project, not for production
