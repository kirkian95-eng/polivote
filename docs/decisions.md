# Architecture Decision Log

Each decision records what we chose, why, what we almost chose, and why we didn't.

---

## ADR-001: Map Renderer — MapLibre GL JS

**Chosen:** MapLibre GL JS (BSD 3-Clause)

**Why:** Free, open-source, no API key, no per-load billing, WebGL-powered, same API lineage as Mapbox GL JS. Backed by AWS, Meta, Microsoft. The map is our primary UI — we can't have a variable cost that scales with every user session.

**Runner-up:** Mapbox GL JS

**Why not:** Switched to a proprietary license in Dec 2020. Requires a Mapbox account and access token. Charges $5 per 1,000 map loads above 50k free tier. At 10k DAU, that's ~$2,500+/month just for the map library. Cannot fork, cannot strip telemetry, cannot self-host without their billing infrastructure. Performance is marginally better (~10-15% in benchmarks above 50k features), but not enough to justify the cost and lock-in.

**Also considered:** Leaflet (BSD 2-Clause) — battle-tested, huge plugin ecosystem, but uses DOM-based SVG/Canvas rendering. Cannot handle 100k+ polygons without freezing. Not viable for our use case. OpenLayers (BSD 2-Clause) — powerful for enterprise GIS, but no WebGL polygon rendering, weak React integration, overly complex API for what we need.

---

## ADR-002: Data Visualization Layer — Deck.gl

**Chosen:** Deck.gl (MIT)

**Why:** Purpose-built for rendering massive datasets on maps using WebGL2/WebGPU. Benchmarked at 1M features at 60fps. Our nationwide precinct count (~170k polygons) is well within its comfort zone. Supports efficient partial updates — can change choropleth fill colors without re-uploading geometry to the GPU. Built by the same vis.gl team that maintains the MapLibre React wrapper, so integration is first-class.

**Runner-up:** Raw MapLibre GL JS layers (no overlay library)

**Why not:** MapLibre can render polygons natively, but deck.gl adds heatmaps, scatterplots, arcs, hex bins, and other layer types we'll need for voter density, canvassing progress, and outreach visualization. Using MapLibre alone would mean reimplementing many of these from scratch. Deck.gl is a complement to MapLibre, not a replacement — there's no downside to adding it.

---

## ADR-003: Vector Tile Server — Martin

**Chosen:** Martin (Apache 2.0)

**Why:** Fastest PostGIS vector tile server available (Rust, 2-3x faster than alternatives in benchmarks). Supports dynamic tile generation from PostGIS functions — critical for serving live election results joined to precinct geometry. Also serves static MBTiles and PMTiles. Maintained by the MapLibre organization, so tight ecosystem alignment. Docker images available, simple YAML config.

**Runner-up:** pg_tileserv (Apache 2.0, by Crunchy Data)

**Why not:** Second fastest, and zero-config auto-discovery of PostGIS tables is great for prototyping. But for production with custom SQL queries (joining election results to precincts, simplifying geometry by zoom level), Martin's explicit function sources give more control. pg_tileserv is a solid fallback if we ever have issues with Martin.

**Also considered:** Tegola (MIT, Go) — good built-in tile cache with invalidation, but noticeably slower than Martin. Cache invalidation is less useful when we want real-time freshness on election night.

---

## ADR-004: Basemap Tiles — Protomaps PMTiles on S3

**Chosen:** Protomaps + PMTiles format on S3/CloudFront

**Why:** Open-source planet basemap as a single PMTiles file. Upload to S3, put CloudFront in front, done. No tile server needed for the base layer. No API key, no per-request billing. ~$5-15/month for storage and bandwidth. PMTiles uses HTTP range requests so the client only downloads the tiles it needs.

**Runner-up:** MapTiler Cloud

**Why not:** Free tier has 100k tile requests/month — would be exhausted quickly with active map use. Paid plans start at $25/month and scale with usage. Adds a vendor dependency for something we can trivially self-host. MapTiler is a good option if we ever need premium cartography styles without designing our own.

**Also considered:** Stadia Maps — similar trade-offs to MapTiler. Good styles, but introduces a vendor dependency and usage-based pricing for something that can be a static file on S3.

---

## ADR-005: Cache & Pub/Sub — Valkey

**Chosen:** Valkey (BSD 3-Clause)

**Why:** Drop-in Redis replacement. Same protocol, same commands, same client libraries. Forked from Redis 7.2.4 by the Linux Foundation after Redis changed its license. BSD 3-Clause means zero licensing concerns for any use case — internal, SaaS, managed service, whatever.

**Runner-up:** Redis under AGPLv3 option

**Why not:** Redis 8.0+ is triple-licensed (RSALv2 / SSPLv1 / AGPLv3). Using unmodified Redis internally under AGPLv3 is technically fine for SaaS, but it requires choosing a license, understanding the constraints, and potentially explaining it to lawyers. Valkey is the same code with a simple BSD license. No reason to accept the complexity.

**Also considered:** Dragonfly (BSL 1.1) — faster than Redis/Valkey in some benchmarks, but the Business Source License converts to Apache 2.0 only after 3 years. Not fully open source today. KeyDB — another Redis fork, but less community momentum than Valkey.

---

## ADR-006: Backend Framework — Fastify

**Chosen:** Fastify (MIT)

**Why:** Fastest mainstream Node.js HTTP framework. Built-in JSON schema validation (we need strict validation for voter data and vote submissions). Native WebSocket support for election night real-time. Plugin architecture keeps code modular. TypeScript support is first-class.

**Runner-up:** Express

**Why not:** Slower (Fastify benchmarks 2-3x faster for JSON serialization). No built-in schema validation — would need to add Joi or Zod separately. Express 5 has been in beta for years. Express is fine for simple APIs but Fastify is better suited for a data-heavy application with strict validation requirements.

**Also considered:** Hono (MIT) — lightweight and fast, but smaller ecosystem and less battle-tested for complex applications with WebSockets, streaming, and plugin composition. Good for edge/serverless, less proven for a full SaaS backend.

---

## ADR-007: Database — PostgreSQL + PostGIS

**Chosen:** PostgreSQL (PostgreSQL License) + PostGIS (GPLv2)

**Why:** Best relational database for this use case by a wide margin. ACID transactions for vote integrity. PostGIS for precinct geometry, spatial queries, and `ST_AsMVT` for vector tile generation. Schema-per-tenant multi-tenancy. JSON columns for flexible voter file attributes. Mature, battle-tested, massive ecosystem. Free.

**Runner-up:** None — there was no serious alternative.

**Why not anything else:** MongoDB lacks ACID transactions across documents (critical for vote counting), has no spatial tile generation, and schema-less design is a liability for voter data integrity. MySQL lacks PostGIS-equivalent spatial capabilities. CockroachDB/Citus are overkill for our scale and add operational complexity. SQLite can't handle concurrent writes from multiple services.

---

## ADR-008: Frontend Framework — Next.js

**Chosen:** Next.js (MIT)

**Why:** SSR for public-facing results pages and SEO. API routes for lightweight endpoints. Good PWA support with next-pwa. File-based routing reduces boilerplate. Image optimization built in. Huge ecosystem and community.

**Runner-up:** Vite + React Router

**Why not:** Vite is faster for pure SPA development, but we need SSR for public results pages (shareable election night links, campaign landing pages). Building SSR on top of Vite means adding our own server, which reinvents what Next.js provides. Vite is the fallback if Next.js proves too opinionated for our needs.

**Lock-in note:** Some Next.js features (ISR, edge middleware, image optimization) work best on Vercel. We plan to self-host on AWS (ECS), which is fully supported but means implementing our own image optimization and caching layer. This is acceptable.

---

## ADR-009: Monorepo — Turborepo

**Chosen:** Turborepo (MIT)

**Why:** Single team building multiple packages (frontend, API, shared types, modeling client, PWA). Turborepo handles build orchestration, caching, and dependency graph. MIT licensed, works anywhere (no Vercel lock-in for the CLI). Simple config compared to Nx.

**Runner-up:** Nx

**Why not:** More powerful but significantly more complex. Nx's plugin system and generators are overkill for our project size. Turborepo's simplicity is an advantage early on — we can always migrate to Nx later if the monorepo outgrows Turborepo's capabilities.

**Also considered:** Polyrepo — separate repos for frontend/backend/modeling. Rejected because it adds overhead for a single team: cross-repo PRs, version synchronization, shared type duplication. Monorepo keeps everything in sync.

---

## ADR-010: Modeling Service — Python (separate microservice)

**Chosen:** Python with scikit-learn, pandas, statsmodels (all BSD 3-Clause)

**Why:** Python is the dominant ecosystem for statistical modeling and ML. scikit-learn for voter scoring models, pandas for data manipulation, statsmodels for regression analysis. The modeling service runs as a separate HTTP microservice called from the Node.js API — clean separation of concerns.

**Runner-up:** All-JavaScript with TensorFlow.js or ml.js

**Why not:** The JS ML ecosystem is immature compared to Python. scikit-learn alone has more model types, better documentation, and more community support than all JS ML libraries combined. Election modeling requires well-understood statistical methods (logistic regression, Bayesian updating) that are production-proven in Python. The overhead of a separate service is small compared to the productivity gain.

---

## ADR-011: Multi-Tenancy — Schema-per-tenant

**Chosen:** PostgreSQL schema-per-tenant

**Why:** Each campaign gets its own schema within a shared database. Strong data isolation — a bug in one campaign's query can't leak another campaign's voter data. Easy to export (pg_dump one schema) or delete (DROP SCHEMA CASCADE) a campaign's data. Shared infrastructure keeps operational costs low.

**Runner-up:** Row-level security (RLS) in a shared schema

**Why not:** RLS is fragile. One missing WHERE clause, one raw query that bypasses the policy, and you've leaked voter data between campaigns. For a product handling voter PII and campaign strategy, the cost of a data leak is catastrophic. Schema isolation makes cross-tenant leaks structurally impossible at the database level, not just policy-level.

**Also considered:** Separate databases per tenant — strongest isolation but operationally expensive. Each tenant needs its own connection pool, backup schedule, migration run. Doesn't scale for a SaaS with potentially hundreds of campaigns.

---

## ADR-012: Job Queue — BullMQ

**Chosen:** BullMQ (MIT)

**Why:** Mature, well-documented job queue for Node.js. Supports delayed jobs, retries, rate limiting, job flows (parent/child dependencies), and priority queues — all needed for voter file ingestion pipelines, batch outreach, and scoring runs. Uses Valkey/Redis as backend (aligns with our existing Valkey choice).

**Runner-up:** pg-boss (MIT)

**Why not:** pg-boss uses PostgreSQL as its backend (no Valkey dependency), which is appealing for simplicity. But it's significantly slower for high-throughput workloads. Voter file ingestion may process millions of records — BullMQ's Valkey-backed performance is better suited. pg-boss is a good fallback if we decide to eliminate Valkey from the stack entirely.

**Also considered:** Temporal (MIT core) — powerful workflow engine, but heavy for our needs. Overkill when BullMQ's job flows cover our use cases. Would add significant operational complexity.

---

## ADR-013: Voter File Ingestion — Schema-on-Read with Mapping Profiles

**Chosen:** JSON mapping profiles per state/vendor format with auto-detection

**Why:** Every state has a different voter file format — different column names, different encodings, different delimiters. Writing custom parsers per state doesn't scale. Instead, mapping profiles define how source columns map to our canonical schema. Adding a new state = adding a config file, not a code change. Auto-detection heuristics match uploaded files to known profiles based on headers and sample rows.

**Runner-up:** Fixed ETL pipelines per state

**Why not:** Requires code changes for every new state. With 50 states plus various commercial voter file vendors (L2, TargetSmart, Aristotle), that's potentially 60+ custom parsers to maintain. The mapping profile approach handles all of them with configuration.

---

## ADR-014: Offline Field App — PWA with Workbox

**Chosen:** Progressive Web App with Workbox service workers

**Why:** Canvassers work in areas with poor cell service. PWA lets us pre-download walk lists, scripts, and map tiles (PMTiles) to IndexedDB. Responses are queued offline and synced when connectivity returns. No app store approval needed — deploy updates instantly. Single codebase serves both desktop dashboard and mobile field app.

**Runner-up:** React Native mobile app

**Why not:** Requires separate codebase (or at least separate build targets), app store submission and approval process, and native build infrastructure. For our use case — filling out forms, viewing a map, recording door responses — a well-built PWA provides equivalent functionality without the overhead. If we later need native features (push notifications on iOS, background GPS tracking), we can add a thin React Native shell around the PWA.
