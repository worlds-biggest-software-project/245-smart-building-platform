# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Smart Building Platform · Created: 2026-05-22

## Philosophy

This model uses typed relational columns for the core structural skeleton of a smart building (sites, buildings, equipment, points) while storing variable, protocol-specific, and jurisdiction-dependent attributes in JSONB columns. The insight is that a building's spatial hierarchy and equipment taxonomy are universal, but the details vary enormously: a BACnet-connected AHU has different metadata than an MQTT-connected IAQ sensor; a European deployment needs ISO 52120 BACS classification while a US deployment needs ENERGY STAR scores; a Brick Schema user wants different tags than a Haystack user.

Rather than creating separate columns (or worse, separate tables) for every possible attribute across every protocol, ontology, and jurisdiction, JSONB columns absorb this variability. The relational skeleton provides fast indexed queries on the dimensions that every deployment shares (location, equipment type, point type, timestamps), while JSONB columns accommodate the dimensions that differ without requiring ALTER TABLE migrations.

This pattern is widely used in modern SaaS platforms (Shopify's metafields, Stripe's metadata, Salesforce's custom fields) and is particularly well-suited to IoT platforms where device types and protocols proliferate faster than schema migrations can keep up. PostgreSQL's JSONB support -- including GIN indexes, containment operators, and jsonpath queries -- makes this viable at scale.

**Best for:** Multi-protocol IoT platforms that must support diverse device types, building ontologies, and regional requirements without constant schema migrations. Ideal for rapid MVP development and platforms targeting both SMB and enterprise markets.

**Trade-offs:**
- Pro: Fewer tables (~30) with less migration complexity than fully normalized model
- Pro: New device types, protocols, and regional requirements need no schema changes
- Pro: Supports Brick Schema, Haystack, DTDL, and custom ontologies simultaneously via flexible tagging
- Pro: Rapid prototyping; schema can evolve through application code without database migrations
- Pro: PostgreSQL JSONB is well-understood, well-indexed, and battle-tested
- Con: JSONB fields lack foreign key constraints; data integrity relies on application validation
- Con: Complex JSONB queries can be harder to optimize than simple column queries
- Con: Risk of "schema-on-read" chaos if JSONB structures are not documented and validated
- Con: ORMs handle JSONB inconsistently; may require custom query builders
- Con: JSONB columns can grow large if not managed, impacting row-level I/O

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Brick Schema | Brick class URIs and tag sets stored in equipment/point `ontology` JSONB field |
| Project Haystack | Haystack markers and tags stored in same `ontology` JSONB field with `haystack:` prefix |
| DTDL v3 | DTDL model IDs and interface definitions stored in `ontology.dtdl_model_id` |
| BACnet (ASHRAE 135) | BACnet device/object properties stored in equipment/point `protocol_config` JSONB |
| MQTT (ISO/IEC 20922) | MQTT topic patterns and broker config in data source `config` JSONB |
| OPC UA (IEC 62541) | OPC UA node IDs and namespace indexes in point `protocol_config` JSONB |
| ISO 3166-1/2 | Country/subdivision codes as typed columns (queried frequently enough to warrant indexing) |
| ISO 52120 | BACS class stored in building `properties` JSONB under regional compliance section |
| W3C WoT Thing Description | Device capability descriptions storable in equipment `properties` JSONB |
| NGSI-LD / FIWARE | Entity context and property graph relationships expressible via `properties` and `relationships` JSONB |

---

## Core Schema

```sql
-- ============================================================
-- MULTI-TENANCY
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "default_ontology": "brick",
    --   "default_units": "metric",
    --   "features_enabled": ["ai_optimization", "carbon_reporting"],
    --   "data_retention_days": 730
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    auth_subject    VARCHAR(500),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example preferences:
    -- {
    --   "comfort": {"preferred_temp_c": 22.5, "preferred_humidity_pct": 45},
    --   "notifications": {"channels": ["email", "push"], "quiet_hours": {"start": "22:00", "end": "07:00"}},
    --   "dashboard": {"default_view": "portfolio", "favorite_buildings": ["uuid1", "uuid2"]}
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);
CREATE INDEX idx_users_tenant ON users(tenant_id);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions:
    -- [
    --   {"resource": "building", "actions": ["read", "write"]},
    --   {"resource": "equipment", "actions": ["read"]},
    --   {"resource": "alarm", "actions": ["read", "acknowledge"]},
    --   {"resource": "work_order", "actions": ["read", "write", "assign"]}
    -- ]
    is_system_role  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    scope           JSONB,
    -- Example scope (NULL = tenant-wide):
    -- {"type": "building", "ids": ["uuid1", "uuid2"]}
    -- {"type": "site", "ids": ["uuid1"]}
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);
```

---

## Spatial Hierarchy

```sql
-- ============================================================
-- UNIFIED LOCATION HIERARCHY
-- Uses a single table with type discrimination and JSONB properties
-- ============================================================

CREATE TABLE locations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    parent_id       UUID REFERENCES locations(id),
    location_type   VARCHAR(30) NOT NULL,              -- 'portfolio', 'site', 'building', 'floor', 'zone', 'space'
    name            VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    -- Materialized path for fast ancestor/descendant queries
    path            TEXT NOT NULL,                     -- e.g., '/portfolio-uuid/site-uuid/building-uuid/floor-uuid'
    depth           SMALLINT NOT NULL,                 -- 0 = portfolio, 1 = site, etc.
    -- Common indexed fields (present on most location types)
    country_code    CHAR(2),                           -- ISO 3166-1 (sites, buildings)
    timezone        VARCHAR(50),                       -- IANA timezone (sites)
    area_sqm        NUMERIC(12, 2),
    -- Type-specific properties in JSONB
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for site:
    -- {
    --   "address": {"line1": "123 Tech Blvd", "city": "Austin", "state": "TX", "postal": "78701"},
    --   "coordinates": {"latitude": 30.2672, "longitude": -97.7431},
    --   "subdivision_code": "US-TX"
    -- }
    --
    -- Example for building:
    -- {
    --   "building_type": "office",
    --   "floor_count": 12,
    --   "year_built": 2019,
    --   "compliance": {
    --     "bacs_class": "B",
    --     "energy_star_score": 82,
    --     "leed_certification": "Gold"
    --   }
    -- }
    --
    -- Example for space:
    -- {
    --   "space_type": "meeting_room",
    --   "capacity": 12,
    --   "is_bookable": true,
    --   "amenities": ["whiteboard", "video_conferencing", "phone"]
    -- }
    --
    -- Example for zone:
    -- {
    --   "zone_type": "hvac_zone",
    --   "served_by_equipment": ["ahu-uuid"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_locations_tenant ON locations(tenant_id);
CREATE INDEX idx_locations_parent ON locations(parent_id);
CREATE INDEX idx_locations_type ON locations(location_type);
CREATE INDEX idx_locations_path ON locations USING GIST (path gist_trgm_ops);
CREATE INDEX idx_locations_country ON locations(country_code) WHERE country_code IS NOT NULL;
CREATE INDEX idx_locations_properties ON locations USING GIN (properties);

-- ============================================================
-- EXAMPLE QUERIES: Spatial hierarchy traversal
-- ============================================================

-- Find all spaces in a specific building (using materialized path):
-- SELECT * FROM locations
-- WHERE path LIKE '%/building-uuid/%'
--   AND location_type = 'space';

-- Find all descendants of a site:
-- SELECT * FROM locations
-- WHERE path LIKE '/portfolio-uuid/site-uuid/%'
-- ORDER BY depth, name;

-- Find all bookable meeting rooms with video conferencing:
-- SELECT * FROM locations
-- WHERE location_type = 'space'
--   AND properties @> '{"space_type": "meeting_room", "is_bookable": true}'
--   AND properties->'amenities' ? 'video_conferencing';
```

---

## Equipment and Points

```sql
-- ============================================================
-- EQUIPMENT
-- ============================================================

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    location_id     UUID NOT NULL REFERENCES locations(id), -- building, floor, zone, or space
    parent_id       UUID REFERENCES equipment(id),          -- equipment hierarchy (AHU > VAV)
    name            VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    category        VARCHAR(50) NOT NULL,                   -- 'hvac', 'lighting', 'access', 'fire', 'electrical', 'elevator', 'meter'
    equipment_type  VARCHAR(200) NOT NULL,                  -- 'AHU', 'VAV', 'Chiller', 'Boiler', etc.
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    -- Ontology alignment (supports multiple ontologies simultaneously)
    ontology        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "brick": {
    --     "class_uri": "https://brickschema.org/schema/Brick#AHU",
    --     "tags": ["ahu", "equip"]
    --   },
    --   "haystack": {
    --     "markers": ["ahu", "equip", "elec"],
    --     "tags": {"dis": "AHU-3-01", "siteRef": "site-uuid"}
    --   },
    --   "dtdl_model_id": "dtmi:com:willowinc:AirHandlingUnit;1"
    -- }
    -- Protocol-specific configuration
    protocol_config JSONB NOT NULL DEFAULT '{}',
    -- Example for BACnet device:
    -- {
    --   "protocol": "bacnet_ip",
    --   "device_id": 3001,
    --   "ip_address": "192.168.1.100",
    --   "network": 1,
    --   "mac": "0A",
    --   "max_apdu": 1476,
    --   "segmentation": "segmented-both"
    -- }
    --
    -- Example for MQTT device:
    -- {
    --   "protocol": "mqtt",
    --   "topic_prefix": "building/floor3/ahu01",
    --   "client_id": "ahu-3-01"
    -- }
    -- Manufacturer and asset details
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "manufacturer": "Carrier",
    --   "model": "39M",
    --   "serial_number": "CR-2024-88432",
    --   "firmware_version": "4.2.1",
    --   "install_date": "2024-03-15",
    --   "warranty_expiry": "2029-03-15",
    --   "rated_capacity_kw": 150,
    --   "refrigerant_type": "R-410A",
    --   "last_service_date": "2026-02-10"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_tenant ON equipment(tenant_id);
CREATE INDEX idx_equipment_location ON equipment(location_id);
CREATE INDEX idx_equipment_parent ON equipment(parent_id);
CREATE INDEX idx_equipment_category ON equipment(category);
CREATE INDEX idx_equipment_type ON equipment(equipment_type);
CREATE INDEX idx_equipment_status ON equipment(status);
CREATE INDEX idx_equipment_ontology ON equipment USING GIN (ontology);
CREATE INDEX idx_equipment_protocol ON equipment USING GIN (protocol_config);

-- ============================================================
-- POINTS (Sensors, Setpoints, Commands, Status)
-- ============================================================

CREATE TABLE points (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    name            VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    point_category  VARCHAR(30) NOT NULL,              -- 'sensor', 'setpoint', 'command', 'status', 'calculated'
    data_type       VARCHAR(30) NOT NULL,              -- 'numeric', 'boolean', 'string', 'enum'
    unit            VARCHAR(50),                       -- 'degC', 'kWh', 'Pa', 'CFM', 'ppm', 'lux', '%RH'
    -- Ontology
    ontology        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "brick": {
    --     "class_uri": "https://brickschema.org/schema/Brick#Zone_Air_Temperature_Sensor",
    --     "tags": ["zone", "air", "temp", "sensor", "point"]
    --   },
    --   "haystack": {
    --     "markers": ["zone", "air", "temp", "sensor", "point", "cur"],
    --     "tags": {"kind": "Number", "unit": "°C"}
    --   }
    -- }
    -- Protocol addressing
    protocol_config JSONB NOT NULL DEFAULT '{}',
    -- Example for BACnet:
    -- {
    --   "object_type": "analogInput",
    --   "object_instance": 1001,
    --   "property": "presentValue",
    --   "cov_increment": 0.5
    -- }
    --
    -- Example for OPC UA:
    -- {
    --   "node_id": "ns=2;s=AHU01.ZoneTemp",
    --   "namespace_uri": "urn:vendor:opcua:building"
    -- }
    --
    -- Example for MQTT:
    -- {
    --   "topic": "building/floor3/ahu01/zone_temp",
    --   "payload_path": "$.temperature",
    --   "payload_format": "json"
    -- }
    -- Thresholds and limits
    thresholds      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "alarm_high": 28.0,
    --   "alarm_low": 16.0,
    --   "warning_high": 26.0,
    --   "warning_low": 18.0,
    --   "deadband": 0.5,
    --   "rate_of_change_limit": 2.0
    -- }
    -- Current value cache
    current_value   JSONB,
    -- Example:
    -- {
    --   "value": 23.4,
    --   "quality": "good",
    --   "timestamp": "2026-05-22T14:30:00Z",
    --   "source": "bacnet"
    -- }
    is_writable     BOOLEAN NOT NULL DEFAULT false,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_points_tenant ON points(tenant_id);
CREATE INDEX idx_points_equipment ON points(equipment_id);
CREATE INDEX idx_points_category ON points(point_category);
CREATE INDEX idx_points_ontology ON points USING GIN (ontology);
CREATE INDEX idx_points_enabled ON points(is_enabled) WHERE is_enabled = true;

-- ============================================================
-- EXAMPLE QUERIES: Ontology-aware equipment/point lookups
-- ============================================================

-- Find all AHUs using Brick Schema class:
-- SELECT * FROM equipment
-- WHERE ontology @> '{"brick": {"class_uri": "https://brickschema.org/schema/Brick#AHU"}}';

-- Find all temperature sensors using Haystack tags:
-- SELECT * FROM points
-- WHERE ontology @> '{"haystack": {"markers": ["temp", "sensor"]}}';

-- Find all BACnet devices on a specific network:
-- SELECT * FROM equipment
-- WHERE protocol_config @> '{"protocol": "bacnet_ip", "network": 1}';

-- Find all points with high alarm threshold below 25:
-- SELECT * FROM points
-- WHERE (thresholds->>'alarm_high')::numeric < 25.0;
```

---

## Telemetry

```sql
-- ============================================================
-- TELEMETRY (TimescaleDB hypertable recommended)
-- ============================================================

CREATE TABLE telemetry (
    time            TIMESTAMPTZ NOT NULL,
    point_id        UUID NOT NULL,
    value_numeric   NUMERIC(20, 6),
    value_string    VARCHAR(500),
    value_boolean   BOOLEAN,
    quality         VARCHAR(20) NOT NULL DEFAULT 'good',
    source          VARCHAR(50) NOT NULL DEFAULT 'device',
    metadata        JSONB                              -- optional per-reading metadata
    -- Example metadata:
    -- {"bacnet_status_flags": "0000", "cov_notification": true}
    -- {"override_reason": "manual_operator", "previous_value": 22.0}
);
-- SELECT create_hypertable('telemetry', 'time');
CREATE INDEX idx_telemetry_point_time ON telemetry(point_id, time DESC);
CREATE INDEX idx_telemetry_time ON telemetry(time DESC);

-- Pre-aggregated rollups
CREATE TABLE telemetry_rollup (
    time            TIMESTAMPTZ NOT NULL,
    point_id        UUID NOT NULL,
    interval_type   VARCHAR(10) NOT NULL,              -- 'hourly', 'daily', 'monthly'
    min_value       NUMERIC(20, 6),
    max_value       NUMERIC(20, 6),
    avg_value       NUMERIC(20, 6),
    sum_value       NUMERIC(20, 6),
    sample_count    INTEGER NOT NULL,
    PRIMARY KEY (point_id, interval_type, time)
);
```

---

## Energy, Occupancy, and IAQ

```sql
-- ============================================================
-- ENERGY METERS (unified with equipment via type)
-- Energy meters are equipment with category = 'meter'
-- Readings are stored in a dedicated time-series table
-- ============================================================

CREATE TABLE energy_readings (
    time            TIMESTAMPTZ NOT NULL,
    equipment_id    UUID NOT NULL,                     -- references equipment where category = 'meter'
    readings        JSONB NOT NULL,
    -- Example for electricity meter:
    -- {
    --   "consumption_kwh": 245.7,
    --   "demand_kw": 89.2,
    --   "power_factor": 0.95,
    --   "voltage_v": 480,
    --   "current_a": 185.4,
    --   "reactive_power_kvar": 28.1
    -- }
    --
    -- Example for gas meter:
    -- {
    --   "consumption_therms": 12.3,
    --   "flow_rate_cfh": 4.1
    -- }
    --
    -- Example for water meter:
    -- {
    --   "consumption_gallons": 850,
    --   "flow_rate_gpm": 12.5
    -- }
    reading_type    VARCHAR(50) NOT NULL DEFAULT 'interval_15min',
    source          VARCHAR(50) NOT NULL DEFAULT 'meter'
);
CREATE INDEX idx_energy_readings_equip_time ON energy_readings(equipment_id, time DESC);

-- ============================================================
-- OCCUPANCY
-- ============================================================

CREATE TABLE occupancy_readings (
    time            TIMESTAMPTZ NOT NULL,
    location_id     UUID NOT NULL,                     -- references locations (space or zone)
    sensor_config   JSONB NOT NULL,
    -- Example:
    -- {
    --   "sensor_id": "uuid",
    --   "sensor_type": "passive_wifi",
    --   "vendor": "Occuspace",
    --   "is_privacy_safe": true
    -- }
    occupant_count  INTEGER,
    occupancy_pct   NUMERIC(5, 2),
    dwell_time_avg_min NUMERIC(8, 2)
);
CREATE INDEX idx_occ_readings_location_time ON occupancy_readings(location_id, time DESC);

-- ============================================================
-- INDOOR AIR QUALITY
-- ============================================================

CREATE TABLE iaq_readings (
    time            TIMESTAMPTZ NOT NULL,
    location_id     UUID NOT NULL,                     -- references locations (space or zone)
    readings        JSONB NOT NULL,
    -- Example:
    -- {
    --   "co2_ppm": 612,
    --   "temperature_c": 23.1,
    --   "humidity_pct": 45.2,
    --   "pm25_ugm3": 8.3,
    --   "pm10_ugm3": 15.1,
    --   "tvoc_ppb": 220,
    --   "iaq_score": 78
    -- }
    source          VARCHAR(50) NOT NULL DEFAULT 'sensor'
);
CREATE INDEX idx_iaq_readings_location_time ON iaq_readings(location_id, time DESC);
```

---

## Alarms, Work Orders, and Bookings

```sql
-- ============================================================
-- ALARMS
-- ============================================================

CREATE TABLE alarm_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL,
    -- Example config:
    -- {
    --   "condition_type": "threshold",
    --   "target": {"point_category": "sensor", "ontology_match": {"brick.tags": ["zone", "air", "temp"]}},
    --   "operator": ">",
    --   "value": 28.0,
    --   "duration_minutes": 15,
    --   "auto_create_ticket": true,
    --   "escalation": [
    --     {"level": 1, "delay_minutes": 0, "channels": ["push"], "recipients": {"role": "facility_manager"}},
    --     {"level": 2, "delay_minutes": 30, "channels": ["email", "sms"], "recipients": {"role": "building_engineer"}}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alarm_rules_tenant ON alarm_rules(tenant_id);

CREATE TABLE alarms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    alarm_rule_id   UUID REFERENCES alarm_rules(id),
    point_id        UUID,
    equipment_id    UUID REFERENCES equipment(id),
    location_id     UUID NOT NULL REFERENCES locations(id),
    severity        VARCHAR(20) NOT NULL,
    state           VARCHAR(20) NOT NULL DEFAULT 'active',
    message         TEXT,
    trigger_data    JSONB,
    -- Example:
    -- {"actual_value": 29.3, "threshold": 28.0, "duration_minutes": 18}
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES users(id),
    resolution_notes TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alarms_tenant_state ON alarms(tenant_id, state);
CREATE INDEX idx_alarms_location ON alarms(location_id);
CREATE INDEX idx_alarms_equipment ON alarms(equipment_id);
CREATE INDEX idx_alarms_created ON alarms(created_at DESC);

-- ============================================================
-- WORK ORDERS
-- ============================================================

CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    location_id     UUID NOT NULL REFERENCES locations(id),
    equipment_id    UUID REFERENCES equipment(id),
    alarm_id        UUID REFERENCES alarms(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    category        VARCHAR(100) NOT NULL,
    requested_by    UUID REFERENCES users(id),
    assigned_to     UUID REFERENCES users(id),
    due_date        TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    actual_hours    NUMERIC(6, 2),
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "checklist": [
    --     {"item": "Inspect filter condition", "completed": true},
    --     {"item": "Check belt tension", "completed": true},
    --     {"item": "Verify refrigerant levels", "completed": false}
    --   ],
    --   "parts_used": [
    --     {"part": "Air filter 20x25x4", "quantity": 2, "cost": 45.00}
    --   ],
    --   "resolution_notes": "Replaced clogged filters. Airflow restored.",
    --   "photos": ["s3://bucket/wo-123/before.jpg", "s3://bucket/wo-123/after.jpg"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_work_orders_tenant_status ON work_orders(tenant_id, status);
CREATE INDEX idx_work_orders_location ON work_orders(location_id);
CREATE INDEX idx_work_orders_equipment ON work_orders(equipment_id);
CREATE INDEX idx_work_orders_assigned ON work_orders(assigned_to) WHERE status IN ('assigned', 'in_progress');
CREATE INDEX idx_work_orders_due ON work_orders(due_date) WHERE status NOT IN ('completed', 'cancelled');

-- ============================================================
-- SPACE BOOKINGS
-- ============================================================

CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    location_id     UUID NOT NULL REFERENCES locations(id),  -- space
    booked_by       UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(500),
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'confirmed',
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "auto_release_minutes": 15,
    --   "attendees": ["uuid1", "uuid2"],
    --   "room_setup": "boardroom",
    --   "av_requirements": ["projector", "video_call"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT no_booking_overlap EXCLUDE USING GIST (
        location_id WITH =,
        tstzrange(starts_at, ends_at) WITH &&
    ) WHERE (status = 'confirmed')
);
CREATE INDEX idx_bookings_location_time ON bookings(location_id, starts_at, ends_at);
CREATE INDEX idx_bookings_user ON bookings(booked_by);
```

---

## Data Sources and AI

```sql
-- ============================================================
-- DATA SOURCES AND CONNECTORS
-- ============================================================

CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    location_id     UUID NOT NULL REFERENCES locations(id),  -- typically building-level
    name            VARCHAR(255) NOT NULL,
    protocol        VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'configured',
    config          JSONB NOT NULL,
    -- Example for BACnet gateway:
    -- {
    --   "ip": "192.168.1.100",
    --   "port": 47808,
    --   "network": 1,
    --   "security": {"bacnet_sc": true, "certificate": "cert-ref"},
    --   "discovery": {"auto_discover": true, "scan_interval_hours": 24}
    -- }
    --
    -- Example for MQTT broker:
    -- {
    --   "broker": "mqtt.building.example.com",
    --   "port": 8883,
    --   "tls": true,
    --   "username": "platform",
    --   "topic_prefix": "building/+/+",
    --   "qos": 1
    -- }
    --
    -- Example for REST API:
    -- {
    --   "base_url": "https://api.vendor.com/v2",
    --   "auth_type": "oauth2",
    --   "token_url": "https://auth.vendor.com/token",
    --   "client_id": "xxx",
    --   "poll_interval_seconds": 300,
    --   "endpoints": [
    --     {"path": "/meters", "method": "GET", "mapping": "energy_meter"},
    --     {"path": "/sensors/{id}/readings", "method": "GET", "mapping": "telemetry"}
    --   ]
    -- }
    last_seen_at    TIMESTAMPTZ,
    error_info      JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_data_sources_location ON data_sources(location_id);
CREATE INDEX idx_data_sources_protocol ON data_sources(protocol);

-- ============================================================
-- AI MODELS AND RECOMMENDATIONS
-- ============================================================

CREATE TABLE ai_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    model_type      VARCHAR(100) NOT NULL,
    version         VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'training',
    config          JSONB NOT NULL,
    metrics         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    model_id        UUID NOT NULL REFERENCES ai_models(id),
    location_id     UUID NOT NULL REFERENCES locations(id),
    equipment_id    UUID REFERENCES equipment(id),
    recommendation_type VARCHAR(100) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    confidence      NUMERIC(5, 4),
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "estimated_impact": {
    --     "energy_savings_kwh_year": 12000,
    --     "cost_savings_usd_year": 1800,
    --     "co2_reduction_kg_year": 5400
    --   },
    --   "supporting_data": {
    --     "occupancy_analysis_period": "2026-04-01/2026-05-01",
    --     "average_occupancy_pct": 38,
    --     "current_setpoint": 22.0,
    --     "recommended_setpoint": 24.0,
    --     "affected_zones": ["uuid1", "uuid2"]
    --   }
    -- }
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    actioned_by     UUID REFERENCES users(id),
    actioned_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_recs_tenant_status ON ai_recommendations(tenant_id, status);
CREATE INDEX idx_ai_recs_location ON ai_recommendations(location_id);

-- ============================================================
-- CARBON FACTORS (reference data)
-- ============================================================

CREATE TABLE carbon_factors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,
    grid_region     VARCHAR(100),
    energy_type     VARCHAR(50) NOT NULL,
    factor_kgco2_per_kwh NUMERIC(10, 6) NOT NULL,
    valid_from      DATE NOT NULL,
    valid_to        DATE,
    source          VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_carbon_factors_lookup ON carbon_factors(country_code, energy_type, valid_from);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID,
    changes         JSONB,
    context         JSONB,
    -- Example context:
    -- {"ip_address": "192.168.1.50", "user_agent": "Mozilla/5.0...", "session_id": "uuid"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Identity | 4 | tenants, users, roles, user_roles |
| Spatial Hierarchy | 1 | locations (unified with type discrimination) |
| Equipment & Points | 2 | equipment, points |
| Telemetry | 2 | telemetry, telemetry_rollup |
| Energy | 1 | energy_readings |
| Occupancy & IAQ | 2 | occupancy_readings, iaq_readings |
| Alarms | 3 | alarm_rules, alarms |
| Work Orders | 1 | work_orders |
| Bookings | 1 | bookings |
| Data Sources | 1 | data_sources |
| AI | 2 | ai_models, ai_recommendations |
| Reference Data | 1 | carbon_factors |
| Audit | 1 | audit_log |
| **Total** | **22** | Significantly fewer tables than normalized model |

---

## Key Design Decisions

1. **Unified `locations` table with materialized path.** Instead of six separate tables for portfolio/site/building/floor/zone/space, a single `locations` table uses `location_type` discrimination and a materialized `path` column for fast hierarchical queries. This reduces table count by five and makes adding new hierarchy levels (e.g., "campus" between portfolio and site) a data-only change, not a schema migration.

2. **Three-column JSONB strategy on equipment and points.** Each entity has `ontology` (Brick/Haystack/DTDL alignment), `protocol_config` (BACnet/MQTT/OPC UA addressing), and `properties` (manufacturer, capacity, commissioning details). Separating these concerns means ontology queries don't scan protocol data and vice versa, improving GIN index selectivity.

3. **Ontology-agnostic by design.** The `ontology` JSONB field supports Brick Schema, Project Haystack, and DTDL simultaneously on the same entity. A building using Brick for analytics and Haystack for BMS integration does not need to choose one or duplicate data. This matches the real-world convergence trend where ontologies are aligning but not yet unified.

4. **Protocol-agnostic point addressing.** The `protocol_config` JSONB on points stores addressing information specific to the data source protocol (BACnet object/instance, MQTT topic/path, OPC UA node ID). Adding support for a new protocol (KNX, Modbus, Matter) requires no schema changes -- just a new JSON structure in the config field.

5. **JSONB for alarm rule configuration and escalation.** Alarm rules embed their full condition logic and escalation chain in a single JSONB `config` field. This allows arbitrarily complex rules (multi-condition, rate-of-change, pattern-based) and multi-level escalation without a separate escalation rules table and junction tables.

6. **Energy readings with flexible JSONB payload.** Different meter types (electricity, gas, water, steam) produce different measurements. Rather than nullable columns for every possible reading type, the `readings` JSONB column accommodates any meter type's output. GIN indexing supports queries like "find all meters with power factor below 0.9."

7. **Work order details as embedded JSONB.** Checklists, parts used, photos, and resolution notes are embedded in a `details` JSONB field rather than normalized into separate tables. This keeps the work order as a single queryable document, simplifying the API and reducing JOIN complexity, at the cost of not being able to query across all checklists with pure SQL (though JSONB containment queries partially address this).

8. **Current value cached as JSONB on points.** The `current_value` field on points is a JSONB object containing value, quality, timestamp, and source. This denormalization avoids JOINing to the massive telemetry table for real-time dashboards. The JSONB format accommodates both numeric and string values without separate columns.

9. **Permissions as JSONB arrays on roles.** Instead of a separate permissions table and role-permissions junction table, permissions are embedded as a JSONB array on the roles table. This simplifies role management for the common case (most deployments have <20 roles) while supporting arbitrarily granular resource/action pairs.

10. **GIN indexes on all JSONB columns.** Every JSONB column that will be queried gets a GIN index, enabling efficient containment (`@>`), existence (`?`), and path queries. The trade-off is slower write performance and larger index sizes, but smart building platforms are overwhelmingly read-heavy for configuration data.
