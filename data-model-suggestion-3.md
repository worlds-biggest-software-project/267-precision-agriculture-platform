# Data Model Suggestion 3: Event-Sourced / Audit-First (CQRS)

> Project: Precision Agriculture Platform · Created: 2026-05-24

## Philosophy

This model treats every change to the system as an immutable event in a central event store. The event store is the single source of truth. All queryable state — field boundaries, current sensor readings, season summaries, prescription statuses — is derived by replaying or projecting events into materialised read models (views). This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from purpose-built projections.

This architecture is a natural fit for precision agriculture because the domain is inherently event-driven: a sensor emits a reading, a tractor completes a pass, a satellite captures an image, an agronomist approves a prescription, a boundary changes. Every one of these is a discrete event with a timestamp, an actor, and a payload. In conventional relational models, these events are destructively merged into the current state of a row. In an event-sourced model, the full history is preserved, enabling temporal queries ("what was the NDVI trend for this field between March and June 2025?"), complete audit trails for carbon credit MRV verification, and AI training on historical event sequences.

Event sourcing is used in production by financial trading platforms, healthcare record systems, and IoT telemetry platforms where audit requirements are strict and temporal queries are common. For a precision agriculture platform that must support carbon credit verification (which requires proving what practices were applied and when), GDPR data access logging, and AI-powered trend analysis across seasons, the event-sourced approach provides these capabilities as inherent properties of the data model rather than bolted-on features.

**Best for:** Platforms where full audit trails are non-negotiable, AI/ML training requires historical event sequences, carbon credit MRV verification demands provable practice records, and temporal queries are a core feature.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is recorded forever
- (+) Temporal queries are trivial: replay events up to any point in time
- (+) AI/ML training on event sequences (e.g., predict yield from operation event patterns)
- (+) Carbon credit MRV verification backed by provable event chains
- (+) GDPR "right to access" satisfied by filtering event stream per user
- (+) Adding new event types requires zero schema migrations
- (-) Higher storage requirements (events are never deleted, only compacted)
- (-) Read model projections must be maintained — adds operational complexity
- (-) Eventually consistent reads (small delay between write and projection update)
- (-) Simple CRUD queries require projection tables — more infrastructure
- (-) Debugging projection bugs requires replaying event streams
- (-) Team must learn event sourcing patterns — less familiar than CRUD

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OGC SensorThings API | Sensor observations modelled as `SensorReadingRecorded` events; projections rebuild Datastream views |
| ADAPT Standard 1.0 | Field operations recorded as events (`OperationStarted`, `OperationCompleted`, `SpatialRecordCaptured`) |
| fiboa | Field boundary changes recorded as `FieldBoundaryUpdated` events; current boundary is latest projection |
| ISO 28258 | Soil sampling workflow captured as event chain: `SoilSiteCreated` > `SampleCollected` > `AnalysisCompleted` |
| AGROVOC | Vocabulary URIs embedded in event payloads for crop, pest, and growth stage references |
| GDPR (EU 2016/679) | Event stream provides complete data access history; crypto-erasure for right-to-be-forgotten |
| Carbon MRV | Immutable event chain proves practice adoption timeline for verification |
| RFC 7946 (GeoJSON) | Geometry data in event payloads stored as GeoJSON; projected into PostGIS columns in read models |

---

## Event Store

```sql
-- The central event store — append-only, immutable
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,           -- aggregate root ID (field_id, device_id, etc.)
    stream_type     TEXT NOT NULL,            -- 'field', 'device', 'operation', 'prescription', etc.
    event_type      TEXT NOT NULL,            -- e.g. 'FieldCreated', 'SensorReadingRecorded'
    event_version   INTEGER NOT NULL,         -- version within the stream for optimistic concurrency
    occurred_at     TIMESTAMPTZ NOT NULL,     -- when the real-world event happened
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when we stored it
    actor_id        UUID,                     -- user or system that caused the event
    actor_type      TEXT NOT NULL DEFAULT 'user',  -- 'user', 'system', 'sensor', 'provider_sync'
    organisation_id UUID NOT NULL,            -- tenant partition
    payload         JSONB NOT NULL,           -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation IDs, causation chains, IP, etc.
    UNIQUE (stream_id, event_version)
);

-- Primary query path: events for a specific aggregate, in order
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);

-- Query by type for building projections
CREATE INDEX idx_event_type ON event_store(event_type, recorded_at);

-- Tenant-scoped queries
CREATE INDEX idx_event_org ON event_store(organisation_id, recorded_at);

-- Time-range queries for temporal analysis
CREATE INDEX idx_event_occurred ON event_store(occurred_at);

-- Partition by month for performance at scale
-- (In production, use declarative partitioning: PARTITION BY RANGE (recorded_at))
```

---

## Event Type Catalogue

Below are the core event types. Each event type has a defined payload schema.

### Farm & Field Events

```
FieldCreated
  stream_type: 'field'
  payload: {
    "farm_id": "uuid",
    "name": "North 40",
    "boundary": {"type": "Polygon", "coordinates": [...]},   -- GeoJSON
    "area_ha": 16.2,
    "determination_method": "gps_survey",
    "source_provider": "john_deere"
  }

FieldBoundaryUpdated
  payload: {
    "previous_boundary": {"type": "Polygon", "coordinates": [...]},
    "new_boundary": {"type": "Polygon", "coordinates": [...]},
    "new_area_ha": 16.5,
    "reason": "survey_correction"
  }

FieldDeactivated
  payload: {
    "reason": "sold"
  }

SeasonCreated
  stream_type: 'field'
  payload: {
    "season_id": "uuid",
    "crop_code": "corn",
    "agrovoc_uri": "http://aims.fao.org/aos/agrovoc/c_4932",
    "year": 2026,
    "variety": "DKC 62-08",
    "target_population_seeds_ha": 80000
  }
```

### Sensor & IoT Events

```
DeviceRegistered
  stream_type: 'device'
  payload: {
    "name": "Soil Probe Station A",
    "device_type": "soil_probe",
    "location": {"type": "Point", "coordinates": [-89.5, 41.3]},
    "field_id": "uuid",
    "hardware": {"manufacturer": "Sentek", "model": "Drill&Drop", "serial": "DD-1234"}
  }

SensorReadingRecorded
  stream_type: 'device'
  payload: {
    "readings": {
      "soil_moisture_pct": 32.5,
      "soil_temperature_c": 18.2,
      "soil_ec_ms_cm": 0.45,
      "depth_cm": 30
    },
    "quality_flags": {"outlier_detected": false}
  }

DeviceOffline
  payload: {
    "last_reading_at": "2026-05-23T08:00:00Z",
    "reason": "battery_low"
  }
```

### Field Operation Events

```
OperationPlanned
  stream_type: 'operation'
  payload: {
    "field_id": "uuid",
    "season_id": "uuid",
    "operation_type": "application",
    "planned_date": "2026-06-15",
    "inputs": [{"product": "Urea 46-0-0", "rate": 200, "unit": "kg/ha"}],
    "equipment_id": "uuid"
  }

OperationStarted
  payload: {
    "started_at": "2026-06-15T07:30:00Z",
    "operator_id": "uuid",
    "weather": {"temp_c": 18, "wind_kmh": 8, "humidity_pct": 55}
  }

SpatialRecordCaptured
  payload: {
    "geometry": {"type": "Point", "coordinates": [-89.501, 41.302]},
    "recorded_at": "2026-06-15T07:35:22Z",
    "values": {"rate_kg_ha": 195.3, "speed_kmh": 8.5, "depth_cm": 5.0}
  }

OperationCompleted
  payload: {
    "completed_at": "2026-06-15T14:20:00Z",
    "actual_area_ha": 16.0,
    "actual_inputs": [{"product": "Urea 46-0-0", "total_kg": 3200, "avg_rate_kg_ha": 200}]
  }

HarvestRecorded
  payload: {
    "crop": "corn",
    "total_yield_kg": 204000,
    "yield_per_ha": 12750,
    "moisture_pct": 15.2,
    "test_weight_kg_hl": 72.5,
    "quality_grade": "US No. 2"
  }
```

### Satellite & Imagery Events

```
SatelliteImageCaptured
  stream_type: 'field'
  payload: {
    "image_id": "uuid",
    "provider": "sentinel_2",
    "captured_at": "2026-06-10T10:30:00Z",
    "cloud_cover_pct": 5.2,
    "resolution_m": 10.0,
    "storage_url": "s3://imagery/..."
  }

VegetationIndexComputed
  stream_type: 'field'
  payload: {
    "image_id": "uuid",
    "index_type": "ndvi",
    "mean": 0.72,
    "min": 0.15,
    "max": 0.88,
    "std": 0.08,
    "raster_url": "s3://indices/..."
  }

CropStressDetected
  payload: {
    "image_id": "uuid",
    "stress_type": "nutrient_deficiency",
    "severity": "medium",
    "affected_area_pct": 22,
    "affected_geometry": {"type": "Polygon", "coordinates": [...]},
    "confidence": 0.87,
    "model_version": "stress-detect-v3.1"
  }
```

### Scouting Events

```
ScoutingObservationRecorded
  stream_type: 'field'
  payload: {
    "observer_id": "uuid",
    "location": {"type": "Point", "coordinates": [-89.503, 41.304]},
    "observation_type": "pest",
    "pest_name": "corn rootworm",
    "pest_agrovoc_uri": "http://aims.fao.org/aos/agrovoc/c_...",
    "severity": "medium",
    "count_per_trap": 15,
    "photos": [{"url": "s3://...", "ai_classification": {"label": "corn_rootworm_adult", "confidence": 0.92}}],
    "growth_stage_bbch": "65"
  }
```

### Prescription Events

```
PrescriptionCreated
  stream_type: 'prescription'
  payload: {
    "field_id": "uuid",
    "prescription_type": "fertiliser",
    "product": "Urea 46-0-0",
    "zones": [
      {"zone_id": "uuid", "target_rate": 250, "rate_unit": "kg/ha"},
      {"zone_id": "uuid", "target_rate": 180, "rate_unit": "kg/ha"}
    ],
    "algorithm": "linear_interpolation",
    "data_layers_used": ["yield_2025", "soil_om", "ndvi_2026_03"]
  }

PrescriptionApproved
  payload: {
    "approved_by": "uuid",
    "notes": "Rates adjusted based on soil test results"
  }

PrescriptionExported
  payload: {
    "format": "shapefile",
    "export_url": "s3://exports/...",
    "target_equipment_id": "uuid"
  }
```

### Carbon & Sustainability Events

```
CarbonAssessmentStarted
  stream_type: 'carbon_assessment'
  payload: {
    "field_id": "uuid",
    "assessment_year": 2026,
    "methodology": "verra_vm0042"
  }

CarbonLineItemCalculated
  payload: {
    "category": "soil_carbon_change",
    "co2e_tonnes": -18.0,
    "data_source": "modelled",
    "model_inputs": {"soc_2025": 2.1, "soc_2026": 2.3, "area_ha": 16.2}
  }

CarbonAssessmentVerified
  payload: {
    "verifier": "Verra",
    "total_co2e_tonnes": -12.5,
    "certificate_id": "VCS-2026-12345",
    "verification_report_url": "https://..."
  }
```

---

## Read Model Projections

Projections are materialised views rebuilt from the event store. They are the query side of CQRS.

### Field Current State Projection

```sql
-- Rebuilt by processing FieldCreated, FieldBoundaryUpdated, FieldDeactivated events
CREATE TABLE proj_field (
    field_id        UUID PRIMARY KEY,
    farm_id         UUID NOT NULL,
    organisation_id UUID NOT NULL,
    name            TEXT NOT NULL,
    boundary        GEOMETRY(Polygon, 4326),
    area_ha         NUMERIC(12, 4),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    source_provider TEXT,
    last_event_id   UUID NOT NULL,           -- watermark for projection rebuild
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_field_org ON proj_field(organisation_id);
CREATE INDEX idx_proj_field_farm ON proj_field(farm_id);
CREATE INDEX idx_proj_field_boundary ON proj_field USING GIST(boundary);
```

### Season Summary Projection

```sql
CREATE TABLE proj_season (
    season_id       UUID PRIMARY KEY,
    field_id        UUID NOT NULL,
    crop_code       TEXT NOT NULL,
    year            INTEGER NOT NULL,
    variety         TEXT,
    planting_date   DATE,
    harvest_date    DATE,
    status          TEXT NOT NULL,
    yield_per_ha    NUMERIC(10, 2),
    operations_count INTEGER NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_season_field ON proj_season(field_id);
CREATE INDEX idx_proj_season_year ON proj_season(year);
```

### Latest Sensor Reading Projection

```sql
-- One row per device, updated on each SensorReadingRecorded event
CREATE TABLE proj_device_latest (
    device_id       UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    name            TEXT NOT NULL,
    device_type     TEXT NOT NULL,
    field_id        UUID,
    location        GEOMETRY(Point, 4326),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    latest_readings JSONB,
    latest_reading_at TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_device_org ON proj_device_latest(organisation_id);
CREATE INDEX idx_proj_device_field ON proj_device_latest(field_id);
```

### Field Operation Projection

```sql
CREATE TABLE proj_operation (
    operation_id    UUID PRIMARY KEY,
    field_id        UUID NOT NULL,
    season_id       UUID,
    operation_type  TEXT NOT NULL,
    status          TEXT NOT NULL,
    planned_date    DATE,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    operator_id     UUID,
    equipment_id    UUID,
    details         JSONB NOT NULL DEFAULT '{}',
    spatial_records_count INTEGER NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_op_field ON proj_operation(field_id);
CREATE INDEX idx_proj_op_season ON proj_operation(season_id);
```

### NDVI Time Series Projection

```sql
-- One row per field per image capture, for trend charts
CREATE TABLE proj_ndvi_timeseries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL,
    captured_at     TIMESTAMPTZ NOT NULL,
    ndvi_mean       NUMERIC(8, 6),
    ndvi_min        NUMERIC(8, 6),
    ndvi_max        NUMERIC(8, 6),
    ndvi_std        NUMERIC(8, 6),
    provider        TEXT,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_ndvi_field ON proj_ndvi_timeseries(field_id);
CREATE INDEX idx_proj_ndvi_time ON proj_ndvi_timeseries(captured_at);
CREATE UNIQUE INDEX idx_proj_ndvi_unique ON proj_ndvi_timeseries(field_id, captured_at);
```

### Alert Projection

```sql
CREATE TABLE proj_alert (
    alert_id        UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    field_id        UUID,
    alert_type      TEXT NOT NULL,
    severity        TEXT NOT NULL,
    title           TEXT NOT NULL,
    message         TEXT NOT NULL,
    source          TEXT NOT NULL,
    context         JSONB NOT NULL DEFAULT '{}',
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_alert_org ON proj_alert(organisation_id);
CREATE INDEX idx_proj_alert_time ON proj_alert(created_at DESC);
```

### Carbon Assessment Projection

```sql
CREATE TABLE proj_carbon_assessment (
    assessment_id   UUID PRIMARY KEY,
    field_id        UUID NOT NULL,
    assessment_year INTEGER NOT NULL,
    methodology     TEXT NOT NULL,
    status          TEXT NOT NULL,
    total_co2e_tonnes NUMERIC(12, 4),
    line_items      JSONB NOT NULL DEFAULT '[]',
    verifier        TEXT,
    verified_at     DATE,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_carbon_field ON proj_carbon_assessment(field_id);
```

---

## Projection Checkpoint Tracking

```sql
-- Tracks the last processed event for each projection
CREATE TABLE projection_checkpoint (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'running',  -- running, paused, rebuilding, error
    error_message   TEXT,
    events_processed BIGINT NOT NULL DEFAULT 0
);
```

---

## Snapshot Store (Optional Optimisation)

```sql
-- Periodic snapshots of aggregate state to avoid replaying full event history
CREATE TABLE aggregate_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INTEGER NOT NULL,       -- event_version at snapshot time
    state           JSONB NOT NULL,          -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

---

## Supporting Tables (Non-Event)

```sql
-- Organisation and user tables remain conventional (they are infrastructure, not domain)
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,
    preferences     JSONB NOT NULL DEFAULT '{}',
    gdpr_consent_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES user_account(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'viewer',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

-- Provider connections (OAuth tokens) are infrastructure
CREATE TABLE provider_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    credentials     JSONB NOT NULL DEFAULT '{}',
    sync_state      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, provider)
);
```

---

## Example Queries

### Temporal query: What was the field boundary on a specific date?

```sql
-- Replay boundary events up to a target date
SELECT payload->>'new_boundary' AS boundary
FROM event_store
WHERE stream_id = $1                        -- field UUID
  AND stream_type = 'field'
  AND event_type IN ('FieldCreated', 'FieldBoundaryUpdated')
  AND occurred_at <= '2025-09-01T00:00:00Z'
ORDER BY event_version DESC
LIMIT 1;
```

### Audit query: All actions taken on a field in the last 30 days

```sql
SELECT event_type, occurred_at, actor_id, payload
FROM event_store
WHERE stream_id = $1
  AND occurred_at >= now() - INTERVAL '30 days'
ORDER BY event_version ASC;
```

### Carbon MRV: Prove practice adoption timeline for a field

```sql
SELECT event_type, occurred_at, payload
FROM event_store
WHERE stream_id IN (
    -- All operation streams for this field
    SELECT DISTINCT stream_id
    FROM event_store
    WHERE event_type = 'OperationPlanned'
      AND payload->>'field_id' = $1
)
AND event_type IN ('OperationPlanned', 'OperationStarted', 'OperationCompleted')
AND occurred_at BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY occurred_at ASC;
```

### AI training: Extract sensor reading sequences for a field

```sql
SELECT occurred_at, payload->'readings' AS readings
FROM event_store
WHERE stream_type = 'device'
  AND event_type = 'SensorReadingRecorded'
  AND stream_id IN (
    SELECT stream_id FROM event_store
    WHERE event_type = 'DeviceRegistered'
      AND payload->>'field_id' = $1
  )
  AND occurred_at BETWEEN '2026-03-01' AND '2026-09-30'
ORDER BY occurred_at ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | event_store (partitioned by month in production) |
| Snapshots | 1 | aggregate_snapshot |
| Projection Tracking | 1 | projection_checkpoint |
| Read Model Projections | 7 | proj_field, proj_season, proj_device_latest, proj_operation, proj_ndvi_timeseries, proj_alert, proj_carbon_assessment |
| Infrastructure (non-event) | 4 | organisation, user_account, organisation_member, provider_connection |
| **Total** | **14** | Plus additional projections as needed |

---

## Key Design Decisions

1. **Single event_store table with stream partitioning** — all events live in one table, partitioned by `recorded_at` for performance. The `stream_id` + `stream_type` combination identifies the aggregate root. This is simpler than per-aggregate-type event tables and allows cross-aggregate temporal queries.

2. **Optimistic concurrency via event_version** — the `UNIQUE (stream_id, event_version)` constraint prevents concurrent writes to the same aggregate. Writers must include the expected next version, and the database rejects conflicts.

3. **GeoJSON in event payloads** — geometry data is stored as GeoJSON within JSONB payloads (not PostGIS columns) in the event store. The projection layer converts GeoJSON to PostGIS geometry columns in read models, keeping the event store format-agnostic and portable.

4. **Projections are disposable** — every projection table can be dropped and rebuilt by replaying the event store from the beginning (or from the last snapshot). This makes schema changes to read models safe: add a new projection, rebuild it, switch traffic, drop the old one.

5. **Snapshot store for long-lived aggregates** — fields with thousands of events (sensor readings over years) would be slow to rebuild from scratch. Periodic snapshots store the aggregate state at a point in time, allowing replay to start from the snapshot.

6. **Infrastructure tables outside the event model** — organisations, users, and OAuth tokens use conventional CRUD tables because they are platform infrastructure, not domain events. Putting user password changes in the event store would create unnecessary complexity.

7. **Event type catalogue as documentation** — the event types are not enforced by the database but documented as a contract. Application code validates event payloads against JSON Schema definitions before writing to the store.

8. **Carbon MRV as a first-class benefit** — the immutable event chain directly proves what practices were applied and when, satisfying carbon credit verification requirements without any additional audit infrastructure.

9. **GDPR compliance through crypto-erasure** — instead of deleting events (which would break the event stream), personal data in payloads is encrypted with per-user keys. "Right to be forgotten" is implemented by destroying the user's encryption key, rendering their data in events unreadable.

10. **AI training pipeline** — the event store is a natural source for ML training data. Sequences of sensor readings, operations, and yield outcomes for a field across seasons form the training examples for yield prediction and prescription optimisation models.
