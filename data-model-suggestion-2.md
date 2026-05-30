# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Smart Building Platform · Created: 2026-05-22

## Philosophy

This model treats every change in the system as an immutable event appended to an event store. The current state of any entity -- a building's configuration, an equipment's status, a setpoint's value -- is derived by replaying the sequence of events that created it. Separate read-optimized projections (materialized views) serve dashboards, API queries, and reporting, following the CQRS (Command Query Responsibility Segregation) pattern.

Smart buildings are inherently event-driven: sensors emit readings, alarms fire, setpoints change, occupants enter and leave, work orders transition through states. An event-sourced architecture makes this natural flow the primary data structure rather than retrofitting audit logs onto mutable tables. Every telemetry reading, every configuration change, every AI recommendation is preserved forever in its original form, enabling temporal queries ("what was the HVAC configuration at 3 AM when the complaint was filed?"), deterministic replay for debugging, and rich AI training data from complete operational histories.

This pattern is used in production by financial trading platforms, healthcare record systems, and industrial IoT platforms where full auditability and temporal reconstruction are non-negotiable requirements.

**Best for:** Deployments requiring complete audit trails, temporal reconstruction of building state, AI training on historical operational patterns, and regulatory compliance where proving "what happened when" is critical.

**Trade-offs:**
- Pro: Complete, immutable audit trail by design -- not a bolted-on afterthought
- Pro: Temporal queries are native ("show building configuration as of date X")
- Pro: AI/ML training benefits from rich, timestamped operational event streams
- Pro: Event replay enables debugging and root cause analysis for building incidents
- Pro: Schema evolution via new event types without altering existing data
- Con: Higher storage requirements (events are never deleted, only compacted via snapshots)
- Con: Read queries require projections; eventual consistency between writes and reads
- Con: More complex application architecture; teams unfamiliar with CQRS face a learning curve
- Con: Snapshot management needed to prevent slow replay for entities with long histories
- Con: Harder to perform ad-hoc SQL queries directly; requires maintaining projection views

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Brick Schema | Entity class URIs embedded in equipment and point configuration events |
| Project Haystack | Tag vocabulary carried in entity configuration events as structured payloads |
| BACnet (ASHRAE 135 / ISO 16484-5) | BACnet COV (Change of Value) notifications map directly to telemetry events |
| ISO 52120 | BACS efficiency class transitions recorded as building configuration events |
| MQTT (ISO/IEC 20922) | MQTT messages ingested directly as raw events before normalization |
| OCSF (Open Cybersecurity Schema Framework) | Event schema structure influenced by OCSF for security and access events |
| CloudEvents (CNCF) | Event envelope format follows CloudEvents specification for interoperability |
| ISO 3166-1/2 | Jurisdiction codes embedded in site configuration events |
| OpenAPI 3.x | Read-model API endpoints documented via OpenAPI; command endpoints follow CQRS conventions |

---

## Event Store Core

```sql
-- ============================================================
-- EVENT STORE: The single source of truth
-- ============================================================

-- All domain events are appended to this table. It is NEVER updated or deleted.
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                     -- aggregate root ID (equipment ID, building ID, etc.)
    stream_type     VARCHAR(100) NOT NULL,             -- 'building', 'equipment', 'point', 'alarm', 'work_order', 'user'
    event_type      VARCHAR(200) NOT NULL,             -- e.g., 'equipment.installed', 'point.value_changed', 'alarm.triggered'
    event_version   INTEGER NOT NULL,                  -- monotonically increasing per stream
    tenant_id       UUID NOT NULL,
    -- CloudEvents-compatible envelope
    source          VARCHAR(500) NOT NULL,             -- e.g., 'bacnet://192.168.1.100', 'mqtt://broker/topic', 'ui://user/abc'
    subject         VARCHAR(500),                      -- human-readable subject
    correlation_id  UUID,                              -- links related events (e.g., alarm -> work_order -> resolution)
    causation_id    UUID,                              -- the event that caused this event
    -- Payload
    data            JSONB NOT NULL,                    -- event-specific payload (see examples below)
    metadata        JSONB NOT NULL DEFAULT '{}',       -- non-domain context: user_id, ip_address, user_agent
    -- Timestamps
    occurred_at     TIMESTAMPTZ NOT NULL,              -- when the event actually happened in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(), -- when we persisted it
    schema_version  SMALLINT NOT NULL DEFAULT 1,       -- for event payload schema evolution
    UNIQUE(stream_id, event_version)                   -- optimistic concurrency control
);

-- Partitioning by month for manageability
-- CREATE TABLE events PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type, recorded_at);
CREATE INDEX idx_events_tenant_time ON events(tenant_id, recorded_at DESC);
CREATE INDEX idx_events_correlation ON events(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_events_occurred ON events(occurred_at DESC);

-- ============================================================
-- SNAPSHOTS: Periodic state snapshots for fast replay
-- ============================================================

CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    snapshot_version INTEGER NOT NULL,                  -- event_version at which snapshot was taken
    state           JSONB NOT NULL,                    -- full aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Keep only latest N snapshots per stream via application logic
CREATE INDEX idx_snapshots_stream ON snapshots(stream_id, snapshot_version DESC);
```

---

## Event Type Catalogue

```sql
-- ============================================================
-- EVENT TYPE REGISTRY (documentation / validation reference)
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      VARCHAR(200) PRIMARY KEY,
    stream_type     VARCHAR(100) NOT NULL,
    description     TEXT NOT NULL,
    data_schema     JSONB NOT NULL,                    -- JSON Schema for the event's data payload
    schema_version  SMALLINT NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- KEY EVENT TYPES AND THEIR PAYLOADS
-- ============================================================

-- Below are example events showing the data payload structure for each event type.
-- These are documentation; the actual data is stored as JSONB in the events table.

/*
EVENT TYPE: site.created
STREAM TYPE: site
DATA PAYLOAD:
{
    "name": "Corporate Campus Alpha",
    "address": {"line1": "123 Tech Blvd", "city": "Austin", "state": "TX", "country": "US", "postal": "78701"},
    "timezone": "America/Chicago",
    "latitude": 30.2672,
    "longitude": -97.7431,
    "gross_area_sqm": 45000
}

EVENT TYPE: building.configured
STREAM TYPE: building
DATA PAYLOAD:
{
    "site_id": "uuid",
    "name": "Building A",
    "building_type": "office",
    "floor_count": 12,
    "bacs_class": "B",
    "gross_area_sqm": 15000
}

EVENT TYPE: equipment.installed
STREAM TYPE: equipment
DATA PAYLOAD:
{
    "equipment_type": "AHU",
    "brick_class_uri": "https://brickschema.org/schema/Brick#AHU",
    "building_id": "uuid",
    "floor_id": "uuid",
    "zone_id": "uuid",
    "name": "AHU-3-01",
    "manufacturer": "Carrier",
    "model": "39M",
    "serial_number": "CR-2024-88432",
    "bacnet_device_id": 3001
}

EVENT TYPE: equipment.status_changed
STREAM TYPE: equipment
DATA PAYLOAD:
{
    "previous_status": "active",
    "new_status": "maintenance",
    "reason": "Scheduled quarterly maintenance",
    "work_order_id": "uuid"
}

EVENT TYPE: point.registered
STREAM TYPE: point
DATA PAYLOAD:
{
    "equipment_id": "uuid",
    "point_type": "Zone_Air_Temperature_Sensor",
    "brick_class_uri": "https://brickschema.org/schema/Brick#Zone_Air_Temperature_Sensor",
    "unit": "degC",
    "data_type": "numeric",
    "bacnet_object_type": "analogInput",
    "bacnet_object_instance": 1001,
    "alarm_high": 28.0,
    "alarm_low": 16.0
}

EVENT TYPE: point.value_changed
STREAM TYPE: point
DATA PAYLOAD:
{
    "value": 23.4,
    "quality": "good",
    "source": "bacnet"
}

EVENT TYPE: point.setpoint_overridden
STREAM TYPE: point
DATA PAYLOAD:
{
    "previous_value": 22.0,
    "new_value": 24.0,
    "override_source": "ai_optimization",
    "model_id": "uuid",
    "confidence": 0.92,
    "reason": "Occupancy below 30%, weather forecast 35C, energy price peak"
}

EVENT TYPE: alarm.triggered
STREAM TYPE: alarm
DATA PAYLOAD:
{
    "alarm_definition_id": "uuid",
    "point_id": "uuid",
    "equipment_id": "uuid",
    "building_id": "uuid",
    "severity": "high",
    "condition": {"operator": ">", "threshold": 28.0, "actual_value": 29.3, "duration_minutes": 15},
    "message": "Zone air temperature exceeded 28°C for 15 minutes"
}

EVENT TYPE: alarm.acknowledged
STREAM TYPE: alarm
DATA PAYLOAD:
{
    "acknowledged_by": "uuid",
    "notes": "Investigating - may be sensor calibration issue"
}

EVENT TYPE: alarm.resolved
STREAM TYPE: alarm
DATA PAYLOAD:
{
    "resolved_by": "uuid",
    "resolution": "Sensor recalibrated; readings normal",
    "root_cause": "sensor_drift"
}

EVENT TYPE: work_order.created
STREAM TYPE: work_order
DATA PAYLOAD:
{
    "title": "AHU-3-01 temperature sensor calibration",
    "building_id": "uuid",
    "equipment_id": "uuid",
    "alarm_id": "uuid",
    "priority": "high",
    "category": "corrective",
    "description": "Zone 3 temperature readings drifting high. Sensor requires recalibration."
}

EVENT TYPE: work_order.assigned
STREAM TYPE: work_order
DATA PAYLOAD:
{
    "assigned_to": "uuid",
    "assigned_by": "uuid",
    "due_date": "2026-05-25T17:00:00Z"
}

EVENT TYPE: work_order.completed
STREAM TYPE: work_order
DATA PAYLOAD:
{
    "completed_by": "uuid",
    "actual_hours": 1.5,
    "resolution_notes": "Sensor recalibrated. Offset was +2.1°C. Verified against reference thermometer.",
    "parts_used": []
}

EVENT TYPE: occupancy.reading
STREAM TYPE: space
DATA PAYLOAD:
{
    "sensor_id": "uuid",
    "occupant_count": 12,
    "occupancy_pct": 48.0,
    "sensing_method": "passive_wifi"
}

EVENT TYPE: energy.reading
STREAM TYPE: energy_meter
DATA PAYLOAD:
{
    "consumption_kwh": 245.7,
    "demand_kw": 89.2,
    "power_factor": 0.95,
    "reading_type": "interval_15min"
}

EVENT TYPE: iaq.reading
STREAM TYPE: space
DATA PAYLOAD:
{
    "co2_ppm": 612,
    "temperature_c": 23.1,
    "humidity_pct": 45.2,
    "pm25_ugm3": 8.3
}

EVENT TYPE: space.booking_created
STREAM TYPE: space
DATA PAYLOAD:
{
    "booked_by": "uuid",
    "title": "Sprint Planning",
    "starts_at": "2026-05-22T09:00:00Z",
    "ends_at": "2026-05-22T10:00:00Z"
}

EVENT TYPE: ai.recommendation_generated
STREAM TYPE: building
DATA PAYLOAD:
{
    "model_id": "uuid",
    "recommendation_type": "setpoint_change",
    "title": "Reduce cooling setpoint by 1.5°C during off-peak hours",
    "description": "Analysis of 30-day occupancy and energy data suggests...",
    "confidence": 0.88,
    "estimated_impact": {
        "energy_savings_kwh_year": 12000,
        "cost_savings_usd_year": 1800,
        "co2_reduction_kg_year": 5400
    }
}
*/
```

---

## Read-Model Projections (CQRS Query Side)

```sql
-- ============================================================
-- PROJECTION: Current Building State
-- Rebuilt from events; can be dropped and replayed at any time
-- ============================================================

CREATE TABLE proj_tenants (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_sites (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    timezone        VARCHAR(50) NOT NULL,
    latitude        NUMERIC(10, 7),
    longitude       NUMERIC(10, 7),
    gross_area_sqm  NUMERIC(12, 2),
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_sites_tenant ON proj_sites(tenant_id);

CREATE TABLE proj_buildings (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    site_id         UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    building_type   VARCHAR(100),
    floor_count     SMALLINT,
    bacs_class      CHAR(1),
    gross_area_sqm  NUMERIC(12, 2),
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_buildings_site ON proj_buildings(site_id);

CREATE TABLE proj_floors (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    building_id     UUID NOT NULL,
    name            VARCHAR(100) NOT NULL,
    level_number    SMALLINT NOT NULL,
    gross_area_sqm  NUMERIC(12, 2),
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_floors_building ON proj_floors(building_id);

CREATE TABLE proj_zones (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    floor_id        UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    zone_type       VARCHAR(100) NOT NULL,
    area_sqm        NUMERIC(12, 2),
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_zones_floor ON proj_zones(floor_id);

CREATE TABLE proj_spaces (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    floor_id        UUID NOT NULL,
    zone_id         UUID,
    name            VARCHAR(255) NOT NULL,
    space_type      VARCHAR(100) NOT NULL,
    capacity        SMALLINT,
    area_sqm        NUMERIC(10, 2),
    is_bookable     BOOLEAN NOT NULL DEFAULT false,
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_spaces_floor ON proj_spaces(floor_id);

-- ============================================================
-- PROJECTION: Current Equipment and Point State
-- ============================================================

CREATE TABLE proj_equipment (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    equipment_type  VARCHAR(200) NOT NULL,
    brick_class_uri VARCHAR(500),
    parent_equipment_id UUID,
    building_id     UUID NOT NULL,
    floor_id        UUID,
    zone_id         UUID,
    space_id        UUID,
    name            VARCHAR(255) NOT NULL,
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    serial_number   VARCHAR(200),
    status          VARCHAR(50) NOT NULL,
    bacnet_device_id INTEGER,
    install_date    DATE,
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_equip_tenant ON proj_equipment(tenant_id);
CREATE INDEX idx_proj_equip_building ON proj_equipment(building_id);
CREATE INDEX idx_proj_equip_status ON proj_equipment(status);
CREATE INDEX idx_proj_equip_type ON proj_equipment(equipment_type);

CREATE TABLE proj_points (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    equipment_id    UUID NOT NULL,
    point_type      VARCHAR(200) NOT NULL,
    brick_class_uri VARCHAR(500),
    unit            VARCHAR(50),
    data_type       VARCHAR(30) NOT NULL,
    -- Current value (updated by telemetry event projector)
    current_value   NUMERIC(20, 6),
    current_value_str VARCHAR(500),
    current_quality VARCHAR(20),
    current_timestamp TIMESTAMPTZ,
    -- Thresholds
    alarm_high      NUMERIC(20, 6),
    alarm_low       NUMERIC(20, 6),
    is_writable     BOOLEAN NOT NULL DEFAULT false,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_points_equipment ON proj_points(equipment_id);
CREATE INDEX idx_proj_points_type ON proj_points(point_type);

-- ============================================================
-- PROJECTION: Active Alarms
-- ============================================================

CREATE TABLE proj_active_alarms (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    alarm_definition_id UUID,
    point_id        UUID,
    equipment_id    UUID,
    building_id     UUID NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    state           VARCHAR(20) NOT NULL,
    message         TEXT,
    triggered_at    TIMESTAMPTZ NOT NULL,
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID,
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_alarms_tenant ON proj_active_alarms(tenant_id, state);
CREATE INDEX idx_proj_alarms_building ON proj_active_alarms(building_id);
CREATE INDEX idx_proj_alarms_severity ON proj_active_alarms(severity);

-- ============================================================
-- PROJECTION: Work Orders
-- ============================================================

CREATE TABLE proj_work_orders (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    building_id     UUID NOT NULL,
    equipment_id    UUID,
    alarm_id        UUID,
    title           VARCHAR(500) NOT NULL,
    priority        VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    assigned_to     UUID,
    due_date        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_wo_tenant_status ON proj_work_orders(tenant_id, status);
CREATE INDEX idx_proj_wo_building ON proj_work_orders(building_id);
CREATE INDEX idx_proj_wo_assigned ON proj_work_orders(assigned_to) WHERE status IN ('assigned', 'in_progress');
```

---

## Telemetry Event Stream (High-Volume)

```sql
-- ============================================================
-- TELEMETRY EVENT STREAM
-- Separate from the main event store due to volume (millions/day)
-- Same immutable, append-only pattern but optimized for time-series
-- ============================================================

CREATE TABLE telemetry_events (
    time            TIMESTAMPTZ NOT NULL,
    point_id        UUID NOT NULL,
    value_numeric   NUMERIC(20, 6),
    value_string    VARCHAR(500),
    value_boolean   BOOLEAN,
    quality         VARCHAR(20) NOT NULL DEFAULT 'good',
    source          VARCHAR(50) NOT NULL DEFAULT 'device'
);
-- SELECT create_hypertable('telemetry_events', 'time');
CREATE INDEX idx_telemetry_events_point ON telemetry_events(point_id, time DESC);

-- Continuous aggregates (TimescaleDB)
CREATE TABLE telemetry_hourly_agg (
    time            TIMESTAMPTZ NOT NULL,
    point_id        UUID NOT NULL,
    min_value       NUMERIC(20, 6),
    max_value       NUMERIC(20, 6),
    avg_value       NUMERIC(20, 6),
    sample_count    INTEGER NOT NULL,
    PRIMARY KEY (point_id, time)
);

CREATE TABLE telemetry_daily_agg (
    time            DATE NOT NULL,
    point_id        UUID NOT NULL,
    min_value       NUMERIC(20, 6),
    max_value       NUMERIC(20, 6),
    avg_value       NUMERIC(20, 6),
    sample_count    INTEGER NOT NULL,
    PRIMARY KEY (point_id, time)
);

-- ============================================================
-- OCCUPANCY EVENT STREAM (separate high-volume stream)
-- ============================================================

CREATE TABLE occupancy_events (
    time            TIMESTAMPTZ NOT NULL,
    space_id        UUID NOT NULL,
    sensor_id       UUID NOT NULL,
    occupant_count  INTEGER,
    occupancy_pct   NUMERIC(5, 2),
    sensing_method  VARCHAR(50) NOT NULL
);
CREATE INDEX idx_occ_events_space ON occupancy_events(space_id, time DESC);

-- ============================================================
-- ENERGY EVENT STREAM
-- ============================================================

CREATE TABLE energy_events (
    time            TIMESTAMPTZ NOT NULL,
    meter_id        UUID NOT NULL,
    consumption_kwh NUMERIC(14, 4),
    demand_kw       NUMERIC(12, 4),
    power_factor    NUMERIC(5, 4),
    reading_type    VARCHAR(50) NOT NULL DEFAULT 'interval_15min'
);
CREATE INDEX idx_energy_events_meter ON energy_events(meter_id, time DESC);
```

---

## Temporal Query Examples

```sql
-- ============================================================
-- EXAMPLE: Reconstruct equipment state at a specific point in time
-- "What was the status of AHU-3-01 at 3 AM on May 15?"
-- ============================================================

SELECT e.data
FROM events e
WHERE e.stream_id = 'ahu-3-01-uuid'
  AND e.stream_type = 'equipment'
  AND e.occurred_at <= '2026-05-15T03:00:00Z'
ORDER BY e.event_version DESC
LIMIT 1;

-- Or replay all events up to that point for full state reconstruction:
SELECT e.event_type, e.data, e.occurred_at
FROM events e
WHERE e.stream_id = 'ahu-3-01-uuid'
  AND e.stream_type = 'equipment'
  AND e.occurred_at <= '2026-05-15T03:00:00Z'
ORDER BY e.event_version ASC;

-- ============================================================
-- EXAMPLE: Find all setpoint overrides by AI in the last 7 days
-- ============================================================

SELECT
    e.stream_id AS point_id,
    e.data->>'previous_value' AS old_setpoint,
    e.data->>'new_value' AS new_setpoint,
    e.data->>'reason' AS reason,
    e.data->>'confidence' AS confidence,
    e.occurred_at
FROM events e
WHERE e.event_type = 'point.setpoint_overridden'
  AND e.tenant_id = 'tenant-uuid'
  AND e.data->>'override_source' = 'ai_optimization'
  AND e.occurred_at >= now() - INTERVAL '7 days'
ORDER BY e.occurred_at DESC;

-- ============================================================
-- EXAMPLE: Alarm-to-resolution timeline (event correlation)
-- ============================================================

SELECT
    e.event_type,
    e.occurred_at,
    e.data,
    e.metadata
FROM events e
WHERE e.correlation_id = 'alarm-correlation-uuid'
ORDER BY e.occurred_at ASC;
-- Returns: alarm.triggered -> alarm.acknowledged -> work_order.created ->
--          work_order.assigned -> work_order.completed -> alarm.resolved

-- ============================================================
-- EXAMPLE: Building configuration diff between two dates
-- ============================================================

WITH config_before AS (
    SELECT DISTINCT ON (stream_id) stream_id, data, occurred_at
    FROM events
    WHERE stream_type = 'equipment'
      AND event_type = 'equipment.installed'
      AND occurred_at <= '2026-01-01T00:00:00Z'
    ORDER BY stream_id, event_version DESC
),
config_after AS (
    SELECT DISTINCT ON (stream_id) stream_id, data, occurred_at
    FROM events
    WHERE stream_type = 'equipment'
      AND event_type IN ('equipment.installed', 'equipment.status_changed')
      AND occurred_at <= '2026-05-01T00:00:00Z'
    ORDER BY stream_id, event_version DESC
)
SELECT
    COALESCE(b.stream_id, a.stream_id) AS equipment_id,
    b.data AS config_jan,
    a.data AS config_may
FROM config_after a
FULL OUTER JOIN config_before b ON a.stream_id = b.stream_id
WHERE b.data IS DISTINCT FROM a.data;
```

---

## Projection Rebuild Infrastructure

```sql
-- ============================================================
-- PROJECTION TRACKING: Tracks which events each projector has processed
-- ============================================================

CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(200) PRIMARY KEY,         -- 'proj_equipment', 'proj_active_alarms', etc.
    last_event_id   UUID NOT NULL,
    last_recorded_at TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    status          VARCHAR(30) NOT NULL DEFAULT 'running', -- 'running', 'rebuilding', 'paused', 'error'
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- DEAD LETTER QUEUE: Events that failed projection processing
-- ============================================================

CREATE TABLE projection_dead_letters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(200) NOT NULL,
    event_id        UUID NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    last_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_dead_letters_projection ON projection_dead_letters(projection_name);
```

---

## Multi-Tenancy and Identity (via Events)

```sql
-- ============================================================
-- IDENTITY: These tables are conventional (not event-sourced)
-- because authentication is a cross-cutting concern
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
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
    is_system_role  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    scope_type      VARCHAR(50),
    scope_id        UUID,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);
```

---

## Reference Data

```sql
-- ============================================================
-- REFERENCE DATA: Static/slow-changing lookup tables
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

CREATE TABLE units_of_measure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    symbol          VARCHAR(30) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    quantity_kind   VARCHAR(100) NOT NULL,
    si_conversion_factor NUMERIC(20, 10),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, snapshots |
| Event Infrastructure | 3 | event_type_registry, projection_checkpoints, projection_dead_letters |
| Telemetry Streams | 5 | telemetry_events, telemetry_hourly_agg, telemetry_daily_agg, occupancy_events, energy_events |
| Projections: Spatial | 6 | proj_tenants, proj_sites, proj_buildings, proj_floors, proj_zones, proj_spaces |
| Projections: Equipment | 2 | proj_equipment, proj_points |
| Projections: Operations | 2 | proj_active_alarms, proj_work_orders |
| Identity (conventional) | 4 | tenants, users, roles, user_roles |
| Reference Data | 2 | carbon_factors, units_of_measure |
| **Total** | **26** | Plus projections can be added without schema migration |

---

## Key Design Decisions

1. **Single event store for configuration, dual streams for telemetry.** Configuration events (equipment installed, status changed, alarm triggered) go to the main `events` table with optimistic concurrency via `stream_id + event_version`. High-volume telemetry, occupancy, and energy readings use separate append-only tables optimized for time-series queries, avoiding the overhead of stream versioning for data that is inherently idempotent.

2. **CloudEvents-compatible envelope.** Every event carries `source`, `subject`, `correlation_id`, and `causation_id` fields following the CNCF CloudEvents specification. This enables event forwarding to external systems, distributed tracing across microservices, and standardized event processing in downstream consumers.

3. **Correlation chains for incident tracking.** The `correlation_id` links all events related to a building incident: from the initial alarm trigger, through acknowledgement, work order creation, assignment, completion, and alarm resolution. Querying by `correlation_id` reconstructs the full incident timeline -- a powerful tool for root cause analysis and SLA reporting.

4. **Projections are disposable.** Every `proj_*` table can be dropped and rebuilt from the event store at any time. The `projection_checkpoints` table tracks each projector's progress, enabling incremental catch-up after restarts. This means read-model schema changes don't require data migration -- just rebuild the projection.

5. **Separate identity tables (not event-sourced).** User authentication and RBAC are modeled as conventional mutable tables rather than event streams. Authentication is a cross-cutting infrastructure concern where the complexity of event sourcing adds no value -- you never need to answer "what roles did user X have on date Y?"

6. **Schema evolution via event versioning.** Each event carries a `schema_version` field. When an event's payload structure changes, the version increments. Projectors handle multiple versions via upcasting logic, and old events are never rewritten. This enables forward-compatible schema evolution without breaking the immutable event log.

7. **Snapshot strategy for long-lived aggregates.** Equipment and buildings accumulate thousands of events over their lifetime. Snapshots capture the full aggregate state at periodic intervals, so state reconstruction starts from the latest snapshot rather than replaying from event zero. Application logic manages snapshot frequency (e.g., every 100 events or every 24 hours).

8. **Dead letter queue for projection failures.** Events that fail projection processing are captured in `projection_dead_letters` rather than blocking the entire projection pipeline. This prevents a single malformed event from halting all read-model updates, while preserving the failed event for diagnosis and retry.

9. **Temporal queries are first-class.** Because events carry both `occurred_at` (real-world time) and `recorded_at` (persistence time), the system supports bi-temporal queries. This is essential for investigating building incidents where the question is "what did we know, and when did we know it?"

10. **AI training data is a free byproduct.** The complete event stream -- every setpoint change, every alarm, every resolution, every occupancy pattern -- becomes the training corpus for AI models. Unlike mutable databases where historical states are overwritten, the event store preserves the full operational history that predictive models need.
