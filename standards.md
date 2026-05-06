# Standards & API Reference

> Project: Smart Building Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 16484 — Building Automation and Control Systems (BACS)**
- Parts: 16484-1 (Project specification, 2024), 16484-2 (Hardware, 2025), 16484-3 (Functions), 16484-4 (Control applications, 2025), 16484-5 (Data communication protocol / BACnet, 2012)
- URL: https://www.iso.org/standard/60689.html (Part 5); https://landingpage.bsigroup.com/LandingPage/Series?UPI=BS+EN+ISO+16484
- The foundational BACS standard series. Part 5 defines the BACnet protocol (co-published as ANSI/ASHRAE 135). Required for specification and implementation of any building automation system, defining hardware, software functions, control applications, and device interoperability.

**ISO 16484-5 / ANSI/ASHRAE 135-2024 — BACnet**
- URL: https://bacnet.org/ · https://www.ashrae.org/technical-resources/bookstore/bacnet
- The dominant open protocol for building automation device communication globally. ASHRAE 135-2024 (published December 2024) adds BACnet/SC (Secure Connect) for encrypted, authenticated device communication, certificate management, and Device Proxying. Also published as ISO 16484-5:2012 (internationally). Mandatory knowledge for any platform integrating HVAC, lighting, access, or fire systems.

**ISO/IEC 20922:2016 — MQTT v3.1.1**
- URL: https://www.iso.org/standard/69466.html · https://mqtt.org/
- Lightweight publish-subscribe messaging protocol standardised by ISO/IEC from the OASIS MQTT specification. Ideal for constrained IoT environments. Widely deployed in smart building sensor data streams. MQTT v5.0 published by OASIS offers enhanced error reporting and session expiry; v3.1.1 remains the most widely implemented version in building systems.

**ISO 52120-1:2021 — Energy Performance of Buildings — BACS Contribution**
- URL: https://www.iso.org/standard/65883.html · https://eubac.org/news/the-new-en-iso-52120-is-replacing-en-15232-eu-bac-guide/
- Replaces EN 15232. Defines a framework for assessing the contribution of building automation, controls, and building management to energy performance. Classifies BACS into four efficiency classes (A–D). Mandatory reference for any platform making energy efficiency claims in European markets. Compliance can reduce building energy consumption by up to 40%.

**IEC 62541 — OPC Unified Architecture (OPC UA)**
- URL: https://opcfoundation.org/about/opc-technologies/opc-ua/
- Cross-platform, open-source IEC standard for data exchange from industrial sensors to cloud applications. Supports both client-server and publish-subscribe (Pub/Sub) patterns. Increasingly used in building automation as the bridge between industrial OT systems and IT cloud platforms. Honeywell Forge explicitly lists OPC UA as a supported ingestion protocol.

---

### W3C & IETF Standards

**W3C Web of Things (WoT) Architecture 1.1 and Thing Description 1.1**
- URL: https://www.w3.org/WoT/ · https://w3c.github.io/wot-architecture/ · https://w3c.github.io/wot-usecases/
- W3C standard for IoT device interoperability across platforms and domains. A Thing Description defines a virtual or physical device's available actions, events, and properties in JSON-LD with semantic vocabulary. Siemens Desigo CC is a named WoT implementation. Relevant for designing a vendor-neutral device abstraction layer.

**IETF RFC 6455 — WebSocket Protocol**
- URL: https://datatracker.ietf.org/doc/html/rfc6455
- Standard for full-duplex communication over a single TCP connection. Used by smart building platforms for real-time telemetry streaming to browser-based dashboards and operational consoles.

**IETF RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Foundational HTTP standard underpinning all REST API designs in building platforms.

**IETF RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Standard for hypermedia linking in HTTP APIs; relevant for HAL/JSON:API-style building data APIs.

**JSON-LD 1.1 (W3C)**
- URL: https://www.w3.org/TR/json-ld11/
- JSON-based serialisation for Linked Data. Used as the foundation for NGSI-LD, DTDL, and Brick Schema RDF representations. Essential for semantic interoperability between building ontologies.

---

### Data Model & Ontology Standards

**Brick Schema (Open Source)**
- URL: https://brickschema.org/ · https://github.com/BrickSchema/Brick
- Open-source BSD-licensed ontology for describing building systems, equipment, sensors, and their relationships. Based on RDF/SPARQL. Adopted by Mapped as the core of their normalised building graph. Enables semantic interoperability between platforms and analytics applications. Reference implementation in Python available. Actively maintained with academic and industry contributors.

**Project Haystack v4/v5**
- URL: https://project-haystack.org/ · https://marketing.project-haystack.org/submitted-articles/haystack-5
- Open-source tagging and data modelling standard for building and industrial equipment. Version 4 introduced ontology capabilities (definitions, containment relationships, space modelling). Version 5 extends semantic engine capabilities. Adopted by Siemens, J2 Innovation, and others. Academic Free Licence 3.0. Cross-compatibility with Brick Schema is a stated long-term goal.

**DTDL — Digital Twins Definition Language v3 (Microsoft/Open)**
- URL: https://azure.github.io/opendigitaltwins-dtdl/DTDL/v3/DTDL.v3.html · https://github.com/Azure/opendigitaltwins-dtdl
- JSON-LD-based modelling language for IoT digital twins published by Microsoft. Based on open W3C standards (JSON-LD, RDF). Used by Azure Digital Twins and adopted by Siemens and Willow for building asset models. WillowTwin DTDL building ontology available on GitHub (Apache 2.0).

**Google Digital Buildings Ontology (DBO)**
- URL: https://google.github.io/digitalbuildings/ · https://github.com/google/digitalbuildings
- Apache 2.0-licensed open ontology and SDK for representing structured building data. Used internally by Google for 130+ buildings. Cross-compatibility goal with Brick Schema and Haystack. Includes ABEL (configuration tool), Instance Validator, and Explorer. Safe to use in commercial or open-source projects without IP risk.

**FIWARE NGSI-LD / ETSI CIM**
- URL: https://ngsi-ld.org/ · https://www.fiware.org/catalogue/
- ETSI-standardised context information management API and information model for smart cities and buildings. Builds on JSON-LD and Property Graphs. Used in EU-funded smart building and smart city projects. Provides a REST API for right-time context data with subscribe/notify patterns. Relevant for projects targeting European public sector or smart city deployments.

**OpenAPI Specification (OAS) 3.x**
- URL: https://www.openapis.org/
- De-facto standard for documenting REST APIs. Siemens Building X, Willow, and Honeywell Forge all publish OpenAPI v3 specifications. Required for any serious developer-facing building platform API.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URL: https://datatracker.ietf.org/doc/html/rfc6749 · https://openid.net/connect/
- Industry-standard authorisation and identity protocols. Used by Willow (Client ID/Secret OAuth flow) and Siemens Building X for API authentication. Required for any cloud-connected building platform API.

**BACnet/SC (BACnet Secure Connect — ASHRAE 135-2024 Addendum)**
- URL: https://bacnet.org/
- Security extension to BACnet adding TLS 1.3 encryption, X.509 certificate-based authentication, and Device Proxying. Required for secure BACnet device communication in modern deployments. Siemens Desigo CC is among the first BMS to achieve BACnet/SC certification.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- Reference checklist for securing APIs exposed by smart building platforms. Particularly relevant given the safety-critical nature of building system actuation APIs (HVAC setpoints, access control, fire systems).

**NIST Cybersecurity Framework 2.0**
- URL: https://www.nist.gov/cyberframework
- US federal framework for managing cybersecurity risk. Increasingly referenced in smart building procurement for OT/IT security requirements.

**IEC 62443 — Industrial Automation and Control Systems Security**
- URL: https://www.iec.ch/iec62443
- Security standard series for industrial and building automation control systems. Defines security levels for OT components and systems. Reference standard for BMS and smart building security architecture. Relevant for any platform integrating with building OT infrastructure.

---

### Connectivity Protocols (Additional)

**Matter v1.5 (Connectivity Standards Alliance)**
- URL: https://csa-iot.org/all-solutions/matter/
- Open-source, IP-based smart home and light-commercial building device standard. Version 1.5 (November 2025) adds cameras, soil moisture sensors, and energy management features. Primarily residential/light-commercial; increasingly relevant for commercial smart building edge devices. Runs over Thread, Wi-Fi, and Ethernet.

**KNX**
- URL: https://www.knx.org/
- Worldwide open standard (ISO/IEC 14543-3) for building automation — lighting, HVAC, access, energy metering. Widely deployed in European commercial buildings. Supported as a native protocol by Mapped's Universal Gateway.

**Modbus (IDA Modbus Application Protocol Specification)**
- URL: https://modbus.org/
- De-facto serial/TCP communication standard for industrial and building automation devices. Still widely deployed in legacy HVAC, meters, and sensors. Supported by all major platforms.

---

## Similar Products — Developer Documentation & APIs

### Siemens Building X

- **Description:** Cloud-based, AI-enabled open building platform connecting BMS, IoT, and enterprise systems. JSON:API and RESTful architecture with OpenAPI v3 specification.
- **API Documentation:** https://developer.siemens.com/building-x-openness/api/building-operations/overview.html
- **Getting Started:** https://developer.siemens.com/building-x-openness/dev-guide/gettingstarted.html
- **Developer Portal:** https://developer.siemens.com/building-x-openness/overview.html
- **MCP Server (Agentic AI):** https://developer.siemens.com/building-x-openness/dev-guide/mcp.html
- **Standards:** REST/JSON, JSON:API, OpenAPI 3.x
- **Authentication:** OAuth 2.0 (via Siemens developer registration)

---

### Honeywell Forge Buildings

- **Description:** Enterprise building operations platform with predictive maintenance, energy management, and sustainability modules. Exposes building data via API Marketplace.
- **API Marketplace:** https://buildings.honeywell.com/in/en/products/by-category/building-management/software/cloud-software/honeywell-forge-api-marketplace
- **Developer Portal:** https://devportal.prod.honeywell.com/
- **Getting Started:** https://developers.honeywell.com/get-started
- **SDKs/Libraries:** JAVA and Python SDKs available via developer portal
- **Microsoft Connector:** https://learn.microsoft.com/en-us/connectors/honeywellforge/
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** OAuth 2.0 / API Key

---

### Willow Digital Twin

- **Description:** Operational AI platform for buildings using 3D digital twin with live telemetry. Knowledge graph-based data model. REST API v2 with comprehensive twin, site, and asset endpoints.
- **API Documentation:** https://developers.willowinc.com/openapi/willowtwin/v2/
- **Developer Portal:** https://developers.willowinc.com/
- **Digital Twin Guide:** https://developers.willowinc.com/docs/resource-guide-digital-twin/
- **Open Ontology (GitHub):** https://github.com/WillowInc/opendigitaltwins-building (Apache 2.0)
- **Standards:** REST/JSON, OpenAPI v2, DTDL v3
- **Authentication:** OAuth 2.0 (Client ID/Secret — registration via developers@willowinc.com)

---

### Mapped

- **Description:** AI-powered data normalisation middleware for smart buildings. Exposes a single GraphQL API across all building systems, normalised against a Brick Schema-extended ontology.
- **API Documentation:** https://developer.mapped.com/docs/introduction
- **Standards:** GraphQL, Brick Schema ontology
- **Authentication:** API Key (developer portal registration)
- **Azure Marketplace:** https://appsource.microsoft.com/en-us/product/web-apps/mapped1653597792213.mapped_idl

---

### Facilio

- **Description:** API-first connected CMMS and buildings platform for multi-site portfolios. REST and GraphQL APIs with full developer documentation.
- **API Reference:** https://facilio.com/developers/docs/api-reference/
- **Data API (SDK Reference):** https://facilio.com/developers/docs/connected-apps/sdk-reference/api/
- **Developer Portal:** https://facilio.com/developers/docs/
- **Standards:** REST/JSON, GraphQL
- **Authentication:** OAuth 2.0 / API Key

---

### Occuspace

- **Description:** Privacy-safe occupancy analytics platform using passive WiFi/BLE sensing. REST API for live and historical occupancy data; BACnet/Modbus output for direct BMS integration.
- **API Documentation:** https://docs.occuspace.io/
- **Occupancy Guides:** https://docs.occuspace.io/guides/understanding-your-data/occupancy
- **Standards:** REST/JSON, BACnet, Modbus
- **Authentication:** API Key

---

### Azure Digital Twins (Microsoft)

- **Description:** PaaS service for creating live digital representations of real-world environments using DTDL models. Backend platform used by Willow and other smart building solutions.
- **API Documentation:** https://learn.microsoft.com/en-us/azure/digital-twins/
- **DTDL Specification:** https://azure.github.io/opendigitaltwins-dtdl/DTDL/v3/DTDL.v3.html
- **SDKs/Libraries:** .NET, Java, JavaScript, Python, Go SDKs available via Azure SDK
- **Standards:** REST/JSON, DTDL v3, OpenAPI
- **Authentication:** Azure Active Directory / OAuth 2.0

---

### Google Digital Buildings (Open Source)

- **Description:** Open-source ontology, SDK, and tooling for representing building asset metadata. Apache 2.0 licensed. Python SDK with validation and explorer tools.
- **GitHub Repository:** https://github.com/google/digitalbuildings
- **Ontology Documentation:** https://google.github.io/digitalbuildings/ontology/
- **SDKs/Libraries:** Python SDK in repository
- **Standards:** RDF-based ontology; CSV/YAML instance configuration files
- **Licence:** Apache 2.0

---

### Brick Schema (Open Source)

- **Description:** Open-source RDF ontology for describing building systems, equipment, and sensor relationships. SPARQL and GraphQL queryable. BSD licensed. Core data model for Mapped and others.
- **Website:** https://brickschema.org/
- **GitHub:** https://github.com/BrickSchema/Brick
- **Tools & Libraries:** https://brickschema.org/resources/ (Python tools, validators, converters)
- **Standards:** RDF, OWL, SPARQL, JSON-LD
- **Licence:** BSD 3-Clause

---

### Project Haystack (Open Source)

- **Description:** Open-source tagging and semantic modelling standard for building and industrial equipment data. Zinc, JSON, Trio data encoding formats. REST API specification (Haystack REST).
- **Website:** https://project-haystack.org/
- **Documentation:** https://project-haystack.org/doc/Intro
- **Standards:** Haystack REST API, Zinc/JSON/Trio encoding
- **Licence:** Academic Free Licence 3.0 (AFL 3.0)

---

## Notes

- **Ontology convergence is ongoing:** Brick Schema, Project Haystack, and Google DBO are converging toward shared semantic representations. A platform built today should monitor the RealEstateCore ontology (DTDL-based, widely used in Nordic countries) as an additional alignment target.
- **MCP integration is emerging:** Siemens Building X has documented MCP (Model Context Protocol) server support (developer.siemens.com/building-x-openness/dev-guide/mcp.html), making it the first major smart building platform to formally expose an agentic AI integration point. This is a strong signal for where the sector is heading.
- **BACnet/SC adoption is accelerating:** The 2024 ASHRAE 135 edition with BACnet/SC support is becoming a compliance requirement in new projects. Any platform integrating BACnet devices should plan for BACnet/SC from day one.
- **Matter protocol relevance:** Currently primarily residential and light-commercial. Monitor for commercial building expansion — Matter 2.x is expected to extend to more complex HVAC and access control device classes.
- **GDPR and privacy-by-design:** Platforms collecting occupancy, space utilisation, or environmental data tied to individuals (even indirectly via access card data) are subject to GDPR in EU deployments. Privacy-preserving sensing approaches (Occuspace-style passive WiFi/BLE without PII) are a design best practice.
