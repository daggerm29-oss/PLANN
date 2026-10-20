# WORLDVIEW OSINT — Progress Log

> Hand-off file. Any AI/dev continuing this work: read this first, then continue from **Next steps**.
> Source repo: https://github.com/daggerm29-oss/sky8 (branch `main`, imported into the v0 workspace without `.git`).

## Architecture (as found)

```
next dev (port 3000, scripts/dev.mjs)  --rewrites /*-->  Vite dev server (5173)
                                                           artifacts/worldview (React 19 + MapLibre GL v5 + Cesium + three)
                                                           vite.config.ts mounts the Express routers from
                                                           artifacts/api-server/src/routes (proxy.ts, cctv.ts) at /api
Express proxy (/api/proxy/*)  --> upstream maritime backend (SHIPS_UPSTREAM_URL) + adsb.lol + CelesTrak + CCTV feeds
```

- `artifacts/worldview/src/pages/WorldviewMap.tsx` (~2.5k lines) — main map page, state, layers, toolbar.
- `components/ShipsLayer.tsx`, `SatelliteLayer.tsx`, `PlaybackLayer.tsx`, `CctvLayer.tsx` — MapLibre layers.
- `components/CesiumFollowCam.tsx` — Cesium 3D follow camera (pre-existing, aircraft only).
- `components/CinematicOverlay.tsx`, `FloatingPanel.tsx`, `MaritimePanel.tsx`, `MasterToolbar.tsx`.
- `artifacts/api-server/src/routes/proxy.ts` — all upstream proxies (ships, dark-fleet, oil, status, playback, adsb).
- 3D models in `artifacts/worldview/public/`: `aeroplane.glb`, `military-aircraft.glb`, `satellite.glb`, `iss.glb`, `ship.glb`, `tanker.glb`.
- npm workspaces (root `package.json`); install with `npm install` at repo root.

## New backend: https://worldview-intel-i09c.onrender.com

Verified live 2026-10-17 (same JSON schema as the old Render backend):
- `/api/status` → `{live_vessels, dark_ships_tracked, ais:{...}, uptime_min}`
- `/api/radar` → `[{mmsi, vessel_name, lat, lon, sog, cog, last_updated, dwt, est_barrels, category, source}]` (~3.3 MB)
- `/api/live` → same without dwt/category
- `/api/dark-fleet` → `[{mmsi, ship_name, last_known_lat, last_known_lon, time_went_dark, last_sog, last_cog}]`
- `/api/analytics/live-oil`, `/api/playback/config` → OK

## Step log

- [x] **Step 1** — Cloned repo, mapped architecture, probed new backend (all endpoints 200).
- [x] **Step 2** — Imported repo into v0 workspace; removed conflicting v0 template files.
- [x] **Step 3** — Backend migration: `proxy.ts` default `SHIPS_UPSTREAM_URL` → `https://worldview-intel-i09c.onrender.com`,
      `.env.example` + `replit.md` updated. `grep -r onrender` is clean.
- [x] **Step 4** — Single-click → right-side `EntityInfoPanel` (ship / aircraft / satellite). `ShipsLayer` got `onSelect/onTrack`
      (padded bbox picking); `SatelliteLayer` got `onFocus/onTrack` (dblclick); aircraft already had click/dblclick and now also feeds the panel via `focus` state in `WorldviewMap.tsx`.
- [x] **Step 5** — Double-click → cinematic 3D follow view. New `lib/tracked-entity.ts`, `components/CinematicTracker.tsx`
      (Cesium, own lookAt camera: log-range intro zoom-in from 3000 km, drag-orbit, wheel zoom, dead reckoning, trail, fade),
      `CinematicHost.tsx` (keeps last target mounted so exit = pull-back + fade, ~1.5s), `CinematicController` in `WorldviewMap.tsx`.
      Models: aeroplane/military-aircraft/ship/tanker/satellite/iss .glb. Esc or panel EXIT 3D exits. Old `CesiumFollowCam.tsx` is now unused (safe to delete).
- [x] **Step 6** — `npm install` + `node scripts/dev.mjs` (Next :3000 -> Vite :5173). Browser-verified aircraft path end to end:
      click -> right panel, TRACK 3D -> intro zoom + GLB aircraft over 3D terrain with panel kept visible, Esc -> pull-back + fade + map restore.
      Ships/satellites share the same code path and type-check, but were NOT live-tested: the new backend currently reports
      `live_vessels: 0` (AIS socket closed), only 50 dark-fleet gap candidates; satellites need 3D ORBITAL + SATS on.

- [x] **Step 7** — Aircraft feed hardening: `proxy.ts` slims adsb.lol payload (7 MB → ~2 MB), dedupes in-flight
      upstream fetches, serves stale while refreshing; `WorldviewMap.tsx` `fetchData` skips overlapping polls,
      45 s timeout, keeps last aircraft on transient failure (no more "FEED INTERRUPTED" + 0 contacts).

## Step 8 — 3D cinematic view fixes (IN PROGRESS, all in `components/CinematicTracker.tsx` unless noted)
User report: dblclick 3D slow / sometimes no model; ship floats above sea; plane faces wrong way;
model jitters forward/back; remove red/white/green dots (wingtip lights) and the line (trail).
Root causes found:
- Slow: entity was created only AFTER `await loadGoogleTileset()` → model waited on tiles. Fix: add entity first,
  load tiles in background; prefetch GLBs on app start (`lib/tracked-entity.ts` `preloadModels`).
- Floating ship: placed at ellipsoid h=0, Google tiles sea is at geoid (~-30..-60 m in Arabian Sea). Fix: sample
  surface height under the vessel from the tileset (`scene.clampToHeightMostDetailed`) and put hull at waterline.
- Jitter: each new sample restarted dead reckoning from an older position → snap back. Fix: display has its own
  velocity, error corrected smoothly with forward-only along-track clamp.
- Wrong facing: per-model heading offset verified visually (see `MODEL_HEADING_OFFSET`).
- Extra root cause of jitter: `CinematicController` (WorldviewMap.tsx) rebuilds `target` every render, and the
  tracker treated each render as a new sample → dead reckoning restarted from the stale fix. Now keyed by value.
- [x] 8a remove lights + trail  - [x] 8b non-blocking load + `preloadModels` (called in WorldviewMap)
- [x] 8c sea-level: `scene.sampleHeightMostDetailed` every 5 s, hull at `surface + keelDepth*scale` (`MODEL_GEOMETRY`)
- [x] 8d smooth motion (own velocity + clamped error correction; aircraft `seen` → `sampleAgeSec`)
- [x] Fixed vessel "LAST UPDATE: Invalid Date" (ISO string timestamps)
- [x] 8e heading offset verified in browser (side view, yaw -90°): all 4 aircraft/ship models need 180°
      (was -90 → sideways). Satellites kept at -90 (unchanged, unverified).
- [x] Model is now a `Cesium.Model` primitive whose matrix is set in the same preRender as the camera
      (entity version trailed ~1 frame behind the camera → stutter).
- [x] Ship waterline verified over Google tiles sea (Spain coast, sampled surface ≈ +37 m ellipsoidal).
- [x] Temp debug hooks removed. **Step 8 DONE.**
- tsc clean (except pre-existing `src/components/ui/*`). `scripts/analyze-glb.mjs` = model bounds tool.

## Step 9 — VISION MODES (IN PROGRESS, started 2026-10-19)
Request: switchable vision modes over the map AND the photorealistic Google 3D tiles (cinematic view):
Tactical (existing dark, default), Normal (optical), CRT/Retro, NVG, FLIR (thermal), Anime, Noir, Snow, Satellite.
Aircraft/ship/satellite icons + 3D models must match each mode.

Design (all new code is self-contained):
- `src/lib/vision-modes.ts` — `VisionMode` type, `VISION_MODES` config (label, hotkey, CSS `filter` string,
  imagery on/off, entity paint overrides), `applyVisionToMap(map, mode, baseLayerIds)`:
  adds Esri World Imagery raster under the base-style labels for every mode except Tactical, and overrides
  icon/circle/line colours + halos of all app (non-base-style) layers (FLIR/NVG/Noir → white-hot; Anime/Snow/Optical → dark outline). Originals are cached and restored.
- `src/components/VisionFilters.tsx` — hidden `<svg>` with SVG filter chains (NVG phosphor+bloom, FLIR WHOT/BHOT/IRONBOW,
  CRT RGB split+bloom, Anime posterize+ink edges, Noir S-curve, Snow whitening, Satellite sharpen).
- `src/components/VisionOverlay.tsx` — pointer-events:none effects + HUD per mode (grain, scanlines, aperture grille,
  tube vignette, falling-snow canvas, FLIR reticle + palette bar, satellite telemetry/grid).
- `src/components/VisionModeSelector.tsx` — top-centre switcher (hotkeys 1–9) + FLIR palette sub-toggle. Persisted in localStorage (UI pref only).
- `WorldviewMap.tsx` — all world layers (map canvas, satellite canvas, Cesium cinematic) wrapped in one `<div style={{filter}}>`
  so map, icons and 3D models get the identical treatment; records base-style layer ids on load; re-applies on `styledata`.
- `CinematicTracker.tsx` — `visionMode` prop: night tile shader only in Tactical; Esri imagery globe otherwise.

Checklist:
- [x] 9a vision-modes.ts  - [x] 9b VisionFilters  - [x] 9c VisionOverlay  - [x] 9d Selector
- [x] 9e wired into WorldviewMap (+CinematicController/Host/Tracker `visionMode` prop). tsc clean.
- [x] 9f browser-verified all 9 modes on the 2D map + cinematic 3D (Google photorealistic tiles) in OPT/NVG/FLIR/ANIME.
      Tuning done: Anime blur 1.6→0.8, saturate 2.1→1.7, entities white + ink halo (cyan icons vanished into posterized sea).
      3D model now matches each mode: `MODEL_VISION` + `applyModelVision` in `CinematicTracker.tsx`
      (FLIR yellow→white-hot, NVG/Noir bright, Snow dark, Anime/CRT silhouette outline).
- [x] Bug fix: after EXIT 3D, TRACK 3D was silently cancelled (selectedAircraft null → "selection cleared" effect).
      `trackFocused` now re-selects the aircraft. **Step 9 DONE.**
- [x] Bug fix: map remount (HMR) threw `getLayer` on undefined / "Style is not done loading" in Ships/Cctv/Playback layers.
      WorldviewMap cleanup now `setMapLoadedState(false)`; layer teardowns bail when `!map.style` (map already removed).
- Note: switching modes, aircraft tiles may briefly keep the previous mode's colour until MapLibre re-renders them (transient).

## Step 10 — 3D cinematic upgrades (IN PROGRESS, started 2026-10-19, repo now github.com/daggerm29-oss/PLANJ)
Requests: (a) aircraft camera a little closer + zoom in/out control for 3D models; (b) ship view: animated
turquoise neon ocean shader (procedural waves, crest highlights, starry dark backdrop, glowing wake ripples,
splash particles); (c) satellite 3D: 2D satellite icon canvas was drawn on top of the 3D view (SatelliteLayer
canvas zIndex 6 > Cesium 5) -> hide it while tracking; (d) animated marching dashed glowing-green orbit path.
Plan / files:
- `CinematicTracker.tsx`: realistic aircraft scale (zoom now changes model size), closer default range,
  zoom HUD (+ / - / reset, portal to body, also +/- keys). Ship: no Google tiles; `lib/ocean-shader.ts`
  builds a GLB grid + CustomShader (waves in VS, neon crests/grid/Kelvin wake/rings in FS) and a Cesium
  ParticleSystem bow/stern spray; ship bobs with the waves. Satellite: if `target.ecefAt` exists the model is
  placed exactly on the SGP4 propagated position each frame and an orbit polyline with a custom
  marching-dash glow material is drawn (rebuilt every 2 s, dense samples near the satellite).
- `SatelliteLayer.tsx`: `hidden` prop (canvas hidden, propagation continues) + exported `satEcefAt(id, date)`.
- `WorldviewMap.tsx`: pass `hidden={!!trackedSatId}`; CinematicController adds `ecefAt` to the satellite target.
Checklist: - [x] 10a aircraft scale/zoom HUD (`components/CinematicZoomControl.tsx`)  - [x] 10b ocean shader + spray
(`lib/ocean-shader.ts`)  - [x] 10c hide sat canvas  - [x] 10d orbit path (`lib/orbit-path.ts`)  — all code written, tsc clean.
- [ ] 10e browser verify (aircraft, ship, satellite) + tune

## Step 11 — Ship 3D on the REAL map + real forward motion (IN PROGRESS, 2026-10-19, repo github.com/daggerm29-oss/PLANK)
User report (with screenshots): ship view ocean was an artificial square tile floating in black space (zooming out
showed the whole tile, no land/coast); ship looked like it floated in place (feed speed/course = N/A -> speed 0).
Plan / files:
- `CinematicTracker.tsx` (ship): real globe = Esri World Imagery via `ArcGisMapServerImageryProvider` (discards
  "map data not available" ocean tiles) + Cesium World Terrain with water mask (animated real ocean, real coastlines),
  sky atmosphere + stars. Sea level sampled from terrain (geoid), neon ocean sits on it.
- `lib/ocean-shader.ts`: mesh becomes TRANSLUCENT; alpha fades radially (blends into the real sea),
  fades with camera range (zoom out -> pure real map), land-mask texture (terrain-sampled grid around ship) hides
  the neon water over land. JS port of the swell (`swellHeight`) drives ship heave/pitch/roll so hull rides the visible waves.
- `lib/tracked-entity.ts` `shipCruiseSpeedMs(category)` + `WorldviewMap.tsx`: when sog/cog unknown -> realistic
  estimated cruise speed (tanker 12 kn, cargo 14 kn, military 18 kn) so the ship sails forward through the
  world-anchored waves/grid with a wake.
Checklist: - [x] 11a real globe/terrain/imagery  - [x] 11b translucent fading ocean + land mask (+ neon coastline)
- [x] 11c estimated cruise speed (`motionEstimated` -> free-run dead reckoning, steers away from land via mask)
      + wave-driven heave/pitch/roll  — code written, tsc clean.
- [x] 11d browser verified (via `window.__wv.trackShip(mmsi)`):
      Port Qasim tanker 563005900 -> hull on real sea (terrain sea level ≈ -43.7 m), real land/imagery around;
      open-sea tanker 563094300 (7.77N 72.84E) -> neon ocean + wake on real globe, zoom out to 400 km shows
      real globe, atmosphere, stars, coast on the horizon; 207 km = top-down real ocean.
      Tuning: land threshold sea+3 m, clear channel of 3 ship lengths around the vessel (creeks/tidal flats),
      coast glow 0.5; camera auto-tilts to top-down when zooming out (`effectivePitch`); distance uses real
      elapsed time (`moveDt`, cap 1 s) so low-fps devices don't slow the ship. **Step 11 DONE.**
- Note: sandbox GPU renders ~1 fps, so live speed there reads low; on real hardware tanker = 6.2 m/s (12 kn).
- `window.__wv` debug hook in `WorldviewMap.tsx` (pre-existing) is used for testing; remove before production.

## Step 12 — LIVE AIRPORT VIEW (IN PROGRESS, started 2026-10-19, repo github.com/daggerm29-oss/PLANL)
Request: "drop the camera into any busy airport" -> photorealistic 3D airport with LIVE aircraft, including
aircraft on the ground (taxiing / parked) sitting on the runway/taxiway surface (like bilawalsidhu/gods-eye-view).
Approach (fits our architecture: Express proxy -> adsb.lol, Vite client, Cesium + Google 3D tiles):
- `api-server/src/routes/proxy.ts`:
  - `GET /api/proxy/adsb/airport?lat&lon&radius` (nm, 2-40, default 12) -> adsb.lol `/point`, 3 s cache,
    richer slim (alt_geom, baro_rate, true_heading, ground flag) so ground traffic is kept.
  - `GET /api/proxy/adsb/route/:callsign` -> adsbdb.com callsign route enrichment (origin/destination), 6 h cache + in-flight dedupe.
- `worldview/src/lib/airports.ts` — curated busy airports (ICAO/IATA/name/lat/lon/elevFt) + `nearestAirport()`.
- `worldview/src/lib/airport-traffic.ts` — pure helpers: normalise contact (ground detection: alt_baro "ground"
  or gs<40kt & AGL<60 m), model + scale per ADS-B category, AGL from field elevation, dead reckoning.
- `worldview/src/components/AirportView.tsx` — Cesium viewer (Google photorealistic tiles via exported
  `loadGoogleTileset` from CinematicTracker), free camera (orbit/pan/zoom), one `Cesium.Model` per contact,
  ground snapping: `scene.sampleHeightMostDetailed` at 5 points (min, so jet bridges/roofs/baked-in aircraft
  meshes don't lift it) + wheel offset (`model minY * scale`); airborne = field surface + (alt_baro - elev) AGL.
  Labels (callsign + GND/alt), click to select -> HUD info + route enrichment. Polls every 3 s.
- `worldview/src/components/AirportHud.tsx` — portal HUD (outside the vision filter): airport header,
  ground/airborne counts, contact list, selected contact card, view buttons (TOP/OBLIQUE), EXIT (Esc).
- `worldview/src/components/AirportPanel.tsx` — FloatingPanel airport picker (search) + "DROP AT MAP CENTRE".
- `WorldviewMap.tsx` — toolbar button "Airport View", `airportView` state, auto chip when map zoom >= 11 near
  a known airport, mounts AirportView inside the world stack (vision modes apply).
Checklist:
- [x] 12a proxy endpoints (`/proxy/adsb/airport`, `/proxy/adsb/info`)  - [x] 12b `lib/airports.ts` + `lib/airport-traffic.ts`
  (CinematicTracker now exports `loadGoogleTileset`, `TILESET_NIGHT_SHADER`, `applyModelVision`)
- [x] 12c `components/AirportView.tsx` + `components/AirportHud.tsx` written
- [x] 12d `components/AirportPanel.tsx` (+ `AirportProximityChip`), `AirportIcon` in MasterToolbar, WorldviewMap wiring
      (`enterAirport` / `exitAirport` / `dropAirportAtCenter`, moveend proximity check)
- [x] 12e tsc clean (only pre-existing `src/components/ui/*` errors). Browser verified at KJFK: 67-71 live contacts
      (~35 parked, ~15-20 taxi, rolling + ~16 airborne), GLB models sitting on the Google tiles surface, labels
      (parked labels only < 1.8 km to avoid gate clutter), click list -> fly-to + follow + cyan silhouette, adsbdb
      enrichment (AAL525 DFW -> ORD, A319, photo), Esc -> exits and restores Aviation panel.
      Fixes during verify: sky atmosphere/fog disabled (white-out once globe hidden); proxy returns `fetchedAt` so the
      HUD shows the real snapshot age (adsb.lol is intermittently unreachable -> cached snapshot served, "FEED DEGRADED" > 20 s).
      **Step 12 DONE.**
- Ideas not done yet: taxi trails draped on the surface, runway/taxiway overlay from OSM `aeroway`, aircraft pitch on
  climb/descent, show ground aircraft distinctly on the 2D map, larger airport DB (OurAirports CSV) for name lookup on custom drops.
- Note: the proxy routes are mounted by `vite.config.ts`, so changes in `api-server/src/routes/*` need a Vite restart.

## Step 13 — AIRCRAFT COCKPIT VIEW (DONE, started 2026-10-19, repo github.com/daggerm29-oss/PLANM)
Request: while in the aircraft 3D (chase) view, add a first-person COCKPIT view like the reference
(circular visor over photorealistic tiles, labelled nearby flights, GS / HDG / ALT readout, side intel panels,
RESET + EXIT COCKPIT). Built on our architecture: same Cesium viewer in `CinematicTracker.tsx`, same live adsb.lol
aircraft array from `WorldviewMap.tsx` (no new polling), route via existing `/api/proxy/adsb/info` (adsbdb).
Plan / files:
- `lib/cockpit.ts` — types (`CockpitContact`, `CockpitContext`, `CockpitTelemetry`), haversine distance/bearing,
  dead reckoning helpers.
- `lib/cockpit-contacts.ts` — Cesium Label+Point collections for nearby aircraft (<= 150 km), dead-reckoned
  from the last feed sample, depth test off so they read over tiles; shown only in cockpit.
- `components/CockpitHud.tsx` — portal HUD (outside vision filter, zIndex 5000): visor vignette + ring, heading tape,
  pitch ladder (rolls with bank), reticle, GS/HDG/ALT, top status bar (FIRST PERSON, callsign, vision mode, Z clock),
  left flight card, right CONTACT panel (nearest contact bearing/dist, cohort counts aircraft/military/AIS within 100 km)
  + ESTIMATED FLIGHT PLAN (adsbdb origin -> destination), RESET / EXIT COCKPIT buttons, help line.
- `CinematicTracker.tsx` — `cockpit` mode (aircraft only): camera at pilot eye (model hidden), heading + look offset,
  flight pitch from vertical rate, bank from turn rate, drag = look around, wheel/+/- = FOV, `C` toggles,
  sky atmosphere on + sun at local solar noon; exiting 3D drops back to chase first.
- `CinematicZoomControl.tsx` — optional COCKPIT button.
- `tracked-entity.ts` FollowTarget gains callsign/registration/typeCode/squawk/military.
- `WorldviewMap.tsx` CinematicController passes those + `getCockpitContext` (latest aircraft + ships via ref).
Checklist:
- [x] 13a lib/cockpit.ts + cockpit-contacts.ts
- [x] 13b CockpitHud.tsx (+ styles in components/cockpit-hud-css.ts)
- [x] 13c CinematicTracker cockpit camera + input + wiring, zoom-control COCKPIT button
- [x] 13d WorldviewMap wiring (target fields + `getCockpitContext` via useMemo/ref in CinematicController)
- [x] tsc clean (only pre-existing src/components/ui/* errors)
- [x] 13e browser verified (RYR9QX FL370 over France, ECA3ER): COCKPIT button / `C` -> visor HUD, 40+ labelled live
      contacts (callsign, FL, NM), nearest contact BRG/DIST/ΔALT, cohort counts (45 aircraft / 149 AIS within 54 NM),
      adsbdb route NCL -> LPA + airline, GS/HDG/ALT, heading tape, pitch ladder; EXIT COCKPIT -> chase view + model restored.
      Fix during verify: Google photorealistic tiles stay at coarse LOD at cruise altitude (giant blocks at grazing angles),
      so the cockpit uses the Esri imagery globe + fog above 2.5 km (`handles.cockpitGlobe`, hysteresis 2.0 km) and Google
      tiles below. **Step 13 DONE.**
- Ideas not done: Cesium World Terrain in cockpit (mountains on the horizon), declutter labels (only inside visor / top N
  by distance), radio/weather/news pages from the reference, cockpit for military jets with a different HUD colour.
- Workspace note: this v0 workspace was re-imported from PLANM (no .git); upload the changed files to GitHub.
  Run with `npm install` + `node scripts/dev.mjs` (or Vite alone on 5173 behind `next dev`).

## Step 14 — CCTV VIEWSHED (IN PROGRESS, started 2026-10-20)
Request: like the reference "God's Eye View" project — cycle CCTV coverage to VIEWSHED and every camera draws its
estimated coverage (where it reaches, where it goes blind). Reference builds a pose frustum (heading/pitch/FOV/range/
mount height) and clamps range with a ray pick. Our version, on our architecture (MapLibre 2D/2.5D map, cameras from
`/api/proxy/cctv/cameras` + OSM `/api/proxy/cctv/osm` via `hooks/use-cctv.ts`, NO new fetches):
- `lib/viewshed.ts` — pure math:
  * `estimatePose(camera)` — heading parsed from the feed's `direction` (numeric deg, N/NE/.., North, Northbound,
    N/B, "East Facing", "Facing South"); Caltrans/Florida/TfL have it, NYC/Maryland/Austin/Singapore don't ->
    heading unknown -> omnidirectional (360°) line-of-sight at reduced range (honest, labelled "HDG UNKNOWN").
    Per-source mount height / FOV / range defaults (highway 12 m/60°/320 m, urban 8 m/70°/170 m, OSM by camera:type).
  * `computeViewshed(pose, buildings)` — 2D ray cast (1 ray / ~2°) against building footprints from the basemap vector
    tiles (`openmaptiles` source, `building` layer, `render_height`, z13+). Building of height hb at distance d blocks
    everything beyond if hb >= mount, else shadow to d*H/(H-hb) (blocked if that reaches range). The camera's own
    building is ignored (point-in-polygon). Outputs visible polygon, blind polygons, 3D wedge bands, visible %.
- `components/CctvViewshedLayer.tsx` — MapLibre layers: ground fill + outline (source colour), blind fill (red),
  fill-extrusion wedge volume (height falls from mount height to 0 at range), optional 3D buildings; recomputed on
  moveend/idle for <= 250 cameras nearest the centre at z>=13 (CONE mode: unoccluded, z>=12).
- Coverage mode cycle OFF / CONE / VIEWSHED in `CctvPanel.tsx` (+ hotkey V while CCTV panel open), summary stats and
  selected camera pose (heading source, FOV, range, mount, visible %).
- `WorldviewMap.tsx` — state `cctvCoverage`, render layer, pass stats to the panel.
Checklist:
- [ ] 14a lib/viewshed.ts
- [ ] 14b CctvViewshedLayer.tsx
- [ ] 14c CctvPanel coverage control + stats, WorldviewMap wiring
- [ ] 14d tsc + browser verify (London / NYC / Caltrans at z16, pitched)

## Next steps
- Live-test ship (dblclick a dark-fleet dot) and satellite (3D ORBITAL, SATS) cinematic views; tune `KIND_CONFIG` in `CinematicTracker.tsx` (scale/range per kind).
- Aircraft wingtip/strobe lights use fixed metre offsets and look too wide vs the min-pixel-sized model; scale them or drop them.
- Exiting 3D currently also deselects the aircraft (legacy behaviour); ships/satellites keep their panel open.
- Remaining pre-existing TS errors are only in `src/components/ui/*` (React type version mismatch).

## Design for steps 4–5

New / changed files:
- `src/lib/tracked-entity.ts` — `EntityRef = {kind:"aircraft"|"ship"|"satellite", id}`, `FollowTarget`
  (lat, lon, altM, headingDeg, speedMs), model-URL/scale picker per entity.
- `src/components/EntityInfoPanel.tsx` — right-side panel (top-right, zIndex 30) with live name / position /
  movement / intel fields + "TRACK 3D" / "EXIT 3D" button. Stays visible during tracking.
- `src/components/CinematicTracker.tsx` — generic Cesium follow-cam (replaces aircraft-only CesiumFollowCam usage):
  camera.lookAt each frame with own drag/wheel orbit, dead-reckoning between samples (heading+speed),
  intro zoom-in + fade, exit zoom-out + fade then `onExited`. Trail polyline behind the model.
- `ShipsLayer.tsx` — `onSelect(ship)` on click, `onTrack(ship)` on dblclick.
- `SatelliteLayer.tsx` — `onSelect` + `onTrack` (dblclick) via the projected hit test.
- `WorldviewMap.tsx` — `selectedEntity` + `trackedEntity` state.

Models: `aeroplane.glb` (civil), `military-aircraft.glb` (mil — old code pointed at a missing `jet.glb`),
`ship.glb`, `tanker.glb` (TANKER), `satellite.glb`, `iss.glb` (ISS/ZARYA). All ~1.9 units long on X →
same `heading - 90°` orientation; scale per kind.

