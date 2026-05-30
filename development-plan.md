# Warehouse Management System — Phased Development Plan

> Project: 207-warehouse-management-system · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Java 21 (Spring Boot 3.3) | Enterprise WMS domain aligns with Java ecosystem; OpenWMS.org (Apache 2.0) is Java/Spring-based; GS1 libraries, EDI parsers, and barcode generation libraries are strongest in Java; Spring Boot provides production-grade REST APIs, scheduling, and transaction management |
| API framework | Spring Boot 3.3 + Spring Web | OpenAPI 3.1 generation via springdoc-openapi; OAuth 2.0 via Spring Security; robust transaction management critical for inventory accuracy |
| Database | PostgreSQL 16 | Normalised schema from data-model-suggestion-1 (~50 tables); strong transactional guarantees for concurrent inventory operations; row-level locking for pick/put-away |
| Migrations | Flyway | Version-controlled SQL migrations; standard in Spring Boot projects |
| ORM | Spring Data JPA (Hibernate 6) | Mature JPA implementation; handles complex WMS entity relationships; batch operations for bulk receiving |
| Task queue | Spring Scheduler + Redis (Spring Data Redis) | Scheduled slotting analysis, cycle count generation, wave planning, EDI polling |
| Message broker | Apache Kafka (or Redis Streams for MVP) | Real-time inventory events, pick task distribution, automation integration; Kafka for production scale |
| Barcode / Labels | ZXing (barcode generation) + JasperReports (label layout) | GS1 SSCC/GTIN barcode generation; configurable label templates for GS1-128 shipping labels |
| EDI processing | Smooks or custom X12 parser | Parse/generate ANSI X12 856 (ASN), 940 (Warehouse Order), 945 (Shipping Advice) |
| ML / AI | Python microservice (FastAPI + scikit-learn) | Slotting optimisation, labour forecasting, cycle count prioritisation — Python ML ecosystem; called via REST from Java backend |
| LLM integration | Anthropic Java SDK or REST API | Natural-language warehouse performance queries |
| Frontend | Next.js 15 (App Router) + Tailwind CSS | Operations dashboard, wave management, inventory views; responsive for tablet use |
| Mobile / RF | React Native (Expo) or PWA | Handheld RF scanner app for pick/pack/putaway; camera-based barcode scanning |
| Containerisation | Docker + docker-compose | PostgreSQL, Redis/Kafka, Spring Boot API, Python ML service, Next.js frontend |
| Testing | JUnit 5 + Mockito + Testcontainers | Unit (JUnit), integration (Testcontainers with real PostgreSQL), API (MockMvc) |
| Linting | Checkstyle + SpotBugs | Java code quality |
| Type checking | Java compiler (strongly typed) | Compile-time type safety |
| Build tool | Gradle (Kotlin DSL) | Fast builds; multi-module support |

### Project Structure

```
warehouse-management/
├── build.gradle.kts
├── settings.gradle.kts
├── docker-compose.yml
├── Dockerfile
├── modules/
│   ├── core/                        # Domain models, repositories, shared services
│   │   └── src/main/java/com/wms/core/
│   │       ├── model/               # JPA entities (from data-model-suggestion-1)
│   │       │   ├── Warehouse.java
│   │       │   ├── Zone.java
│   │       │   ├── Location.java
│   │       │   ├── Product.java
│   │       │   ├── Inventory.java
│   │       │   ├── InboundReceipt.java
│   │       │   ├── OutboundOrder.java
│   │       │   ├── PickTask.java
│   │       │   ├── ShipmentContainer.java
│   │       │   └── CycleCount.java
│   │       ├── repository/
│   │       └── service/
│   ├── api/                         # REST API layer
│   │   └── src/main/java/com/wms/api/
│   │       ├── controller/
│   │       │   ├── InventoryController.java
│   │       │   ├── ReceivingController.java
│   │       │   ├── PickingController.java
│   │       │   ├── ShippingController.java
│   │       │   ├── WaveController.java
│   │       │   ├── CycleCountController.java
│   │       │   └── ReportsController.java
│   │       ├── dto/
│   │       └── config/
│   ├── edi/                         # EDI X12 processing
│   │   └── src/main/java/com/wms/edi/
│   │       ├── parser/
│   │       │   ├── X12_856Parser.java
│   │       │   ├── X12_940Parser.java
│   │       │   └── X12_945Generator.java
│   │       └── service/
│   ├── labels/                      # GS1 barcode and label generation
│   │   └── src/main/java/com/wms/labels/
│   └── ml/                          # Python ML microservice
│       ├── pyproject.toml
│       └── src/
│           ├── main.py              # FastAPI
│           ├── slotting.py
│           ├── labour_forecast.py
│           └── cycle_count_priority.py
├── frontend/
│   ├── package.json
│   └── src/app/
│       ├── dashboard/
│       ├── receiving/
│       ├── inventory/
│       ├── picking/
│       ├── shipping/
│       ├── wave-planning/
│       ├── cycle-counts/
│       └── settings/
├── mobile/                          # React Native RF scanner app
├── src/main/resources/
│   ├── db/migration/                # Flyway migrations
│   ├── labels/                      # JasperReports label templates
│   └── application.yml
└── src/test/
    ├── unit/
    ├── integration/
    └── fixtures/
        ├── edi_856_sample.txt
        ├── edi_940_sample.txt
        └── sample_products.json
```

---

## Phase 1: Foundation

### Purpose
Establish the project skeleton, database schema from data-model-suggestion-1 (warehouses, zones, locations, products, inventory), authentication, and Docker environment. After this phase, users can configure a warehouse with zone/aisle/bin locations and register products.

### Tasks

#### 1.1 — Project Scaffold and Configuration

**What**: Create Gradle multi-module project, Spring Boot app, configuration, Docker with PostgreSQL and Redis.

**Design**:

```yaml
# application.yml
wms:
  database:
    url: jdbc:postgresql://localhost:5432/wms
  redis:
    url: redis://localhost:6379
  edi:
    inbound-dir: /data/edi/inbound
    outbound-dir: /data/edi/outbound
  labels:
    printer-type: zebra  # zebra, brother, pdf
  ml-service:
    url: http://localhost:8001
```

**Testing**:
- Unit: application context loads with all beans
- Integration: `docker-compose up -d` → PostgreSQL, Redis, API healthy

#### 1.2 — Database Schema — Warehouse and Inventory Tables

**What**: Implement warehouse, zone, aisle, location, product, inventory, lot, and serial_number tables from data-model-suggestion-1.

**Design**:

```java
@Entity
@Table(name = "location")
public class Location {
    @Id @GeneratedValue private UUID id;
    @ManyToOne @JoinColumn(name = "zone_id") private Zone zone;
    @Column(nullable = false) private String locationCode;  // e.g., "A-03-02-B"
    @Enumerated(EnumType.STRING) private LocationType locationType; // PICK, BULK, STAGING, RECEIVING, SHIPPING
    private Integer maxWeight;
    private Integer maxVolumeCubicInches;
    @Enumerated(EnumType.STRING) private TemperatureZone temperatureZone; // AMBIENT, REFRIGERATED, FROZEN
    private boolean isActive = true;
}

@Entity
@Table(name = "inventory")
public class Inventory {
    @Id @GeneratedValue private UUID id;
    @ManyToOne @JoinColumn(name = "product_id") private Product product;
    @ManyToOne @JoinColumn(name = "location_id") private Location location;
    @Column(nullable = false) private Integer quantity;
    @Column(nullable = false) private Integer allocatedQuantity = 0;
    private String lotNumber;
    private LocalDate expirationDate;
    @Enumerated(EnumType.STRING) private InventoryStatus status; // AVAILABLE, ALLOCATED, HELD, DAMAGED
    @Column(nullable = false) private String uom; // EA, CS, PL
}
```

Location hierarchy: warehouse → zone → aisle → level → position (composite locationCode). GS1 fields: product.gtin (14-digit), warehouse.gln (13-digit).

**Testing**:
- Integration: Flyway migrations apply → all tables created
- Integration: create warehouse → zone → 100 locations → inventory records → FK chain holds
- Unit: locationCode format validated (pattern: `[A-Z]-\\d{2}-\\d{2}-[A-Z]`)

#### 1.3 — Authentication and Roles

**What**: Spring Security with OAuth 2.0; WMS-specific roles: admin, warehouse_manager, supervisor, picker, receiver, shipping_clerk, api_service.

**Design**:

Roles map to WMS functions: `picker` (pick tasks, confirm picks), `receiver` (inbound receipts, putaway), `shipping_clerk` (pack, ship, label generation), `supervisor` (wave planning, cycle counts, reports), `warehouse_manager` (full access), `admin` (system config).

**Testing**:
- Unit: picker can confirm pick → allowed
- Unit: picker cannot modify wave plan → 403
- Unit: API service can read inventory → allowed

---

## Phase 2: Inbound Receiving and Put-Away

### Purpose
Implement the receiving workflow: ASN processing, goods receipt, quality inspection, and directed put-away. This is the entry point for all warehouse inventory.

### Tasks

#### 2.1 — ASN Processing and Receiving

**What**: Receive inbound shipments against ASN (EDI 856 or manual entry); create receipt records and add inventory.

**Design**:

```java
// POST /api/v1/receiving/receipts
public record ReceiptCreateRequest(
    String poNumber,
    UUID supplierId,
    List<ReceiptLineRequest> lines
) {}

public record ReceiptLineRequest(
    UUID productId,
    int expectedQuantity,
    String lotNumber,        // optional
    LocalDate expirationDate // optional
) {}

// Receipt state: EXPECTED → RECEIVING → RECEIVED → PUT_AWAY
```

ASN (EDI 856) creates an expected receipt with line items. Receiver scans barcodes to confirm actual quantities received. Over/short/damage variances recorded per line.

**Testing**:
- Unit: ASN with 3 lines → receipt with 3 expected lines
- Unit: receive 100 of 100 expected → status = RECEIVED, no variance
- Unit: receive 95 of 100 → short variance recorded
- Integration: EDI 856 parsed → receipt auto-created with correct product/qty mapping

#### 2.2 — Directed Put-Away

**What**: System suggests optimal bin location for received goods; receiver confirms or overrides.

**Design**:

```java
// PUT-AWAY LOGIC:
// 1. Check product's designated zone (ambient/refrigerated/frozen)
// 2. Filter locations by zone + available capacity (weight, volume)
// 3. Prefer locations in product's pick zone (reduce future travel)
// 4. Apply rotation rules: FIFO (earliest receipt first), FEFO (earliest expiry first)
// 5. For AI-enhanced put-away (Phase 7): call ML service for multi-factor recommendation

public record PutawayDirective(
    UUID inventoryId,
    UUID suggestedLocationId,
    String locationCode,
    String reason  // "Zone match + pick zone proximity"
) {}

// POST /api/v1/receiving/putaway/{receiptLineId}/confirm
// Request: { "locationId": "uuid" }  -- can override suggestion
```

**Testing**:
- Unit: frozen product → only frozen zone locations suggested
- Unit: no capacity in pick zone → bulk zone suggested with reason
- Unit: receiver overrides suggestion → inventory placed at override location
- Integration: receive → putaway directive generated → confirm → inventory visible at location

---

## Phase 3: Outbound Picking and Packing

### Purpose
Implement pick, pack, and ship workflows — the core outbound flow. After this phase, orders can be processed through to shipment.

### Tasks

#### 3.1 — Order Import and Pick Task Generation

**What**: Import outbound orders and generate pick tasks with location-directed picking.

**Design**:

```java
// POST /api/v1/orders (or via EDI 940 import)
public record OrderCreateRequest(
    String orderNumber,
    UUID customerId,
    String shipToAddress,
    List<OrderLineRequest> lines
) {}

// Order state: IMPORTED → ALLOCATED → PICKING → PICKED → PACKING → PACKED → SHIPPED
// Allocation: reserve inventory for order lines (decrement available, increment allocated)
// Pick task generation: one task per order line per location
```

Allocation logic: FIFO by receipt date (or FEFO by expiration for perishables). If insufficient inventory → partial allocation with backorder flag.

Pick task includes: location code, product, quantity, lot (if applicable). Tasks sorted by pick path (location sequence within zone) to minimise travel.

**Testing**:
- Unit: order for 50 units, 50 available → fully allocated
- Unit: order for 50 units, 30 available → partial allocation, 20 backordered
- Unit: FEFO enabled, 2 lots (exp 2026-06, exp 2026-09) → earlier lot picked first
- Integration: import order → allocate → pick tasks generated with correct locations

#### 3.2 — Pick Confirmation and Pack/Ship

**What**: Picker confirms picks via RF scan; packer packs into containers; shipping generates labels and EDI 945.

**Design**:

```java
// POST /api/v1/picking/tasks/{taskId}/confirm
public record PickConfirmRequest(
    UUID locationId,       // scanner confirms correct location
    UUID productId,        // scanner confirms correct product
    int quantityPicked
) {}

// Short pick: quantityPicked < expected → system reallocates or flags exception
// Pack: group picked items into shipping containers
// POST /api/v1/shipping/containers
public record ContainerCreateRequest(
    UUID orderId,
    String containerType,  // CARTON, PALLET, TOTE
    List<UUID> pickTaskIds // picks going into this container
) {}

// Ship: generate GS1 SSCC label, update order status, generate EDI 945
// POST /api/v1/shipping/containers/{id}/ship
```

**Testing**:
- Unit: confirm pick with correct location/product → task completed
- Unit: wrong product scanned → error "Product mismatch"
- Unit: short pick → remaining quantity reallocated to another location
- Unit: pack into container → SSCC generated (18-digit GS1)
- Unit: ship → order status = SHIPPED, EDI 945 generated
- Integration: full flow: order → allocate → pick → pack → ship → inventory decremented

---

## Phase 4: Wave Planning

### Purpose
Group orders into waves for efficient batch picking. Wave planning determines which orders are processed together, optimising pick paths and labour allocation.

### Tasks

#### 4.1 — Wave Creation and Release

**What**: Create waves from pending orders based on configurable criteria; release waves to generate pick tasks.

**Design**:

```java
// POST /api/v1/waves
public record WaveCreateRequest(
    String waveName,
    WaveCriteria criteria
) {}

public record WaveCriteria(
    LocalDate shipDate,           // orders shipping today
    String carrier,               // group by carrier
    String zone,                  // group by pick zone
    int maxOrders,                // cap wave size
    String priority               // standard, expedited, next_day
) {}

// Wave state: PLANNED → RELEASED → PICKING → COMPLETE
// Release: generates pick tasks for all orders in the wave
// Task interleaving: combines picks from multiple orders into a single
//   optimised pick path through the warehouse
```

**Testing**:
- Unit: 50 orders for same ship date → wave with 50 orders
- Unit: release wave → pick tasks generated, sorted by pick path
- Unit: task interleaving → picker visits each location once for multiple orders
- E2E: create wave → release → pick tasks visible on RF device

---

## Phase 5: Cycle Counting and Inventory Accuracy

### Purpose
Maintain inventory accuracy through scheduled and ad-hoc cycle counts. Critical for warehouse operational reliability.

### Tasks

#### 5.1 — Cycle Count Generation and Execution

**What**: Generate cycle count tasks by location; counters confirm quantities; discrepancies investigated and adjusted.

**Design**:

```java
// POST /api/v1/cycle-counts/generate
public record CycleCountGenerateRequest(
    String strategy,              // ABC, RANDOM, ZONE, FULL
    String zoneId,                // optional: specific zone
    int locationCount             // number of locations to count
) {}

// ABC strategy: A items (high value/velocity) counted more frequently
// Count state: GENERATED → COUNTING → COUNTED → RECONCILED
// If count ≠ system quantity → DISCREPANCY status, requires supervisor review
// Adjustment creates audit trail record
```

**Testing**:
- Unit: ABC strategy → A-class SKUs selected 3x more than C-class
- Unit: count matches system → status = RECONCILED
- Unit: count differs → DISCREPANCY, adjustment requires supervisor approval
- Unit: adjustment creates audit_log record with before/after quantities

---

## Phase 6: GS1 Labels and EDI Integration

### Purpose
Generate GS1-compliant shipping labels (SSCC, GTIN barcodes) and process EDI transactions (856, 940, 945) for trading partner integration.

### Tasks

#### 6.1 — GS1 Label Generation

**What**: Generate GS1-128 shipping labels with SSCC, GTIN, lot, expiry, and quantity barcodes.

**Design**:

```java
// GS1-128 label contains Application Identifiers:
// (00) SSCC - 18 digits
// (01) GTIN - 14 digits
// (10) Lot Number
// (17) Expiration Date (YYMMDD)
// (37) Quantity

// POST /api/v1/labels/generate
public record LabelRequest(
    UUID containerId,
    String labelFormat  // GS1_128, GS1_DATAMATRIX
) {}
// Response: PDF or ZPL (Zebra Printer Language) depending on printer config
```

ZXing generates barcode images; JasperReports composes the label layout with barcodes + text.

**Testing**:
- Unit: SSCC generated → 18 digits, valid check digit
- Unit: label PDF contains SSCC, GTIN, lot, expiry barcodes
- Unit: ZPL output → valid Zebra printer commands
- Fixture: sample label layout template

#### 6.2 — EDI 856 / 940 / 945 Processing

**What**: Parse inbound EDI 856 (ASN) and 940 (Warehouse Order); generate outbound EDI 945 (Shipping Advice).

**Design**:

```java
// X12 856 ASN → InboundReceipt
// Segments: BSN (begin), HL (hierarchy: shipment/order/pack/item),
//           TD1/TD5 (transport), REF (references), MAN (marks/numbers)

// X12 940 Warehouse Order → OutboundOrder
// Segments: W05 (order header), N1 (ship-to), W06 (order lines), W12 (quantities)

// X12 945 Shipping Advice ← ShippedContainer
// Generated on ship confirmation; segments: W06, W12, W27 (shipped detail)
```

**Testing**:
- Unit: parse EDI 856 → receipt with correct PO, supplier, line items
- Unit: parse EDI 940 → order with correct ship-to, line items
- Unit: generate EDI 945 → valid X12 structure with correct segments
- Fixture: sample EDI files in `tests/fixtures/`

---

## Phase 7: AI-Driven Optimisation (v1.1)

### Purpose
Add the AI differentiators: intelligent slotting, predictive labour scheduling, and cycle count prioritisation. These move the WMS from rule-based to continuously optimised.

### Tasks

#### 7.1 — AI Slotting Optimisation

**What**: ML service recommends optimal bin assignments for SKUs based on velocity, co-pick affinity, weight, and size.

**Design**:

```python
# modules/ml/src/slotting.py (FastAPI)
@dataclass
class SlottingRecommendation:
    product_id: str
    current_location_id: str
    recommended_location_id: str
    reason: str                    # "High velocity (A-class) → forward pick zone"
    estimated_pick_time_savings_pct: float

class SlottingOptimiser:
    def optimise(self, products: list[dict], locations: list[dict],
                 pick_history: list[dict]) -> list[SlottingRecommendation]:
        """
        1. Classify SKUs by velocity (ABC analysis from pick_history)
        2. Identify co-pick affinities (products frequently picked together)
        3. Assign A-class SKUs to ergonomic golden zone locations
        4. Group co-pick SKUs in adjacent locations
        5. Consider weight (heavy items at floor level) and temperature zone
        """
```

Runs weekly via scheduled task; recommendations surfaced in dashboard for supervisor approval.

**Testing**:
- Unit: A-class SKU in bulk zone → recommendation to move to pick zone
- Unit: 2 SKUs frequently co-picked → recommended for adjacent locations
- Unit: heavy product → floor-level location recommended
- Integration: Java backend calls Python ML service → recommendations stored

#### 7.2 — Predictive Labour Forecasting

**What**: Forecast inbound and outbound volumes by hour; generate optimal staffing plans.

**Design**:

```python
class LabourForecaster:
    def forecast(self, warehouse_id: str, forecast_date: date) -> HourlyForecast:
        """
        Features: day_of_week, month, historical_volume_same_day,
                  pending_orders, expected_receipts, weather, promotions
        Model: XGBoost regressor per activity (receiving, picking, packing, shipping)
        Output: predicted_units_per_hour × labour_standard = recommended_headcount
        """

@dataclass
class HourlyForecast:
    hour: int
    receiving_units: int
    picking_units: int
    packing_units: int
    recommended_receivers: int
    recommended_pickers: int
    recommended_packers: int
```

**Testing**:
- Unit: Monday with high historical volume → higher headcount recommended
- Unit: 500 pending orders → picking headcount scales appropriately
- Integration: forecast displayed in dashboard with hourly bar chart

#### 7.3 — Natural-Language Warehouse Query Assistant

**What**: DC managers ask operational questions in plain language.

**Design**:

```python
# POST /api/v1/ai/query
# Request: {"question": "What's the pick rate by zone today?"}
# Response: {"answer": "Zone A: 145 lines/hour, Zone B: 98 lines/hour...", "data": [...]}
```

Claude receives warehouse KPI context (pick rates, receiving volumes, inventory accuracy, labour utilisation) and answers queries.

**Testing**:
- Unit (mocked): "today's pick rate" → returns per-zone picks per hour
- Unit (mocked): "which locations have the most discrepancies?" → returns ranked list

---

## Phase 8: Mobile RF Scanner App

### Purpose
Handheld scanner app for floor operations: receiving confirmation, directed picking, putaway, and cycle counting. The primary interface for warehouse operators.

### Tasks

#### 8.1 — RF Scanner PWA/App

**What**: Mobile app with barcode scanning for pick, receive, putaway, and count workflows.

**Design**:

Screens: Login → Task List (pick tasks sorted by path) → Scan Location → Scan Product → Confirm Quantity → Next Task. Camera-based barcode scanning via `expo-camera` or web `BarcodeDetector` API.

Offline support: cache assigned tasks in local storage; queue confirmations for sync when reconnected.

**Testing**:
- E2E: login → see assigned pick tasks → scan location → scan product → confirm quantity → task completed
- E2E: offline mode → complete 3 picks → reconnect → synced to server
- Unit: wrong location barcode → error "Incorrect location, expected A-03-02-B"

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Inbound Receiving & Put-Away  ─── requires Phase 1
    │
Phase 3: Outbound Picking & Packing    ─── requires Phase 1 (parallel with Phase 2)
    │
Phase 4: Wave Planning                 ─── requires Phase 3
    │
Phase 5: Cycle Counting                ─── requires Phase 1 (parallel with Phases 2-4)
    │
Phase 6: GS1 Labels & EDI             ─── requires Phases 2 + 3
    │
Phase 7: AI Optimisation              ─── requires Phases 2 + 3 + 5
    │
Phase 8: Mobile RF Scanner            ─── requires Phases 2 + 3 + 5
```

Parallelism opportunities:
- Phases 2 and 3 can be developed concurrently after Phase 1
- Phase 5 can be developed concurrently with Phases 2-4
- Phases 7 and 8 can be developed concurrently after Phases 2+3+5

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specification.
2. All unit and integration tests pass (`./gradlew test` with 0 failures).
3. Checkstyle and SpotBugs pass with zero warnings.
4. Java compiles with zero warnings (`-Xlint:all`).
5. Docker build succeeds for API, ML service, and frontend images.
6. `docker-compose up` brings all services to healthy state.
7. Feature works end-to-end (manual or automated test).
8. Inventory accuracy maintained: all pick/put-away/count operations update inventory atomically.
9. New API endpoints appear in auto-generated OpenAPI spec at `/v3/api-docs`.
10. Flyway migrations created and tested (up + down).
11. Role-based permissions enforced on all new endpoints.
12. GS1 identifiers (SSCC, GTIN) validate against check digit algorithms.
13. New configuration properties documented in application.yml.
