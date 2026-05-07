# Standards & API Reference

> Project: Infrastructure Asset Registry · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 55001:2024 — Asset Management: Asset Management System Requirements**
- URL: https://www.iso.org/standard/83054.html
- The primary international standard governing asset management systems. Specifies requirements for establishing, implementing, maintaining, and improving an asset management system. Requires a Strategic Asset Management Plan (SAMP) and Asset Management Plans (AMPs) per asset class. Applicable to any organisation managing tangible assets (infrastructure, property, equipment). Updated in 2024 from the 2014 edition.

**ISO 55000:2014 — Asset Management: Overview, Principles and Terminology**
- URL: https://committee.iso.org/home/tc251
- The foundational vocabulary and principles standard in the ISO 55000 family. Defines what constitutes an asset, an asset management system, and the value assets deliver. Required reading for understanding the ISO 55001 requirements context.

**ISO 19156:2023 — Geographic Information: Observations, Measurements and Samples**
- URL: https://www.iso.org/standard/82463.html
- Defines a conceptual schema for observations and measurement acts, including samples taken from assets. Directly relevant to condition inspection recording and sensor-based asset monitoring. Jointly published by ISO/TC 211 and the Open Geospatial Consortium (OGC) as OGC Abstract Specification Topic 20.

**ISO 16739-1:2024 — Industry Foundation Classes (IFC) for Data Sharing in the Construction and Facility Management Industries**
- URL: https://www.iso.org/standard/84123.html
- Open, vendor-neutral standard for Building Information Model (BIM) data exchange. IFC 4.3 (the basis for the 2024 edition) introduced comprehensive infrastructure asset definitions (roads, bridges, tunnels, pipelines, rail). Critical for digital twin integration and data exchange with design/construction systems.

### W3C & IETF Standards

**RFC 7946 — The GeoJSON Format**
- URL: https://datatracker.ietf.org/doc/html/rfc7946
- Defines GeoJSON, the de facto open standard for encoding geographic data structures in JSON. The recommended format for expressing asset locations, geometries (point, line, polygon), and spatial properties in web APIs. Widely supported by all GIS platforms and spatial databases.

**RFC 9110 — HTTP Semantics (supersedes RFC 7231)**
- URL: https://datatracker.ietf.org/doc/html/rfc9110
- Defines the semantics of the HTTP protocol underpinning REST APIs. Relevant to designing the asset registry's REST API: status codes, content negotiation, caching headers, and method semantics (GET, POST, PUT, PATCH, DELETE).

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header and linking relations used in hypermedia APIs. Relevant for OGC API Features compliance and pagination in asset collection responses.

**W3C PROV-O — The PROV Ontology**
- URL: https://www.w3.org/TR/prov-o/
- W3C recommendation for expressing provenance information. Relevant to audit trails: tracking who inspected an asset, when, and what changes were made to records — a compliance requirement under ISO 55001 and GASB 34.

### Data Model & API Specifications

**OGC API — Features (OGC API Features, Parts 1–4)**
- URL: https://www.ogc.org/standards/ogcapi-features/ and https://github.com/opengeospatial/ogcapi-features
- The modern OGC standard for querying and managing geospatial features via REST APIs using OpenAPI, JSON, and HTML. Directly applicable as the spatial query interface for an infrastructure asset registry. Part 1 (Core) covers feature retrieval; Part 4 covers CRUD operations. All responses are GeoJSON feature collections. Royalty-free open standard.

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://swagger.io/specification/
- The industry-standard format for describing RESTful APIs. All new asset registry API endpoints should be documented using OAS 3.1. The OGC API standards family uses OpenAPI as its API description language, making OAS 3.1 the natural choice for the asset registry API definition.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- Standard for validating and documenting JSON data structures. Used to define and validate the asset record schema, inspection record schema, and work order schema. Integrates directly with OpenAPI 3.1.

**GML (Geography Markup Language) — OGC/ISO 19136**
- URL: https://www.ogc.org/standards/gml/
- XML-based standard for encoding geographic features. Required for interoperability with legacy GIS systems and government data exchanges that predate GeoJSON adoption. The asset registry should support GML import/export for compatibility with older municipal GIS systems.

**GTFS (General Transit Feed Specification)**
- URL: https://gtfs.org/
- Open standard for public transport data. Relevant for municipalities managing transit infrastructure assets (stops, shelters, vehicles) alongside roads and utilities.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URL: https://oauth.net/2/ and https://openid.net/connect/
- Industry-standard protocols for secure API authentication and user identity. An infrastructure asset registry serving multiple municipalities and departments requires role-based access control; OAuth 2.0 + OIDC is the recommended approach for federated identity management and API token issuance.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Defines the most critical security risks in APIs. Directly applicable to the asset registry REST API design: broken object-level authorisation, excessive data exposure, lack of rate limiting, and injection attacks are all risks in an asset data platform.

**NIST SP 800-53 — Security and Privacy Controls for Information Systems (Rev. 5)**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- US federal standard for information security controls. Relevant for asset registries deployed by US government agencies; many state and local governments require NIST 800-53 alignment for procurement.

**FIPS 140-3 — Security Requirements for Cryptographic Modules**
- URL: https://csrc.nist.gov/projects/cryptographic-module-validation-program
- Federal encryption standard. Required for US government data storage and transmission. An asset registry storing infrastructure location and condition data (potential security-sensitive information) may require FIPS 140-3 validated encryption in federal deployments.

### Condition Rating & Inspection Standards

**ASTM D6433-20 — Standard Practice for Roads and Parking Lots Pavement Condition Index Surveys**
- URL: https://store.astm.org/d6433-20.html
- Defines the Pavement Condition Index (PCI) methodology: a 0–100 numerical index for road and pavement condition. The de facto standard for road asset condition rating in the US and internationally. An asset registry should implement PCI as a native condition scoring schema for road assets.

**FHWA National Bridge Inspection Standards (NBIS) and National Bridge Inventory (NBI)**
- URL: https://www.fhwa.dot.gov/bridge/nbis.cfm
- US federal standards for bridge inspection frequency, inspector qualifications, and condition rating. The NBI condition rating system (0–9 scale for structural elements) is the required schema for bridge assets in the US. An asset registry must support NBI element-level condition recording.

**AASHTO CoRe Element Standards (Commonly Recognised Bridge Elements)**
- URL: http://www.pdth.com/images/coreelem.pdf
- FHWA and AASHTO-endorsed standard for bridge element-level condition data. More granular than NBI ratings; supports bridge health index calculations used by modern bridge management systems.

### Regulatory & Compliance Frameworks

**GASB Statement No. 34 (GASB 34) — Basic Financial Statements for State and Local Governments**
- URL: https://www.gasb.org/page/PageContent?pageId=%2Fstandards-and-guidance%2Fpronouncements%2Fsummary-statement-no-34.html
- US accounting standard requiring all major infrastructure assets acquired after June 15, 1980 to be capitalised and reported. Mandates either depreciation accounting or the modified approach (condition-based maintenance cost reporting). A compliant asset registry must support both approaches and generate GASB 34-aligned financial reports.

**MCP Server Specifications**
- The Model Context Protocol (MCP) is relevant for exposing asset registry data to AI agents and LLM-based workflows. Publishing an MCP server for the asset registry would allow AI assistants to query asset condition, work history, and capital plans through natural language without custom integration.
- MCP documentation: https://modelcontextprotocol.io/

---

## Similar Products — Developer Documentation & APIs

### IBM Maximo (Manage REST API)

- **Description:** Enterprise asset management platform with decades of adoption in utilities, transit, government, and manufacturing. Manages full asset lifecycle including work orders, procurement, and condition-based maintenance.
- **API Documentation:** https://ibm-maximo-dev.github.io/maximo-restapi-documentation/
- **IBM API Hub:** https://developer.ibm.com/apis/catalog/maximo--maximo-manage-rest-api/
- **SDKs/Libraries:** No official client SDK; REST/JSON API usable from any HTTP client. Community Python clients exist.
- **Developer Guide:** https://www.ibm.com/docs/en/mio?topic=api-data-rest-developers-guide
- **Standards:** REST/JSON; OSLC (older API version); BULK operations via POST with x-method-override header
- **Authentication:** API key / session token; OAuth 2.0 in newer MAS cloud deployments

### OpenGov (Cartegraph EAM API)

- **Description:** Cloud asset management platform for local government, built on Cartegraph. Manages infrastructure assets, work orders, capital planning, and constituent requests.
- **API Documentation:** https://developer.opengov.com/docs/overview
- **API Portal:** https://api.docs.opengov.com/
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** https://developer.opengov.com/docs/quickstart
- **Standards:** REST/JSON; OpenAPI described
- **Authentication:** API Key; OAuth 2.0 for user authentication

### Trimble Unity Maintain (formerly Cityworks) API

- **Description:** GIS-centric asset management platform built on Esri ArcGIS. Manages public infrastructure assets, work orders, permits, and licensing for municipalities and utilities.
- **API Documentation:** https://developer.trimble.com/docs/unity-maintain-permit/
- **App Xchange Connector:** https://appxchange.trimble.com/connectors/trimble-unity-maintain
- **SDKs/Libraries:** FME Platform connectors for ETL workflows
- **Developer Guide:** https://support.safe.com/hc/en-us/articles/25407469587981-Tutorial-Getting-Started-with-Trimble-Unity-Maintain-Cityworks
- **Standards:** REST API with OpenAPI specification; ArcGIS REST API for GIS layer access
- **Authentication:** OAuth 2.0 / API Key

### Esri ArcGIS REST API (Feature Services)

- **Description:** The dominant GIS platform. Asset Management applications built on ArcGIS store assets as geodatabase feature classes; the ArcGIS Feature Service REST API exposes these for CRUD operations, spatial queries, and attachments.
- **API Documentation:** https://developers.arcgis.com/rest/services-reference/enterprise/fs-assets/
- **SDKs/Libraries:** ArcGIS API for JavaScript, ArcGIS API for Python (arcgis package), ArcGIS SDK for .NET, Java, Swift, Kotlin
- **Developer Guide:** https://developers.arcgis.com/documentation/
- **Standards:** OGC WFS, OGC API Features (partially); proprietary JSON feature format; REST
- **Authentication:** API Key, OAuth 2.0 (ArcGIS Identity), Named User tokens

### NEXGEN Asset Management API

- **Description:** Public works and utility asset management platform serving government agencies. Manages horizontal and vertical assets with GIS integration, condition assessment, and capital planning.
- **API Documentation:** https://docs.nexgenbusiness.net/api/
- **SDKs/Libraries:** Pre-written web APIs for common integration scenarios (GIS, ERP, SCADA, financial systems)
- **Developer Guide:** Not publicly documented beyond integration overview
- **Standards:** REST/JSON; GIS data exchange via Esri geodatabase import
- **Authentication:** Not publicly documented

### Facilio REST API

- **Description:** IoT-powered connected CMMS for building and infrastructure asset management. API-first architecture with vendor-agnostic sensor integration.
- **API Documentation:** Available via Facilio developer portal (requires registration)
- **SDKs/Libraries:** REST API usable from any HTTP client; vendor-agnostic IoT connectors
- **Developer Guide:** Contact Facilio for developer onboarding
- **Standards:** REST/JSON; OpenAPI described; webhook support
- **Authentication:** API Key; OAuth 2.0

### OGC API — Features Reference Implementation (pygeoapi)

- **Description:** Open-source Python server implementing OGC API Features. Provides a reference for how a standards-compliant geospatial feature API should behave. Useful as the spatial query interface for an open-source asset registry.
- **API Documentation:** https://pygeoapi.io/ and https://features.developer.ogc.org/
- **SDKs/Libraries:** Python (pygeoapi server), any OGC API Features-compatible client
- **Developer Guide:** https://dive.pygeoapi.io/
- **Standards:** OGC API Features Parts 1–3; GeoJSON; OpenAPI 3.0; GML
- **Authentication:** Configurable (API Key, OAuth 2.0)

---

## Notes

**Esri Dependency Risk:** The dominant commercial platforms (Trimble Unity, OpenGov, NEXGEN) rely on Esri ArcGIS, which carries significant licence costs. An open-source infrastructure asset registry should use open GIS standards (PostGIS, OGC API Features, GeoJSON, QGIS) to avoid this dependency and enable adoption by under-resourced municipalities.

**Condition Rating Standards Fragmentation:** There is no single universal condition rating schema across asset types. PCI covers roads, NBI covers bridges, and each asset class (pipes, signals, parks) has its own inspection methodology. The asset registry data model must be extensible to accommodate multiple condition schemas without hard-coding any single one.

**GASB 34 Modified Approach Tooling Gap:** While most platforms claim GASB 34 compliance, few provide a self-service report generator that walks finance staff through the modified approach computation. This is a clear unmet need.

**MCP Integration Opportunity:** No current platform in this space has published an MCP (Model Context Protocol) server, creating an early-mover opportunity to expose asset registry data to AI agents for natural language querying, automated report generation, and capital planning assistance.

**IFC 4.3 and Digital Twin Readiness:** IFC 4.3 now covers infrastructure assets (roads, bridges, pipelines). An open-source asset registry that can import and export IFC 4.3 data would bridge the gap between BIM/design systems (used during construction) and operational asset management (used during the maintenance lifecycle), addressing a documented data handoff problem in the industry.
