# Data Model Suggestion 4: Graph + Time-Series Polyglot Architecture (Neo4j + TimescaleDB + PostGIS)

> Project: Infrastructure Asset Registry (Candidate #439)
> Generated: 2026-05-25

## Overview

This model uses a polyglot persistence architecture that pairs each data concern with the storage engine optimally suited to it:

- **Neo4j (Graph Database)**: Models the asset hierarchy, network topology, and dependency relationships between infrastructure assets. Graph traversal answers questions like "If this water main fails, which downstream assets are affected?" and "What is the shortest route that avoids all bridges with condition below 40?"
- **TimescaleDB (Time-Series Database)**: Stores condition measurements, sensor readings, and inspection scores as time-series data with automatic compression, continuous aggregates, and retention policies. Optimized for queries like "Show me the condition trend for all arterial roads over the last 10 years."
- **PostGIS (Spatial Database)**: Remains the spatial data engine for GIS queries, map rendering, and OGC API Features compliance.

This approach directly addresses three characteristics of infrastructure asset management that relational models handle poorly:

1. **Network topology**: Roads connect to intersections connect to bridges connect to pipe crossings. This is fundamentally a graph problem.
2. **Condition over time**: Decades of inspection scores at varying intervals form time-series data best served by purpose-built engines.
3. **Impact analysis**: "What happens if this asset fails?" requires multi-hop graph traversal that would need expensive recursive CTEs in SQL.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                       Application Layer                       │
│   (REST/GraphQL API, OGC API Features, Mobile App, AI/ML)    │
└───────────┬──────────────────┬──────────────────┬────────────┘
            │                  │                  │
   ┌────────▼────────┐ ┌──────▼───────┐ ┌────────▼────────┐
   │     Neo4j       │ │  TimescaleDB │ │    PostGIS      │
   │  (Graph Store)  │ │ (Time-Series)│ │  (Spatial +     │
   │                 │ │              │ │   Relational)   │
   │ - Asset nodes   │ │ - Condition  │ │ - Geometries    │
   │ - Relationships │ │   scores     │ │ - Work orders   │
   │ - Hierarchy     │ │ - Sensor     │ │ - Financials    │
   │ - Dependencies  │ │   readings   │ │ - Users/Orgs    │
   │ - Impact paths  │ │ - Metrics    │ │ - Audit trail   │
   │ - Network       │ │ - Trends     │ │ - Documents     │
   │   topology      │ │ - Forecasts  │ │                 │
   └─────────────────┘ └──────────────┘ └─────────────────┘
            │                  │                  │
   ┌────────▼──────────────────▼──────────────────▼────────────┐
   │              Event Bus (Kafka / NATS)                      │
   │     (Synchronizes state across the three databases)        │
   └───────────────────────────────────────────────────────────┘
```

## Design Principles

1. **Each database does what it does best**: Graph for relationships and traversal, time-series for temporal aggregation and compression, spatial-relational for GIS and transactional workloads.
2. **Asset ID is the universal join key**: Every asset has a UUID that appears as a node property in Neo4j, a tag in TimescaleDB, and a primary key in PostGIS.
3. **Event-driven synchronization**: Changes in any database are published to an event bus. Consumers update the other databases to maintain consistency.
4. **Read-optimized denormalization**: Each database stores the data it needs to answer its queries without cross-database joins at query time.
5. **PostGIS as the system of record**: For transactional operations (work orders, financial records, user management), PostGIS/PostgreSQL is the authoritative source.

---

## Neo4j Graph Model

### Node Types

```cypher
// ===== ASSET NODES =====

// Base asset node -- every infrastructure asset
CREATE (a:Asset {
    asset_id: "uuid-here",
    organization_id: "uuid-here",
    asset_number: "RD-2024-0142",
    name: "Main Street - Elm to Oak",
    asset_class: "ROAD_ARTERIAL",
    asset_category: "horizontal",
    material: "asphalt",
    installation_date: date("1998-06-15"),
    status: "active",
    criticality: 4,
    current_condition_score: 62.5,
    current_condition_rating: "fair",
    replacement_cost: 1385000.00
})

// Asset type labels (multi-label nodes)
// An asset node carries both :Asset and its type label
CREATE (r:Asset:Road {
    asset_id: "...",
    road_class: "arterial",
    surface_type: "asphalt",
    lane_count: 4,
    traffic_volume_aadt: 12500,
    speed_limit_kph: 56
})

CREATE (b:Asset:Bridge {
    asset_id: "...",
    nbi_structure_number: "NY-1234567",
    bridge_type: "beam",
    span_count: 3,
    load_rating_tons: 36.0,
    scour_critical: false,
    fracture_critical: false,
    year_built: 1962
})

CREATE (p:Asset:Pipe {
    asset_id: "...",
    pipe_system: "sanitary_sewer",
    pipe_material: "vitrified_clay",
    nominal_diameter_mm: 300,
    flow_direction: "downstream"
})

CREATE (s:Asset:Signal {
    asset_id: "...",
    signal_type: "traffic_signal",
    controller_type: "actuated",
    is_ada_compliant: true
})

CREATE (f:Asset:Facility {
    asset_id: "...",
    facility_type: "pump_station",
    floor_count: 1,
    year_built: 1985
})

CREATE (c:Asset:Culvert {
    asset_id: "...",
    culvert_type: "pipe",
    shape: "circular",
    span_m: 1.2
})

// ===== ORGANIZATIONAL NODES =====

CREATE (o:Organization {
    org_id: "...",
    name: "City of Springfield",
    org_type: "municipality"
})

CREATE (d:Department {
    dept_id: "...",
    name: "Public Works",
    code: "PW"
})

// ===== NETWORK NODES =====

CREATE (n:NetworkNode {
    node_id: "...",
    node_type: "intersection",  // intersection, junction, manhole, valve, meter
    name: "Main St / Oak St",
    elevation_m: 42.5
})

CREATE (z:Zone {
    zone_id: "...",
    zone_type: "pressure_zone",  // pressure_zone, drainage_basin, maintenance_district, council_district
    name: "Pressure Zone 3"
})
```

### Relationship Types

```cypher
// ===== ASSET HIERARCHY =====
// Network > Segment > Component relationships

// Road network contains road segments
CREATE (network:Asset:RoadNetwork)-[:CONTAINS]->(segment:Asset:Road)

// Bridge has structural components
CREATE (bridge:Asset:Bridge)-[:HAS_COMPONENT {component_type: "deck"}]->(deck:Asset:BridgeDeck)
CREATE (bridge:Asset:Bridge)-[:HAS_COMPONENT {component_type: "superstructure"}]->(super:Asset:BridgeSuperstructure)
CREATE (bridge:Asset:Bridge)-[:HAS_COMPONENT {component_type: "substructure"}]->(sub:Asset:BridgeSubstructure)

// Building contains systems
CREATE (building:Asset:Facility)-[:HAS_COMPONENT {component_type: "hvac"}]->(hvac:Asset:HVACSystem)
CREATE (building:Asset:Facility)-[:HAS_COMPONENT {component_type: "roof"}]->(roof:Asset:Roof)


// ===== NETWORK TOPOLOGY =====
// How assets connect to form infrastructure networks

// Road connectivity (road segments connect at intersections)
CREATE (road1:Asset:Road)-[:CONNECTS_TO {direction: "eastbound", distance_m: 342.5}]->(intersection:NetworkNode)
CREATE (intersection:NetworkNode)-[:CONNECTS_TO {direction: "northbound", distance_m: 215.0}]->(road2:Asset:Road)

// Pipe network topology (flow direction matters)
CREATE (pipe1:Asset:Pipe)-[:FLOWS_INTO {connection_type: "direct"}]->(manhole:NetworkNode)
CREATE (manhole:NetworkNode)-[:FLOWS_INTO {connection_type: "direct"}]->(pipe2:Asset:Pipe)

// Bridge carries road over obstacle
CREATE (bridge:Asset:Bridge)-[:CARRIES]->(road:Asset:Road)
CREATE (bridge:Asset:Bridge)-[:CROSSES_OVER]->(river:Asset)
CREATE (bridge:Asset:Bridge)-[:CROSSES_OVER]->(railroad:Asset)

// Culvert connects to drainage
CREATE (culvert:Asset:Culvert)-[:DRAINS_INTO]->(pipe:Asset:Pipe)
CREATE (culvert:Asset:Culvert)-[:UNDER]->(road:Asset:Road)

// Signal controls intersection
CREATE (signal:Asset:Signal)-[:CONTROLS]->(intersection:NetworkNode)

// Pump station feeds into pipe network
CREATE (pump:Asset:Facility)-[:FEEDS_INTO]->(pipe:Asset:Pipe)


// ===== SPATIAL AND ORGANIZATIONAL =====

// Zone membership
CREATE (asset:Asset)-[:IN_ZONE]->(zone:Zone)
CREATE (asset:Asset)-[:MANAGED_BY]->(dept:Department)
CREATE (dept:Department)-[:BELONGS_TO]->(org:Organization)

// Asset co-location (assets that share physical space)
CREATE (road:Asset:Road)-[:RUNS_ALONG]->(pipe:Asset:Pipe)
CREATE (signal:Asset:Signal)-[:LOCATED_AT]->(intersection:NetworkNode)


// ===== DEPENDENCY AND IMPACT =====

// Service dependencies (failure propagation paths)
CREATE (upstream_pipe:Asset:Pipe)-[:SUPPLIES {service: "water", criticality: "high"}]->(downstream_pipe:Asset:Pipe)
CREATE (pump_station:Asset:Facility)-[:POWERS {service: "water_pressure"}]->(zone:Zone)
CREATE (treatment_plant:Asset:Facility)-[:TREATS_FLOW_FROM]->(trunk_main:Asset:Pipe)

// Redundancy relationships
CREATE (pipe1:Asset:Pipe)-[:REDUNDANT_WITH {failover_capacity_pct: 60}]->(pipe2:Asset:Pipe)
CREATE (road1:Asset:Road)-[:ALTERNATE_ROUTE {detour_distance_m: 1200}]->(road2:Asset:Road)


// ===== CAPITAL PROJECT RELATIONSHIPS =====

CREATE (project:CapitalProject)-[:TREATS]->(asset:Asset)
CREATE (project:CapitalProject)-[:FUNDED_BY]->(fund:FundingSource)
CREATE (project:CapitalProject)-[:DEPENDS_ON]->(prereq:CapitalProject)
```

### Graph Queries for Infrastructure Analysis

```cypher
// 1. IMPACT ANALYSIS: If a water main fails, what downstream assets are affected?
MATCH (failed:Asset:Pipe {asset_id: $pipe_id})-[:SUPPLIES|FLOWS_INTO*1..10]->(downstream)
RETURN downstream.asset_number, downstream.name, downstream.asset_class,
       downstream.criticality, downstream.current_condition_score,
       length(shortestPath((failed)-[:SUPPLIES|FLOWS_INTO*]->(downstream))) AS hops_away
ORDER BY downstream.criticality DESC, hops_away ASC

// 2. NETWORK ROUTING: Find all roads from point A to point B avoiding bridges with condition below 40
MATCH path = shortestPath(
    (start:NetworkNode {name: $start_intersection})-[:CONNECTS_TO*]-(end:NetworkNode {name: $end_intersection})
)
WHERE ALL(n IN nodes(path) WHERE 
    NOT (n:Bridge) OR n.current_condition_score >= 40
)
RETURN path, reduce(total = 0, r IN relationships(path) | total + r.distance_m) AS total_distance_m

// 3. HIERARCHY TRAVERSAL: Get all components of a bridge and their conditions
MATCH (bridge:Asset:Bridge {asset_id: $bridge_id})-[:HAS_COMPONENT*1..3]->(component)
RETURN component.name, component.asset_class, component.current_condition_score,
       component.criticality
ORDER BY component.current_condition_score ASC

// 4. CO-LOCATED ASSETS: Find all assets that share space with a road being reconstructed
MATCH (road:Asset:Road {asset_id: $road_id})<-[:RUNS_ALONG|UNDER|CROSSES_OVER|LOCATED_AT]-(adjacent)
RETURN adjacent.asset_number, adjacent.name, adjacent.asset_class,
       adjacent.current_condition_score, adjacent.status
// This answers: "When we reconstruct Main Street, what pipes, culverts, and signals are we going to encounter?"

// 5. REDUNDANCY ANALYSIS: Find single points of failure in the water network
MATCH (source:Asset:Facility {facility_type: "treatment_plant"})-[:SUPPLIES|FLOWS_INTO*]->(consumer:Zone)
WITH source, consumer, count(*) as path_count
WHERE path_count = 1
MATCH path = (source)-[:SUPPLIES|FLOWS_INTO*]->(consumer)
UNWIND nodes(path) AS critical_node
WHERE critical_node:Asset
RETURN DISTINCT critical_node.asset_number, critical_node.name, critical_node.asset_class,
       critical_node.current_condition_score, critical_node.criticality
ORDER BY critical_node.criticality DESC

// 6. CAPITAL PROJECT SEQUENCING: Find project dependencies
MATCH (project:CapitalProject {project_number: $project_number})-[:DEPENDS_ON*0..5]->(dependency)
RETURN dependency.project_number, dependency.name, dependency.status,
       dependency.planned_start_date, dependency.estimated_cost,
       length(shortestPath((project)-[:DEPENDS_ON*]->(dependency))) AS dependency_depth
ORDER BY dependency_depth ASC

// 7. RISK PROPAGATION: Score assets by downstream impact if they fail
MATCH (asset:Asset)-[:SUPPLIES|FLOWS_INTO|CONNECTS_TO*1..5]->(affected)
WHERE asset.organization_id = $org_id AND asset.status = 'active'
WITH asset, count(affected) AS downstream_count,
     sum(affected.replacement_cost) AS downstream_value,
     max(affected.criticality) AS max_downstream_criticality
RETURN asset.asset_number, asset.name, asset.asset_class,
       asset.current_condition_score, asset.criticality,
       downstream_count, downstream_value, max_downstream_criticality,
       (1.0 - asset.current_condition_score/100.0) * downstream_value AS risk_exposure
ORDER BY risk_exposure DESC
LIMIT 50

// 8. NEIGHBORHOOD ANALYSIS: Find all assets within 2 hops of a failing asset
MATCH (center:Asset {asset_id: $asset_id})-[*1..2]-(neighbor:Asset)
RETURN DISTINCT neighbor.asset_number, neighbor.name, neighbor.asset_class,
       neighbor.current_condition_score, neighbor.status
```

---

## TimescaleDB Time-Series Model

TimescaleDB extends PostgreSQL with hypertables, continuous aggregates, and compression for time-series data. It is used here for all temporal measurements: condition scores, sensor readings, and operational metrics.

### Schema

```sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- ===== CONDITION SCORE TIME SERIES =====
-- Every condition score ever recorded for every asset

CREATE TABLE ts_condition_scores (
    time                TIMESTAMPTZ NOT NULL,
    asset_id            UUID NOT NULL,
    organization_id     UUID NOT NULL,
    framework_code      VARCHAR(30) NOT NULL,
    score               NUMERIC(8,2) NOT NULL,
    rating_band         VARCHAR(50),
    inspection_type     VARCHAR(30),
    scoring_method      VARCHAR(30), -- 'manual', 'ai_assisted', 'sensor_based'
    confidence          NUMERIC(3,2),
    inspector_id        UUID,
    inspection_id       UUID
);

-- Convert to hypertable (time-partitioned)
SELECT create_hypertable('ts_condition_scores', 'time',
    chunk_time_interval => INTERVAL '1 year');

-- Indexes for common query patterns
CREATE INDEX idx_ts_cond_asset ON ts_condition_scores (asset_id, time DESC);
CREATE INDEX idx_ts_cond_org ON ts_condition_scores (organization_id, time DESC);
CREATE INDEX idx_ts_cond_framework ON ts_condition_scores (framework_code, organization_id, time DESC);


-- ===== SENSOR READINGS =====
-- IoT sensor data from smart infrastructure (future capability)

CREATE TABLE ts_sensor_readings (
    time                TIMESTAMPTZ NOT NULL,
    asset_id            UUID NOT NULL,
    sensor_id           VARCHAR(100) NOT NULL,
    sensor_type         VARCHAR(50) NOT NULL, -- 'vibration', 'strain', 'flow', 'level', 'temperature', 'tilt', 'moisture'
    value               DOUBLE PRECISION NOT NULL,
    unit                VARCHAR(30) NOT NULL,
    quality_flag        VARCHAR(10) DEFAULT 'good' -- 'good', 'suspect', 'bad', 'missing'
);

SELECT create_hypertable('ts_sensor_readings', 'time',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_ts_sensor_asset ON ts_sensor_readings (asset_id, sensor_type, time DESC);
CREATE INDEX idx_ts_sensor_id ON ts_sensor_readings (sensor_id, time DESC);


-- ===== DETERIORATION PREDICTIONS =====
-- AI model predictions stored as time series for comparison with actuals

CREATE TABLE ts_deterioration_predictions (
    time                TIMESTAMPTZ NOT NULL, -- When the prediction was made
    asset_id            UUID NOT NULL,
    model_id            VARCHAR(100) NOT NULL,
    model_version       VARCHAR(30) NOT NULL,
    prediction_date     DATE NOT NULL,       -- The date being predicted
    predicted_score     NUMERIC(8,2) NOT NULL,
    prediction_interval_lower NUMERIC(8,2),
    prediction_interval_upper NUMERIC(8,2),
    confidence          NUMERIC(3,2)
);

SELECT create_hypertable('ts_deterioration_predictions', 'time',
    chunk_time_interval => INTERVAL '1 year');


-- ===== WORK ORDER METRICS =====
-- Time series of work order activity for operational dashboards

CREATE TABLE ts_work_order_metrics (
    time                TIMESTAMPTZ NOT NULL,
    organization_id     UUID NOT NULL,
    department_id       UUID,
    work_type           VARCHAR(30),
    priority            VARCHAR(20),
    metric_name         VARCHAR(50) NOT NULL, -- 'opened', 'completed', 'response_hours', 'cost'
    metric_value        DOUBLE PRECISION NOT NULL
);

SELECT create_hypertable('ts_work_order_metrics', 'time',
    chunk_time_interval => INTERVAL '1 month');


-- ===== ASSET FINANCIAL METRICS =====
-- Time series of financial data for capital planning

CREATE TABLE ts_financial_metrics (
    time                TIMESTAMPTZ NOT NULL,
    organization_id     UUID NOT NULL,
    asset_class_code    VARCHAR(30),
    metric_name         VARCHAR(50) NOT NULL, -- 'replacement_cost', 'backlog', 'maintenance_spend', 'capital_spend'
    metric_value        DOUBLE PRECISION NOT NULL,
    fiscal_year         INTEGER
);

SELECT create_hypertable('ts_financial_metrics', 'time',
    chunk_time_interval => INTERVAL '1 year');
```

### Continuous Aggregates

```sql
-- Pre-computed condition averages by asset class per year
-- (Automatically updated as new data arrives)

CREATE MATERIALIZED VIEW cagg_condition_by_class_yearly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 year', time) AS bucket,
    organization_id,
    framework_code,
    COUNT(*) AS inspection_count,
    AVG(score) AS avg_score,
    MIN(score) AS min_score,
    MAX(score) AS max_score,
    percentile_cont(0.25) WITHIN GROUP (ORDER BY score) AS p25_score,
    percentile_cont(0.50) WITHIN GROUP (ORDER BY score) AS median_score,
    percentile_cont(0.75) WITHIN GROUP (ORDER BY score) AS p75_score
FROM ts_condition_scores
GROUP BY bucket, organization_id, framework_code;

-- Refresh policy: update every day
SELECT add_continuous_aggregate_policy('cagg_condition_by_class_yearly',
    start_offset => INTERVAL '2 years',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');


-- Sensor reading aggregates (hourly, daily, monthly)
CREATE MATERIALIZED VIEW cagg_sensor_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    asset_id,
    sensor_id,
    sensor_type,
    AVG(value) AS avg_value,
    MIN(value) AS min_value,
    MAX(value) AS max_value,
    COUNT(*) AS reading_count
FROM ts_sensor_readings
WHERE quality_flag = 'good'
GROUP BY bucket, asset_id, sensor_id, sensor_type;

SELECT add_continuous_aggregate_policy('cagg_sensor_hourly',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');


-- Work order metrics weekly rollup
CREATE MATERIALIZED VIEW cagg_work_orders_weekly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 week', time) AS bucket,
    organization_id,
    work_type,
    SUM(metric_value) FILTER (WHERE metric_name = 'opened') AS orders_opened,
    SUM(metric_value) FILTER (WHERE metric_name = 'completed') AS orders_completed,
    AVG(metric_value) FILTER (WHERE metric_name = 'response_hours') AS avg_response_hours,
    SUM(metric_value) FILTER (WHERE metric_name = 'cost') AS total_cost
FROM ts_work_order_metrics
GROUP BY bucket, organization_id, work_type;
```

### Compression and Retention

```sql
-- Enable compression on older condition data
ALTER TABLE ts_condition_scores SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'asset_id, organization_id, framework_code',
    timescaledb.compress_orderby = 'time DESC'
);

-- Compress data older than 2 years (still queryable, just compressed)
SELECT add_compression_policy('ts_condition_scores', INTERVAL '2 years');

-- Sensor data: compress after 7 days (high volume)
ALTER TABLE ts_sensor_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'asset_id, sensor_id, sensor_type',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('ts_sensor_readings', INTERVAL '7 days');

-- Retain raw sensor data for 2 years, then drop
-- (Continuous aggregates preserve summaries indefinitely)
SELECT add_retention_policy('ts_sensor_readings', INTERVAL '2 years');

-- Deterioration predictions: compress after 1 year
ALTER TABLE ts_deterioration_predictions SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'asset_id, model_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('ts_deterioration_predictions', INTERVAL '1 year');
```

### Time-Series Queries

```sql
-- Condition trend for an asset over its lifetime
SELECT time, score, rating_band, scoring_method, confidence
FROM ts_condition_scores
WHERE asset_id = :asset_id
ORDER BY time ASC;

-- Average condition by year for all arterial roads in an organization
SELECT bucket AS year,
       avg_score,
       median_score,
       inspection_count
FROM cagg_condition_by_class_yearly
WHERE organization_id = :org_id
  AND framework_code = 'PCI'
ORDER BY bucket ASC;

-- Deterioration rate calculation: annual decline in condition
SELECT asset_id,
       time_bucket('1 year', time) AS year,
       first(score, time) AS score_start,
       last(score, time) AS score_end,
       last(score, time) - first(score, time) AS annual_change
FROM ts_condition_scores
WHERE organization_id = :org_id
  AND framework_code = 'PCI'
  AND time >= now() - INTERVAL '10 years'
GROUP BY asset_id, time_bucket('1 year', time)
ORDER BY asset_id, year;

-- Sensor anomaly detection: readings outside 3 standard deviations
WITH stats AS (
    SELECT sensor_id,
           AVG(value) AS mean_val,
           STDDEV(value) AS std_val
    FROM ts_sensor_readings
    WHERE asset_id = :asset_id
      AND sensor_type = 'vibration'
      AND time >= now() - INTERVAL '30 days'
    GROUP BY sensor_id
)
SELECT r.time, r.sensor_id, r.value, s.mean_val, s.std_val,
       ABS(r.value - s.mean_val) / NULLIF(s.std_val, 0) AS z_score
FROM ts_sensor_readings r
JOIN stats s ON r.sensor_id = s.sensor_id
WHERE r.asset_id = :asset_id
  AND r.sensor_type = 'vibration'
  AND r.time >= now() - INTERVAL '30 days'
  AND ABS(r.value - s.mean_val) > 3 * s.std_val
ORDER BY r.time DESC;

-- Compare predicted vs actual deterioration
SELECT p.prediction_date,
       p.predicted_score,
       p.prediction_interval_lower,
       p.prediction_interval_upper,
       a.score AS actual_score,
       ABS(p.predicted_score - a.score) AS prediction_error
FROM ts_deterioration_predictions p
LEFT JOIN ts_condition_scores a
    ON p.asset_id = a.asset_id
    AND a.time::date = p.prediction_date
WHERE p.asset_id = :asset_id
  AND p.model_id = :model_id
ORDER BY p.prediction_date ASC;

-- GASB 34 modified approach: 3-year condition assessment trend by asset class
SELECT
    time_bucket('1 year', time) AS assessment_year,
    framework_code,
    COUNT(DISTINCT asset_id) AS assets_assessed,
    AVG(score) AS avg_condition,
    percentile_cont(0.10) WITHIN GROUP (ORDER BY score) AS p10_condition,
    COUNT(*) FILTER (WHERE score >= :target_condition) AS assets_meeting_target,
    COUNT(*) AS total_assessments,
    ROUND(100.0 * COUNT(*) FILTER (WHERE score >= :target_condition) / COUNT(*), 1) AS pct_meeting_target
FROM ts_condition_scores
WHERE organization_id = :org_id
  AND time >= now() - INTERVAL '3 years'
GROUP BY time_bucket('1 year', time), framework_code
ORDER BY assessment_year DESC;
```

---

## PostGIS Relational Model (System of Record)

PostGIS handles the transactional, spatial, and relational data that does not fit naturally into graph or time-series stores.

```sql
-- Core tables remain in PostGIS (abbreviated -- see Suggestion 1 for full detail)

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    parent_id       UUID REFERENCES departments(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    user_role       VARCHAR(30) NOT NULL,
    department_id   UUID REFERENCES departments(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Assets: master record with spatial data
CREATE TABLE assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    asset_number        VARCHAR(50) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    asset_class_code    VARCHAR(30) NOT NULL,
    department_id       UUID REFERENCES departments(id),
    parent_asset_id     UUID REFERENCES assets(id),
    
    -- Spatial (PostGIS native)
    geom_point          GEOMETRY(Point, 4326),
    geom_line           GEOMETRY(LineString, 4326),
    geom_polygon        GEOMETRY(Polygon, 4326),
    
    -- Core attributes
    material            VARCHAR(100),
    installation_date   DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    
    -- Denormalized from Neo4j (for fast SQL queries without graph hop)
    current_condition_score NUMERIC(5,2),
    current_condition_date  DATE,
    criticality_rating  INTEGER,
    
    -- Type-specific attributes (JSONB for flexibility)
    attributes          JSONB NOT NULL DEFAULT '{}'::jsonb,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),
    
    UNIQUE (organization_id, asset_number)
);

CREATE INDEX idx_assets_geom_point ON assets USING GIST (geom_point);
CREATE INDEX idx_assets_geom_line ON assets USING GIST (geom_line);
CREATE INDEX idx_assets_geom_polygon ON assets USING GIST (geom_polygon);
CREATE INDEX idx_assets_org ON assets (organization_id, asset_class_code);
CREATE INDEX idx_assets_status ON assets (organization_id, status);

-- Work orders (transactional -- stays in PostGIS)
CREATE TABLE work_orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    work_order_number   VARCHAR(30) NOT NULL,
    asset_id            UUID NOT NULL REFERENCES assets(id),
    work_type           VARCHAR(30) NOT NULL,
    priority            VARCHAR(20) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    assigned_to         UUID REFERENCES users(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'open',
    scheduled_start     TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    total_cost          NUMERIC(14,2),
    work_details        JSONB NOT NULL DEFAULT '{}'::jsonb,
    geom_point          GEOMETRY(Point, 4326),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, work_order_number)
);

-- Financial records (strict normalization for GASB 34)
CREATE TABLE asset_financials (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id            UUID NOT NULL REFERENCES assets(id),
    acquisition_cost    NUMERIC(16,2),
    replacement_cost    NUMERIC(16,2),
    net_book_value      NUMERIC(16,2),
    accumulated_depreciation NUMERIC(16,2),
    depreciation_method VARCHAR(30),
    useful_life_years   INTEGER,
    gl_asset_account    VARCHAR(30),
    fund_code           VARCHAR(30),
    financial_metadata  JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Inspections (master record -- detail scores go to TimescaleDB)
CREATE TABLE inspections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    asset_id            UUID NOT NULL REFERENCES assets(id),
    framework_code      VARCHAR(30) NOT NULL,
    inspection_type     VARCHAR(30) NOT NULL,
    inspection_date     DATE NOT NULL,
    inspected_by        UUID REFERENCES users(id),
    overall_score       NUMERIC(8,2),
    rating_band         VARCHAR(50),
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    inspection_data     JSONB NOT NULL DEFAULT '{}'::jsonb,
    ai_analysis         JSONB,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Citizen requests
CREATE TABLE citizen_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    request_number      VARCHAR(30) NOT NULL,
    category            VARCHAR(100) NOT NULL,
    description         TEXT NOT NULL,
    geom_point          GEOMETRY(Point, 4326),
    asset_id            UUID REFERENCES assets(id),
    work_order_id       UUID REFERENCES work_orders(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'submitted',
    request_metadata    JSONB NOT NULL DEFAULT '{}'::jsonb,
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, request_number)
);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL,
    changes         JSONB NOT NULL,
    user_id         UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);
```

---

## Cross-Database Synchronization

### Event-Driven Sync Architecture

```
┌─────────────┐    ┌──────────────┐    ┌───────────────┐
│  PostGIS    │───▶│  Event Bus   │───▶│    Neo4j      │
│  (Source of │    │  (Kafka /    │    │  (Graph       │
│   Record)   │    │   NATS)      │    │   Updater)    │
│             │    │              │    │               │
│ INSERT/     │    │ asset.created│    │ CREATE node   │
│ UPDATE/     │    │ asset.updated│    │ SET properties│
│ DELETE      │    │ wo.completed │    │ CREATE rels   │
└─────────────┘    │ inspection.  │    └───────────────┘
                   │   approved   │
                   │              │    ┌───────────────┐
                   │              │───▶│  TimescaleDB  │
                   │              │    │  (Time-Series │
                   │              │    │   Writer)     │
                   │              │    │               │
                   │              │    │ INSERT score  │
                   │              │    │ INSERT metric │
                   └──────────────┘    └───────────────┘
```

### Sync Event Types

```json
// Published when an asset is created or updated in PostGIS
{
    "event_type": "asset.created",
    "asset_id": "...",
    "organization_id": "...",
    "data": {
        "asset_number": "RD-2024-0142",
        "name": "Main Street - Elm to Oak",
        "asset_class_code": "ROAD_ARTERIAL",
        "attributes": {...},
        "parent_asset_id": "...",
        "geom": {...}
    }
}

// Published when an inspection is approved
{
    "event_type": "inspection.approved",
    "inspection_id": "...",
    "asset_id": "...",
    "data": {
        "framework_code": "PCI",
        "score": 62.5,
        "rating_band": "fair",
        "inspection_date": "2026-04-15"
    }
}

// Consumers:
// - Neo4j updater: Creates/updates Asset nodes, creates relationships
// - TimescaleDB writer: Inserts condition scores, computes metrics
// - Both update denormalized fields back in PostGIS (current_condition_score, etc.)
```

### Consistency Guarantees

- **PostGIS is the system of record** for all mutations. Commands always write to PostGIS first.
- **Neo4j and TimescaleDB are eventually consistent** read models, rebuilt from events.
- **Idempotent consumers**: Each sync consumer tracks its position in the event stream and can replay from any point.
- **Compensating actions**: If a sync fails, events are retried. Dead letter queues capture permanently failed events for manual intervention.
- **Health monitoring**: A sync lag dashboard tracks the delay between PostGIS writes and Neo4j/TimescaleDB updates.

---

## Pros and Cons

### Pros

1. **Natural Network Modeling**: Infrastructure IS a network. Roads connect at intersections, pipes flow through manholes, bridges carry roads over obstacles. Neo4j models these relationships natively with labeled, directed edges. Queries that require 5-level recursive CTEs in SQL become simple 1-line Cypher traversals.

2. **Impact Analysis in Milliseconds**: "If this water main fails, what is affected downstream?" traverses the graph in milliseconds even for networks with millions of nodes. In a relational database, this requires recursive CTEs with JOINs that scale poorly beyond 3-4 hops.

3. **Optimal Time-Series Storage**: TimescaleDB compresses decades of condition data by 90%+ compared to plain PostgreSQL tables. Continuous aggregates pre-compute annual/monthly/weekly summaries without manual materialized view maintenance.

4. **Sensor Data Ready**: The architecture natively supports IoT sensor ingestion (vibration sensors on bridges, flow meters on pipes, smart culvert level monitors) at high frequency without affecting the transactional database.

5. **Deterioration Analysis at Scale**: Time-series functions (`first()`, `last()`, `time_bucket()`, percentile functions) make deterioration rate calculations, trend analysis, and prediction accuracy measurement trivial queries rather than complex window function chains.

6. **Spatial Queries Remain Fast**: PostGIS handles all spatial queries (proximity search, intersection analysis, OGC API Features) without compromise. Geometry data stays in its optimal storage engine.

7. **Independent Scaling**: Each database scales independently. Neo4j can be clustered for read-heavy graph traversal workloads. TimescaleDB can be scaled for high-frequency sensor ingestion. PostGIS scales for transactional work order processing.

8. **AI/ML Pipeline Support**: The clear separation of time-series condition data in TimescaleDB simplifies data pipeline construction for training deterioration models, anomaly detection algorithms, and computer vision scoring systems.

### Cons

1. **Operational Complexity**: Three database technologies (PostgreSQL/PostGIS, Neo4j, TimescaleDB) require three sets of expertise for administration, monitoring, backup, security patching, and capacity planning. This is significantly more complex than a single PostgreSQL instance.

2. **Cross-Database Consistency**: There is no distributed transaction across PostGIS, Neo4j, and TimescaleDB. The event bus provides eventual consistency, but temporary inconsistencies are possible. A failed sync could leave the graph out of date with the relational data.

3. **Data Duplication**: Asset data exists in all three databases (node properties in Neo4j, tags in TimescaleDB, rows in PostGIS). This triplicates storage for asset metadata and requires careful synchronization.

4. **Deployment Cost**: Neo4j Enterprise (required for clustering) is commercially licensed. TimescaleDB cloud adds cost. Total infrastructure cost is 2-3x a single PostgreSQL deployment.

5. **Development Complexity**: Developers must learn three query languages (SQL, Cypher, TimescaleDB-extended SQL) and understand which database serves which query pattern. Incorrect routing (e.g., doing graph traversal in SQL) negates the architectural benefits.

6. **Integration Testing**: Testing cross-database workflows (e.g., "create asset in PostGIS, verify node in Neo4j, verify time-series in TimescaleDB") requires integration test infrastructure for three databases.

7. **Vendor Risk**: Neo4j changed its licensing model in 2023 (moving to SSPL for Community Edition). Dependency on a non-PostgreSQL database introduces vendor risk that the project philosophy explicitly aims to avoid.

8. **Over-Engineering for Small Deployments**: A municipality with 5,000 assets does not need graph traversal or time-series compression. The polyglot architecture adds complexity without proportional benefit for small-to-medium deployments.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Graph Database** | Neo4j 5.x Community Edition for single-instance deployments. Consider Apache AGE (PostgreSQL extension providing graph capabilities) as a lighter-weight alternative that avoids a separate database. |
| **Time-Series Database** | TimescaleDB 2.x (PostgreSQL extension). Runs within the same PostgreSQL instance as PostGIS, significantly reducing operational complexity compared to a separate database. |
| **Spatial + Relational** | PostgreSQL 16+ with PostGIS 3.4+ |
| **Event Bus** | Apache Kafka for production; NATS JetStream for simpler deployments. PostgreSQL LISTEN/NOTIFY for single-instance development. |
| **API Layer** | GraphQL (Apollo Server or Strawberry) with data source routing to the appropriate database per query field. REST endpoints for OGC API Features compliance. |
| **Graph Visualization** | D3.js or vis.js for interactive network topology visualization in the browser. Neo4j Bloom for administrative graph exploration. |
| **Monitoring** | Prometheus + Grafana for all three databases. Custom dashboard tracking sync lag between databases. |

### Simplified Variant: PostgreSQL-Only Polyglot

To reduce operational complexity while retaining the conceptual benefits:

- **Apache AGE** (PostgreSQL extension) instead of Neo4j -- provides Cypher-compatible graph queries within PostgreSQL
- **TimescaleDB** (PostgreSQL extension) instead of a separate time-series database -- hypertables and continuous aggregates within the same PostgreSQL instance
- **PostGIS** (PostgreSQL extension) for spatial

This gives you graph + time-series + spatial in a single PostgreSQL database with three extensions, dramatically reducing operational overhead while preserving the specialized query capabilities.

```sql
-- All three extensions in one PostgreSQL instance
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS age;

-- Graph queries via AGE
SELECT * FROM cypher('infrastructure', $$
    MATCH (pipe:Pipe)-[:FLOWS_INTO*1..5]->(downstream)
    WHERE pipe.asset_id = 'uuid-here'
    RETURN downstream.asset_number, downstream.name
$$) AS (asset_number agtype, name agtype);

-- Time-series queries via TimescaleDB
SELECT time_bucket('1 year', time), AVG(score)
FROM ts_condition_scores
WHERE asset_id = 'uuid-here'
GROUP BY 1 ORDER BY 1;

-- Spatial queries via PostGIS
SELECT asset_number, name,
       ST_Distance(geom_point::geography, ST_MakePoint(-73.985, 40.748)::geography) AS distance_m
FROM assets
WHERE ST_DWithin(geom_point::geography, ST_MakePoint(-73.985, 40.748)::geography, 1000);
```

---

## Migration and Scaling Considerations

### Initial Data Migration

1. **PostGIS first**: Load all asset data, organizations, users, and financial records into PostGIS as the system of record.

2. **Neo4j graph construction**: After PostGIS is loaded, run a bulk graph builder that:
   - Creates Asset nodes from the `assets` table
   - Creates hierarchy relationships from `parent_asset_id`
   - Creates network topology relationships from pipe upstream/downstream references
   - Creates zone and department relationships
   - Processes a separate topology configuration file for road connectivity and bridge crossings

3. **TimescaleDB backfill**: Load historical condition scores from inspection records into `ts_condition_scores`. Run continuous aggregate refreshes.

4. **Verify cross-database consistency**: Run a reconciliation check that counts assets in PostGIS vs nodes in Neo4j vs distinct asset_ids in TimescaleDB.

### Scaling Strategy

1. **Small deployment (< 50K assets)**: Use the PostgreSQL-only variant with AGE + TimescaleDB + PostGIS extensions. Single instance, no event bus needed.

2. **Medium deployment (50K-500K assets)**: Separate Neo4j instance for graph queries. TimescaleDB remains as PostgreSQL extension. Kafka for event-driven sync.

3. **Large deployment (500K-5M assets)**: Neo4j cluster for read scaling. TimescaleDB multi-node for high-volume sensor ingestion. PostGIS with read replicas for spatial queries. Full Kafka event bus with consumer groups.

4. **Very large (5M+ assets, multi-state)**: Federated architecture with per-region PostGIS instances, global Neo4j cluster for cross-region network analysis, and TimescaleDB cloud for centralized time-series analytics.

### Operational Considerations

- **Backup coordination**: Schedule backups of all three databases within a consistent time window. Use Kafka consumer offsets as a consistency marker.
- **Schema evolution**: PostGIS schema migrations via Flyway. Neo4j schema changes via Cypher DDL scripts. TimescaleDB schema changes via standard SQL migrations.
- **Monitoring**: Unified Grafana dashboard with panels for each database's health, query performance, and storage utilization. Sync lag is the most critical metric.
- **Disaster recovery**: PostGIS is the system of record and is restored first. Neo4j and TimescaleDB are rebuilt from PostGIS data + event replay if needed.
- **Team structure**: Ideally, one team member has graph database expertise and another has time-series experience. The rest of the team works primarily with PostgreSQL/PostGIS.
