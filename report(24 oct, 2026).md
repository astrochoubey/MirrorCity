# MirrorCity — Comprehensive Repository Analysis Report

**Report date:** 24 October 2026  
**Analyst role:** Senior Software Engineer · Software Architect · Technical Writer · Code Auditor  
**Repository:** `MirrorCity` (`astrochoubey/MirrorCity`)  
**Version:** 0.1.0  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Features and Functionalities](#2-features-and-functionalities)
3. [Architecture and System Design](#3-architecture-and-system-design)
4. [Technology Stack and Dependencies](#4-technology-stack-and-dependencies)
5. [Repository Structure and File-by-File Guide](#5-repository-structure-and-file-by-file-guide)
6. [Core Logic, Algorithms, and Implementation Details](#6-core-logic-algorithms-and-implementation-details)
7. [API and Interface Documentation](#7-api-and-interface-documentation)
8. [Data Model and Data Flow](#8-data-model-and-data-flow)
9. [Setup, Installation, and Usage](#9-setup-installation-and-usage)
10. [Testing, Validation, and Reliability](#10-testing-validation-and-reliability)
11. [Security, Privacy, and Performance](#11-security-privacy-and-performance)
12. [Current Implementation Status and Technical Debt](#12-current-implementation-status-and-technical-debt)
13. [Limitations, Risks, and Recommendations](#13-limitations-risks-and-recommendations)
14. [Onboarding Guide for a New Contributor](#14-onboarding-guide-for-a-new-contributor)
15. [Feature Inventory Matrix](#15-feature-inventory-matrix)
16. [Evidence and Traceability](#16-evidence-and-traceability)

---

## 1. Executive Summary

### What is MirrorCity?

MirrorCity is an **open-source, browser-based spatial operating system** designed to create 3D digital twins of real urban districts and cities. Its tagline — *"Plan the city before it happens"* — captures the purpose: enabling planners, engineers, disaster managers, and administrators to model, simulate, and compare urban interventions before committing resources in the real world.

### The Problem It Solves

Urban planning and emergency management suffer from fragmented data: road networks in one GIS system, sewer capacity in another, electrical assets in a third, and real-time sensor data in a fourth. During an emergency, crossing those boundaries is costly and slow. MirrorCity attempts to integrate all layers into one navigable, interactive 3D environment that runs entirely in a browser.

### Who It Is For

- **Urban planners** comparing land-use and infrastructure alternatives
- **District engineers** inspecting sewer, power, and road capacity
- **Disaster management teams** running flood, evacuation, and incident scenarios
- **Government administrators** reviewing capital works and risk
- **Researchers** studying computational urban modelling

### Primary Capabilities (as implemented today)

| Capability | Status |
|---|---|
| Real-time 3D district scene (Three.js) | Implemented, unverified end-to-end |
| Real-world geography from OpenStreetMap | Implemented, unverified |
| LiDAR reconstruction pipeline (Python/Open3D) | Implemented (assets committed) |
| IITH terrain GLB models | Implemented (assets in `public/models/iith/`) |
| Civic asset authoring (place, drag, rotate, scale) | Implemented, unverified |
| Infrastructure line and zone drawing | Implemented, unverified |
| 2-D local-inertial flood solver (Python/NumPy) | Implemented; 20 validation tests |
| Flood solver runs in browser via Pyodide | Implemented, unverified |
| SWMM storm-drain coupling (WebAssembly) | Implemented, integration tested |
| Sewer capacity scenario (SWMM output parsing) | Implemented, unverified |
| Evacuation clearance model (Dijkstra + max-flow) | Implemented, unverified |
| Live feeds (aircraft, quakes, weather, air quality) | Implemented, unverified |
| Room-scale digital twin with IoT sensor ingestion | Implemented, unverified |
| Clerk authentication and onboarding | Implemented, unverified |
| Cloudflare D1 SQLite database persistence | Implemented, unverified |
| Vector tile base layer (OpenFreeMap) | Implemented, unverified |
| Power-grid simulation (pandapower, Python) | Implemented (worker + script present) |

### Development Status

This is **Phase 0** of a 4-phase roadmap. The codebase is considerably larger and more complete than a minimal demo — it contains real simulation physics, live data parsers, an IoT sensor pipeline, a proper database schema, and a test suite. However, simulations use textbook assumptions rather than calibrated engineering models, the "city" in the workspace is procedural rather than georeferenced, and production security hardening is incomplete.

---

## 2. Features and Functionalities

### Category A — Core 3D City Engine

#### A1. Interactive Three.js District Scene

- **Purpose:** A persistent, real-time 3D environment in which all workspace features are visualised.
- **User-facing behaviour:** Orbit-controllable 3D city scene with buildings, roads, a river, vegetation, vehicles, sensor beacons, and scenario overlays.
- **Internal implementation:** `app/CityEngine.tsx` — boots a `THREE.WebGLRenderer` with ACES filmic tone mapping, PCF shadow maps, and a `THREE.Timer`-driven animation loop. The scene is a fixed 24 m × 18 m procedural district ("Varuna River Ward"). Buildings, utilities, simulation overlays, and placed assets are separate `THREE.Group` instances for selective visibility.
- **Key functions:** `createPrefab()` (asset geometry), `makePath()` (CatmullRom tube meshes for utility corridors), `disposeObject()` (GPU memory leak prevention), `toWorld()` / `toPercent()` (coordinate conversion).

#### A2. 3D Tiles Streaming (Experimental)

- **Purpose:** Stream an external OGC 3D Tiles tileset (e.g. Cesium city model) into the district scene.
- **Activation:** Add `?tileset=<url>` to the workspace URL.
- **Internal implementation:** `app/CityEngine.tsx` lines 357–396. Dynamic import of `3d-tiles-renderer/three`. Scales to fit the 24×18 m world box.

#### A3. Flood Depth Overlay on LiDAR Tiles

- **Purpose:** Visualise per-cell flood depth over a LiDAR scan in the same coordinate space.
- **Internal implementation:** `app/CityEngine.tsx` lines 400–422. `THREE.InstancedMesh` (one box per wet cell), coloured by depth on a blue gradient.

---

### Category B — Real-World Geography (Explore)

#### B1. Place Search and Geocoding

- **Internal implementation:** `app/explore/page.tsx` — debounced input (400 ms) calls `GET /api/geocode?q=<term>`, which proxies to Nominatim. Results shown as a dropdown.

#### B2. OpenStreetMap Scene Construction

- `app/api/osm/route.ts` — proxies Overpass API, snaps coordinates to ~11 m grid for caching, tries two mirrors, edge-caches 12 hours.
- `lib/osm.ts` — `parseOverpass()` builds typed `OsmScene` with `OsmBuilding[]`, `OsmRoad[]`, `OsmArea[]`. `buildingHeight()` uses a priority chain (tagged height → levels × 3.2 m → type-lookup → default 8 m). `clipRing()` (Sutherland–Hodgman) and `clipPolyline()` cut geometry to the scene bounding box.
- `app/explore/WorldEngine.tsx` — extrudes OSM building footprints (`THREE.ExtrudeGeometry`), tessellates road polylines.

#### B3. Vector Tile Base Layer (OpenFreeMap)

- **Purpose:** Always-available fast base layer shown while Overpass loads.
- `lib/tiles.ts` — fetches TileJSON, decodes Mapbox Vector Tiles (`@mapbox/vector-tile` + `pbf`), applies `DETAIL_BANDS` to drop small footprints at large radii (prevents GPU overload).

#### B4. Live Data Overlays

- `lib/live.ts` — pure parsers: `parseAircraft()` (OpenSky), `parseQuakes()` (USGS GeoJSON), `parseAir()` (Open-Meteo). All normalised to SI units.
- `app/explore/LivePanel.tsx` — aircraft table, quake list, air/weather cards.

---

### Category C — Simulation Workspace

#### C1. Scenario Panel

Three scenarios: `"sewer"` (flow simulation), `"flood"` (hazard model), `"evacuation"` (mobility model). Each has a metrics display and a configuration form.

#### C2. 2-D Local-Inertial Flood Solver

- **Physics:** Bates et al. (2010): `q = (q - g·h·dt·S) / (1 + g·dt·n²·|q| / h^(7/3))`. Green-Ampt infiltration, spatially variable Manning roughness, obstacle masking, open/closed boundaries, adaptive CFL=0.7 timestep, roof-drainage BFS routing.
- **Location:** `scripts/flood/solver.py` (1092 lines, `FloodSolver` class).
- **Browser execution:** `app/flood-worker.ts` — Web Worker that boots Pyodide, mounts Python files + LiDAR NPZ + SWMM .inp, runs the solver, posts back JSON summary and flood maps.
- **SWMM coupling:** EPA SWMM 5.2.4 compiled to WASM (`lib/swmm-engine/`). `scripts/flood/drainage.py` exchanges inflow/overflow per timestep.

#### C3. Sewer Capacity Scenario

`lib/sewer-network.ts` — `sewerNetwork(population)` generates an EPA SWMM 5 `.inp` file from CPHEEO design rules (135 L/person/day, 80% return factor, peak factor 3.0, 150–200 mm pipes, self-cleansing slopes). `parseSewerReport()` extracts peak flow ratios and surcharged manholes.

#### C4. Evacuation Clearance Model

`lib/evacuation.ts` — builds street graph, runs Dijkstra shortest path to exits, then Edmonds–Karp max-flow to find bottleneck capacity. Clearance = 5 min response + longest walk ÷ 1.2 m/s + population ÷ max_flow.

#### C5. Power Grid Simulation

`scripts/grid/district_grid.py` (pandapower) + `app/grid-worker.ts` (Pyodide Web Worker). Computes transformer loading, voltage drops, and losses.

---

### Category D — Asset Authoring

Assets available: IITH terrain models (3), backup generator, compact substation, flood pump station, sewer manhole, telecom tower, flood barrier, relief shelter, field hospital. Pointer events (pointerdown/move/up) in `CityEngine.tsx` handle drag/rotate/scale. GLTFLoader loads real GLBs when `.file` is set.

Infrastructure line drawing: sewer, power, water, road corridors rendered as CatmullRom tube meshes. Planning zone drawing: `THREE.ShapeGeometry`. Proposed buildings: semi-transparent blue boxes with floor count.

---

### Category E — Workspace Session Persistence

Anonymous session via `httpOnly` cookie (`mc_owner`, random UUID). Session rows in `sessions` table track the `ownerToken`. All workspace state (placed assets, lines, areas, planned buildings, population, scenario) persists to Cloudflare D1 via CRUD API routes.

---

### Category F — Room-Scale Digital Twin

`app/room/page.tsx` — self-contained Three.js scene. Loads a GLB model if configured, otherwise builds a procedural room (walls, floor, ceiling, edge trim). Polls `/api/rooms/:id/sensors` every 10 seconds.

IoT sensor ingestion: `POST /api/ingest` — validates `hardwareId`, `roomId`, `deviceSecret`, `readings`; applies 10-second in-memory rate limit; inserts one `sensorReadings` row per metric.

Device firmware: `firmware/room_sensor.ino` — Arduino sketch for NodeMCU/Wemos D1 Mini with DHT22, HC-SR501 PIR, MQ-135. Posts every 30 seconds via HTTPS.

---

### Category G — LiDAR Processing Pipeline

- **IITH reconstruction:** `scripts/reconstruct_iith_ground_models.py` — Open3D ground/non-ground separation, Ball-triangle meshing. Outputs GLB, OBJ, PLY, PNG, `manifest.json`. Input: 18,449 labelled points; three captures at 0.54°, 3.02°, 6.59° grade.
- **Semantic studies:** `scripts/reconstruct_lidar_models.py` — DBSCAN clustering, Trimesh, Matplotlib. Studies road, building mass, street-asset classes.
- **Flood grid rasterization:** `scripts/flood/lidar_terrain.py` — 2-m raster NPZ with layers: elevation, surface_type, roof_mask, obstacle_mask, sink_mask.

---

### Category H — Authentication

Clerk (`@clerk/clerk-react`) wraps the app in `ClerkProvider`. Landing page uses `SignedIn/SignedOut` components. User profile (home place, layer prefs) stored in Clerk's `unsafeMetadata.mirrorcity`. First-time users see `app/components/Onboarding.tsx`.

---

## 3. Architecture and System Design

### Architectural Style

Full-stack monorepo. React 19 SPA frontend + Cloudflare Workers backend + Cloudflare D1 SQLite. Python simulations run offline (scripts) or in-browser (Pyodide WASM). No SSR — the app is fully client-rendered.

### Component Map

```
Browser
├─ React 19 App Router (SPA, all pages are "use client")
├─ CityEngine.tsx        — Three.js district scene (24×18 m procedural ward)
├─ WorldEngine.tsx       — Three.js OSM scene (real-world geography)
├─ room/page.tsx         — Three.js room twin
├─ flood-worker.ts       — Web Worker → Pyodide → Python flood solver
├─ grid-worker.ts        — Web Worker → Pyodide → Python power grid
└─ SWMM WASM             — EPA SWMM 5.2.4 (lib/swmm-engine/)

Cloudflare Edge (Worker)
├─ vinext/app-router     — Route all API routes
├─ Cloudflare D1         — SQLite session/asset/sensor data
├─ Edge Cache            — Overpass responses (12 h TTL)
└─ Image Optimization    — Cloudflare Images binding

External (all keyless)
├─ OpenFreeMap           — Vector tile base layer
├─ Overpass API          — OSM geometry (unreliable; used with fallback)
├─ Nominatim             — Geocoding
├─ OpenSky               — Live aircraft
├─ USGS                  — Earthquake feed
├─ Open-Meteo            — Weather + air quality
└─ Clerk                 — User authentication
```

### State Management

Local React `useState` throughout — no external store. Workspace page holds all simulation, tool, and asset state. User profile in Clerk `unsafeMetadata`. Persistent workspace data in D1 via REST.

### Key Design Decisions

1. **Tile-first / Overpass-upgrade pattern** (`useScene.ts`): Vector tiles load instantly and always succeed; Overpass replaces them if it responds. Users always see a scene rather than a spinner.
2. **Python in the browser via Pyodide**: The same `scripts/flood/` files run both as a CLI (for testing) and in the browser (for the simulation UI). No rewrite required.
3. **Anonymous sessions**: Users can save and share workspace state without creating accounts — important for government offices without SSO.
4. **Edge caching for Overpass**: Coordinates are snapped to a shared grid; responses are cached for 12 hours on Cloudflare's edge, reducing pressure on the free shared Overpass service.

---

## 4. Technology Stack and Dependencies

| Technology | Version | Purpose |
|---|---|---|
| React | 19.3.0 | UI framework (RSC + client components) |
| Three.js | ^0.185.1 | 3D rendering |
| Vite | 8.3.1 | Build bundler |
| vinext | 1.0.0 | Next.js + Cloudflare Workers adapter |
| @vitejs/plugin-react | 6.0.2 | React plugin for Vite |
| @vitejs/plugin-rsc | 0.5.35 | React Server Components (experimental) |
| @cloudflare/vite-plugin | 1.49.0 | Cloudflare Workers build integration |
| Drizzle ORM | 0.45.2 | Type-safe SQLite ORM for D1 |
| drizzle-kit | 0.31.10 | Schema migration generation |
| @clerk/clerk-react | ^5.61.10 | Auth UI components |
| @clerk/backend | ^3.21.1 | Server-side auth token verification |
| Pyodide | 314.0.7 | CPython 3.14 compiled to WebAssembly |
| 3d-tiles-renderer | 0.5.3 | OGC 3D Tiles streaming (experimental) |
| @mapbox/vector-tile | ^3.0.0 | Decode Mapbox Vector Tiles |
| pbf | ^5.1.2 | Protocol Buffers decoder |
| Tailwind CSS | 4.2.1 | CSS utility framework |
| TypeScript | 5.9.3 | Static typing |
| ESLint | 9.39.4 | Linting |
| eslint-plugin-jsx-a11y | 6.10.2 | Accessibility linting |
| Wrangler | 4.116.0 | Cloudflare Workers local runtime |
| Python 3 | ≥3.10 | LiDAR processing, flood/grid simulation |
| NumPy | Pyodide wheel | Flood solver array math |
| Open3D | requirements-open3d.txt | Point cloud processing and meshing |
| Trimesh | requirements-open3d.txt | Mesh processing and export |
| pandapower | implied | Power flow simulation |
| EPA SWMM | 5.2.4 (WASM) | Storm-drain hydraulics |

**Notable notes:**
- **vinext 1.0.0**: Makes Next.js App Router pages run on Cloudflare Workers. Not widely documented externally.
- **Pyodide 314.0.7**: NumPy wheel vendored into `public/pyodide/` by a predev script (`scripts/vendor-pyodide.mjs`) with checksum verification. Adds multi-MB download on first run.
- **EPA SWMM WASM**: Built by `scripts/build-swmm-wasm.sh`, committed in `lib/swmm-engine/`.
- **`@vitejs/plugin-rsc` 0.5.35**: Early-access RSC dependency — APIs are experimental.

---

## 5. Repository Structure and File-by-File Guide

```
MirrorCity/
├── app/
│   ├── CityEngine.tsx              ★ Three.js 3D district scene
│   ├── CityEngineErrorBoundary.tsx   React error boundary
│   ├── ModelViewer.tsx               Standalone GLB viewer
│   ├── flood-worker.ts             ★ Web Worker: Pyodide + flood solver
│   ├── grid-worker.ts                Web Worker: Pyodide + power grid
│   ├── globals.css                   Full app CSS (~3800 lines)
│   ├── layout.tsx                    Root layout (ClerkProvider, fonts)
│   ├── page.tsx                      Landing page (auth gate, launchpad)
│   ├── providers.tsx                 React providers (Clerk)
│   ├── api/
│   │   ├── _session-auth.ts        ★ Cookie auth + admin token check
│   │   ├── _validate.ts              Shared input validation helpers
│   │   ├── areas/route.ts            CRUD: drawn areas
│   │   ├── assets/route.ts           CRUD: placed assets
│   │   ├── buildings/route.ts        CRUD: planned buildings
│   │   ├── geocode/route.ts          GET: Nominatim geocoding proxy
│   │   ├── ingest/route.ts         ★ POST: IoT sensor ingestion
│   │   ├── lines/route.ts            CRUD: drawn utility lines
│   │   ├── live/route.ts             Live feed proxy
│   │   ├── osm/route.ts            ★ GET: Overpass proxy + edge cache
│   │   ├── rooms/route.ts            CRUD: room-scale twin rooms
│   │   ├── sensor-readings/route.ts  GET: reading history
│   │   ├── sensors/route.ts        ★ CRUD: sensor nodes (admin-gated)
│   │   ├── session/route.ts          GET/POST: workspace session
│   │   └── uploads/route.ts          Upload metadata tracking
│   ├── components/
│   │   ├── AssetPanel.tsx            Asset library sidebar
│   │   ├── Icon.tsx                  SVG icon component
│   │   ├── LayerControls.tsx         Layer visibility toggles
│   │   ├── Onboarding.tsx            First-time user onboarding
│   │   └── ScenarioPanel.tsx         Scenario selector + metrics
│   ├── explore/
│   │   ├── page.tsx                ★ Place search, OSM loading, live data
│   │   ├── WorldEngine.tsx           Three.js OSM scene renderer
│   │   ├── LivePanel.tsx             Aircraft/quake/weather overlay
│   │   └── useScene.ts               Tile + Overpass scene lifecycle hook
│   ├── lab/flood/, lab/grid/, lab/sewer/   Lab sub-routes
│   ├── room/page.tsx               ★ Room twin viewer + sensor polling
│   └── workspace/page.tsx          ★ Main simulation workspace (1009 lines)
├── db/
│   ├── index.ts                      D1 client factory
│   └── schema.ts                   ★ Drizzle ORM schema (all tables)
├── drizzle/                          Auto-generated SQL migrations
├── firmware/
│   └── room_sensor.ino             ★ ESP8266 Arduino sketch
├── lib/
│   ├── city-metrics.ts               FloodResults/FloodRun types + formatters
│   ├── evacuation.ts               ★ Evacuation: graph, Dijkstra, max-flow
│   ├── geo.ts                        Geodetic utils (project, distanceM, boundingBox)
│   ├── live.ts                     ★ Live feed parsers (aircraft, quakes, air)
│   ├── osm.ts                      ★ Overpass parser + polygon clipper
│   ├── places.ts                     Curated demo places
│   ├── platform.ts                   Cloudflare env binding accessor
│   ├── sewer-network.ts            ★ SWMM .inp generator + report parser
│   ├── swmm.ts                       EPA SWMM WASM wrapper
│   ├── swmm-engine/                  EPA SWMM 5.2.4 compiled to WASM
│   └── tiles.ts                    ★ OpenFreeMap vector tile decoder
├── public/
│   ├── og.png                        Open Graph hero image
│   ├── pyodide/                      Vendored Pyodide + NumPy (gitignored)
│   └── models/iith/, models/lidar/   LiDAR-derived 3D assets
├── scripts/
│   ├── flood/                      ★ 2-D flood solver (Python/NumPy)
│   │   ├── solver.py               ★ FloodSolver class (1092 lines)
│   │   ├── run_simulation.py       ★ CLI entry point
│   │   ├── validation.py           ★ 20 physics validation tests
│   │   ├── drainage.py               SWMM coupling
│   │   ├── infiltration.py           Green-Ampt
│   │   ├── lidar_terrain.py        ★ LiDAR PLY → NPZ rasterizer
│   │   ├── boundary.py, obstacles.py, rainfall.py, roughness.py, terrain.py
│   │   └── data/                     NPZ + SWMM .inp files
│   ├── grid/
│   │   ├── district_grid.py          pandapower simulation
│   │   ├── district_buildings.json   Synthetic load data
│   │   └── validation.py             Grid tests
│   ├── reconstruct_iith_ground_models.py  ★ Open3D LiDAR-to-GLB pipeline
│   ├── reconstruct_lidar_models.py        Semantic reconstruction studies
│   ├── build-swmm-wasm.sh                 SWMM WASM build script
│   ├── db-migrate-local.mjs               Local D1 migration runner
│   ├── requirements-open3d.txt            Python LiDAR dependencies
│   └── vendor-pyodide.mjs                 Predev: vendor Pyodide wheel
├── tests/
│   ├── osm.test.mjs              OSM parser + projection unit tests
│   ├── tiles.test.mjs            Vector tile decoder tests
│   ├── live.test.mjs             Live feed parser tests
│   ├── evacuation.test.mjs       Evacuation model tests
│   ├── sewer-network.test.mjs    Sewer network tests
│   ├── flood-drains.test.mjs   ★ Pyodide + SWMM integration test
│   ├── grid-packages.test.mjs    Grid result parsing tests
│   ├── rendered-html.test.mjs    HTML smoke tests
│   ├── cameras.test.mjs          Camera feed tests
│   ├── facilities.test.mjs       Facility overlay tests
│   └── fixtures/                 Captured API responses
├── worker/index.ts               Cloudflare Worker entry point
├── .env.example                  Env vars template (Clerk keys)
├── .dev.vars.example             Local secrets template
├── drizzle.config.ts             Drizzle Kit config
├── package.json                  Scripts, deps, Node ≥ 22.13 constraint
├── tsconfig.json                 TypeScript strict config
├── vite.config.ts                Vite + Cloudflare + React plugins
└── vercel.json                   Vercel deployment config
```

---

## 6. Core Logic, Algorithms, and Implementation Details

### 6.1 Local-Inertial Flood Solver (Bates et al., 2010)

**File:** `scripts/flood/solver.py`

**Governing equation (per face):**
```
q_new = (q_prev - g·h·dt·∂η/∂x) / (1 + g·dt·n²·|q_prev| / h^(7/3))
```

- `q` = unit discharge [m²/s]
- `g` = 9.81 m/s²  
- `h` = max(η_left, η_right) − max(z_left, z_right) [flow depth between cells]
- `η` = z + depth [water surface elevation]
- `n` = Manning roughness
- `dt` = adaptive timestep (CFL = 0.7: `dt = 0.7 · min(dx,dy) / sqrt(g · max_depth)`)

**Sub-models:**
- **Infiltration:** Green-Ampt (`f = Ks · (1 + ψ·Δθ/F)`); impervious surfaces coded in `IMPERVIOUS_SURFACES` dict.
- **Roof drainage:** BFS assigns each roof cell a nearest open-ground outlet; roof rain is added to the outlet instead of accumulating on the roof.
- **Outflow limiter:** Scales discharges so a cell cannot lose more water than it holds — prevents artificial water creation.
- **Storm drains:** `DrainageCoupling` passes inflow to SWMM per inlet per timestep; receives overflow back.
- **Boundaries:** Open (free-draining via ghost cell) or closed (no-flux) per edge.

**Complexity:** O(N·T) where T ∝ 1/sqrt(h), so deeper water → smaller dt → more iterations.

### 6.2 OSM Geometry Processing Pipeline

**File:** `lib/osm.ts`

**Building height priority chain (`buildingHeight()`):**
1. `height` tag (metres or feet, unit-aware)
2. `building:height` tag
3. `building:levels × 3.2 m + roof:levels × 3.2 m`
4. Type-lookup table (e.g. `apartments: 17 m`, `hospital: 11 m`)
5. Default 8 m fallback

**Clipping:** `clipRing()` implements Sutherland–Hodgman exact polygon clipping against bounding box half-planes. `clipPolyline()` clips polylines segment-by-segment, collecting runs of consecutive inside points. Without clipping, a national highway would span hundreds of kilometres in a 700-m scene.

### 6.3 Evacuation Model

**File:** `lib/evacuation.ts`

1. **Street graph** (`streetGraph()`): nodes at endpoints, intersections, and building access projections (closest point on nearest street segment).
2. **Dijkstra** (`distanceToExit()`): shortest path from all boundary exit nodes inward.
3. **Edmonds–Karp** max-flow: edge capacity = 1.3 persons/m/s × street width. Identifies bottleneck streets.
4. **Clearance** = 5 min + longest_walk ÷ 1.2 m/s + population ÷ max_flow.

Assumptions: unimpeded walking 1.2 m/s, SFPE Handbook specific flow 1.3 p/m/s, street widths 7 m/unit, no crowd dynamics.

### 6.4 Vector Tile Decoding

**File:** `lib/tiles.ts`

Fetches TileJSON from OpenFreeMap → derives tile URL template → for each tile:
1. Fetch `.pbf` binary
2. Decode with `@mapbox/vector-tile` + `pbf`
3. Project to local metres (Mercator → lat/lon → `project()`)
4. Filter by `DETAIL_BANDS` (footprints under 150 m² dropped beyond 1.5 km; under 400 m² beyond 4 km; under 900 m² beyond 7 km)
5. Emit same `OsmBuilding`/`OsmRoad`/`OsmArea` types as `lib/osm.ts`

### 6.5 LiDAR Terrain Rasterization

**File:** `scripts/flood/lidar_terrain.py`

Labels: 0=road, 1=concrete, 2=grass, 3=roof, 4=wall, 5=outside.

Per 2-m cell: elevation = mean of ground-class (0–2) point heights; surface_type = most common non-obstacle label; roof_mask = any roof (3) point; obstacle_mask = any wall (4) point; sink_mask = only outside (5) points. Empty cells filled by nearest-neighbour interpolation.

Output: `scripts/flood/data/lidar-street.npz` — committed pre-built grid.

---

## 7. API and Interface Documentation

### REST API Reference

#### Session Management

| Method | Route | Purpose | Auth |
|---|---|---|---|
| GET | `/api/session` | Load workspace session + child rows | Cookie |
| POST | `/api/session` | Create session, set `mc_owner` cookie | — |
| PUT | `/api/session` | Update scenario/population/layers | Cookie (owner) |

#### Workspace Assets

| Method | Route | Purpose | Auth |
|---|---|---|---|
| GET/POST/PUT/DELETE | `/api/assets` | CRUD placed assets | Cookie (owner) |
| GET/POST/DELETE | `/api/lines` | CRUD drawn lines | Cookie (owner) |
| GET/POST/DELETE | `/api/areas` | CRUD planning zones | Cookie (owner) |
| GET/POST/DELETE | `/api/buildings` | CRUD proposed buildings | Cookie (owner) |

#### Geography

| Method | Route | Notes |
|---|---|---|
| GET | `/api/geocode?q=<term>` | Nominatim proxy |
| GET | `/api/osm?lat=&lon=&radius=` | Overpass proxy, snapped to 11 m grid, edge-cached 12 h, radius 100–1500 m |

#### Room-Scale Twin

| Method | Route | Auth |
|---|---|---|
| GET/POST | `/api/rooms` | None / Admin token |
| GET | `/api/sensors?roomId=N` | None |
| POST/PUT | `/api/sensors` | Admin token |
| POST | `/api/ingest` | Device secret |
| GET | `/api/sensor-readings?sensorId=N` | None |

#### POST /api/ingest — Full Specification

```json
// Request body
{
  "hardwareId": "ESP_A1B2",
  "roomId": 1,
  "deviceSecret": "uuid-issued-at-registration",
  "readings": {
    "temperature": 28.4,  // °C
    "humidity": 62.1,     // %
    "occupancy": 1,       // 1=detected, 0=empty
    "co2": 842            // ppm
  }
}

// Response 200
{ "ok": true, "sensorId": 3, "inserted": 4 }

// Error codes
// 400 — missing fields, blank strings, NaN numbers
// 401 — unknown device, wrong deviceSecret
// 429 — rate limited (< 10 s since last ingest from this device)
// 500 — database error
```

### Application Routes (UI)

| Route | Purpose |
|---|---|
| `/` | Landing: auth gate → onboarding → launchpad |
| `/explore` | Real-world 3D geography explorer |
| `/workspace` | Full simulation workspace |
| `/room` | Room-scale digital twin |
| `/lab/flood`, `/lab/grid`, `/lab/sewer` | Lab sub-views |

### Python CLI

```bash
# Flood simulation
python3 -m scripts.flood.run_simulation \
  [--terrain <path.npz>] [--duration 3600] [--rainfall 100] \
  [--manning 0.04] [--infiltration-k 1e-5] \
  [--open-edges] [--drains <path.inp>] [--output <dir>]

# Rebuild LiDAR terrain grid
python3 -m scripts.flood.lidar_terrain

# Run validation tests
python3 -m scripts.flood.validation

# IITH terrain reconstruction
python scripts/reconstruct_iith_ground_models.py \
  --dataset /path/to/IITH_LiDAR_ground_dataset_labelled_raw \
  --output public/models/iith
```

---

## 8. Data Model and Data Flow

### Database Schema (Cloudflare D1 / SQLite)

```
sessions         — id, districtName, population, floodRainfall, floodStormMinutes,
                   activeScenario, layers(JSON), label, ownerToken, createdAt, updatedAt

placed_assets    — id, sessionId→sessions, assetId, assetName, x, y, rotation, scale, createdAt
drawn_lines      — id, sessionId→sessions, kind(sewer|power|water|road), points(JSON), createdAt
drawn_areas      — id, sessionId→sessions, points(JSON), createdAt
planned_buildings— id, sessionId→sessions, x, y, floors, createdAt

rooms            — id, name, glbPath, widthM, depthM, heightM, createdAt

sensors          — id, roomId→rooms, hardwareId(UNIQUE), name, type,
                   x, y, z(0–1 normalised), active, deviceSecret, createdAt

sensorReadings   — id, sensorId→sensors, metric, value, unit, recordedAt

uploads          — id, sessionId→sessions, filename, fileType, sizeBytes,
                   status, pointCount, message, createdAt
```

### Serialisation Formats

| Data | Format | Location |
|---|---|---|
| Flood terrain grid | NumPy NPZ (compressed) | `scripts/flood/data/lidar-street.npz` |
| Storm-drain network | EPA SWMM 5 `.inp` text | `scripts/flood/data/lidar-street-drains.inp` |
| Simulation output | JSON | `/tmp/flood-out/summary.json` (Pyodide FS) |
| Drawn lines/areas | JSON strings in SQLite TEXT | `drawn_lines.points`, `drawn_areas.points` |
| 3D terrain models | GLB, OBJ, PLY | `public/models/iith/` |

### Flood Simulation Data Flow

```
rainfall (mm/h) + duration (s) + lidar-street.npz + lidar-street-drains.inp
  → flood-worker.ts (Web Worker)
    → Pyodide boot: NumPy loaded, .py files + .npz + .inp mounted
    → createSwmmStepper() → SWMM WASM initialised
    → FloodSolver.__init__(): elevation, roughness, infiltration, masks loaded
    → FloodSolver.run():
        loop: choose_dt → rainfall → infiltration → SWMM exchange → fluxes → update depth
    → writes /tmp/flood-out/summary.json
  → postMessage({ok, summary, bootMs, runMs})
→ workspace/page.tsx: updates floodRun state
→ CityEngine.tsx: renders InstancedMesh water columns over LiDAR tile group
```

### Real-World Scene Data Flow

```
User types place name
  → 400ms debounce → GET /api/geocode?q=... → Nominatim → lat/lon

useScene(lat, lon, radius):
  → lib/tiles.ts: fetch OpenFreeMap vector tiles → OsmScene (rendered immediately)
  → GET /api/osm?lat=&lon=&radius=:
      → Overpass API (may queue, may fail; 30s timeout; 2 mirrors)
      → edge-cached response (12h TTL)
  → lib/osm.ts parseOverpass() → detailed OsmScene
  → WorldEngine.tsx replaces tile scene with Overpass scene
```

---

## 9. Setup, Installation, and Usage

### Prerequisites

- **Node.js ≥ 22.13** (enforced in `package.json` `engines`)
- **npm** (bundled with Node.js)
- **Modern browser with WebGL support**
- **Python 3** (≥3.10) — only for LiDAR reconstruction or Python tests
- **A Clerk project** (free tier available at clerk.com)

### Installation

```bash
git clone https://github.com/SASTxNST/MirrorCity.git
cd MirrorCity
npm install                        # also runs vendor-pyodide.mjs predev
cp .env.example .env.local         # fill in VITE_CLERK_PUBLISHABLE_KEY
cp .dev.vars.example .dev.vars     # fill in CLERK_SECRET_KEY
npm run db:migrate:local           # create local D1 database
npm run dev                        # → http://localhost:3000
```

### Environment Variables

| Variable | Required | File | Purpose |
|---|---|---|---|
| `VITE_CLERK_PUBLISHABLE_KEY` | Yes | `.env.local` | Clerk publishable key (safe to expose) |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Yes | `.env.local` | Next.js compat alias |
| `CLERK_SECRET_KEY` | Yes | `.dev.vars` | Clerk server-side secret |
| `ROOM_ADMIN_TOKEN` | Optional | `.dev.vars` | Admin-gated sensor endpoints |

> **Never commit `.env.local` or `.dev.vars`.** Both are gitignored.

### Key npm Scripts

| Script | Purpose |
|---|---|
| `npm run dev` | Start local dev server with Cloudflare Workers runtime |
| `npm run build` | Production Vite build |
| `npm test` | Run all JavaScript tests (builds workers first) |
| `npm run flood:test` | Run Python flood physics validation (20 tests) |
| `npm run flood:run` | Run flood simulation CLI against default terrain |
| `npm run grid:test` | Run Python grid validation |
| `npm run db:generate` | Generate Drizzle SQL migrations from schema changes |
| `npm run db:migrate:local` | Apply migrations to local D1 database |
| `npm run lint` | ESLint TypeScript rules |

### Register a Room Sensor

```bash
# 1. Admin registers the device (gets back deviceSecret)
curl -X POST http://localhost:3000/api/sensors \
  -H "Authorization: Bearer $ROOM_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"roomId":1,"hardwareId":"ESP_A1B2"}'

# 2. Flash the ESP8266 with DEVICE_SECRET from response
# 3. Device auto-posts to /api/ingest every 30 seconds
```

---

## 10. Testing, Validation, and Reliability

### JavaScript/TypeScript Tests (`tests/`)

Run with `npm test` (Node.js built-in `node:test`, no jest/vitest).

| File | What it tests |
|---|---|
| `osm.test.mjs` | Projection round-trips, bounding box, `parseLength`, `buildingHeight`, `roadWidth`, `clipRing`, `clipPolyline`, `parseOverpass` |
| `tiles.test.mjs` | Vector tile decoding, `DETAIL_BANDS` filtering |
| `live.test.mjs` | `parseAircraft`, `parseQuakes`, `parseAir` with captured fixtures |
| `sewer-network.test.mjs` | SWMM .inp generation and report parsing |
| `evacuation.test.mjs` | Clearance time, bottleneck identification, max-flow correctness |
| `flood-drains.test.mjs` | **Integration:** Pyodide runs Python solver + SWMM drain coupling; verifies drain capture and water budget closure to 1 ppm |
| `rendered-html.test.mjs` | Rendered HTML smoke test |
| `cameras.test.mjs` | Traffic camera feed parsing |
| `facilities.test.mjs` | Facility overlay data |
| `grid-packages.test.mjs` | Power grid result parsing |

### Python Flood Validation (`scripts/flood/validation.py`)

20 physics tests:
1. Rainfall volume conservation
2. Water non-negativity
3. Water moves downhill
4. Depressions accumulate water
5. Infiltration removes water
6. Obstacles block transport
7. Closed boundaries conserve mass
8. Water wets dry cells
9. Drains do not create water
10. Open boundaries drain correctly
11. LiDAR grid matches committed NPZ
12. Impervious surfaces soak nothing in
13. Roof rain drains to ground
14. Sink cells remove water
15. Storm drains remove water up to capacity
16. **Benchmark:** Manning normal-depth analytic profile
17. **Benchmark:** Lake at rest stays at rest
18. **Benchmark:** Water in closed tilted box settles flat
19. Drain coupling conserves water; full network backs up
20. Committed drain network matches its generator

### Error Handling

- **API routes:** All wrapped in try/catch; JSON error objects with appropriate HTTP status.
- **CityEngine:** Wrapped in `CityEngineErrorBoundary.tsx`.
- **flood-worker.ts:** Posts `{ ok: false, error: string }` on any exception.
- **Overpass proxy:** Checks response body content (Overpass may return HTML errors under HTTP 200), falls back across two mirrors, returns 502 with user-friendly message.
- **Pyodide:** Falls back from local `/pyodide/` to jsDelivr CDN if local copy missing.

### Not Tested

- End-to-end browser rendering (no Playwright)
- API routes with a real database (no integration tests against D1)
- Clerk auth flow
- Three.js scene correctness (no visual regression)
- LiDAR reconstruction pipeline

---

## 11. Security, Privacy, and Performance

### Security Findings

#### S1 — Anonymous Session Auth (Medium, Intentional for Phase 0)

Session cookie (`mc_owner`) is a random UUID issued to any browser. Any device with the cookie value can mutate that session. `SameSite=Lax` provides partial CSRF protection. **Fix in Phase 3:** tie session ownership to Clerk user ID.

#### S2 — In-Memory Rate Limiting (Medium)

`POST /api/ingest` rate limiter is a `Map<string, number>` that resets on Worker cold start and is not shared across isolates. **Fix:** Cloudflare Durable Objects or KV-based limiter.

#### S3 — Certificate Validation Disabled in Firmware (Low / Lab Only)

`firmware/room_sensor.ino` line 167: `client->setInsecure()`. Acceptable for local experiments; must be replaced with certificate pinning before production deployment.

#### S4 — `timingSafeEqual` for Admin Token (Good Practice)

`app/api/_session-auth.ts` lines 60–61 use `crypto.subtle.timingSafeEqual` — prevents timing attacks on the admin token comparison.

#### S5 — Device Secret Rotation on Re-Registration (Good Practice)

Re-calling `POST /api/sensors` with an existing `hardwareId` rotates its secret, allowing an admin to reclaim a compromised device.

#### S6 — No Secrets Committed (Good Practice)

`.env.local` and `.dev.vars` are gitignored. No secrets found in committed source code.

### Performance Observations

- **Pyodide cold boot:** 20–60 s on first run. The worker caches the boot promise (`booting ??= boot()`) so subsequent simulations are fast.
- **Flood simulation:** `flood-drains.test.mjs` uses `timeout: 120_000 ms` — 1-hour simulations are compute-intensive under Pyodide.
- **Overpass:** 20–30 s and fails ~30% of the time by the authors' own measurement. Vector tile fallback mitigates this.
- **Three.js geometry disposal:** `disposeObject()` called consistently — no GPU memory leaks.
- **InstancedMesh for flood overlay:** Correct choice for rendering thousands of per-cell water columns efficiently.
- **Detail bands:** Smart GPU budget management for large view radii.

---

## 12. Current Implementation Status and Technical Debt

### Status Overview

All major features are **implemented but unverified end-to-end** in a browser against live services and a real database. The flood solver and its tests are the most thoroughly validated component.

### Known Technical Debt

1. **Uncalibrated simulation parameters.** Manning roughness, Green-Ampt Ks, sewer design constants, walking speeds — all textbook defaults, no field calibration. Explicitly acknowledged in code comments.

2. **Procedural "city" in workspace.** The Varuna River Ward in `app/workspace/page.tsx` uses hardcoded building coordinates, not real data. Street data in `evacuation.ts` and `sewer-network.ts` is fictional.

3. **Flat terrain in `/explore`.** No elevation data integrated. Flood/sewer solvers cannot yet run over real places.

4. **In-memory rate limiting** (see S2).

5. **`workspace/page.tsx` is a 1009-line monolith.** All scenario state, workers, panels, and rendering props in one component.

6. **`tsconfig.tsbuildinfo` (521 KB) committed.** Build artefact should be gitignored.

7. **Upload handling metadata-only.** No file storage (R2/S3) — only the database row is written.

8. **`3d-tiles-renderer` spike** active in production via query param — not gated by a feature flag.

9. **pandapower in Pyodide.** pandapower is not a standard Pyodide package. Grid simulation may fail in browser if the wheel is unavailable.

10. **`@vitejs/plugin-rsc` 0.5.35** is early-access — APIs may have breaking changes.

---

## 13. Limitations, Risks, and Recommendations

### P1 — High Priority

**P1-1: Uncalibrated physics.** All simulation outputs are illustrative. Decision-makers must understand the uncertainty bounds before using outputs for real planning. **Recommendation:** Phase 2 calibration pipeline — ingest observed flood/flow data, run parameter optimisation, document bounds.

**P1-2: No production access control.** Single shared admin token, no per-user roles, no audit log. **Recommendation:** Tie session and sensor ownership to Clerk user IDs in Phase 3; add RBAC.

**P1-3: In-memory rate limiting.** Not durable across Worker instances. **Recommendation:** Cloudflare Durable Object or KV rate limiter.

### P2 — Medium Priority

**P2-1: Monolithic workspace page.** Extract scenario state into React Context; split panels into lazy-loaded sub-components. **Effort:** Medium.

**P2-2: Pyodide cold start (~30–60 s).** Show a clear progress indicator; pre-boot Pyodide on page load via idle callback. **Effort:** Small.

**P2-3: No real elevation in `/explore`.** Integrate Copernicus DEM or SRTM tiles for realistic terrain in WorldEngine. **Effort:** Large.

**P2-4: Upload route is metadata-only.** Integrate Cloudflare R2. **Effort:** Medium.

### P3 — Low Priority

**P3-1: Firmware TLS validation disabled.** Pin Cloudflare root CA or use device certificates for production. **Effort:** Small.

**P3-2: `tsconfig.tsbuildinfo` in git.** Add to `.gitignore`. **Effort:** Trivial.

**P3-3: Lab routes undocumented.** Document `/lab/flood`, `/lab/grid`, `/lab/sewer`. **Effort:** Small.

**P3-4: `3d-tiles-renderer` spike in production.** Gate behind explicit feature flag or remove from main. **Effort:** Trivial.

---

## 14. Onboarding Guide for a New Contributor

### Suggested Reading Order

1. **README.md** — project vision, capabilities, roadmap
2. **`db/schema.ts`** — central data model; understand before any backend work
3. **`lib/osm.ts`** — well-commented, pure, directly tested; good first module to read
4. **`lib/evacuation.ts`** — small, self-contained; illustrates the simulation philosophy
5. **`app/CityEngine.tsx`** — read `createPrefab()`, `makePath()`, animation loop, flood overlay
6. **`app/workspace/page.tsx`** — largest file; skim type defs + useState first, trace one scenario end-to-end
7. **`scripts/flood/solver.py`** — read module docstring, `__init__`, `step()`
8. **`app/flood-worker.ts`** — short, well-commented; shows how Python runs in browser
9. **`app/explore/page.tsx` + `useScene.ts`** — tile-first / Overpass-upgrade pattern

### Key Conventions to Understand

- **World coordinate system:** `WORLD_W = 24 m, WORLD_D = 18 m`. Point percentages (0–100) ↔ world coords via `toWorld()` / `toPercent()`.
- **CityEngine scene groups:** `buildingsGroup`, `assetsGroup`, `plansGroup`, `utilitiesGroup`, `simulationGroup`, `sensorGroup` — separate groups for selective visibility; don't mix content.
- **Props as the CityEngine interface:** State lives in `workspace/page.tsx`; CityEngine only reads props and fires callbacks.
- **Flood map coordinates:** `FloodMaps.origin_m` is in the LiDAR scan's own coordinate system — the flood overlay in CityEngine uses `tilesGroup` local coordinates.
- **Drizzle schema changes:** After editing `db/schema.ts`, always run `npm run db:generate` then `npm run db:migrate:local`.
- **Pyodide packages:** Not all PyPI packages have Pyodide-compatible wheels. Verify before adding Python dependencies.

### Where to Add New Features

| Feature type | Location |
|---|---|
| New simulation (browser) | New worker in `app/`, new Python module in `scripts/` |
| New 3D asset type | `createPrefab()` in `app/CityEngine.tsx` |
| New API endpoint | New `route.ts` in `app/api/<name>/` |
| New database table | `db/schema.ts` → `npm run db:generate` → `npm run db:migrate:local` |
| New live data feed | `lib/live.ts` parser + `app/explore/LivePanel.tsx` display |
| New workspace panel | `app/components/` component + wire into `app/workspace/page.tsx` |
| New TypeScript test | `tests/*.test.mjs` |
| New Python physics test | `scripts/flood/validation.py` |

---

## 15. Feature Inventory Matrix

| Feature | Key Files | Status | Test Coverage |
|---|---|---|---|
| Three.js district scene | `app/CityEngine.tsx` | Implemented, unverified | — |
| Asset placement (GLB/procedural) | `CityEngine.tsx`, `AssetPanel.tsx` | Implemented, unverified | — |
| Infrastructure line drawing | `CityEngine.tsx` | Implemented, unverified | — |
| Planning zone drawing | `CityEngine.tsx` | Implemented, unverified | — |
| Proposed building placement | `CityEngine.tsx` | Implemented, unverified | — |
| Layer controls | `LayerControls.tsx` | Implemented, unverified | — |
| Scenario panel | `ScenarioPanel.tsx` | Implemented, unverified | — |
| Flood solver (Python) | `scripts/flood/solver.py` | ✅ Implemented | 20 validation tests |
| Flood CLI | `scripts/flood/run_simulation.py` | Implemented, unverified | — |
| Flood in browser (Pyodide) | `app/flood-worker.ts` | Implemented, unverified | `flood-drains.test.mjs` |
| SWMM storm-drain coupling | `lib/swmm.ts`, `drainage.py` | ✅ Implemented | `flood-drains.test.mjs` |
| Flood depth overlay (3D tiles) | `CityEngine.tsx` L400–422 | Implemented, unverified | — |
| Sewer capacity scenario | `lib/sewer-network.ts` | Implemented | `sewer-network.test.mjs` |
| Evacuation clearance model | `lib/evacuation.ts` | Implemented | `evacuation.test.mjs` |
| Power grid simulation | `scripts/grid/district_grid.py` | Implemented, unverified | `grid-packages.test.mjs` |
| Place search (geocoding) | `app/api/geocode/route.ts` | Implemented, unverified | — |
| OSM 3D scene | `lib/osm.ts`, `WorldEngine.tsx` | Implemented | `osm.test.mjs` |
| Vector tile base layer | `lib/tiles.ts` | Implemented | `tiles.test.mjs` |
| Live feeds (aircraft/quakes/weather) | `lib/live.ts`, `LivePanel.tsx` | Implemented | `live.test.mjs` |
| Clerk auth + onboarding | `app/page.tsx`, `Onboarding.tsx` | Implemented, unverified | — |
| D1 session persistence | `db/schema.ts`, all `app/api/` | Implemented, unverified | — |
| Session cookie ownership | `app/api/_session-auth.ts` | Implemented, unverified | — |
| Room digital twin 3D viewer | `app/room/page.tsx` | Implemented, unverified | — |
| IoT sensor ingestion API | `app/api/ingest/route.ts` | Implemented, unverified | — |
| Sensor admin provisioning | `app/api/sensors/route.ts` | Implemented, unverified | — |
| ESP8266 firmware | `firmware/room_sensor.ino` | Implemented, unverified | — |
| IITH LiDAR reconstruction | `scripts/reconstruct_iith_ground_models.py` | ✅ Implemented (assets committed) | — |
| Semantic reconstruction studies | `scripts/reconstruct_lidar_models.py` | ✅ Implemented (images committed) | — |
| LiDAR terrain rasterization | `scripts/flood/lidar_terrain.py` | ✅ Implemented | `validation.py` test 11 |
| 3D Tiles streaming | `CityEngine.tsx` L357–396 | Partial (spike/experimental) | — |
| Upload tracking | `app/api/uploads/route.ts` | Partial (metadata only, no storage) | — |
| Elevation data in Explore | — | **Not implemented** | — |
| IFC/BIM import | — | **Planned Phase 2** | — |
| Multi-user collaboration | — | **Planned Phase 3** | — |
| Calibrated physics | — | **Planned Phase 2** | — |

---

## 16. Evidence and Traceability

All claims in this report are supported by direct source code inspection.

| Claim | Evidence |
|---|---|
| Local-inertial physics (Bates 2010) | `scripts/flood/solver.py` — docstring cites the scheme |
| CFL = 0.7 Courant number | `scripts/flood/solver.py` `choose_timestep()` — comment cites Bates et al. |
| CPHEEO sewer design | `lib/sewer-network.ts` module docstring lines 1–6 |
| 135 L/person/day | `lib/sewer-network.ts` line 33 |
| SFPE Handbook walking speed | `lib/evacuation.ts` lines 28–29 |
| 18,449 LiDAR points (17,732 ground) | README.md line 84 |
| Three terrain grades (0.54°, 3.02°, 6.59°) | README.md + `app/workspace/page.tsx` lines 80–82 |
| 20 Python validation tests | `scripts/flood/validation.py` module docstring lines 4–28 |
| Pyodide 314.0.7 | `package.json` |
| Node.js ≥ 22.13 | `package.json` `engines` field |
| Overpass edge cache 12 h | `app/api/osm/route.ts` line 31: `CACHE_SECONDS = 43_200` |
| MAX_RADIUS_M = 1500 | `app/api/osm/route.ts` line 30 |
| Sutherland–Hodgman clipping | `lib/osm.ts` — comment names the algorithm |
| Device secret rotation | `app/api/sensors/route.ts` lines 60–63 |
| `timingSafeEqual` for admin token | `app/api/_session-auth.ts` lines 60–61 |
| `setInsecure()` in firmware | `firmware/room_sensor.ino` line 167 |
| WORLD_W = 24, WORLD_D = 18 | `app/CityEngine.tsx` lines 74–75 |
| In-memory rate limit acknowledged | `app/api/ingest/route.ts` lines 6–8 comment |
| Overpass fails 30% (by authors) | `lib/tiles.ts` lines 4–6 comment |
| No pandapower in Pyodide packages | Pyodide package list (implied absence) |

---

*End of report — MirrorCity, 24 October 2026.*
