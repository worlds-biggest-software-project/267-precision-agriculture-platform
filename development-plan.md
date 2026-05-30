# Precision Agriculture Platform — Phased Development Plan

> Project: 267-precision-agriculture-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The relational schema is based primarily on **data-model-suggestion-2 (Hybrid Relational + JSONB)** — the best fit for a small team ingesting data from diverse equipment providers — with high-volume time-series tables (`sensor_reading`, `equipment_telemetry`, `spatial_record`) promoted to **TimescaleDB hypertables** as proposed in data-model-suggestion-4. ISO 28258 soil structure and the OGC SensorThings entity chain from data-model-suggestion-1 are reconstructable on export but stored flattened.

---

## Core Requirements Summary

**What it does.** An open, standards-aligned precision agriculture platform that unifies satellite imagery (NDVI and vegetation indices), IoT sensor telemetry, soil sampling, weather, and field operations into one data store, then uses AI to generate variable-rate prescriptions, detect crop stress, and predict yield — without locking farms into a single equipment OEM or input supplier.

**Who uses it.** Large arable operators (1,000+ acres) cutting input costs via VRA; independent agronomists prioritising scouting; crop consultants building per-field advisory services; agri-retailers adding digital layers; sustainability-focused farms needing emissions/carbon audit trails.

**Key differentiators.** Cross-brand interoperability (ISOBUS / ADAPT Standard 1.0 / fiboa / GeoJSON / GeoParquet); AI-native end-to-end VRA generation from fused data layers; an MCP server exposing the unified farm data API to AI agronomic agents (matching Leaf Agriculture's 2025 move); open standards alignment no incumbent has incentive to provide.

**MVP (features.md "Must-have").** Satellite crop monitoring (NDVI/indices), IoT sensor integration, field mapping & visualisation, yield prediction, reporting & analytics.

**v1.1 (features.md "Should-have").** AI imagery analysis (pest/disease/stress), variable-rate prescription generation, weather & agronomic models, equipment integration, scouting & alerts, real-time decision support.

**Backlog.** Hyperspectral imaging, autonomous drone integration, automated decision execution on equipment, supply-chain optimisation, carbon/sustainability tracking.

**Deployment model.** Hybrid: self-hostable (Docker Compose) and SaaS-ready (Kubernetes), with a REST + OpenAPI 3.1 API, a React web dashboard, and an MCP server. Edge analytics on equipment is a backlog concern, scoped as a thin agent that calls the same prescription engine.

**Integration surface.** Sentinel Hub / Copernicus (satellite); Tomorrow.io and Open-Meteo (weather); John Deere Operations Center, Climate FieldView, Trimble Ag, CNH FieldOps, Leaf Agriculture (provider sync via OAuth 2.0); MQTT/CoAP sensor ingestion; LLM provider (Anthropic Claude) for AI features.

**Standards to implement.** RFC 7946 GeoJSON; ADAPT Standard 1.0 (GeoParquet + GeoTIFF + WKT) for prescription/as-applied export; fiboa for field boundaries; ISOXML (ISO 11783-10) for VRA export to controllers; OGC SensorThings API surface for sensor data; OAuth 2.0 (RFC 6749) + OIDC for auth and provider pass-through; OpenAPI 3.1; OWASP API Security Top 10; GDPR workflows; AGROVOC/BBCH reference vocabularies.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | Python 3.12 | The domain is geospatial + ML + remote-sensing heavy; Python has the deepest ecosystem (rasterio, geopandas, shapely, scikit-learn, sentinelhub-py, the Anthropic SDK). Agronomic modelling and AI features dominate the value proposition. |
| API framework | FastAPI | Native OpenAPI 3.1 generation (a hard requirement from standards.md), Pydantic v2 validation for the heavy JSONB-validation burden the hybrid data model imposes, async I/O for fan-out to external provider APIs, first-class dependency injection for tenancy/auth. |
| Database | PostgreSQL 16 + PostGIS 3.4 + TimescaleDB 2.x | PostGIS is mandatory for field boundaries, zones, and spatial records (SRID 4326, GeoJSON export). TimescaleDB hypertables handle the high-ingest sensor/telemetry/spatial-record tables (data-model-suggestion-4). JSONB + GIN gives the provider-agnostic flexibility from data-model-suggestion-2. |
| Object storage | S3-compatible (MinIO self-hosted, AWS S3 SaaS) | Satellite rasters (GeoTIFF), computed index rasters, scouting photos, and GeoParquet exports are large binary blobs unsuited to the DB. |
| Task queue | Celery + Redis | Async workloads: satellite tile fetch/index computation, provider sync jobs, AI inference, prescription export, scheduled NDVI-drop checks. Redis doubles as cache and Celery broker. |
| Scheduled jobs | Celery Beat | Periodic satellite refresh, weather pulls, provider re-sync, alert evaluation. |
| Sensor ingestion | EMQX (MQTT v5 broker) + CoAP gateway (aiocoap) | standards.md calls out MQTT v5 and CoAP (RFC 7252) as the dominant field-sensor transports. A bridge service subscribes to MQTT topics and writes `sensor_reading` rows. |
| ML / geospatial libs | rasterio, geopandas, shapely, numpy, scikit-learn, scikit-image, xarray | NDVI/index math on rasters, zone clustering (k-means), yield regression, image preprocessing. |
| Satellite SDK | sentinelhub-py | Direct Sentinel-2 L2A access from Copernicus Data Space; OGC WMS/WCS compatible. |
| LLM / AI | Anthropic Claude via official Python SDK, with prompt caching | Stress classification, VRA recommendation synthesis, natural-language report generation. Prompt caching cuts cost on repeated system prompts. |
| MCP server | Official MCP Python SDK | Expose unified farm data API as MCP tools (matches Leaf Agriculture; named in standards.md and README as a differentiator). |
| ORM / migrations | SQLAlchemy 2.0 + GeoAlchemy2 + Alembic | Mature PostGIS support via GeoAlchemy2; Alembic for versioned migrations (required by the Definition of Done). |
| Auth | OAuth 2.0 / OIDC via Authlib; Keycloak for self-host SSO | standards.md mandates OAuth 2.0 for both consuming provider APIs and exposing the platform API. Authlib handles both authorization-code provider flows and resource-server token validation. |
| Frontend | React 18 + TypeScript + Vite | SPA dashboard with heavy interactive mapping. TypeScript types generated from the OpenAPI spec keep client/server in sync. |
| Mapping UI | MapLibre GL JS + deck.gl | Open-source (no Mapbox token lock-in), renders field boundaries, NDVI raster tiles, management zones, prescription maps, sensor locations. deck.gl for large point layers (spatial records, yield maps). |
| Frontend data/state | TanStack Query + Zustand | Server-state caching for the API; light client state for map/UI. |
| Map tile serving | TiTiler (FastAPI-based COG tiler) | Serves NDVI/index GeoTIFFs as XYZ/WMS tiles directly from object storage (Cloud-Optimised GeoTIFF). |
| Containerisation | Docker + Docker Compose (dev/self-host); Helm chart (SaaS K8s) | README specifies self-host + SaaS hybrid. |
| Testing | pytest, pytest-asyncio, testcontainers, httpx, Playwright | Unit + integration (real Postgres/PostGIS/Redis via testcontainers) + E2E browser tests. |
| Code quality | ruff (lint+format), mypy (strict), pre-commit | Single fast toolchain; mypy strict given the JSONB-validation risk surface. |
| Package manager | uv (backend), pnpm (frontend) | Fast, lockfile-based, reproducible. |
| Secrets/encryption | SOPS + age (config); application-layer AES-GCM for provider tokens | provider_connection tokens must be encrypted at rest (standards.md OAuth + OWASP). |
| CI | GitHub Actions | Lint, type-check, test matrix, Docker build, Alembic migration check. |
| Observability | OpenTelemetry + Prometheus + structured JSON logging | Required for the audit-trail and SaaS-ops posture. |

### Project Structure

```
precision-agriculture-platform/
├── pyproject.toml                  # uv-managed, backend deps + tool config (ruff, mypy, pytest)
├── uv.lock
├── docker-compose.yml              # postgres+postgis+timescale, redis, minio, emqx, api, worker, web
├── docker-compose.override.yml     # dev hot-reload
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .pre-commit-config.yaml
├── alembic.ini
├── deploy/
│   └── helm/precision-ag/          # SaaS Kubernetes chart
├── migrations/                     # Alembic versions
│   └── versions/
├── src/
│   └── pag/                        # the importable package
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory, router mounting, OpenAPI customisation
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py          # async SQLAlchemy engine/session
│       │   ├── base.py             # declarative base, mixins (TimestampMixin, TenantMixin)
│       │   └── models/             # SQLAlchemy ORM models grouped by domain
│       │       ├── identity.py     # organisation, user_account, organisation_member
│       │       ├── farm.py         # grower, farm, field, season
│       │       ├── iot.py          # device, sensor_reading
│       │       ├── operations.py   # field_operation, spatial_record
│       │       ├── equipment.py    # equipment, equipment_telemetry
│       │       ├── imagery.py      # satellite_image
│       │       ├── soil.py         # soil_sample
│       │       ├── prescription.py # management_zone, prescription
│       │       ├── scouting.py     # scouting_observation, alert
│       │       ├── carbon.py       # carbon_assessment
│       │       ├── integration.py  # provider_connection
│       │       └── audit.py        # audit_log
│       ├── schemas/                # Pydantic v2 request/response + JSONB payload schemas
│       ├── api/
│       │   ├── deps.py             # auth, tenancy, pagination, db-session dependencies
│       │   ├── errors.py           # exception handlers -> RFC 9457 problem+json
│       │   └── routers/            # one module per resource group
│       ├── auth/                   # OAuth2/OIDC resource server + provider authorization flows
│       ├── services/               # business logic (engine layer)
│       │   ├── geo.py              # GeoJSON<->PostGIS, area calc, fiboa validation
│       │   ├── imagery/            # satellite fetch, index computation, COG generation
│       │   ├── zones.py            # management-zone clustering
│       │   ├── prescription/       # VRA generation, ISOXML/ADAPT/shapefile export
│       │   ├── yield_model.py      # yield prediction
│       │   ├── ai/                 # Claude prompts: stress classification, VRA synthesis, reports
│       │   ├── weather.py          # Tomorrow.io / Open-Meteo adapters
│       │   ├── alerts.py           # alert evaluation rules
│       │   └── carbon.py           # MRV calculations
│       ├── integrations/           # external provider clients
│       │   ├── base.py             # ProviderClient protocol, normalisation contract
│       │   ├── john_deere.py
│       │   ├── climate_fieldview.py
│       │   ├── trimble.py
│       │   ├── cnh.py
│       │   ├── leaf.py
│       │   └── sentinelhub.py
│       ├── ingestion/              # MQTT bridge + CoAP gateway + HTTP SensorThings ingest
│       ├── workers/                # Celery app, tasks, beat schedule
│       ├── mcp/                    # MCP server exposing farm data tools
│       └── reporting/             # report builders (PDF/HTML/CSV)
├── web/                            # React + TS + Vite frontend
│   ├── package.json
│   ├── src/
│   │   ├── api/                    # generated OpenAPI client + TanStack Query hooks
│   │   ├── map/                    # MapLibre + deck.gl layers
│   │   ├── features/               # fields, imagery, sensors, prescriptions, scouting, reports
│   │   └── routes/
│   └── tests/                      # Playwright E2E
├── tests/                          # backend tests mirroring src/pag
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                   # sample GeoJSON, ISOXML, Sentinel tiles, provider payloads
└── docs/
    └── api/                        # generated OpenAPI spec snapshot
```

The structure groups by concern (models, schemas, services, integrations) so each phase adds modules without restructuring.

---

## Phase 1: Foundation, Tenancy & Geospatial Core

### Purpose
Stand up the project skeleton, database with PostGIS + TimescaleDB, configuration, auth scaffolding, multi-tenant data model for identity and farm/field hierarchy, and the geospatial primitives (GeoJSON ↔ PostGIS, area calculation, fiboa validation). After this phase a user can authenticate, create an organisation, and CRUD farms and fields with real boundary geometry — the spine everything else hangs from.

### Tasks

#### 1.1 — Project scaffolding & tooling

**What**: Initialise the repo, dependency management, linting/typing, Docker Compose stack, and CI.

**Design**:
- `pyproject.toml` with uv; dependencies: `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `geoalchemy2`, `alembic`, `asyncpg`, `pydantic`, `pydantic-settings`, `authlib`, `celery[redis]`, `redis`, `shapely`, `geopandas`, `rasterio`, `httpx`, `anthropic`, `boto3`. Dev: `pytest`, `pytest-asyncio`, `testcontainers`, `ruff`, `mypy`, `pre-commit`.
- `config.py` Pydantic `Settings`:
```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://redis:6379/0"
    s3_endpoint: str
    s3_bucket: str = "pag"
    s3_access_key: str
    s3_secret_key: str
    jwt_issuer: str
    jwt_audience: str = "pag-api"
    oidc_jwks_url: str
    token_encryption_key: str        # 32-byte base64, for provider tokens (AES-GCM)
    anthropic_api_key: str | None = None
    sentinelhub_client_id: str | None = None
    sentinelhub_client_secret: str | None = None
    environment: Literal["dev", "staging", "prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="PAG_", env_file=".env")
```
- `docker-compose.yml` services: `db` (image `timescale/timescaledb-ha:pg16` with PostGIS), `redis`, `minio`, `emqx`, `api`, `worker`, `beat`, `web`.
- `main.py` app factory: mounts routers, registers exception handlers, sets OpenAPI 3.1 metadata, adds OTel + CORS middleware.
- GitHub Actions: `ruff check`, `ruff format --check`, `mypy src`, `pytest`, `docker build`, `alembic upgrade head` against an ephemeral DB.

**Testing**:
- `Unit: Settings loads from env with PAG_ prefix → correct typed values; missing required field → ValidationError naming the field`.
- `Integration: docker compose up db → connect, run "CREATE EXTENSION postgis, timescaledb" → both extensions present`.
- `E2E (CI): fresh checkout → uv sync, alembic upgrade head, pytest → all green; ruff/mypy pass`.

#### 1.2 — Database base, mixins & migration harness

**What**: Declarative base, common mixins, Alembic configured for PostGIS/TimescaleDB.

**Design**:
```python
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class TenantMixin:
    organisation_id: Mapped[uuid.UUID] = mapped_column(
        ForeignKey("organisation.id", ondelete="CASCADE"), index=True)
```
- Alembic `env.py` imports `geoalchemy2` and configures `compare_type`; first migration enables `postgis`, `timescaledb`, `pgcrypto` (for `gen_random_uuid()`).
- A helper `create_hypertable(table, time_column)` invoked in migrations for time-series tables.

**Testing**:
- `Unit: TimestampMixin sets created_at/updated_at on insert/update (in-memory SQLite-compatible columns where geometry absent)`.
- `Integration (testcontainers Postgres): alembic upgrade head then downgrade base → no errors, extensions created and dropped cleanly`.

#### 1.3 — Identity & multi-tenancy model

**What**: `organisation`, `user_account`, `organisation_member` tables and CRUD.

**Design**: DDL per data-model-suggestion-2 (Identity section): `organisation(settings JSONB)`, `user_account(preferences JSONB, gdpr_consent_at)`, `organisation_member(role, permissions JSONB, UNIQUE(organisation_id,user_id))`. Roles enum: `owner, admin, agronomist, operator, viewer`. Pydantic schemas validate `settings`/`preferences`/`permissions` shapes.

**Testing**:
- `Unit: OrganisationCreate schema rejects missing country_code; slug auto-derived and unique-validated`.
- `Integration: create org, add member with role=agronomist → row persisted; duplicate (org,user) → IntegrityError surfaced as 409`.

#### 1.4 — Auth: OAuth2/OIDC resource server

**What**: Validate bearer JWTs, resolve current user + active organisation, enforce role-based scopes.

**Design**:
- `auth/resource_server.py` validates RS256 JWTs against `oidc_jwks_url` (cached JWKS), checks `iss`/`aud`/`exp`.
- FastAPI deps in `api/deps.py`:
```python
async def current_user(token=Depends(bearer)) -> User: ...
async def current_org(x_org_id: UUID = Header(...), user=Depends(current_user)) -> Organisation: ...
def require_role(*roles: str) -> Callable: ...   # raises 403 if membership role not in roles
```
- All tenant-scoped queries filter by `current_org.id`. Self-host ships Keycloak realm export; SaaS uses hosted OIDC.

**Testing**:
- `Unit: expired token → 401; wrong audience → 401; valid token → User resolved`.
- `Integration (mocked JWKS): request with viewer role to a write endpoint guarded by require_role("admin") → 403`.
- `Integration: valid token, X-Org-Id for an org the user is not a member of → 403`.

#### 1.5 — Geospatial service primitives

**What**: GeoJSON ↔ PostGIS conversion, geodesic area (hectares), fiboa boundary validation.

**Design**:
```python
def geojson_to_geom(feature: dict, expect: GeomType) -> WKBElement: ...   # validates RFC 7946
def geom_to_geojson(geom: WKBElement) -> dict: ...
def area_hectares(polygon: WKBElement) -> Decimal: ...   # ST_Area(geography) / 10000
def validate_fiboa(feature: dict) -> list[FiobaIssue]: ...  # id, geometry, area, determination_method
```
- Reject self-intersecting polygons (`ST_IsValid`), non-4326 input, multipolygons where polygon expected.

**Testing**:
- `Unit: valid Polygon GeoJSON → WKB; area within 0.5% of known reference field`.
- `Unit: self-intersecting polygon → GeometryError`.
- `Unit: fiboa feature missing determination_method → one issue listed; complete feature → no issues`.

#### 1.6 — Farm & field hierarchy CRUD

**What**: `grower`, `farm`, `field`, `season` tables and REST endpoints with GeoJSON I/O.

**Design**: DDL per data-model-suggestion-2 (Farm & Field section). `field.boundary GEOMETRY(Polygon,4326)`, `field.properties JSONB` (fiboa `determination_method`, `source_provider`, `merged_field_id`, agronomic metadata) with GIN index. `area_ha` computed server-side on write via `area_hectares`. Endpoints:
```
POST   /v1/farms                      {grower_id, name, region, ...}
GET    /v1/fields?farm_id=&bbox=      -> FeatureCollection (GeoJSON)
POST   /v1/fields                     Feature (boundary) -> Field
GET    /v1/fields/{id}                -> Feature
PATCH  /v1/fields/{id}
POST   /v1/seasons                    {field_id, crop_code, year, planting_date}
```
- List endpoints support `bbox` spatial filter (`ST_Intersects`) and cursor pagination.

**Testing**:
- `Unit: FieldCreate with valid boundary → area_ha computed and stored`.
- `Integration: POST field then GET → round-trips boundary as valid GeoJSON; bbox query excludes non-intersecting fields`.
- `Integration: create field under farm belonging to another org → 404 (tenant isolation)`.

---

## Phase 2: IoT Sensor Ingestion & Time-Series Store

### Purpose
Add the IoT pipeline: device registry, the high-volume `sensor_reading` hypertable, and the MQTT/CoAP/HTTP ingestion paths aligned to OGC SensorThings. After this phase the platform continuously ingests soil, weather, and telemetry data and can serve it back as time-series — the foundation for monitoring, alerts, and downstream models.

### Tasks

#### 2.1 — Device registry

**What**: `device` table and CRUD with location + capabilities.

**Design**: DDL per data-model-suggestion-2 IoT section: `device(device_type, location GEOMETRY(Point,4326), field_id, hardware JSONB, capabilities TEXT[])`. `device_type` enum: `soil_probe, weather_station, flow_meter, gateway`. An optional `ingest_token` per device (hashed) authenticates HTTP/MQTT writes.

**Testing**:
- `Unit: DeviceCreate validates capabilities against known observed-property vocabulary`.
- `Integration: register device with location → GIST-indexed; query devices within field boundary returns it`.

#### 2.2 — `sensor_reading` hypertable

**What**: TimescaleDB hypertable storing timestamped multi-value JSONB readings.

**Design**: DDL per data-model-suggestion-2 (`sensor_reading`) promoted to hypertable on `observed_at` (chunk interval 7 days). Indexes: `(device_id, observed_at DESC)`, GIN on `readings`, `field_id`. Add a continuous aggregate `sensor_reading_hourly` (TimescaleDB) computing per-device hourly avg/min/max for common keys, refreshed every 30 min. Pydantic `ReadingPayload` validates known keys and ranges (e.g. `soil_moisture_pct` 0–100, `air_temperature_c` −60..60).

**Testing**:
- `Integration: insert 10k readings across 3 devices → time_bucket query returns correct hourly aggregates`.
- `Unit: reading with soil_moisture_pct=150 → ValidationError`.
- `Integration: continuous aggregate refresh → hourly mean matches manual computation on fixture`.

#### 2.3 — Ingestion endpoints (HTTP + SensorThings-compatible)

**What**: HTTP ingest endpoint and an OGC SensorThings-shaped read surface.

**Design**:
```
POST /v1/ingest/readings        # batch: [{device_id, observed_at, readings, quality_flags}]
                                 # auth via device ingest_token OR org bearer
GET  /v1/Things                  # SensorThings: devices as Things
GET  /v1/Datastreams({id})/Observations?$filter=phenomenonTime ge ...
```
- Batch ingest validates and bulk-inserts; rejects future timestamps > 5 min skew. SensorThings read layer maps `device`→Thing, `capability`→Datastream, `sensor_reading` key→Observation.

**Testing**:
- `Integration: batch of 500 valid readings → 202, all persisted; one invalid row → 207 multi-status listing the failure index`.
- `Integration: ingest with wrong device token → 401, nothing written`.
- `Unit: SensorThings $filter on phenomenonTime parsed to time range`.

#### 2.4 — MQTT bridge & CoAP gateway

**What**: Services that subscribe to MQTT v5 topics / accept CoAP POSTs and write readings.

**Design**:
- `ingestion/mqtt_bridge.py`: subscribes to `pag/{org}/{device}/readings`, validates against device token in payload header, inserts via the same service used by HTTP ingest. Runs as a separate process in compose.
- `ingestion/coap_gateway.py` (aiocoap): exposes `coap://.../ingest`, translates CoAP→same ingest path (RFC 7252 ↔ HTTP mapping).
- Backpressure: writes batched every N readings or 2 s.

**Testing**:
- `Integration (testcontainers EMQX): publish reading to MQTT topic → row appears in sensor_reading within 3 s`.
- `Integration: malformed MQTT payload → dropped, error logged, broker connection stays up`.
- `Unit: CoAP payload decoded to ReadingPayload correctly`.

---

## Phase 3: Satellite Imagery & Vegetation Indices

### Purpose
Deliver the headline MVP capability: pull Sentinel-2 imagery for field boundaries, compute NDVI and related indices, store them as Cloud-Optimised GeoTIFFs, and serve them as map tiles with per-field statistics. After this phase a user sees NDVI maps and index time-series for any field — the core "satellite crop monitoring" feature.

### Tasks

#### 3.1 — Satellite fetch service

**What**: Fetch Sentinel-2 L2A scenes clipped to a field's boundary from Copernicus.

**Design**:
- `services/imagery/fetch.py` using sentinelhub-py: given `field_id` + date range + max cloud cover, request bands `B02,B03,B04,B05,B08,B11,B12` clipped to boundary bbox, store raw GeoTIFF to S3 at `imagery/{field_id}/{captured_at}/raw.tif`. Persist a `satellite_image` row (DDL per data-model-suggestion-2): `provider, captured_at, cloud_cover_pct, resolution_m, bbox, storage_url, metadata JSONB`.
- Runs as a Celery task `fetch_imagery(field_id, start, end)`.

**Testing**:
- `Integration (mocked sentinelhub): fetch → satellite_image row created, GeoTIFF uploaded to MinIO`.
- `Unit: scene with cloud_cover > threshold → skipped, logged`.

#### 3.2 — Vegetation index computation

**What**: Compute NDVI, NDRE, EVI, SAVI, LAI rasters and per-field statistics.

**Design**:
- `services/imagery/indices.py` with rasterio/numpy:
```python
def ndvi(nir, red): return (nir - red) / (nir + red + 1e-9)
def ndre(nir, rededge): ...
def evi(nir, red, blue): ...
# stats: mean, min, max, std over field-boundary mask
```
- Output COG per index to `imagery/{field_id}/{captured_at}/{index}.tif`. Store stats + raster_url into `satellite_image.indices JSONB` (shape per data-model-suggestion-2: `{"ndvi": {"mean","min","max","std","raster_url"}}`). GIN index on `indices`.
- Celery chains `fetch_imagery → compute_indices`.

**Testing**:
- `Unit: NDVI on synthetic NIR/red arrays → expected values; division-by-zero guarded`.
- `Integration: compute on fixture scene → COG written, stats within tolerance of reference`.
- `Fixture: known field + scene → NDVI mean matches committed golden value`.

#### 3.3 — Tile serving & imagery API

**What**: Serve index rasters as XYZ tiles and expose imagery/time-series endpoints.

**Design**:
- TiTiler service reads COGs from S3, serves `/tiles/{z}/{x}/{y}?url=...&colormap=...`.
- API:
```
GET /v1/fields/{id}/imagery?index=ndvi&from=&to=  -> [{captured_at, stats, tile_url}]
GET /v1/fields/{id}/imagery/timeseries?index=ndvi  -> [{date, mean}]   # for charts
POST /v1/fields/{id}/imagery/refresh                # enqueue fetch+compute
```

**Testing**:
- `Integration: refresh then GET imagery → list includes new capture with tile_url; timeseries ordered by date`.
- `E2E: TiTiler returns a 256x256 PNG tile for a stored NDVI COG`.

---

## Phase 4: Field Mapping UI, Reporting & Yield Prediction

### Purpose
Complete the MVP by building the web dashboard (field map, NDVI overlays, sensor charts), the reporting/analytics layer, and a yield-prediction model. After this phase the MVP feature set from features.md is fully shippable end-to-end.

### Tasks

#### 4.1 — Web app shell, auth & map

**What**: React/Vite SPA with OIDC login, MapLibre field map, generated API client.

**Design**:
- Generate TS client + types from the OpenAPI spec (openapi-typescript); wrap in TanStack Query hooks.
- MapLibre base map; `FieldsLayer` renders boundaries from the `/v1/fields` FeatureCollection; click → field detail drawer.
- OIDC PKCE login (oidc-client-ts); attach bearer + `X-Org-Id` to requests; org switcher in the header.

**Testing**:
- `E2E (Playwright, mocked API): login → fields render on map; clicking a field opens its detail drawer`.
- `Unit: API client attaches X-Org-Id header from active org store`.

#### 4.2 — Imagery & sensor visualisation

**What**: NDVI tile overlay with time slider; sensor time-series charts.

**Design**: `ImageryLayer` adds TiTiler XYZ source with a colormap legend and a date slider driven by `/imagery` results. `SensorPanel` charts `/sensor-readings` hourly aggregates per device/property.

**Testing**:
- `E2E: select field with imagery → NDVI overlay appears; moving slider swaps tile date`.
- `E2E: device with readings → chart renders non-empty series`.

#### 4.3 — Reporting & analytics

**What**: Field/season summary reports (HTML/PDF/CSV) and an analytics aggregation API.

**Design**:
- `reporting/builders.py` composes a season report: field metadata, NDVI trend, sensor summaries, operations, yield. Render HTML via Jinja2 → PDF via WeasyPrint; CSV export of tabular sections.
```
GET /v1/reports/season/{season_id}?format=pdf|html|csv
GET /v1/analytics/fields/{id}/summary   -> {area_ha, ndvi_trend, last_readings, op_count, ...}
```

**Testing**:
- `Integration: generate season report for fixture data → PDF non-empty, contains field name and NDVI section`.
- `Unit: CSV export of operations has one row per operation with expected columns`.

#### 4.4 — Yield prediction model

**What**: Predict field yield from NDVI trajectory, weather, soil, and historical yield.

**Design**:
- `services/yield_model.py`: feature vector = [peak NDVI, NDVI integral over season, GDD from weather, soil OM/pH, prior-year yield]. v1 model = gradient-boosted regressor (scikit-learn) trained on historical `field_operation` harvest details; falls back to a documented linear baseline when training data is sparse.
- Stored predictions as a `field_operation` of type `yield_prediction` with `details JSONB` `{predicted_yield_kg_ha, confidence_interval, model_version, features}`.
```
POST /v1/fields/{id}/yield/predict?season_id=  -> prediction
```

**Testing**:
- `Unit: feature extraction from fixture season → expected vector`.
- `Integration: predict with no training data → baseline used, model_version="baseline", flagged low-confidence`.
- `Unit: trained model on synthetic monotonic data → prediction increases with peak NDVI`.

---

## Phase 5: Weather, Soil Sampling & Management Zones

### Purpose
Add the agronomic data layers that feed prescriptions: weather ingestion/forecasts, soil sampling, and data-driven management zones. After this phase the platform has all the input layers (NDVI, soil, yield, weather) needed to generate variable-rate prescriptions in Phase 6.

### Tasks

#### 5.1 — Weather integration

**What**: Pull current/forecast/historical weather per field from Tomorrow.io and Open-Meteo.

**Design**:
- `services/weather.py` with adapters behind a `WeatherProvider` protocol returning a normalised `WeatherObservation` (temp, humidity, wind, rainfall, solar, soil temp). Open-Meteo is the free default; Tomorrow.io optional (API key). Scheduled Celery Beat task pulls hourly per active field centroid; computes Growing Degree Days.
```
GET /v1/fields/{id}/weather?from=&to=
GET /v1/fields/{id}/weather/forecast
```

**Testing**:
- `Unit: Open-Meteo response → normalised WeatherObservation`.
- `Integration (mocked HTTP): scheduled pull → weather rows stored; GDD computed correctly`.

#### 5.2 — Soil sampling

**What**: `soil_sample` table (flattened ISO 28258) with JSONB lab results and point geometry.

**Design**: DDL per data-model-suggestion-2 (`soil_sample`): `location Point`, `depth_range NUMRANGE`, `results JSONB` (`{pH, organic_matter_pct, phosphorus_ppm, potassium_ppm, ...}`), GIN on `results`. Endpoints for create/list, plus CSV import of common lab formats. ISO 28258 hierarchy reconstructable on export.

**Testing**:
- `Unit: lab CSV row → SoilSample with results JSONB mapped to canonical keys`.
- `Integration: query samples where (results->'pH'->>'value')::numeric < 6 → returns acidic samples only`.

#### 5.3 — Management zone generation

**What**: Cluster field into management zones from selected data layers.

**Design**:
- `services/zones.py`: rasterise/align chosen layers (NDVI, yield, soil EC/OM) onto a common grid over the field; k-means cluster (configurable k 2–8); polygonise clusters → `management_zone` rows (`geometry`, `area_ha`, `creation_method`, `attributes JSONB` with per-zone means + `cluster_label`).
```
POST /v1/fields/{id}/zones  {layers:["ndvi_2026_05","yield_2025"], k:3, method:"kmeans"}
```

**Testing**:
- `Unit: k-means on synthetic 2-cluster grid → 2 zones with separable means`.
- `Integration: generate zones from fixture layers → polygons cover field, areas sum to field area within tolerance`.

---

## Phase 6: Variable-Rate Prescriptions & Standards Export

### Purpose
Deliver the central differentiating capability: generate variable-rate prescriptions from management zones and agronomic logic, with an approval workflow, then export them in the formats real controllers and partners consume (Shapefile, ISOXML, ADAPT Standard 1.0 GeoParquet). After this phase the platform closes the loop from data to actionable field prescription.

### Tasks

#### 6.1 — Prescription model & rule-based generation

**What**: `management_zone` + `prescription` tables and a rule-based VRA generator.

**Design**: DDL per data-model-suggestion-2 (`prescription`): `prescription_type, status, config JSONB, zones JSONB[]`, status state machine `draft → approved → exported → applied` (+ `archived`). Rule-based generator maps a target nutrient/seeding goal across zones (e.g. higher N where yield potential high, lower where soil already rich), bounded by `min_rate`/`max_rate`.
```
POST /v1/fields/{id}/prescriptions   {type, product, goal, min_rate, max_rate, zone_source}
POST /v1/prescriptions/{id}/approve   (require_role admin/agronomist)
```

**Testing**:
- `Unit: status transition draft→exported without approval → rejected; draft→approved→exported allowed`.
- `Unit: generated rates clamped to [min_rate, max_rate]`.
- `Integration: approve as viewer → 403; as agronomist → status=approved, approved_by/at set`.

#### 6.2 — AI-assisted prescription synthesis

**What**: Use Claude to synthesise an end-to-end VRA recommendation from fused layers, per the AI-native opportunity.

**Design**:
- `services/ai/prescription.py` builds a prompt with field context (crop, zones, NDVI stats, soil results, weather, prior yield) and asks Claude to propose per-zone rates + rationale, returned as structured JSON (tool use). System prompt is cached (prompt caching). Output validated against the prescription schema; always produced as `status=draft` for human approval. `config.algorithm="ai_claude"`, `data_layers_used` recorded.
- Prompt structure (system): role as agronomist, list available layers, require JSON output `{zones:[{zone_id,target_rate,rationale}], overall_rationale}` within rate bounds.

**Testing**:
- `Integration (mocked Claude): fused-layer input → valid per-zone rates within bounds, draft status`.
- `Unit: malformed LLM JSON → schema validation error → task retried/flagged, no partial prescription saved`.

#### 6.3 — Standards-compliant export

**What**: Export prescriptions to Shapefile, ISOXML (ISO 11783-10), and ADAPT Standard 1.0 GeoParquet.

**Design**:
- `services/prescription/export.py`:
  - Shapefile: geopandas write zone polygons + rate attribute, zipped.
  - ISOXML: build TASKDATA.XML with TSK/TZN/PDT/PFD elements and rate grid (the format VRA controllers ingest).
  - ADAPT Standard 1.0: GeoParquet (WKT geometries) + JSON metadata sidecar per the spec.
```
GET /v1/prescriptions/{id}/export?format=shapefile|isoxml|adapt  -> file (sets status→exported)
```

**Testing**:
- `Unit: export shapefile → valid .shp/.dbf/.prj in zip; attribute table has rate column`.
- `Unit: ISOXML validates against ISO 11783-10 schema; TZN count == zone count`.
- `Unit: ADAPT export → GeoParquet readable by geopandas; metadata JSON conforms to ADAPT 1.0 keys`.
- `Integration: export sets prescription.status=exported and audit_log entry`.

---

## Phase 7: External Provider Integration

### Purpose
Connect to incumbent platforms (John Deere, Climate FieldView, Trimble, CNH, Leaf) via OAuth 2.0, normalising their fields, operations, and as-applied data into the unified model — the cross-brand interoperability that is the project's core market wedge. After this phase a farm's existing data flows in automatically and prescriptions flow back out.

### Tasks

#### 7.1 — Provider connection & OAuth flows

**What**: `provider_connection` table with encrypted tokens and authorization-code flows per provider.

**Design**: DDL per data-model-suggestion-2 (`provider_connection`): `credentials JSONB` (tokens AES-GCM encrypted with `token_encryption_key`), `sync_state JSONB`, `UNIQUE(org, provider)`. Authlib clients per provider.
```
GET  /v1/integrations/{provider}/authorize   -> redirect to provider consent
GET  /v1/integrations/{provider}/callback    -> store encrypted tokens, status=active
POST /v1/integrations/{provider}/sync        -> enqueue sync task
```
- Token refresh handled centrally; expired refresh → status=expired + alert.

**Testing**:
- `Unit: token encrypt→decrypt round-trips; ciphertext != plaintext at rest`.
- `Integration (mocked provider OAuth): callback exchanges code → connection active, tokens encrypted`.

#### 7.2 — Provider data normalisation

**What**: `ProviderClient` protocol + per-provider adapters that normalise fields/operations to the unified model.

**Design**:
```python
class ProviderClient(Protocol):
    async def list_fields(self) -> list[NormalisedField]: ...      # -> field rows (GeoJSON boundary)
    async def list_operations(self, field_ext_id) -> list[NormalisedOperation]: ...
    async def push_prescription(self, presc: Prescription) -> str: ...
```
- Adapters for John Deere, Climate FieldView, Trimble, CNH (ISO 15143-3 telematics), Leaf (already-normalised). Sync task upserts by `(source_provider, external_id)`; conflicting boundaries create a `merged_field_id` link (Leaf-style).

**Testing**:
- `Integration (mocked JD API fixture): list_fields → NormalisedField with valid GeoJSON; sync upserts without duplicates on re-run`.
- `Unit: each adapter maps provider operation payload → unified field_operation.details keys`.
- `Integration: push_prescription (mocked) → external_id recorded, status=applied path available`.

---

## Phase 8: AI Crop Stress Detection, Scouting & Alerts

### Purpose
Add AI-native crop intelligence: classify pest/disease/nutrient stress from imagery, capture scouting observations, and run an alert engine over satellite, sensor, and weather signals. After this phase the platform proactively surfaces problems and recommends actions.

### Tasks

#### 8.1 — Scouting observations

**What**: `scouting_observation` table with point geometry, findings, and photos.

**Design**: DDL per data-model-suggestion-2: `observation_type, severity, findings JSONB, photos JSONB[]`, GIN on `findings`. Photo upload to S3; AGROVOC/BBCH references in findings. Mobile-friendly create endpoint.

**Testing**:
- `Integration: create observation with photo → row + S3 object; query by bbox returns it`.
- `Unit: findings for pest type validated against expected keys`.

#### 8.2 — AI stress classification

**What**: Classify scouting/drone photos and detect NDVI-derived stress.

**Design**:
- `services/ai/stress.py`: send a scouting photo to Claude (vision) with a classification prompt → `{label, confidence, recommended_action}` written into `scouting_photo.ai_classification`. NDVI-anomaly detection: compare current vs trailing field NDVI; flag zones with significant drop.
- Treatment recommendation references `input_product` and crop/pest context.

**Testing**:
- `Integration (mocked Claude vision): photo → classification stored with confidence`.
- `Unit: NDVI current 0.45 vs previous 0.72 → drop 37% → stress flag raised`.

#### 8.3 — Alert engine

**What**: `alert` table + rule evaluation over sensors, weather, imagery.

**Design**: DDL per data-model-suggestion-2 (`alert`): `alert_type, severity, source, context JSONB`. Celery Beat evaluates rules: NDVI drop, frost forecast, soil-moisture threshold, sensor offline (no reading in N hours), pest detected. Each rule emits an alert with `recommended_action`; notifications via email/push per user prefs.
```
GET  /v1/alerts?status=unacknowledged
POST /v1/alerts/{id}/acknowledge
```

**Testing**:
- `Unit: frost rule on forecast min_temp < 0 → critical alert created`.
- `Integration: device silent > threshold → sensor_offline alert; acknowledge → acknowledged_at set`.
- `Unit: duplicate alert within dedupe window → suppressed`.

---

## Phase 9: MCP Server, Public API Hardening & GDPR

### Purpose
Make the platform agent-accessible and production-safe: ship an MCP server exposing farm data to AI agents (a named differentiator), harden the public API against the OWASP API Security Top 10, finalise the OpenAPI spec, and implement GDPR consent/access-log/deletion workflows. After this phase the platform is a secure, agent-ready, standards-compliant product.

### Tasks

#### 9.1 — MCP server

**What**: Expose unified farm data as MCP tools for AI agents.

**Design**:
- `mcp/server.py` (MCP Python SDK) exposes tools: `list_fields`, `get_field_imagery`, `get_sensor_readings`, `get_weather`, `list_operations`, `generate_prescription_draft`, `list_alerts`. Auth via per-org API key mapped to scopes; all calls tenant-scoped and audit-logged. Read tools are safe; `generate_prescription_draft` only ever produces drafts.

**Testing**:
- `Integration (MCP client harness): list_fields tool → fields for the authenticated org only`.
- `Unit: tool input schema validation rejects missing field_id`.
- `Integration: cross-tenant field_id via MCP → denied`.

#### 9.2 — API hardening (OWASP API Security Top 10)

**What**: Rate limiting, object-level authorization audit, input/output validation, error hygiene.

**Design**:
- Per-token + per-IP rate limiting (Redis). Centralised object-level auth helper asserting tenant ownership on every `{id}` route (mitigates BOLA/API1). RFC 9457 problem+json errors with no stack leakage. Security headers; CORS allowlist. OpenAPI spec snapshot committed to `docs/api/` and diffed in CI.

**Testing**:
- `Integration: exceed rate limit → 429 with Retry-After`.
- `Integration: access another org's resource by guessed UUID across all resource types → 404/403 (BOLA test matrix)`.
- `CI: generated OpenAPI matches committed snapshot or build fails`.

#### 9.3 — GDPR workflows

**What**: Consent capture, `audit_log` access tracking, data export and deletion.

**Design**: `audit_log` (data-model-suggestion-2) records read/write/export/delete with before/after diffs and IP. Endpoints: record consent (`user_account.gdpr_consent_at`), `GET /v1/me/export` (full personal/farm data bundle), `POST /v1/me/deletion-request` → async erase/anonymise across tenant data with audit entry. Deletion respects legal-hold flags.

**Testing**:
- `Integration: export request → bundle includes user's fields/observations; no other tenant data`.
- `Integration: deletion request → personal data anonymised, audit_log retains the deletion event`.
- `Unit: every write endpoint emits an audit_log row (middleware coverage test)`.

---

## Phase 10: Carbon/Sustainability, Equipment Telemetry & Deployment

### Purpose
Complete the backlog-tier differentiators and productionise: carbon MRV tracking, equipment telemetry hypertable, the as-applied `spatial_record` hypertable, and full deployment artefacts (Helm chart, observability, backups). After this phase the platform is feature-complete against the README roadmap and deployable to SaaS scale.

### Tasks

#### 10.1 — Equipment telemetry & spatial records

**What**: `equipment`, `equipment_telemetry` (hypertable), `spatial_record` (hypertable) for as-applied/yield point data.

**Design**: DDL per data-model-suggestion-2; `equipment_telemetry` and `spatial_record` are TimescaleDB hypertables on their time columns. Telemetry ingested from provider sync (Phase 7) and ISO 15143-3 feeds. Spatial records back as-applied/yield maps and feed the yield model and zone generation.

**Testing**:
- `Integration: bulk-insert 100k spatial records → hypertable chunked; bbox+time query performant`.
- `Unit: as-applied points aggregate to per-zone applied-rate summary`.

#### 10.2 — Carbon & sustainability MRV

**What**: `carbon_assessment` table and emissions/sequestration calculations.

**Design**: DDL per data-model-suggestion-2 (`carbon_assessment.summary JSONB` with line items). `services/carbon.py` computes line items from `field_operation` inputs (fertiliser emissions via default factors), fuel from telemetry, tillage-reduction credit, modelled soil-carbon change; status `draft→submitted→verified→issued`; methodology e.g. `verra_vm0042`.
```
POST /v1/fields/{id}/carbon/assess?season_id=&methodology=
```

**Testing**:
- `Unit: fertiliser line item = rate × area × emission_factor (matches reference)`.
- `Integration: assess fixture season → total_co2e_tonnes equals sum of line items`.

#### 10.3 — Deployment, observability & backups

**What**: Helm chart, OTel/Prometheus dashboards, automated DB backups, runbook.

**Design**: Helm chart for API/worker/beat/web/ingestion + managed Postgres(Timescale)/Redis/S3. OTel traces → collector; Prometheus metrics (ingest rate, task queue depth, API latency); Grafana dashboards. `pg_dump`/continuous archiving for Postgres; lifecycle policies for S3. Self-host path remains `docker compose up`.

**Testing**:
- `Integration: helm template renders valid manifests (kubeconform)`.
- `E2E (kind cluster, smoke): deploy chart → /healthz green, one field create round-trips`.
- `Integration: backup then restore into fresh DB → row counts match`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Geospatial Core        ─── required by everything
    │
    ├── Phase 2: IoT Sensor Ingestion & Time-Series    ─── requires P1
    │       │
    │       └── (feeds) Phase 8 alerts, Phase 5 zones
    │
    ├── Phase 3: Satellite Imagery & Indices           ─── requires P1
    │       │
    │       └── (feeds) Phase 4 UI, Phase 5 zones, Phase 8 stress
    │
    └── Phase 4: Mapping UI, Reporting & Yield         ─── requires P1, P2, P3   ← MVP COMPLETE
            │
Phase 5: Weather, Soil Sampling & Management Zones     ─── requires P1, P3 (P2 optional)
    │
Phase 6: Variable-Rate Prescriptions & Export         ─── requires P5
    │
Phase 7: External Provider Integration                ─── requires P1 (P6 for push-back)
    │
Phase 8: AI Stress Detection, Scouting & Alerts       ─── requires P2, P3
    │
Phase 9: MCP Server, API Hardening & GDPR             ─── requires P1 (broad surface; do late)
    │
Phase 10: Carbon, Equipment Telemetry & Deployment    ─── requires P6, P7
```

**Parallelism opportunities:**
- After Phase 1: **Phases 2 and 3 can be developed concurrently** (independent pipelines).
- After the MVP (Phase 4): **Phase 5 and Phase 7 can proceed in parallel** (agronomic layers vs provider integration).
- **Phase 8** can begin once Phases 2 and 3 are done, in parallel with Phases 5–7.
- The **frontend (Phases 4.1–4.2)** can be built incrementally against the OpenAPI spec as backend endpoints land.

**MVP milestone:** Phases 1–4 deliver the complete "Must-have" feature set from features.md.
**v1.1 milestone:** Phases 5–8 deliver the "Should-have" set.
**Backlog/productionisation:** Phases 9–10.

---

## Definition of Done (per phase)

Every phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass (`pytest`); new code has meaningful coverage.
3. Linting and formatting pass (`ruff check`, `ruff format --check`).
4. Type checking passes (`mypy src` in strict mode).
5. Docker images build (`docker build` for affected services) and `docker compose up` brings the stack healthy.
6. The phase's feature works end-to-end (demonstrated by an integration or E2E test).
7. New configuration options are documented and added to `config.py` / `.env.example`.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec, and the committed `docs/api/` snapshot is updated (CI diff passes).
9. Database changes ship as Alembic migrations that `upgrade head` and `downgrade` cleanly, with hypertables/extensions created idempotently.
10. Tenant isolation is verified for every new resource (a cross-org access test exists and passes).
11. Any new write endpoint emits an `audit_log` entry (from Phase 9 onward, enforced by middleware test).
12. Standards-affecting outputs (GeoJSON, ISOXML, ADAPT GeoParquet, SensorThings, OpenAPI) validate against their respective schemas in tests.
```
