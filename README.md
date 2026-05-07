# Infrastructure Asset Registry

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native platform for managing the full lifecycle of physical infrastructure assets -- from inventory and condition assessment to capital planning and compliance reporting -- without vendor lock-in to proprietary GIS platforms.

Infrastructure Asset Registry provides municipalities, transportation departments, and utilities with a structured, GIS-linked inventory of roads, bridges, pipes, culverts, signals, and other long-lived assets. It replaces reactive, complaint-driven maintenance with condition-based priority planning backed by data. The system records location, condition, maintenance history, and financial data for every asset, connecting inspection workflows to capital improvement programmes so agencies can make defensible investment decisions.

---

## Why Infrastructure Asset Registry?

- **Proprietary GIS lock-in is the norm.** Nearly every incumbent (Trimble Unity, OpenGov/Cartegraph, AssetWorks) requires Esri ArcGIS licences, significantly increasing total cost of ownership. No open-source alternative with comparable condition management depth exists today.
- **Enterprise platforms price out small agencies.** IBM Maximo and Infor EAM require specialist consultants and six-figure implementation budgets, putting modern asset management out of reach for smaller municipalities and school districts.
- **Incumbent mobile apps are unreliable.** OpenGov/Cartegraph's mobile app is reported as battery-draining and crash-prone; frequent UI changes create retraining burdens for field crews who need simple, stable tools.
- **Compliance reporting is manual and fragmented.** GASB 34 and grant-application reporting requires assembling data across multiple systems. No incumbent offers automated, data-driven compliance report generation out of the box.
- **AI capabilities are siloed or absent.** Only iFactory and IBM Maximo offer meaningful AI, but iFactory is a prediction overlay (not a full registry) and Maximo's AI is locked inside IBM's ecosystem. Transparent, explainable AI for condition assessment remains an open gap.

---

## Key Features

### Asset Inventory and Hierarchy

- Unique identification and classification for every asset with full attributes: type, location, dimensions, material, installation date, replacement cost, responsible department
- Hierarchical organisation from network to segment to component
- Support for both horizontal (roads, pipes, sewers) and vertical (buildings, facilities) asset types
- Import wizard with AI-assisted field mapping for migrating legacy spreadsheet, GIS, and paper records

### GIS Integration (Vendor-Neutral)

- Spatial data linking via open standards: GeoJSON, OGC API Features, PostGIS
- Map-based search, proximity analysis, and network-level views without Esri licence dependency
- Compatible with QGIS, OpenStreetMap, Leaflet, and OpenLayers
- Near real-time data synchronisation with external GIS sources

### Condition Inspection and Rating

- Mobile-friendly inspection recording with photo capture and offline support for remote field locations
- Standardised scoring frameworks: Pavement Condition Index (PCI/ASTM D6433), Bridge Health Index (NBI), CCTV pipe grading, Asset Condition Index (ACI)
- AI-assisted condition scoring from inspection photos using computer vision
- Multi-standard condition index support in a unified scoring model

### Capital Planning and Scenario Modelling

- Replacement cost forecasting by asset class and condition band
- Scenario modelling showing long-term funding impact of different investment levels, including cost of deferral
- AI-recommended CIP investment allocations based on deterioration models
- Predictive deterioration modelling with maintenance deferral cost analysis

### Work Order Management and Maintenance

- Work order creation, assignment, tracking, and completion
- Preventive maintenance scheduling with automated work order generation
- Natural language work order creation from field voice or text notes
- Job costing and cost tracking per asset

### Reporting and Compliance

- GASB 34 and ISO 55001-aligned reporting templates
- Automated grant-application report generation
- AI-synthesised plain-language reports for elected officials and non-technical stakeholders
- Natural language query interface for asset data retrieval and ad-hoc reporting

### Community and Stakeholder Access

- Citizen/constituent issue submission portal linked to asset records
- Community-facing transparency dashboards showing asset conditions and planned capital works
- Role-based access control for field crews, supervisors, administrators, and finance staff

---

## AI-Native Advantage

Unlike incumbents that bolt AI onto legacy platforms or offer it only as a proprietary overlay, Infrastructure Asset Registry integrates AI as a core capability. Computer vision automates condition scoring from inspection photos, replacing subjective manual ratings. Predictive deterioration models forecast asset failure likelihood based on condition trends, age, material, and environmental factors, giving agencies 30-60 days of advance warning. Natural language interfaces let non-technical staff query asset data and generate reports without BI skills. Automated data migration uses AI to map and transform legacy spreadsheet formats into structured registry schemas, dramatically reducing implementation time.

---

## Tech Stack & Deployment

- **GIS foundation:** PostGIS, GeoJSON, OGC API Features -- no proprietary GIS licence required
- **Frontend mapping:** Leaflet or OpenLayers for browser-based map views; QGIS integration for advanced spatial analysis
- **Mobile:** Offline-capable mobile inspection app for field crews in low-connectivity environments
- **Deployment:** Self-hosted, cloud, or hybrid
- **Integration:** REST API for connectivity to financial systems, ERP (SAP, Oracle, Munis, Tyler), and SCADA
- **Standards:** PCI (ASTM D6433), NBI bridge inspection methodology, ISO 55001 asset management framework, GASB 34 financial reporting

---

## Market Context

The global infrastructure asset management software market serves municipalities, transportation departments, and utilities managing assets worth billions in replacement value. Incumbent pricing follows annual SaaS subscriptions tiered by asset count or municipality population, with enterprise platforms like IBM Maximo and Infor EAM requiring substantial implementation services on top of licence fees. Primary buyers are public works directors, asset management coordinators, and municipal CIOs under increasing regulatory pressure from GASB 34 requirements and federal/state infrastructure funding programmes that mandate demonstrated asset management capability.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
