# Infrastructure Asset Registry — Feature & Functionality Survey

> Candidate #439 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| OpenGov Enterprise Asset Management (Cartegraph) | Cloud SaaS | Commercial / Subscription | https://opengov.com/products/asset-management/ |
| NEXGEN Asset Management | Cloud SaaS | Commercial / Subscription | https://www.nexgenam.com/ |
| Trimble Unity Maintain (Cityworks) | Cloud SaaS | Commercial / Subscription | https://assetlifecycle.trimble.com/en/products/software/unity-maintain |
| IBM Maximo Application Suite | On-premises / Cloud | Commercial / Subscription | https://www.ibm.com/products/maximo |
| Infor EAM (HxGN EAM) | On-premises / Cloud | Commercial / Subscription | https://www3.technologyevaluation.com/solutions/16945/infor-eam |
| AssetWorks EAM | Cloud SaaS | Commercial / Subscription | https://www.assetworks.com/eam/ |
| FMX | Cloud SaaS | Commercial / Subscription | https://www.gofmx.com/infrastructure-asset-management-software/ |
| iFactory | Cloud SaaS / AI Overlay | Commercial / Subscription | https://ifactoryapp.com/industries/infrastructure-management/ |
| Facilio | Cloud SaaS | Commercial / Subscription | https://facilio.com/ |

---

## Feature Analysis by Solution

### OpenGov Enterprise Asset Management (Cartegraph)

**Core features**
- Asset inventory with detailed attributes (type, location, dimensions, material, installation date, replacement cost)
- GIS integration for map-based search, proximity analysis, and network-level views
- Preventive maintenance scheduling and automation
- Condition inspections with mobile field capture
- Work order planning and tracking
- Live KPI dashboards and reporting
- Constituent request management portal
- Workflow automation engine

**Differentiating features**
- Deep integration with ArcGIS (Esri partner) for GIS-centric asset management
- Built specifically for public sector / government workflows
- Capital Improvement Program (CIP) planning module linked to condition data

**UX patterns**
- Web and mobile clients; field crews use mobile app for inspections
- Interactive map view as the primary navigation surface
- Dashboard-first reporting for executives and managers

**Integration points**
- ArcGIS, Salesforce, Microsoft Office, Google Workspace
- REST API available for custom integrations
- CivicPlus SeeClickFix connector for constituent requests

**Known gaps**
- Mobile app reported as battery-draining and prone to instability
- Reporting module considered overly complex for non-technical users
- GIS integration described as less GIS-centric than purpose-built tools like Cityworks
- Feature request portal described as ineffective; updates lag user demand
- Documentation quality inconsistent; help search returns accurate results only ~50% of the time
- Frequent UI changes create retraining burden for field staff

**Licence / IP notes**
- Proprietary commercial SaaS. OpenGov acquired Cartegraph; brand now merged under OpenGov.

---

### NEXGEN Asset Management

**Core features**
- Asset inventory supporting both horizontal (roads, pipes) and vertical (buildings, facilities) assets
- Asset hierarchy organisation with import from Esri GIS geodatabase
- Asset Condition Index (ACI) automatically calculated from inspection data
- Automated preventive maintenance work orders triggered by schedules or run-time thresholds
- Job costing and cost tracking per asset
- Lifecycle funding forecasts for capital planning
- Multi-division transparency across public works departments

**Differentiating features**
- Single platform covering both horizontal and vertical asset types (uncommon in this space)
- Pre-built integrations with financial, ERP, CIS, SCADA, and GIS systems
- Government purchasing vehicle via Carahsoft

**UX patterns**
- Web-based interface with GIS map view
- Role-based access for different public works divisions
- Automated work order generation reduces manual scheduling overhead

**Integration points**
- Esri ArcGIS GIS database import
- Financial systems, ERP, CIS, SCADA, MS Exchange/Outlook
- Pre-written web APIs for common integration scenarios

**Known gaps**
- Limited public developer documentation for REST API
- Less widely reviewed than OpenGov/Cartegraph on independent review platforms
- Mobile field app capabilities not prominently documented

**Licence / IP notes**
- Proprietary commercial SaaS. Available via government procurement vehicles.

---

### Trimble Unity Maintain (formerly Cityworks)

**Core features**
- GIS-centric architecture built exclusively on Esri ArcGIS — assets managed as geodatabase features
- Service requests, work orders, and inspection tasks configurable per asset type
- Manages any feature in the geodatabase: water, wastewater, stormwater, streets, signs, signals, parks, facilities
- Z-value support for 3D asset representation in ArcGIS Scene Viewer and ArcGIS Pro
- Storeroom inventory management
- Operational insights and analytics dashboards
- Licensing and permitting module (Unity Permit)

**Differentiating features**
- Most deeply GIS-integrated platform in the market — the GIS IS the asset database
- Related-assets visualisation for assets hard to display on a flat map view
- End-to-end asset lifecycle suite (design, construction, maintenance, decommission) via Trimble product family

**UX patterns**
- Map-centric UI — users navigate to assets via GIS map
- Integrated mobile app (Trimble Unity Field) for field-to-office workflows
- Predictive analytics for what-if scenario planning

**Integration points**
- Esri ArcGIS platform (native)
- Trimble App Xchange connector marketplace
- REST API with OpenAPI specification; developer docs at developer.trimble.com
- FME Platform for ETL and data integration workflows

**Known gaps**
- Heavy dependency on Esri ArcGIS licences increases total cost of ownership
- Platform rebrand (Cityworks → Unity Maintain) has caused user uncertainty
- API documentation for Unity Maintain still maturing (was listed as "coming early 2026")
- Complexity may be overkill for smaller municipalities

**Licence / IP notes**
- Proprietary commercial SaaS. Built on Esri ArcGIS; Esri licence required. Trimble owns Cityworks and AgileAssets brands.

---

### IBM Maximo Application Suite

**Core features**
- End-to-end enterprise asset management from procurement through decommissioning
- Work order management with automated workflows and real-time tracking
- Preventive, condition-based, and predictive maintenance
- Asset performance management using AI (IBM watsonx), analytics, and IoT sensor data
- Materials management and procurement
- Safety and compliance case management
- IT asset management module
- Mobile field access via smart devices

**Differentiating features**
- IBM watsonx AI integration for conversational retrieval of asset data and generative summaries
- IoT sensor ingestion and anomaly detection for predictive maintenance
- Mature, proven platform for large-scale critical infrastructure (utilities, transit, government)
- Broadest industry coverage: manufacturing, transportation, facilities, utilities, government

**UX patterns**
- Web-based with configurable dashboards per role
- Mobile-first field execution with offline capability
- AI-powered assistant for quick data retrieval

**Integration points**
- REST API (documented at ibm-maximo-dev.github.io and IBM API Hub)
- SAP, Oracle ERP integration
- IoT platform connectors
- ArcGIS GIS integration

**Known gaps**
- High implementation cost and complexity; requires specialist consultants
- Total cost of ownership (licence + implementation) prohibitive for smaller agencies
- On-premises deployments require significant IT infrastructure
- UI can feel dated compared to newer cloud-native competitors

**Licence / IP notes**
- Proprietary commercial software (IBM). On-premises and cloud SaaS available. Well-established IP with decades of development.

---

### Infor EAM (HxGN EAM)

**Core features**
- Asset hierarchy management tracking costs throughout the full lifecycle
- Work management covering preventive, corrective, and predictive maintenance
- Materials management and procurement
- Inspection management with compliance tracking
- Project management for capital works
- Budget management and cost reporting
- Linear asset management for roads, pipelines, sewers
- IoT and predictive analytics for condition-based maintenance
- Governance, risk, and compliance (GRC) module

**Differentiating features**
- Industry-specific configurations for process manufacturing, public sector, life sciences, fleet, and facilities
- Strong linear asset support (roads, pipelines) alongside point assets
- Fully web-architected with cloud and on-premises deployment flexibility

**UX patterns**
- Web-based interface with industry-specific workflow configurations
- Advanced Reporting module for customised analytics
- Role-based access with department-level views

**Integration points**
- SAP, Oracle ERP connectors
- IoT platform integration
- REST/JSON API
- GIS integration capabilities

**Known gaps**
- Complex configuration; significant implementation effort required
- Less commonly adopted by small-to-medium municipalities
- GIS integration less prominent than Trimble or OpenGov offerings

**Licence / IP notes**
- Proprietary commercial software (Infor / HxGN). Available cloud and on-premises.

---

### AssetWorks EAM

**Core features**
- Asset tracking across unlimited asset types: spatial, linear, boundary, and fleet
- Full lifecycle management from acquisition to decommissioning
- Preventive maintenance scheduling and work order management
- Asset depreciation tracking and usage history
- Fuel and fleet management integration
- GIS mapping with near real-time Esri data synchronisation
- Inventory and parts management
- Government-grade data security and uptime standards

**Differentiating features**
- 40+ years of government/public sector focus
- Combined fleet and infrastructure asset management in one platform
- Competitively solicited government purchasing vehicle available

**UX patterns**
- Web-based with GIS map view
- Reporting dashboards for performance monitoring
- Mobile access for field operations

**Integration points**
- Esri ArcGIS (near real-time sync)
- Government ERP and financial systems
- REST API for custom integrations

**Known gaps**
- Primarily US government market focus; limited international case studies
- Fleet-centric heritage may mean infrastructure asset depth is lower than pure-play competitors
- Limited independent review coverage compared to OpenGov/Maximo

**Licence / IP notes**
- Proprietary commercial SaaS. Government procurement vehicles available.

---

### FMX

**Core features**
- Infrastructure asset registry with lifecycle tracking
- Preventive maintenance scheduling and automated work order generation
- Inspection scheduling and repair tracking
- Inventory management
- GIS mapping via Esri ArcGIS Online integration
- Community request portal for citizen issue submission
- Forecasting dashboards for asset replacement cost planning
- Capital improvement project management

**Differentiating features**
- Designed for accessibility by smaller municipalities and school districts
- Shorter implementation time and lower cost than enterprise platforms
- Citizen-facing request portal integrated into the asset management workflow

**UX patterns**
- Clean, modern cloud UI designed for non-technical staff
- Centralised dashboard combining assets, maintenance, and service requests
- Mobile access for field teams

**Integration points**
- Esri ArcGIS Online for GIS mapping
- API available for integration with other systems

**Known gaps**
- Less depth in advanced condition modelling and deterioration forecasting
- Limited analytics compared to Maximo or Infor
- May lack compliance reporting depth for larger agencies (GASB 34, grant requirements)

**Licence / IP notes**
- Proprietary commercial SaaS. Positioned as a mid-market alternative.

---

### iFactory

**Core features**
- AI-powered predictive maintenance for roads, bridges, and utilities
- Interferometric Synthetic Aperture Radar (InSAR) for millimetre-level subsidence monitoring
- Ground Penetrating Radar analysis for subsurface void and moisture detection
- Vibration and strain sensor analysis for bridge deck fatigue and pier stability
- Pavement surface roughness and subsurface moisture analysis
- Smart culvert level and debris accumulation monitoring
- Unified high-density dashboard consolidating all sensor feeds
- 30–60 day advance failure prediction capability

**Differentiating features**
- AI vision on patrol vehicles for continuous road condition assessment
- Micro-fracture detection 60 days before visual cracks appear
- 94% accuracy pothole formation prediction based on weather and traffic load
- Network-to-bolt drill-down visualisation from national view to individual component

**UX patterns**
- Command centre dashboard for infrastructure network monitoring
- Drill-down from macro network view to individual asset/component
- Alert-driven workflow: AI flags priority assets for engineer review

**Integration points**
- Sensor hardware integrations (vibration sensors, InSAR, GPR, smart culverts)
- Feeds into existing asset management platforms as a data/intelligence layer
- API for integration with municipal asset registries

**Known gaps**
- Primarily an intelligence/prediction layer, not a full asset registry
- Hardware sensor dependency increases deployment cost
- Limited direct GASB 34 / compliance reporting capability
- Not a standalone replacement for full EAM platforms

**Licence / IP notes**
- Proprietary commercial SaaS. AI/sensor IP likely protected.

---

### Facilio

**Core features**
- Connected CMMS integrating maintenance, vendors, tenants, compliance, and financial workflows
- IoT-powered real-time asset tracking and health monitoring
- Preventive maintenance scheduling with mobile execution
- Automated work order management with SLA tracking
- Condition-based and AI-powered predictive maintenance
- Energy management with real-time remote command and control
- Multi-site portfolio management
- Vendor-agnostic sensor and system integration

**Differentiating features**
- API-first architecture with robust REST API for ERP, CRM, ticketing, accounting integration
- IoT-native design for smart building/infrastructure sensor ingestion
- Low-code automation tools for SLAs, escalations, and approvals
- Strong facilities management angle (commercial real estate, campuses)

**UX patterns**
- Dashboard-driven multi-site portfolio view
- Mobile app for field technicians
- Low-code workflow builder for non-technical configuration

**Integration points**
- REST API (documented)
- Salesforce, Oracle ERP, Microsoft Teams, IBM Maximo, QuickBooks
- IoT devices via API and data connectors
- Vendor-agnostic sensor integration

**Known gaps**
- Stronger in commercial facilities than in public infrastructure (roads, bridges, pipes)
- Less GIS-centric than Trimble or OpenGov
- Limited public-sector compliance reporting (GASB 34, grant management)

**Licence / IP notes**
- Proprietary commercial SaaS.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Asset inventory with unique identification, attributes, and hierarchical organisation
- GIS integration for spatial asset location and map-based navigation
- Work order management: creation, assignment, tracking, and completion
- Preventive maintenance scheduling with automated work order generation
- Condition inspection recording with mobile field capture
- Basic reporting and dashboard for asset performance KPIs
- Role-based access control for different user types (field, supervisor, administrator, finance)
- Import tools for migrating data from spreadsheets or legacy systems

### Differentiating Features
- AI-powered predictive maintenance and failure forecasting (iFactory, IBM Maximo, Facilio)
- Deep GIS-centric architecture where GIS is the native asset database (Trimble Unity)
- Digital twin and sensor ingestion for real-time condition monitoring (iFactory, Facilio)
- Scenario-based capital planning showing funding impact of deferred maintenance
- Citizen/constituent request portal integrated into the asset workflow (FMX)
- Linear asset management for roads, pipelines, and sewers as first-class entities (Infor, NEXGEN)
- End-to-end lifecycle suite from design and construction through to decommissioning (Trimble)
- Government procurement vehicles and GASB 34 compliance reporting (OpenGov, AssetWorks)

### Underserved Areas / Opportunities
- Open-source, vendor-neutral asset registry with no Esri licence dependency
- Transparent, explainable AI for condition predictions that field inspectors can trust and challenge
- Natural language interface for querying asset data and generating reports without BI skills
- Automated GASB 34 and grant-application report generation from underlying asset data
- Low-cost mobile inspection app usable by small municipalities without enterprise licences
- Data import and cleansing assistance for migrating legacy spreadsheet and paper records
- Community-facing transparency dashboards showing asset conditions and planned capital works
- Multi-standard condition scoring (PCI, NBI, ACI) in a single unified registry

### AI-Augmentation Candidates
- Automated condition scoring from inspection photos (computer vision replacing manual rating)
- Natural language work order creation from field voice or text notes
- Anomaly detection on sensor streams to flag assets for priority inspection
- Generative report drafting: AI synthesises asset data into plain-language reports for elected officials
- Predictive capital planning: AI recommends CIP investment allocations based on deterioration models
- Automated data migration: AI maps and transforms legacy spreadsheet formats into structured registry schema

---

## Legal & IP Summary

All tools surveyed are proprietary commercial software with standard SaaS licence agreements. No open-source infrastructure asset registry with GIS integration and condition management depth comparable to these commercial platforms was identified during research. The dominant GIS platform (Esri ArcGIS) is proprietary; building a competitive open-source tool would require reliance on open GIS standards (OGC API, GeoJSON, QGIS) and open-source GIS libraries (PostGIS, Leaflet, OpenLayers). Pavement Condition Index (PCI) methodology is standardised by ASTM (D6433) and is freely implementable. Bridge inspection methodologies (NBI, CoRe elements) are US federal standards, freely available. No patent concerns were identified for the core data modelling, inspection, or condition-rating functions. AI/sensor IP held by iFactory is likely protected but the underlying detection approaches use published research methods.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Structured asset inventory: unique IDs, classification hierarchy, attributes (type, location, material, install date, replacement cost)
- GIS integration via open standards (GeoJSON, OGC API Features, PostGIS) — no Esri dependency
- Mobile-friendly condition inspection recording with photo capture and offline support
- Work order management: create, assign, track, close
- Basic capital planning dashboard: replacement cost forecasting by asset class and condition band
- GASB 34 and ISO 55001-aligned reporting templates

**Should-have (v1.1)**
- AI-assisted condition scoring from inspection photos using computer vision
- Predictive deterioration modelling with scenario planning (maintenance deferral cost modelling)
- Natural language query interface for asset data retrieval and report generation
- Citizen/constituent issue submission portal linked to asset records
- Import wizard with AI-assisted field mapping for legacy spreadsheet and GIS data migration
- Multi-agency / multi-department access with granular role-based permissions

**Nice-to-have (backlog)**
- IoT sensor stream ingestion for real-time condition monitoring
- Digital twin integration for complex assets (bridges, treatment plants)
- Automated grant application report generation
- Integration connectors for common ERP and financial systems (SAP, Oracle, Munis, Tyler)
- Multi-standard condition index support: PCI, NBI, ACI in a unified scoring model
- Open API marketplace for third-party extensions
