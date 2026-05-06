# Standards & API Reference

> Project: Precision Agriculture Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 11783 (ISOBUS)**
- **URL:** https://www.iso.org/standard/57556.html
- Serial control and communications data network for tractors and agricultural machinery. A 14-part standard based on SAE J1939/CAN bus that enables plug-and-play interoperability between implements (sprayers, planters) and vehicles (tractors) from different manufacturers. The global ISOBUS component market reached ~US$2.1 billion in 2024. Essential for any platform that reads or writes variable-rate application data directly to/from field machinery.

**ISO/TC 347 — Data-Driven Agrifood Systems**
- **URL:** https://www.iso.org/committee/9983782.html
- Technical committee established October 2023 to create standards supporting interoperability in data-driven agrifood systems. Focuses on uniform terminology, semantic resources (controlled vocabularies and ontologies), and "agrisemantics" infrastructure. Relevant to data model design and API vocabulary alignment for any precision agriculture platform targeting international markets.

**ISO 11784 / ISO 11785 — RFID of Animals**
- **URL:** https://en.wikipedia.org/wiki/ISO_11784_and_ISO_11785
- Defines the code structure (ISO 11784) and technical concept (ISO 11785) for radio-frequency identification of livestock. Uses 134.2 kHz activation frequency with a 64-bit identification code including a 3-digit ISO 3166 country code and 12-digit national animal ID. Mandatory in the EU for sheep and goats since 2010. Relevant to livestock-integrated precision agriculture platforms and traceability modules.

**ISO 19157 — Geographic Information: Data Quality**
- **URL:** https://www.iso.org/standard/32575.html
- Defines principles for describing the quality of geographic data. Applicable to field boundary datasets, soil sampling records, and yield maps where data quality metadata must be communicated to downstream consumers.

**ISO 15143-3 — Earth-Moving Machinery and Mobile Road Construction Machinery: Worksite Data Exchange**
- Basis for CNH Industrial's FieldOps telematics API. Relevant when integrating telematics data from mixed fleets that include non-agricultural heavy equipment used on farms.

---

### OGC (Open Geospatial Consortium) Standards

**OGC SensorThings API**
- **URL:** https://ogcapi.ogc.org/sensorthings/
- Open, geospatial-enabled standard for interconnecting IoT devices, data, and applications over the Web. Follows REST principles, uses JSON encoding, and supports the OASIS OData protocol, with MQTT and CoAP extensions. Highly relevant for ingesting sensor data from soil probes, weather stations, and connected irrigation systems into a unified platform data store.

**OGC API — Features (successor to WFS)**
- **URL:** https://www.ogc.org/standards/wfs/
- The modern standard for serving geospatial feature data (field boundaries, management zones, soil sample locations) as JSON over HTTP. Replaces the older Web Feature Service (WFS) standard. Most contemporary precision agriculture platforms expose field boundary data through WFS-compatible or OGC API Features endpoints.

**OGC Web Map Service (WMS) / OGC API — Maps**
- **URL:** https://www.ogc.org/standards/wms/
- Standard HTTP interface for requesting geo-registered map images from geospatial databases. Used for serving vegetation index layers (NDVI, NDRE), yield maps, and satellite imagery tiles. Sentinel Hub and similar earth observation services expose data through WMS-compatible endpoints.

**OGC Geography Markup Language (GML)**
- Joint OGC/ISO standard (ISO 19136) defining an XML grammar for encoding and transporting geospatial content. Underlying encoding used by some ISOXML-adjacent tools for field data transfer. Less common in modern REST APIs but still required for legacy system compatibility.

---

### IETF Standards

**RFC 7946 — GeoJSON**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7946
- The IETF-standardised format for encoding geographic data structures (points, polygons, feature collections) as JSON. The de facto format for field boundary interchange, prescription map geometry, scouting observation coordinates, and sensor location data in modern precision agriculture APIs. Leaf Agriculture, Climate FieldView, and John Deere's Operations Center all use GeoJSON for spatial data transfer.

**RFC 9110 / HTTP Semantics**
- **URL:** https://datatracker.ietf.org/doc/html/rfc9110
- Foundation for all REST APIs in the ecosystem. Precision agriculture platforms should align to HTTP semantics for caching, conditional requests (ETags for large geospatial datasets), and content negotiation.

**MQTT v5.0 (OASIS Standard / IETF complementary)**
- **URL:** https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html
- Lightweight publish-subscribe messaging protocol widely used for IoT sensor telemetry in agriculture. TCP-based; suited for soil moisture sensors, weather stations, and automated irrigation controllers that need reliable delivery. Supported by the OGC SensorThings API as an extension.

**CoAP — RFC 7252 (Constrained Application Protocol)**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7252
- UDP-based IoT protocol for resource-constrained sensors in environments with limited connectivity (rural fields). Designed to translate easily to HTTP. Used alongside MQTT in smart farming sensor networks, particularly for battery-powered field sensors with intermittent connectivity.

---

### Data Model & API Specifications

**ADAPT Standard 1.0 (AgGateway)**
- **URL:** https://adaptstandard.org/ | Release announcement: https://aggateway.org/News/2024PressReleases/AgGatewayReleasesAdaptStandardVersion10.aspx
- Released June 2024. The world's first open standard for business-to-business transfer of agricultural production data (planting, application, harvest). Uses GeoParquet for vector data, GeoTIFF for raster data, and WKT for geometries embedded in JSON. Has no software dependencies — pure data standard. Supersedes the AgGateway ADAPT Framework toolkit (2015). Any precision agriculture platform that aims to be interoperable should implement ADAPT Standard 1.0 for prescription and as-applied data exchange.

**fiboa — Field Boundaries for Agriculture**
- **URL:** https://fiboa.org/ | GitHub: https://github.com/fiboa/specification
- Open specification for representing agricultural field boundary data in GeoJSON and GeoParquet. Defines a core schema with a small mandatory attribute set, plus an extension mechanism (GitHub-template-based) for additional attributes. Supported by a CLI validation tool (`pip install fiboa-cli`). Backed by Radiant Earth and cloud-native geospatial community. Emerging standard for interoperable field boundary datasets.

**OpenAPI Specification 3.1 / 3.2**
- **URL:** https://swagger.io/specification/ | https://www.openapis.org/
- Vendor-neutral specification for describing REST APIs. Version 3.2.0 (September 2025) adds structured tag nesting and native streaming media type support. Precision agriculture platform APIs should be described with OpenAPI to enable automatic SDK generation, interactive documentation, and ecosystem tooling compatibility.

**NGSI-LD (ETSI TS 103 790)**
- **URL:** https://ngsi-ld.org/ | ETSI white paper: https://www.etsi.org/images/files/ETSIWhitePapers/etsi_wp31_NGSI_API.pdf
- ETSI-standardised context information API for IoT, cyber-physical systems, and digital twins. Uses JSON-LD for semantic linking. The FIWARE open-source community maintains NGSI-LD data models for smart agriculture (AgriApp, AgriCrop, AgriParcel, AgriParcelOperation, AgriSoil entities). Relevant for platforms targeting European public-sector or smart-city adjacent deployments.

**ISO 28258 — Soil Data Model**
- **URL:** https://www.iso.org/standard/44595.html
- International standard defining a conceptual soil data model. Relevant to any platform that stores, exchanges, or analyses soil sampling data as part of variable-rate prescription workflows.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) + OpenID Connect 1.0**
- **URL:** https://datatracker.ietf.org/doc/html/rfc6749 | https://openid.net/connect/
- The standard authentication and authorisation framework used by John Deere Operations Center API, Climate FieldView API, Trimble Ag API, and Leaf Agriculture API. Precision agriculture platforms must implement OAuth 2.0 to consume third-party data feeds and to allow third-party applications to access their own APIs safely.

**GDPR (EU 2016/679) and ePrivacy**
- **URL:** https://gdpr-info.eu/
- EU data protection regulation requiring data minimisation, explicit consent, and the right to erasure. Farm data — including field boundaries, yield records, and machinery telemetry — is increasingly treated as personal or commercially sensitive data. The EU Farm Data Spaces initiative (2024) explicitly applies GDPR principles to agricultural data sharing. Platforms operating in EU markets must implement GDPR-compliant consent management, data access logs, and deletion workflows.

**OWASP API Security Top 10**
- **URL:** https://owasp.org/www-project-api-security/
- Authoritative checklist for API security risks. Particularly relevant for precision agriculture platforms exposing REST/GraphQL APIs with farm-level field data, machinery credentials, and prescription maps that represent significant commercial assets.

**NIST Cybersecurity Framework 2.0**
- **URL:** https://www.nist.gov/cyberframework
- US federal framework for managing cybersecurity risk. Relevant for precision agriculture platforms deployed by US agribusiness enterprises, co-operatives, or government-adjacent organisations where NIST alignment is expected in procurement.

---

### MCP Server Specifications

**Model Context Protocol (Anthropic MCP)**
- **URL:** https://modelcontextprotocol.io/
- Open protocol for exposing tool capabilities to AI agents. Leaf Agriculture launched an MCP server (2025/2026) that exposes its unified farm data API to any AI agent, enabling agents to read docs, call endpoints, and chain tools (field boundaries, operations, imagery, weather) without custom integration work. A precision agriculture platform with an MCP server would be immediately accessible to AI-powered agronomic advisory agents.

---

## Similar Products — Developer Documentation & APIs

### John Deere Operations Center API
- **Description:** Cloud platform storing telemetry and field operation data from John Deere equipment. Provides APIs for accessing field boundaries, machine data, prescriptions, and as-applied records. Supports cross-brand data via the Operations Center organisation model.
- **API Documentation:** https://developer.deere.com/
- **SDKs/Libraries:** Community Ruby gem: https://github.com/RealmFive/my_john_deere_api ; Leaf Agriculture abstraction layer: https://withleaf.io/providers/johndeere
- **Developer Guide:** https://developer.deere.com/precision/get-started
- **Standards:** REST/JSON; OAuth 2.0 for authentication; GeoJSON for spatial data
- **Authentication:** OAuth 2.0 (Authorization Code flow)

### Climate FieldView API (Bayer)
- **Description:** One of the largest digital agriculture platforms globally, covering 120+ million acres. API exposes field boundaries, planting prescriptions, imagery analytics, and harvest data. Backend-only calls required (no browser-side API access).
- **API Documentation:** https://dev.fieldview.com/technical-documentation/
- **SDKs/Libraries:** Leaf Agriculture abstraction: https://withleaf.io/providers/climatefieldview
- **Developer Guide:** https://dev.fieldview.com/
- **Standards:** REST/JSON; GeoJSON spatial data
- **Authentication:** OAuth 2.0; developer sandbox available; contact fieldview.developer@bayer.com

### Trimble Agriculture API
- **Description:** Exposes Trimble Agriculture Cloud capabilities including agronomic field data, guidance lines, boundaries, prescriptions, and machinery telematics. Separate Ag Data API and Ag Telematics API with access granted based on use-case approval.
- **API Documentation:** https://agdeveloper.trimble.com/api-docs
- **SDKs/Libraries:** https://developer.trimble.com/docs/ag/ag-api/
- **Developer Guide:** https://agdeveloper.trimble.com/documentation/trimble-ag-api-developer-guide-3-3-4-2-2-2-2-2-3-2-2-2-2/
- **Standards:** REST/JSON; OpenAPI-described endpoints; ADAPT framework compatible
- **Authentication:** OAuth 2.0; credential request form required

### CNH Industrial (Case IH / New Holland) FieldOps API
- **Description:** Unified platform for agronomic and machine data from CNH Industrial brands. FieldOps API vehicle telematics endpoints are based on the ISO 15143-3 specification. AFS Connect integration enables data sharing with third-party software providers.
- **API Documentation:** https://develop.cnh.com/api-guides/fieldops-api
- **SDKs/Libraries:** https://develop.cnh.com/
- **Developer Guide:** https://develop.cnh.com/api-guides
- **Standards:** REST/JSON; ISO 15143-3 for telematics; OAuth 2.0
- **Authentication:** OAuth 2.0

### Leaf Agriculture Unified Farm Data API
- **Description:** Aggregation layer that normalises data from John Deere, Climate FieldView, Trimble, CNH, Ag Leader, and 15+ other providers into a single REST API with consistent GeoJSON output. Offers field operations, boundaries, imagery, weather, and prescriptions through one integration. Launched an MCP server in 2025 for AI agent access.
- **API Documentation:** https://withleaf.io/for-developers/
- **SDKs/Libraries:** REST API; MCP server at https://withleaf.io/en/whats-new/leaf-mcp-launch/
- **Developer Guide:** https://withleaf.io/for-developers/tutorials
- **Standards:** REST/JSON; GeoJSON; OpenAPI described
- **Authentication:** API Key + OAuth 2.0 for provider pass-through

### Sentinel Hub / Copernicus Data Space Ecosystem
- **Description:** Multi-spectral satellite imagery service for precision agriculture. Provides access to Sentinel-2 (10–20 m resolution, 5-day revisit), Sentinel-1 SAR, and Landsat archives. Processing API returns NDVI, LAI, and custom spectral index calculations. Python SDK available.
- **API Documentation:** https://documentation.dataspace.copernicus.eu/APIs/SentinelHub.html
- **SDKs/Libraries:** sentinelsat (Python): https://github.com/sentinelsat/sentinelsat ; Sentinel Hub Python SDK
- **Developer Guide:** https://documentation.dataspace.copernicus.eu/notebook-samples/sentinelhub/introduction_to_SH_APIs.html
- **Standards:** REST/JSON; OGC WMS/WCS; OpenAPI described
- **Authentication:** OAuth 2.0 (Copernicus Data Space account)

### Tomorrow.io Weather API
- **Description:** High-resolution weather intelligence platform with 60+ data layers including soil temperature, precipitation probability, wind, and agronomic risk indices. Supports historical data, real-time, and 14-day forecasts. Widely used as a weather data backend for precision agriculture platforms.
- **API Documentation:** https://docs.tomorrow.io/reference
- **SDKs/Libraries:** REST API; Postman collection at https://www.postman.com/postman/free-public-apis/documentation/t2w3990/tomorrow-io-api
- **Developer Guide:** https://support.tomorrow.io/hc/en-us/articles/31227543026708-How-to-Use-the-Tomorrow-io-API
- **Standards:** REST/JSON; OpenAPI described
- **Authentication:** API Key

### GeoPard Agriculture API
- **Description:** Precision agriculture software platform with WFS/GeoJSON endpoints for field boundaries and spatial data layers. March 2026 release added MCP server support and generative synthetic yield maps. Supports field trials design and AI-powered agronomic analytics.
- **API Documentation:** https://docs.geopard.tech/geopard-tutorials/api-docs/geo-endpoints/wfs-get-spatial-data-layers-in-vector-format-shp-geojson/1.-get-the-field-boundary-as-geojson
- **SDKs/Libraries:** REST/WFS endpoints
- **Developer Guide:** https://docs.geopard.tech/
- **Standards:** OGC WFS; GeoJSON; GeoParquet
- **Authentication:** API Key / session token

---

## Notes

**Emerging convergence around GeoParquet:** Both the ADAPT Standard 1.0 and fiboa specification adopted GeoParquet as the preferred format for large-scale vector data exchange. Platforms should plan to support GeoParquet ingestion and export alongside GeoJSON, particularly for bulk field boundary and prescription data.

**MCP as an emerging integration layer:** Leaf Agriculture's 2025 MCP server launch signals that the sector is beginning to treat MCP as a first-class integration point for AI agronomic agents. A precision agriculture platform that publishes an MCP server would gain immediate discoverability by AI tooling without requiring custom integrations.

**ADAPT Standard 1.0 adoption:** Released June 2024 with no software dependencies, ADAPT Standard 1.0 is positioned to become the dominant business-to-business data transfer format for production agriculture data. Early adoption is recommended for platforms targeting enterprise agribusiness and co-operative customers.

**EU Farm Data Spaces:** The EU's ongoing work on agricultural data spaces (building on GDPR and the Data Governance Act) may produce additional interoperability mandates for platforms operating in European markets. The ISO/TC 347 committee work is partially aligned with this initiative and should be monitored for emerging compliance obligations.

**Standards still evolving:** Semantic interoperability (ontologies, controlled vocabularies for crop types, pest species, growth stages) remains fragmented. ISO/TC 347 is the primary standardisation effort in this space but has not yet published normative standards. Platforms should align with AGROVOC (FAO-maintained multilingual agricultural thesaurus) as a provisional vocabulary standard.
