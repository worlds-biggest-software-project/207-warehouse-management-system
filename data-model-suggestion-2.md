# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Warehouse Management System · Created: 2026-05-20

## Philosophy

This model treats every warehouse operation as an immutable event appended to an event store. The event store is the single source of truth. Current inventory positions, order statuses, and task states are materialized views (projections) derived by replaying events. The Command side validates and writes events; the Query side reads from denormalized projections optimized for specific access patterns.

This approach is inspired by CQRS (Command Query Responsibility Segregation) and event sourcing patterns used in financial ledger systems, blockchain-style audit trails, and high-throughput logistics platforms. Every state change — receipt of goods, pick confirmation, inventory adjustment, location transfer — is captured as a timestamped, immutable fact. You can always answer "what was the inventory position at location X at 3:47 PM last Tuesday?" by replaying events up to that point.

For a WMS, this is particularly powerful because warehouses are fundamentally event-driven: goods arrive, move, get counted, get picked, get shipped. The traditional approach of mutating inventory rows loses the history. Event sourcing preserves it natively, which is critical for FDA FSMA 204 and DSCSA traceability compliance, and provides a rich dataset for AI model training.

**Best for:** Operations requiring full temporal audit trails, regulatory traceability (pharma, food), AI training on historical movement patterns, and environments where "what happened and when?" is as important as "what is the current state?"

**Trade-offs:**
- Pro: Complete, immutable audit trail built into the architecture — not bolted on
- Pro: Temporal queries are trivial (reconstruct state at any point in time)
- Pro: Natural fit for EPCIS 2.0 event model alignment
- Pro: Rich event stream for AI/ML training on warehouse operation patterns
- Pro: Independently scalable read and write paths
- Con: Higher complexity — developers must understand event sourcing and projections
- Con: Eventual consistency between event store and read models (typically milliseconds, but not zero)
- Con: Schema evolution of events requires careful versioning (upcasting)
- Con: Simple queries ("current stock at location X") require reading a projection rather than a single table
- Con: Storage grows without bound; requires archival strategy for old events

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 EPCIS 2.0 | Event store events map directly to EPCIS ObjectEvent, AggregationEvent, TransactionEvent types — the event store IS the EPCIS repository |
| GS1 GTIN / SSCC / GLN | Stored as identifiers within event payloads and reference data tables |
| ANSI X12 856 | ASN receipt triggers `GoodsReceived` events with ASN reference data |
| ANSI X12 940/945 | Warehouse orders and shipping advices map to command/event pairs |
| FDA FSMA 204 | Critical Tracking Events (CTEs) are native events in the store — no separate compliance logging needed |
| DSCSA | Serialized drug events (receipt, storage, pick, dispatch) are first-class events |
| ISO 8601 | All event timestamps use TIMESTAMPTZ; event versioning uses monotonic sequence numbers |
| OAuth 2.0 / OIDC | Reference data tables for user identity; `actor_id` on every event |

---

## Event Store (Core)

The event store is the heart of the system. All state changes flow through it.

```sql
-- The immutable event store. Every warehouse operation produces one or more events.
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                             -- aggregate root ID (e.g., inventory_item_id, order_id)
    stream_type     VARCHAR(50) NOT NULL,                      -- InventoryItem, InboundOrder, OutboundOrder, PickTask, etc.
    event_type      VARCHAR(80) NOT NULL,                      -- GoodsReceived, PutawayCompleted, PickConfirmed, etc.
    event_version   INTEGER NOT NULL,                          -- monotonic version within stream (optimistic concurrency)
    event_data      JSONB NOT NULL,                            -- event payload
    metadata        JSONB NOT NULL DEFAULT '{}',               -- correlation_id, causation_id, actor info
    actor_id        UUID NOT NULL,                             -- user who caused the event
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user',       -- user, system, ai_engine
    warehouse_id    UUID NOT NULL,                             -- partition key
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),        -- when the real-world event happened
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),        -- when it was written to the store
    UNIQUE(stream_id, event_version)                           -- optimistic concurrency control
);

-- Partitioned by month for storage management
-- In production, use declarative partitioning: PARTITION BY RANGE (recorded_at)

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_event_warehouse_time ON event_store(warehouse_id, occurred_at);
CREATE INDEX idx_event_actor ON event_store(actor_id, occurred_at);
CREATE INDEX idx_event_data_gin ON event_store USING GIN (event_data);
```

### Event Type Taxonomy

```
-- Inbound events
GoodsExpected            -- ASN received / PO created
GoodsArrived             -- trailer checked in at dock
GoodsReceived            -- items scanned and received at location
ReceiptQualityPassed     -- QC inspection passed
ReceiptQualityFailed     -- QC inspection failed
PutawayTaskCreated       -- AI or rule engine creates putaway task
PutawayTaskAssigned      -- task assigned to operator
PutawayCompleted         -- items moved to storage location

-- Inventory events
InventoryAdjusted        -- manual adjustment (damage, shrinkage)
InventoryTransferred     -- location-to-location move
InventoryHeld            -- placed on hold (quarantine)
InventoryReleased        -- released from hold
LotExpiryApproaching     -- system-generated alert event

-- Outbound events
OrderReceived            -- new outbound order created
OrderAllocated           -- inventory reserved for order
WavePlanned              -- orders grouped into wave
WaveReleased             -- wave released for picking
PickTaskCreated          -- individual pick task generated
PickTaskAssigned         -- pick assigned to operator
PickConfirmed            -- item picked and scanned
PickShortReported        -- short pick recorded
PackTaskCreated          -- pack task generated
PackCompleted            -- items packed into container
ShipmentLoaded           -- container loaded onto trailer
ShipmentDispatched       -- trailer departed dock

-- Cycle count events
CycleCountPlanned        -- count session created
CycleCountTaskAssigned   -- location assigned to counter
CycleCountRecorded       -- physical count submitted
CycleCountVarianceResolved -- variance investigated and resolved

-- Slotting / AI events
SlottingRecommendation   -- AI suggests new slot for product
SlottingAccepted         -- operator accepts recommendation
SlottingRejected         -- operator rejects recommendation
PutawayAISuggestion      -- AI suggests putaway location
LabourForecastGenerated  -- AI generates staffing forecast
```

### Example Event Payloads

```json
-- GoodsReceived event_data:
{
  "inbound_order_id": "abc-123",
  "line_number": 1,
  "product_sku": "WIDGET-001",
  "product_gtin": "00012345678905",
  "quantity": 100,
  "uom": "EA",
  "lot_number": "LOT-2026-0520",
  "expiry_date": "2027-05-20",
  "location_barcode": "RCV-DOCK-01",
  "lpn": "LPN-00045678",
  "asn_number": "ASN-2026-1234",
  "supplier_code": "SUP-ACME"
}

-- PickConfirmed event_data:
{
  "outbound_order_id": "ORD-789",
  "pick_task_id": "pt-456",
  "product_sku": "WIDGET-001",
  "from_location": "A3-04-02-B",
  "quantity_requested": 5,
  "quantity_picked": 5,
  "lot_number": "LOT-2026-0520",
  "serial_numbers": ["SN001", "SN002", "SN003", "SN004", "SN005"],
  "wave_id": "WAVE-2026-0520-003"
}

-- SlottingRecommendation event_data:
{
  "product_sku": "WIDGET-001",
  "current_location": "C2-08-03-A",
  "recommended_location": "A1-02-01-B",
  "reason": "velocity_increase",
  "avg_daily_picks_30d": 45.2,
  "velocity_class_current": "B",
  "velocity_class_new": "A",
  "model_version": "slot-v3.2",
  "confidence": 0.92
}
```

## Reference Data Tables (Mutable)

These are traditional relational tables for slowly-changing reference data that does not flow through the event store.

```sql
CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    gln             VARCHAR(13),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    code            VARCHAR(20) NOT NULL,
    name            VARCHAR(100) NOT NULL,
    zone_type       VARCHAR(30) NOT NULL,
    temperature_min NUMERIC(5,2),
    temperature_max NUMERIC(5,2),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, code)
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    zone_id         UUID NOT NULL REFERENCES zone(id),
    barcode         VARCHAR(50) NOT NULL,
    location_type   VARCHAR(30) NOT NULL,
    level           INTEGER,
    position        INTEGER,
    max_weight_kg   NUMERIC(10,2),
    max_volume_m3   NUMERIC(10,4),
    is_pickable     BOOLEAN NOT NULL DEFAULT TRUE,
    is_receivable   BOOLEAN NOT NULL DEFAULT TRUE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, barcode)
);

CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku             VARCHAR(50) NOT NULL UNIQUE,
    gtin            VARCHAR(14),
    name            VARCHAR(300) NOT NULL,
    uom_base        VARCHAR(10) NOT NULL DEFAULT 'EA',
    weight_kg       NUMERIC(10,4),
    length_cm       NUMERIC(10,2),
    width_cm        NUMERIC(10,2),
    height_cm       NUMERIC(10,2),
    is_lot_tracked  BOOLEAN NOT NULL DEFAULT FALSE,
    is_serial_tracked BOOLEAN NOT NULL DEFAULT FALSE,
    is_expiry_tracked BOOLEAN NOT NULL DEFAULT FALSE,
    rotation_rule   VARCHAR(10) NOT NULL DEFAULT 'FIFO',
    temperature_zone VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE trading_partner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(30) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    partner_type    VARCHAR(20) NOT NULL,
    gln             VARCHAR(13),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(50) NOT NULL UNIQUE,
    email           VARCHAR(200) NOT NULL UNIQUE,
    full_name       VARCHAR(200) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Read Model Projections (Materialized from Events)

These tables are rebuilt from the event store. They can be dropped and reconstructed at any time. They are the Query side of CQRS.

```sql
-- Current inventory position: the most-queried projection
CREATE TABLE proj_inventory_position (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL,
    location_id     UUID NOT NULL,
    product_id      UUID NOT NULL,
    lot_number      VARCHAR(50),
    serial_number   VARCHAR(100),
    lpn             VARCHAR(50),
    quantity_on_hand NUMERIC(12,4) NOT NULL DEFAULT 0,
    quantity_allocated NUMERIC(12,4) NOT NULL DEFAULT 0,
    quantity_available NUMERIC(12,4) NOT NULL DEFAULT 0,
    uom             VARCHAR(10) NOT NULL DEFAULT 'EA',
    status          VARCHAR(20) NOT NULL DEFAULT 'available',
    expiry_date     DATE,
    last_event_id   UUID NOT NULL,                             -- last event that updated this projection
    last_event_at   TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()          -- when the projection was last updated
);

CREATE INDEX idx_proj_inv_warehouse_product ON proj_inventory_position(warehouse_id, product_id);
CREATE INDEX idx_proj_inv_location ON proj_inventory_position(location_id);
CREATE INDEX idx_proj_inv_lot ON proj_inventory_position(lot_number) WHERE lot_number IS NOT NULL;
CREATE INDEX idx_proj_inv_expiry ON proj_inventory_position(expiry_date) WHERE expiry_date IS NOT NULL;

-- Current order status projection
CREATE TABLE proj_outbound_order (
    id              UUID PRIMARY KEY,                          -- same as the order's stream_id
    warehouse_id    UUID NOT NULL,
    order_number    VARCHAR(50) NOT NULL,
    customer_code   VARCHAR(30),
    status          VARCHAR(20) NOT NULL,
    total_lines     INTEGER NOT NULL DEFAULT 0,
    lines_allocated INTEGER NOT NULL DEFAULT 0,
    lines_picked    INTEGER NOT NULL DEFAULT 0,
    lines_packed    INTEGER NOT NULL DEFAULT 0,
    lines_shipped   INTEGER NOT NULL DEFAULT 0,
    priority        INTEGER NOT NULL DEFAULT 50,
    required_date   DATE,
    wave_id         UUID,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, order_number)
);

-- Current task board projection (for floor operations screens)
CREATE TABLE proj_active_tasks (
    id              UUID PRIMARY KEY,                          -- task stream_id
    warehouse_id    UUID NOT NULL,
    task_type       VARCHAR(20) NOT NULL,                      -- pick, putaway, pack, count, replenish
    status          VARCHAR(20) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 50,
    assigned_to     UUID,
    product_sku     VARCHAR(50),
    from_location   VARCHAR(50),
    to_location     VARCHAR(50),
    quantity        NUMERIC(12,4),
    wave_id         UUID,
    order_number    VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_proj_tasks_status ON proj_active_tasks(warehouse_id, status)
    WHERE status IN ('pending', 'assigned', 'in_progress');
CREATE INDEX idx_proj_tasks_assigned ON proj_active_tasks(assigned_to)
    WHERE assigned_to IS NOT NULL;

-- Labour productivity projection (for dashboards and AI training)
CREATE TABLE proj_labour_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL,
    user_id         UUID NOT NULL,
    metric_date     DATE NOT NULL,
    metric_hour     INTEGER NOT NULL,                          -- 0-23
    task_type       VARCHAR(20) NOT NULL,
    tasks_completed INTEGER NOT NULL DEFAULT 0,
    units_processed NUMERIC(12,4) NOT NULL DEFAULT 0,
    avg_seconds_per_task NUMERIC(10,2),
    zone_id         UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, user_id, metric_date, metric_hour, task_type)
);

-- Slotting velocity projection (for AI slotting engine)
CREATE TABLE proj_product_velocity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL,
    product_id      UUID NOT NULL,
    velocity_class  CHAR(1),                                   -- A, B, C, D
    picks_7d        INTEGER NOT NULL DEFAULT 0,
    picks_30d       INTEGER NOT NULL DEFAULT 0,
    picks_90d       INTEGER NOT NULL DEFAULT 0,
    units_7d        NUMERIC(12,2) NOT NULL DEFAULT 0,
    units_30d       NUMERIC(12,2) NOT NULL DEFAULT 0,
    units_90d       NUMERIC(12,2) NOT NULL DEFAULT 0,
    current_location_id UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, product_id)
);

-- EPCIS compliance projection (for regulatory queries)
CREATE TABLE proj_epcis_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    epc_uri         VARCHAR(200) NOT NULL,                     -- urn:epc:id:sgtin:... or urn:epc:id:sscc:...
    event_id        UUID NOT NULL,
    event_type      VARCHAR(30) NOT NULL,
    event_time      TIMESTAMPTZ NOT NULL,
    action          VARCHAR(10) NOT NULL,
    biz_step        VARCHAR(100),
    disposition     VARCHAR(100),
    read_point_gln  VARCHAR(13),
    biz_location_gln VARCHAR(13),
    product_sku     VARCHAR(50),
    lot_number      VARCHAR(50),
    serial_number   VARCHAR(100)
);

CREATE INDEX idx_proj_epcis_epc ON proj_epcis_timeline(epc_uri, event_time);
CREATE INDEX idx_proj_epcis_product ON proj_epcis_timeline(product_sku, event_time);
CREATE INDEX idx_proj_epcis_lot ON proj_epcis_timeline(lot_number, event_time)
    WHERE lot_number IS NOT NULL;
```

## Projection Checkpoint Tracking

```sql
-- Tracks where each projection has consumed events up to, enabling replay
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(80) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Example Queries

### Reconstruct inventory at a past point in time

```sql
-- "What was the stock of WIDGET-001 at location A3-04-02-B at 3:00 PM yesterday?"
WITH events_up_to AS (
    SELECT
        e.event_type,
        (e.event_data->>'quantity')::NUMERIC AS qty,
        e.event_data->>'product_sku' AS sku,
        e.event_data->>'location_barcode' AS loc,
        e.event_data->>'from_location' AS from_loc
    FROM event_store e
    WHERE e.warehouse_id = 'warehouse-uuid'
      AND e.occurred_at <= '2026-05-19 15:00:00+00'
      AND (
          (e.event_data->>'product_sku' = 'WIDGET-001' AND e.event_data->>'location_barcode' = 'A3-04-02-B')
          OR (e.event_data->>'product_sku' = 'WIDGET-001' AND e.event_data->>'from_location' = 'A3-04-02-B')
      )
)
SELECT
    SUM(CASE
        WHEN event_type IN ('GoodsReceived', 'PutawayCompleted', 'InventoryAdjusted') THEN qty
        WHEN event_type IN ('PickConfirmed', 'InventoryTransferred') THEN -qty
        ELSE 0
    END) AS quantity_at_time
FROM events_up_to;
```

### Trace full chain of custody for a serial number (DSCSA compliance)

```sql
SELECT
    event_type,
    occurred_at,
    event_data->>'product_sku' AS product,
    event_data->>'location_barcode' AS location,
    event_data->>'lot_number' AS lot,
    actor_id
FROM event_store
WHERE event_data @> '{"serial_numbers": ["SN001"]}'
   OR event_data->>'serial_number' = 'SN001'
ORDER BY occurred_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single immutable event_store table (partitioned by month) |
| Reference Data | 6 | warehouse, zone, location, product, trading_partner, user_account |
| Read Projections | 6 | inventory position, orders, tasks, labour metrics, velocity, EPCIS timeline |
| Infrastructure | 1 | projection_checkpoint |
| **Total** | **14** | Far fewer tables; complexity is in event types and projections |

---

## Key Design Decisions

1. **Single event_store table** — all events for all aggregate types live in one partitioned table. The `stream_type` and `event_type` columns enable filtering. This is simpler to operate than per-aggregate event tables and enables cross-cutting queries ("show me all events in warehouse X in the last hour").

2. **Optimistic concurrency via (stream_id, event_version) uniqueness** — when two processes try to append event version 5 to the same stream, one wins and the other gets a unique constraint violation. This replaces row-level locking on mutable inventory tables.

3. **JSONB event_data rather than typed columns** — event payloads vary by event type. JSONB provides schema-on-read flexibility while GIN indexes enable containment queries (`event_data @> '{"serial_numbers": ["SN001"]}'`).

4. **Projections are disposable** — every `proj_*` table can be dropped and rebuilt by replaying the event store. This makes schema changes to read models trivial: add a column, rebuild the projection.

5. **EPCIS alignment is architectural, not bolted on** — the event store IS the EPCIS event repository. `GoodsReceived` maps to an EPCIS ObjectEvent with bizStep=`receiving`. Exporting EPCIS 2.0 JSON-LD is a projection query, not a separate compliance system.

6. **Reference data stays relational** — warehouses, zones, locations, and products change infrequently and need referential integrity. They remain as normal tables. Only state that changes through operational events (inventory, orders, tasks) flows through the event store.

7. **AI training benefits** — the event store is a natural training dataset. Historical pick patterns, putaway decisions, velocity changes, and slotting outcomes are all timestamped events that can be replayed into ML pipelines without building separate ETL.

8. **Archival strategy** — event store partitions older than a configurable retention period (e.g., 3 years) can be detached and moved to cold storage (S3/GCS) while projections retain current state. For compliance domains requiring longer retention, partitions can be compressed but kept queryable.
