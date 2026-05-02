# Smart Building Platform

> Candidate #245 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| Johnson Controls OpenBlue | Cloud-based smart building platform integrating HVAC, security, fire, and energy with GenAI diagnostics | Commercial SaaS | Custom enterprise | Broad system integration; strong OEM hardware base; high cost and complexity |
| Siemens Desigo CC + Enlighted | Combined building management and IoT occupancy sensing platform (Siemens acquired Enlighted) | Commercial SaaS/on-prem | Custom enterprise | Deep BMS + sensor fusion; multi-protocol (BACnet, MQTT, DTDL); niche expertise required |
| Honeywell Forge Buildings | Unified building operations platform: energy, maintenance, security, and occupant experience | Commercial SaaS | Custom (per sqft/site) | Strong analytics; multi-year contract model; slower to innovate than start-ups |
| Willow Digital Twin | Digital twin platform for complex buildings and infrastructure; partnered with Johnson Controls | Commercial SaaS | Custom | Best-in-class digital twin visualisation; built for large campuses and airports |
| Facilio | API-first connected buildings platform for portfolios: maintenance, energy, and tenant engagement | Commercial SaaS | Custom | Modern architecture; strong multi-site management; growing ecosystem |
| Kaiterra / Awair | Indoor air quality sensing and analytics platforms feeding building automation adjustments | Commercial SaaS | From ~$500/year per sensor | Specialist IAQ focus; limited to sensing layer; needs integration work |
| Occuspace | Occupancy sensing and analytics using passive WiFi/BLE signals; no cameras required | Commercial SaaS | Custom | Privacy-preserving occupancy data; integrates with HVAC and room booking |
| Mapped | Normalisation and integration layer connecting disparate building systems to apps via standard APIs | Commercial SaaS | Custom | Solves the integration data problem; not a full platform |
| Cohesion | Tenant experience platform combining access, amenities, and building operations data | Commercial SaaS | Custom | Strong tenant engagement; primarily large commercial real estate |

## Relevant Industry Standards or Protocols

- **BACnet (ASHRAE 135 / ISO 16484-5)** — dominant open protocol for building automation device communication; required for interoperability across HVAC, lighting, and access systems
- **MQTT (ISO/IEC 20922)** — lightweight publish-subscribe messaging protocol widely used for IoT sensor data streams in smart buildings
- **DTDL (Digital Twin Definition Language)** — Microsoft's open modelling language for defining digital twin schemas; adopted by Siemens and others for building asset models
- **Brick Schema** — open-source ontology for describing building systems, equipment, and sensor relationships; enables semantic interoperability between platforms
- **Project Haystack** — open-source tagging and data modelling initiative for building and industrial equipment data
- **FIWARE / NGSI-LD** — EU-backed open standard for context data management in smart buildings and smart cities
- **ISO 52120** — Standard for energy performance of buildings and building automation control systems (replacing EN 15232)

## Available Research Materials

1. Precedence Research (2026). *Smart Building Market Size, Share and Trends 2026 to 2035*. precedenceresearch.com. https://www.precedenceresearch.com/smart-building-market
2. Grand View Research (2026). *Smart Building Market Size and Industry Report, 2033*. grandviewresearch.com. https://www.grandviewresearch.com/industry-analysis/global-smart-buildings-market
3. COR Advisors (April 2026). *Smart Building Technology Companies: Leading Innovators and Market Dynamics in 2026*. coradvisors.net. https://www.coradvisors.net/2026/04/smart-building-technology-companies-2026.html
4. Johnson Controls (2023). *Johnson Controls and Willow to Collaborate on Digital Solutions for Smart Buildings*. johnsoncontrols.com. https://www.johnsoncontrols.com/media-center/news/press-releases/2023/02/08/johnson-controls-and-willow-to-collaborate-on-digital-solutions
5. Siemens (2023). *Siemens Drives Digital Transformation in Buildings with Acquisition of Enlighted*. siemens.com. https://www.siemens.com/us/en/company/press/press-releases/smart-infrastructure/siemens-drives-digital-transformation-in-buildings-with-acquisition-of-enlighted-inc.html
6. Automatedbuildings.com (April 2026). *Ten Years of IoT in Building Automation: 2016 to 2026*. automatedbuildings.com. https://www.automatedbuildings.com/2026/04/ten-years-of-iot-in-building-automation-2016-to-2026/
7. Occuspace (2026). *Smart Building Technology: A Simple Guide to IoT and Sensors*. occuspace.com. https://www.occuspace.com/blog/smart-building-technology-simple-guide-to-iot-and-sensors
8. Research and Markets (2026). *Occupancy Sensor Market Outlook 2026–2034*. researchandmarkets.com. https://www.researchandmarkets.com/reports/6184172/occupancy-sensor-market-outlook-market-share

## Market Research

**Market Size:** The global smart building market is valued at approximately USD 163 billion in 2026, growing at a CAGR of 23.1% through 2030. The occupancy sensor segment alone is projected to grow from USD 3.39 billion (2025) to USD 8.67 billion by 2034.

**Funding:** Willow raised ~$63M (Series B, 2022); Facilio raised ~$35M; Mapped raised ~$15M. Incumbent players (Johnson Controls, Siemens, Honeywell) are investing billions in platform development and acquisitions rather than raising external capital.

**Pricing Landscape:** Enterprise platforms are exclusively custom-priced based on building area, system count, and module selection. Sensor-layer vendors (Occuspace, Kaiterra) use per-device or per-location models. Integration middleware (Mapped) is per-connection or per-site. No dominant SMB-accessible offering has emerged.

**Key Buyer Personas:** Corporate real estate and workplace directors managing large office portfolios; hospital and campus facility managers; commercial property managers seeking tenant experience differentiation; sustainability and ESG teams; smart city programme managers.

**Notable Trends:** Digital twins are transitioning from demonstration projects to operational standard for large portfolios; edge AI is pushing intelligence closer to sensors to reduce latency and cloud dependency; generative AI for building diagnostics and setpoint optimisation is being added by all major vendors; occupancy-based HVAC and lighting control is now table stakes.

## AI-Native Opportunity

- Autonomous HVAC and lighting optimisation using real-time occupancy data, weather forecasts, and energy prices — continuously adjusting setpoints without human intervention while maintaining comfort thresholds
- AI-generated space utilisation insights from anonymised occupancy patterns, enabling real estate teams to make evidence-based decisions about floor plate redesign, lease renewals, and desk sharing ratios
- Predictive fault detection across all building systems (HVAC, elevators, lighting, access) using fused sensor data to identify developing issues before occupants notice or service requests are filed
- Natural language building queries for facilities teams — "show me which meeting rooms on floors 5–8 were over 500 ppm CO₂ during business hours last week" — replacing manual BI dashboard navigation
- Carbon and energy performance forecasting that models the impact of specific operational changes (setback schedules, LED retrofits, occupancy thresholds) on Scope 1/2 emissions before they are implemented
