# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Smart Building Platform · Created: 2026-05-22

## Philosophy

This model applies rigorous third-normal-form relational design to every concept in the smart building domain. Every entity type — site, building, floor, zone, equipment, sensor, point — gets its own dedicated table with explicit foreign keys enforcing referential integrity. The ontology concepts from Brick Schema and Project Haystack are mapped directly onto relational structures: Brick's class hierarchy becomes lookup tables, Haystack's tag system becomes a normalised tag/entity junction, and telemetry is separated into a dedicated time-series partition.

This is the approach used by traditional BMS platforms and enterprise CMMS tools. It favours correctness and queryability over flexibility — every relationship is explicit, every constraint is enforced at the database level, and complex cross-entity queries (e.g., "find all VAV boxes on floor 3 that feed zones with CO2 above 800 ppm in the last hour") can be answered with standard SQL joins. The trade-off is schema rigidity: adding a new equipment type or sensor category requires DDL changes.

The model is designed for PostgreSQL with TimescaleDB for the telemetry partition. Core operational tables use standard relational design, while high-volume sensor readings use hypertables with automatic time-based partitioning and compression.

**Best for:** Regulated deployments requiring strict data integrity, auditable foreign-key chains, and complex cross-entity SQL reporting.

**Trade-offs:**
- (+) Full referential integrity — no orphaned records, no dangling relationships
- (+) Standard SQL for all queries — no special query languages or graph traversals needed
- (+) Well-understood by most developers; extensive tooling support
- (+) Strong alignment with Brick Schema entity classes and Haystack tag taxonomy
- (-) High table count (~44 tables) increases migration and ORM complexity
- (-) Schema changes require ALTER TABLE migrations; less flexible for jurisdiction-specific fields
- (-) Many JOIN operations for cross-domain queries (e.g., "all sensors in zones with high CO2 on floor 3")
- (-) Temporal queries ("what was the equipment configuration on date X?") require separate history tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Brick Schema | Equipment, point, and space class hierarchies mapped to `brick_class_uri` columns in entity tables |
| Project Haystack | Tag vocabulary stored in `haystack_tag_definitions` reference table; entity tags in junction table |
| BACnet (ASHRAE 135 / ISO 16484-5) | BACnet object types, instance numbers, and device IDs stored as dedicated columns on points and equipment |
| ISO 3166-1/2 | Country and subdivision codes on site records |
| ISO 52120 | BACS efficiency class (A-D) stored on site records |
| MQTT (ISO/IEC 20922) | MQTT topic patterns stored on data source configuration; telemetry ingestion via MQTT normalised to point references |
| IEC 62541 (OPC UA) | OPC UA node IDs stored on point records for OPC UA data sources |
| DTDL v3 | Optional `dtdl_model_id` column on equipment types for Azure Digital Twins interoperability |
| OpenAPI 3.x | API resource structure mirrors table names and relationships directly |
| OAuth 2.0 / OIDC | User authentication and tenant scoping via `tenant_id` foreign key on all operational tables |

---

## Multi-Tenancy and Identity

```sql
-- ============================================================
-- TENANT AND IDENTITY
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local', -- 'local', 'oidc', 'saml'
    auth_subject    VARCHAR(500),                          -- external IdP subject ID
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system_role  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   VARCHAR(100) NOT NULL,  -- 'site', 'building', 'equipment', 'alarm', etc.
    action          VARCHAR(50) NOT NULL,    -- 'read', 'write', 'delete', 'configure', 'acknowledge'
    description     TEXT,
    UNIQUE(resource_type, action)
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    scope_type      VARCHAR(50),            -- NULL = tenant-wide, 'site', 'building'
    scope_id        UUID,                   -- ID of specific site/building if scoped
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id, COALESCE(scope_type, ''), COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_user_roles_user ON user_roles(user_id);
CREATE INDEX idx_user_roles_scope ON user_roles(scope_type, scope_id) WHERE scope_type IS NOT NULL;
```

---

## Spatial Hierarchy

```sql
-- ============================================================
-- SPATIAL HIERARCHY: Portfolio > Site > Building > Floor > Zone > Space
-- ============================================================

CREATE TABLE portfolios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_portfolios_tenant ON portfolios(tenant_id);

CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    portfolio_id    UUID REFERENCES portfolios(id),
    name            VARCHAR(255) NOT NULL,
    address_line1   VARCHAR(500),
    address_line2   VARCHAR(500),
    city            VARCHAR(255),
    state_province  VARCHAR(255),
    postal_code     VARCHAR(50),
    country_code    CHAR(2) NOT NULL,            -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),                  -- ISO 3166-2
    latitude        NUMERIC(10, 7),
    longitude       NUMERIC(10, 7),
    timezone        VARCHAR(50) NOT NULL,         -- IANA timezone (e.g., 'America/New_York')
    gross_area_sqm  NUMERIC(12, 2),
    year_built      SMALLINT,
    bacs_class      CHAR(1) CHECK (bacs_class IN ('A', 'B', 'C', 'D')),  -- ISO 52120
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sites_tenant ON sites(tenant_id);
CREATE INDEX idx_sites_portfolio ON sites(portfolio_id);
CREATE INDEX idx_sites_country ON sites(country_code);

CREATE TABLE buildings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    name            VARCHAR(255) NOT NULL,
    building_type   VARCHAR(100),                 -- 'office', 'hospital', 'campus', 'retail', 'warehouse'
    gross_area_sqm  NUMERIC(12, 2),
    floor_count     SMALLINT,
    year_built      SMALLINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_buildings_tenant ON buildings(tenant_id);
CREATE INDEX idx_buildings_site ON buildings(site_id);

CREATE TABLE floors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    building_id     UUID NOT NULL REFERENCES buildings(id),
    name            VARCHAR(100) NOT NULL,        -- 'Ground', 'L1', 'B1' (basement)
    level_number    SMALLINT NOT NULL,             -- numeric ordering; 0 = ground
    gross_area_sqm  NUMERIC(12, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_floors_building ON floors(building_id);

CREATE TABLE zones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    floor_id        UUID NOT NULL REFERENCES floors(id),
    name            VARCHAR(255) NOT NULL,
    zone_type       VARCHAR(100) NOT NULL,        -- 'hvac_zone', 'lighting_zone', 'security_zone', 'fire_zone'
    area_sqm        NUMERIC(12, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_zones_floor ON zones(floor_id);
CREATE INDEX idx_zones_type ON zones(zone_type);

CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    floor_id        UUID NOT NULL REFERENCES floors(id),
    zone_id         UUID REFERENCES zones(id),
    name            VARCHAR(255) NOT NULL,
    space_type      VARCHAR(100) NOT NULL,        -- 'office', 'meeting_room', 'lobby', 'server_room', 'restroom'
    capacity        SMALLINT,
    area_sqm        NUMERIC(10, 2),
    is_bookable     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_spaces_floor ON spaces(floor_id);
CREATE INDEX idx_spaces_zone ON spaces(zone_id);
CREATE INDEX idx_spaces_bookable ON spaces(is_bookable) WHERE is_bookable = true;
```

---

## Equipment and Points

```sql
-- ============================================================
-- REFERENCE DATA
-- ============================================================

CREATE TABLE equipment_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL UNIQUE,     -- 'AHU', 'VAV', 'Chiller', 'Boiler', 'Lighting_Panel'
    brick_class_uri VARCHAR(500),                      -- e.g., 'https://brickschema.org/schema/Brick#AHU'
    haystack_marker VARCHAR(200),                      -- e.g., 'ahu'
    dtdl_model_id   VARCHAR(500),                      -- e.g., 'dtmi:com:willowinc:AirHandlingUnit;1'
    category        VARCHAR(100) NOT NULL,             -- 'hvac', 'lighting', 'access', 'fire', 'electrical', 'elevator'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE point_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL UNIQUE,      -- 'Zone_Air_Temperature_Sensor', 'Discharge_Air_Flow_Setpoint'
    brick_class_uri VARCHAR(500),                       -- e.g., 'https://brickschema.org/schema/Brick#Zone_Air_Temperature_Sensor'
    haystack_marker VARCHAR(200),                       -- e.g., 'zone,air,temp,sensor'
    point_category  VARCHAR(50) NOT NULL,               -- 'sensor', 'setpoint', 'command', 'status', 'alarm'
    default_unit    VARCHAR(50),                        -- 'degC', 'Pa', 'CFM', '%RH'
    data_type       VARCHAR(30) NOT NULL DEFAULT 'numeric', -- 'numeric', 'boolean', 'string', 'enum'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE units_of_measure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    symbol          VARCHAR(30) NOT NULL UNIQUE,       -- 'degC', 'kWh', 'Pa', 'CFM', 'ppm', '%RH', 'lux'
    name            VARCHAR(100) NOT NULL,
    quantity_kind   VARCHAR(100) NOT NULL,              -- 'temperature', 'energy', 'pressure', 'airflow', 'concentration'
    si_conversion_factor NUMERIC(20, 10),               -- multiply value by this to get SI base unit
    si_conversion_offset NUMERIC(20, 10) DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- EQUIPMENT AND POINTS
-- ============================================================

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    equipment_type_id UUID NOT NULL REFERENCES equipment_types(id),
    parent_equipment_id UUID REFERENCES equipment(id),   -- self-referencing for equipment hierarchies (AHU > VAV)
    building_id     UUID NOT NULL REFERENCES buildings(id),
    floor_id        UUID REFERENCES floors(id),
    zone_id         UUID REFERENCES zones(id),
    space_id        UUID REFERENCES spaces(id),
    name            VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    serial_number   VARCHAR(200),
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    firmware_version VARCHAR(100),
    install_date    DATE,
    warranty_expiry DATE,
    status          VARCHAR(50) NOT NULL DEFAULT 'active', -- 'active', 'inactive', 'maintenance', 'decommissioned'
    -- BACnet device properties (ISO 16484-5)
    bacnet_device_id INTEGER,
    bacnet_network   INTEGER,
    bacnet_mac       VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_equipment_tenant ON equipment(tenant_id);
CREATE INDEX idx_equipment_building ON equipment(building_id);
CREATE INDEX idx_equipment_type ON equipment(equipment_type_id);
CREATE INDEX idx_equipment_parent ON equipment(parent_equipment_id);
CREATE INDEX idx_equipment_zone ON equipment(zone_id);
CREATE INDEX idx_equipment_status ON equipment(status);
CREATE INDEX idx_equipment_bacnet ON equipment(bacnet_device_id) WHERE bacnet_device_id IS NOT NULL;

CREATE TABLE points (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    point_type_id   UUID NOT NULL REFERENCES point_types(id),
    unit_id         UUID REFERENCES units_of_measure(id),
    name            VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    -- BACnet object properties (ISO 16484-5)
    bacnet_object_type VARCHAR(50),                     -- 'analogInput', 'analogOutput', 'binaryInput', etc.
    bacnet_object_instance INTEGER,
    -- Protocol-specific addressing
    mqtt_topic      VARCHAR(500),
    opcua_node_id   VARCHAR(500),
    -- Current value cache (denormalized for dashboard performance)
    current_value   NUMERIC(20, 6),
    current_value_str VARCHAR(500),                     -- for string/enum point types
    current_quality VARCHAR(20) DEFAULT 'good',         -- 'good', 'uncertain', 'bad', 'offline'
    current_timestamp TIMESTAMPTZ,
    -- Thresholds
    alarm_low       NUMERIC(20, 6),
    alarm_high      NUMERIC(20, 6),
    warning_low     NUMERIC(20, 6),
    warning_high    NUMERIC(20, 6),
    is_writable     BOOLEAN NOT NULL DEFAULT false,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    polling_interval_sec INTEGER DEFAULT 60,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_points_tenant ON points(tenant_id);
CREATE INDEX idx_points_equipment ON points(equipment_id);
CREATE INDEX idx_points_type ON points(point_type_id);
CREATE INDEX idx_points_mqtt ON points(mqtt_topic) WHERE mqtt_topic IS NOT NULL;
CREATE INDEX idx_points_bacnet ON points(bacnet_object_type, bacnet_object_instance)
    WHERE bacnet_object_type IS NOT NULL;
```

---

## Telemetry and Time-Series Data

```sql
-- ============================================================
-- TELEMETRY (TimescaleDB hypertable)
-- ============================================================

CREATE TABLE telemetry (
    time            TIMESTAMPTZ NOT NULL,
    point_id        UUID NOT NULL REFERENCES points(id),
    value_numeric   NUMERIC(20, 6),
    value_string    VARCHAR(500),
    value_boolean   BOOLEAN,
    quality         VARCHAR(20) NOT NULL DEFAULT 'good',
    source          VARCHAR(50) NOT NULL DEFAULT 'device' -- 'device', 'manual', 'calculated', 'ai_override'
);
-- Convert to TimescaleDB hypertable:
-- SELECT create_hypertable('telemetry', 'time');
-- Enable compression for data older than 7 days:
-- ALTER TABLE telemetry SET (timescaledb.compress, timescaledb.compress_segmentby = 'point_id');
-- SELECT add_compression_policy('telemetry', INTERVAL '7 days');
-- Retention policy: drop raw data older than 1 year:
-- SELECT add_retention_policy('telemetry', INTERVAL '1 year');
CREATE INDEX idx_telemetry_point_time ON telemetry(point_id, time DESC);
CREATE INDEX idx_telemetry_time ON telemetry(time DESC);

-- Pre-aggregated hourly rollups (TimescaleDB continuous aggregate or materialised table)
CREATE TABLE telemetry_hourly (
    time            TIMESTAMPTZ NOT NULL,
    point_id        UUID NOT NULL REFERENCES points(id),
    min_value       NUMERIC(20, 6),
    max_value       NUMERIC(20, 6),
    avg_value       NUMERIC(20, 6),
    sample_count    INTEGER NOT NULL,
    PRIMARY KEY (point_id, time)
);

-- Pre-aggregated daily rollups
CREATE TABLE telemetry_daily (
    time            DATE NOT NULL,
    point_id        UUID NOT NULL REFERENCES points(id),
    min_value       NUMERIC(20, 6),
    max_value       NUMERIC(20, 6),
    avg_value       NUMERIC(20, 6),
    sample_count    INTEGER NOT NULL,
    PRIMARY KEY (point_id, time)
);
```

---

## Energy and Sustainability

```sql
-- ============================================================
-- ENERGY MANAGEMENT
-- ============================================================

CREATE TABLE energy_meters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    building_id     UUID NOT NULL REFERENCES buildings(id),
    floor_id        UUID REFERENCES floors(id),
    name            VARCHAR(255) NOT NULL,
    meter_type      VARCHAR(50) NOT NULL,              -- 'electricity', 'gas', 'water', 'steam', 'chilled_water'
    is_main_meter   BOOLEAN NOT NULL DEFAULT false,
    parent_meter_id UUID REFERENCES energy_meters(id), -- sub-metering hierarchy
    utility_account VARCHAR(200),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_energy_meters_building ON energy_meters(building_id);

CREATE TABLE energy_readings (
    time            TIMESTAMPTZ NOT NULL,
    meter_id        UUID NOT NULL REFERENCES energy_meters(id),
    consumption_kwh NUMERIC(14, 4),                    -- or equivalent unit for gas/water
    demand_kw       NUMERIC(12, 4),
    power_factor    NUMERIC(5, 4),
    cost_amount     NUMERIC(12, 4),
    cost_currency   CHAR(3),                           -- ISO 4217
    source          VARCHAR(50) NOT NULL DEFAULT 'meter' -- 'meter', 'manual', 'estimated'
);
-- SELECT create_hypertable('energy_readings', 'time');
CREATE INDEX idx_energy_readings_meter_time ON energy_readings(meter_id, time DESC);

CREATE TABLE carbon_factors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,
    grid_region     VARCHAR(100),
    energy_type     VARCHAR(50) NOT NULL,               -- 'electricity', 'natural_gas', 'district_heating'
    factor_kgco2_per_kwh NUMERIC(10, 6) NOT NULL,
    valid_from      DATE NOT NULL,
    valid_to        DATE,
    source          VARCHAR(255),                       -- e.g., 'IEA 2025', 'EPA eGRID'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_carbon_factors_lookup ON carbon_factors(country_code, energy_type, valid_from);
```

---

## Alarms and Notifications

```sql
-- ============================================================
-- ALARMS AND NOTIFICATIONS
-- ============================================================

CREATE TABLE alarm_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    severity        VARCHAR(20) NOT NULL,              -- 'critical', 'high', 'medium', 'low', 'info'
    category        VARCHAR(100) NOT NULL,             -- 'comfort', 'energy', 'maintenance', 'security', 'fire', 'system'
    condition_type  VARCHAR(50) NOT NULL,              -- 'threshold', 'rate_of_change', 'offline', 'pattern'
    condition_config JSONB NOT NULL,
    -- Example condition_config:
    -- {"point_type": "zone_air_temp_sensor", "operator": ">", "value": 28, "duration_minutes": 15}
    auto_ticket     BOOLEAN NOT NULL DEFAULT false,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alarm_defs_tenant ON alarm_definitions(tenant_id);

CREATE TABLE alarms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    alarm_definition_id UUID NOT NULL REFERENCES alarm_definitions(id),
    point_id        UUID REFERENCES points(id),
    equipment_id    UUID REFERENCES equipment(id),
    building_id     UUID NOT NULL REFERENCES buildings(id),
    severity        VARCHAR(20) NOT NULL,
    state           VARCHAR(20) NOT NULL DEFAULT 'active', -- 'active', 'acknowledged', 'resolved', 'suppressed'
    triggered_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alarms_tenant_state ON alarms(tenant_id, state);
CREATE INDEX idx_alarms_building ON alarms(building_id);
CREATE INDEX idx_alarms_severity ON alarms(severity) WHERE state = 'active';
CREATE INDEX idx_alarms_triggered ON alarms(triggered_at DESC);

CREATE TABLE notification_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    channel_type    VARCHAR(50) NOT NULL,              -- 'email', 'sms', 'webhook', 'push', 'slack', 'teams'
    config          JSONB NOT NULL,                    -- channel-specific configuration
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alarm_escalation_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alarm_definition_id UUID NOT NULL REFERENCES alarm_definitions(id),
    escalation_level SMALLINT NOT NULL DEFAULT 1,
    delay_minutes   INTEGER NOT NULL DEFAULT 0,
    channel_id      UUID NOT NULL REFERENCES notification_channels(id),
    recipient_user_id UUID REFERENCES users(id),
    recipient_role_id UUID REFERENCES roles(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Maintenance and Work Orders

```sql
-- ============================================================
-- MAINTENANCE AND WORK ORDERS
-- ============================================================

CREATE TABLE work_order_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100) NOT NULL,             -- 'preventive', 'inspection', 'calibration'
    estimated_hours NUMERIC(6, 2),
    checklist       JSONB,                             -- structured checklist items
    recurrence_rule VARCHAR(255),                       -- iCal RRULE format
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    template_id     UUID REFERENCES work_order_templates(id),
    alarm_id        UUID REFERENCES alarms(id),
    building_id     UUID NOT NULL REFERENCES buildings(id),
    equipment_id    UUID REFERENCES equipment(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium', -- 'critical', 'high', 'medium', 'low'
    status          VARCHAR(30) NOT NULL DEFAULT 'open',   -- 'open', 'assigned', 'in_progress', 'on_hold', 'completed', 'cancelled'
    category        VARCHAR(100) NOT NULL,                 -- 'corrective', 'preventive', 'inspection', 'emergency'
    requested_by    UUID REFERENCES users(id),
    assigned_to     UUID REFERENCES users(id),
    due_date        TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    actual_hours    NUMERIC(6, 2),
    resolution_notes TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_work_orders_tenant_status ON work_orders(tenant_id, status);
CREATE INDEX idx_work_orders_building ON work_orders(building_id);
CREATE INDEX idx_work_orders_equipment ON work_orders(equipment_id);
CREATE INDEX idx_work_orders_assigned ON work_orders(assigned_to) WHERE status IN ('assigned', 'in_progress');
CREATE INDEX idx_work_orders_due ON work_orders(due_date) WHERE status NOT IN ('completed', 'cancelled');

CREATE TABLE work_order_comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_comments_wo ON work_order_comments(work_order_id);
```

---

## Occupancy and Indoor Air Quality

```sql
-- ============================================================
-- OCCUPANCY
-- ============================================================

CREATE TABLE occupancy_sensors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    sensor_type     VARCHAR(50) NOT NULL,              -- 'passive_wifi', 'ble', 'pir', 'camera_count', 'desk_sensor'
    vendor          VARCHAR(200),
    mac_address     VARCHAR(50),
    is_privacy_safe BOOLEAN NOT NULL DEFAULT true,     -- true if no PII is collected
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_occ_sensors_space ON occupancy_sensors(space_id);

CREATE TABLE occupancy_readings (
    time            TIMESTAMPTZ NOT NULL,
    sensor_id       UUID NOT NULL REFERENCES occupancy_sensors(id),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    occupant_count  INTEGER,
    occupancy_pct   NUMERIC(5, 2),                     -- percentage of capacity
    dwell_time_avg_min NUMERIC(8, 2),
    source          VARCHAR(50) NOT NULL DEFAULT 'sensor'
);
-- SELECT create_hypertable('occupancy_readings', 'time');
CREATE INDEX idx_occ_readings_space_time ON occupancy_readings(space_id, time DESC);
CREATE INDEX idx_occ_readings_time ON occupancy_readings(time DESC);

-- ============================================================
-- INDOOR AIR QUALITY (IAQ)
-- ============================================================

CREATE TABLE iaq_readings (
    time            TIMESTAMPTZ NOT NULL,
    point_id        UUID NOT NULL REFERENCES points(id),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    co2_ppm         NUMERIC(8, 2),
    temperature_c   NUMERIC(6, 2),
    humidity_pct    NUMERIC(5, 2),
    pm25_ugm3       NUMERIC(8, 2),
    pm10_ugm3       NUMERIC(8, 2),
    tvoc_ppb        NUMERIC(8, 2),
    iaq_score       NUMERIC(5, 2)                      -- composite 0-100 score
);
-- SELECT create_hypertable('iaq_readings', 'time');
CREATE INDEX idx_iaq_space_time ON iaq_readings(space_id, time DESC);
```

---

## Tenant/Occupant Experience

```sql
-- ============================================================
-- TENANT AND OCCUPANT EXPERIENCE
-- ============================================================

CREATE TABLE building_tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),   -- platform tenant
    building_id     UUID NOT NULL REFERENCES buildings(id),
    company_name    VARCHAR(500) NOT NULL,
    lease_start     DATE,
    lease_end       DATE,
    contact_email   VARCHAR(320),
    contact_phone   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bldg_tenants_building ON building_tenants(building_id);

CREATE TABLE building_tenant_spaces (
    building_tenant_id UUID NOT NULL REFERENCES building_tenants(id) ON DELETE CASCADE,
    space_id           UUID NOT NULL REFERENCES spaces(id),
    PRIMARY KEY (building_tenant_id, space_id)
);

CREATE TABLE space_bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    booked_by       UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(500),
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'confirmed', -- 'confirmed', 'cancelled', 'auto_released'
    auto_release_minutes INTEGER DEFAULT 15,                  -- release if no occupancy detected
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT no_overlap EXCLUDE USING GIST (
        space_id WITH =,
        tstzrange(starts_at, ends_at) WITH &&
    ) WHERE (status = 'confirmed')
);
CREATE INDEX idx_bookings_space_time ON space_bookings(space_id, starts_at, ends_at);
CREATE INDEX idx_bookings_user ON space_bookings(booked_by);

CREATE TABLE comfort_preferences (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    preferred_temp_c NUMERIC(4, 1),
    preferred_humidity_pct NUMERIC(5, 2),
    preferred_lighting_lux NUMERIC(7, 1),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Data Sources and Integration

```sql
-- ============================================================
-- DATA SOURCES AND INTEGRATION CONNECTORS
-- ============================================================

CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    building_id     UUID NOT NULL REFERENCES buildings(id),
    name            VARCHAR(255) NOT NULL,
    protocol        VARCHAR(50) NOT NULL,              -- 'bacnet_ip', 'bacnet_sc', 'mqtt', 'opcua', 'modbus_tcp', 'rest_api'
    connection_config JSONB NOT NULL,
    -- Example for BACnet: {"ip": "192.168.1.100", "port": 47808, "network": 1}
    -- Example for MQTT: {"broker": "mqtt.example.com", "port": 8883, "tls": true, "topic_prefix": "building/ahu"}
    -- Example for REST: {"base_url": "https://api.vendor.com/v1", "auth_type": "oauth2"}
    status          VARCHAR(30) NOT NULL DEFAULT 'configured', -- 'configured', 'connected', 'error', 'disabled'
    last_seen_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_data_sources_building ON data_sources(building_id);
CREATE INDEX idx_data_sources_status ON data_sources(status);

CREATE TABLE point_data_source_map (
    point_id        UUID NOT NULL REFERENCES points(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    source_address  VARCHAR(500) NOT NULL,             -- protocol-specific address within the source
    transform_expr  VARCHAR(500),                      -- optional value transformation expression
    PRIMARY KEY (point_id, data_source_id)
);
```

---

## AI and Analytics

```sql
-- ============================================================
-- AI MODELS AND RECOMMENDATIONS
-- ============================================================

CREATE TABLE ai_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    model_type      VARCHAR(100) NOT NULL,             -- 'hvac_optimization', 'fault_detection', 'energy_forecast', 'occupancy_prediction'
    version         VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'training', -- 'training', 'active', 'retired'
    config          JSONB NOT NULL,
    performance_metrics JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    model_id        UUID NOT NULL REFERENCES ai_models(id),
    building_id     UUID NOT NULL REFERENCES buildings(id),
    equipment_id    UUID REFERENCES equipment(id),
    recommendation_type VARCHAR(100) NOT NULL,         -- 'setpoint_change', 'maintenance_alert', 'energy_saving', 'space_optimization'
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    confidence      NUMERIC(5, 4),                     -- 0.0 to 1.0
    estimated_impact JSONB,
    -- Example: {"energy_savings_kwh_year": 12000, "cost_savings_usd_year": 1800, "co2_reduction_kg_year": 5400}
    status          VARCHAR(30) NOT NULL DEFAULT 'pending', -- 'pending', 'accepted', 'rejected', 'implemented', 'expired'
    actioned_by     UUID REFERENCES users(id),
    actioned_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_recs_tenant_status ON ai_recommendations(tenant_id, status);
CREATE INDEX idx_ai_recs_building ON ai_recommendations(building_id);
```

---

## Audit Log

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,              -- 'create', 'update', 'delete', 'login', 'acknowledge_alarm', 'override_setpoint'
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID,
    changes         JSONB,                             -- {"field": {"old": x, "new": y}}
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- SELECT create_hypertable('audit_log', 'created_at');
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
```

---

## Haystack Tag Support

```sql
-- ============================================================
-- HAYSTACK TAG SUPPORT (optional layer)
-- ============================================================

CREATE TABLE haystack_tag_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tag_name        VARCHAR(200) NOT NULL UNIQUE,
    tag_kind        VARCHAR(30) NOT NULL,               -- 'marker', 'str', 'number', 'bool', 'ref', 'uri'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE entity_tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(50) NOT NULL,               -- 'equipment', 'point', 'space', 'site'
    entity_id       UUID NOT NULL,
    tag_id          UUID NOT NULL REFERENCES haystack_tag_definitions(id),
    value_str       VARCHAR(500),
    value_num       NUMERIC(20, 6),
    value_bool      BOOLEAN,
    value_ref       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_entity_tags_entity ON entity_tags(entity_type, entity_id);
CREATE INDEX idx_entity_tags_tag ON entity_tags(tag_id);
CREATE INDEX idx_entity_tags_lookup ON entity_tags(entity_type, tag_id, value_str);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Identity | 6 | tenants, users, roles, permissions, role_permissions, user_roles |
| Spatial Hierarchy | 6 | portfolios, sites, buildings, floors, zones, spaces |
| Reference Data | 3 | equipment_types, point_types, units_of_measure |
| Equipment & Points | 2 | equipment, points |
| Telemetry | 3 | telemetry, telemetry_hourly, telemetry_daily |
| Energy | 3 | energy_meters, energy_readings, carbon_factors |
| Alarms & Notifications | 4 | alarm_definitions, alarms, notification_channels, alarm_escalation_rules |
| Maintenance | 3 | work_order_templates, work_orders, work_order_comments |
| Occupancy & IAQ | 3 | occupancy_sensors, occupancy_readings, iaq_readings |
| Tenant Experience | 4 | building_tenants, building_tenant_spaces, space_bookings, comfort_preferences |
| Data Sources | 2 | data_sources, point_data_source_map |
| AI & Analytics | 2 | ai_models, ai_recommendations |
| Audit | 1 | audit_log |
| Haystack Tags | 2 | haystack_tag_definitions, entity_tags |
| **Total** | **44** | |

---

## Key Design Decisions

1. **Explicit spatial hierarchy rather than adjacency list.** The chain `portfolio > site > building > floor > zone > space` uses dedicated tables with direct foreign keys rather than a single recursive `locations` table. This makes cross-level queries straightforward (e.g., "all equipment on floor 3") but means adding a new hierarchy level requires a new table and migration.

2. **Separate telemetry table with time-series partitioning.** Raw telemetry is isolated from the point definition table to support billions of rows with TimescaleDB hypertable partitioning. Pre-aggregated hourly and daily rollup tables prevent expensive full-table scans for dashboard queries.

3. **Denormalized current value on points table.** The `current_value` and `current_timestamp` on the `points` table duplicate the latest telemetry reading. This avoids a JOIN to the massive telemetry table for real-time dashboard rendering, at the cost of maintaining consistency during writes.

4. **Brick Schema and Haystack alignment via reference tables.** Equipment and point types carry `brick_class_uri` and `haystack_marker` columns, enabling ontology-aware queries without requiring an RDF triple store. The optional `entity_tags` junction table provides full Haystack tag support for teams that need it.

5. **JSONB used sparingly and only for configuration.** JSONB columns appear only in configuration contexts (data source connection, alarm condition, AI model config) where the schema varies by type. All queryable operational data uses typed columns with proper indexes.

6. **Row-level security via tenant_id.** Every operational table carries a `tenant_id` foreign key. PostgreSQL RLS policies should be applied on all tables to enforce tenant isolation at the database level, regardless of application logic correctness.

7. **Equipment self-referencing hierarchy.** The `parent_equipment_id` on `equipment` supports containment relationships (e.g., an AHU contains multiple VAV boxes) without a separate relationship table. Traversal uses recursive CTEs:
   ```sql
   WITH RECURSIVE equip_tree AS (
       SELECT id, name, parent_equipment_id, 1 AS depth
       FROM equipment WHERE id = '<ahu_id>'
       UNION ALL
       SELECT e.id, e.name, e.parent_equipment_id, et.depth + 1
       FROM equipment e JOIN equip_tree et ON e.parent_equipment_id = et.id
   )
   SELECT * FROM equip_tree;
   ```

8. **Exclusion constraint for space bookings.** The `space_bookings` table uses a PostgreSQL EXCLUDE constraint with GiST index to prevent overlapping confirmed bookings at the database level, eliminating race conditions that application-level checks would miss.

9. **Carbon factors as a temporal reference table.** Carbon emission factors change annually as grid mixes shift. The `valid_from`/`valid_to` pattern allows historical emissions calculations to use the correct factor for each time period, supporting accurate Scope 1/2 reporting.

10. **Audit log as append-only table.** The `audit_log` table captures all user and system actions with before/after change diffs in JSONB. It is never updated or deleted, providing a compliance-grade audit trail. For very high-volume deployments, this table should be a TimescaleDB hypertable partitioned by time.
