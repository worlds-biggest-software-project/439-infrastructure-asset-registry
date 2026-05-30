# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL + JSONB)

> Project: Infrastructure Asset Registry (Candidate #439)
> Generated: 2026-05-25

## Overview

This model uses PostgreSQL as a hybrid relational-document database, combining normalized tables for stable, well-understood entities (organizations, users, work orders, financial records) with JSONB columns for variable, type-specific, and evolving data (asset attributes, inspection distress details, AI analysis results, custom fields). This approach directly addresses the most significant tension in infrastructure asset management data modeling: every municipality tracks the same core concepts (assets, conditions, work orders) but the specific attributes, scoring frameworks, and field configurations vary wildly between organizations, asset types, and evolving standards.

## Design Rationale

Infrastructure asset registries face a unique data modeling challenge:

1. **Asset type diversity**: A road segment has surface type, lane count, and traffic volume. A bridge has span count, load rating, and scour criticality. A pipe has diameter, material, and flow direction. A signal has controller type and detection method. There are dozens of asset types, each with 10-30 unique attributes.

2. **Municipal customization**: One municipality tracks curb ramp compliance; another tracks tree canopy coverage. Custom fields are the norm, not the exception.

3. **Evolving standards**: The FHWA updates bridge inspection elements. ASTM revises PCI distress types. New condition frameworks emerge. The schema must accommodate changes without DDL migrations.

4. **Inspection flexibility**: Different condition frameworks (PCI, NBI, ACI, NASSCO PACP) have completely different distress catalogs, severity scales, and scoring algorithms.

The hybrid approach gives us strong typing and referential integrity where it matters (financial records, work order workflow, user management) and document flexibility where rigidity would be harmful (asset attributes, inspection details, AI outputs).

## Design Principles

1. **Normalize what is stable**: Organizations, users, departments, work orders, financial records -- these have predictable structures shared across all deployments.
2. **JSONB what varies**: Asset type-specific attributes, inspection distress details, AI analysis payloads, custom fields -- these change per asset class, per organization, and over time.
3. **GIN indexes on JSONB**: Every JSONB column used for querying gets a GIN index. Frequently queried keys within JSONB get expression indexes.
4. **JSON Schema validation**: Application-layer validation using JSON Schema definitions stored in the database, ensuring JSONB data is structured even though the database schema is flexible.
5. **PostGIS for spatial**: Geometry columns remain native PostGIS types (not stored in JSONB) for spatial index performance.

---

## Core Schema

### Organizations and Users (Fully Normalized)

```sql
CREATE TABLE organizations (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name              VARCHAR(255) NOT NULL,
    org_type          VARCHAR(50) NOT NULL,
    jurisdiction_code VARCHAR(20),
    state_province    VARCHAR(100),
    country_code      CHAR(2) NOT NULL DEFAULT 'US',
    timezone          VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    fiscal_year_start INTEGER NOT NULL DEFAULT 1,
    
    -- Organization-wide settings and configuration (JSONB for flexibility)
    settings          JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example settings: {
    --   "gasb34_approach": "modified",
    --   "default_condition_framework": "PCI",
    --   "map_center": [-73.985, 40.748],
    --   "map_default_zoom": 13,
    --   "branding": {"logo_url": "...", "primary_color": "#2563EB"},
    --   "features_enabled": ["ai_scoring", "citizen_portal", "capital_planning"]
    -- }
    
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    department_type VARCHAR(50),
    parent_id       UUID REFERENCES departments(id),
    config          JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    user_role       VARCHAR(30) NOT NULL,
    department_id   UUID REFERENCES departments(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    
    -- User preferences and settings
    preferences     JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {
    --   "default_map_view": "satellite",
    --   "notification_prefs": {"email": true, "push": true, "sms": false},
    --   "dashboard_layout": [...]
    -- }
    
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Asset Classification with Schema Definitions

```sql
-- Asset classes define what attributes each asset type has,
-- using JSON Schema stored in the database
CREATE TABLE asset_classes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    code            VARCHAR(30) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES asset_classes(id),
    hierarchy_level INTEGER NOT NULL DEFAULT 0,
    asset_category  VARCHAR(30) NOT NULL, -- 'horizontal', 'vertical', 'point', 'linear', 'area'
    description     TEXT,
    
    -- JSON Schema defining the structure of the attributes JSONB column
    -- for assets of this class. Used for validation at the application layer.
    attribute_schema JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example for roads:
    -- {
    --   "$schema": "https://json-schema.org/draft/2020-12/schema",
    --   "type": "object",
    --   "properties": {
    --     "road_class": {"type": "string", "enum": ["interstate", "arterial", "collector", "local"]},
    --     "surface_type": {"type": "string", "enum": ["asphalt", "concrete", "gravel", "unpaved"]},
    --     "lane_count": {"type": "integer", "minimum": 1, "maximum": 12},
    --     "speed_limit_kph": {"type": "integer"},
    --     "traffic_volume_aadt": {"type": "integer"},
    --     "pavement_thickness_mm": {"type": "number"},
    --     "last_overlay_date": {"type": "string", "format": "date"},
    --     "sidewalk_present": {"type": "boolean"}
    --   },
    --   "required": ["road_class", "surface_type"]
    -- }
    
    -- Default values for new assets of this class
    attribute_defaults JSONB NOT NULL DEFAULT '{}'::jsonb,
    
    -- Display configuration (field order, labels, grouping for UI)
    display_config  JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {
    --   "field_groups": [
    --     {"label": "Road Classification", "fields": ["road_class", "functional_class", "surface_type"]},
    --     {"label": "Dimensions", "fields": ["lane_count", "speed_limit_kph", "pavement_thickness_mm"]},
    --     {"label": "Traffic", "fields": ["traffic_volume_aadt"]}
    --   ]
    -- }
    
    -- Financial defaults
    default_useful_life_years INTEGER,
    depreciation_method VARCHAR(30),
    
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);
```

### Assets: Normalized Core + JSONB Attributes

```sql
-- The core asset table: stable columns + flexible JSONB
CREATE TABLE assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    asset_class_id      UUID NOT NULL REFERENCES asset_classes(id),
    department_id       UUID REFERENCES departments(id),
    
    -- === NORMALIZED CORE (stable across all asset types) ===
    
    -- Identification
    asset_number        VARCHAR(50) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    
    -- Hierarchy
    parent_asset_id     UUID REFERENCES assets(id),
    hierarchy_level     INTEGER NOT NULL DEFAULT 0,
    hierarchy_path      LTREE,
    
    -- Location (always normalized for spatial queries)
    address             VARCHAR(500),
    locality            VARCHAR(255),
    geom_point          GEOMETRY(Point, 4326),
    geom_line           GEOMETRY(LineString, 4326),
    geom_polygon        GEOMETRY(Polygon, 4326),
    
    -- Universal physical attributes
    material            VARCHAR(100),
    installation_date   DATE,
    
    -- Lifecycle status (universal workflow)
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    criticality_rating  INTEGER CHECK (criticality_rating BETWEEN 1 AND 5),
    
    -- Current condition (denormalized for fast queries)
    current_condition_score  NUMERIC(5,2),
    current_condition_date   DATE,
    current_condition_rating VARCHAR(20),
    
    -- === JSONB FLEXIBLE DATA ===
    
    -- Type-specific attributes validated against asset_class.attribute_schema
    attributes          JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Roads example: {
    --   "road_class": "arterial",
    --   "surface_type": "asphalt",
    --   "lane_count": 4,
    --   "speed_limit_kph": 56,
    --   "traffic_volume_aadt": 12500,
    --   "pavement_thickness_mm": 150.0,
    --   "sidewalk_present": true,
    --   "last_overlay_date": "2018-06-15"
    -- }
    --
    -- Bridge example: {
    --   "nbi_structure_number": "NY-1234567",
    --   "bridge_type": "beam",
    --   "deck_type": "concrete",
    --   "span_count": 3,
    --   "max_span_length_m": 25.5,
    --   "load_rating_tons": 36.0,
    --   "scour_critical": false,
    --   "fracture_critical": false,
    --   "year_built": 1962,
    --   "year_reconstructed": 2005
    -- }
    --
    -- Pipe example: {
    --   "pipe_system": "sanitary_sewer",
    --   "pipe_material": "vitrified_clay",
    --   "nominal_diameter_mm": 300,
    --   "slope_percent": 0.015,
    --   "lining_type": "cipp",
    --   "lining_date": "2020-09-01",
    --   "upstream_node_id": "...",
    --   "downstream_node_id": "..."
    -- }
    
    -- Organization-specific custom fields (no schema enforcement)
    custom_fields       JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {
    --   "council_district": 7,
    --   "ada_ramp_count": 4,
    --   "snow_route_priority": "A",
    --   "tree_canopy_pct": 45.2,
    --   "historical_designation": true
    -- }
    
    -- Risk assessment data
    risk_data           JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {
    --   "risk_score": 72.5,
    --   "likelihood_of_failure": 0.15,
    --   "consequence_of_failure": 4,
    --   "risk_factors": {
    --     "age_factor": 0.8,
    --     "condition_factor": 0.6,
    --     "traffic_factor": 0.9,
    --     "environmental_factor": 0.3
    --   },
    --   "last_risk_assessment_date": "2026-01-15"
    -- }
    
    -- Import and data quality metadata
    import_metadata     JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {
    --   "data_source": "legacy_gis_import",
    --   "import_batch_id": "...",
    --   "original_id": "ROAD_SEG_4521",
    --   "data_quality_score": 0.78,
    --   "data_quality_issues": ["missing_installation_date", "estimated_dimensions"],
    --   "ai_field_mapping_confidence": 0.92
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),
    updated_by          UUID REFERENCES users(id),
    
    UNIQUE (organization_id, asset_number)
);

-- Spatial indexes (native PostGIS, not JSONB)
CREATE INDEX idx_assets_geom_point ON assets USING GIST (geom_point);
CREATE INDEX idx_assets_geom_line ON assets USING GIST (geom_line);
CREATE INDEX idx_assets_geom_polygon ON assets USING GIST (geom_polygon);

-- Hierarchy index
CREATE INDEX idx_assets_hierarchy ON assets USING GIST (hierarchy_path);

-- Standard B-tree indexes
CREATE INDEX idx_assets_org_class ON assets (organization_id, asset_class_id);
CREATE INDEX idx_assets_status ON assets (organization_id, status);
CREATE INDEX idx_assets_condition ON assets (organization_id, current_condition_score);
CREATE INDEX idx_assets_parent ON assets (parent_asset_id);

-- GIN index on attributes JSONB for containment queries
CREATE INDEX idx_assets_attributes ON assets USING GIN (attributes jsonb_path_ops);
CREATE INDEX idx_assets_custom_fields ON assets USING GIN (custom_fields jsonb_path_ops);

-- Expression indexes for frequently queried JSONB keys
CREATE INDEX idx_assets_road_class ON assets ((attributes->>'road_class'))
    WHERE attributes ? 'road_class';
CREATE INDEX idx_assets_surface_type ON assets ((attributes->>'surface_type'))
    WHERE attributes ? 'surface_type';
CREATE INDEX idx_assets_pipe_system ON assets ((attributes->>'pipe_system'))
    WHERE attributes ? 'pipe_system';
CREATE INDEX idx_assets_pipe_material ON assets ((attributes->>'pipe_material'))
    WHERE attributes ? 'pipe_material';
CREATE INDEX idx_assets_nbi_number ON assets ((attributes->>'nbi_structure_number'))
    WHERE attributes ? 'nbi_structure_number';
```

### Condition Inspections: Normalized Wrapper + JSONB Details

```sql
-- Condition frameworks with JSONB configuration
CREATE TABLE condition_frameworks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    code            VARCHAR(30) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    standard_ref    VARCHAR(100),
    score_min       NUMERIC(8,2) NOT NULL,
    score_max       NUMERIC(8,2) NOT NULL,
    score_direction VARCHAR(10) NOT NULL, -- 'asc' (higher=better) or 'desc'
    
    -- Full framework configuration in JSONB
    framework_config JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- PCI example: {
    --   "distress_types": [
    --     {"code": "1", "name": "alligator_cracking", "unit": "sq_m", "severities": ["L", "M", "H"],
    --      "deduct_curves": {"L": [[0,0],[5,8],[10,15],...], "M": [...], "H": [...]}},
    --     {"code": "2", "name": "bleeding", "unit": "sq_m", "severities": ["L", "M", "H"],
    --      "deduct_curves": {...}},
    --     ... (21 asphalt distress types)
    --   ],
    --   "rating_bands": [
    --     {"label": "Excellent", "min": 86, "max": 100, "color": "#22C55E", "treatment": "Routine maintenance only"},
    --     {"label": "Good", "min": 71, "max": 85, "color": "#84CC16", "treatment": "Preventive maintenance"},
    --     {"label": "Satisfactory", "min": 56, "max": 70, "color": "#EAB308", "treatment": "Minor rehabilitation"},
    --     {"label": "Fair", "min": 41, "max": 55, "color": "#F97316", "treatment": "Major rehabilitation"},
    --     {"label": "Poor", "min": 26, "max": 40, "color": "#EF4444", "treatment": "Reconstruction needed"},
    --     {"label": "Serious", "min": 11, "max": 25, "color": "#DC2626", "treatment": "Urgent reconstruction"},
    --     {"label": "Failed", "min": 0, "max": 10, "color": "#7F1D1D", "treatment": "Immediate closure/reconstruction"}
    --   ],
    --   "sample_unit_size_m2": 225,
    --   "max_corrected_deduct_value_curves": {...},
    --   "scoring_algorithm": "astm_d6433_2020"
    -- }
    --
    -- NBI Bridge example: {
    --   "element_types": [
    --     {"code": "12", "name": "concrete_deck", "unit": "sq_m",
    --      "condition_states": [
    --        {"state": 1, "description": "Good - No notable distress"},
    --        {"state": 2, "description": "Fair - Minor cracking, scaling"},
    --        {"state": 3, "description": "Poor - Moderate cracking, spalling"},
    --        {"state": 4, "description": "Severe - Advanced deterioration"}
    --      ]},
    --     ...
    --   ],
    --   "rating_scale": {"min": 0, "max": 9, "direction": "desc"},
    --   "component_ratings": ["deck", "superstructure", "substructure", "channel"],
    --   "sufficiency_rating_formula": "..."
    -- }
    
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

-- Inspections: normalized wrapper with JSONB details
CREATE TABLE inspections (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id         UUID NOT NULL REFERENCES organizations(id),
    asset_id                UUID NOT NULL REFERENCES assets(id),
    condition_framework_id  UUID NOT NULL REFERENCES condition_frameworks(id),
    
    -- Normalized fields (common across all frameworks)
    inspection_type         VARCHAR(30) NOT NULL,
    inspection_date         DATE NOT NULL,
    inspected_by            UUID REFERENCES users(id),
    overall_score           NUMERIC(8,2),
    rating_band             VARCHAR(50),
    status                  VARCHAR(20) NOT NULL DEFAULT 'draft',
    reviewed_by             UUID REFERENCES users(id),
    reviewed_at             TIMESTAMPTZ,
    
    -- JSONB: Framework-specific inspection data
    inspection_data         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- PCI example: {
    --   "sample_units": [
    --     {
    --       "unit_id": "SU-01",
    --       "station_start_m": 0,
    --       "station_end_m": 50,
    --       "area_m2": 225.0,
    --       "distresses": [
    --         {"code": "1", "severity": "M", "quantity": 12.5, "density": 5.56, "deduct_value": 28.3},
    --         {"code": "7", "severity": "L", "quantity": 8.0, "density": 3.56, "deduct_value": 4.2}
    --       ],
    --       "total_deduct_value": 32.5,
    --       "corrected_deduct_value": 28.0,
    --       "pci": 72.0
    --     },
    --     { ... more sample units ... }
    --   ],
    --   "network_level_summary": {
    --     "total_area_m2": 4500,
    --     "sampled_area_m2": 900,
    --     "sampling_rate": 0.20,
    --     "weighted_pci": 62.5,
    --     "standard_deviation": 8.3
    --   }
    -- }
    --
    -- NBI Bridge example: {
    --   "component_ratings": {
    --     "deck": 6,
    --     "superstructure": 7,
    --     "substructure": 5,
    --     "channel": 7,
    --     "culvert": null
    --   },
    --   "elements": [
    --     {"code": "12", "name": "concrete_deck", "total_quantity": 500,
    --      "condition_states": {"1": 250, "2": 150, "3": 80, "4": 20}},
    --     {"code": "107", "name": "steel_girder", "total_quantity": 12,
    --      "condition_states": {"1": 8, "2": 3, "3": 1, "4": 0}}
    --   ],
    --   "load_rating": {"inventory_rating_tons": 28, "operating_rating_tons": 36},
    --   "sufficiency_rating": 68.2,
    --   "health_index": 72.5
    -- }
    
    -- JSONB: Environmental context
    context_data            JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {
    --   "weather": "clear",
    --   "temperature_c": 18.5,
    --   "recent_precipitation": false,
    --   "traffic_control": "lane_closure",
    --   "equipment_used": ["measuring_wheel", "straight_edge", "camera"]
    -- }
    
    -- JSONB: AI analysis results (if applicable)
    ai_analysis             JSONB,
    -- Example: {
    --   "model_id": "cv-road-condition-v3.2",
    --   "model_version": "3.2.1",
    --   "predicted_score": 58.0,
    --   "confidence": 0.82,
    --   "detected_distresses": [
    --     {"type": "alligator_cracking", "severity": "M", "confidence": 0.91,
    --      "bounding_box": [0.12, 0.34, 0.56, 0.78], "area_m2_estimate": 8.5}
    --   ],
    --   "processing_time_ms": 2340,
    --   "input_images": 12,
    --   "requires_human_review": true,
    --   "review_reason": "Low confidence on severity classification"
    -- }
    
    notes                   TEXT,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspections_asset ON inspections (asset_id, inspection_date DESC);
CREATE INDEX idx_inspections_org_date ON inspections (organization_id, inspection_date DESC);
CREATE INDEX idx_inspections_status ON inspections (organization_id, status);
CREATE INDEX idx_inspections_data ON inspections USING GIN (inspection_data jsonb_path_ops);

-- Inspection media
CREATE TABLE inspection_media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspections(id) ON DELETE CASCADE,
    media_type      VARCHAR(20) NOT NULL,
    file_path       VARCHAR(500) NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    caption         TEXT,
    taken_at        TIMESTAMPTZ,
    geom_point      GEOMETRY(Point, 4326),
    
    -- AI analysis per image (JSONB)
    ai_results      JSONB,
    -- Example: {
    --   "detected_distresses": [...],
    --   "suggested_severity": "medium",
    --   "suggested_score_impact": -5.2,
    --   "confidence": 0.87
    -- }
    
    display_order   INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Work Orders (Mostly Normalized)

```sql
CREATE TABLE work_orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    work_order_number   VARCHAR(30) NOT NULL,
    asset_id            UUID NOT NULL REFERENCES assets(id),
    inspection_id       UUID REFERENCES inspections(id),
    
    -- Normalized workflow fields
    work_type           VARCHAR(30) NOT NULL,
    priority            VARCHAR(20) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    
    assigned_department_id UUID REFERENCES departments(id),
    assigned_to         UUID REFERENCES users(id),
    
    requested_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    
    status              VARCHAR(20) NOT NULL DEFAULT 'open',
    completion_notes    TEXT,
    
    geom_point          GEOMETRY(Point, 4326),
    
    -- JSONB: Flexible work details
    work_details        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {
    --   "failure_code": "PVMT-CRACK-ALG",
    --   "cause_code": "AGE-FATIGUE",
    --   "remedy_code": "PATCH-HOTMIX",
    --   "crew": "Crew Alpha",
    --   "crew_members": ["...", "..."],
    --   "equipment_used": ["dump_truck_14", "roller_07", "saw_cutter_03"],
    --   "materials_used": [
    --     {"item": "Hot mix asphalt", "quantity": 2.5, "unit": "tons"},
    --     {"item": "Tack coat", "quantity": 20, "unit": "liters"}
    --   ],
    --   "permits_required": false,
    --   "traffic_control": "lane_closure",
    --   "safety_notes": "Hard hat and reflective vest required"
    -- }
    
    -- JSONB: Task checklist (for PM work orders)
    task_checklist      JSONB,
    -- Example: [
    --   {"task": "Inspect area and mark boundaries", "completed": true, "completed_at": "..."},
    --   {"task": "Set up traffic control", "completed": true, "completed_at": "..."},
    --   {"task": "Saw-cut edges", "completed": true, "completed_at": "..."},
    --   {"task": "Remove damaged pavement", "completed": true, "completed_at": "..."},
    --   {"task": "Apply tack coat", "completed": false},
    --   {"task": "Place and compact hot mix", "completed": false}
    -- ]
    
    citizen_request_id  UUID,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),
    updated_by          UUID REFERENCES users(id),
    
    UNIQUE (organization_id, work_order_number)
);

CREATE INDEX idx_wo_asset ON work_orders (asset_id);
CREATE INDEX idx_wo_status ON work_orders (organization_id, status);
CREATE INDEX idx_wo_assigned ON work_orders (assigned_to, status);
CREATE INDEX idx_wo_details ON work_orders USING GIN (work_details jsonb_path_ops);

-- Work order costs (normalized -- financial data should be strict)
CREATE TABLE work_order_costs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id) ON DELETE CASCADE,
    cost_type       VARCHAR(30) NOT NULL,
    description     VARCHAR(500) NOT NULL,
    quantity        NUMERIC(12,3),
    unit_cost       NUMERIC(14,4),
    total_cost      NUMERIC(14,2) NOT NULL,
    cost_date       DATE NOT NULL DEFAULT CURRENT_DATE,
    gl_account_code VARCHAR(30),
    
    -- JSONB for additional cost metadata
    cost_metadata   JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {"vendor": "ABC Supplies", "po_number": "PO-2026-0142", "invoice_number": "INV-8821"}
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Financial Records (Normalized Core + JSONB Supplements)

```sql
CREATE TABLE asset_financials (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id            UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    
    -- Normalized financial core (critical for GASB 34 compliance)
    acquisition_date    DATE,
    acquisition_cost    NUMERIC(16,2),
    replacement_cost    NUMERIC(16,2),
    replacement_cost_date DATE,
    salvage_value       NUMERIC(16,2),
    
    depreciation_method VARCHAR(30),
    useful_life_years   INTEGER,
    accumulated_depreciation NUMERIC(16,2),
    net_book_value      NUMERIC(16,2),
    
    -- GL integration (normalized for financial system export)
    gl_asset_account    VARCHAR(30),
    gl_depreciation_account VARCHAR(30),
    gl_expense_account  VARCHAR(30),
    fund_code           VARCHAR(30),
    cost_center         VARCHAR(30),
    
    -- JSONB: Flexible financial metadata
    financial_metadata  JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {
    --   "funding_sources": [
    --     {"source": "General Fund", "amount": 800000, "pct": 0.64},
    --     {"source": "Federal Highway Grant HM-2020-045", "amount": 450000, "pct": 0.36}
    --   ],
    --   "grant_numbers": ["HM-2020-045"],
    --   "insurance_policy": "MUN-2026-INF-4521",
    --   "insured_value": 1250000,
    --   "valuation_history": [
    --     {"date": "2024-01-01", "replacement_cost": 1250000, "method": "unit_cost"},
    --     {"date": "2026-01-01", "replacement_cost": 1385000, "method": "unit_cost", "inflation_factor": 1.108}
    --   ],
    --   "erp_sync": {"system": "munis", "asset_id": "FA-2024-0891", "last_sync": "2026-05-01"}
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Capital projects
CREATE TABLE capital_projects (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    project_number      VARCHAR(30) NOT NULL,
    name                VARCHAR(500) NOT NULL,
    description         TEXT,
    project_type        VARCHAR(30),
    
    estimated_cost      NUMERIC(16,2),
    approved_budget     NUMERIC(16,2),
    actual_cost         NUMERIC(16,2),
    
    planned_start_date  DATE,
    planned_end_date    DATE,
    actual_start_date   DATE,
    actual_end_date     DATE,
    
    status              VARCHAR(30) NOT NULL DEFAULT 'proposed',
    fiscal_year         INTEGER NOT NULL,
    department_id       UUID REFERENCES departments(id),
    priority_score      NUMERIC(5,2),
    
    -- JSONB: Flexible project details
    project_details     JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {
    --   "funding_sources": [
    --     {"source": "Capital Fund", "amount": 1500000},
    --     {"source": "State STIP Grant", "amount": 900000}
    --   ],
    --   "justification": "PCI scores below 40 for 3 consecutive assessments...",
    --   "cost_of_deferral": 3200000,
    --   "benefit_cost_ratio": 2.13,
    --   "affected_population": 15000,
    --   "related_projects": ["CIP-2025-042"],
    --   "council_approval_date": "2026-02-15",
    --   "contractor": "ABC Construction LLC",
    --   "contract_number": "CON-2026-018"
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, project_number)
);

-- Many-to-many: assets in capital projects
CREATE TABLE capital_project_assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    capital_project_id  UUID NOT NULL REFERENCES capital_projects(id) ON DELETE CASCADE,
    asset_id            UUID NOT NULL REFERENCES assets(id),
    treatment_type      VARCHAR(100),
    estimated_cost      NUMERIC(14,2),
    notes               TEXT,
    UNIQUE (capital_project_id, asset_id)
);
```

### Deterioration Modeling and Scenario Planning

```sql
CREATE TABLE deterioration_models (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    asset_class_id      UUID NOT NULL REFERENCES asset_classes(id),
    model_name          VARCHAR(255) NOT NULL,
    model_type          VARCHAR(30) NOT NULL,
    
    -- JSONB: Full model specification
    model_spec          JSONB NOT NULL,
    -- Linear example: {
    --   "type": "linear",
    --   "coefficients": {"intercept": 100, "slope": -1.5},
    --   "variables": ["age_years"],
    --   "r_squared": 0.72
    -- }
    --
    -- Markov example: {
    --   "type": "markov",
    --   "states": ["excellent", "good", "fair", "poor", "failed"],
    --   "transition_matrix": [
    --     [0.85, 0.12, 0.03, 0.00, 0.00],
    --     [0.00, 0.80, 0.15, 0.05, 0.00],
    --     [0.00, 0.00, 0.78, 0.18, 0.04],
    --     [0.00, 0.00, 0.00, 0.75, 0.25],
    --     [0.00, 0.00, 0.00, 0.00, 1.00]
    --   ],
    --   "transition_period_years": 1,
    --   "training_sample_size": 4521,
    --   "material_adjustment_factors": {
    --     "asphalt": 1.0,
    --     "concrete": 0.85,
    --     "gravel": 1.3
    --   }
    -- }
    --
    -- AI/ML example: {
    --   "type": "ai_trained",
    --   "model_framework": "scikit-learn",
    --   "model_file": "s3://models/road_deterioration_v4.pkl",
    --   "features": ["age", "traffic_aadt", "material", "climate_zone", "last_treatment_type"],
    --   "training_metrics": {"rmse": 4.2, "mae": 3.1, "r2": 0.88},
    --   "training_date": "2026-03-01",
    --   "training_sample_size": 12000
    -- }
    
    is_active           BOOLEAN NOT NULL DEFAULT true,
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE planning_scenarios (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    
    -- JSONB: Full scenario configuration and results
    scenario_config     JSONB NOT NULL,
    -- Example: {
    --   "analysis_period_years": 20,
    --   "annual_budgets": [5000000, 5150000, 5304500, ...],
    --   "inflation_rate": 0.03,
    --   "discount_rate": 0.05,
    --   "budget_allocation_strategy": "worst_first",
    --   "treatment_options": [
    --     {"type": "preventive", "condition_range": [70, 85], "cost_per_m2": 15, "life_extension_years": 5},
    --     {"type": "rehabilitation", "condition_range": [40, 70], "cost_per_m2": 65, "life_extension_years": 12},
    --     {"type": "reconstruction", "condition_range": [0, 40], "cost_per_m2": 180, "life_extension_years": 25}
    --   ],
    --   "asset_classes_included": ["ROAD_ARTERIAL", "ROAD_COLLECTOR", "ROAD_LOCAL"]
    -- }
    
    scenario_results    JSONB,
    -- Example: {
    --   "yearly_projections": [
    --     {"year": 2026, "avg_condition": 68.2, "backlog": 45000000, "investment": 5000000, ...},
    --     {"year": 2027, "avg_condition": 67.5, "backlog": 48500000, "investment": 5150000, ...},
    --     ...
    --   ],
    --   "summary": {
    --     "total_investment": 115000000,
    --     "backlog_start": 45000000,
    --     "backlog_end": 125000000,
    --     "avg_condition_start": 68.2,
    --     "avg_condition_end": 52.1,
    --     "cost_of_deferral": 180000000,
    --     "assets_reaching_failure": 234
    --   }
    -- }
    
    status              VARCHAR(20) DEFAULT 'draft',
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Citizen Requests and Community Portal

```sql
CREATE TABLE citizen_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    request_number      VARCHAR(30) NOT NULL,
    
    -- Normalized contact info
    submitter_name      VARCHAR(255),
    submitter_email     VARCHAR(255),
    submitter_phone     VARCHAR(50),
    
    category            VARCHAR(100) NOT NULL,
    description         TEXT NOT NULL,
    geom_point          GEOMETRY(Point, 4326),
    
    asset_id            UUID REFERENCES assets(id),
    work_order_id       UUID REFERENCES work_orders(id),
    
    status              VARCHAR(20) NOT NULL DEFAULT 'submitted',
    resolution_notes    TEXT,
    
    -- JSONB: Flexible request metadata
    request_metadata    JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {
    --   "submission_channel": "web_portal",
    --   "photos": ["s3://citizen-uploads/2026/05/IMG_001.jpg"],
    --   "address_text": "Main St near Oak St intersection",
    --   "urgency_self_reported": "high",
    --   "ai_category_suggestion": "pothole",
    --   "ai_category_confidence": 0.94,
    --   "duplicate_check": {"possible_duplicate_of": "CR-2026-4518", "similarity": 0.87}
    -- }
    
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    acknowledged_at     TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    UNIQUE (organization_id, request_number)
);

CREATE INDEX idx_cr_geom ON citizen_requests USING GIST (geom_point);
CREATE INDEX idx_cr_status ON citizen_requests (organization_id, status);
```

### Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL,
    
    -- Store diff as JSONB for flexible querying
    changes         JSONB NOT NULL,
    -- Example: {
    --   "changed_fields": ["current_condition_score", "current_condition_date"],
    --   "old": {"current_condition_score": 71.0, "current_condition_date": "2024-04-10"},
    --   "new": {"current_condition_score": 62.5, "current_condition_date": "2026-04-15"}
    -- }
    
    user_id         UUID REFERENCES users(id),
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_table_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_org_date ON audit_log (organization_id, created_at DESC);
CREATE INDEX idx_audit_changes ON audit_log USING GIN (changes jsonb_path_ops);
```

### GIS Data Exchange

```sql
CREATE TABLE gis_data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    source_type     VARCHAR(30) NOT NULL,
    
    -- Connection and sync configuration in JSONB
    connection_config JSONB NOT NULL,
    -- OGC API Features example: {
    --   "url": "https://gis.municipality.gov/ogc/features/v1",
    --   "collection": "road_centerlines",
    --   "auth_type": "api_key",
    --   "auth_key_encrypted": "...",
    --   "sync_schedule": "0 2 * * *",
    --   "field_mapping": {
    --     "ROAD_NAME": "name",
    --     "SURF_TYPE": "attributes.surface_type",
    --     "NUM_LANES": "attributes.lane_count",
    --     "ROAD_CLASS": "attributes.road_class"
    --   },
    --   "crs": "EPSG:4326",
    --   "bbox_filter": [-73.99, 40.74, -73.97, 40.76]
    -- }
    
    last_sync_at    TIMESTAMPTZ,
    last_sync_status VARCHAR(20),
    
    -- Sync results
    sync_history    JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- Example: [
    --   {"date": "2026-05-01", "status": "success", "records": 1247, "created": 3, "updated": 12, "errors": 0},
    --   {"date": "2026-04-01", "status": "success", "records": 1244, "created": 5, "updated": 8, "errors": 1}
    -- ]
    
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Query Examples

```sql
-- Find all roads with asphalt surface in poor condition
SELECT a.asset_number, a.name, a.current_condition_score,
       a.attributes->>'road_class' AS road_class,
       a.attributes->>'surface_type' AS surface_type,
       (a.attributes->>'traffic_volume_aadt')::int AS aadt
FROM assets a
JOIN asset_classes ac ON a.asset_class_id = ac.id
WHERE a.organization_id = :org_id
  AND ac.code = 'ROAD'
  AND a.attributes @> '{"surface_type": "asphalt"}'::jsonb
  AND a.current_condition_score < 40
ORDER BY a.current_condition_score ASC;

-- Find all bridges with scour-critical status
SELECT a.asset_number, a.name,
       a.attributes->>'nbi_structure_number' AS nbi_number,
       a.attributes->>'bridge_type' AS bridge_type,
       (a.attributes->>'load_rating_tons')::numeric AS load_rating
FROM assets a
WHERE a.organization_id = :org_id
  AND a.attributes @> '{"scour_critical": true}'::jsonb;

-- Find all pipes of a specific material within 500m of a point
SELECT a.asset_number, a.name,
       a.attributes->>'pipe_system' AS system,
       a.attributes->>'pipe_material' AS material,
       (a.attributes->>'nominal_diameter_mm')::int AS diameter,
       ST_Distance(a.geom_line::geography, ST_MakePoint(-73.985, 40.748)::geography) AS distance_m
FROM assets a
WHERE a.organization_id = :org_id
  AND a.attributes @> '{"pipe_material": "cast_iron"}'::jsonb
  AND ST_DWithin(a.geom_line::geography, ST_MakePoint(-73.985, 40.748)::geography, 500);

-- Aggregate condition by road class using JSONB attribute
SELECT a.attributes->>'road_class' AS road_class,
       COUNT(*) AS asset_count,
       AVG(a.current_condition_score) AS avg_condition,
       SUM(af.replacement_cost) AS total_replacement_value
FROM assets a
JOIN asset_financials af ON a.id = af.asset_id
WHERE a.organization_id = :org_id
  AND a.attributes ? 'road_class'
GROUP BY a.attributes->>'road_class'
ORDER BY avg_condition ASC;

-- Search custom fields across organizations
SELECT a.asset_number, a.name,
       a.custom_fields->>'council_district' AS district,
       a.custom_fields->>'snow_route_priority' AS snow_priority
FROM assets a
WHERE a.organization_id = :org_id
  AND a.custom_fields @> '{"snow_route_priority": "A"}'::jsonb;
```

---

## Pros and Cons

### Pros

1. **Asset Type Flexibility Without Schema Changes**: Adding a new asset type (e.g., stormwater detention ponds, electric vehicle charging stations) requires only a new `asset_classes` row with its `attribute_schema` -- no DDL migration. This is critical for a product serving diverse municipalities.

2. **Municipal Customization Built-In**: The `custom_fields` JSONB column lets each organization add fields specific to their needs without affecting other organizations or requiring code changes. Council districts, snow routes, historical designations -- all accommodated without schema changes.

3. **Condition Framework Adaptability**: The entire PCI distress catalog, NBI element definitions, and NASSCO PACP codes are stored as JSONB configuration. New standards or updates to existing standards are handled by updating the framework configuration, not migrating tables.

4. **Strong Financial Integrity**: Financial data (acquisition costs, depreciation, GL accounts) remains in normalized columns with proper NUMERIC types and referential integrity. GASB 34 reporting draws from well-typed, indexed columns.

5. **Spatial Performance Preserved**: Geometry columns remain native PostGIS types with GIST indexes, avoiding the performance penalty of storing coordinates in JSONB.

6. **Single Database Technology**: PostgreSQL handles relational data, document data, and spatial data in one system. No need to operate and synchronize multiple databases.

7. **Progressive Schema Strictness**: Start with JSONB attributes for a new asset type, then promote frequently queried attributes to expression indexes as usage patterns emerge. The schema evolves with usage data.

8. **AI Output Storage**: AI model results (condition scoring, distress detection, NLP analysis) are naturally variable and evolving. JSONB columns accommodate new model outputs without migrations.

9. **Simpler Import**: Legacy data with inconsistent column structures maps naturally into JSONB without forcing every field into a strict column. Data quality improves incrementally.

### Cons

1. **Weaker Type Safety**: JSONB columns do not enforce types at the database level. A string "42" and an integer 42 are different values. Application-layer JSON Schema validation is required but can be bypassed.

2. **JOIN Limitations**: You cannot FOREIGN KEY into a JSONB value. If `attributes.upstream_node_id` references another asset, this relationship is not enforced by the database.

3. **Index Complexity**: GIN indexes on JSONB support containment queries (`@>`) efficiently but not range queries on nested values. Expression indexes (`(attributes->>'traffic_volume_aadt')::int`) are needed for each frequently-queried key, creating an index management burden.

4. **Query Verbosity**: Accessing JSONB fields requires operator syntax (`->`, `->>`, `@>`, `?`) that is less readable than column references. Developers must understand PostgreSQL JSONB operators.

5. **Schema Discovery**: With normalized tables, `\d assets` shows all columns. With JSONB, developers must consult `attribute_schema` definitions or documentation to know what keys exist. Tooling and discipline are needed.

6. **Reporting Complexity**: Business intelligence tools (Metabase, Grafana, Tableau) work best with flat columns. JSONB data must be unpacked with expressions in queries or views, which is harder for non-technical report builders.

7. **Storage Overhead**: JSONB stores key names with every row. An attribute like `"traffic_volume_aadt": 12500` stores the key string in every row, whereas a column named `traffic_volume_aadt` stores it once in the schema catalog.

8. **Migration from JSONB to Columns**: If a JSONB attribute needs to be promoted to a proper column (for performance, integrity, or reporting reasons), backfilling requires a data migration.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with PostGIS 3.4+, ltree, pg_trgm extensions |
| **JSONB Validation** | Application-layer JSON Schema validation (e.g., `ajv` in Node.js, `jsonschema` in Python). Store schemas in `asset_classes.attribute_schema`. |
| **Indexing Strategy** | GIN with `jsonb_path_ops` on all JSONB columns used for queries. Expression indexes on top 5-10 most-queried attribute keys per asset class. |
| **API Layer** | GraphQL (excellent for variable-shape data) or REST with OGC API Features for spatial endpoints |
| **Full-Text Search** | pg_trgm for fuzzy asset name search. Consider extracting searchable text from JSONB into a tsvector column for full-text search. |
| **BI/Reporting** | Create views that unpack JSONB into flat columns for each asset class. BI tools connect to these views. |
| **Schema Registry** | Store JSON Schema definitions in `asset_classes.attribute_schema`. Version them and validate on write. |
| **Object Storage** | MinIO or S3 for inspection photos and documents. Store only file references in the database. |

---

## Migration and Scaling Considerations

### Initial Data Migration

1. **Spreadsheet import advantage**: Legacy spreadsheets with inconsistent columns map naturally into JSONB. Each row becomes an asset with its varied columns stored as key-value pairs in the `attributes` field. No need to predefine every possible column.

2. **AI-assisted field mapping**: The AI import wizard can map source columns to either normalized fields (name, installation_date, material) or JSONB attributes (surface_type, lane_count), guided by the target asset class's `attribute_schema`.

3. **Data quality tracking**: Import quality issues are recorded in `import_metadata` JSONB, allowing incremental data cleanup without blocking the initial import.

### Scaling Strategy

1. **0-500K assets**: Single PostgreSQL instance with GIN and expression indexes on JSONB. This handles the vast majority of municipal deployments.

2. **500K-5M assets**: Add read replicas for map rendering and reporting. Materialized views for dashboard metrics, refreshed on schedule. Consider partitioning assets by organization_id.

3. **5M+ assets**: Deploy Citus for distributed PostgreSQL. The `organization_id` column serves as the distribution key, keeping all of one organization's data on the same shard.

### Schema Evolution Strategy

- **Adding new asset types**: Insert a new `asset_classes` row with `attribute_schema`. No DDL changes.
- **Adding custom fields**: Organizations add keys to `custom_fields` JSONB through the UI. No DDL changes.
- **Promoting JSONB to columns**: When a JSONB attribute is queried frequently across all organizations (e.g., it becomes a standard filter), promote it to a proper column with a backfill migration.
- **Updating condition frameworks**: Update `framework_config` JSONB in `condition_frameworks`. Existing inspections retain their original framework version.
- **Version tracking**: Store `schema_version` in each `attribute_schema` definition. Application code handles backward compatibility.

### Performance Tuning

- **GIN index selectivity**: GIN indexes work best with selective containment queries. For low-selectivity keys (e.g., `status`), use B-tree expression indexes instead.
- **JSONB size monitoring**: Monitor average JSONB document size. If `attributes` grows beyond 10KB per row, consider extracting large nested structures into separate tables.
- **Expression index pruning**: Audit expression indexes quarterly. Remove indexes on keys that are rarely queried.
- **TOAST compression**: PostgreSQL automatically compresses large JSONB values using TOAST. For very large documents (AI analysis results), consider storing them in a separate table to avoid row bloat.
