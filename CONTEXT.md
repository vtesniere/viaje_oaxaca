# Project Context — Viaje Oaxaca

> Read this file at the start of every conversation about this project.
> Update it whenever new features are added, decisions are made, or the user gives feedback.

---

## What This Is

An interactive road trip map for a ~3-week journey from **Oaxaca City → Guatemala City** via Chiapas, returning by flight GUA → Mexico City. Built as a single-file static HTML app, deployed on GitHub Pages.

**Live URL:** https://vtesniere.github.io/viaje_oaxaca/
**Local path:** `/Users/vladimirtesniere/Dev/Claude/viaje_oaxaca/`
**GitHub repo:** `vtesniere/viaje_oaxaca` (SSH push via `~/.ssh/id_ed25519`)

---

## Route

Oaxaca City → Tehuantepec → Tuxtla Gutiérrez → San Cristóbal de las Casas → La Mesilla (border) → Huehuetenango → Quetzaltenango (Xela) → Antigua Guatemala → Guatemala City ✈️ Mexico City

- ~1,200 km · 9 stops · 1 land border crossing · ~22 days
- Max 6 h driving per day (hard constraint from user)

---

## File Structure

```
viaje_oaxaca/
├── index.html                    ← entire app (single file, ~700 lines)
├── README.md                     ← trip summary + link to live site
├── CONTEXT.md                    ← this file
└── .github/
    └── workflows/
        └── pages.yml             ← GitHub Actions auto-deploy to Pages
```

---

## index.html — Architecture

### Data Structures (top of `<script>`)

| Variable | Type | Description |
|----------|------|-------------|
| `stops[]` | Array(9) | Each stop: `{num, name, country, lat, lng, color, days, desc, highlights[], tip}` |
| `stops[i].highlights[]` | Array | Each: `{name, time, lat, lng}` — `time` is a human string like `"12 km · ~20 min"` |
| `carSegs[]` | Array(8) | Car segments: `{km, t, from, to, road, notes, border}` |
| `busSegs[]` | Array(8) | Bus segments: `{t, company, cost, freq, terminal, notes, border}` |
| `hybridSegs[]` | Array(8) | `{mode: 'car'|'bus'|'shuttle', rec}` |
| `modeDescs` | Object | Description text for each mode tab |
| `GT_COLORS` | Object | `{A: '#4ade80', B: '#818cf8'}` — route colours for Guatemala panel |
| `gtPaths` | Object | `{A: [[lat,lng],…], B: …}` — waypoint arrays for OSRM routing |
| `gtRoutes` | Object | Keys A/B, each with `stops[], segments[], timeline[]` |
| `gtMarkerLayers` | Object | Leaflet layer groups keyed A/B — GT stop markers |
| `gtRouteLayers` | Object | Leaflet layer groups keyed A/B — GT polylines |
| `mainFlightArc` | L.Polyline | GUA → MEX arc (stored to hide/show when GT panel toggles) |
| `mainMexMarker` | L.Marker | Mexico City marker (stored to hide/show) |

### Key Functions

| Function | What it does |
|----------|-------------|
| `fetchRoute(fromLng, fromLat, toLng, toLat)` | Calls OSRM API, returns `[[lat,lng],…]` |
| `loadRoutes()` | Parallel-fetches all 8 main segments on page load |
| `drawSegment(i, mode)` | Draws/redraws a polyline; skips `addTo(map)` when `gtOpen` |
| `switchMode(mode)` | Updates map lines + sidebar for car/bus/hybrid |
| `renderSegContent(i, mode)` | Returns HTML string for segment sidebar row |
| `updateSidebarSegments(mode)` | Re-renders all 8 sidebar segment rows |
| `buildPopup(stop)` | Builds Leaflet popup HTML with clickable highlights |
| `window.showHL(idx)` | Fetches + draws highlight route, shows `#hl-box` |
| `clearHL()` | Removes highlight layer + hides `#hl-box` |
| `mobView(view)` | Toggles map/sidebar on mobile — uses GT panel when `gtOpen` |
| `toggleGT()` | Opens/closes Guatemala panel; swaps sidebar ↔ GT panel, hides/shows main trip map layers |
| `hideMainTrip()` | Removes all main-trip markers, polylines, flight arc, MX marker from map |
| `showMainTrip()` | Restores all main-trip map layers |
| `selectGTRoute(key)` | Switches between A/B, renders body, swaps map layers, fits bounds |
| `renderGTBody(key)` | Builds stop cards + segment rows + timeline HTML for GT panel |
| `buildGTRoute(key)` | Adds GT markers + OSRM polylines to their layer group |

### Highlight onclick Pattern
Popups are HTML strings, so click handlers use a global registry:
- `window._hls[]` — array of `{baseLat, baseLng, lat, lng, name, time, stopName}`
- Each `.hl-item` has `onclick="showHL(${idx})"` and `data-idx="${idx}"`

### Routing
- **Library:** Leaflet.js 1.9.4 (CDN)
- **Tiles:** CartoDB Dark (`carto.com/basemaps`) — no API key needed
- **Road routing:** OSRM public API (`router.project-osrm.org/route/v1/driving/`)
  - OSRM returns coordinates as `[lng, lat]` → must swap to `[lat, lng]` for Leaflet
  - All 8 main segment routes fetched in parallel on load; stored in `routeCoords[]`
  - Highlight routes fetched on demand when user clicks a highlight

### Travel Mode Tabs
Three modes: 🚗 Car / 🚌 Bus / 🔀 Hybrid
- Switching mode redraws all 8 polylines (same OSRM geometry, different color/tooltip)
- Car border segments: `#ef4444` (red); other car: `#f97316` (orange)
- Bus/shuttle: `#14b8a6` (teal)
- Highlight route: `#38bdf8` (sky blue)

### Body Layout (flex-row on desktop)
```
#sidebar (400px) | #gt-panel (400px, hidden) | #map-wrap (flex:1)
```
Only one of `#sidebar` / `#gt-panel` is visible at a time:
- Default: sidebar shown, gt-panel `display:none`
- GT open: sidebar gets `.gt-hidden` (`display:none!important`), gt-panel gets `.open` (`display:flex`)
- This ensures zero overlap — gt-panel occupies the same column slot as the sidebar

### Guatemala Panel — DOM / state
- `gtOpen` (boolean) global flag guards `drawSegment` and `mobView`
- When GT opens: `hideMainTrip()` removes all main-trip map layers; GT layers added by `buildGTRoute()`
- When GT closes: `showMainTrip()` restores main-trip layers; GT layers removed; map recentred
- `buildGTRoute` is called once per route key (guarded by `gtBuilt{}`); on re-open just re-adds cached layers

### Mobile Layout
- Bottom tab bar: Map / Itinerary
- Map shown by default; tapping a stop or highlight auto-switches to map view
- Uses `100dvh`, `env(safe-area-inset-bottom)` for iPhone Safari
- `map.invalidateSize()` called after reveal to fix Leaflet tile rendering
- Breakpoint: `max-width: 768px`
- When GT panel is open on mobile, "Itinerary" tab shows `#gt-panel.mob-visible` instead of `#sidebar`
- `#gt-panel.open` is overridden to `display:none` in the mobile media query; only `.mob-visible` shows it

---

## GitHub / Deploy Setup

- **Auth:** SSH via `~/.ssh/id_ed25519` (titled "macbook M2" on GitHub)
- **Deploy:** GitHub Actions workflow (`.github/workflows/pages.yml`) triggers on push to `main`
- **PAT limitations:** Fine-grained PAT only has "Administration: Read & Write" — no Contents write, no SSH key upload, no Pages API. All pushes done via SSH.
- **Always run before `gh` CLI:** `eval "$(/opt/homebrew/bin/brew shellenv)"`

---

## User Preferences & Decisions

- Max 6 h/day driving — hard constraint
- Wants to compare car vs bus vs hybrid options side by side
- Prefers OSRM real road routing over straight-line polylines
- Highlights should be clickable to show exact driving route
- Mobile: map first, itinerary accessible via tab
- **Do NOT use hooks** unless explicitly requested
- Ask before taking any action not explicitly instructed

---

## Session Notes

| Date | Change |
|------|--------|
| Session 1 | Initial HTML map built: stops, highlights, OSRM routing, 3 travel modes |
| Session 1 | GitHub repo created, SSH auth set up, deployed to Pages via Actions |
| Session 1 | Mobile responsive layout added: bottom tab bar, `mobView()`, safe area padding |
| 2026-04-04 | Guatemala Trip Proposal panel added: floating 🇬🇹 button, two routes (A Classic South, B Full Highlands), OSRM routing, Chichicastenango day-trip marker |
| 2026-04-05 | GT panel refactored: moved from absolute overlay to sidebar sibling — no more overlap with main trip; `hideMainTrip()`/`showMainTrip()` swap map layers; mobile: Itinerary tab shows GT panel when open |
