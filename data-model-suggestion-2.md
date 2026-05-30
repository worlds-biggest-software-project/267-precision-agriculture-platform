# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Precision Agriculture Platform · Created: 2026-05-24

## Philosophy

This model uses PostgreSQL's JSONB columns strategically alongside relational tables to handle the inherent variability of precision agriculture data. Core entities (organisations, farms, fields, seasons) remain fully relational with strict foreign keys. However, sensor readings, observation payloads, operation parameters, and jurisdiction-specific metadata use JSONB columns, avoiding the combinatorial explosion of columns that arises when different sensor types, equipment brands, and regional requirements each demand their own fields.

This is the approach taken by modern agtech platforms like Leaf Agriculture, which normalises data from 15+ equipment providers into a unified JSON structure. It is also how FIWARE's NGSI-LD models work: entities have a small set of mandatory attributes and unlimited extensible properties. The ADAPT Standard 1.0 itself uses JSON for non-spatial metadata, recognising that agricultural data varies widely by region, crop type, and equipment manufacturer.

The key insight is that precision agriculture data has a stable structural skeleton (farms contain fields, fields have seasons, seasons have operations) but highly variable flesh (soil probe A reports 3 parameters while probe B reports 12; German regulations require different fields than Brazilian ones; a John Deere combine sends different telemetry than a CLAAS machine). JSONB handles this variability without schema migrations while GIN indexes keep queries fast.

**Best for:** Rapid development with a small team, multi-region deployments with jurisdiction-specific requirements, and platforms that must ingest data from diverse equipment brands without per-brand schema changes.

**Trade-offs:**
- (+) Far fewer tables (~30) than the fully normalized model
- (+) New sensor types, equipment brands, or regional fields require zero schema migrations
- (+) JSONB GIN indexes support fast containment and key-exists queries
- (+) Natural fit for Leaf Agriculture-style provider-agnostic data normalisation
- (+) Easier to evolve MVP to v1.1 without breaking changes
- (-) No database-level type enforcement on JSONB fields (must validate in application layer)
- (-) JSONB columns can become dumping grounds without discipline
- (-) Some analytical queries on nested JSONB require more complex SQL
- (-) Foreign key relationships inside JSONB are not enforced by the database
- (-) Storage overhead: JSONB stores keys with every row

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ADAPT Standard 1.0 | Operation metadata stored as JSON, matching ADAPT's JSON metadata + GeoParquet spatial approach |
| fiboa | Field boundary attributes use fiboa core fields relationally; extensions stored in JSONB |
| OGC SensorThings API | Thing/Datastream structure maintained relationally; Observation results use JSONB for variable payloads |
| FIWARE NGSI-LD | Entity-property model mirrors NGSI-LD's extensible attribute pattern |
| AGROVOC | Reference data URIs stored in JSONB metadata for extensible vocabulary alignment |
| RFC 7946 (GeoJSON) | Geometries in PostGIS columns; JSONB properties map directly to GeoJSON Feature properties |
| ISO 11783 (ISOBUS) | Equipment JSONB properties capture ISOBUS device description XML fields |
| ISO 28258 | Soil sample results stored as JSONB arrays of {property, value, unit, method} objects |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings: {
    --   "subscription_tier": "pro",
    --   "default_units": "metric",
    --   "gdpr_enabled": true,
    --   "carbon_tracking": true,
    --   "locale": "de-DE",
    --   "regional_config": {"tax_id_required": true, "parcel_registry_code": "DE-NRW"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example preferences: {
    --   "locale": "en-US",
    --   "units": "imperial",
    --   "map_style": "satellite",
    --   "notifications": {"email": true, "push": true, "sms": false}
    -- }
    gdpr_consent_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES user_account(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'viewer',
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions: ["fields:read", "fields:write", "prescriptions:approve", "equipment:manage"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_member_org ON organisation_member(organisation_id);
CREATE INDEX idx_org_member_user ON organisation_member(user_id);
```

---

## Farm & Field Management

```sql
CREATE TABLE grower (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata: {
    --   "external_ids": {"leaf_id": "...", "jd_org_id": "..."},
    --   "contact": {"phone": "+1...", "address": "..."},
    --   "certifications": ["organic_eu", "global_gap"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_grower_org ON grower(organisation_id);
CREATE INDEX idx_grower_metadata ON grower USING GIN(metadata);

CREATE TABLE farm (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grower_id       UUID NOT NULL REFERENCES grower(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    country_code    CHAR(2),
    region          TEXT,
    centroid        GEOMETRY(Point, 4326),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties: {
    --   "total_area_ha": 1250.5,
    --   "elevation_m": 340,
    --   "climate_zone": "Cfb",
    --   "water_source": "groundwell",
    --   "parcel_registry_id": "DE-NRW-12345"
    -- }
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
    is_active       BOOLEAN NOT NULL DEFAULT true,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties: {
    --   "determination_method": "gps_survey",        -- fiboa
    --   "source_provider": "john_deere",
    --   "merged_field_id": "uuid-...",               -- Leaf merged field
    --   "soil_texture_dominant": "silt_loam",
    --   "irrigation_type": "center_pivot",
    --   "drainage_tiles": true,
    --   "previous_crops": ["soybean", "corn", "corn"],
    --   "organic_certified": false,
    --   "field_parcel_id": "DE-NRW-67890"            -- regional identifier
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_field_farm ON field(farm_id);
CREATE INDEX idx_field_boundary ON field USING GIST(boundary);
CREATE INDEX idx_field_properties ON field USING GIN(properties);

CREATE TABLE season (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    crop_code       TEXT NOT NULL,            -- AGROVOC short code or common name
    year            INTEGER NOT NULL,
    planting_date   DATE,
    harvest_date    DATE,
    status          TEXT NOT NULL DEFAULT 'planned',
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties: {
    --   "agrovoc_uri": "http://aims.fao.org/aos/agrovoc/c_4932",
    --   "variety": "DKC 62-08",
    --   "seed_treatment": "Poncho/Votivo",
    --   "target_population_seeds_ha": 80000,
    --   "crop_insurance_policy": "MPCI-2026-4567"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_season_field ON season(field_id);
CREATE INDEX idx_season_year ON season(year);
```

---

## IoT Sensors & Observations

```sql
-- Device — a physical sensor station, weather station, or gateway
CREATE TABLE device (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    device_type     TEXT NOT NULL,            -- soil_probe, weather_station, flow_meter, gateway
    location        GEOMETRY(Point, 4326),
    field_id        UUID REFERENCES field(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    hardware        JSONB NOT NULL DEFAULT '{}',
    -- Example hardware: {
    --   "manufacturer": "Davis Instruments",
    --   "model": "Vantage Pro2",
    --   "serial_number": "VP2-12345",
    --   "firmware": "3.80",
    --   "battery_type": "solar",
    --   "connectivity": "lorawan",
    --   "gateway_id": "uuid-..."
    -- }
    capabilities    TEXT[] NOT NULL DEFAULT '{}',  -- ['air_temperature','humidity','rainfall','wind_speed']
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_device_org ON device(organisation_id);
CREATE INDEX idx_device_field ON device(field_id);
CREATE INDEX idx_device_location ON device USING GIST(location);

-- Sensor reading — a timestamped multi-value observation from a device
CREATE TABLE sensor_reading (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id       UUID NOT NULL REFERENCES device(id) ON DELETE CASCADE,
    field_id        UUID REFERENCES field(id),
    observed_at     TIMESTAMPTZ NOT NULL,
    readings        JSONB NOT NULL,
    -- Example readings (soil probe): {
    --   "soil_moisture_pct": 32.5,
    --   "soil_temperature_c": 18.2,
    --   "soil_ec_ms_cm": 0.45,
    --   "depth_cm": 30,
    --   "battery_v": 3.62
    -- }
    -- Example readings (weather station): {
    --   "air_temperature_c": 22.4,
    --   "humidity_pct": 65.3,
    --   "rainfall_mm": 0.0,
    --   "wind_speed_ms": 3.2,
    --   "wind_direction_deg": 225,
    --   "solar_radiation_wm2": 450.0,
    --   "barometric_pressure_hpa": 1013.25,
    --   "dew_point_c": 15.8
    -- }
    quality_flags   JSONB,                   -- {"outlier_detected": false, "gap_filled": false}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reading_device ON sensor_reading(device_id);
CREATE INDEX idx_reading_time ON sensor_reading(observed_at);
CREATE INDEX idx_reading_device_time ON sensor_reading(device_id, observed_at DESC);
CREATE INDEX idx_reading_field ON sensor_reading(field_id);
CREATE INDEX idx_reading_readings ON sensor_reading USING GIN(readings);
```

---

## Field Operations

```sql
CREATE TABLE field_operation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    operation_type  TEXT NOT NULL,            -- planting, application, harvest, tillage, scouting, irrigation
    status          TEXT NOT NULL DEFAULT 'planned',
    planned_date    DATE,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    operator_id     UUID REFERENCES user_account(id),
    equipment_id    UUID REFERENCES equipment(id),
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details (planting): {
    --   "crop": "corn",
    --   "variety": "DKC 62-08",
    --   "population_seeds_ha": 80000,
    --   "row_spacing_cm": 76,
    --   "depth_cm": 5.0,
    --   "seed_treatment": "Poncho/Votivo",
    --   "inputs": [
    --     {"product": "10-34-0 Starter", "rate": 28, "unit": "L/ha"}
    --   ]
    -- }
    -- Example details (harvest): {
    --   "crop": "corn",
    --   "total_yield_kg": 425000,
    --   "yield_per_ha": 12500,
    --   "moisture_pct": 15.2,
    --   "test_weight_kg_hl": 72.5,
    --   "quality_grade": "US No. 2"
    -- }
    -- Example details (application): {
    --   "application_type": "fertiliser",
    --   "inputs": [
    --     {"product": "Urea 46-0-0", "rate": 200, "unit": "kg/ha", "method": "broadcast"},
    --     {"product": "MAP 11-52-0", "rate": 100, "unit": "kg/ha", "method": "banded"}
    --   ],
    --   "weather_at_application": {"wind_kmh": 8, "temp_c": 18, "humidity_pct": 55}
    -- }
    source_provider TEXT,                    -- john_deere, climate_fieldview, manual
    external_id     TEXT,                    -- provider's operation ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_field_op_field ON field_operation(field_id);
CREATE INDEX idx_field_op_season ON field_operation(season_id);
CREATE INDEX idx_field_op_type ON field_operation(operation_type);
CREATE INDEX idx_field_op_details ON field_operation USING GIN(details);

-- Spatial record — georeferenced pass-level data (as-applied, as-planted, yield points)
CREATE TABLE spatial_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id    UUID NOT NULL REFERENCES field_operation(id) ON DELETE CASCADE,
    geometry        GEOMETRY(Point, 4326) NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL,
    values          JSONB NOT NULL,
    -- Example values (as-applied): {"rate_kg_ha": 152.3, "speed_kmh": 9.1, "section_width_m": 27.4}
    -- Example values (yield): {"yield_kg_ha": 12800, "moisture_pct": 14.8, "flow_kg_s": 12.5}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spatial_op ON spatial_record(operation_id);
CREATE INDEX idx_spatial_geom ON spatial_record USING GIST(geometry);
CREATE INDEX idx_spatial_time ON spatial_record(recorded_at);
```

---

## Equipment

```sql
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    equipment_type  TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    specs           JSONB NOT NULL DEFAULT '{}',
    -- Example specs: {
    --   "manufacturer": "John Deere",
    --   "model": "S780",
    --   "year": 2024,
    --   "serial_number": "1H0S780SLN0123456",
    --   "isobus_compatible": true,
    --   "header_width_m": 12.2,
    --   "tank_capacity_bu": 400,
    --   "telematics": {
    --     "provider": "john_deere",
    --     "device_id": "JD-OPS-123",
    --     "last_sync": "2026-05-23T14:30:00Z"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_org ON equipment(organisation_id);
CREATE INDEX idx_equipment_specs ON equipment USING GIN(specs);

-- Equipment telemetry
CREATE TABLE equipment_telemetry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id) ON DELETE CASCADE,
    recorded_at     TIMESTAMPTZ NOT NULL,
    location        GEOMETRY(Point, 4326),
    data            JSONB NOT NULL,
    -- Example data: {
    --   "engine_hours": 4523.5,
    --   "fuel_level_pct": 72,
    --   "speed_kmh": 8.2,
    --   "heading_deg": 195,
    --   "status": "working",
    --   "def_level_pct": 85,
    --   "engine_rpm": 1850,
    --   "hydraulic_pressure_bar": 185
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equip_telem_equip ON equipment_telemetry(equipment_id);
CREATE INDEX idx_equip_telem_time ON equipment_telemetry(recorded_at);
```

---

## Satellite Imagery & Vegetation Indices

```sql
CREATE TABLE satellite_image (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,
    captured_at     TIMESTAMPTZ NOT NULL,
    cloud_cover_pct NUMERIC(5, 2),
    resolution_m    NUMERIC(6, 2),
    bbox            GEOMETRY(Polygon, 4326),
    storage_url     TEXT NOT NULL,
    indices         JSONB NOT NULL DEFAULT '{}',
    -- Example indices: {
    --   "ndvi": {"mean": 0.72, "min": 0.15, "max": 0.88, "std": 0.08, "raster_url": "s3://..."},
    --   "ndre": {"mean": 0.45, "min": 0.10, "max": 0.62, "std": 0.05, "raster_url": "s3://..."},
    --   "evi":  {"mean": 0.58, "min": 0.12, "max": 0.75, "std": 0.07, "raster_url": "s3://..."},
    --   "lai":  {"mean": 3.2, "min": 0.5, "max": 5.1, "std": 0.9, "raster_url": "s3://..."}
    -- }
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata: {
    --   "satellite": "Sentinel-2A",
    --   "product_type": "L2A",
    --   "bands_available": ["B02","B03","B04","B05","B06","B07","B08","B11","B12"],
    --   "processing_level": "2A",
    --   "orbit_number": 12345
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sat_image_field ON satellite_image(field_id);
CREATE INDEX idx_sat_image_time ON satellite_image(captured_at);
CREATE INDEX idx_sat_image_indices ON satellite_image USING GIN(indices);
```

---

## Soil Sampling

```sql
-- Soil sample — a collected and analysed soil sample (flattened from ISO 28258)
CREATE TABLE soil_sample (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    location        GEOMETRY(Point, 4326) NOT NULL,
    sampled_at      DATE NOT NULL,
    sampled_by      UUID REFERENCES user_account(id),
    depth_range     NUMRANGE NOT NULL,       -- e.g. '[0,30)' cm
    lab_id          TEXT,
    results         JSONB NOT NULL DEFAULT '{}',
    -- Example results: {
    --   "pH": {"value": 6.4, "method": "1:1_water"},
    --   "organic_matter_pct": {"value": 3.8, "method": "loss_on_ignition"},
    --   "phosphorus_ppm": {"value": 28, "method": "bray_p1"},
    --   "potassium_ppm": {"value": 185, "method": "ammonium_acetate"},
    --   "nitrogen_total_pct": {"value": 0.18},
    --   "cec_meq_100g": {"value": 18.5},
    --   "soil_texture": {"sand_pct": 35, "silt_pct": 40, "clay_pct": 25, "class": "loam"},
    --   "organic_carbon_pct": {"value": 2.2},
    --   "electrical_conductivity_ds_m": {"value": 0.45}
    -- }
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata: {
    --   "lab_name": "Midwest Laboratories",
    --   "sample_type": "composite",
    --   "cores_per_sample": 15,
    --   "zone_id": "uuid-..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_soil_sample_field ON soil_sample(field_id);
CREATE INDEX idx_soil_sample_location ON soil_sample USING GIST(location);
CREATE INDEX idx_soil_sample_results ON soil_sample USING GIN(results);
```

---

## Prescriptions & Management Zones

```sql
CREATE TABLE management_zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    name            TEXT NOT NULL,
    zone_number     INTEGER NOT NULL,
    geometry        GEOMETRY(Polygon, 4326) NOT NULL,
    area_ha         NUMERIC(12, 4),
    creation_method TEXT,
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example attributes: {
    --   "mean_ndvi": 0.68,
    --   "mean_yield_kg_ha": 11200,
    --   "soil_om_pct": 3.5,
    --   "dominant_soil_type": "silt_loam",
    --   "ec_ms_cm": 0.42,
    --   "cluster_label": "high_productivity"
    -- }
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
    -- Example config: {
    --   "product": "Urea 46-0-0",
    --   "product_id": "uuid-...",
    --   "rate_unit": "kg/ha",
    --   "min_rate": 100,
    --   "max_rate": 300,
    --   "export_format": "shapefile",
    --   "algorithm": "linear_interpolation",
    --   "data_layers_used": ["yield_2025", "soil_om", "ndvi_2026_03"]
    -- }
    zones           JSONB NOT NULL DEFAULT '[]',
    -- Example zones: [
    --   {"zone_id": "uuid-...", "target_rate": 250, "geometry": null},
    --   {"zone_id": "uuid-...", "target_rate": 180, "geometry": null},
    --   {"zone_id": null, "target_rate": 150, "geometry": {"type":"Polygon","coordinates":[...]}}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_prescription_field ON prescription(field_id);
```

---

## Scouting & Alerts

```sql
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
    -- Example findings (pest): {
    --   "pest_name": "corn rootworm",
    --   "pest_agrovoc_uri": "http://aims.fao.org/aos/agrovoc/c_...",
    --   "lifecycle_stage": "adult",
    --   "count_per_trap": 15,
    --   "affected_area_pct": 30,
    --   "growth_stage_bbch": "65"
    -- }
    -- Example findings (disease): {
    --   "disease_name": "gray leaf spot",
    --   "pathogen": "Cercospora zeae-maydis",
    --   "incidence_pct": 45,
    --   "severity_scale_1_9": 4,
    --   "leaf_position": "lower_canopy"
    -- }
    photos          JSONB NOT NULL DEFAULT '[]',
    -- Example photos: [
    --   {"url": "s3://...", "ai_classification": {"label": "gray_leaf_spot", "confidence": 0.89}}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scouting_field ON scouting_observation(field_id);
CREATE INDEX idx_scouting_location ON scouting_observation USING GIST(location);
CREATE INDEX idx_scouting_findings ON scouting_observation USING GIN(findings);

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
    -- Example context: {
    --   "trigger": "ndvi_drop",
    --   "current_ndvi": 0.45,
    --   "previous_ndvi": 0.72,
    --   "drop_pct": 37.5,
    --   "image_id": "uuid-...",
    --   "recommended_action": "scout_immediately"
    -- }
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES user_account(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_org ON alert(organisation_id);
CREATE INDEX idx_alert_created ON alert(created_at DESC);
```

---

## Carbon & Sustainability

```sql
CREATE TABLE carbon_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    assessment_year INTEGER NOT NULL,
    methodology     TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    summary         JSONB NOT NULL DEFAULT '{}',
    -- Example summary: {
    --   "total_co2e_tonnes": -12.5,
    --   "line_items": [
    --     {"category": "soil_carbon_change", "co2e_tonnes": -18.0, "source": "modelled"},
    --     {"category": "fertiliser_emissions", "co2e_tonnes": 3.2, "source": "calculated"},
    --     {"category": "fuel_emissions", "co2e_tonnes": 1.8, "source": "measured"},
    --     {"category": "tillage_reduction_credit", "co2e_tonnes": -0.5, "source": "default_factor"}
    --   ],
    --   "verifier": "Verra",
    --   "verified_at": "2026-09-15",
    --   "certificate_url": "https://..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carbon_field ON carbon_assessment(field_id);
```

---

## External Provider Integration

```sql
CREATE TABLE provider_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    credentials     JSONB NOT NULL DEFAULT '{}',  -- encrypted at application layer
    -- Example credentials: {
    --   "access_token": "enc:...",
    --   "refresh_token": "enc:...",
    --   "expires_at": "2026-06-01T00:00:00Z",
    --   "scopes": ["read:fields", "read:operations", "write:prescriptions"],
    --   "external_org_id": "JD-ORG-456"
    -- }
    sync_state      JSONB NOT NULL DEFAULT '{}',
    -- Example sync_state: {
    --   "last_sync_at": "2026-05-23T14:30:00Z",
    --   "last_sync_status": "success",
    --   "fields_synced": 42,
    --   "operations_synced": 156,
    --   "next_cursor": "abc123"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, provider)
);

CREATE INDEX idx_provider_conn_org ON provider_connection(organisation_id);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES user_account(id),
    organisation_id UUID REFERENCES organisation(id),
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    changes         JSONB,
    -- Example changes: {
    --   "before": {"status": "draft"},
    --   "after": {"status": "approved", "approved_by": "uuid-..."},
    --   "diff_keys": ["status", "approved_by"]
    -- }
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Example Queries

### Find all soil samples with high phosphorus in a field
```sql
SELECT id, sampled_at, location,
       (results->'phosphorus_ppm'->>'value')::numeric AS phosphorus_ppm
FROM soil_sample
WHERE field_id = $1
  AND (results->'phosphorus_ppm'->>'value')::numeric > 50
ORDER BY sampled_at DESC;
```

### Get latest sensor readings with specific properties
```sql
SELECT DISTINCT ON (device_id)
       device_id, observed_at, readings
FROM sensor_reading
WHERE field_id = $1
  AND readings ? 'soil_moisture_pct'   -- key exists
ORDER BY device_id, observed_at DESC;
```

### Find fields with specific properties across an organisation
```sql
SELECT f.id, f.name, f.area_ha, f.properties
FROM field f
JOIN farm fa ON f.farm_id = fa.id
JOIN grower g ON fa.grower_id = g.id
WHERE g.organisation_id = $1
  AND f.properties @> '{"irrigation_type": "center_pivot"}'::jsonb
  AND f.is_active = true;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organisation, user_account, organisation_member |
| Farm & Field Management | 4 | grower, farm, field, season |
| IoT Sensors | 2 | device, sensor_reading |
| Field Operations | 2 | field_operation, spatial_record |
| Equipment | 2 | equipment, equipment_telemetry |
| Satellite Imagery | 1 | satellite_image (indices embedded as JSONB) |
| Soil Sampling | 1 | soil_sample (flattened from ISO 28258 with JSONB results) |
| Prescriptions & Zones | 2 | management_zone, prescription (zone rates embedded as JSONB) |
| Scouting & Alerts | 2 | scouting_observation, alert |
| Carbon & Sustainability | 1 | carbon_assessment (line items embedded as JSONB) |
| External Integration | 1 | provider_connection |
| Audit | 1 | audit_log |
| **Total** | **22** | |

---

## Key Design Decisions

1. **JSONB for variable-payload data** — sensor readings, operation details, soil analysis results, and equipment specs use JSONB because the schema varies by device type, equipment brand, crop, and jurisdiction. This eliminates the need for per-type tables or wide sparse-column tables.

2. **GIN indexes on all JSONB columns** — ensures that containment queries (`@>`), key-exists queries (`?`), and path queries remain fast even as data volume grows.

3. **Flattened soil model** — instead of replicating ISO 28258's 5-level hierarchy (Site > Profile > ProfileElement > Specimen > Result), a single `soil_sample` table with a `depth_range` column and JSONB `results` captures the same information with dramatically simpler queries. The full ISO structure can be reconstructed for export.

4. **Embedded prescription zones** — prescription zone rates are stored as a JSONB array within the prescription table, avoiding a junction table. This works well because zones are always loaded and saved together with the prescription, and individual zone-rate queries are rare.

5. **Unified sensor_reading table** — all sensor types (soil probes, weather stations, flow meters) write to a single table with device-specific readings in JSONB. This avoids a separate table per sensor type and matches the Leaf Agriculture pattern of normalising diverse data into a common structure.

6. **NUMRANGE for soil depth** — PostgreSQL's range type cleanly represents depth intervals (e.g., `[0,30)` cm) and supports overlap and containment queries for finding samples at specific depths.

7. **Properties on core entities** — field, farm, grower, and season each have a `properties` JSONB column for metadata that varies by region, certification, or regulatory context, avoiding region-specific columns.

8. **Audit log with before/after diffs** — the JSONB `changes` column captures what changed, enabling a single-table audit trail without triggers on every table.

9. **22 tables vs ~45 in normalized model** — roughly half the table count, achieved by consolidating reference data, observation results, and configuration into JSONB columns. This reduces migration complexity and makes the API layer thinner.

10. **Application-layer validation required** — the trade-off for JSONB flexibility is that data quality rules (e.g., "soil pH must be between 0 and 14") must be enforced in application code or database CHECK constraints on JSONB paths, not through column types.
