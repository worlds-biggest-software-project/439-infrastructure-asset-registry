# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Infrastructure Asset Registry (Candidate #439)
> Generated: 2026-05-25

## Overview

This model treats every change to the infrastructure asset registry as an immutable domain event. Instead of storing only the current state of assets, inspections, and work orders, the system persists a complete, append-only log of every action that has ever occurred. Read-optimized projections (materialized views) are derived from the event stream to serve queries. This approach is especially well-suited to an infrastructure asset registry because:

1. **Regulatory audit trails are mandatory**: GASB 34, ISO 55001, and federal bridge inspection standards all require demonstrable, reproducible records of who changed what, when, and why. Event sourcing provides this by design rather than as an afterthought.
2. **Condition history is the core value**: An asset's condition trajectory over decades drives capital planning decisions. The event log IS the history -- there is no separate audit table to maintain.
3. **Multi-agency collaboration**: When multiple departments or contractors contribute data, an event log provides clear attribution and conflict resolution.
4. **AI model training**: The full event history provides rich training data for deterioration prediction models.

## Architecture Overview

```
                    ┌─────────────────┐
                    │   Command API   │
                    │  (Write Side)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Command Handler│
                    │  (Validates &   │
                    │   Emits Events) │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Event Store   │
                    │  (Append-Only   │
                    │   PostgreSQL)   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼──────┐ ┌────▼─────┐ ┌──────▼───────┐
     │  Asset State  │ │  Map /   │ │  Reporting   │
     │  Projection   │ │  GIS     │ │  Projection  │
     │  (Read Model) │ │Projection│ │  (GASB 34,   │
     │               │ │(PostGIS) │ │   Dashboards)│
     └───────────────┘ └──────────┘ └──────────────┘
              │              │              │
     ┌────────▼──────────────▼──────────────▼───────┐
     │              Query API (Read Side)            │
     └──────────────────────────────────────────────┘
```

## Design Principles

1. **Events are facts**: Each event records something that happened in the real world (an asset was registered, an inspection was conducted, a condition score was recorded). Events are immutable and never deleted.
2. **Commands are intentions**: A user or system submits a command (register an asset, record an inspection). The command handler validates it against current state and either emits events or rejects the command.
3. **Projections are derived**: Read models are materialized views rebuilt from events. They can be rebuilt from scratch at any time by replaying the event log.
4. **Temporal queries are native**: "What was the condition of Bridge #4421 on March 15, 2024?" is answered by replaying events up to that timestamp -- no special temporal tables needed.

---

## Event Store Schema

```sql
-- Core event store: the single source of truth
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,            -- Aggregate root ID (asset_id, work_order_id, etc.)
    stream_type     VARCHAR(50) NOT NULL,      -- e.g., 'Asset', 'WorkOrder', 'Inspection', 'CapitalProject'
    event_type      VARCHAR(100) NOT NULL,     -- e.g., 'AssetRegistered', 'ConditionScoreRecorded'
    event_version   INTEGER NOT NULL,          -- Per-stream sequence number (optimistic concurrency)
    event_data      JSONB NOT NULL,            -- The event payload
    metadata        JSONB NOT NULL DEFAULT '{}', -- Correlation IDs, causation, user agent, IP
    organization_id UUID NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL,      -- When the real-world event happened
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(), -- When it was stored
    recorded_by     UUID,                      -- User who triggered the command
    
    -- Optimistic concurrency: no two events can have the same version in a stream
    UNIQUE (stream_id, event_version)
);

-- Global ordering for projections that need total order
CREATE SEQUENCE event_global_position;
ALTER TABLE event_store ADD COLUMN global_position BIGINT NOT NULL DEFAULT nextval('event_global_position');
CREATE UNIQUE INDEX idx_event_global_position ON event_store (global_position);

-- Indexes for common access patterns
CREATE INDEX idx_event_stream ON event_store (stream_id, event_version);
CREATE INDEX idx_event_type ON event_store (event_type, recorded_at);
CREATE INDEX idx_event_org_time ON event_store (organization_id, occurred_at);
CREATE INDEX idx_event_stream_type ON event_store (stream_type, organization_id);

-- Partition by month for scalability (infrastructure assets accumulate events over decades)
-- CREATE TABLE event_store_y2026m01 PARTITION OF event_store
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Snapshots for streams with many events (avoid replaying decades of events)
CREATE TABLE event_snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,        -- Event version at snapshot time
    snapshot_data   JSONB NOT NULL,            -- Serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Projection checkpoints: track which global_position each projection has processed
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_position   BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Type Catalog

### Asset Lifecycle Events

```sql
-- Examples of event_data payloads for each event type

-- AssetRegistered: A new asset is added to the registry
-- stream_type: 'Asset'
-- event_data: {
--   "asset_number": "RD-2024-0142",
--   "name": "Main Street - Elm to Oak",
--   "asset_class_code": "ROAD_ARTERIAL",
--   "department_code": "PW",
--   "geometry": {"type": "LineString", "coordinates": [[-73.985, 40.748], [-73.982, 40.751]]},
--   "material": "asphalt",
--   "installation_date": "1998-06-15",
--   "dimensions": {"length_m": 342.5, "width_m": 12.2, "lane_count": 4},
--   "data_source": "legacy_gis_import",
--   "import_batch_id": "..."
-- }

-- AssetAttributeUpdated: An attribute was corrected or added
-- event_data: {
--   "attribute": "surface_type",
--   "old_value": "concrete",
--   "new_value": "asphalt",
--   "reason": "Corrected from field verification"
-- }

-- AssetLocationCorrected: GIS geometry was updated
-- event_data: {
--   "old_geometry": {"type": "LineString", "coordinates": [...]},
--   "new_geometry": {"type": "LineString", "coordinates": [...]},
--   "correction_method": "survey_grade_gps",
--   "accuracy_m": 0.05
-- }

-- AssetReclassified: Asset moved to a different class
-- event_data: {
--   "old_class_code": "ROAD_COLLECTOR",
--   "new_class_code": "ROAD_ARTERIAL",
--   "reason": "Reclassified per 2025 functional class update"
-- }

-- AssetDecommissioned: Asset removed from active service
-- event_data: {
--   "decommission_date": "2026-03-15",
--   "reason": "Replaced by new construction under CIP-2025-042",
--   "replacement_asset_id": "...",
--   "disposal_method": "demolished",
--   "final_condition_score": 22.5
-- }

-- AssetHierarchyChanged: Parent/child relationship modified
-- event_data: {
--   "old_parent_id": null,
--   "new_parent_id": "...",
--   "hierarchy_level": 2,
--   "reason": "Assigned to road network segment"
-- }
```

### Condition and Inspection Events

```sql
-- InspectionScheduled: An inspection was planned
-- stream_type: 'Asset' (event on the asset stream)
-- event_data: {
--   "inspection_id": "...",
--   "framework_code": "PCI",
--   "inspection_type": "routine",
--   "scheduled_date": "2026-04-15",
--   "assigned_inspector": "..."
-- }

-- InspectionConducted: An inspection was performed in the field
-- stream_type: 'Inspection'
-- event_data: {
--   "asset_id": "...",
--   "framework_code": "PCI",
--   "inspection_type": "routine",
--   "inspection_date": "2026-04-15",
--   "inspector_id": "...",
--   "weather": "clear",
--   "temperature_c": 18.5,
--   "sample_units": [
--     {
--       "unit_id": "SU-01",
--       "area_m2": 225.0,
--       "distresses": [
--         {"code": "1", "name": "alligator_cracking", "severity": "medium", "quantity_m2": 12.5, "density_pct": 5.56, "deduct_value": 28.3},
--         {"code": "7", "name": "edge_cracking", "severity": "low", "quantity_m": 8.0, "density_pct": 3.56, "deduct_value": 4.2}
--       ]
--     }
--   ]
-- }

-- ConditionScoreRecorded: Final score computed (may be from inspection or AI)
-- stream_type: 'Asset'
-- event_data: {
--   "inspection_id": "...",
--   "framework_code": "PCI",
--   "score": 62.5,
--   "rating_band": "fair",
--   "previous_score": 71.0,
--   "previous_score_date": "2024-04-10",
--   "delta": -8.5,
--   "scoring_method": "manual_calculation",
--   "confidence": 0.95
-- }

-- AIConditionAssessment: AI model produced a condition estimate
-- stream_type: 'Asset'
-- event_data: {
--   "model_id": "cv-road-condition-v3.2",
--   "model_version": "3.2.1",
--   "input_media_ids": ["...", "..."],
--   "predicted_score": 58.0,
--   "confidence": 0.82,
--   "detected_distresses": [
--     {"type": "alligator_cracking", "severity": "medium", "bounding_box": [0.12, 0.34, 0.56, 0.78], "confidence": 0.91},
--     {"type": "pothole", "severity": "high", "bounding_box": [0.45, 0.22, 0.58, 0.35], "confidence": 0.87}
--   ],
--   "requires_human_review": true
-- }

-- InspectionMediaAttached: Photo/video added to inspection
-- stream_type: 'Inspection'
-- event_data: {
--   "media_id": "...",
--   "media_type": "photo",
--   "file_path": "s3://bucket/inspections/2026/04/15/IMG_4521.jpg",
--   "file_size_bytes": 4521890,
--   "geom_point": {"type": "Point", "coordinates": [-73.985, 40.748]},
--   "taken_at": "2026-04-15T10:32:15Z",
--   "caption": "Alligator cracking near station 0+125"
-- }

-- InspectionApproved / InspectionRejected: Review workflow
-- stream_type: 'Inspection'
-- event_data: {
--   "reviewed_by": "...",
--   "review_notes": "Score confirmed. Photo evidence supports medium severity rating.",
--   "final_score": 62.5
-- }
```

### Work Order Events

```sql
-- WorkOrderCreated: A new work order was opened
-- stream_type: 'WorkOrder'
-- event_data: {
--   "work_order_number": "WO-2026-0891",
--   "asset_id": "...",
--   "work_type": "corrective",
--   "priority": "high",
--   "title": "Pothole repair - Main Street station 0+125",
--   "description": "High severity pothole detected during PCI inspection...",
--   "triggered_by_inspection_id": "...",
--   "triggered_by_citizen_request_id": null
-- }

-- WorkOrderAssigned
-- event_data: {
--   "assigned_to": "...",
--   "assigned_department": "PW",
--   "assigned_crew": "Crew Alpha",
--   "scheduled_start": "2026-04-22T08:00:00Z",
--   "scheduled_end": "2026-04-22T16:00:00Z"
-- }

-- WorkOrderStarted
-- event_data: {
--   "actual_start": "2026-04-22T08:15:00Z",
--   "crew_members": ["...", "..."],
--   "equipment_used": ["dump_truck_14", "roller_07"]
-- }

-- WorkOrderCostRecorded
-- event_data: {
--   "cost_type": "material",
--   "description": "Hot mix asphalt - 2.5 tons",
--   "quantity": 2.5,
--   "unit_cost": 85.00,
--   "total_cost": 212.50,
--   "gl_account": "5400.300"
-- }

-- WorkOrderLaborRecorded
-- event_data: {
--   "worker_id": "...",
--   "work_date": "2026-04-22",
--   "hours_regular": 7.5,
--   "hours_overtime": 0,
--   "labor_rate": 42.50
-- }

-- WorkOrderCompleted
-- event_data: {
--   "actual_end": "2026-04-22T14:30:00Z",
--   "completion_notes": "Pothole filled with hot mix asphalt. Compacted and sealed.",
--   "total_labor_cost": 637.50,
--   "total_material_cost": 212.50,
--   "total_cost": 850.00,
--   "post_repair_condition_estimate": 75.0
-- }
```

### Financial and Capital Planning Events

```sql
-- AssetValuationRecorded: Financial valuation updated
-- stream_type: 'Asset'
-- event_data: {
--   "valuation_type": "replacement_cost",
--   "previous_value": 1250000.00,
--   "new_value": 1385000.00,
--   "valuation_date": "2026-01-01",
--   "valuation_method": "unit_cost_model",
--   "inflation_factor": 1.108,
--   "source": "2026 RSMeans cost data"
-- }

-- DepreciationCalculated: Annual depreciation entry
-- stream_type: 'Asset'
-- event_data: {
--   "fiscal_year": 2026,
--   "method": "straight_line",
--   "depreciation_amount": 25000.00,
--   "accumulated_depreciation": 700000.00,
--   "net_book_value": 550000.00,
--   "useful_life_remaining_years": 22
-- }

-- GASB34AssessmentCompleted
-- stream_type: 'Organization' (or a dedicated GASB stream)
-- event_data: {
--   "asset_class_code": "ROADS",
--   "assessment_year": 2026,
--   "target_condition": 65.0,
--   "actual_condition": 68.2,
--   "condition_met": true,
--   "total_assets_assessed": 1247,
--   "total_replacement_value": 485000000.00,
--   "preservation_cost_estimated": 4200000.00,
--   "preservation_cost_actual": 3850000.00,
--   "assessment_method": "Statistical sampling per ASTM D6433"
-- }

-- CapitalProjectProposed / CapitalProjectApproved / CapitalProjectFunded
-- stream_type: 'CapitalProject'
-- event_data: {
--   "project_number": "CIP-2026-018",
--   "name": "Main Street Reconstruction - Phase 2",
--   "project_type": "rehabilitation",
--   "estimated_cost": 2400000.00,
--   "fiscal_year": 2027,
--   "affected_asset_ids": ["...", "...", "..."],
--   "justification": "PCI scores below 40 for 3 consecutive assessments. Cost of deferral exceeds rehabilitation cost by 2.3x.",
--   "priority_score": 92.5
-- }

-- ScenarioModelRun
-- stream_type: 'PlanningScenario'
-- event_data: {
--   "scenario_name": "Maintain Current Funding",
--   "analysis_period_years": 20,
--   "annual_budget": 5000000.00,
--   "inflation_rate": 0.03,
--   "discount_rate": 0.05,
--   "results": {
--     "total_investment": 100000000.00,
--     "backlog_year_1": 45000000.00,
--     "backlog_year_20": 125000000.00,
--     "avg_condition_year_1": 68.2,
--     "avg_condition_year_20": 52.1,
--     "cost_of_deferral": 180000000.00
--   }
-- }
```

### Citizen and External Events

```sql
-- CitizenRequestSubmitted
-- stream_type: 'CitizenRequest'
-- event_data: {
--   "request_number": "CR-2026-4521",
--   "submitter_name": "Jane Smith",
--   "category": "pothole",
--   "description": "Large pothole in the northbound lane near the intersection with Oak Street",
--   "location_description": "Main St near Oak St",
--   "geom_point": {"type": "Point", "coordinates": [-73.985, 40.748]},
--   "media_ids": ["..."]
-- }

-- CitizenRequestLinkedToAsset
-- event_data: {
--   "asset_id": "...",
--   "asset_number": "RD-2024-0142",
--   "linked_by": "...",
--   "link_method": "spatial_proximity" -- or "manual"
-- }

-- CitizenRequestResolved
-- event_data: {
--   "resolution_notes": "Pothole repaired under WO-2026-0891",
--   "work_order_id": "...",
--   "resolved_by": "...",
--   "satisfaction_survey_sent": true
-- }
```

---

## Read Model Projections

### Asset State Projection (Primary Read Model)

```sql
-- Current state of every asset, rebuilt from events
CREATE TABLE rm_assets (
    asset_id            UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    asset_number        VARCHAR(50) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    asset_class_code    VARCHAR(30) NOT NULL,
    asset_class_name    VARCHAR(255),
    department_code     VARCHAR(20),
    department_name     VARCHAR(255),
    
    -- Hierarchy
    parent_asset_id     UUID,
    hierarchy_level     INTEGER,
    hierarchy_path      TEXT,
    
    -- Location
    address             VARCHAR(500),
    geom_point          GEOMETRY(Point, 4326),
    geom_line           GEOMETRY(LineString, 4326),
    geom_polygon        GEOMETRY(Polygon, 4326),
    
    -- Physical
    material            VARCHAR(100),
    installation_date   DATE,
    
    -- Current condition
    current_condition_score     NUMERIC(5,2),
    current_condition_framework VARCHAR(30),
    current_condition_date      DATE,
    current_condition_rating    VARCHAR(50),
    condition_trend             VARCHAR(20), -- 'improving', 'stable', 'declining'
    
    -- Financial
    acquisition_cost    NUMERIC(16,2),
    replacement_cost    NUMERIC(16,2),
    net_book_value      NUMERIC(16,2),
    total_maintenance_spend NUMERIC(16,2),
    
    -- Status
    status              VARCHAR(30) NOT NULL,
    criticality_rating  INTEGER,
    risk_score          NUMERIC(5,2),
    
    -- Denormalized counts for dashboards
    total_inspections   INTEGER DEFAULT 0,
    total_work_orders   INTEGER DEFAULT 0,
    open_work_orders    INTEGER DEFAULT 0,
    
    -- Type-specific attributes (denormalized from events)
    type_attributes     JSONB DEFAULT '{}',
    
    -- Projection metadata
    last_event_version  INTEGER NOT NULL,
    last_event_id       UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_assets_org ON rm_assets (organization_id);
CREATE INDEX idx_rm_assets_class ON rm_assets (organization_id, asset_class_code);
CREATE INDEX idx_rm_assets_status ON rm_assets (organization_id, status);
CREATE INDEX idx_rm_assets_condition ON rm_assets (organization_id, current_condition_score);
CREATE INDEX idx_rm_assets_geom_point ON rm_assets USING GIST (geom_point);
CREATE INDEX idx_rm_assets_geom_line ON rm_assets USING GIST (geom_line);
CREATE INDEX idx_rm_assets_geom_polygon ON rm_assets USING GIST (geom_polygon);
```

### Condition History Projection

```sql
-- Full condition history for trend analysis and deterioration modeling
CREATE TABLE rm_condition_history (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id            UUID NOT NULL,
    organization_id     UUID NOT NULL,
    framework_code      VARCHAR(30) NOT NULL,
    score               NUMERIC(8,2) NOT NULL,
    rating_band         VARCHAR(50),
    inspection_date     DATE NOT NULL,
    inspection_id       UUID,
    scoring_method      VARCHAR(30), -- 'manual', 'ai_assisted', 'sensor_based'
    confidence          NUMERIC(3,2),
    inspector_id        UUID,
    
    -- Delta from previous
    previous_score      NUMERIC(8,2),
    score_delta         NUMERIC(8,2),
    days_since_previous INTEGER,
    annual_deterioration_rate NUMERIC(6,3),
    
    source_event_id     UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_condition_asset ON rm_condition_history (asset_id, inspection_date DESC);
CREATE INDEX idx_rm_condition_org ON rm_condition_history (organization_id, inspection_date DESC);
CREATE INDEX idx_rm_condition_framework ON rm_condition_history (framework_code, organization_id);
```

### Work Order Projection

```sql
-- Current state of work orders
CREATE TABLE rm_work_orders (
    work_order_id       UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    work_order_number   VARCHAR(30) NOT NULL,
    asset_id            UUID NOT NULL,
    asset_number        VARCHAR(50),
    asset_name          VARCHAR(255),
    
    work_type           VARCHAR(30) NOT NULL,
    priority            VARCHAR(20) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    
    assigned_to         UUID,
    assigned_crew       VARCHAR(100),
    assigned_department VARCHAR(100),
    
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    
    status              VARCHAR(20) NOT NULL,
    
    total_labor_cost    NUMERIC(14,2) DEFAULT 0,
    total_material_cost NUMERIC(14,2) DEFAULT 0,
    total_cost          NUMERIC(14,2) DEFAULT 0,
    
    triggered_by_inspection_id UUID,
    triggered_by_citizen_request_id UUID,
    
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_wo_asset ON rm_work_orders (asset_id);
CREATE INDEX idx_rm_wo_status ON rm_work_orders (organization_id, status);
CREATE INDEX idx_rm_wo_assigned ON rm_work_orders (assigned_to, status);
```

### Financial Reporting Projection

```sql
-- Asset financial summary for GASB 34 and capital planning
CREATE TABLE rm_asset_financials (
    asset_id                UUID PRIMARY KEY,
    organization_id         UUID NOT NULL,
    asset_class_code        VARCHAR(30) NOT NULL,
    
    acquisition_date        DATE,
    acquisition_cost        NUMERIC(16,2),
    replacement_cost        NUMERIC(16,2),
    replacement_cost_date   DATE,
    salvage_value           NUMERIC(16,2),
    
    depreciation_method     VARCHAR(30),
    useful_life_years       INTEGER,
    accumulated_depreciation NUMERIC(16,2),
    net_book_value          NUMERIC(16,2),
    
    total_maintenance_cost  NUMERIC(16,2) DEFAULT 0,
    total_capital_cost      NUMERIC(16,2) DEFAULT 0,
    current_fiscal_year_cost NUMERIC(14,2) DEFAULT 0,
    
    funding_source          VARCHAR(100),
    gl_asset_account        VARCHAR(30),
    fund_code               VARCHAR(30),
    cost_center             VARCHAR(30),
    
    projected_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_fin_org_class ON rm_asset_financials (organization_id, asset_class_code);

-- GASB 34 assessment summary (pre-computed for report generation)
CREATE TABLE rm_gasb34_summary (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id         UUID NOT NULL,
    asset_class_code        VARCHAR(30) NOT NULL,
    assessment_year         INTEGER NOT NULL,
    target_condition        NUMERIC(5,2),
    actual_condition        NUMERIC(5,2),
    condition_met           BOOLEAN,
    total_assets            INTEGER,
    total_replacement_value NUMERIC(18,2),
    preservation_cost_est   NUMERIC(16,2),
    preservation_cost_actual NUMERIC(16,2),
    projected_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, asset_class_code, assessment_year)
);
```

### GIS / Map Projection

```sql
-- Lightweight spatial projection for map rendering
-- (Contains only the data needed for map tiles and popups)
CREATE TABLE rm_map_features (
    asset_id            UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    asset_number        VARCHAR(50),
    name                VARCHAR(255),
    asset_class_code    VARCHAR(30),
    asset_category      VARCHAR(30),
    status              VARCHAR(30),
    condition_score     NUMERIC(5,2),
    condition_rating    VARCHAR(50),
    condition_color     VARCHAR(7), -- Hex color for map rendering
    criticality         INTEGER,
    
    -- Single geometry column (whichever is populated)
    geom                GEOMETRY(Geometry, 4326) NOT NULL,
    centroid            GEOMETRY(Point, 4326),
    
    -- Popup summary
    popup_html          TEXT,
    
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_map_geom ON rm_map_features USING GIST (geom);
CREATE INDEX idx_rm_map_org ON rm_map_features (organization_id);
CREATE INDEX idx_rm_map_class ON rm_map_features (organization_id, asset_class_code);
CREATE INDEX idx_rm_map_condition ON rm_map_features (organization_id, condition_score);
```

### Dashboard Metrics Projection

```sql
-- Pre-aggregated dashboard metrics (rebuilt periodically or on event)
CREATE TABLE rm_dashboard_metrics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    metric_period       VARCHAR(10) NOT NULL, -- 'daily', 'weekly', 'monthly', 'yearly'
    period_start        DATE NOT NULL,
    
    -- Asset counts
    total_assets        INTEGER,
    active_assets       INTEGER,
    assets_by_class     JSONB, -- {"ROAD": 1247, "BRIDGE": 89, "PIPE": 3421, ...}
    assets_by_condition JSONB, -- {"excellent": 234, "good": 456, "fair": 312, ...}
    
    -- Condition summary
    avg_condition_score NUMERIC(5,2),
    pct_above_target    NUMERIC(5,2),
    assets_below_threshold INTEGER,
    
    -- Work order summary
    wo_opened           INTEGER,
    wo_completed        INTEGER,
    wo_backlog          INTEGER,
    avg_completion_days NUMERIC(8,2),
    
    -- Financial summary
    total_replacement_value NUMERIC(18,2),
    maintenance_spend   NUMERIC(16,2),
    capital_spend       NUMERIC(16,2),
    backlog_value       NUMERIC(18,2),
    
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, metric_period, period_start)
);
```

---

## Projection Engine (Pseudocode)

```python
# Projection rebuilder: processes events from the event store
# and updates read model tables

class AssetProjection:
    """Maintains the rm_assets read model."""
    
    projection_name = "asset_state"
    
    def handle_event(self, event):
        match event.event_type:
            
            case "AssetRegistered":
                data = event.event_data
                upsert_rm_assets(
                    asset_id=event.stream_id,
                    organization_id=event.organization_id,
                    asset_number=data["asset_number"],
                    name=data["name"],
                    asset_class_code=data["asset_class_code"],
                    material=data.get("material"),
                    installation_date=data.get("installation_date"),
                    geom=parse_geojson(data.get("geometry")),
                    status="active",
                    last_event_version=event.event_version,
                    last_event_id=event.event_id,
                )
            
            case "ConditionScoreRecorded":
                data = event.event_data
                update_rm_assets(
                    asset_id=event.stream_id,
                    current_condition_score=data["score"],
                    current_condition_framework=data["framework_code"],
                    current_condition_date=event.occurred_at.date(),
                    current_condition_rating=data["rating_band"],
                    condition_trend=compute_trend(event.stream_id, data["score"]),
                    total_inspections=increment(),
                )
                insert_rm_condition_history(
                    asset_id=event.stream_id,
                    framework_code=data["framework_code"],
                    score=data["score"],
                    rating_band=data["rating_band"],
                    inspection_date=event.occurred_at.date(),
                    previous_score=data.get("previous_score"),
                    score_delta=data.get("delta"),
                    source_event_id=event.event_id,
                )
            
            case "AssetDecommissioned":
                update_rm_assets(
                    asset_id=event.stream_id,
                    status="decommissioned",
                )
            
            case "WorkOrderCreated":
                update_rm_assets(
                    asset_id=event.event_data["asset_id"],
                    total_work_orders=increment(),
                    open_work_orders=increment(),
                )
            
            case "WorkOrderCompleted":
                asset_id = get_work_order_asset(event.stream_id)
                update_rm_assets(
                    asset_id=asset_id,
                    open_work_orders=decrement(),
                    total_maintenance_spend=add(event.event_data["total_cost"]),
                )

    def rebuild(self):
        """Full rebuild: truncate read model, replay all events."""
        truncate("rm_assets")
        truncate("rm_condition_history")
        for event in get_all_events(stream_type="Asset", order_by="global_position"):
            self.handle_event(event)
        update_checkpoint(self.projection_name)
```

---

## Temporal Queries (Time Travel)

One of the most powerful features of event sourcing for infrastructure asset management:

```sql
-- "What was the condition of asset X on a specific date?"
-- Answer: replay ConditionScoreRecorded events up to that date
SELECT event_data->>'score' AS condition_score,
       event_data->>'framework_code' AS framework,
       event_data->>'rating_band' AS rating,
       occurred_at
FROM event_store
WHERE stream_id = :asset_id
  AND event_type = 'ConditionScoreRecorded'
  AND occurred_at <= :target_date
ORDER BY occurred_at DESC
LIMIT 1;

-- "What was the full state of asset X at a point in time?"
-- Answer: replay all events for that stream up to the target date
SELECT event_type, event_data, occurred_at
FROM event_store
WHERE stream_id = :asset_id
  AND occurred_at <= :target_date
ORDER BY event_version ASC;
-- Application code replays these events to reconstruct state

-- "Show me all changes to asset X in the last 90 days"
-- (Built-in audit trail -- no separate audit table needed)
SELECT event_type,
       event_data,
       occurred_at,
       recorded_by
FROM event_store
WHERE stream_id = :asset_id
  AND occurred_at >= now() - INTERVAL '90 days'
ORDER BY event_version ASC;

-- "Which assets had condition changes greater than 10 points in the last year?"
SELECT DISTINCT stream_id AS asset_id,
       event_data->>'score' AS current_score,
       event_data->>'previous_score' AS previous_score,
       (event_data->>'delta')::numeric AS delta
FROM event_store
WHERE event_type = 'ConditionScoreRecorded'
  AND organization_id = :org_id
  AND occurred_at >= now() - INTERVAL '1 year'
  AND ABS((event_data->>'delta')::numeric) > 10
ORDER BY ABS((event_data->>'delta')::numeric) DESC;
```

---

## Pros and Cons

### Pros

1. **Complete Audit Trail by Design**: Every change is permanently recorded with who, what, when, and why. This is not a separate audit table that can fall out of sync -- it IS the data. This directly satisfies ISO 55001 clause 7.5 (documented information), GASB 34 modified approach documentation requirements, and FHWA bridge inspection record-keeping mandates.

2. **Temporal Queries Are Free**: "What was the condition of this bridge on the day of the last federal inspection?" requires no special temporal tables or slowly-changing dimension logic. Simply replay events to a timestamp.

3. **Condition Trajectory Analysis**: The event stream provides the exact sequence of condition changes over an asset's multi-decade life. Deterioration models can be trained directly on the event log without ETL into an analytics warehouse.

4. **Conflict Resolution for Multi-User Field Data**: When multiple inspectors or departments submit overlapping data, the event log preserves all submissions with clear timestamps and attribution. Conflicts are resolved by business rules applied to events, not by last-write-wins overwrites.

5. **Projection Flexibility**: New read models (e.g., a new reporting requirement, a new dashboard metric, a new AI feature) can be added by creating a new projection and replaying the existing event log. No schema migration required for read-side changes.

6. **Data Lineage**: Every piece of data can be traced to the event that created it and the user who triggered it. Import batches, AI assessments, manual corrections -- all have clear provenance.

7. **Bug Fix Replay**: If a projection had a bug (e.g., calculating PCI scores incorrectly), fix the projection code and replay the events. The read model is rebuilt correctly without any data loss.

8. **Integration-Friendly**: External systems can subscribe to the event stream (via polling or pub/sub) to maintain their own read models. A GIS system, a financial system, and an analytics platform can each consume the same events.

### Cons

1. **Architectural Complexity**: Event sourcing and CQRS are significantly more complex than CRUD. The team needs to understand event design, projection management, eventual consistency, and idempotency. This is a steep learning curve for teams accustomed to traditional application development.

2. **Eventual Consistency**: Read models lag behind writes. After a field inspector submits an inspection, the dashboard may take milliseconds to seconds to reflect the change. For most infrastructure asset management workflows this is acceptable, but users accustomed to immediate consistency may find it confusing.

3. **Event Schema Evolution**: Changing the structure of an event type (e.g., adding a new field to `AssetRegistered`) requires upcasting logic to handle old events that lack the field. Over years and decades of operation, this version management becomes non-trivial.

4. **Storage Growth**: Storing every event forever requires significantly more storage than a mutable state model. An asset with 50 years of quarterly inspections accumulates hundreds of events. Snapshots mitigate replay cost but don't reduce storage.

5. **Query Complexity**: Ad-hoc queries against the event store are harder than against normalized tables. Analysts cannot simply `SELECT * FROM assets WHERE condition < 50`. They must query read model projections, which may not contain the exact view they need.

6. **Projection Rebuild Time**: As the event store grows into millions of events, rebuilding a projection from scratch can take hours. This must be planned for and tested regularly.

7. **Tooling Maturity**: Event sourcing tooling for PostgreSQL (e.g., Marten, EventStoreDB connectors) is less mature than traditional ORM tooling. Custom projection management code is typically required.

8. **Team Expertise**: Finding developers with event sourcing experience is harder than finding those with CRUD/ORM experience. This affects hiring and onboarding costs.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Event Store** | PostgreSQL 16+ (single source of truth). Use the `event_store` table with JSONB payloads. Partition by month for long-lived infrastructure data. |
| **Message Bus** | Apache Kafka or NATS JetStream for event distribution to projections and external subscribers. PostgreSQL LISTEN/NOTIFY for low-volume, single-instance deployments. |
| **Projection Engine** | Custom application service (Python, Node.js, or Rust) that polls the event store or consumes from Kafka. Use a projection checkpoint table for exactly-once processing. |
| **Read Model Database** | PostgreSQL + PostGIS for all projections (same database cluster, separate schema). This keeps operations simple. |
| **Snapshot Store** | PostgreSQL `event_snapshots` table. Snapshot every 100 events per stream, or every year for long-lived assets. |
| **API Gateway** | Separate Command API (mutations go through command handlers) and Query API (reads go against projections). |
| **Spatial** | PostGIS on the read model for GIS queries. The `rm_map_features` projection is purpose-built for map rendering. |
| **Monitoring** | Track projection lag (difference between latest event global_position and projection checkpoint). Alert if lag exceeds threshold. |
| **Serialization** | JSON Schema for event validation. Version each event type with a `schema_version` field in metadata. |

---

## Migration and Scaling Considerations

### Migrating from Legacy Systems

1. **Legacy data becomes events**: When importing from spreadsheets or legacy GIS, generate `AssetRegistered` events with `data_source: "legacy_import"` in the metadata. The event store becomes the canonical record, and the import is fully traceable.

2. **Bulk import optimization**: For initial migration of thousands of assets, use batch inserts into the event store, then run projection rebuilds once. Do not process events one-by-one during migration.

3. **Parallel projection builds**: After bulk import, rebuild all projections in parallel to minimize downtime.

### Scaling Strategy

1. **0-100K events/day**: Single PostgreSQL instance handles both event store and projections. LISTEN/NOTIFY for projection updates.

2. **100K-1M events/day**: Add Kafka for event distribution. Projections run as separate services consuming from Kafka topics. Read replicas for query load.

3. **1M+ events/day**: Partition event store by month. Consider EventStoreDB as a dedicated event store if PostgreSQL write throughput becomes a bottleneck. Scale projection workers horizontally.

4. **Multi-region**: Event store remains in the primary region. Read model projections can be replicated to edge regions for low-latency map queries.

### Event Store Maintenance

- **Never delete events** (append-only invariant). For data privacy (GDPR), use crypto-shredding: encrypt PII fields with per-entity keys, and delete the key when required.
- **Partition by time**: Monthly partitions allow efficient archival of old partitions to cold storage.
- **Snapshot cadence**: Create snapshots for assets with >100 events to keep replay fast. For a 50-year-old bridge with quarterly inspections, that is ~200 events -- snapshot annually.
- **Event store compaction**: Optionally, create compacted streams that merge superseded events (e.g., if `AssetAttributeUpdated` for the same attribute occurs 50 times, the compacted stream keeps only the latest). The original event store remains untouched.

### Schema Evolution

- **Upcasting**: When an event schema changes, implement an upcaster function that transforms old event versions to the new format at read time. This avoids rewriting the event store.
- **Weak schema**: Use JSONB for event_data and validate at the application layer with JSON Schema. This provides flexibility while maintaining validation.
- **Versioned event types**: Include `schema_version` in event metadata. Projections use version-aware handlers.
