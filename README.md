# Smart Building Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source smart building platform that integrates IoT sensors, occupancy analytics, and automated controls without enterprise-only pricing or vendor lock-in.

Smart Building Platform is a multi-protocol building operations platform for facility managers, corporate real estate teams, and property operators. It ingests data from BMS, IoT sensors, and meters, normalises it on open ontologies, and applies AI to optimise HVAC, lighting, energy, and occupant comfort across single buildings and portfolios.

---

## Why Smart Building Platform?

- All full-featured incumbents (Johnson Controls OpenBlue, Siemens Building X / Desigo CC, Honeywell Forge, Willow, Facilio, Cohesion) are proprietary and exclusively custom-priced, leaving SMB and mid-market buildings without an accessible offering.
- Enterprise platforms require multi-year contracts, vendor professional services, or specialist BMS engineers to commission, slowing adoption and locking customers into one OEM hardware base.
- Integration is fragmented: products like Mapped solve normalisation but are not full platforms, while Occuspace and Kaiterra cover only the sensing layer and require downstream apps to deliver value.
- No production-grade open-source smart building platform exists. The strongest open assets — Brick Schema, Project Haystack, Google Digital Buildings Ontology, WillowTwin DTDL — are ontologies and tooling, not operational platforms.
- Generative AI diagnostics, natural language queries, and agentic AI (MCP) are appearing only in enterprise-tier products, leaving most buyers without modern AI capabilities.

---

## Key Features

### Data Ingestion and Normalisation

- Multi-protocol connectivity covering BACnet, MQTT, OPC UA, Modbus, and REST APIs
- Normalisation layer based on open ontologies (Brick Schema or Project Haystack)
- Cloud-to-cloud ingestion from third-party IoT sensor and BMS vendor APIs
- OpenAPI 3.x specification for third-party integration
- Webhook and event-driven automation hooks

### Real-Time Operations and Monitoring

- Real-time dashboards for occupancy, energy, and indoor air quality (CO2, humidity, temperature, particulates)
- Alarm management with configurable prioritisation and escalation
- Mobile access for field engineers and building operators
- Multi-site portfolio management with drill-down to floor and zone level
- Role-based access control and audit logging

### AI-Powered Optimisation

- HVAC and lighting setpoint optimisation driven by real-time occupancy, weather, and energy price signals
- Predictive fault detection across HVAC, lighting, access, and other major building systems
- Natural language query interface translating questions into API queries over building data
- AI-generated space utilisation insights and recommendations
- Carbon and energy performance forecasting with scenario modelling for proposed operational changes

### Maintenance and Workflow

- Predictive and condition-based maintenance with automatic ticket creation
- Work order management, contractor dispatch, and resolution tracking
- Compliance and audit workflows with digital records
- Cleaning schedule automation triggered by actual occupancy

### Occupant and Tenant Experience

- Tenant/occupant mobile experience covering access, comfort, and amenity booking
- Privacy-preserving occupancy sensing patterns (no camera or PII dependency required)
- Personalised environment preferences per occupant
- Room booking integration with automatic release of unused bookings

---

## AI-Native Advantage

Unlike incumbents that retrofit GenAI onto rule-based BMS stacks, this platform treats AI as a first-class layer: continuous setpoint optimisation against live occupancy, weather, and energy prices; multivariate anomaly detection that catches developing faults before threshold-based alerts fire; AI-generated space utilisation recommendations that replace manual BI dashboard review; and a natural language interface so non-specialist operators can query building data directly. Carbon and energy forecasting models the impact of operational changes on Scope 1/2 emissions before they are implemented.

---

## Tech Stack & Deployment

The platform is designed for self-hosted, cloud, and hybrid deployment. Data modelling builds on open standards: Brick Schema and Project Haystack for semantic interoperability, DTDL for digital twin definitions, BACnet (ASHRAE 135 / ISO 16484-5) and MQTT (ISO/IEC 20922) for device communication, and FIWARE/NGSI-LD where context data exchange is required. Integration is API-first with REST (OpenAPI 3.x) and optional GraphQL, plus an MCP server option for agentic AI workflows. ISO 52120 informs the energy performance and automation control approach.

---

## Market Context

The global smart building market is valued at approximately USD 163 billion in 2026, growing at a CAGR of 23.1% through 2030 (Precedence Research; Grand View Research). The occupancy sensor segment alone is projected to grow from USD 3.39 billion in 2025 to USD 8.67 billion by 2034 (Research and Markets). Enterprise platforms are exclusively custom-priced; primary buyers are corporate real estate and workplace directors, hospital and campus facility managers, commercial property managers, and sustainability/ESG teams.

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
