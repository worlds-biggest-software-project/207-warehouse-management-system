# Warehouse Management System (WMS)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source warehouse management system covering receiving, putaway, picking, packing, shipping, and cycle counts — built to make modern WMS capability accessible beyond the enterprise tier.

The Warehouse Management System project is a candidate open-source platform for distribution centres, 3PLs, manufacturers, and e-commerce operators. It targets the gap between cost-prohibitive enterprise WMS suites (Manhattan, Blue Yonder, SAP EWM) and SMB inventory tools (Fishbowl) by providing AI-driven optimisation on an open foundation.

---

## Why a New WMS?

- Enterprise WMS first-year costs commonly run **$500K–$3M+** (Manhattan, Blue Yonder, SAP EWM), with custom-quote-only pricing that creates a barrier to evaluation.
- Mid-market SaaS WMS pricing typically runs **$33K–$300K per year**, still out of reach for many operators (CPC Group, 2026).
- SMB tools such as Fishbowl ($329/month) lack advanced wave picking, labour engineering, slotting, and automation integration.
- Implementation timelines for SAP EWM commonly run **12–24 months**, and Blue Yonder/Manhattan deployments require significant professional services investment.
- The only notable open-source alternative, OpenWMS.org (Apache 2.0), has narrow functional depth, no native EDI or GS1 label printing, and limited commercial support — leaving the open-source landscape effectively unserved.

---

## Key Features

### Inbound and Inventory

- Inbound receiving with ASN (EDI 856) support
- Directed put-away with bin-level location tracking
- Lot and serial number tracking with FIFO / FEFO / LIFO rotation rules
- Cycle counting and inventory reconciliation
- Multi-location and multi-site inventory management

### Outbound and Fulfilment

- Pick, pack, and ship workflows with barcode and RF scanning
- Wave and waveless order picking with task interleaving
- Returns processing with disposition workflows
- GS1 SSCC / GTIN label generation
- EDI 940 / 945 shipping order and shipping advice support

### AI-Driven Optimisation

- AI-driven directed put-away that reasons about size, weight, temperature zone, demand pattern, and co-pick affinity
- AI-powered slotting optimisation with continuous velocity-based recommendations
- AI-optimised wave planning and task interleaving
- Predictive cycle-count prioritisation ranking locations by transaction frequency, discrepancy history, and value
- Predictive labour scheduling forecasting inbound and outbound volumes by hour

### Operations and Insight

- Natural-language warehouse performance query interface for DC managers
- Labour tracking and productivity reporting
- Role-based access controls and audit trails
- Real-time operational dashboards and KPIs

### Extended Capabilities (Backlog)

- Yard management and dock appointment scheduling
- Embedded Warehouse Execution System (WES) for automation coordination (AMR, AS/RS, conveyor, sortation)
- 3PL multi-client billing and onboarding
- Native omnichannel OMS integration (ship-from-store, BOPIS)
- FDA FSMA 204 and DSCSA compliance traceability

---

## AI-Native Advantage

Incumbent WMS suites largely treat slotting, labour planning, and put-away as periodic or rule-based exercises. This project treats them as continuous AI-driven workloads: SKUs are re-slotted as velocity changes, staffing plans are forecast hour-by-hour from inbound and outbound signals, and put-away locations are recommended from multi-factor reasoning rather than static zone rules. A natural-language assistant lets DC managers ask operational questions ("today's pick rate by zone", "locations with the most putaway errors") without building custom reports.

---

## Tech Stack and Deployment

The project is intended for cloud-native deployment using a microservices architecture, drawing on the Twelve-Factor patterns demonstrated by OpenWMS.org. Expected integration surface includes REST APIs with OAuth 2.0, EDI connectivity for ASN (856) and shipping transactions (940 / 945), GS1 standards (SSCC, GTIN, EPCIS) for identification and labelling, and pluggable connectors for ERP systems (SAP, Oracle, Microsoft) and automation hardware (RF, voice, AMR, AS/RS, conveyors). Mobile handheld support is a first-class requirement for floor operations.

---

## Market Context

The global WMS market is estimated at **USD 3.88–4.77 billion in 2025/2026**, growing at a **17–22% CAGR** to reach **USD 10–16 billion by 2031–2033**, with North America holding ~35% share and Asia-Pacific the fastest-growing region (Grand View Research, Mordor Intelligence, MarketsandMarkets). Cloud WMS is growing at a 22.6% CAGR as on-premise licences migrate. Primary buyers are VPs of Supply Chain or Distribution at retailers, 3PLs, and manufacturers; DC managers; IT directors managing WMS / ERP integration; and e-commerce operations leads scaling fulfilment.

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
