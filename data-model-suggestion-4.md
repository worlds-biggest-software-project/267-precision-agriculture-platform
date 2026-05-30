# Data Model Suggestion 4: Time-Series + Relational Hybrid (TimescaleDB)

> Project: Precision Agriculture Platform · Created: 2026-05-24

## Philosophy

This model recognises that a precision agriculture platform is fundamentally a time-series data system with a relational management layer. The overwhelming majority of data volume comes from three sources: IoT sensor readings (soil moisture, temperature, weather, every 5-15 minutes), satellite imagery indices (NDVI, NDRE, per field per capture), and equipment telemetry (GPS position, speed, rates, every second during operations). These are all time-stamped, append-mostly datasets that grow continuously and are queried primarily by time range.

Rather than forcing time-series data into conventional relational tables (which leads to performance problems at scale) or moving it to a separate time-series database (which creates integration complexity), this model uses TimescaleDB — a PostgreSQL extension that provides hypertables with automatic time-based partitioning, continuous aggregates, and compression. The relational management layer (farms, fields, seasons, prescriptions, users) remains standard PostgreSQL. Both live in the same database, share the same connection, and can be joined in a single query.

This is the architecture used by production IoT agricultural platforms and is recommended by infrastructure discussions in the agtech community. TimescaleDB's `time_bucket` function enables efficient temporal aggregations ("average soil moisture per day for the last 90 days") that would require expensive sequential scans in a conventional table. Compression typically reduces storage by 90-95% for older time-series data while keeping it queryable.

**Best for:** Platforms where IoT sensor data volume is high, real-time dashboards with temporal aggregations are a core feature, and the team wants to keep everything in a single PostgreSQL instance rather than operating separate databases.

**Trade-offs:**
- (+) Purpose-built storage engine for time-series data — orders of magnitude faster for temporal queries
- (+) Automatic time-based partitioning and chunk management
- (+) Continuous aggregates for pre-computed hourly/daily/weekly rollups
- (+) 90-95% compression on older data without losing queryability
- (+) Single PostgreSQL database — no separate InfluxDB/Prometheus to operate
- (+) Standard SQL for both relational and time-series queries; can JOIN across both
- (+) `time_bucket` function enables efficient temporal GROUP BY queries
- (-) Requires TimescaleDB extension (not available on all managed PostgreSQL services)
- (-) Hypertables have restrictions: no unique constraints spanning chunks (except on time + partition key)
- (-) Schema design must carefully separate time-series from relational data
- (-) INSERT-heavy workload on hypertables requires tuning (chunk size, compression policies)
- (-) Less familiar to teams that have not used TimescaleDB before

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OGC SensorThings API | Observation hypertable maps directly to SensorThings Observation entity; Datastream is the relational dimension table |
| ADAPT Standard 1.0 | Spatial records from field operations stored in a time-series hypertable for efficient as-applied analysis |
| fiboa | Field boundary in relational layer; fiboa core attributes as columns |
| ISO 28258 | Soil data in relational tables (sampled infrequently, not time-series) |
| AGROVOC | Reference data in relational tables with AGROVOC URI links |
| RFC 7946 (GeoJSON) | PostGIS geometry columns in both relational and hypertable layers |
| ISO 11783 (ISOBUS) | Equipment metadata in relational table; telemetry in hypertable |
| MQTT v5.0 | Sensor readings ingested via MQTT flow directly into hypertable via INSERT |

---

## Relational Layer: Core Management Tables

```sql
-- Organisation (tenant)
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

CREATE INDEX idx_org_member_org ON organisation_member(organisation_id);

CREATE TABLE grower (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_grower_org ON grower(organisation_id);

CREATE TABLE farm (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grower_id       UUID NOT NULL REFERENCES grower(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    country_code    CHAR(2),
    region          TEXT,
    centroid        GEOMETRY(Point, 4326),
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_farm_grower ON farm(grower_id);
CREATE INDEX idx_farm_centroid ON farm USING GIST(centroid);

CREATE TABLE field (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    farm_id         UUID NOT NULL REFERENCES farm(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    boundary        GEOMETRY(Polygon, 4326) NOT NULL,
    area_ha         NUMERIC(12, 4),
    determination_method TEXT,
    source_provider TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_field_farm ON field(farm_id);
CREATE INDEX idx_field_boundary ON field USING GIST(boundary);

CREATE TABLE season (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    crop_code       TEXT NOT NULL,
    year            INTEGER NOT NULL,
    variety         TEXT,
    planting_date   DATE,
    harvest_date    DATE,
    status          TEXT NOT NULL DEFAULT 'planned',
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_season_field ON season(field_id);
CREATE INDEX idx_season_year ON season(year);
```

---

## Relational Layer: Devices & Datastreams (SensorThings Dimension Tables)

```sql
-- Device — the physical IoT thing (dimension table for time-series)
CREATE TABLE device (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    device_type     TEXT NOT NULL,
    location        GEOMETRY(Point, 4326),
    field_id        UUID REFERENCES field(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    hardware        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_device_org ON device(organisation_id);
CREATE INDEX idx_device_field ON device(field_id);

-- Datastream — a specific measurement stream from a device (dimension table)
CREATE TABLE datastream (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id       UUID NOT NULL REFERENCES device(id) ON DELETE CASCADE,
    field_id        UUID REFERENCES field(id),
    observed_property TEXT NOT NULL,         -- soil_moisture_pct, air_temperature_c, rainfall_mm, etc.
    unit            TEXT NOT NULL,           -- %, °C, mm, m/s
    description     TEXT,
    depth_cm        NUMERIC(6, 2),          -- for soil sensors at specific depths
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_datastream_device ON datastream(device_id);
CREATE INDEX idx_datastream_field ON datastream(field_id);
CREATE INDEX idx_datastream_property ON datastream(observed_property);

-- Equipment (dimension table for telemetry time-series)
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    equipment_type  TEXT NOT NULL,
    specs           JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_org ON equipment(organisation_id);
```

---

## Time-Series Layer: Hypertables

### Sensor Observation Hypertable

```sql
-- Core sensor observation time-series — the highest volume table
CREATE TABLE ts_sensor_observation (
    time            TIMESTAMPTZ NOT NULL,
    datastream_id   UUID NOT NULL,          -- FK to datastream (dimension)
    value           DOUBLE PRECISION NOT NULL,
    quality         SMALLINT DEFAULT 0      -- 0=good, 1=suspect, 2=bad, 3=gap_filled
);

-- Convert to TimescaleDB hypertable with 1-day chunks
SELECT create_hypertable('ts_sensor_observation', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

-- Primary query pattern: readings for a datastream in a time range
CREATE INDEX idx_ts_sensor_ds_time ON ts_sensor_observation(datastream_id, time DESC);

-- Compression policy: compress chunks older than 7 days
ALTER TABLE ts_sensor_observation SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'datastream_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ts_sensor_observation', INTERVAL '7 days');

-- Retention policy: drop raw data older than 3 years (aggregates retained indefinitely)
SELECT add_retention_policy('ts_sensor_observation', INTERVAL '3 years');
```

### Continuous Aggregates for Sensor Data

```sql
-- Hourly aggregates — auto-updated by TimescaleDB
CREATE MATERIALIZED VIEW ts_sensor_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    datastream_id,
    avg(value) AS avg_value,
    min(value) AS min_value,
    max(value) AS max_value,
    count(*) AS reading_count
FROM ts_sensor_observation
GROUP BY bucket, datastream_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('ts_sensor_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Daily aggregates — for dashboard charts and trend analysis
CREATE MATERIALIZED VIEW ts_sensor_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', bucket) AS bucket,
    datastream_id,
    avg(avg_value) AS avg_value,
    min(min_value) AS min_value,
    max(max_value) AS max_value,
    sum(reading_count) AS reading_count
FROM ts_sensor_hourly
GROUP BY time_bucket('1 day', bucket), datastream_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('ts_sensor_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);
```

### Equipment Telemetry Hypertable

```sql
CREATE TABLE ts_equipment_telemetry (
    time            TIMESTAMPTZ NOT NULL,
    equipment_id    UUID NOT NULL,
    location        GEOMETRY(Point, 4326),
    speed_kmh       REAL,
    heading_deg     REAL,
    engine_hours    REAL,
    fuel_level_pct  REAL,
    engine_rpm      SMALLINT,
    status          TEXT                    -- idle, moving, working
);

SELECT create_hypertable('ts_equipment_telemetry', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

CREATE INDEX idx_ts_equip_id_time ON ts_equipment_telemetry(equipment_id, time DESC);
CREATE INDEX idx_ts_equip_location ON ts_equipment_telemetry USING GIST(location);

ALTER TABLE ts_equipment_telemetry SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'equipment_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ts_equipment_telemetry', INTERVAL '7 days');
```

### Spatial Record Hypertable (As-Applied / Yield Points)

```sql
CREATE TABLE ts_spatial_record (
    time            TIMESTAMPTZ NOT NULL,
    operation_id    UUID NOT NULL,
    location        GEOMETRY(Point, 4326) NOT NULL,
    rate_value      DOUBLE PRECISION,       -- application rate or yield value
    speed_kmh       REAL,
    section_width_m REAL,
    moisture_pct    REAL,                   -- for harvest records
    extra           JSONB                   -- any additional pass-level data
);

SELECT create_hypertable('ts_spatial_record', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

CREATE INDEX idx_ts_spatial_op ON ts_spatial_record(operation_id, time DESC);
CREATE INDEX idx_ts_spatial_location ON ts_spatial_record USING GIST(location);

ALTER TABLE ts_spatial_record SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'operation_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ts_spatial_record', INTERVAL '30 days');
```

### Vegetation Index Time-Series

```sql
CREATE TABLE ts_vegetation_index (
    time            TIMESTAMPTZ NOT NULL,   -- image capture time
    field_id        UUID NOT NULL,
    index_type      TEXT NOT NULL,           -- ndvi, ndre, evi, lai, savi
    mean_value      REAL,
    min_value       REAL,
    max_value       REAL,
    std_dev         REAL,
    provider        TEXT,                   -- sentinel_2, landsat_8, planet, drone
    cloud_cover_pct REAL,
    raster_url      TEXT
);

SELECT create_hypertable('ts_vegetation_index', 'time',
    chunk_time_interval => INTERVAL '30 days'
);

CREATE INDEX idx_ts_veg_field_time ON ts_vegetation_index(field_id, time DESC);
CREATE INDEX idx_ts_veg_type ON ts_vegetation_index(index_type, time DESC);

-- No compression — low volume, frequently queried for trend charts
```

### Weather Observation Hypertable

```sql
CREATE TABLE ts_weather (
    time                TIMESTAMPTZ NOT NULL,
    station_id          UUID NOT NULL,      -- FK to a weather station (could be a device)
    air_temp_c          REAL,
    humidity_pct        REAL,
    wind_speed_ms       REAL,
    wind_direction_deg  REAL,
    rainfall_mm         REAL,
    solar_radiation_wm2 REAL,
    soil_temp_c         REAL,
    dew_point_c         REAL,
    pressure_hpa        REAL
);

SELECT create_hypertable('ts_weather', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

CREATE INDEX idx_ts_weather_station ON ts_weather(station_id, time DESC);

ALTER TABLE ts_weather SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'station_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ts_weather', INTERVAL '7 days');

-- Daily weather summary
CREATE MATERIALIZED VIEW ts_weather_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    station_id,
    avg(air_temp_c) AS avg_temp_c,
    min(air_temp_c) AS min_temp_c,
    max(air_temp_c) AS max_temp_c,
    sum(rainfall_mm) AS total_rainfall_mm,
    avg(humidity_pct) AS avg_humidity_pct,
    avg(wind_speed_ms) AS avg_wind_speed_ms,
    avg(solar_radiation_wm2) AS avg_solar_wm2
FROM ts_weather
GROUP BY bucket, station_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('ts_weather_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);
```

---

## Relational Layer: Operations, Prescriptions, Scouting

```sql
CREATE TABLE field_operation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    operation_type  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'planned',
    planned_date    DATE,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    operator_id     UUID REFERENCES user_account(id),
    equipment_id    UUID REFERENCES equipment(id),
    details         JSONB NOT NULL DEFAULT '{}',
    source_provider TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_field_op_field ON field_operation(field_id);
CREATE INDEX idx_field_op_season ON field_operation(season_id);

CREATE TABLE management_zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    name            TEXT NOT NULL,
    zone_number     INTEGER NOT NULL,
    geometry        GEOMETRY(Polygon, 4326) NOT NULL,
    area_ha         NUMERIC(12, 4),
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mgmt_zone_field ON management_zone(field_id);
CREATE INDEX idx_mgmt_zone_geom ON management_zone USING GIST(geometry);

CREATE TABLE prescription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    name            TEXT NOT NULL,
    prescription_type TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    created_by      UUID REFERENCES user_account(id),
    approved_by     UUID REFERENCES user_account(id),
    approved_at     TIMESTAMPTZ,
    config          JSONB NOT NULL DEFAULT '{}',
    zones           JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_prescription_field ON prescription(field_id);

CREATE TABLE soil_sample (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    location        GEOMETRY(Point, 4326) NOT NULL,
    sampled_at      DATE NOT NULL,
    sampled_by      UUID REFERENCES user_account(id),
    depth_range     NUMRANGE NOT NULL,
    results         JSONB NOT NULL DEFAULT '{}',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_soil_sample_field ON soil_sample(field_id);
CREATE INDEX idx_soil_sample_location ON soil_sample USING GIST(location);

CREATE TABLE scouting_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    observer_id     UUID NOT NULL REFERENCES user_account(id),
    observed_at     TIMESTAMPTZ NOT NULL,
    location        GEOMETRY(Point, 4326) NOT NULL,
    observation_type TEXT NOT NULL,
    severity        TEXT,
    description     TEXT,
    findings        JSONB NOT NULL DEFAULT '{}',
    photos          JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scouting_field ON scouting_observation(field_id);
CREATE INDEX idx_scouting_location ON scouting_observation USING GIST(location);

CREATE TABLE alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    field_id        UUID REFERENCES field(id),
    alert_type      TEXT NOT NULL,
    severity        TEXT NOT NULL,
    title           TEXT NOT NULL,
    message         TEXT NOT NULL,
    source          TEXT NOT NULL,
    context         JSONB NOT NULL DEFAULT '{}',
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES user_account(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_org ON alert(organisation_id);
CREATE INDEX idx_alert_created ON alert(created_at DESC);

CREATE TABLE carbon_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    assessment_year INTEGER NOT NULL,
    methodology     TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    summary         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carbon_field ON carbon_assessment(field_id);

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

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES user_account(id),
    organisation_id UUID REFERENCES organisation(id),
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    changes         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_time ON audit_log(created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Example Queries

### Dashboard: 90-day soil moisture trend for a field (using daily aggregates)

```sql
SELECT
    d.bucket AS date,
    ds.observed_property,
    ds.depth_cm,
    d.avg_value,
    d.min_value,
    d.max_value
FROM ts_sensor_daily d
JOIN datastream ds ON ds.id = d.datastream_id
WHERE ds.field_id = $1
  AND ds.observed_property = 'soil_moisture_pct'
  AND d.bucket >= now() - INTERVAL '90 days'
ORDER BY ds.depth_cm, d.bucket;
```

### Real-time: Current readings across all devices for a field

```sql
SELECT DISTINCT ON (ds.id)
    dv.name AS device_name,
    ds.observed_property,
    ds.unit,
    ds.depth_cm,
    obs.time,
    obs.value
FROM datastream ds
JOIN device dv ON dv.id = ds.device_id
JOIN LATERAL (
    SELECT time, value
    FROM ts_sensor_observation
    WHERE datastream_id = ds.id
    ORDER BY time DESC
    LIMIT 1
) obs ON true
WHERE ds.field_id = $1
ORDER BY ds.id, obs.time DESC;
```

### Analytics: Growing degree days (GDD) accumulation for a season

```sql
SELECT
    bucket AS date,
    avg_temp_c,
    GREATEST(avg_temp_c - 10, 0) AS daily_gdd,  -- base temp 10°C for corn
    SUM(GREATEST(avg_temp_c - 10, 0)) OVER (ORDER BY bucket) AS cumulative_gdd
FROM ts_weather_daily
WHERE station_id = $1
  AND bucket BETWEEN '2026-04-15' AND '2026-10-01'
ORDER BY bucket;
```

### Correlation: NDVI trend vs soil moisture for a field

```sql
SELECT
    vi.time AS ndvi_date,
    vi.mean_value AS ndvi_mean,
    sm.avg_value AS soil_moisture_avg
FROM ts_vegetation_index vi
JOIN LATERAL (
    SELECT avg(d.avg_value) AS avg_value
    FROM ts_sensor_daily d
    JOIN datastream ds ON ds.id = d.datastream_id
    WHERE ds.field_id = vi.field_id
      AND ds.observed_property = 'soil_moisture_pct'
      AND d.bucket BETWEEN vi.time - INTERVAL '3 days' AND vi.time
) sm ON true
WHERE vi.field_id = $1
  AND vi.index_type = 'ndvi'
  AND vi.time >= '2026-01-01'
ORDER BY vi.time;
```

### Storage: Check compression statistics

```sql
SELECT
    hypertable_name,
    total_chunks,
    compressed_chunks,
    pg_size_pretty(before_compression_total_bytes) AS uncompressed_size,
    pg_size_pretty(after_compression_total_bytes) AS compressed_size,
    ROUND((1 - after_compression_total_bytes::numeric / before_compression_total_bytes) * 100, 1) AS compression_pct
FROM timescaledb_information.hypertable_compression_stats;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organisation, user_account, organisation_member |
| Farm & Field Management | 4 | grower, farm, field, season |
| Devices & Datastreams | 3 | device, datastream, equipment |
| Time-Series Hypertables | 5 | ts_sensor_observation, ts_equipment_telemetry, ts_spatial_record, ts_vegetation_index, ts_weather |
| Continuous Aggregates | 3 | ts_sensor_hourly, ts_sensor_daily, ts_weather_daily |
| Operations & Prescriptions | 3 | field_operation, management_zone, prescription |
| Sampling & Scouting | 2 | soil_sample, scouting_observation |
| Alerts & Carbon | 2 | alert, carbon_assessment |
| Infrastructure | 3 | provider_connection, audit_log |
| **Total** | **28** | 17 relational + 5 hypertables + 3 continuous aggregates + 3 infrastructure |

---

## Key Design Decisions

1. **TimescaleDB hypertables for all high-volume time-series data** — sensor observations, equipment telemetry, spatial records, vegetation indices, and weather all use hypertables. This gives automatic time-based partitioning, compression, and efficient range queries without manual partition management.

2. **Separation of dimension and fact tables** — the `device` and `datastream` tables are relational dimension tables that describe what is being measured. The `ts_sensor_observation` hypertable is the fact table that stores the measurements. This star-schema pattern enables efficient JOINs between descriptive metadata and high-volume readings.

3. **One value per row in sensor observations** — each datastream measures a single property (soil_moisture_pct, air_temperature_c). This denormalised format is optimal for TimescaleDB compression (segment by datastream_id, order by time) and avoids sparse wide rows where most columns are NULL.

4. **Continuous aggregates for dashboard queries** — hourly and daily pre-computed aggregates mean that dashboard time-range queries never touch raw data. TimescaleDB automatically refreshes these as new data arrives, with configurable lag.

5. **Compression policies with segment-by** — compressing by `datastream_id` or `equipment_id` segment keeps data for each sensor physically co-located on disk, making single-sensor time-range queries fast even on compressed chunks. Typical compression ratios are 90-95%.

6. **Retention policy on raw data** — raw 5-minute sensor readings are retained for 3 years; hourly and daily aggregates are retained indefinitely. This balances storage cost with the ability to drill down into recent data when investigating anomalies.

7. **Spatial records as a hypertable** — as-applied and yield point data from equipment passes can generate millions of rows per season per field. Storing them in a hypertable with compression ensures efficient storage and enables time-range queries for comparing application passes.

8. **Relational tables for infrequent data** — soil samples (a few per field per year), prescriptions, scouting observations, and carbon assessments are low-volume and access-pattern diverse. These stay in conventional PostgreSQL tables with standard indexing.

9. **LATERAL joins for "latest value" queries** — the pattern of joining a dimension table to the most recent reading in a hypertable uses LATERAL subqueries, which TimescaleDB optimises well with its skip-scan functionality.

10. **Single database, two paradigms** — the entire system runs in one PostgreSQL instance with the TimescaleDB extension. Relational and time-series tables can be JOINed in a single query, eliminating the data synchronisation challenges that arise from operating separate relational and time-series databases.
