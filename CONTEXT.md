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

### Key Functions

| Function | What it does |
|----------|-------------|
| `fetchRoute(fromLng, fromLat, toLng, toLat)` | Calls OSRM API, returns `[[lat,lng],…]` |
| `loadRoutes()` | Parallel-fetches all 8 main segments on page load |
| `drawSegment(i, mode)` | Draws/redraws a polyline with tooltip for segment `i` |
| `switchMode(mode)` | Updates map lines + sidebar for car/bus/hybrid |
| `renderSegContent(i, mode)` | Returns HTML string for segment sidebar row |
| `updateSidebarSegments(mode)` | Re-renders all 8 sidebar segment rows |
| `buildPopup(stop)` | Builds Leaflet popup HTML with clickable highlights |
| `window.showHL(idx)` | Fetches + draws highlight route, shows `#hl-box` |
| `clearHL()` | Removes highlight layer + hides `#hl-box` |
| `mobView(view)` | Toggles map/sidebar on mobile (`'map'` or `'itinerary'`) |

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

### Mobile Layout
- Bottom tab bar: Map / Itinerary
- Map shown by default; tapping a stop or highlight auto-switches to map view
- Uses `100dvh`, `env(safe-area-inset-bottom)` for iPhone Safari
- `map.invalidateSize()` called after reveal to fix Leaflet tile rendering
- Breakpoint: `max-width: 768px`

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
