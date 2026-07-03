# Political Analytics — Interactive U.S. Election Explorer

An interactive, full-stack explorer for historical U.S. presidential election results — a choropleth map, a year-by-year timeline, state and candidate profiles, year-over-year comparison, and a statistics dashboard, inspired by election-night broadcast maps and built as a production-style full-stack application.

**Live interactive demo:** `ElectionExplorer.jsx` (see [`/demo`](./demo)) — self-contained React artifact, runs with no backend required.
**Production architecture:** `/backend` (FastAPI + PostGIS) and `/frontend` (React + TypeScript + Mapbox GL) — a real, deployable full-stack scaffold.

---

## Features

- **Choropleth map** of all 50 states + DC, colored by winning party and margin, rendered from real Albers-projected state boundary geometry (not a schematic grid)
- **Timeline scrubber** across 17 presidential elections (1960–2024), with play/pause auto-advance
- **Search** — jump straight to any state or any candidate
- **State profiles** — full voting history, largest D/R win, closest race, longest party streak, biggest swing, first/most recent election on record
- **Candidate profiles** — elections contested, states/EVs won per cycle, best/worst-performing states
- **Comparison mode** — pick any two election years and see which states flipped party, with margin deltas
- **Statistics dashboard** — electoral vote trend over time, party distribution, closest races, biggest landslides

## Tech stack

| Layer | Stack |
|---|---|
| Frontend | React, TypeScript, Mapbox GL JS / `react-map-gl`, Tailwind CSS, Recharts |
| Backend | Python, FastAPI, SQLAlchemy |
| Database | PostgreSQL + PostGIS |
| Data ingestion | Python, GeoPandas, Shapely |
| Deployment | Docker, Docker Compose |

## Architecture

```
frontend (React + TS + Mapbox GL + Tailwind + Recharts)
        │  REST (JSON / GeoJSON)
        ▼
backend (FastAPI)
   ├── GET /api/elections/{year}/counties.geojson   choropleth per year
   ├── GET /api/counties/search, /api/counties/{fips}
   ├── GET /api/candidates/search, /api/candidates/{id}
   ├── GET /api/compare?year_a=&year_b=             flipped counties
   └── GET /api/stats/{year}                        dashboard aggregates
        │  SQL (SQLAlchemy)
        ▼
PostgreSQL + PostGIS
   states, counties (geom), elections, candidates, county_results,
   county_winners (materialized view)
```

## Data sources

- **Election results:** [MIT Election Data and Science Lab (MEDSL)](https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/VOQCHQ) county-level presidential returns, keyed by FIPS county code.
- **Geography:** [U.S. Census TIGER/Line](https://www2.census.gov/geo/tiger/) county and state boundary shapefiles.
- **Demo map geometry:** simplified state-boundary paths derived from Wikimedia Commons' public-domain U.S. states map (same source used by several popular open-source React US-map libraries).

> **Note on the bundled demo:** `ElectionExplorer.jsx` ships with a compact, hand-compiled dataset (state winners are accurate to the historical record; exact vote percentages are illustrative approximations, not sourced directly from MEDSL) so the interaction design is fully explorable without a live backend. See `/backend/scripts/ingest.py` for the pipeline that loads real MEDSL + TIGER data into Postgres for production use.

## Getting started

### Just want to see it work?
Open `demo/ElectionExplorer.jsx` in any React sandbox (or drop it into a Vite/CRA app) — no setup, no backend, no API keys.

### Running the full stack

```bash
git clone https://github.com/nikhilchandrashaker/Political-Analytics.git
cd Political-Analytics

# 1. Get real data (see backend/scripts/ingest.py docstring for exact URLs)
#    place under ./data/

# 2. Bring up Postgres + Redis + API + frontend
docker compose up --build

# 3. Load real election + geography data into Postgres
docker compose exec backend python scripts/ingest.py \
  --medsl ./data/countypres_2000-2020.csv \
  --tiger ./data/tl_2023_us_county.zip
```

- Frontend: http://localhost:5173
- API docs (FastAPI auto-generated): http://localhost:8000/docs

Set a real `VITE_MAPBOX_TOKEN` in `frontend/.env` (copy from `.env.example`) — free at [mapbox.com](https://mapbox.com).

## Performance notes

- County GeoJSON is simplified server-side (`ST_SimplifyPreserveTopology`); for smoother pan/zoom at scale, pre-generate vector tiles with `tippecanoe` per election year and serve from a CDN instead of raw GeoJSON.
- `county_winners` is a materialized view refreshed after ingestion, so hot-path map/search/stats queries never recompute "who won this county" from raw vote rows.
- County name search uses Postgres `pg_trgm` for fuzzy matching.

## Roadmap

- [ ] Animated party-realignment view (scrub the timeline, watch counties flip in place)
- [ ] County similarity search (nearest neighbors by historical vote-share vector, via `pgvector`)
- [ ] Turnout data (registered voters / voting-age population per county)
- [ ] Shareable URLs for a given map view, search, or comparison
- [ ] Export charts and maps as PNG/PDF
- [ ] Vector-tile pipeline for full county-level rendering at all zoom levels

## License

MIT. Election result geometry and figures are subject to their original sources' licensing (MEDSL, U.S. Census Bureau — both public domain / open license; Wikimedia state boundary paths — CC BY-SA 3.0).
