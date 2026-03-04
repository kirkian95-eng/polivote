# Polivote Architecture

## Constraints Driving the Design
- **SaaS, single-campaign tenants** — every campaign gets isolated data, shared infrastructure
- **Any election level** — data model can't assume a fixed jurisdiction hierarchy
- **Jurisdiction-agnostic** — must ingest wildly different voter file formats without custom code per state
- **Fully automated** — minimal manual data entry; pipelines handle ingestion, scoring, projections
- **Field workers on mobile web** — PWA that works offline in areas with poor connectivity
- **Map-centric UI** — primary interface is a map with heavy data overlays (100k+ precinct polygons)
- **All commercial-friendly licenses** — no proprietary dependencies, no per-usage fees from vendors

---

## Licensing Audit

Every dependency has been verified as free for commercial SaaS use:

| Component | License | Notes |
|-----------|---------|-------|
| React, Next.js, Fastify, Tailwind, Recharts, Visx, BullMQ, Workbox, Turborepo | MIT | No restrictions |
| TypeScript, Apache Arrow/Parquet, Docker Engine, Martin | Apache 2.0 | No restrictions |
| MapLibre GL JS, Deck.gl, scikit-learn, pandas, statsmodels, Valkey | BSD 3-Clause | No restrictions |
| PostgreSQL | PostgreSQL License (BSD-style) | No restrictions |
| PostGIS | GPLv2 | Safe for SaaS — copyleft only triggers on binary distribution, not server-side use |
| Node.js, Python | MIT / PSF | No restrictions |

**Rejected on licensing grounds:**
- ~~Mapbox GL JS~~ — proprietary, $5/1k loads above 50k free tier, mandatory API key, cannot fork
- ~~Redis~~ — changed to RSALv2/SSPLv1/AGPLv3 in 2024. Usable but complex. Valkey (BSD fork) is simpler.

---

## High-Level Architecture

```
┌─────────────┐  ┌─────────────┐  ┌─────────────────┐
│  Campaign    │  │  Field App  │  │  War Room        │
│  Dashboard   │  │  (PWA)      │  │  Dashboard       │
│  (React SPA) │  │  (React)    │  │  (React + WS)    │
└──────┬───────┘  └──────┬───────┘  └───────┬──────────┘
       │                 │                  │
       └────────────┬────┴──────────────────┘
                    │
              ┌─────▼──────┐
              │   API       │
              │   Gateway   │
              │  (REST +    │
              │   WebSocket)│
              └─────┬───────┘
                    │
       ┌────────────┼────────────┐
       │            │            │
┌──────▼───┐ ┌─────▼────┐ ┌────▼─────┐
│ Core API │ │ Modeling  │ │ Ingest   │
│ Service  │ │ Service   │ │ Pipeline │
└──────┬───┘ └─────┬────┘ └────┬─────┘
       │           │            │
       └───────────┼────────────┘
                   │
       ┌───────────┼───────────┐
       │                       │
┌──────▼──────┐         ┌─────▼──────┐
│  PostgreSQL  │         │   Valkey    │
│  + PostGIS   │         │  (cache +   │
│  (per-tenant │         │   pub/sub)  │
│   schema)    │         └────────────┘
└──────┬───────┘
       │
┌──────▼──────┐
│   Martin     │
│  (vector     │
│   tile       │
│   server)    │
└─────────────┘
```

---

## Tech Stack

### Frontend
| Choice | License | Rationale |
|--------|---------|-----------|
| **React + TypeScript** | MIT / Apache 2.0 | Dominant ecosystem, massive hiring pool, mature tooling |
| **Next.js** | MIT | SSR for public pages, API routes, good PWA support |
| **Tailwind CSS** | MIT | Rapid UI development, consistent design system |
| **Recharts + Visx** | MIT | Data visualization for dashboards, charts, election night needle |
| **MapLibre GL JS** | BSD 3-Clause | WebGL map renderer — free, no API key, no per-load billing |
| **Deck.gl** | MIT | GPU-accelerated data layers — handles 1M+ features at 60fps |
| **@vis.gl/react-maplibre** | MIT | First-class React wrapper for MapLibre (by the same vis.gl team as deck.gl) |
| **Workbox** | MIT | PWA offline support — cache walk lists, queue responses |

### Backend
| Choice | License | Rationale |
|--------|---------|-----------|
| **Node.js + TypeScript** | MIT | Shared language with frontend, strong async I/O for real-time |
| **Fastify** | MIT | Faster than Express, schema validation built in, WebSocket support |
| **PostgreSQL + PostGIS** | PostgreSQL / GPLv2 | ACID for vote data, PostGIS for precinct geometry and spatial queries |
| **Valkey** | BSD 3-Clause | Drop-in Redis replacement. Cache + pub/sub. No license complexity. |
| **BullMQ** | MIT | Job queues for voter file ingestion, scoring pipelines, batch outreach |
| **Martin** | Apache 2.0 | Fastest PostGIS vector tile server (Rust). Maintained by MapLibre org. |

### Data & Modeling
| Choice | License | Rationale |
|--------|---------|-----------|
| **Python** | PSF | Best ecosystem for stats/ML — scikit-learn, pandas, statsmodels |
| **Fastify ↔ Python via HTTP** | — | Keep modeling as a separate microservice, call from Node |
| **Apache Arrow / Parquet** | Apache 2.0 | Efficient columnar storage for large voter files (millions of rows) |

### Infrastructure
| Choice | License | Rationale |
|--------|---------|-----------|
| **Docker + Docker Compose** (dev) | Apache 2.0 | Reproducible local environment |
| **AWS** (prod) | — | ECS for services, RDS for Postgres, S3 for tiles + uploads |
| **S3 + CloudFront** | — | Serve PMTiles basemap + voter file storage + exports |
| **GitHub Actions** | — | CI/CD pipeline |
| **Turborepo** | MIT | Monorepo build orchestration |

### Auth & Multi-Tenancy
| Choice | License | Rationale |
|--------|---------|-----------|
| **Schema-per-tenant in PostgreSQL** | — | Strong data isolation without operational overhead of separate DBs |
| **JWT + refresh tokens** | — | Stateless auth, works well with SaaS multi-tenant |
| **Role-based access control** | — | Campaign manager, field director, canvasser, viewer roles |

---

## Map Stack (Detail)

The map is the primary UI. It must handle 170,000 precincts nationwide at 60fps with real-time color updates on election night.

### Architecture

```
┌───────────────────────────────────────────────────────┐
│  React Frontend                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │  @vis.gl/react-maplibre (basemap + navigation)  │   │
│  │  + Deck.gl overlay layers:                      │   │
│  │    - MVTLayer (precinct choropleth from Martin)  │   │
│  │    - HeatmapLayer (voter density)                │   │
│  │    - ScatterplotLayer (canvassing pins)          │   │
│  │    - GeoJsonLayer (turf assignments)             │   │
│  └──────────────┬──────────────────┬───────────────┘   │
└─────────────────┼──────────────────┼───────────────────┘
                  │                  │
      ┌───────────▼────────┐  ┌─────▼──────────────┐
      │  PMTiles on S3/CDN │  │  Martin tile server │
      │  (basemap tiles    │  │  (dynamic data from │
      │   via Protomaps)   │  │   PostGIS)          │
      └────────────────────┘  └─────┬──────────────┘
                                    │
                              ┌─────▼──────┐
                              │  PostGIS    │
                              │  precincts  │
                              │  + results  │
                              └────────────┘
```

### Why This Stack

| Criterion | Detail |
|-----------|--------|
| **Cost** | $0 licensing. Infrastructure only: ~$35-105/month (PostGIS + Martin + S3/CDN) |
| **Performance** | Deck.gl renders 1M features at 60fps. 170k precincts is comfortable. |
| **Real-time** | Update `election_results` table → Martin serves fresh tiles → WebSocket triggers client re-fetch |
| **Offline** | PMTiles can be pre-downloaded to IndexedDB for field workers. MapLibre's `addProtocol` supports custom tile loading from local storage. |
| **No vendor lock-in** | Every component is open source and replaceable |
| **Proven** | NYT, Washington Post, and VoteHub use similar stacks for election visualization |

### Basemap Tiles

**Protomaps** — open-source planet basemap as a single PMTiles file. Host on S3 + CloudFront. No API key, no per-request billing. ~$5-15/month for storage + bandwidth.

### Tile Serving Strategy

- **Static geometry** (precinct boundaries): Pre-generate PMTiles with Tippecanoe, serve from S3
- **Dynamic data** (election results, scores): Martin serves vector tiles with live PostGIS joins
- **Low zoom levels** (z0-z8): Pre-cache as MBTiles for fast rendering at national scale
- **High zoom levels** (z9+): Dynamic from Martin with PostGIS spatial index

### Martin Function for Live Election Results

```sql
CREATE OR REPLACE FUNCTION precinct_results(z integer, x integer, y integer, race_id integer DEFAULT 1)
RETURNS bytea AS $$
    SELECT ST_AsMVT(tile, 'precincts', 4096, 'geom') FROM (
        SELECT
            p.precinct_id,
            p.precinct_name,
            p.state_fips,
            er.votes_dem,
            er.votes_rep,
            er.total_votes,
            er.pct_reporting,
            CASE WHEN er.total_votes > 0
                THEN ROUND(er.votes_dem::numeric / er.total_votes * 100, 1)
                ELSE NULL END AS dem_pct,
            ST_AsMVTGeom(
                p.geom,
                ST_TileEnvelope(z, x, y),
                4096, 64, true
            ) AS geom
        FROM precincts p
        LEFT JOIN election_results er
            ON p.precinct_id = er.precinct_id
            AND er.race_id = precinct_results.race_id
        WHERE p.geom && ST_TileEnvelope(z, x, y)
    ) AS tile
$$ LANGUAGE sql IMMUTABLE PARALLEL SAFE;
```

### Precinct Boundary Data Sources

| Source | Coverage | Cost | License |
|--------|----------|------|---------|
| **NYT Presidential Precinct Map 2024** | Nationwide 2024 | Free | Open source |
| **VEST (Harvard Dataverse)** | All 50 states, 2016-2024 | Free (2016-2020), paid (2022+) | CC BY-NC-ND 4.0 |
| **Redistricting Data Hub** | All 50 states | Free (most datasets) | Varies |
| **Census TIGER/Line VTDs** | All 50 states | Free | Public domain |

~170,000 precincts nationwide. No single authoritative source exists — boundaries are maintained at the county level.

### PostGIS Performance Tips

- Minimize properties per tile — going from 5 to 42 columns makes `ST_AsMVT` 9x slower
- Use PostGIS 3.1+ — wagyu clipping is 20x faster for complex polygons
- Simplify geometry at lower zoom levels with `ST_Simplify`
- GIST index on geometry column is critical
- Connection pooling (PgBouncer) between Martin and PostGIS

---

## Multi-Tenancy Design

Each campaign = one tenant = one PostgreSQL schema.

```
database: polivote
├── public          (shared: users, tenants, billing, plans)
├── campaign_abc    (tenant: voters, donors, contacts, precincts, results)
├── campaign_xyz    (tenant: voters, donors, contacts, precincts, results)
```

**Why schema-per-tenant:**
- Row-level security is fragile at scale and easy to mess up with voter data
- Separate databases are operationally expensive for a SaaS
- Schema-per-tenant gives real isolation with shared infrastructure
- Easy to export/delete a campaign's data (drop schema)

---

## Voter File Ingestion (Jurisdiction-Agnostic)

This is the hardest problem. Every state has a different voter file format.

**Approach: Schema-on-read with mapping profiles**

```
Raw File → Upload to S3 → Detect Format → Apply Mapping Profile → Normalize → Load
```

1. **Mapping profiles** — JSON configs that map source columns to canonical schema. Community-contributed, one per state/vendor format. Stored in a `mappings/` directory.
2. **Auto-detection** — heuristics to guess the format based on headers and sample rows
3. **Canonical voter schema** — standardized internal model regardless of source
4. **Dedup pipeline** — probabilistic matching on name + DOB + address to merge records across sources

This means adding a new state is a config file, not a code change.

---

## Modeling Architecture

**Scoring pipeline (batch, runs on import + nightly):**
```
Voter records → Feature extraction → Model inference → Scores written to DB
```

- Support score: logistic regression on vote history + demographics + contact responses
- Turnout score: historical turnout patterns + registration recency + age
- Persuadability: distance from decision boundary in support model
- Donor score: income proxies + past giving + engagement signals

**Election night projection (real-time):**
```
Reported precincts → Update Bayesian model → Project unreported → Confidence intervals
```

- As each precinct reports, update priors
- Weight by precinct similarity to unreported ones
- Output: projected final vote share + win probability + confidence band

---

## Real-Time Election Night

- **WebSocket connections** from war room dashboard to API gateway
- **Valkey pub/sub** — when a result is entered/ingested, publish to channel
- **Projection service** recalculates on every new precinct result, publishes updated model
- **Martin** serves fresh vector tiles with updated results on each request
- **Deck.gl** re-renders precinct colors without re-uploading geometry to GPU
- **Dashboard** renders live: map coloring, needle/probability, outstanding vote table

---

## Offline-First Field App (PWA)

Canvassers often work in areas with poor cell service.

- **Pre-sync:** Download assigned walk list + scripts + PMTiles map for target area
- **Offline operation:** All door responses stored in IndexedDB
- **Background sync:** Queue uploads, sync when connectivity returns
- **Conflict resolution:** Last-write-wins for contact responses (simple, good enough)
- **Map tiles:** MapLibre's `addProtocol` loads tiles from IndexedDB when offline

---

## Key Data Models (Conceptual)

**Voter**
- voter_id, state_voter_id, first_name, last_name, dob, gender
- address (structured), precinct_id, district_ids[]
- party_registration, registration_date, status
- vote_history[] (election, method, voted_y/n)
- scores: support, turnout, persuadability, donor

**Donor**
- donor_id, voter_id (nullable — donors may not be local voters)
- contributions[] (date, amount, method, designation)
- total_given, last_gift_date, donor_score

**Precinct**
- precinct_id, name, jurisdiction, geometry (PostGIS)
- historical_results[] (election, candidates, votes, turnout)
- demographic_profile, registration_counts

**Contact**
- contact_id, voter_id, canvasser_id, timestamp
- channel (door, phone, text, email, mail)
- result (support_level, issues[], volunteer_interest, donation_response)
- script_id, notes

**Election Night Result**
- precinct_id, election_id, batch_type (early, election_day, mail, provisional)
- candidate_votes{}, total_votes, pct_reporting
- entered_by, entered_at, source

---

## Estimated Infrastructure Cost (Production)

| Component | Monthly Cost |
|-----------|-------------|
| PostgreSQL + PostGIS (RDS db.t3.medium) | $30-70 |
| Martin tile server (ECS or same instance) | $0-20 |
| S3 + CloudFront (PMTiles basemap + uploads) | $5-15 |
| Valkey (ElastiCache t3.micro) | $15-25 |
| ECS (API + workers) | $30-60 |
| **Total** | **$80-190/month** |

All software licensing: $0.

---

## Remaining Decisions

- **Email/SMS provider?** SendGrid + Twilio? Resend + something cheaper?
- **Payment/billing?** Stripe for SaaS subscriptions.
- **Precinct data pipeline** — automate download + load of TIGER/Line + NYT boundaries?
