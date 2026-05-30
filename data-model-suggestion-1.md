# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Precision Agriculture Platform · Created: 2026-05-24

## Philosophy

This model follows the classic normalized relational approach where every domain concept receives its own table, relationships are expressed through foreign keys and junction tables, and data integrity is enforced at the database level. The schema is designed to align directly with established precision agriculture standards: ISO 28258 for soil data, OGC SensorThings API for IoT sensor structure, ADAPT Standard 1.0 for field operations interchange, fiboa for field boundaries, and AGROVOC for agricultural vocabulary reference.

The normalized approach ensures that every entity has a single authoritative location, eliminating data duplication and making cross-entity queries straightforward. This is the model that traditional agronomic software companies (SST Software, Trimble Agriculture) have historically used, and it maps naturally to the resource structures exposed by APIs like John Deere Operations Center and Leaf Agriculture.

The trade-off is a higher table count and more complex joins for queries that span multiple domain areas. However, for a platform that must integrate with multiple external systems and maintain regulatory audit trails for carbon credit verification and GDPR compliance, the explicitness of a fully normalized schema provides the clearest data lineage.

**Best for:** Enterprise deployments where data integrity, regulatory compliance, and multi-system integration are paramount.

**Trade-offs:**
- (+) Maximum data integrity through foreign key constraints
- (+) Direct alignment with ISO/OGC/ADAPT standards simplifies data interchange
- (+) Clear audit trail through explicit relationship chains
- (+) Well-understood by database teams; excellent tooling support
- (-) High table count (~55-65 tables) increases migration complexity
- (-) Complex joins for cross-domain queries (e.g., "all observations for a field's current crop")
- (-) Adding new sensor types or observation properties requires schema migrations
- (-) Less flexible for jurisdiction-specific or experimental data fields

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 28258 (Soil Data Model) | Soil tables mirror the Site > Plot > Profile > ProfileElement > SoilSpecimen hierarchy |
| OGC SensorThings API | IoT tables follow Thing > Location > Datastream > Observation entity structure |
| ADAPT Standard 1.0 | Field operations tables align with ADAPT's Farm > Field > Operation > SpatialRecord concepts |
| fiboa | Field boundary table schema matches fiboa core attributes (id, geometry, area, determination_method) |
| AGROVOC (FAO) | Reference tables for crops, pests, growth stages use AGROVOC URIs as canonical identifiers |
| ISO 3166 | Jurisdiction/country codes stored as ISO 3166-1 alpha-2 |
| RFC 7946 (GeoJSON) | All geometry columns stored as PostGIS geometry types, exportable as GeoJSON |
| ISO 11783 (ISOBUS) | Equipment and implement tables model ISOBUS device description structures |
| GDPR (EU 2016/679) | Consent and data access log tables support GDPR compliance workflows |
| OAuth 2.0 (RFC 6749) | API credential and token tables support OAuth 2.0 provider pass-through |

---

## Core Identity & Multi-Tenancy

```sql
-- Organisation (tenant) — top-level account
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,  -- ISO 3166-1 alpha-2
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- User account
CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,                    -- null for SSO-only users
    locale          TEXT NOT NULL DEFAULT 'en',
    gdpr_consent_at TIMESTAMPTZ,             -- GDPR consent timestamp
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Organisation membership with role
CREATE TABLE organisation_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES user_account(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'viewer',  -- owner, admin, agronomist, operator, viewer
    invited_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    accepted_at     TIMESTAMPTZ,
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_member_org ON organisation_member(organisation_id);
CREATE INDEX idx_org_member_user ON organisation_member(user_id);
```

---

## Farm & Field Management

```sql
-- Grower / client — the farming entity (may differ from platform user)
CREATE TABLE grower (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    external_ids    JSONB NOT NULL DEFAULT '{}',  -- e.g. {"leaf_grower_id": "...", "jd_org_id": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_grower_org ON grower(organisation_id);

-- Farm — a named grouping of fields
CREATE TABLE farm (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grower_id       UUID NOT NULL REFERENCES grower(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    country_code    CHAR(2),                -- ISO 3166-1
    region          TEXT,                   -- state/province
    centroid        GEOMETRY(Point, 4326),  -- approximate centre
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_farm_grower ON farm(grower_id);
CREATE INDEX idx_farm_centroid ON farm USING GIST(centroid);

-- Field — an individual agricultural parcel (fiboa-aligned)
CREATE TABLE field (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    farm_id         UUID NOT NULL REFERENCES farm(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    area_ha         NUMERIC(12, 4),              -- hectares
    boundary        GEOMETRY(Polygon, 4326) NOT NULL,  -- fiboa: geometry
    determination_method TEXT,                    -- fiboa: how boundary was determined
    source_provider TEXT,                         -- e.g. 'john_deere', 'climate_fieldview', 'manual'
    merged_field_id UUID,                         -- Leaf-style merged field reference
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_field_farm ON field(farm_id);
CREATE INDEX idx_field_boundary ON field USING GIST(boundary);

-- Field boundary history — track boundary changes over time
CREATE TABLE field_boundary_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    boundary        GEOMETRY(Polygon, 4326) NOT NULL,
    area_ha         NUMERIC(12, 4),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    source          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_field_boundary_ver_field ON field_boundary_version(field_id);

-- Season — a crop growing season for a field
CREATE TABLE season (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    crop_id         UUID NOT NULL REFERENCES crop(id),
    year            INTEGER NOT NULL,
    planting_date   DATE,
    harvest_date    DATE,
    status          TEXT NOT NULL DEFAULT 'planned', -- planned, active, harvested
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_season_field ON season(field_id);
CREATE INDEX idx_season_year ON season(year);
```

---

## Reference Data (AGROVOC-Aligned)

```sql
-- Crop type reference
CREATE TABLE crop (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agrovoc_uri     TEXT UNIQUE,             -- e.g. 'http://aims.fao.org/aos/agrovoc/c_4932' (maize)
    common_name     TEXT NOT NULL,
    scientific_name TEXT,
    crop_group      TEXT,                    -- cereals, oilseeds, pulses, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Pest / disease reference
CREATE TABLE pest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agrovoc_uri     TEXT UNIQUE,
    common_name     TEXT NOT NULL,
    scientific_name TEXT,
    pest_type       TEXT NOT NULL,           -- insect, fungus, bacteria, virus, weed, nematode
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Growth stage reference (BBCH scale)
CREATE TABLE growth_stage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bbch_code       TEXT NOT NULL UNIQUE,    -- e.g. '13' (3 leaves unfolded)
    description     TEXT NOT NULL,
    principal_stage TEXT NOT NULL,            -- germination, leaf_development, tillering, etc.
    agrovoc_uri     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Soil type reference
CREATE TABLE soil_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    usda_class      TEXT,                    -- clay, silt, sand, loam, etc.
    fao_class       TEXT,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Input product reference (fertilisers, pesticides, seeds)
CREATE TABLE input_product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    product_type    TEXT NOT NULL,            -- fertiliser, herbicide, insecticide, fungicide, seed
    manufacturer    TEXT,
    active_ingredient TEXT,
    unit_of_measure TEXT NOT NULL,            -- kg, L, seeds
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## IoT Sensors (OGC SensorThings-Aligned)

```sql
-- Thing — a physical IoT device (sensor station, weather station, soil probe)
CREATE TABLE iot_thing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    thing_type      TEXT NOT NULL,            -- soil_probe, weather_station, flow_meter, drone, gateway
    manufacturer    TEXT,
    model           TEXT,
    serial_number   TEXT,
    firmware_version TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    properties      JSONB NOT NULL DEFAULT '{}',  -- manufacturer-specific metadata
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_iot_thing_org ON iot_thing(organisation_id);

-- Thing location (current and historical)
CREATE TABLE iot_thing_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thing_id        UUID NOT NULL REFERENCES iot_thing(id) ON DELETE CASCADE,
    location        GEOMETRY(Point, 4326) NOT NULL,
    field_id        UUID REFERENCES field(id),
    installed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    removed_at      TIMESTAMPTZ,
    description     TEXT
);

CREATE INDEX idx_iot_thing_loc_thing ON iot_thing_location(thing_id);
CREATE INDEX idx_iot_thing_loc_geom ON iot_thing_location USING GIST(location);

-- Sensor — a sensing capability on a Thing
CREATE TABLE iot_sensor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thing_id        UUID NOT NULL REFERENCES iot_thing(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    sensor_type     TEXT NOT NULL,            -- thermometer, hygrometer, tensiometer, rain_gauge, etc.
    encoding_type   TEXT NOT NULL DEFAULT 'application/json',
    metadata_url    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_iot_sensor_thing ON iot_sensor(thing_id);

-- Observed property — what the sensor measures
CREATE TABLE observed_property (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,     -- soil_moisture, air_temperature, rainfall, ndvi, etc.
    description     TEXT,
    unit_of_measurement TEXT NOT NULL,        -- %, °C, mm, index
    agrovoc_uri     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Datastream — links a Sensor to an ObservedProperty on a Thing
CREATE TABLE iot_datastream (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL REFERENCES iot_sensor(id) ON DELETE CASCADE,
    observed_property_id UUID NOT NULL REFERENCES observed_property(id),
    field_id        UUID REFERENCES field(id),
    name            TEXT NOT NULL,
    observation_type TEXT NOT NULL DEFAULT 'measurement',
    unit_of_measurement TEXT NOT NULL,
    observed_area   GEOMETRY(Polygon, 4326),
    phenomenon_time_start TIMESTAMPTZ,
    phenomenon_time_end   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_iot_datastream_sensor ON iot_datastream(sensor_id);
CREATE INDEX idx_iot_datastream_field ON iot_datastream(field_id);

-- Observation — individual sensor reading
CREATE TABLE iot_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    datastream_id   UUID NOT NULL REFERENCES iot_datastream(id) ON DELETE CASCADE,
    phenomenon_time TIMESTAMPTZ NOT NULL,
    result_time     TIMESTAMPTZ NOT NULL DEFAULT now(),
    result_value    NUMERIC NOT NULL,
    result_quality  TEXT,
    valid_time_start TIMESTAMPTZ,
    valid_time_end  TIMESTAMPTZ,
    parameters      JSONB,                   -- additional observation context
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_iot_obs_datastream ON iot_observation(datastream_id);
CREATE INDEX idx_iot_obs_time ON iot_observation(phenomenon_time);
CREATE INDEX idx_iot_obs_datastream_time ON iot_observation(datastream_id, phenomenon_time DESC);
```

---

## Soil Sampling (ISO 28258-Aligned)

```sql
-- Soil site — the investigation context
CREATE TABLE soil_site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    name            TEXT,
    land_use        TEXT,
    terrain_description TEXT,
    sampled_at      DATE NOT NULL,
    sampled_by      UUID REFERENCES user_account(id),
    location        GEOMETRY(Point, 4326),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_soil_site_field ON soil_site(field_id);

-- Soil profile — vertical column at a site
CREATE TABLE soil_profile (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES soil_site(id) ON DELETE CASCADE,
    profile_type    TEXT NOT NULL DEFAULT 'trial_pit',  -- trial_pit, borehole, surface
    total_depth_cm  NUMERIC(6, 2),
    location        GEOMETRY(Point, 4326),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_soil_profile_site ON soil_profile(site_id);

-- Soil profile element — horizon or layer within a profile
CREATE TABLE soil_profile_element (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id      UUID NOT NULL REFERENCES soil_profile(id) ON DELETE CASCADE,
    element_type    TEXT NOT NULL,            -- horizon, layer
    upper_depth_cm  NUMERIC(6, 2) NOT NULL,
    lower_depth_cm  NUMERIC(6, 2) NOT NULL,
    designation     TEXT,                    -- e.g. 'Ap', 'Bt', 'C'
    soil_type_id    UUID REFERENCES soil_type(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_soil_element_profile ON soil_profile_element(profile_id);

-- Soil specimen — a collected sample from a profile element
CREATE TABLE soil_specimen (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_element_id UUID NOT NULL REFERENCES soil_profile_element(id) ON DELETE CASCADE,
    lab_id          TEXT,                     -- external lab sample identifier
    collected_at    TIMESTAMPTZ NOT NULL,
    storage_method  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Soil analysis result — lab or in-field measurement
CREATE TABLE soil_analysis_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    specimen_id     UUID NOT NULL REFERENCES soil_specimen(id) ON DELETE CASCADE,
    property_name   TEXT NOT NULL,            -- pH, organic_carbon, nitrogen, phosphorus, potassium, etc.
    value           NUMERIC,
    unit            TEXT NOT NULL,
    method          TEXT,                     -- analysis method
    analysed_at     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_soil_result_specimen ON soil_analysis_result(specimen_id);
CREATE INDEX idx_soil_result_property ON soil_analysis_result(property_name);
```

---

## Field Operations (ADAPT-Aligned)

```sql
-- Field operation — a planting, application, harvest, or tillage event
CREATE TABLE field_operation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id) ON DELETE CASCADE,
    field_id        UUID NOT NULL REFERENCES field(id),
    operation_type  TEXT NOT NULL,            -- planting, application, harvest, tillage, scouting
    status          TEXT NOT NULL DEFAULT 'planned', -- planned, in_progress, completed, cancelled
    planned_date    DATE,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    operator_id     UUID REFERENCES user_account(id),
    equipment_id    UUID REFERENCES equipment(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_field_op_season ON field_operation(season_id);
CREATE INDEX idx_field_op_field ON field_operation(field_id);
CREATE INDEX idx_field_op_type ON field_operation(operation_type);

-- Operation input — product applied during an operation
CREATE TABLE operation_input (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id    UUID NOT NULL REFERENCES field_operation(id) ON DELETE CASCADE,
    input_product_id UUID NOT NULL REFERENCES input_product(id),
    target_rate     NUMERIC(12, 4),          -- planned rate
    actual_rate     NUMERIC(12, 4),          -- as-applied rate
    rate_unit       TEXT NOT NULL,            -- kg/ha, L/ha, seeds/ha
    total_quantity  NUMERIC(12, 4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_op_input_operation ON operation_input(operation_id);

-- Harvest result — yield data from a harvest operation
CREATE TABLE harvest_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id    UUID NOT NULL REFERENCES field_operation(id) ON DELETE CASCADE,
    crop_id         UUID NOT NULL REFERENCES crop(id),
    total_yield_kg  NUMERIC(14, 2),
    yield_per_ha    NUMERIC(10, 2),
    moisture_pct    NUMERIC(5, 2),
    test_weight     NUMERIC(8, 2),
    quality_grade   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Spatial record — georeferenced as-applied or as-planted data (ADAPT spatial records)
CREATE TABLE spatial_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id    UUID NOT NULL REFERENCES field_operation(id) ON DELETE CASCADE,
    geometry        GEOMETRY(Point, 4326) NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    values          JSONB NOT NULL,          -- {"rate": 150.2, "speed_kmh": 8.5, "depth_cm": 5.0}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spatial_record_op ON spatial_record(operation_id);
CREATE INDEX idx_spatial_record_geom ON spatial_record USING GIST(geometry);
CREATE INDEX idx_spatial_record_time ON spatial_record(timestamp);
```

---

## Equipment (ISOBUS-Aligned)

```sql
-- Equipment — tractor, sprayer, planter, combine, drone
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    equipment_type  TEXT NOT NULL,            -- tractor, sprayer, planter, combine, drone, irrigation_system
    manufacturer    TEXT,
    model           TEXT,
    serial_number   TEXT,
    year            INTEGER,
    isobus_compatible BOOLEAN NOT NULL DEFAULT false,
    telematics_provider TEXT,                -- john_deere, cnh, trimble, etc.
    telematics_device_id TEXT,               -- external device identifier
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_org ON equipment(organisation_id);

-- Equipment telemetry snapshot
CREATE TABLE equipment_telemetry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id) ON DELETE CASCADE,
    timestamp       TIMESTAMPTZ NOT NULL,
    location        GEOMETRY(Point, 4326),
    engine_hours    NUMERIC(10, 1),
    fuel_level_pct  NUMERIC(5, 2),
    speed_kmh       NUMERIC(6, 2),
    heading_deg     NUMERIC(5, 2),
    status          TEXT,                    -- idle, moving, working
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equip_telem_equip ON equipment_telemetry(equipment_id);
CREATE INDEX idx_equip_telem_time ON equipment_telemetry(timestamp);
```

---

## Satellite Imagery & Vegetation Indices

```sql
-- Satellite image capture
CREATE TABLE satellite_image (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,            -- sentinel_2, landsat_8, planet, drone
    captured_at     TIMESTAMPTZ NOT NULL,
    cloud_cover_pct NUMERIC(5, 2),
    resolution_m    NUMERIC(6, 2),
    bands_available TEXT[],                   -- {'B02','B03','B04','B08','B11','B12'}
    bbox            GEOMETRY(Polygon, 4326),
    storage_url     TEXT NOT NULL,            -- object storage path
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sat_image_field ON satellite_image(field_id);
CREATE INDEX idx_sat_image_time ON satellite_image(captured_at);

-- Vegetation index computation
CREATE TABLE vegetation_index (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    image_id        UUID NOT NULL REFERENCES satellite_image(id) ON DELETE CASCADE,
    index_type      TEXT NOT NULL,            -- ndvi, ndre, evi, lai, savi, msavi
    mean_value      NUMERIC(8, 6),
    min_value       NUMERIC(8, 6),
    max_value       NUMERIC(8, 6),
    std_dev         NUMERIC(8, 6),
    raster_url      TEXT,                    -- path to computed raster
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_veg_index_image ON vegetation_index(image_id);
CREATE INDEX idx_veg_index_type ON vegetation_index(index_type);
```

---

## Prescriptions & Management Zones

```sql
-- Management zone — a sub-field area with uniform treatment
CREATE TABLE management_zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    name            TEXT NOT NULL,
    zone_number     INTEGER NOT NULL,
    geometry        GEOMETRY(Polygon, 4326) NOT NULL,
    area_ha         NUMERIC(12, 4),
    method          TEXT,                    -- kmeans, manual, yield_based, soil_ec
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mgmt_zone_field ON management_zone(field_id);
CREATE INDEX idx_mgmt_zone_geom ON management_zone USING GIST(geometry);

-- Prescription — variable rate application plan
CREATE TABLE prescription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    name            TEXT NOT NULL,
    prescription_type TEXT NOT NULL,          -- seeding, fertiliser, herbicide, irrigation
    input_product_id UUID REFERENCES input_product(id),
    status          TEXT NOT NULL DEFAULT 'draft', -- draft, approved, exported, applied
    created_by      UUID REFERENCES user_account(id),
    approved_by     UUID REFERENCES user_account(id),
    approved_at     TIMESTAMPTZ,
    export_format   TEXT,                    -- shapefile, isoxml, adapt_geoparquet
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_prescription_field ON prescription(field_id);

-- Prescription zone rate — rate per management zone
CREATE TABLE prescription_zone_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prescription_id UUID NOT NULL REFERENCES prescription(id) ON DELETE CASCADE,
    zone_id         UUID REFERENCES management_zone(id),
    geometry        GEOMETRY(Polygon, 4326) NOT NULL,  -- zone geometry (may differ from mgmt zone)
    target_rate     NUMERIC(12, 4) NOT NULL,
    rate_unit       TEXT NOT NULL,            -- kg/ha, L/ha, seeds/ha
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_presc_zone_presc ON prescription_zone_rate(prescription_id);
CREATE INDEX idx_presc_zone_geom ON prescription_zone_rate USING GIST(geometry);
```

---

## Scouting & Alerts

```sql
-- Scouting observation — field visit finding
CREATE TABLE scouting_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    observer_id     UUID NOT NULL REFERENCES user_account(id),
    observed_at     TIMESTAMPTZ NOT NULL,
    location        GEOMETRY(Point, 4326) NOT NULL,
    observation_type TEXT NOT NULL,           -- pest, disease, weed, nutrient_deficiency, growth_stage, general
    pest_id         UUID REFERENCES pest(id),
    growth_stage_id UUID REFERENCES growth_stage(id),
    severity        TEXT,                    -- low, medium, high, critical
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scouting_field ON scouting_observation(field_id);
CREATE INDEX idx_scouting_location ON scouting_observation USING GIST(location);

-- Scouting photo
CREATE TABLE scouting_photo (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    observation_id  UUID NOT NULL REFERENCES scouting_observation(id) ON DELETE CASCADE,
    storage_url     TEXT NOT NULL,
    ai_classification JSONB,                 -- {"pest": "aphid", "confidence": 0.92, "model": "v3.1"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Alert — system-generated notification
CREATE TABLE alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    field_id        UUID REFERENCES field(id),
    alert_type      TEXT NOT NULL,            -- pest_detected, frost_warning, ndvi_drop, sensor_offline, irrigation_needed
    severity        TEXT NOT NULL,            -- info, warning, critical
    title           TEXT NOT NULL,
    message         TEXT NOT NULL,
    source          TEXT NOT NULL,            -- ai_model, sensor_threshold, weather_forecast, satellite
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES user_account(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_org ON alert(organisation_id);
CREATE INDEX idx_alert_field ON alert(field_id);
CREATE INDEX idx_alert_created ON alert(created_at DESC);
```

---

## Weather Data

```sql
-- Weather station (may be an IoT thing or external source)
CREATE TABLE weather_station (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisation(id),
    thing_id        UUID REFERENCES iot_thing(id),  -- null if external data source
    name            TEXT NOT NULL,
    location        GEOMETRY(Point, 4326) NOT NULL,
    source          TEXT NOT NULL,            -- on_farm, tomorrow_io, open_meteo, national_weather_service
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_weather_station_loc ON weather_station USING GIST(location);

-- Weather observation — hourly or sub-hourly reading
CREATE TABLE weather_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    station_id      UUID NOT NULL REFERENCES weather_station(id) ON DELETE CASCADE,
    observed_at     TIMESTAMPTZ NOT NULL,
    air_temp_c      NUMERIC(5, 2),
    humidity_pct    NUMERIC(5, 2),
    wind_speed_ms   NUMERIC(6, 2),
    wind_direction_deg NUMERIC(5, 2),
    rainfall_mm     NUMERIC(8, 2),
    solar_radiation_wm2 NUMERIC(8, 2),
    soil_temp_c     NUMERIC(5, 2),
    dew_point_c     NUMERIC(5, 2),
    pressure_hpa    NUMERIC(7, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_weather_obs_station ON weather_observation(station_id);
CREATE INDEX idx_weather_obs_time ON weather_observation(observed_at);
```

---

## Carbon & Sustainability

```sql
-- Carbon assessment period
CREATE TABLE carbon_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    season_id       UUID REFERENCES season(id),
    assessment_year INTEGER NOT NULL,
    methodology     TEXT NOT NULL,            -- verra_vm0042, gold_standard, national
    status          TEXT NOT NULL DEFAULT 'draft', -- draft, submitted, verified, issued
    verifier        TEXT,
    verified_at     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carbon_assess_field ON carbon_assessment(field_id);

-- Carbon emission / sequestration line item
CREATE TABLE carbon_line_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id   UUID NOT NULL REFERENCES carbon_assessment(id) ON DELETE CASCADE,
    category        TEXT NOT NULL,            -- soil_carbon_change, fertiliser_emissions, fuel_emissions, tillage_reduction
    co2e_tonnes     NUMERIC(12, 4) NOT NULL,  -- positive = emission, negative = sequestration
    data_source     TEXT NOT NULL,            -- measured, modelled, default_factor
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carbon_line_assess ON carbon_line_item(assessment_id);
```

---

## Audit & GDPR Compliance

```sql
-- Data access log — GDPR-required access tracking
CREATE TABLE data_access_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES user_account(id),
    organisation_id UUID REFERENCES organisation(id),
    action          TEXT NOT NULL,            -- read, create, update, delete, export, share
    resource_type   TEXT NOT NULL,            -- field, observation, prescription, etc.
    resource_id     UUID,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_access_log_user ON data_access_log(user_id);
CREATE INDEX idx_access_log_time ON data_access_log(created_at);
CREATE INDEX idx_access_log_resource ON data_access_log(resource_type, resource_id);

-- GDPR data deletion request
CREATE TABLE gdpr_deletion_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES user_account(id),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'pending', -- pending, processing, completed, rejected
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## External Provider Integration

```sql
-- External provider connection (OAuth tokens for JD, FieldView, Trimble, etc.)
CREATE TABLE provider_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,            -- john_deere, climate_fieldview, trimble, cnh, leaf
    access_token    TEXT,                     -- encrypted at rest
    refresh_token   TEXT,                     -- encrypted at rest
    token_expires_at TIMESTAMPTZ,
    scopes          TEXT[],
    external_org_id TEXT,
    status          TEXT NOT NULL DEFAULT 'active', -- active, expired, revoked
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_provider_conn_org ON provider_connection(organisation_id);
CREATE UNIQUE INDEX idx_provider_conn_unique ON provider_connection(organisation_id, provider);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organisation, user_account, organisation_member |
| Farm & Field Management | 5 | grower, farm, field, field_boundary_version, season |
| Reference Data | 5 | crop, pest, growth_stage, soil_type, input_product |
| IoT Sensors | 6 | iot_thing, iot_thing_location, iot_sensor, observed_property, iot_datastream, iot_observation |
| Soil Sampling | 5 | soil_site, soil_profile, soil_profile_element, soil_specimen, soil_analysis_result |
| Field Operations | 4 | field_operation, operation_input, harvest_result, spatial_record |
| Equipment | 2 | equipment, equipment_telemetry |
| Satellite Imagery | 2 | satellite_image, vegetation_index |
| Prescriptions & Zones | 3 | management_zone, prescription, prescription_zone_rate |
| Scouting & Alerts | 3 | scouting_observation, scouting_photo, alert |
| Weather | 2 | weather_station, weather_observation |
| Carbon & Sustainability | 2 | carbon_assessment, carbon_line_item |
| Audit & GDPR | 2 | data_access_log, gdpr_deletion_request |
| External Integration | 1 | provider_connection |
| **Total** | **45** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation, safe cross-system references, and compatibility with Leaf Agriculture's merged field ID pattern.

2. **PostGIS geometry columns with SRID 4326** — all geospatial data stored as WGS 84 geometries, directly exportable as GeoJSON per RFC 7946 and compatible with fiboa and ADAPT Standard requirements.

3. **OGC SensorThings entity hierarchy** — IoT tables follow the Thing > Location > Sensor > Datastream > Observation chain, enabling standards-compliant API exposure and interoperability with existing SensorThings-compatible tools.

4. **ISO 28258 soil data hierarchy** — soil tables implement Site > Profile > ProfileElement > Specimen > Result, ensuring soil data can be exported in ISO-compliant formats for regulatory or research purposes.

5. **AGROVOC URIs as reference data identifiers** — crops, pests, and growth stages reference FAO AGROVOC URIs, enabling multilingual support and interoperability with international agricultural data systems.

6. **Separate spatial_record table for as-applied data** — aligns with ADAPT Standard 1.0's spatial records concept, storing point-level georeferenced data from field passes for detailed as-applied analysis.

7. **Field boundary versioning** — a dedicated history table tracks boundary changes over time, supporting temporal queries and regulatory audit requirements without complicating the active field table.

8. **Carbon assessment as first-class entity** — dedicated tables for MRV (measurement, reporting, verification) workflows support the platform's sustainability tracking feature and carbon credit verification use case.

9. **Provider connection table with encrypted tokens** — centralises OAuth 2.0 credential management for multi-provider integration (John Deere, Climate FieldView, Trimble, CNH, Leaf Agriculture).

10. **GDPR compliance tables** — explicit data access logging and deletion request tracking tables satisfy EU data protection requirements for agricultural data platforms operating in European markets.
