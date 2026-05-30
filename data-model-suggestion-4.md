# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Smart Building Platform · Created: 2026-05-22

## Philosophy

Smart buildings are fundamentally graph problems. A zone is served by an AHU. The AHU feeds multiple VAV boxes. Each VAV box contains a temperature sensor that measures a space. The space is occupied by tenants. The building connects to a power grid via meters. An alarm on the AHU affects every zone it serves. Understanding these interconnections -- and traversing them efficiently -- is central to fault propagation analysis, impact assessment, AI optimization scope determination, and occupant comfort management.

This model uses a property graph layer (implemented as PostgreSQL tables with `graph_nodes` and `graph_edges`) for relationship-heavy queries, combined with conventional relational tables for operational CRUD (telemetry, work orders, bookings). The graph layer is modeled after Brick Schema's RDF triple structure and NGSI-LD's entity-relationship pattern, using labeled edges with properties to represent typed relationships (`feeds`, `hasPart`, `hasPoint`, `isLocationOf`, `servesZone`).

The relational tables handle what relational databases do best: time-series data ingestion, transactional work order management, and user authentication. The graph layer handles what graphs do best: traversal queries ("what equipment is upstream of this fault?"), topology analysis ("show me all the spaces affected if this chiller goes offline"), and semantic queries aligned with building ontologies.

This can be implemented entirely in PostgreSQL using recursive CTEs and graph-pattern tables, or with a dedicated graph database (Neo4j, Apache AGE extension for PostgreSQL) for the graph layer while keeping operational data in PostgreSQL.

**Best for:** Platforms where relationship traversal is central -- fault impact analysis, equipment dependency mapping, AI optimization scope discovery, semantic interoperability with Brick Schema/NGSI-LD, and digital twin knowledge graphs.

**Trade-offs:**
- Pro: Natural representation of building system relationships (feeds, serves, contains, hasPoint)
- Pro: Efficient multi-hop traversal queries without complex JOINs
- Pro: Direct alignment with Brick Schema RDF and NGSI-LD property graphs
- Pro: Enables powerful impact analysis ("what is affected if equipment X fails?")
- Pro: Extensible relationships without schema changes (new edge type = new row)
- Con: Two-model complexity (graph + relational) increases architectural surface area
- Con: Graph queries require different skills than standard SQL
- Con: Transaction management across graph and relational stores adds complexity
- Con: Graph layer needs careful index design for large buildings (10,000+ nodes)
- Con: ORM support for graph patterns is limited; requires custom query builders

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Brick Schema | Graph node classes and edge types directly mirror Brick ontology classes and relationships (`feeds`, `hasPart`, `hasPoint`, `isLocationOf`) |
| Project Haystack | Haystack reference tags (`siteRef`, `equipRef`, `spaceRef`) mapped to graph edges |
| DTDL v3 | DTDL relationship definitions modeled as edge types; DTDL interfaces as node types |
| NGSI-LD / FIWARE | Entity-Property-Relationship pattern directly maps to graph_nodes and graph_edges |
| BACnet (ASHRAE 135) | BACnet device/object hierarchy modeled as `hasPart` edges in the graph |
| W3C WoT Thing Description | Thing-to-Thing relationships expressed as graph edges |
| Building Topology Ontology (BOT) | Spatial containment relationships (`containsZone`, `adjacentElement`, `containsElement`) modeled as edge types |
| ISO 3166-1/2 | Jurisdiction properties on site nodes |
| ISO 52120 | BACS class as a property on building nodes |

---

## Graph Layer

```sql
-- ============================================================
-- GRAPH LAYER: Property Graph in PostgreSQL
-- ============================================================

-- Graph nodes represent every identifiable entity in the building
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(100) NOT NULL,             -- 'Site', 'Building', 'Floor', 'Zone', 'Space',
                                                       -- 'AHU', 'VAV', 'Chiller', 'Boiler', 'Lighting_Panel',
                                                       -- 'Temperature_Sensor', 'CO2_Sensor', 'Occupancy_Sensor',
                                                       -- 'Setpoint', 'Meter', 'Alarm_Definition'
    -- Ontology classification
    brick_class     VARCHAR(500),                      -- e.g., 'https://brickschema.org/schema/Brick#AHU'
    haystack_type   VARCHAR(200),                      -- e.g., 'ahu'
    dtdl_model_id   VARCHAR(500),                      -- e.g., 'dtmi:com:willowinc:AirHandlingUnit;1'
    -- Display
    name            VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    -- Category for quick filtering
    category        VARCHAR(50) NOT NULL,              -- 'spatial', 'equipment', 'point', 'meter', 'virtual'
    -- Node properties (type-specific attributes)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for a Building node:
    -- {
    --   "building_type": "office",
    --   "floor_count": 12,
    --   "gross_area_sqm": 15000,
    --   "year_built": 2019,
    --   "bacs_class": "B",
    --   "address": {"line1": "123 Tech Blvd", "city": "Austin", "country_code": "US"},
    --   "coordinates": {"latitude": 30.2672, "longitude": -97.7431},
    --   "timezone": "America/Chicago"
    -- }
    --
    -- Example for an AHU equipment node:
    -- {
    --   "manufacturer": "Carrier",
    --   "model": "39M",
    --   "serial_number": "CR-2024-88432",
    --   "status": "active",
    --   "install_date": "2024-03-15",
    --   "rated_capacity_kw": 150,
    --   "bacnet_device_id": 3001,
    --   "protocol": "bacnet_ip"
    -- }
    --
    -- Example for a Temperature_Sensor point node:
    -- {
    --   "point_category": "sensor",
    --   "data_type": "numeric",
    --   "unit": "degC",
    --   "bacnet_object_type": "analogInput",
    --   "bacnet_object_instance": 1001,
    --   "current_value": 23.4,
    --   "current_quality": "good",
    --   "current_timestamp": "2026-05-22T14:30:00Z",
    --   "alarm_high": 28.0,
    --   "alarm_low": 16.0
    -- }
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_tenant ON graph_nodes(tenant_id);
CREATE INDEX idx_gn_type ON graph_nodes(node_type);
CREATE INDEX idx_gn_category ON graph_nodes(category);
CREATE INDEX idx_gn_brick ON graph_nodes(brick_class) WHERE brick_class IS NOT NULL;
CREATE INDEX idx_gn_status ON graph_nodes(status);
CREATE INDEX idx_gn_properties ON graph_nodes USING GIN (properties);
CREATE INDEX idx_gn_name ON graph_nodes(tenant_id, name);

-- Graph edges represent typed, directed relationships between nodes
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,
    -- Core Brick Schema relationship types:
    --   'feeds'          - AHU feeds VAV; Chiller feeds AHU
    --   'hasPart'        - Building hasPart Floor; AHU hasPart Fan
    --   'hasPoint'       - Equipment hasPoint Sensor/Setpoint/Command
    --   'isLocationOf'   - Space isLocationOf Equipment
    --   'hasLocation'    - Equipment hasLocation Space
    --   'isPartOf'       - Floor isPartOf Building (inverse of hasPart)
    --   'isFedBy'        - VAV isFedBy AHU (inverse of feeds)
    --
    -- Spatial (BOT-aligned):
    --   'containsZone'   - Floor containsZone HVAC_Zone
    --   'containsSpace'  - Floor containsSpace Office_101
    --   'adjacentTo'     - Space adjacentTo Space
    --
    -- Operational:
    --   'servesZone'     - Equipment servesZone Zone
    --   'monitors'       - Sensor monitors Space
    --   'controls'       - Setpoint controls Equipment
    --   'metersEnergy'   - Meter metersEnergy Building/Floor
    --   'subMeterOf'     - SubMeter subMeterOf MainMeter
    --
    -- Organizational:
    --   'occupiedBy'     - Space occupiedBy Tenant
    --   'managedBy'      - Building managedBy User
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for 'feeds' edge:
    -- {"medium": "air", "design_flow_cfm": 5000}
    --
    -- Example for 'metersEnergy' edge:
    -- {"energy_type": "electricity", "is_main_meter": true}
    --
    -- Example for 'occupiedBy' edge:
    -- {"lease_start": "2025-01-01", "lease_end": "2028-12-31", "company": "Acme Corp"}
    weight          NUMERIC(10, 4),                    -- optional weight for graph algorithms
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                       -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_tenant ON graph_edges(tenant_id);
CREATE INDEX idx_ge_source ON graph_edges(source_id);
CREATE INDEX idx_ge_target ON graph_edges(target_id);
CREATE INDEX idx_ge_type ON graph_edges(edge_type);
CREATE INDEX idx_ge_source_type ON graph_edges(source_id, edge_type);
CREATE INDEX idx_ge_target_type ON graph_edges(target_id, edge_type);
CREATE INDEX idx_ge_active ON graph_edges(source_id, edge_type) WHERE valid_to IS NULL;
CREATE INDEX idx_ge_properties ON graph_edges USING GIN (properties);
```

---

## Graph Query Examples

```sql
-- ============================================================
-- EXAMPLE: Find all equipment downstream of a chiller (feeds traversal)
-- "If Chiller-01 fails, what equipment is affected?"
-- ============================================================

WITH RECURSIVE downstream AS (
    -- Start from the chiller
    SELECT
        gn.id,
        gn.name,
        gn.node_type,
        1 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.name = 'Chiller-01'
      AND gn.tenant_id = 'tenant-uuid'

    UNION ALL

    -- Traverse 'feeds' edges
    SELECT
        target.id,
        target.name,
        target.node_type,
        d.depth + 1,
        d.path || target.id
    FROM downstream d
    JOIN graph_edges ge ON ge.source_id = d.id
        AND ge.edge_type = 'feeds'
        AND ge.valid_to IS NULL
    JOIN graph_nodes target ON target.id = ge.target_id
    WHERE target.id != ALL(d.path)  -- prevent cycles
      AND d.depth < 10             -- safety limit
)
SELECT id, name, node_type, depth
FROM downstream
ORDER BY depth, name;

-- Returns: Chiller-01 -> AHU-01, AHU-02 -> VAV-01, VAV-02, VAV-03...

-- ============================================================
-- EXAMPLE: Find all spaces affected by an equipment failure
-- ============================================================

WITH RECURSIVE impact AS (
    SELECT gn.id, gn.name, gn.node_type, gn.category, 1 AS depth
    FROM graph_nodes gn
    WHERE gn.id = 'failed-equipment-uuid'

    UNION ALL

    SELECT target.id, target.name, target.node_type, target.category, i.depth + 1
    FROM impact i
    JOIN graph_edges ge ON ge.source_id = i.id
        AND ge.edge_type IN ('feeds', 'servesZone', 'hasLocation', 'containsSpace')
        AND ge.valid_to IS NULL
    JOIN graph_nodes target ON target.id = ge.target_id
    WHERE i.depth < 10
)
SELECT DISTINCT id, name, node_type
FROM impact
WHERE category = 'spatial' AND node_type IN ('Space', 'Zone');

-- ============================================================
-- EXAMPLE: Find all sensors for a specific space (multi-hop)
-- "Show me all sensors monitoring Meeting Room 5A"
-- ============================================================

SELECT
    sensor.id,
    sensor.name,
    sensor.node_type,
    sensor.properties->>'unit' AS unit,
    sensor.properties->>'current_value' AS current_value
FROM graph_nodes space
JOIN graph_edges e1 ON (
    -- Equipment located in this space
    (e1.target_id = space.id AND e1.edge_type = 'hasLocation')
    OR
    -- Equipment serving the zone this space belongs to
    (e1.target_id IN (
        SELECT ge.source_id FROM graph_edges ge
        WHERE ge.target_id = space.id AND ge.edge_type = 'containsSpace'
    ) AND e1.edge_type = 'servesZone')
) AND e1.valid_to IS NULL
JOIN graph_edges e2 ON e2.source_id = e1.source_id
    AND e2.edge_type = 'hasPoint'
    AND e2.valid_to IS NULL
JOIN graph_nodes sensor ON sensor.id = e2.target_id
    AND sensor.category = 'point'
WHERE space.name = 'Meeting Room 5A'
  AND space.tenant_id = 'tenant-uuid';

-- ============================================================
-- EXAMPLE: Equipment dependency path (BFS shortest path)
-- "What is the connection path between Chiller-01 and VAV-3-07?"
-- ============================================================

WITH RECURSIVE path_search AS (
    SELECT
        gn.id,
        gn.name,
        ARRAY[gn.id] AS path,
        ARRAY[gn.name] AS name_path,
        0 AS depth
    FROM graph_nodes gn
    WHERE gn.name = 'Chiller-01'
      AND gn.tenant_id = 'tenant-uuid'

    UNION ALL

    SELECT
        neighbor.id,
        neighbor.name,
        ps.path || neighbor.id,
        ps.name_path || (ge.edge_type || ' -> ' || neighbor.name),
        ps.depth + 1
    FROM path_search ps
    JOIN graph_edges ge ON (ge.source_id = ps.id OR ge.target_id = ps.id)
        AND ge.valid_to IS NULL
    JOIN graph_nodes neighbor ON neighbor.id = CASE
        WHEN ge.source_id = ps.id THEN ge.target_id
        ELSE ge.source_id
    END
    WHERE neighbor.id != ALL(ps.path)
      AND ps.depth < 10
)
SELECT name_path, depth
FROM path_search
WHERE name = 'VAV-3-07'
ORDER BY depth
LIMIT 1;

-- ============================================================
-- EXAMPLE: Brick Schema-style SPARQL-equivalent query
-- "Find all Zone_Air_Temperature_Sensors that are points of AHUs"
-- ============================================================

SELECT
    sensor.id,
    sensor.name,
    ahu.name AS ahu_name,
    sensor.properties->>'current_value' AS temperature
FROM graph_nodes sensor
JOIN graph_edges hp ON hp.target_id = sensor.id
    AND hp.edge_type = 'hasPoint'
    AND hp.valid_to IS NULL
JOIN graph_nodes ahu ON ahu.id = hp.source_id
WHERE sensor.brick_class = 'https://brickschema.org/schema/Brick#Zone_Air_Temperature_Sensor'
  AND ahu.brick_class = 'https://brickschema.org/schema/Brick#AHU'
  AND sensor.tenant_id = 'tenant-uuid';
```

---

## Operational Tables (Relational)

```sql
-- ============================================================
-- MULTI-TENANCY AND IDENTITY
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
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    auth_subject    VARCHAR(500),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    preferences     JSONB NOT NULL DEFAULT '{}',
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
    is_system_role  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    scope_type      VARCHAR(50),
    scope_id        UUID,                              -- graph_nodes.id for scoped access
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

-- ============================================================
-- TELEMETRY (time-series, references graph nodes)
-- ============================================================

CREATE TABLE telemetry (
    time            TIMESTAMPTZ NOT NULL,
    node_id         UUID NOT NULL,                     -- references graph_nodes.id (point node)
    value_numeric   NUMERIC(20, 6),
    value_string    VARCHAR(500),
    value_boolean   BOOLEAN,
    quality         VARCHAR(20) NOT NULL DEFAULT 'good',
    source          VARCHAR(50) NOT NULL DEFAULT 'device'
);
-- SELECT create_hypertable('telemetry', 'time');
CREATE INDEX idx_telemetry_node_time ON telemetry(node_id, time DESC);
CREATE INDEX idx_telemetry_time ON telemetry(time DESC);

CREATE TABLE telemetry_hourly (
    time            TIMESTAMPTZ NOT NULL,
    node_id         UUID NOT NULL,
    min_value       NUMERIC(20, 6),
    max_value       NUMERIC(20, 6),
    avg_value       NUMERIC(20, 6),
    sample_count    INTEGER NOT NULL,
    PRIMARY KEY (node_id, time)
);

CREATE TABLE telemetry_daily (
    time            DATE NOT NULL,
    node_id         UUID NOT NULL,
    min_value       NUMERIC(20, 6),
    max_value       NUMERIC(20, 6),
    avg_value       NUMERIC(20, 6),
    sample_count    INTEGER NOT NULL,
    PRIMARY KEY (node_id, time)
);

-- ============================================================
-- ENERGY READINGS
-- ============================================================

CREATE TABLE energy_readings (
    time            TIMESTAMPTZ NOT NULL,
    node_id         UUID NOT NULL,                     -- references graph_nodes.id (meter node)
    consumption_kwh NUMERIC(14, 4),
    demand_kw       NUMERIC(12, 4),
    power_factor    NUMERIC(5, 4),
    cost_amount     NUMERIC(12, 4),
    cost_currency   CHAR(3),
    source          VARCHAR(50) NOT NULL DEFAULT 'meter'
);
CREATE INDEX idx_energy_node_time ON energy_readings(node_id, time DESC);

-- ============================================================
-- OCCUPANCY READINGS
-- ============================================================

CREATE TABLE occupancy_readings (
    time            TIMESTAMPTZ NOT NULL,
    node_id         UUID NOT NULL,                     -- references graph_nodes.id (space or zone node)
    sensor_node_id  UUID NOT NULL,                     -- references graph_nodes.id (sensor node)
    occupant_count  INTEGER,
    occupancy_pct   NUMERIC(5, 2),
    dwell_time_avg_min NUMERIC(8, 2)
);
CREATE INDEX idx_occ_node_time ON occupancy_readings(node_id, time DESC);

-- ============================================================
-- IAQ READINGS
-- ============================================================

CREATE TABLE iaq_readings (
    time            TIMESTAMPTZ NOT NULL,
    node_id         UUID NOT NULL,                     -- references graph_nodes.id (space or zone)
    co2_ppm         NUMERIC(8, 2),
    temperature_c   NUMERIC(6, 2),
    humidity_pct    NUMERIC(5, 2),
    pm25_ugm3       NUMERIC(8, 2),
    pm10_ugm3       NUMERIC(8, 2),
    tvoc_ppb        NUMERIC(8, 2),
    iaq_score       NUMERIC(5, 2)
);
CREATE INDEX idx_iaq_node_time ON iaq_readings(node_id, time DESC);
```

---

## Alarms, Work Orders, and Bookings

```sql
-- ============================================================
-- ALARMS
-- ============================================================

CREATE TABLE alarm_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    condition_config JSONB NOT NULL,
    escalation_config JSONB,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alarm_defs_tenant ON alarm_definitions(tenant_id);

CREATE TABLE alarms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    alarm_definition_id UUID REFERENCES alarm_definitions(id),
    trigger_node_id UUID NOT NULL,                     -- graph_nodes.id (point or equipment that triggered)
    location_node_id UUID NOT NULL,                    -- graph_nodes.id (building/floor/zone)
    severity        VARCHAR(20) NOT NULL,
    state           VARCHAR(20) NOT NULL DEFAULT 'active',
    message         TEXT,
    trigger_data    JSONB,
    -- Impact analysis: auto-populated by graph traversal
    impact_analysis JSONB,
    -- Example (auto-computed from graph):
    -- {
    --   "affected_equipment": [{"id": "uuid", "name": "VAV-3-01"}, {"id": "uuid", "name": "VAV-3-02"}],
    --   "affected_spaces": [{"id": "uuid", "name": "Office 301"}, {"id": "uuid", "name": "Meeting Room 3A"}],
    --   "affected_occupant_count": 45,
    --   "upstream_equipment": [{"id": "uuid", "name": "Chiller-01"}],
    --   "traversal_depth": 3
    -- }
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES users(id),
    resolution_notes TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alarms_tenant_state ON alarms(tenant_id, state);
CREATE INDEX idx_alarms_trigger_node ON alarms(trigger_node_id);
CREATE INDEX idx_alarms_location ON alarms(location_node_id);
CREATE INDEX idx_alarms_created ON alarms(created_at DESC);

-- ============================================================
-- WORK ORDERS
-- ============================================================

CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    location_node_id UUID NOT NULL,                    -- graph_nodes.id
    equipment_node_id UUID,                            -- graph_nodes.id
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_tenant_status ON work_orders(tenant_id, status);
CREATE INDEX idx_wo_location ON work_orders(location_node_id);
CREATE INDEX idx_wo_equipment ON work_orders(equipment_node_id);
CREATE INDEX idx_wo_assigned ON work_orders(assigned_to) WHERE status IN ('assigned', 'in_progress');

-- ============================================================
-- SPACE BOOKINGS
-- ============================================================

CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    space_node_id   UUID NOT NULL,                     -- graph_nodes.id (space node)
    booked_by       UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(500),
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'confirmed',
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT no_booking_overlap EXCLUDE USING GIST (
        space_node_id WITH =,
        tstzrange(starts_at, ends_at) WITH &&
    ) WHERE (status = 'confirmed')
);
CREATE INDEX idx_bookings_space_time ON bookings(space_node_id, starts_at);
CREATE INDEX idx_bookings_user ON bookings(booked_by);
```

---

## Data Sources and AI

```sql
-- ============================================================
-- DATA SOURCES
-- ============================================================

CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    building_node_id UUID NOT NULL,                    -- graph_nodes.id (building node)
    name            VARCHAR(255) NOT NULL,
    protocol        VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'configured',
    config          JSONB NOT NULL,
    last_seen_at    TIMESTAMPTZ,
    error_info      JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ds_building ON data_sources(building_node_id);

-- Point-to-data-source mapping
CREATE TABLE point_source_map (
    point_node_id   UUID NOT NULL,                     -- graph_nodes.id (point node)
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    source_address  VARCHAR(500) NOT NULL,
    transform_expr  VARCHAR(500),
    PRIMARY KEY (point_node_id, data_source_id)
);

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
    -- Graph scope: which subgraph does this model operate on?
    scope           JSONB NOT NULL,
    -- Example:
    -- {
    --   "root_node_id": "building-uuid",
    --   "node_types": ["AHU", "VAV", "Zone_Air_Temperature_Sensor"],
    --   "edge_types": ["feeds", "hasPoint", "servesZone"],
    --   "traversal_depth": 5
    -- }
    config          JSONB NOT NULL,
    metrics         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    model_id        UUID NOT NULL REFERENCES ai_models(id),
    target_node_id  UUID NOT NULL,                     -- graph_nodes.id (equipment or building)
    recommendation_type VARCHAR(100) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    confidence      NUMERIC(5, 4),
    impact          JSONB,
    -- Graph-aware impact: which nodes are affected?
    affected_nodes  JSONB,
    -- Example:
    -- [
    --   {"node_id": "uuid", "name": "VAV-3-01", "type": "VAV", "effect": "setpoint_change"},
    --   {"node_id": "uuid", "name": "Office 301", "type": "Space", "effect": "temperature_change"}
    -- ]
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    actioned_by     UUID REFERENCES users(id),
    actioned_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_recs_tenant_status ON ai_recommendations(tenant_id, status);
CREATE INDEX idx_ai_recs_target ON ai_recommendations(target_node_id);

-- ============================================================
-- CARBON FACTORS
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
CREATE INDEX idx_carbon_lookup ON carbon_factors(country_code, energy_type, valid_from);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,             -- 'graph_node', 'graph_edge', 'alarm', 'work_order', etc.
    resource_id     UUID,
    changes         JSONB,
    context         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Graph Analytics Views

```sql
-- ============================================================
-- MATERIALIZED VIEWS FOR COMMON GRAPH ANALYTICS
-- ============================================================

-- Building topology summary: equipment count by type per building
CREATE MATERIALIZED VIEW mv_building_equipment_summary AS
SELECT
    b.id AS building_id,
    b.name AS building_name,
    b.tenant_id,
    e.node_type AS equipment_type,
    COUNT(*) AS equipment_count,
    COUNT(*) FILTER (WHERE e.status = 'active') AS active_count
FROM graph_nodes b
JOIN graph_edges ge ON ge.source_id = b.id
    AND ge.edge_type = 'hasPart'
    AND ge.valid_to IS NULL
JOIN graph_nodes e ON e.id = ge.target_id
    AND e.category = 'equipment'
WHERE b.node_type = 'Building'
GROUP BY b.id, b.name, b.tenant_id, e.node_type;

CREATE UNIQUE INDEX idx_mv_bes ON mv_building_equipment_summary(building_id, equipment_type);

-- Space-to-equipment mapping: which equipment serves each space?
CREATE MATERIALIZED VIEW mv_space_equipment AS
WITH RECURSIVE space_equipment AS (
    -- Direct: equipment located in space
    SELECT
        s.id AS space_id,
        s.name AS space_name,
        e.id AS equipment_id,
        e.name AS equipment_name,
        e.node_type AS equipment_type,
        'direct' AS relationship
    FROM graph_nodes s
    JOIN graph_edges ge ON ge.target_id = s.id
        AND ge.edge_type = 'hasLocation'
        AND ge.valid_to IS NULL
    JOIN graph_nodes e ON e.id = ge.source_id
        AND e.category = 'equipment'
    WHERE s.node_type = 'Space'

    UNION

    -- Via zone: equipment serves zone that contains space
    SELECT
        s.id AS space_id,
        s.name AS space_name,
        e.id AS equipment_id,
        e.name AS equipment_name,
        e.node_type AS equipment_type,
        'via_zone' AS relationship
    FROM graph_nodes s
    JOIN graph_edges cs ON cs.target_id = s.id
        AND cs.edge_type = 'containsSpace'
        AND cs.valid_to IS NULL
    JOIN graph_edges sz ON sz.target_id = cs.source_id
        AND sz.edge_type = 'servesZone'
        AND sz.valid_to IS NULL
    JOIN graph_nodes e ON e.id = sz.source_id
        AND e.category = 'equipment'
    WHERE s.node_type = 'Space'
)
SELECT DISTINCT * FROM space_equipment;

CREATE INDEX idx_mv_se_space ON mv_space_equipment(space_id);
CREATE INDEX idx_mv_se_equipment ON mv_space_equipment(equipment_id);

-- ============================================================
-- GRAPH STATISTICS (for monitoring graph health)
-- ============================================================

CREATE MATERIALIZED VIEW mv_graph_statistics AS
SELECT
    tenant_id,
    (SELECT COUNT(*) FROM graph_nodes gn2 WHERE gn2.tenant_id = t.id) AS total_nodes,
    (SELECT COUNT(*) FROM graph_edges ge2 WHERE ge2.tenant_id = t.id AND ge2.valid_to IS NULL) AS active_edges,
    (SELECT COUNT(DISTINCT node_type) FROM graph_nodes gn3 WHERE gn3.tenant_id = t.id) AS node_type_count,
    (SELECT COUNT(DISTINCT edge_type) FROM graph_edges ge3 WHERE ge3.tenant_id = t.id AND ge3.valid_to IS NULL) AS edge_type_count
FROM tenants t;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges (core of the model) |
| Multi-tenancy & Identity | 4 | tenants, users, roles, user_roles |
| Telemetry | 3 | telemetry, telemetry_hourly, telemetry_daily |
| Energy | 1 | energy_readings |
| Occupancy & IAQ | 2 | occupancy_readings, iaq_readings |
| Alarms | 2 | alarm_definitions, alarms |
| Work Orders | 1 | work_orders |
| Bookings | 1 | bookings |
| Data Sources | 2 | data_sources, point_source_map |
| AI | 2 | ai_models, ai_recommendations |
| Reference Data | 1 | carbon_factors |
| Audit | 1 | audit_log |
| Materialized Views | 3 | mv_building_equipment_summary, mv_space_equipment, mv_graph_statistics |
| **Total** | **25** | Plus 3 materialized views |

---

## Key Design Decisions

1. **Two-table graph core rather than dedicated graph database.** The `graph_nodes` / `graph_edges` pattern is implemented in PostgreSQL rather than requiring a separate graph database like Neo4j. This keeps the deployment architecture simple (single database), supports transactional consistency between graph and operational data, and leverages PostgreSQL's mature ecosystem. For very large deployments (100,000+ nodes), Apache AGE (PostgreSQL extension) or a dedicated graph database can be adopted with the same logical schema.

2. **Edge types directly mirror Brick Schema relationships.** The `edge_type` vocabulary (`feeds`, `hasPart`, `hasPoint`, `isLocationOf`, `servesZone`) is taken directly from the Brick Schema relationship ontology. This means the graph can be exported to Brick RDF triples or Brick SPARQL queries without semantic translation -- the PostgreSQL graph IS the Brick model, just stored in relational form.

3. **Temporal edges with `valid_from` / `valid_to`.** Every edge carries validity timestamps, enabling temporal graph queries ("what was the equipment topology in January?") and soft-delete of relationships without losing history. When a VAV box is moved to a different AHU, the old `feeds` edge gets a `valid_to` timestamp and a new edge is created.

4. **Impact analysis auto-populated on alarms.** When an alarm triggers, the application traverses the graph from the triggering point/equipment to discover all downstream spaces, zones, and equipment that may be affected. This pre-computed `impact_analysis` JSONB on the alarm record gives operators immediate context without running graph queries in real time.

5. **AI model scope defined by graph subgraph.** Each AI model's `scope` field specifies which portion of the building graph it operates on (root node, node types, edge types, traversal depth). This enables the optimization engine to automatically discover all sensors and equipment relevant to a particular HVAC subsystem by traversing the graph, rather than requiring manual configuration of input/output points.

6. **All entity types unified in graph_nodes.** Sites, buildings, floors, zones, spaces, equipment, sensors, setpoints, and meters are all `graph_nodes` with different `node_type` values. This eliminates the need for separate entity tables and enables uniform graph traversal across spatial and equipment hierarchies. The `properties` JSONB field accommodates type-specific attributes.

7. **Telemetry tables reference graph_nodes directly.** Time-series tables (`telemetry`, `energy_readings`, `occupancy_readings`, `iaq_readings`) reference `graph_nodes.id` directly rather than using intermediate entity tables. This keeps the query path short: "get readings for this point node" requires no JOINs beyond the telemetry table itself.

8. **Materialized views bridge graph and reporting.** Common graph analytics (equipment counts per building, space-to-equipment mapping) are pre-computed in materialized views refreshed periodically. This avoids expensive recursive CTE execution on every dashboard load while keeping the underlying data model graph-native.

9. **RBAC scope via graph node references.** User role scoping uses `scope_id` referencing `graph_nodes.id`. A user can be scoped to a building node, and the application traverses the graph to determine which child entities (floors, equipment, points) they can access. This is more flexible than separate scope tables for each entity type.

10. **Edge properties carry relationship metadata.** The `properties` JSONB on edges stores relationship-specific data (medium flowing through a pipe, energy type for a meter connection, lease terms for occupancy). This enriches traversal results without requiring lookups to separate tables, following the property graph model that Brick Schema and NGSI-LD are built on.
