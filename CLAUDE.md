# motorcycle-touring — CLAUDE.md

## What this project is

A single-page web app that visualises a motorcycle touring GPX track as an animated 3D map flyover. The user loads a `.gpx` file exported from a GPS logger (e.g. Geo Tracker), and the app plays back the route on a dark 3D map with a moving marker, camera tracking, and elevation stats.

No backend, no build step, no framework.

---

## Repository layout

```
motorcycle-touring/
├── flyover.html    # Entire application (HTML + CSS + JS, ~580 lines)
└── README.md       # Minimal project description
```

---

## Application architecture

Everything lives in `flyover.html`. There is no build process.

### Dependencies (CDN only)

```html
<link href="https://unpkg.com/maplibre-gl@4.7.1/dist/maplibre-gl.css" rel="stylesheet">
<script src="https://unpkg.com/maplibre-gl@4.7.1/dist/maplibre-gl.js"></script>
```

Only MapLibre GL JS v4.7.1. No other external libraries.

### Map setup

- **Base tiles**: CARTO Dark Matter raster (`a/b/c.basemaps.cartocdn.com/dark_all`)
- **Terrain DEM**: Mapzen/AWS terrarium tiles (`s3.amazonaws.com/elevation-tiles-prod/terrarium`)
- **Terrain exaggeration**: `1.3×`
- Initial center: `[130.7, 31.5]` (Kyushu region, Japan), zoom 8

### State variables

| Variable | Type | Description |
|---|---|---|
| `LINE` | `[lon, lat][]` | All track coordinates |
| `ELES` | `(number\|null)[]` | Elevation at each point |
| `CUM` | `number[]` | Cumulative distance (metres) at each point |
| `TOTAL` | `number` | Total route distance (metres) |
| `progress` | `0..1` | Playback position |
| `playing` | `boolean` | Whether animation loop is running |
| `speed` | `1\|2\|4\|8` | Playback multiplier (default 2×) |
| `camMode` | `'off'\|'follow'\|'chase'` | Camera tracking mode |
| `camBearing` | `number` | Current smoothed camera bearing (degrees) |

### Key functions

| Function | Purpose |
|---|---|
| `parseGPX(text)` | DOMParser-based GPX parser; reads `trkpt`, `rtept`, or `wpt` elements |
| `haversine(a, b)` | Great-circle distance in metres between two `{lat, lon}` points |
| `buildRoute(coords)` | Adds/replaces all MapLibre layers and sources; resets playback |
| `sampleAt(distM)` | Binary search + linear interpolation → `{pos, ele, idx}` at a given distance |
| `bearing(a, b)` | Compass bearing (degrees, N=0) between two `[lon,lat]` points |
| `lerpAngle(from, to, t)` | Shortest-path angular interpolation (handles the 0°/360° wrap) |
| `travelBearing(distM)` | Reads 80 m (or 1% of route) ahead to get stable heading |
| `updateCamera(s, distM)` | Applies camera position based on `camMode` |
| `updateScene()` | Main render: updates done-line, bike marker, stats UI, camera |
| `tick(ts)` | `requestAnimationFrame` loop; advances `progress` by `dt/40 * speed` |
| `startPlay()` | Starts animation; eases camera to current position first if tracking |
| `stopPlay()` | Cancels RAF loop |
| `handleFile(file)` | FileReader → parseGPX → buildRoute; computes elevation gain |

### MapLibre sources and layers

| Source ID | Type | Description |
|---|---|---|
| `route` | geojson LineString | Full route (static) |
| `route-done` | geojson LineString | Travelled portion (updated each frame) |
| `route-ends` | geojson FeatureCollection | Start (green) and end (red) markers |
| `bike` | geojson Point | Current position marker (updated each frame) |

| Layer ID | Type | Source |
|---|---|---|
| `route-glow` | line | `route` — orange glow, opacity 0.12 |
| `route-bg` | line | `route` — grey unvisited path |
| `route-done` | line | `route-done` — amber travelled path |
| `route-start` | circle | `route-ends` (filter: start) — green dot |
| `route-end` | circle | `route-ends` (filter: end) — red dot |
| `bike-glow` | circle | `bike` — orange glow |
| `bike-dot` | circle | `bike` — white dot with orange stroke |

### Camera modes

| Mode | Zoom | Pitch | Behaviour |
|---|---|---|---|
| `off` (全体) | — | — | Fits entire route; no tracking |
| `follow` (俯瞰追従) | 12.2 | 45° | Overhead follow with gentle heading |
| `chase` (追走) | 14.2 | 68° | Low rear-chase view, more immersive |

Camera bearing is smoothed each frame: `camBearing = lerpAngle(camBearing, targetBearing, 0.08)`.

### Playback timing

Base duration is **40 seconds** for the full route at 1× speed. At 2× the route completes in ~20 s, at 8× in ~5 s. `speed` is a direct multiplier on `progress` advance per second.

### Progress bar seek

Clicking the progress bar computes `(clickX - barLeft) / barWidth` → sets `progress` directly and calls `updateScene()`.

---

## Development workflow

No build step. Open `flyover.html` directly in a browser, or serve with any static server:

```bash
npx http-server . -p 8080
# Open http://localhost:8080/flyover.html
```

For GPX test files, export a track from Geo Tracker, OsmAnd, or any standard GPS logger app.

---

## Key conventions

- **No build, no npm**: This repo has zero Node.js dependencies. Do not add a `package.json` unless explicitly asked.
- **Single file**: All changes go into `flyover.html`. Do not split CSS or JS into separate files.
- **Japanese UI strings**: Labels, toast messages, and button text are in Japanese. Keep them as-is.
- **MapLibre source/layer lifecycle**: Before calling `map.addSource` / `map.addLayer`, always remove existing layers and sources of the same name (see the loop in `buildRoute`). MapLibre throws if you try to add a duplicate.
- **Terrain requires map load**: `map.setTerrain()` is called inside `map.on('load', ...)`. Any code that accesses terrain must also wait for `idle` or `load`.
- **GPX fallback order**: Parser tries `trkpt` → `rtept` → `wpt` in that order. Do not change this without testing all three types.
- **`window._coords`**: The parsed coords array is stored on `window._coords` for debugging convenience. This is intentional.

---

## What to watch out for

- **Terrain tiles attribution**: The Mapzen/AWS terrarium tiles are used for elevation data. They are free for reasonable use but the attribution (`Terrain: Mapzen / AWS`) must be kept.
- **CARTO tiles**: Dark Matter tiles are free for non-commercial use under CARTO's terms. Attribution `© OpenStreetMap © CARTO` is required and already present.
- **iOS safe area**: The controls panel uses `env(safe-area-inset-bottom)` for bottom padding to avoid the iPhone home indicator. Keep this when modifying the controls layout.
- **No GPX elevation**: If GPX points lack `<ele>` elements, `ELES` entries are `null` and the elevation stat displays `–`. Division-by-null is guarded throughout.
- **Smooth bearing across 0°/360°**: `lerpAngle` handles the wrap-around. If you replace it, make sure the new implementation does the same.
