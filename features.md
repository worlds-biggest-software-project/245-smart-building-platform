# Smart Building Platform — Feature & Functionality Survey

> Candidate #245 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Johnson Controls OpenBlue | Commercial SaaS / enterprise | Proprietary — custom pricing | https://www.johnsoncontrols.com/openblue |
| Siemens Building X | Commercial SaaS / cloud | Proprietary — subscription | https://www.siemens.com/us/en/products/buildingtechnologies/building-x.html |
| Siemens Desigo CC | Commercial on-prem/cloud | Proprietary — custom | https://www.siemens.com/global/en/products/buildings/desigo-building-automation/building-management/desigo-cc.html |
| Honeywell Forge Buildings | Commercial SaaS | Proprietary — custom | https://buildings.honeywell.com/us/en/solutions/buildings |
| Willow Digital Twin | Commercial SaaS | Proprietary — custom | https://willowinc.com/ |
| Facilio | Commercial SaaS | Proprietary — custom | https://facilio.com/ |
| Mapped | Commercial SaaS / middleware | Proprietary — custom | https://www.mapped.com/ |
| Cohesion | Commercial SaaS | Proprietary — custom | https://www.cohesionib.com/ |
| Occuspace | Commercial SaaS | Proprietary — custom | https://www.occuspace.com/ |
| Google Digital Buildings | Open source | Apache 2.0 | https://google.github.io/digitalbuildings/ |
| Home Assistant | Open source | Apache 2.0 | https://www.home-assistant.io/ |
| openHAB | Open source | EPL-2.0 | https://www.openhab.org/ |

---

## Feature Analysis by Solution

### Johnson Controls OpenBlue

**Core features**
- Unified data ingestion from 130+ building system sources, normalised into a single data layer
- AI-powered energy forecasting and autonomous HVAC setpoint optimisation
- Generative AI diagnostics recommending energy savings projects proactively
- Real-time monitoring with dashboards across entire building portfolios
- Indoor air quality (IAQ) monitoring and analytics (CO₂, humidity, temperature, particulates)
- Predictive maintenance across HVAC, fire, security, and access systems
- Compliance management and audit trails
- Visitor tracking and space utilisation analytics across portfolios

**Differentiating features**
- OpenBlue Enterprise Manager Suite integrating HVAC, fire, security, and energy in one UI
- Native integration with Johnson Controls hardware (Metasys BMS, York HVAC, Tyco security)
- Generative AI co-pilot for building diagnostics embedded in operational dashboards
- Cyber-secured architecture with focus on OT/IT security

**UX patterns**
- Role-based dashboards for facility managers, sustainability teams, and executives
- Portfolio-level summarisation with drill-down to individual floor/zone level
- Personalisation of alerts and notification thresholds per user
- Progressive disclosure: operational summary first, detailed analytics on demand

**Integration points**
- REST APIs for third-party app and analytics tool connectivity
- BACnet, Modbus, LonWorks protocol support for device-level integration
- Cloud-to-cloud API ingestion from third-party vendor systems
- Microsoft Azure cloud infrastructure

**Known gaps**
- High implementation cost and complexity — not accessible to mid-market
- Slower product iteration cycle compared to startups
- Requires Johnson Controls professional services for full deployment
- UI can feel overwhelming for non-specialist building operators

**Licence / IP notes**
- Fully proprietary; no open-source components exposed. Custom enterprise contracts required.

---

### Siemens Building X

**Core features**
- Cloud-based open platform connecting OT building systems to IT enterprise applications
- AI and ML-powered real-time analytics and forecasting
- Sustainability Manager for net-zero tracking and Scope 1/2 energy reporting
- BMS integration layer supporting native cloud-connected Siemens hardware and third-party gateways
- Operations Manager for multi-site remote monitoring and management
- 24/7 IT/OT data integration breaking silos across all building systems

**Differentiating features**
- JSON:API-based RESTful developer API with OpenAPI v3 specification published
- MCP (Model Context Protocol) server support for agentic AI integration — one of first in sector
- Open ecosystem with Xcelerator Developer Portal enabling third-party app development
- llms.txt endpoint for LLM-native documentation discovery
- Up to 10% improvement in net operating income reported from operational workflow optimisation

**UX patterns**
- Single cloud console for all connected buildings regardless of BMS vendor
- Mobile-first Flex Client for on-the-go operations
- Vendor-agnostic approach reduces lock-in anxiety in procurement

**Integration points**
- Native integration with Desigo CC via BACnet/SC
- OpenAPI v3 specification available at Xcelerator Developer Portal
- SDKs and low-code development tooling
- AWS-hosted; Siemens provides managed hosting

**Known gaps**
- Requires Siemens hardware or certified gateway for full capability
- Developer ecosystem still maturing relative to general SaaS platforms
- Pricing model not transparent

**Licence / IP notes**
- Proprietary SaaS. Developer API access via subscription. No open-source components.

---

### Siemens Desigo CC

**Core features**
- Multi-discipline BMS managing HVAC, lighting, fire safety, security, and electrical systems
- Modular architecture scaling from single buildings to large multi-site complexes
- Energy management with Powermanager module (electrical meters, consumption dashboards, power quality)
- BACnet B-XAWS profile support for cross-domain workstation integration
- Mobile app (Flex Client) delivering full command and control on phone-sized devices
- Real-time push notifications for alarms and system events

**Differentiating features**
- First major BMS to fulfil BACnet Secure Connect (BACnet/SC) profile
- Native bridge to Building X cloud platform for cloud analytics overlay
- Decades of deployment track record in critical infrastructure (hospitals, airports, campuses)

**UX patterns**
- Operator-centric single-pane-of-glass for all building disciplines
- Mobile app mirrors full desktop functionality for field engineers
- Configurable alarm prioritisation and escalation workflows

**Integration points**
- BACnet, OPC, Modbus, SNMP, LonWorks protocol support
- Device compatibility across hundreds of third-party manufacturers
- REST APIs for cloud and enterprise system connectivity

**Known gaps**
- On-premises deployment complexity; cloud transition requires Building X add-on
- High licensing and engineering cost for full multi-discipline deployment
- Specialist Siemens-trained engineers required for commissioning

**Licence / IP notes**
- Proprietary licensed software. On-premises installation with annual maintenance contracts.

---

### Honeywell Forge Buildings

**Core features**
- Predictive maintenance using equipment models, real-time analytics, and wireless plug-and-play sensors
- Zone comfort monitoring across portfolios (temperature, humidity, CO₂)
- Sustainability+ module for carbon and energy management with Scope 1/2 reporting
- BMS alarm management, schedule adjustment, and remote building control
- SAP Cloud for Real Estate integration for property management workflows
- API Marketplace with self-serve developer console and sandbox environment

**Differentiating features**
- Honeywell Forge API Marketplace exposing standardised building data APIs for third-party development
- Plug-and-play wireless sensors for rapid asset connectivity without cabling
- Low-code ingestion workflows for quick BMS onboarding
- JAVA and Python SDKs available via developer portal

**UX patterns**
- Performance dashboards with near real-time KPIs for facility service teams
- Corrective action tracking from issue identification to resolution closure
- Portfolio-level comfort score summaries with floor-level drill-down

**Integration points**
- OPC UA, Modbus TCP, BACnet IP, MQTT, HTTP/REST data ingestion
- Microsoft Power Platform connector available
- Honeywell Forge API Marketplace at buildings.honeywell.com
- Developer portal at devportal.prod.honeywell.com

**Known gaps**
- Multi-year contract model slows adoption for smaller portfolios
- API Marketplace ecosystem still developing vs. mature cloud platforms
- UI design feels enterprise-heavy rather than consumer-grade

**Licence / IP notes**
- Proprietary SaaS. API access available via commercial agreements.

---

### Willow Digital Twin

**Core features**
- 3D building digital twin fusing spatial geometry with live operational data from 75+ system types
- Willow Knowledge Graph unifying asset, system, and operational data with external context
- Real-time, historical, and predictive analytics for building operations
- Predictive maintenance with continuous condition monitoring across building components
- AI companion for facility teams providing contextual guidance
- 10 million+ telemetry points processed in real time
- REST API with OpenAPI v2 specification

**Differentiating features**
- WillowTwin open DTDL ontology for buildings and real estate (Apache-licensed) published on GitHub
- Best-in-class 3D spatial visualisation overlaid with live operational data
- Industry-specific vertical solutions for healthcare, higher education, aviation, and retail
- Azure Digital Twins integration via DTDL modelling

**UX patterns**
- Immersive 3D building navigator as primary UX paradigm — spaces, floors, assets contextualised
- AI companion reduces reliance on BI dashboard expertise
- Progressive disclosure: 3D view → floor/zone → individual asset → telemetry stream

**Integration points**
- REST API v2 at developers.willowinc.com
- OAuth 2.0 authentication (Client ID/Secret registration via email)
- Azure Digital Twins backend
- Twins, Sites, Assets, Relationships endpoints in API

**Known gaps**
- Complex onboarding — DTDL modelling requires specialist knowledge
- API registration manual (email-based) rather than self-serve
- High cost for initial digital twin creation and validation
- Limited SMB or mid-market offering

**Licence / IP notes**
- Platform is proprietary commercial SaaS. WillowTwin DTDL ontology open-sourced under Apache 2.0 on GitHub (github.com/WillowInc/opendigitaltwins-building). No patent concerns identified.

---

### Facilio

**Core features**
- API-first connected CMMS with real-time BMS data integration
- Multi-site facility management: helpdesk, maintenance, inspections, inventory, asset management
- Tenant and customer servicing portal with contractor management
- Condition-based and predictive maintenance using fused sensor data
- Native analytics for performance trends and cross-system data relationship visualisation
- Compliance and audit tracking with digital workflows
- Workplace management including desk booking and space management

**Differentiating features**
- First "Connected CMMS" — merges traditional CMMS with live building IoT data
- API-first architecture with GraphQL and REST APIs at facilio.com/developers
- Seamless BMS, sensor, BIM, and third-party software integration without rip-and-replace
- Multi-tenant, multi-portfolio architecture designed for property management companies

**UX patterns**
- Unified mobile and web interface for field engineers and portfolio managers
- Configurable workflow builder for maintenance and compliance processes
- Real-time fault detection with integrated ticket creation and resolution tracking

**Integration points**
- REST and Data API at facilio.com/developers/docs/api-reference
- GraphQL SDK reference documentation
- BMS, BIM, IoT sensor, and enterprise software integration connectors
- Webhook and event-driven automation support

**Known gaps**
- Less sophisticated 3D/spatial visualisation vs. digital twin specialists
- Analytics depth limited vs. dedicated building intelligence platforms
- Energy management and sustainability reporting less mature than Honeywell/Siemens

**Licence / IP notes**
- Proprietary SaaS. API documentation publicly accessible. No open-source components.

---

### Mapped

**Core features**
- Automated discovery, extraction, and normalisation of data from building systems and vendor APIs
- Universal Gateway supporting native protocols: BACnet, Crestron, KNX, Lutron, Modbus, LonWorks
- Brick Schema-extended Mapped Graph ontology for normalised data representation
- GraphQL API exposing normalised People, Places, and Things data model
- Console for reviewing and working with normalised building data

**Differentiating features**
- Single standardised GraphQL API replacing dozens of point-to-point integrations
- AI-powered protocol discovery and data extraction — reduces manual mapping effort significantly
- Free-tier access for data normalisation platform announced (lowering entry barrier)
- Microsoft Azure Marketplace listing enabling enterprise procurement

**UX patterns**
- Developer-first: minimal UI; primary interaction is via API and developer portal
- Console for data review and validation rather than operational monitoring
- Integration-layer positioning — feeds data to downstream apps rather than end-user dashboards

**Integration points**
- GraphQL API at developer.mapped.com/docs/introduction
- Universal Gateway hardware or software appliance for on-premises protocol bridging
- Cloud API integrations with leading IoT sensor vendors
- Azure Marketplace listing for enterprise customers

**Known gaps**
- Not a full platform — no operational dashboards, analytics, or maintenance workflows
- Requires downstream app or analytics platform to deliver end-user value
- Hardware gateway dependency for on-premises BMS integration

**Licence / IP notes**
- Proprietary commercial SaaS. GraphQL API access via commercial agreement. Brick Schema ontology (used in data model) is open source under BSD licence.

---

### Cohesion

**Core features**
- Tenant experience platform combining access control, amenities, and workplace services
- Smart Access: mobile wallet credentials, biometric (face recognition), app-based entry and movement
- Occupancy and space demand tracking with movement pattern analytics
- Environmental performance and comfort score monitoring
- EV charging, solar, and demand response integration for energy strategy
- Cleaning schedule optimisation based on actual space usage data

**Differentiating features**
- UL Solutions Smart Systems Verification — first access-driven real estate software to receive this certification
- Unified occupant experience: seamless building entry through workplace services in one app
- Load shifting and grid participation for demand response programmes

**UX patterns**
- Mobile app as primary occupant touchpoint — self-service amenity booking, access control, room environments
- Operations dashboard for building managers with tenant satisfaction metrics
- Personalised environment preferences per occupant (temperature, lighting setpoints)

**Integration points**
- Access control hardware integrations (badge, biometric, mobile)
- BMS integrations for environment control triggers
- APIs for enterprise workplace tools and HR systems

**Known gaps**
- Primarily large commercial real estate — limited outside that vertical
- Energy management depth less than dedicated sustainability platforms
- BMS/IoT integration breadth less comprehensive than specialist platforms

**Licence / IP notes**
- Proprietary SaaS. No open-source components identified.

---

### Occuspace

**Core features**
- Passive WiFi/BLE signal-based occupancy sensing — no cameras, no PII collected
- Real-time and historical occupancy data: count, peak, average, dwell time KPIs
- Space utilisation analytics identifying underused vs. over-occupied spaces
- HVAC and lighting control triggers based on real-time occupancy state
- Room booking system integration for automatic release of unused booked rooms
- Cleaning schedule automation triggered by actual usage data

**Differentiating features**
- Privacy-first by design — no camera infrastructure required, no personally identifiable data
- REST API with live and historical occupancy data for any downstream system
- BACnet and Modbus output for direct BMS integration without middleware
- Works in any space type without physical space reconfiguration

**UX patterns**
- Web portal for analytics access by space planners and facility managers
- API-first for integration into existing IWMS, BMS, booking, and workplace apps
- Minimal hardware footprint — sensor deployment measured in hours, not days

**Integration points**
- REST API at docs.occuspace.io with live and historical endpoints
- BACnet and Modbus native output for BMS integration
- Integrations with IWMS, room booking, and facility management platforms

**Known gaps**
- Occupancy sensing only — no broader building management capabilities
- No native analytics dashboard beyond occupancy metrics
- Requires integration work to connect to HVAC and lighting systems for automation

**Licence / IP notes**
- Proprietary SaaS. REST API access via commercial agreement.

---

### Google Digital Buildings (Open Source)

**Core features**
- Open ontology (Digital Buildings Ontology, DBO) for describing building assets and systems
- ABEL tool for building configuration file creation from Google Sheets templates
- Instance Validator for validating building configuration files against ontology
- Explorer tool for browsing and comparing ontology types
- Telemetry validation tooling

**Differentiating features**
- Apache 2.0 licensed — fully open source with no licence compatibility concerns
- Used internally by Google to manage 130+ Bay Area buildings — production-validated
- Cross-compatibility goal with Brick Schema and Project Haystack

**UX patterns**
- Developer/engineer tooling rather than operational UX
- Command-line and scripting interface

**Integration points**
- GitHub-hosted at github.com/google/digitalbuildings
- Python SDK for ontology tooling
- Designed to integrate with BMS data normalisation pipelines

**Known gaps**
- Ontology/SDK tooling only — not a full operational platform
- Limited community outside Google ecosystem
- No commercial support or SLA

**Licence / IP notes**
- Apache 2.0 licence throughout. No patent concerns. Safe to use in open-source projects.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Multi-protocol BMS device connectivity (BACnet, Modbus, MQTT, OPC UA)
- Real-time operational dashboards for facility managers
- Energy consumption monitoring and reporting
- Predictive/preventive maintenance workflows
- Alarm management and escalation
- Mobile access for field engineers and building operators
- Multi-site portfolio management
- Role-based access control and audit logging
- REST or GraphQL API for third-party integration

### Differentiating Features
- 3D digital twin with live spatial data overlay (Willow)
- Generative AI building diagnostics and natural language queries (OpenBlue, Building X)
- Privacy-preserving occupancy sensing without cameras (Occuspace)
- Single normalised API layer across all building protocols (Mapped)
- MCP server / agentic AI integration (Building X — first in sector)
- Open-source ontology with production validation at scale (Google DBO)
- Tenant experience and mobile occupant engagement (Cohesion)
- Connected CMMS bridging operational and IT/OT data (Facilio)

### Underserved Areas / Opportunities
- No dominant open-source, production-grade smart building platform exists — all full platforms are proprietary enterprise tools
- Natural language interface for building queries is nascent — available in enterprise-only products
- SMB and mid-market buildings have no accessible, affordable platform; all solutions are custom-priced enterprise
- Carbon forecasting before implementing operational changes is missing from most platforms
- Cross-portfolio benchmarking to compare building performance across similar asset types is rare
- Interoperability between competing platforms is absent — no standard data exchange layer (Mapped addresses integration but not interoperability)
- Explainable AI recommendations — most platforms produce alerts without sufficient context for non-specialist operators
- Standardised tenant satisfaction measurement across buildings

### AI-Augmentation Candidates
- HVAC and lighting setpoint optimisation: currently rule-based schedules; AI can use real-time occupancy, weather, and energy price signals continuously
- Fault detection and diagnostics: currently threshold-based alerts; AI can identify multivariate anomaly patterns before failures occur
- Space utilisation analysis and real estate recommendations: manual BI dashboard review; AI can surface evidence-based lease and redesign recommendations automatically
- Predictive energy budgeting: manual forecasting from historical consumption; AI can model impact of operational changes on Scope 1/2 before implementation
- Natural language building queries replacing dashboard navigation for non-specialist users

---

## Legal & IP Summary

All major commercial platforms (OpenBlue, Building X, Desigo CC, Honeywell Forge, Willow, Facilio, Mapped, Cohesion, Occuspace) are fully proprietary with no open-source components that would impose licence obligations. The key open-source assets in this domain are: Google's Digital Buildings Ontology (Apache 2.0), the Brick Schema (BSD), Project Haystack (Academic Free Licence 3.0), and the WillowTwin DTDL ontology (Apache 2.0). All are permissively licensed and safe for use in commercial or open-source projects. No patent concerns were identified in any of the open-source components. The DTDL specification itself is published by Microsoft under a permissive licence. Building new tools on top of Brick Schema, Haystack, or Google DBO carries no IP risk.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Multi-protocol data ingestion (BACnet, MQTT, REST APIs) with normalisation layer using Brick Schema or Haystack ontology
- Real-time occupancy, energy, and environmental (IAQ) monitoring dashboard
- AI-powered HVAC and lighting optimisation based on occupancy and energy price signals
- Predictive fault detection across major building systems with alert and ticket creation
- Natural language query interface for building data (NL → API query translation)
- REST API with OpenAPI 3.x specification for third-party integration

**Should-have (v1.1)**
- Carbon and energy performance forecasting with scenario modelling
- Space utilisation analytics with AI-generated insights and recommendations
- Multi-site portfolio management with cross-building benchmarking
- Tenant/occupant mobile experience app (access, comfort, amenity booking)
- Maintenance workflow management (work orders, contractor dispatch, compliance)

**Nice-to-have (backlog)**
- 3D digital twin visualisation with live data overlay
- Integration with IWMS and enterprise ERP/EAM systems
- Demand response and grid participation automation
- Open ontology publishing for customer data portability
- MCP server for agentic AI integration
