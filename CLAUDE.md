# motorcycle-touring — CLAUDE.md

## What this project is

A single-page web app that visualises a motorcycle touring GPX track as an animated 3D map flyover. The user loads a `.gpx` file exported from a GPS logger (e.g. Geo Tracker), and the app plays back the route on a satellite 3D map with a moving marker, camera tracking, elevation stats, and an optional cinema mode for screen recording.

No backend, no build step, no framework.

---

## Repository layout

```
motorcycle-touring/
├── flyover.html    # Entire application (HTML + CSS + JS, ~700 lines)
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

- **Satellite tiles**: Esri World Imagery raster (`server.arcgisonline.com/…/World_Imagery/MapServer/tile/{z}/{y}/{x}`)
  - API key not required. `maxzoom: 19`. **Tile URL uses `{z}/{y}/{x}` order** (not `{z}/{x}/{y}` — Esri convention).
  - `raster-fade-duration: 100` (reduced from default 300 ms to suppress blurring on tile load)
  - `raster-resampling: 'linear'` (bilinear, explicitly set)
- **Terrain DEM**: Mapzen/AWS terrarium tiles (`s3.amazonaws.com/elevation-tiles-prod/terrarium/{z}/{x}/{y}.png`)
- **Terrain exaggeration**: `1.3×`
- **Pixel ratio**: `Math.min(window.devicePixelRatio || 1, 2)` — renders at 2× on high-DPR phones (DPR=3 devices capped at 2 for GPU budget)
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
| `camMode` | `'off'\|'follow'\|'chase'` | Camera tracking mode |
| `camLon` | `number` | Smoothed camera center longitude (interpolated each frame) |
| `camLat` | `number` | Smoothed camera center latitude (interpolated each frame) |
| `camBearing` | `number` | Smoothed camera bearing (degrees, interpolated each frame) |
| `camInit` | `boolean` | Whether camLon/camLat/camBearing have been initialised for current session |
| `cinemaMode` | `boolean` | Whether cinema mode (UI hidden) is active |
| `cinemaTapTimer` | timer ID | Auto-hide timer for cinema peek mode |
| `elevPts` | `number[]` | 160-point normalised elevation profile (cached in `buildRoute`) |
| `elevMeta` | `{minE, maxE}` | Elevation range for `elevPts` |

### Key functions

| Function | Purpose |
|---|---|
| `parseGPX(text)` | DOMParser-based GPX parser; reads `trkpt`, `rtept`, or `wpt` elements |
| `haversine(a, b)` | Great-circle distance in metres between two `{lat, lon}` points |
| `buildRoute(coords)` | Adds/replaces all MapLibre layers and sources; caches `elevPts`; resets playback |
| `sampleAt(distM)` | Binary search + linear interpolation → `{pos, ele, idx}` at a given distance |
| `bearing(a, b)` | Compass bearing (degrees, N=0) between two `[lon,lat]` points |
| `lerpAngle(from, to, t)` | Shortest-path angular interpolation (handles the 0°/360° wrap) |
| `travelBearing(distM)` | Reads 80 m (or 1% of route) ahead to get stable heading |
| `updateCamera(s, distM)` | Lerps `camLon/camLat` (α=0.08) and `camBearing` (α=0.06) toward target, then `jumpTo` |
| `updateScene()` | Main render: updates done-line, bike marker, stats UI, camera, cinema overlay |
| `tick(ts)` | `requestAnimationFrame` loop; advances `progress` by `dt / BASE_DURATION`; `dt` capped at 100 ms |
| `startPlay()` | Starts animation; eases camera to current position first if tracking |
| `stopPlay()` | Cancels RAF loop |
| `togglePlay()` | Toggles play/pause |
| `handleFile(file)` | FileReader → parseGPX → buildRoute; computes elevation gain |
| `drawElevGraph()` | Draws 160-pt elevation profile + progress cursor on `#elevCanvas` (canvas 2D) |
| `updateCinemaStats()` | Updates `#cinemaEle` and `#cinemaDist` badges in cinema overlay |
| `setCinemaUI(mode)` | Transitions cinema state: `'on'` hides UI, `'off'` restores, `'peek'` shows for 3 s |
| `toggleCinema()` | Called by `#btnCinema`; toggles between on/off |

### MapLibre sources and layers

| Source ID | Type | Description |
|---|---|---|
| `satellite` | raster | Esri World Imagery base map (always present) |
| `terrain` | raster-dem | Mapzen/AWS elevation data (always present) |
| `route` | geojson LineString | Full route (static) |
| `route-done` | geojson LineString | Travelled portion (updated each frame) |
| `route-ends` | geojson FeatureCollection | Start (green) and end (red) markers |
| `bike` | geojson Point | Current position marker (updated each frame) |

| Layer ID | Type | Source |
|---|---|---|
| `bg` | background | — dark fallback while tiles load |
| `satellite` | raster | `satellite` — Esri World Imagery |
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

Camera center and bearing are smoothed each frame toward the target using per-frame lerp:
- Center: `camLon += (target - camLon) * 0.08` (≈200 ms lag at 60 fps — lets satellite tiles pre-load)
- Bearing: `camBearing = lerpAngle(camBearing, targetBearing, 0.06)` (slightly slower for cinematic feel)
- On first frame, mode-switch, or reset: values snap to target immediately (`camInit = false` triggers this)

### Playback timing

`BASE_DURATION = 100` seconds for the full route at fixed pace (no speed selector). `progress` advances by `dt / BASE_DURATION` per frame. `dt` is capped at 100 ms (`Math.min(dt, 0.1)`) to prevent large jumps when the tab was backgrounded.

### Cinema mode

`🎬 シネマ` button (4th in the button row) toggles cinema mode for clean screen recording:

- **ON**: `#topbar` and `#controls` fade to `opacity:0; pointer-events:none` (0.45 s CSS transition). `#cinema-overlay` appears at top-left: a 160×54 canvas elevation profile + distance and current-elevation badges.
- **OFF**: UI fades back in; overlay hidden.
- **Peek**: Tapping the map while cinema is ON restores UI for 3 seconds, then re-hides.
- The elevation profile (`elevPts`) is pre-computed at 160 normalised points in `buildRoute()` so `drawElevGraph()` is cheap per frame (canvas clear + fill + line + cursor line only).

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

- **Esri tile URL order**: Esri uses `{z}/{y}/{x}` (not `{z}/{x}/{y}`). Do not swap these — the tiles will silently return wrong images.
- **Terrain tiles attribution**: The Mapzen/AWS terrarium tiles are used for elevation data. They are free for reasonable use but the attribution (`Terrain: Mapzen / AWS`) must be kept.
- **iOS safe area**: The controls panel uses `env(safe-area-inset-bottom)` for bottom padding to avoid the iPhone home indicator. Keep this when modifying the controls layout.
- **No GPX elevation**: If GPX points lack `<ele>` elements, `ELES` entries are `null` and the elevation stat displays `–`. In cinema mode, `elevPts` stores `0` for null elevations and `drawElevGraph` handles flat range with `|| 1` guard.
- **Smooth bearing across 0°/360°**: `lerpAngle` handles the wrap-around. If you replace it, make sure the new implementation does the same.
- **`camInit` flag**: Must be set to `false` whenever the route is reset or camera mode is switched, so the lerp variables snap to the new position immediately rather than drifting from the old one.
- **Cinema mode pointer-events**: When cinema is ON, `controls.style.pointerEvents = 'none'` lets map-click events through to trigger peek. When OFF, resetting to `''` restores the default (auto).
