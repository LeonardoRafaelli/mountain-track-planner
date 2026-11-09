# Mountain Route & Conditions Planner: Development Roadmap

A web app that computes hiking / ski-touring / approach routes ranked by estimated time **and** terrain/conditions risk, using a custom multi-objective cost function.

**Stack:** Java 25 (LTS) + Spring Boot 4 backend, PostgreSQL/PostGIS, React + TypeScript + MapLibre GL frontend.
**Estimated duration:** ~10 weeks part-time (8-10 h/week). Phases 0-3 give you a complete, showable MVP.

---

## 1. Project Goal & Scope

- **Pick ONE activity** to start (hiking is easiest; ski touring has the richest risk model).
- **Pick ONE pilot region** where you actually go outside, so you can validate against your own GPS tracks.
- **Educational tool only.** State this prominently in the README and UI. Never present output as safety advice.

**Success criteria (numbers you can put on your CV):**
- Route query latency under ~1-2 s for typical routes (measured with JMH / k6)
- Time-estimate error vs. your real GPX tracks (e.g., "median error 12%")
- Risk layer visibly matches known avalanche terrain in your region

---

## 2. Technology Stack

| Layer | Choice | Why |
|---|---|---|
| Language / runtime | **Java 25 (LTS)** | Your strongest language; LTS means long support and best library compatibility. JDK 27 exists but is not LTS; optionally add a CI job that also builds on 27 |
| Backend framework | **Spring Boot 4.x** | REST API, dependency injection, scheduled jobs. Check that guides target Boot 4, not 3 (Jackson 3, modular starters) |
| Build | **Maven or Gradle** | Whichever you know better |
| Spatial DB | **PostgreSQL + PostGIS** | Industry standard for geodata; spatial indexes and queries |
| DB access | **jOOQ or Spring JDBC** (or Hibernate Spatial) | Raw spatial SQL is easier with jOOQ/JDBC than with a full ORM |
| Migrations | **Flyway** | Reproducible schema |
| Geometry | **JTS** | Lines, distances, spatial operations |
| Raster / DEM | **GeoTools** (GeoTIFF reading) + optional `gdaldem` CLI | See Phase 4 |
| OSM import | **osm2pgsql** (CLI tool) | Don't parse OSM in Java yourself |
| Routing engine | **Your own A\*** on primitive-array graphs | The core of the project; JGraphT is fine for prototyping only |
| HTTP / JSON | Spring **WebClient** + Jackson | Open-Meteo, avalanche bulletin (CAAML/JSON) |
| Caching | **Caffeine** (in-process) | Cache forecasts and bulletins |
| GPX | **jpx** library | Validation against your own tracks |
| Frontend | **React + TypeScript + MapLibre GL JS** | Your strength; MapLibre is open source with no API key |
| Charts | **Recharts or D3** | Elevation profile |
| Testing | **JUnit 5, Testcontainers, JMH** | Unit + integration tests; JMH makes latency numbers credible |
| Profiling | **Java Flight Recorder (JFR)** | Find real bottlenecks before optimizing |
| Infra | **Docker Compose, GitHub Actions** | Reproducible setup + CI |
| Hosting | Hetzner / Fly.io / Render | Cheap, simple |
| Optional | Python notebooks | Exploratory analysis and plots only; not on the critical path |

**Rule of thumb:** use what you know on the critical path, and learn only one new thing per phase (PostGIS in Phase 1, MapLibre in Phase 3, raster processing in Phase 4).

---

## 3. Roadmap

### Phase 0: Setup & Scoping (Week 1, first days)

**Goal:** A clear definition of what you're building and a working repo.

**How:**
1. Choose the activity and region; write a one-page spec (inputs, outputs, non-goals).
2. Create the repo layout: `/backend`, `/frontend`, `/data`, `/docs`, `/notebooks`.
3. Set up Docker Compose with Postgres + PostGIS; bootstrap Spring Boot 4 on JDK 25.
4. Set up CI (build + tests) from day one.
5. Record 3-5 of your own GPX tracks in the pilot region. You'll need them for validation in Phase 8.

**Deliverable:** Repo with README skeleton, running DB container, green CI.

---

### Phase 1: Data Foundation (Weeks 1-2)

**Goal:** Trails and terrain data for your region, cleaned and queryable.

**How:**
1. Download an OSM extract for your region (Geofabrik) or query Overpass.
2. Filter to relevant ways: `highway=path|track|footway|steps`, plus `sac_scale`, `surface`, `trail_visibility` tags.
3. Import into PostGIS with `osm2pgsql`; create spatial indexes; manage schema with Flyway.
4. Download a Digital Elevation Model: Copernicus GLO-30 as a baseline; check whether your region publishes a higher-resolution open LiDAR DEM (much better slope accuracy).
5. Sample elevation onto trail nodes (GeoTools); compute per-edge length, ascent, descent.
6. Write data-quality checks (disconnected components, missing elevations, suspicious gradients) as tests.

**Deliverable:** Scripts that rebuild the database from scratch with one command. Document data sources and licenses.

---

### Phase 2: Routing Engine v1 (Weeks 2-3)

**Goal:** Fastest route between two points over the trail graph.

**How:**
1. Load edges from PostGIS into an in-memory graph using flat arrays (node IDs, edge targets, cost arrays). This is fast and a great benchmarking story.
2. Implement a **time model**: Tobler's hiking function (or Naismith's rule as a baseline), direction-aware (uphill is not the same as downhill).
3. Snap arbitrary coordinates to the nearest graph node (PostGIS KNN query).
4. Implement A* yourself, with a heuristic based on straight-line distance and maximum speed.
5. Unit-test against hand-computed small graphs; add JMH benchmarks.
6. Optional: compare results against GraphHopper (an open-source Java router) as a sanity check, not as a dependency.

**Deliverable:** `route(start, end) -> path, time, distance, ascent` with tests and a first benchmark.

---

### Phase 3: API + Frontend MVP (Weeks 3-4)

**Goal:** Usable end-to-end app (your first demo-able milestone).

**How:**
1. Spring Boot endpoints: `POST /route`, `GET /elevation-profile`, `GET /health`; OpenAPI docs (check springdoc compatibility with Boot 4).
2. Frontend (React + TS): MapLibre map, click to set start/end, route drawn as GeoJSON.
3. Elevation profile chart linked to the map (hover on chart shows marker on map).
4. Show distance, ascent, estimated time.
5. Build mobile-first and responsive from the start.
6. Dockerize everything; deploy a first live version.

**Deliverable:** Live URL + demo GIF. **Stop here and commit the MVP before moving on.**

---

### Phase 4: Terrain Analysis (Weeks 5-6)

**Goal:** Know how steep and which direction every piece of terrain faces.

**How:**
1. Compute **slope** and **aspect** from the DEM. Two good options: implement Horn's method (3x3 kernel) yourself in Java, which is simple and makes a good write-up, and cross-check results against `gdaldem`. Or precompute with `gdaldem` and read the GeoTIFFs with GeoTools.
2. Classify cells: flat / moderate / steep / avalanche-prone (roughly 30-45 degree start-zone angles).
3. Optionally derive terrain classes in the spirit of ATES (simple, challenging, complex); document it as a simplified model.
4. Store as tiled rasters / Cloud-Optimized GeoTIFFs; sample along routes.
5. Visualize as a map overlay (slope-angle shading is a familiar format for mountain users).

**Deliverable:** A hazard overlay on the map and per-route statistics (e.g., "320 m exposed to 30-40 degree slopes").

---

### Phase 5: Risk-Aware Routing (Weeks 6-7)

**Goal:** Routes that trade off speed against safety.

**How:**
1. Define edge cost: `cost = time + lambda * risk`.
2. Edge risk comes from slope exposure, aspect, elevation, and terrain above/below the trail (runout, overhead hazard).
3. Add a UI slider for lambda; show the route changing live.
4. Return 2-3 alternatives (fastest, safest, balanced) as a Pareto set.
5. Write property-style tests: higher lambda never increases total risk.

**Deliverable:** Slider + alternatives view. This is the centerpiece of the project.

---

### Phase 6: Live Conditions (Weeks 7-8)

**Goal:** Routes that react to today's weather and avalanche situation.

**How:**
1. **Weather:** Pull Open-Meteo forecasts (temperature, wind, precipitation, freezing level) for route sample points via WebClient; cache with Caffeine.
2. **Avalanche bulletin:** Fetch your region's EAWS-style bulletin (many publish CAAML/JSON); parse with Jackson into danger level, elevation bands, and problem aspects. Check each source's license.
3. Modulate the risk model: danger level x slope/aspect/elevation match from the bulletin.
4. Time-of-day logic: sunset, afternoon thunderstorm risk, snow warming (spring touring).
5. Scheduled refresh jobs (`@Scheduled`); store fetch timestamps. **Always show data age in the UI.**

**Deliverable:** Conditions panel + risk that changes with the day's bulletin.

---

### Phase 7: AI Layer (Weeks 8-9)

**Goal:** Add useful AI features **without** letting AI make safety decisions. See Section 4.

**How:** Implement 1-2 of the features in Section 4 (recommended: the go/no-go explainer and the natural-language query).

**Deliverable:** AI features with documented guardrails and a small evaluation set.

---

### Phase 8: Validation, Performance, Hardening (Weeks 9-10)

**Goal:** Prove it works with numbers.

**How:**
1. **Validate time model:** Compare predictions vs. your real GPX tracks; report mean/median error; calibrate speed parameters.
2. **Validate risk layer:** Compare flagged terrain against your own knowledge of the region.
3. **Benchmark:** JMH for the routing core; k6 for the API (p50/p95 latency, memory, graph size).
4. **Profile and optimize:** JFR first, then fix what it shows (graph layout, spatial index, caching; contraction hierarchies if needed).
5. **Harden:** Input validation, rate limiting, error handling, secrets management.
6. Add disclaimers and data attribution in the UI footer.

**Deliverable:** `docs/validation.md` with tables and charts.

---

### Phase 9: Polish & Presentation (Week 10)

**Goal:** A repo and demo that reads as engineering, not a tutorial.

**Checklist:**
- [ ] README: pitch, demo GIF, architecture diagram, quick-start (`docker compose up`)
- [ ] 2-3 headline numbers (latency, time-estimate error)
- [ ] Write-up: the risk model, trade-offs, what didn't work
- [ ] Tests + CI badge
- [ ] Live demo link
- [ ] Clear disclaimer and data attributions

---

### Mobile (decision: web first)

- Build the frontend responsive and mobile-first from Phase 3.
- After the core works, make it a **PWA** (home-screen install, offline cache of the last planned route and map tiles).
- Optional **Phase 10:** a React Native (Expo) client with MapLibre Native, reusing the same API, if you want GPS track recording and real mobile experience on your CV.

---

### Stretch Goals (after the core is solid)

- Off-trail routing on the DEM grid (huge scope; time-box it)
- GPX import/export and "compare my real track vs. prediction"
- Multi-day itinerary planning
- Learned time model from your own GPX data (see AI section)
- Contraction hierarchies for fast long-distance queries

---

## 4. Where AI Comes In

Two separate questions: **using AI to build it**, and **AI as a feature in the product**.

### A. AI as a development tool (throughout)

| Use | Phase | Notes |
|---|---|---|
| Boilerplate (Spring config, Docker, CI) | 0, 3 | Big time saver; review everything |
| Explaining unfamiliar geospatial APIs (PostGIS, GeoTools, GDAL) | 1, 4 | Verify against docs; coordinate-system mistakes are common |
| Generating test cases and edge cases for the cost function | 2, 5 | Ask for adversarial cases (dead ends, disconnected graphs) |
| Code review and refactoring | All | Paste modules and ask for critique |
| Drafting docs / README | 9 | You still write the "why" yourself |

**Rule:** write the core algorithms (A*, cost function, risk model) yourself. That's what interviewers will ask about.

### B. AI as a product feature

Principle: **the deterministic model decides, the AI explains.**

1. **Go/no-go day summary (recommended):** Feed an LLM the *structured* output (route stats, risk breakdown, bulletin fields, weather) and have it write a plain-language briefing. It must only restate supplied data, never invent.
2. **Natural-language trip query (recommended):** "Moderate half-day ski tour, south-facing, under 1000 m ascent" is converted to structured parameters (JSON schema), and your engine does the actual search.
3. **Bulletin parsing/translation:** If a bulletin is only free text or in another language, an LLM can extract structured fields and translate. Validate against a schema and show the source text.
4. **Learned time model (classic ML):** Train a small regression model on your GPX tracks plus slope/surface features (Smile library in Java, or a Python notebook) and compare honestly against Tobler's function.
5. **Terrain classification (stretch):** Train a classifier on labeled terrain (ATES-style classes) from DEM features.

**Integration:** Call the Claude API over HTTP (WebClient) or via the official Java SDK; request JSON-schema-constrained output for features 2 and 3.

### Guardrails (include these in your write-up)

- The LLM **never** produces a risk score or a "safe" verdict by itself.
- Every AI statement is traceable to structured inputs; show the inputs next to the text.
- Build a small evaluation set (20-30 cases) and check summaries for hallucinated numbers.
- Display "AI-generated summary of model output; not safety advice."
- Handle API failure gracefully: the app must still work without the LLM.

---

## 5. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Safety liability | Educational disclaimer, no "safe route" language, show data age |
| Scope creep (off-trail routing) | Finish trail-based end to end first; time-box stretch work |
| Data licensing / rate limits | Check licenses, cache, attribute sources |
| Raster processing is clunkier in Java | Precompute with `gdaldem` or tile once and cache; keep routing/risk logic in Java |
| Coordinate system bugs | Pick one projected (metric) CRS for computation; convert only at the edges |
| Library lag behind new Spring/JDK versions | Stay on JDK 25 LTS; check compatibility before upgrading |
| Stale conditions data | Timestamps everywhere; fail visibly, not silently |

---

## 6. Suggested Timeline Summary

| Weeks | Phase | Milestone |
|---|---|---|
| 1 | 0 | Repo, DB, CI |
| 1-2 | 1 | Data pipeline |
| 2-3 | 2 | Routing engine v1 |
| 3-4 | 3 | **MVP live** |
| 5-6 | 4 | Slope/aspect layer |
| 6-7 | 5 | Risk-aware routing + slider |
| 7-8 | 6 | Weather + bulletin |
| 8-9 | 7 | AI features |
| 9-10 | 8-9 | Validation, perf, polish |