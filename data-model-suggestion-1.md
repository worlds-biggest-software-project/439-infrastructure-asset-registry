# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL + PostGIS)

> Project: Infrastructure Asset Registry (Candidate #439)
> Generated: 2026-05-25

## Overview

This model uses a fully normalized relational schema in PostgreSQL with the PostGIS extension for spatial data. Every entity is decomposed into well-defined tables with foreign key relationships, enforcing referential integrity across assets, inspections, work orders, financial records, and GIS geometry. This is the most traditional and well-understood approach, mapping directly to the ISO 55001 asset management lifecycle and GASB 34 reporting requirements.

## Design Principles

1. **Third Normal Form (3NF)** for all core entities to eliminate data redundancy
2. **PostGIS geometry columns** on every spatially-located entity with GIST indexes
3. **Polymorphic asset types** handled via a base `assets` table with type-specific child tables (table-per-subclass inheritance)
4. **Temporal tracking** via explicit `valid_from`/`valid_to` columns on condition and financial records
5. **Multi-tenant isolation** via an `organization_id` column on all tables, supporting multi-municipality deployments
6. **Audit columns** (`created_at`, `updated_at`, `created_by`, `updated_by`) on every table

---

## Core Schema

### Organizations and Users

```sql
-- Multi-tenant organization support
CREATE TABLE organizations (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name              VARCHAR(255) NOT NULL,
    org_type          VARCHAR(50) NOT NULL CHECK (org_type IN ('municipality', 'county', 'utility', 'state_dot', 'school_district', 'special_district')),
    jurisdiction_code VARCHAR(20),
    state_province    VARCHAR(100),
    country_code      CHAR(2) NOT NULL DEFAULT 'US',
    timezone          VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    fiscal_year_start INTEGER NOT NULL DEFAULT 1 CHECK (fiscal_year_start BETWEEN 1 AND 12),
    gasb34_approach   VARCHAR(20) CHECK (gasb34_approach IN ('depreciation', 'modified', 'both')),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Departments within an organization
CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    department_type VARCHAR(50) CHECK (department_type IN ('public_works', 'transportation', 'water', 'wastewater', 'stormwater', 'parks', 'facilities', 'fleet', 'finance', 'engineering')),
    parent_id       UUID REFERENCES departments(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

-- Users and roles
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    user_role       VARCHAR(30) NOT NULL CHECK (user_role IN ('admin', 'manager', 'supervisor', 'inspector', 'field_crew', 'finance', 'viewer', 'citizen')),
    department_id   UUID REFERENCES departments(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    permission      VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50),
    resource_id     UUID,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES users(id),
    UNIQUE (user_id, permission, resource_type, resource_id)
);
```

### Asset Classification Taxonomy

```sql
-- Asset classification hierarchy (e.g., Infrastructure > Transportation > Roads > Arterial)
CREATE TABLE asset_classes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    code            VARCHAR(30) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES asset_classes(id),
    hierarchy_level INTEGER NOT NULL DEFAULT 0,
    asset_category  VARCHAR(30) NOT NULL CHECK (asset_category IN ('horizontal', 'vertical', 'point', 'linear', 'area')),
    description     TEXT,
    default_useful_life_years INTEGER,
    depreciation_method VARCHAR(30) CHECK (depreciation_method IN ('straight_line', 'declining_balance', 'units_of_production', 'none')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

-- Attribute definitions per asset class (defines which attributes an asset type has)
CREATE TABLE asset_class_attributes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_class_id  UUID NOT NULL REFERENCES asset_classes(id) ON DELETE CASCADE,
    attribute_name  VARCHAR(100) NOT NULL,
    attribute_label VARCHAR(255) NOT NULL,
    data_type       VARCHAR(30) NOT NULL CHECK (data_type IN ('text', 'integer', 'decimal', 'boolean', 'date', 'select', 'multiselect')),
    unit_of_measure VARCHAR(50),
    is_required     BOOLEAN NOT NULL DEFAULT false,
    select_options  TEXT[], -- For select/multiselect types
    display_order   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (asset_class_id, attribute_name)
);
```

### Core Asset Registry

```sql
-- Base asset table: every infrastructure asset in the registry
CREATE TABLE assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    asset_class_id      UUID NOT NULL REFERENCES asset_classes(id),
    department_id       UUID REFERENCES departments(id),
    
    -- Identification
    asset_number        VARCHAR(50) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    barcode             VARCHAR(100),
    
    -- Hierarchy (network > segment > component)
    parent_asset_id     UUID REFERENCES assets(id),
    hierarchy_level     INTEGER NOT NULL DEFAULT 0,
    hierarchy_path      LTREE, -- PostgreSQL ltree for materialized path queries
    
    -- Location
    address             VARCHAR(500),
    locality            VARCHAR(255),
    region              VARCHAR(255),
    postal_code         VARCHAR(20),
    location_description TEXT,
    
    -- Spatial geometry (PostGIS)
    geom_point          GEOMETRY(Point, 4326),       -- For point assets (signals, hydrants)
    geom_line           GEOMETRY(LineString, 4326),   -- For linear assets (roads, pipes)
    geom_polygon        GEOMETRY(Polygon, 4326),      -- For area assets (parks, buildings)
    
    -- Physical characteristics
    material            VARCHAR(100),
    manufacturer        VARCHAR(255),
    model_number        VARCHAR(255),
    dimensions_length_m NUMERIC(12,3),
    dimensions_width_m  NUMERIC(12,3),
    dimensions_height_m NUMERIC(12,3),
    dimensions_area_m2  NUMERIC(14,3),
    diameter_mm         NUMERIC(10,2),
    capacity            NUMERIC(14,3),
    capacity_unit       VARCHAR(30),
    
    -- Lifecycle
    installation_date   DATE,
    commissioning_date  DATE,
    warranty_expiry     DATE,
    expected_useful_life_years INTEGER,
    expected_end_of_life DATE,
    decommission_date   DATE,
    
    -- Status
    status              VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN ('planned', 'under_construction', 'active', 'inactive', 'decommissioned', 'disposed')),
    criticality_rating  INTEGER CHECK (criticality_rating BETWEEN 1 AND 5),
    risk_score          NUMERIC(5,2),
    
    -- Current condition (denormalized for query performance, updated by triggers)
    current_condition_score NUMERIC(5,2),
    current_condition_date  DATE,
    current_condition_rating VARCHAR(20),
    
    -- Metadata
    data_source         VARCHAR(100),
    data_quality_score  NUMERIC(3,2) CHECK (data_quality_score BETWEEN 0 AND 1),
    imported_from       VARCHAR(255),
    external_id         VARCHAR(255),
    notes               TEXT,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),
    updated_by          UUID REFERENCES users(id),
    
    UNIQUE (organization_id, asset_number)
);

-- Spatial indexes
CREATE INDEX idx_assets_geom_point ON assets USING GIST (geom_point);
CREATE INDEX idx_assets_geom_line ON assets USING GIST (geom_line);
CREATE INDEX idx_assets_geom_polygon ON assets USING GIST (geom_polygon);

-- Hierarchy index
CREATE INDEX idx_assets_hierarchy_path ON assets USING GIST (hierarchy_path);

-- Common query indexes
CREATE INDEX idx_assets_org_class ON assets (organization_id, asset_class_id);
CREATE INDEX idx_assets_status ON assets (organization_id, status);
CREATE INDEX idx_assets_parent ON assets (parent_asset_id);
CREATE INDEX idx_assets_department ON assets (department_id);

-- Extended attributes (EAV pattern for class-specific attributes)
CREATE TABLE asset_attributes (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id                UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    asset_class_attribute_id UUID NOT NULL REFERENCES asset_class_attributes(id),
    value_text              TEXT,
    value_numeric           NUMERIC(18,6),
    value_boolean           BOOLEAN,
    value_date              DATE,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (asset_id, asset_class_attribute_id)
);
```

### Type-Specific Asset Tables (Table-Per-Subclass)

```sql
-- Road segments
CREATE TABLE asset_roads (
    asset_id            UUID PRIMARY KEY REFERENCES assets(id) ON DELETE CASCADE,
    road_class          VARCHAR(30) CHECK (road_class IN ('interstate', 'arterial', 'collector', 'local', 'alley', 'private')),
    functional_class    VARCHAR(50),
    surface_type        VARCHAR(50) CHECK (surface_type IN ('asphalt', 'concrete', 'gravel', 'brick', 'unpaved', 'composite')),
    lane_count          INTEGER,
    speed_limit_kph     INTEGER,
    traffic_volume_aadt INTEGER,
    pavement_thickness_mm NUMERIC(8,2),
    base_type           VARCHAR(50),
    curb_type           VARCHAR(30),
    sidewalk_present    BOOLEAN DEFAULT false,
    last_overlay_date   DATE,
    last_reconstruction_date DATE
);

-- Bridges and structures
CREATE TABLE asset_bridges (
    asset_id            UUID PRIMARY KEY REFERENCES assets(id) ON DELETE CASCADE,
    nbi_structure_number VARCHAR(15),
    bridge_type         VARCHAR(50) CHECK (bridge_type IN ('beam', 'girder', 'truss', 'arch', 'suspension', 'cable_stayed', 'culvert', 'tunnel')),
    deck_type           VARCHAR(50),
    superstructure_type VARCHAR(50),
    substructure_type   VARCHAR(50),
    span_count          INTEGER,
    max_span_length_m   NUMERIC(10,2),
    total_length_m      NUMERIC(10,2),
    deck_width_m        NUMERIC(8,2),
    vertical_clearance_m NUMERIC(6,2),
    load_rating_tons    NUMERIC(8,2),
    posting_status      VARCHAR(30),
    waterway_adequacy   INTEGER CHECK (waterway_adequacy BETWEEN 0 AND 9),
    scour_critical      BOOLEAN DEFAULT false,
    fracture_critical   BOOLEAN DEFAULT false,
    year_built          INTEGER,
    year_reconstructed  INTEGER
);

-- Water/sewer pipes
CREATE TABLE asset_pipes (
    asset_id            UUID PRIMARY KEY REFERENCES assets(id) ON DELETE CASCADE,
    pipe_system         VARCHAR(30) CHECK (pipe_system IN ('water_main', 'water_service', 'sanitary_sewer', 'storm_sewer', 'combined_sewer', 'force_main', 'gas', 'other')),
    pipe_material       VARCHAR(50) CHECK (pipe_material IN ('pvc', 'hdpe', 'ductile_iron', 'cast_iron', 'concrete', 'vitrified_clay', 'steel', 'copper', 'galvanized', 'asbestos_cement', 'corrugated_metal', 'other')),
    nominal_diameter_mm INTEGER,
    slope_percent       NUMERIC(6,4),
    invert_elevation_up_m NUMERIC(10,3),
    invert_elevation_down_m NUMERIC(10,3),
    lining_type         VARCHAR(50),
    lining_date         DATE,
    pressure_rating_kpa NUMERIC(10,2),
    flow_direction      VARCHAR(20) CHECK (flow_direction IN ('upstream', 'downstream', 'bidirectional')),
    upstream_node_id    UUID REFERENCES assets(id),
    downstream_node_id  UUID REFERENCES assets(id)
);

-- Traffic signals and signs
CREATE TABLE asset_signals (
    asset_id            UUID PRIMARY KEY REFERENCES assets(id) ON DELETE CASCADE,
    signal_type         VARCHAR(30) CHECK (signal_type IN ('traffic_signal', 'pedestrian_signal', 'flashing_beacon', 'railroad_crossing', 'school_zone', 'ramp_meter')),
    controller_type     VARCHAR(50),
    detection_type      VARCHAR(50),
    pole_type           VARCHAR(30),
    pole_height_m       NUMERIC(6,2),
    power_source        VARCHAR(30),
    communication_type  VARCHAR(50),
    is_ada_compliant    BOOLEAN DEFAULT false,
    pedestrian_features TEXT[],
    intersection_id     UUID REFERENCES assets(id)
);

-- Buildings and facilities
CREATE TABLE asset_facilities (
    asset_id            UUID PRIMARY KEY REFERENCES assets(id) ON DELETE CASCADE,
    facility_type       VARCHAR(50) CHECK (facility_type IN ('office', 'fire_station', 'police_station', 'library', 'community_center', 'pump_station', 'treatment_plant', 'storage', 'garage', 'park_building', 'school', 'other')),
    floor_count         INTEGER,
    gross_area_m2       NUMERIC(12,2),
    net_usable_area_m2  NUMERIC(12,2),
    year_built          INTEGER,
    year_renovated      INTEGER,
    occupancy_capacity  INTEGER,
    hvac_type           VARCHAR(100),
    roof_type           VARCHAR(50),
    roof_install_date   DATE,
    energy_rating       VARCHAR(20),
    ada_compliant       BOOLEAN DEFAULT false,
    fire_suppression    VARCHAR(50),
    emergency_generator BOOLEAN DEFAULT false
);

-- Culverts
CREATE TABLE asset_culverts (
    asset_id            UUID PRIMARY KEY REFERENCES assets(id) ON DELETE CASCADE,
    culvert_type        VARCHAR(30) CHECK (culvert_type IN ('pipe', 'box', 'arch', 'bridge', 'other')),
    culvert_material    VARCHAR(50),
    shape               VARCHAR(30) CHECK (shape IN ('circular', 'elliptical', 'box', 'arch', 'other')),
    span_m              NUMERIC(8,2),
    rise_m              NUMERIC(8,2),
    barrel_count        INTEGER,
    inlet_type          VARCHAR(50),
    outlet_type         VARCHAR(50),
    headwall_present    BOOLEAN,
    wingwall_present    BOOLEAN,
    stream_name         VARCHAR(255),
    drainage_area_km2   NUMERIC(12,4)
);
```

### Condition Inspection System

```sql
-- Condition rating frameworks (PCI, NBI, ACI, custom)
CREATE TABLE condition_frameworks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    code            VARCHAR(30) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    standard_ref    VARCHAR(100), -- e.g., 'ASTM D6433', 'FHWA NBI', 'NASSCO PACP'
    score_min       NUMERIC(8,2) NOT NULL,
    score_max       NUMERIC(8,2) NOT NULL,
    score_direction VARCHAR(10) NOT NULL CHECK (score_direction IN ('asc', 'desc')), -- asc = higher is better (PCI), desc = lower is better
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

-- Rating bands within a framework (e.g., PCI: 0-10=Failed, 11-25=Serious, etc.)
CREATE TABLE condition_rating_bands (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    condition_framework_id  UUID NOT NULL REFERENCES condition_frameworks(id) ON DELETE CASCADE,
    band_label              VARCHAR(50) NOT NULL,
    score_lower             NUMERIC(8,2) NOT NULL,
    score_upper             NUMERIC(8,2) NOT NULL,
    color_code              VARCHAR(7), -- Hex color for map display
    display_order           INTEGER NOT NULL,
    treatment_guidance      TEXT,
    UNIQUE (condition_framework_id, band_label)
);

-- Distress types within a framework (e.g., PCI has 21 asphalt distress types)
CREATE TABLE distress_types (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    condition_framework_id  UUID NOT NULL REFERENCES condition_frameworks(id) ON DELETE CASCADE,
    code                    VARCHAR(10) NOT NULL,
    name                    VARCHAR(255) NOT NULL,
    description             TEXT,
    severity_levels         TEXT[] NOT NULL, -- e.g., {'low', 'medium', 'high'}
    measurement_unit        VARCHAR(30), -- e.g., 'sq_m', 'linear_m', 'count'
    deduct_value_curve_id   VARCHAR(50), -- Reference to deduct value lookup
    display_order           INTEGER NOT NULL,
    UNIQUE (condition_framework_id, code)
);

-- Inspection records
CREATE TABLE inspections (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id         UUID NOT NULL REFERENCES organizations(id),
    asset_id                UUID NOT NULL REFERENCES assets(id),
    condition_framework_id  UUID NOT NULL REFERENCES condition_frameworks(id),
    
    -- Inspection details
    inspection_type         VARCHAR(30) NOT NULL CHECK (inspection_type IN ('routine', 'detailed', 'emergency', 'follow_up', 'ai_assisted', 'sensor_based')),
    inspection_date         DATE NOT NULL,
    inspected_by            UUID REFERENCES users(id),
    
    -- Scoring
    overall_score           NUMERIC(8,2),
    rating_band             VARCHAR(50),
    confidence_score        NUMERIC(3,2), -- For AI-assisted: confidence 0.00-1.00
    
    -- Environmental context
    weather_conditions      VARCHAR(100),
    temperature_c           NUMERIC(5,1),
    
    -- Location within asset (for linear assets, station/offset)
    sample_unit_id          VARCHAR(50),
    station_start_m         NUMERIC(12,3),
    station_end_m           NUMERIC(12,3),
    
    -- Status
    status                  VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'submitted', 'reviewed', 'approved', 'rejected')),
    reviewed_by             UUID REFERENCES users(id),
    reviewed_at             TIMESTAMPTZ,
    
    notes                   TEXT,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspections_asset ON inspections (asset_id, inspection_date DESC);
CREATE INDEX idx_inspections_org_date ON inspections (organization_id, inspection_date DESC);

-- Individual distress observations within an inspection
CREATE TABLE inspection_distresses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id       UUID NOT NULL REFERENCES inspections(id) ON DELETE CASCADE,
    distress_type_id    UUID NOT NULL REFERENCES distress_types(id),
    severity            VARCHAR(20) NOT NULL,
    quantity            NUMERIC(12,3) NOT NULL,
    density_percent     NUMERIC(6,3),
    deduct_value        NUMERIC(8,3),
    location_description TEXT,
    geom_point          GEOMETRY(Point, 4326), -- Specific location of distress
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Inspection photos and attachments
CREATE TABLE inspection_media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspections(id) ON DELETE CASCADE,
    media_type      VARCHAR(20) NOT NULL CHECK (media_type IN ('photo', 'video', 'document', 'audio')),
    file_path       VARCHAR(500) NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    caption         TEXT,
    taken_at        TIMESTAMPTZ,
    geom_point      GEOMETRY(Point, 4326),
    ai_analysis     TEXT, -- AI condition scoring notes
    ai_score        NUMERIC(5,2),
    ai_confidence   NUMERIC(3,2),
    display_order   INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Work Order Management

```sql
-- Work order templates for preventive maintenance
CREATE TABLE work_order_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    asset_class_id  UUID REFERENCES asset_classes(id),
    work_type       VARCHAR(30) NOT NULL,
    estimated_hours NUMERIC(8,2),
    estimated_cost  NUMERIC(14,2),
    task_checklist  TEXT[],
    required_skills TEXT[],
    required_materials TEXT[],
    recurrence_rule VARCHAR(100), -- iCal RRULE for PM scheduling
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Work orders
CREATE TABLE work_orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    work_order_number   VARCHAR(30) NOT NULL,
    asset_id            UUID NOT NULL REFERENCES assets(id),
    template_id         UUID REFERENCES work_order_templates(id),
    inspection_id       UUID REFERENCES inspections(id), -- Triggered by inspection
    
    -- Classification
    work_type           VARCHAR(30) NOT NULL CHECK (work_type IN ('preventive', 'corrective', 'emergency', 'capital', 'inspection', 'rehabilitation', 'replacement')),
    priority            VARCHAR(20) NOT NULL CHECK (priority IN ('critical', 'high', 'medium', 'low', 'planned')),
    
    -- Description
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    failure_code        VARCHAR(50),
    cause_code          VARCHAR(50),
    remedy_code         VARCHAR(50),
    
    -- Assignment
    assigned_department_id UUID REFERENCES departments(id),
    assigned_to         UUID REFERENCES users(id),
    assigned_crew       VARCHAR(100),
    
    -- Scheduling
    requested_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'assigned', 'in_progress', 'on_hold', 'completed', 'closed', 'cancelled')),
    completion_notes    TEXT,
    
    -- Location
    location_description TEXT,
    geom_point          GEOMETRY(Point, 4326),
    
    -- Citizen request linkage
    citizen_request_id  UUID,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),
    updated_by          UUID REFERENCES users(id),
    
    UNIQUE (organization_id, work_order_number)
);

CREATE INDEX idx_work_orders_asset ON work_orders (asset_id);
CREATE INDEX idx_work_orders_status ON work_orders (organization_id, status);
CREATE INDEX idx_work_orders_assigned ON work_orders (assigned_to, status);
CREATE INDEX idx_work_orders_scheduled ON work_orders (organization_id, scheduled_start);

-- Work order costs (labor, materials, equipment, contractor)
CREATE TABLE work_order_costs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id) ON DELETE CASCADE,
    cost_type       VARCHAR(30) NOT NULL CHECK (cost_type IN ('labor', 'material', 'equipment', 'contractor', 'overhead', 'other')),
    description     VARCHAR(500) NOT NULL,
    quantity        NUMERIC(12,3),
    unit_cost       NUMERIC(14,4),
    total_cost      NUMERIC(14,2) NOT NULL,
    cost_date       DATE NOT NULL DEFAULT CURRENT_DATE,
    gl_account_code VARCHAR(30),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_costs_order ON work_order_costs (work_order_id);

-- Work order labor records
CREATE TABLE work_order_labor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    work_date       DATE NOT NULL,
    hours_regular   NUMERIC(6,2) NOT NULL DEFAULT 0,
    hours_overtime  NUMERIC(6,2) NOT NULL DEFAULT 0,
    labor_rate      NUMERIC(10,2),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Financial and Capital Planning

```sql
-- Asset financial records (acquisition, valuation, depreciation)
CREATE TABLE asset_financials (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id            UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    
    -- Acquisition
    acquisition_date    DATE,
    acquisition_cost    NUMERIC(16,2),
    funding_source      VARCHAR(100),
    grant_number        VARCHAR(100),
    
    -- Valuation
    replacement_cost    NUMERIC(16,2),
    replacement_cost_date DATE,
    salvage_value       NUMERIC(16,2),
    insured_value       NUMERIC(16,2),
    
    -- Depreciation (for GASB 34 depreciation approach)
    depreciation_method VARCHAR(30) CHECK (depreciation_method IN ('straight_line', 'declining_balance', 'units_of_production')),
    useful_life_years   INTEGER,
    accumulated_depreciation NUMERIC(16,2),
    net_book_value      NUMERIC(16,2),
    
    -- Cost tracking
    total_maintenance_cost NUMERIC(16,2) DEFAULT 0,
    total_capital_cost     NUMERIC(16,2) DEFAULT 0,
    annual_operating_cost  NUMERIC(14,2),
    
    -- GL integration
    gl_asset_account    VARCHAR(30),
    gl_depreciation_account VARCHAR(30),
    gl_expense_account  VARCHAR(30),
    fund_code           VARCHAR(30),
    cost_center         VARCHAR(30),
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- GASB 34 condition assessment records (for modified approach)
CREATE TABLE gasb34_condition_assessments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    asset_class_id      UUID NOT NULL REFERENCES asset_classes(id),
    assessment_year     INTEGER NOT NULL,
    
    -- Condition data
    target_condition_level NUMERIC(5,2) NOT NULL,
    actual_condition_level NUMERIC(5,2) NOT NULL,
    condition_met       BOOLEAN GENERATED ALWAYS AS (actual_condition_level >= target_condition_level) STORED,
    
    -- Financial data
    estimated_preservation_cost NUMERIC(16,2),
    actual_preservation_cost    NUMERIC(16,2),
    
    -- Assessment metadata
    assessment_method   TEXT NOT NULL,
    assessor            VARCHAR(255),
    assessment_date     DATE NOT NULL,
    
    -- Assets covered
    total_assets_assessed INTEGER,
    total_replacement_value NUMERIC(18,2),
    
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE (organization_id, asset_class_id, assessment_year)
);

-- Capital Improvement Projects (CIP)
CREATE TABLE capital_projects (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    project_number      VARCHAR(30) NOT NULL,
    name                VARCHAR(500) NOT NULL,
    description         TEXT,
    
    -- Classification
    project_type        VARCHAR(30) CHECK (project_type IN ('new_construction', 'rehabilitation', 'replacement', 'expansion', 'study', 'other')),
    priority_score      NUMERIC(5,2),
    
    -- Budget
    estimated_cost      NUMERIC(16,2),
    approved_budget     NUMERIC(16,2),
    actual_cost         NUMERIC(16,2),
    funding_sources     TEXT[], -- Array of funding source descriptions
    
    -- Timeline
    planned_start_date  DATE,
    planned_end_date    DATE,
    actual_start_date   DATE,
    actual_end_date     DATE,
    
    -- Status
    status              VARCHAR(30) NOT NULL DEFAULT 'proposed' CHECK (status IN ('proposed', 'approved', 'funded', 'in_design', 'bidding', 'under_construction', 'completed', 'deferred', 'cancelled')),
    
    fiscal_year         INTEGER NOT NULL,
    department_id       UUID REFERENCES departments(id),
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE (organization_id, project_number)
);

-- Link assets to capital projects (many-to-many)
CREATE TABLE capital_project_assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    capital_project_id  UUID NOT NULL REFERENCES capital_projects(id) ON DELETE CASCADE,
    asset_id            UUID NOT NULL REFERENCES assets(id),
    treatment_type      VARCHAR(100),
    estimated_cost      NUMERIC(14,2),
    notes               TEXT,
    UNIQUE (capital_project_id, asset_id)
);

-- Deterioration models per asset class
CREATE TABLE deterioration_models (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    asset_class_id      UUID NOT NULL REFERENCES asset_classes(id),
    model_name          VARCHAR(255) NOT NULL,
    model_type          VARCHAR(30) CHECK (model_type IN ('linear', 'exponential', 'polynomial', 'markov', 'weibull', 'ai_trained')),
    
    -- Model parameters (stored as structured data)
    parameters          TEXT NOT NULL, -- JSON string of model coefficients
    r_squared           NUMERIC(5,4),
    training_sample_size INTEGER,
    
    -- Validity
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Scenario models for capital planning
CREATE TABLE planning_scenarios (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    
    -- Scenario parameters
    analysis_period_years INTEGER NOT NULL,
    annual_budget       NUMERIC(16,2),
    inflation_rate      NUMERIC(5,4) DEFAULT 0.03,
    discount_rate       NUMERIC(5,4) DEFAULT 0.05,
    
    -- Results (computed)
    total_investment    NUMERIC(18,2),
    total_backlog_end   NUMERIC(18,2),
    avg_condition_end   NUMERIC(5,2),
    cost_of_deferral    NUMERIC(18,2),
    
    status              VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft', 'running', 'completed', 'archived')),
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Citizen and Stakeholder Access

```sql
-- Citizen-submitted service requests
CREATE TABLE citizen_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    request_number      VARCHAR(30) NOT NULL,
    
    -- Submitter info
    submitter_name      VARCHAR(255),
    submitter_email     VARCHAR(255),
    submitter_phone     VARCHAR(50),
    
    -- Request details
    category            VARCHAR(100) NOT NULL,
    description         TEXT NOT NULL,
    location_description TEXT,
    geom_point          GEOMETRY(Point, 4326),
    
    -- Linkage
    asset_id            UUID REFERENCES assets(id),
    work_order_id       UUID REFERENCES work_orders(id),
    
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'submitted' CHECK (status IN ('submitted', 'acknowledged', 'investigating', 'scheduled', 'resolved', 'closed')),
    resolution_notes    TEXT,
    
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    acknowledged_at     TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    
    UNIQUE (organization_id, request_number)
);

CREATE INDEX idx_citizen_requests_geom ON citizen_requests USING GIST (geom_point);
CREATE INDEX idx_citizen_requests_status ON citizen_requests (organization_id, status);

-- Citizen request media (photos of issues)
CREATE TABLE citizen_request_media (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citizen_request_id  UUID NOT NULL REFERENCES citizen_requests(id) ON DELETE CASCADE,
    file_path           VARCHAR(500) NOT NULL,
    mime_type           VARCHAR(100),
    caption             TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Audit Trail

```sql
-- Comprehensive audit log for regulatory compliance
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    old_values      JSONB,
    new_values      JSONB,
    changed_fields  TEXT[],
    user_id         UUID REFERENCES users(id),
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_table_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_log_org_date ON audit_log (organization_id, created_at DESC);

-- Partitioned by month for performance
-- ALTER TABLE audit_log PARTITION BY RANGE (created_at);
```

### GIS Data Exchange

```sql
-- External GIS data source configuration
CREATE TABLE gis_data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    source_type     VARCHAR(30) NOT NULL CHECK (source_type IN ('ogc_api_features', 'wfs', 'geojson_url', 'shapefile', 'geopackage', 'csv', 'arcgis_rest')),
    url             VARCHAR(500),
    auth_config     TEXT, -- Encrypted connection credentials
    sync_schedule   VARCHAR(100), -- Cron expression
    last_sync_at    TIMESTAMPTZ,
    last_sync_status VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Data import batches for tracking lineage
CREATE TABLE import_batches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    gis_data_source_id UUID REFERENCES gis_data_sources(id),
    import_type     VARCHAR(30) NOT NULL CHECK (import_type IN ('initial', 'incremental', 'manual_upload', 'ai_migration')),
    file_name       VARCHAR(500),
    record_count    INTEGER,
    success_count   INTEGER,
    error_count     INTEGER,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'cancelled')),
    error_log       TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Key Views for Querying

```sql
-- Unified asset view with current condition and financials
CREATE OR REPLACE VIEW v_asset_summary AS
SELECT
    a.id,
    a.organization_id,
    a.asset_number,
    a.name,
    a.status,
    ac.name AS asset_class_name,
    ac.asset_category,
    d.name AS department_name,
    a.installation_date,
    a.current_condition_score,
    a.current_condition_rating,
    a.current_condition_date,
    af.acquisition_cost,
    af.replacement_cost,
    af.net_book_value,
    af.accumulated_depreciation,
    a.criticality_rating,
    a.risk_score,
    a.geom_point,
    a.geom_line,
    a.geom_polygon,
    COALESCE(a.geom_point, ST_Centroid(a.geom_line), ST_Centroid(a.geom_polygon)) AS centroid
FROM assets a
JOIN asset_classes ac ON a.asset_class_id = ac.id
LEFT JOIN departments d ON a.department_id = d.id
LEFT JOIN asset_financials af ON a.id = af.asset_id;

-- GASB 34 reporting view: condition assessment summary by asset class
CREATE OR REPLACE VIEW v_gasb34_condition_summary AS
SELECT
    o.name AS organization_name,
    ac.name AS asset_class_name,
    g.assessment_year,
    g.target_condition_level,
    g.actual_condition_level,
    g.condition_met,
    g.estimated_preservation_cost,
    g.actual_preservation_cost,
    g.total_assets_assessed,
    g.total_replacement_value,
    CASE
        WHEN g.actual_condition_level >= g.target_condition_level THEN 'Compliant'
        ELSE 'Below Target'
    END AS compliance_status
FROM gasb34_condition_assessments g
JOIN organizations o ON g.organization_id = o.id
JOIN asset_classes ac ON g.asset_class_id = ac.id
ORDER BY g.assessment_year DESC, ac.name;

-- Work order backlog and performance view
CREATE OR REPLACE VIEW v_work_order_metrics AS
SELECT
    wo.organization_id,
    wo.status,
    wo.work_type,
    wo.priority,
    COUNT(*) AS order_count,
    SUM(COALESCE(woc.total_cost, 0)) AS total_cost,
    AVG(EXTRACT(EPOCH FROM (wo.actual_end - wo.actual_start)) / 3600) AS avg_completion_hours,
    AVG(EXTRACT(EPOCH FROM (wo.actual_start - wo.requested_date::TIMESTAMPTZ)) / 86400) AS avg_response_days
FROM work_orders wo
LEFT JOIN LATERAL (
    SELECT SUM(total_cost) AS total_cost
    FROM work_order_costs
    WHERE work_order_id = wo.id
) woc ON true
GROUP BY wo.organization_id, wo.status, wo.work_type, wo.priority;
```

---

## Pros and Cons

### Pros

1. **Data Integrity**: Full referential integrity via foreign keys ensures no orphaned records. Every inspection must reference a valid asset; every work order cost must reference a valid work order.

2. **GASB 34 Compliance**: The normalized structure directly maps to GASB 34 reporting requirements. Financial records, condition assessments, and depreciation calculations are cleanly separated and auditable.

3. **Query Predictability**: Standard SQL queries with JOINs are well-understood by every developer and DBA. Query plans are predictable and optimizable with standard indexes.

4. **PostGIS Integration**: Native spatial data types with GIST indexes provide high-performance spatial queries (proximity searches, intersection analysis, network-level views) without leaving the database.

5. **Mature Ecosystem**: PostgreSQL + PostGIS is the most mature open-source spatial database stack. Extensive tooling exists for backup, replication, monitoring, migration, and administration.

6. **OGC Compliance**: PostGIS natively supports OGC standards (WKT, WKB, GeoJSON, ST_* functions), making it straightforward to implement OGC API Features endpoints.

7. **Regulatory Auditability**: The audit_log table with old/new value tracking satisfies ISO 55001 and GASB 34 requirements for change traceability and record reproducibility.

8. **Multi-Tenant Ready**: The `organization_id` column pattern enables multi-municipality deployments with row-level security policies.

### Cons

1. **Schema Rigidity**: Adding new asset types requires DDL changes (new child tables). Each municipality may track different attributes for the same asset class, leading to schema proliferation or excessive use of the EAV pattern for custom attributes.

2. **EAV Performance Penalty**: The `asset_attributes` EAV table is necessary for class-specific attributes but is notoriously slow for queries that filter or aggregate on extended attributes. Pivoting EAV data into usable rows requires complex queries.

3. **Complex Queries for Hierarchies**: While LTREE helps, querying across the full asset hierarchy (network > segment > component) with condition rollups requires recursive CTEs or materialized path logic that can become expensive.

4. **Migration Burden**: The high number of tables (40+) with strict foreign key constraints makes data migration from legacy systems complex. Import processes must respect the correct insertion order.

5. **Condition Framework Flexibility**: The normalized distress/scoring tables add joins to every condition query. A PCI calculation requiring distress densities, deduct values, and corrected deduct values involves 4-5 table joins per asset.

6. **Scaling Limitations**: A single PostgreSQL instance handles millions of assets well, but very large deployments (multiple state DOTs pooled) may require read replicas or partitioning strategies.

7. **No Native Time-Series Optimization**: Condition history and sensor data stored in regular tables lack the compression and continuous aggregate features of purpose-built time-series databases.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with PostGIS 3.4+, pg_trgm, ltree extensions |
| **Spatial Index** | GIST indexes on all geometry columns; SP-GIST for point-only columns |
| **Partitioning** | Range-partition `audit_log` and `inspections` by month/year; hash-partition `assets` by `organization_id` for very large multi-tenant deployments |
| **Connection Pooling** | PgBouncer in transaction mode for high-concurrency mobile inspection traffic |
| **Replication** | Streaming replication with one or more read replicas for reporting queries |
| **Backup** | pg_basebackup + WAL archiving for point-in-time recovery; pgBackRest for large databases |
| **Full-Text Search** | pg_trgm + GIN indexes for asset name/description search; consider adding PostgreSQL full-text search vectors |
| **API Layer** | PostgREST or custom application layer exposing OGC API Features-compliant endpoints |
| **GIS Tooling** | QGIS for advanced spatial analysis; GeoServer or pygeoapi for OGC WFS/API Features serving |
| **Migration Tool** | Flyway or Liquibase for schema versioning and multi-tenant migrations |

---

## Migration and Scaling Considerations

### Initial Data Migration

1. **Legacy Spreadsheet Import**: Create a staging schema (`staging.*`) with loose typing. Use AI-assisted field mapping to load CSV/Excel data into staging tables, then validate and transform into the production schema. The `import_batches` table tracks every import for lineage.

2. **GIS Data Migration**: Load shapefiles and GeoPackages into PostGIS via `ogr2ogr` or `shp2pgsql`. Match imported geometries to asset records using spatial joins (ST_DWithin, ST_Intersects).

3. **Foreign Key Ordering**: Load reference data first (organizations, departments, asset_classes, condition_frameworks), then assets, then dependent records (inspections, work orders, financials).

### Scaling Strategy

1. **0-100K assets**: Single PostgreSQL instance is sufficient. Standard B-tree and GIST indexes handle all query patterns.

2. **100K-1M assets**: Add read replicas for reporting and map rendering. Partition audit_log and inspections by date range. Consider materialized views for dashboard aggregations refreshed on a schedule.

3. **1M-10M assets**: Hash-partition the assets table by organization_id. Use PgBouncer for connection pooling. Move media files to object storage (S3/MinIO) and store only references in the database.

4. **10M+ assets**: Consider Citus for distributed PostgreSQL if a single-node becomes a bottleneck. Alternatively, deploy per-organization database instances with a routing layer.

### Schema Evolution

- Use Flyway or Liquibase for version-controlled migrations
- New asset types: add child tables with a migration script; no downtime required
- New attributes: add columns to child tables or rows to `asset_class_attributes`; backward-compatible
- Condition framework changes: new rows in `condition_frameworks` and `distress_types`; existing inspections remain linked to the original framework version

### Performance Optimization

- Materialized views for dashboard aggregations (refresh every 15 minutes)
- Partial indexes on `status = 'active'` for the assets table (most queries filter active assets)
- Expression indexes on JSONB columns if any are introduced later
- BRIN indexes on timestamp columns in large partitioned tables (audit_log, inspections)
- Connection pooling with PgBouncer to handle mobile inspection app connection spikes
