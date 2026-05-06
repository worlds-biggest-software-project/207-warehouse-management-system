# Standards & API Reference

> Project: Warehouse Management System · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Governs traceability, inspection, non-conformance recording, and corrective action workflows embedded in WMS quality modules. Required for certified operations in aerospace, defence, and medical device supply chains.

**ISO 28000:2022 — Security Management System for the Supply Chain**
- URL: https://www.iso.org/standard/79612.html
- Specifies requirements for a security management system covering financing, manufacturing, information management, transportation, in-transit storage, and warehousing of goods. Relevant to WMS access control, audit trails, and cargo security processes. Aligns with ISO 9001, ISO 14001, ISO 45001, and ISO/IEC 27001 via the Annex SL high-level structure.

**ISO/IEC 19987:2024 — EPCIS 2.0 (Information Technology — EPC Information Services)**
- URL: https://www.iso.org/standard/88071.html
- ISO adoption of the GS1 EPCIS 2.0 standard. Defines the event data model, REST/OpenAPI and SOAP/WSDL interfaces, JSON-LD and XML data formats for capturing and sharing supply chain visibility data. Mandatory for pharmaceutical track-and-trace (DSCSA) and increasingly required in food safety (FSMA 204) compliance.

**ISO 8000 — Data Quality**
- URL: https://www.iso.org/standard/50798.html
- Defines quality requirements for master data (product, location, trading partner) managed within a WMS. Relevant to GTIN, SSCC, and GLN data accuracy standards.

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines HTTP methods (GET, POST, PUT, PATCH, DELETE), status codes, and content negotiation used as the transport layer for all modern WMS REST APIs.

**RFC 6749 — The OAuth 2.0 Authorisation Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundation for WMS API authentication. Manhattan Active, Oracle WMS Cloud, and OpenWMS.org all use OAuth 2.0 for API access control.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Governs how bearer tokens are transmitted in WMS API requests, relevant to any REST API integration.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWTs are widely used as the token format in OAuth 2.0 flows for WMS APIs. Relevant to API security and session management design.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0; used by OpenWMS.org (Spring Authorization Server) and enterprise WMS platforms for multi-tenant identity management.

---

### Data Model & API Specifications

**GS1 EPCIS 2.0 / CBV 2.0**
- URL: https://www.gs1.org/standards/epcis
- Implementation Guideline: https://ref.gs1.org/guidelines/epcis-cbv/
- The primary supply chain visibility event standard. Defines object events, aggregation events, transaction events, and transformation events. EPCIS 2.0 (ratified June 2022) adds REST/OpenAPI and JSON-LD support alongside legacy SOAP/XML. Critical for receiving (SSCC scan), lot genealogy, cold-chain monitoring, pharmaceutical DSCSA, and FDA FSMA 204 compliance.

**GS1 General Specifications (GTIN, SSCC, GLN, GS1-128 Barcode)**
- URL: https://www.gs1.org/standards/gs1-general-specifications
- Defines barcode symbologies (GS1-128, GS1 DataBar, GS1 DataMatrix, QR Code), GTIN product identifiers, SSCC shipping container codes, and GLN location codes. The foundation of all WMS barcode scanning, label printing, and ASN workflows.

**ANSI ASC X12 856 — Advance Ship Notice (ASN)**
- URL: https://www.x12.org/
- The EDI transaction set for advance ship notices sent by suppliers to distribution centres before physical delivery. Triggers automated WMS receiving workflows. Used in receiving, dock scheduling, and cross-docking processes.

**ANSI ASC X12 940 — Warehouse Shipping Order**
- URL: https://www.x12.org/
- EDI transaction set used by manufacturers and sellers to instruct a third-party warehouse (3PL) to ship an order. Core 3PL WMS integration standard.

**ANSI ASC X12 945 — Warehouse Shipping Advice**
- URL: https://www.x12.org/
- EDI transaction sent by the warehouse back to the seller confirming shipment; used to reconcile shipped quantities, generate invoices, and trigger downstream 856 ASN creation.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- Industry standard for REST API definition. Oracle WMS Cloud and Manhattan Active both publish OpenAPI-compatible REST APIs. The recommended format for any new WMS API surface.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Standard for validating the structure of JSON payloads in WMS API requests and responses (inventory records, order objects, location data).

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- Used by Manhattan Active, Oracle WMS Cloud, Blue Yonder, and OpenWMS.org for API authentication. Resource Owner Password Credentials grant and Client Credentials grant are the most common WMS API flows.

**OpenID Connect Core 1.0**
- Multi-tenant identity management for WMS platforms with multiple clients (3PL use case). OpenWMS.org implements OIDC via Spring Authorization Server.

**OSHA 29 CFR 1910.178 — Powered Industrial Trucks (Forklift Safety)**
- URL: https://www.osha.gov/laws-regs/regulations/standardnumber/1910/1910.178
- Federal US regulation governing forklift operator certification, training programmes, refresher training (every 3 years), equipment modification approval, and workplace evaluation. Informs WMS task configuration for forklift-directed operations and labour management certification tracking.

**OSHA Materials Handling and Storage (29 CFR 1910 Subpart N)**
- URL: https://www.osha.gov/sites/default/files/publications/OSHA2236.pdf
- Governs safe stacking, aisle clearance, load limits, and rack stability requirements. Relevant to WMS location configuration, weight limit enforcement, and put-away constraint logic.

**NIST SP 800-53 — Security and Privacy Controls for Information Systems**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- Provides the security control framework relevant to WMS deployments in US government, defence, and regulated industry environments. Informs access control, audit logging, and encryption requirements.

**GDPR (EU) 2016/679**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32016R0679
- Applies where WMS systems process personal data (e-commerce fulfilment with consumer address data, employee productivity tracking). Relevant to data retention, access control, and right-to-erasure workflows.

**FDA FSMA 204 — Food Traceability Rule**
- URL: https://www.fda.gov/food/food-safety-modernization-act-fsma/fsma-final-rule-requirements-additional-traceability-records-certain-foods
- Requires electronic lot-level traceability records (Critical Tracking Events and Key Data Elements) for FDA-listed foods. WMS must capture and share EPCIS-formatted traceability events for covered commodities.

**DSCSA — Drug Supply Chain Security Act (US)**
- URL: https://www.fda.gov/drugs/drug-supply-chain-security-act-dscsa
- Requires serialised track-and-trace for prescription drugs throughout the supply chain. WMS must capture EPCIS events at receipt, storage, pick, and dispatch for pharmaceutical products.

---

### MCP Server Specifications

The Model Context Protocol is relevant to a WMS that exposes an AI assistant interface for natural-language warehouse performance queries. An MCP server sitting in front of the WMS data layer could enable LLM agents to query inventory positions, labour metrics, and order status without custom API integration.

**Model Context Protocol (MCP) Specification**
- URL: https://modelcontextprotocol.io/specification
- Defines the protocol for connecting LLM agents to external data and tool APIs. Applicable to the natural-language WMS query assistant feature described in the AI-native opportunity analysis.

---

## Similar Products — Developer Documentation & APIs

### Manhattan Active WM

- **Description:** Enterprise cloud-native WMS built on microservices. Every functional domain exposed as a REST API including warehouse, labour, slotting, and automation.
- **API Documentation:** https://developer.manh.com/docs/how-to/rest-api/reference/
- **Developer Hub:** https://developer.manh.com/docs/
- **SDKs/Libraries:** No published SDK; REST/JSON calls from any HTTP client
- **Developer Guide:** https://www.manh.com/solutions/manhattan-active-platform/developer-tools-api/developer-hub
- **Standards:** REST/JSON, OpenAPI-compatible, microservices
- **Authentication:** OAuth 2.0 (Resource Owner Password Credentials grant)

---

### Oracle Warehouse Management Cloud

- **Description:** Mid-to-large enterprise cloud WMS providing REST APIs for data integration and key WMS operations including receiving, inventory management, and shipment processing.
- **API Documentation (REST):** https://docs.oracle.com/en/cloud/saas/warehouse-management/26a/owmre/index.html
- **Integration API Guide:** https://docs.oracle.com/en/cloud/saas/warehouse-management/26b/owmap/integration-api-guide.pdf
- **SDKs/Libraries:** No published SDK; REST/JSON standard HTTP client
- **Developer Guide:** https://docs.oracle.com/en/cloud/saas/warehouse-management/20d/owmap/wms-web-service-apis.html
- **Standards:** REST/JSON, OpenAPI-compatible
- **Authentication:** OAuth 2.0 (Client Credentials grant)

---

### SAP Extended Warehouse Management (EWM)

- **Description:** Enterprise WMS integrated with SAP S/4HANA. APIs exposed via the SAP Business Accelerator Hub covering EWM integration for manufacturing execution, goods movements, and inventory management.
- **API Documentation:** https://api.sap.com/api/sapdme_ewm_integration/overview
- **SAP Business Accelerator Hub:** https://api.sap.com/
- **SDKs/Libraries:** SAP Cloud SDK (Java, JavaScript) for SAP BTP integrations
- **Developer Guide:** https://help.sap.com/docs/SAP_EXTENDED_WAREHOUSE_MANAGEMENT
- **Standards:** REST/OData, SOAP/IDoc for legacy integration, OpenAPI on Accelerator Hub
- **Authentication:** OAuth 2.0 via SAP BTP Identity Authentication Service

---

### Blue Yonder WMS

- **Description:** Enterprise AI-powered WMS with REST and SOAP API layers. Developer portal at success.blueyonder.com; integration services available via Cleo, Makini, and system integrators.
- **API Documentation:** https://success.blueyonder.com/ (login required for full docs)
- **Integration Connector Reference:** https://apitracker.io/a/blue-yonder
- **SDKs/Libraries:** No published SDK; REST/JSON and SOAP/WSDL
- **Developer Guide:** https://help.metapack.com/hc/en-gb/articles/4415871013137
- **Standards:** REST/JSON, SOAP/WSDL (legacy); EDI for trading partners
- **Authentication:** OAuth 2.0 / API key (varies by endpoint)

---

### Ongoing WMS

- **Description:** Cloud-based WMS with fully documented REST and SOAP APIs for e-commerce and ERP integration. Positioned as an accessible WMS with strong developer documentation.
- **API Documentation (REST):** https://developer.ongoingwarehouse.com/REST/v1/index.html
- **Automation SOAP API:** https://developer.ongoingwarehouse.com/automation-api
- **Developer Pages:** https://developer.ongoingwarehouse.com/
- **SDKs/Libraries:** No published SDK; REST/JSON and SOAP/WSDL
- **Developer Guide:** https://docs.ongoingwarehouse.com/
- **Standards:** REST/JSON, SOAP/WSDL; OpenAPI-style REST documentation
- **Authentication:** API key / Basic auth

---

### OpenWMS.org

- **Description:** Apache 2.0 open-source WMS with embedded Material Flow Control. Microservices architecture with REST APIs documented in the OpenWMS Cloud Wiki. Best open-source reference for a new project.
- **API Documentation:** https://wiki.openwms.cloud/projects/openwms/wiki/01-dot-01-introduction
- **GitHub Organisation:** https://github.com/openwms
- **SDKs/Libraries:** Java/Spring Boot microservices; Flutter mobile client
- **Developer Guide:** https://openwms.github.io/org.openwms/
- **Standards:** REST/JSON, OAuth 2.0 / OIDC (Spring Authorization Server), Twelve-Factor app methodology
- **Authentication:** OAuth 2.0 / OpenID Connect

---

### Fishbowl Inventory

- **Description:** SMB-targeted inventory and warehouse management with QuickBooks/Xero integration. REST API available for custom integration and ERP connectivity.
- **API Documentation:** https://www.fishbowlinventory.com/integration/api/
- **SDKs/Libraries:** No published SDK; REST/JSON
- **Developer Guide:** Fishbowl developer portal (login required)
- **Standards:** REST/JSON
- **Authentication:** API key

---

## Notes

- **EPCIS 2.0 / ISO IEC 19987:2024** is rapidly becoming a mandatory compliance standard for food and pharmaceutical WMS due to FDA FSMA 204 and DSCSA enforcement. New WMS projects should include EPCIS 2.0 REST event capture from the outset rather than retrofitting.
- **GS1 standards** (GTIN, SSCC, GLN, GS1-128) are the universal foundation of barcode scanning in warehousing; any WMS must support GS1 label generation and scanning to interoperate with trading partners.
- **EDI X12 (856, 940, 945)** remains dominant in North American 3PL and retail distribution; EDIFACT is used in European and global supply chains. Both should be supported for broad market reach.
- **OAuth 2.0** is the de facto standard for WMS API authentication across all reviewed vendors; new implementations should not use API-key-only or Basic auth for production integrations.
- **OpenAPI 3.1** is the recommended format for documenting a new WMS REST API surface, enabling auto-generated client SDKs and interactive documentation.
- **MCP server integration** is an emerging pattern for AI-native WMS applications; standardising on MCP for the natural-language query assistant enables LLM agent interoperability without custom integration work.
