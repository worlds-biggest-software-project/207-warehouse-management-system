# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Warehouse Management System · Created: 2026-05-20

## Philosophy

This model follows a fully normalized relational approach where every domain concept gets its own table with strict foreign key constraints and referential integrity. Each entity — warehouses, zones, locations, products, inventory, orders, picks, shipments — is stored in a dedicated table with clearly typed columns, explicit constraints, and well-defined indexes. No JSONB, no event stores, no graph layers.

This is the approach used by traditional enterprise WMS platforms like SAP EWM and Oracle WMS Cloud, where data integrity and transactional consistency are paramount. The schema is designed for complex cross-entity queries (e.g., "show all lots expiring within 7 days in zone A3 with available quantity > 0") without requiring JSON parsing or event replay.

**Best for:** Teams with strong SQL expertise building a compliance-heavy WMS where data integrity, auditability, and complex ad-hoc reporting are the primary concerns.

**Trade-offs:**
- Pro: Maximum data integrity via foreign keys and constraints
- Pro: Excellent for complex analytical queries and reporting
- Pro: Well understood by most developers and DBAs
- Pro: Standards-aligned field names (GS1, EPCIS, ISO)
- Con: Schema changes require migrations — adding a new attribute means ALTER TABLE
- Con: Higher table count increases join complexity
- Con: Jurisdiction-specific or product-type-specific fields lead to sparse columns or additional tables
- Con: Write-heavy operations (high-volume picking) may hit contention on inventory tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | `product.gtin` stores the 14-digit Global Trade Item Number |
| GS1 SSCC | `shipping_container.sscc` stores the 18-digit Serial Shipping Container Code |
| GS1 GLN | `warehouse.gln` and `trading_partner.gln` store Global Location Numbers |
| GS1 EPCIS 2.0 | `epcis_event` table stores visibility events aligned with EPCIS object/aggregation/transaction event types |
| ISO 3166-1/2 | `address.country_code` and `address.subdivision_code` use ISO country and subdivision codes |
| ANSI X12 856 | `asn` table models Advance Ship Notices with segment-aligned fields |
| ANSI X12 940/945 | `warehouse_order` and `shipping_advice` tables model 3PL EDI transactions |
| ISO 8601 | All timestamp fields use TIMESTAMPTZ; duration fields use ISO 8601 interval notation |
| OAuth 2.0 / OIDC | `user_account` and `api_client` tables support token-based authentication |
| ISO 9001 | `quality_inspection` and `non_conformance` tables support quality management workflows |

---

## Warehouse and Location Management

```sql
CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) NOT NULL UNIQUE,          -- internal warehouse code
    name            VARCHAR(200) NOT NULL,
    gln             VARCHAR(13),                          -- GS1 Global Location Number
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE warehouse_address (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    address_type    VARCHAR(20) NOT NULL DEFAULT 'physical', -- physical, mailing
    line_1          VARCHAR(200) NOT NULL,
    line_2          VARCHAR(200),
    city            VARCHAR(100) NOT NULL,
    subdivision_code VARCHAR(6),                          -- ISO 3166-2
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,                     -- ISO 3166-1 alpha-2
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    code            VARCHAR(20) NOT NULL,
    name            VARCHAR(100) NOT NULL,
    zone_type       VARCHAR(30) NOT NULL,                 -- storage, staging, receiving, shipping, returns, quarantine
    temperature_min NUMERIC(5,2),                         -- celsius, for cold-chain
    temperature_max NUMERIC(5,2),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, code)
);

CREATE TABLE aisle (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    zone_id         UUID NOT NULL REFERENCES zone(id),
    code            VARCHAR(10) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(zone_id, code)
);

CREATE TABLE rack (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aisle_id        UUID NOT NULL REFERENCES aisle(id),
    code            VARCHAR(10) NOT NULL,
    max_weight_kg   NUMERIC(10,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(aisle_id, code)
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    zone_id         UUID NOT NULL REFERENCES zone(id),
    aisle_id        UUID REFERENCES aisle(id),
    rack_id         UUID REFERENCES rack(id),
    barcode         VARCHAR(50) NOT NULL,                 -- scannable location barcode
    location_type   VARCHAR(30) NOT NULL,                 -- bin, floor, pallet, bulk, pick_face
    level           INTEGER,                              -- shelf level (1 = ground)
    position        INTEGER,                              -- position within level
    max_weight_kg   NUMERIC(10,2),
    max_volume_m3   NUMERIC(10,4),
    is_pickable     BOOLEAN NOT NULL DEFAULT TRUE,
    is_receivable   BOOLEAN NOT NULL DEFAULT TRUE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, barcode)
);

CREATE INDEX idx_location_zone ON location(zone_id);
CREATE INDEX idx_location_warehouse_active ON location(warehouse_id) WHERE is_active = TRUE;
```

## Product and Item Master

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku             VARCHAR(50) NOT NULL UNIQUE,
    gtin            VARCHAR(14),                          -- GS1 Global Trade Item Number
    name            VARCHAR(300) NOT NULL,
    description     TEXT,
    uom_base        VARCHAR(10) NOT NULL DEFAULT 'EA',    -- base unit of measure (EA, CS, PL)
    weight_kg       NUMERIC(10,4),
    length_cm       NUMERIC(10,2),
    width_cm        NUMERIC(10,2),
    height_cm       NUMERIC(10,2),
    volume_cm3      NUMERIC(12,2),
    is_lot_tracked  BOOLEAN NOT NULL DEFAULT FALSE,
    is_serial_tracked BOOLEAN NOT NULL DEFAULT FALSE,
    is_expiry_tracked BOOLEAN NOT NULL DEFAULT FALSE,
    rotation_rule   VARCHAR(10) NOT NULL DEFAULT 'FIFO',  -- FIFO, FEFO, LIFO
    hazmat_class    VARCHAR(10),                          -- UN hazmat class if applicable
    temperature_zone VARCHAR(20),                         -- ambient, chilled, frozen
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_sku ON product(sku);

CREATE TABLE product_uom_conversion (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id),
    from_uom        VARCHAR(10) NOT NULL,                 -- e.g. CS (case)
    to_uom          VARCHAR(10) NOT NULL,                 -- e.g. EA (each)
    conversion_factor NUMERIC(12,4) NOT NULL,             -- 1 CS = 24 EA => factor = 24
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(product_id, from_uom, to_uom)
);

CREATE TABLE product_barcode (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id),
    barcode_value   VARCHAR(50) NOT NULL,
    barcode_type    VARCHAR(20) NOT NULL,                 -- GS1-128, EAN-13, UPC-A, DataMatrix, QR
    uom             VARCHAR(10) NOT NULL DEFAULT 'EA',
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(barcode_value, barcode_type)
);

CREATE INDEX idx_product_barcode_value ON product_barcode(barcode_value);
```

## Inventory Management

```sql
CREATE TABLE inventory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    lot_number      VARCHAR(50),
    serial_number   VARCHAR(100),
    lpn             VARCHAR(50),                          -- licence plate number
    quantity_on_hand NUMERIC(12,4) NOT NULL DEFAULT 0,
    quantity_allocated NUMERIC(12,4) NOT NULL DEFAULT 0,
    quantity_available NUMERIC(12,4) GENERATED ALWAYS AS (quantity_on_hand - quantity_allocated) STORED,
    uom             VARCHAR(10) NOT NULL DEFAULT 'EA',
    status          VARCHAR(20) NOT NULL DEFAULT 'available', -- available, quarantine, damaged, hold, in_transit
    expiry_date     DATE,
    manufacture_date DATE,
    received_at     TIMESTAMPTZ,
    cost_per_unit   NUMERIC(12,4),
    currency_code   CHAR(3) DEFAULT 'USD',                -- ISO 4217
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_qty_non_negative CHECK (quantity_on_hand >= 0),
    CONSTRAINT chk_allocated_non_negative CHECK (quantity_allocated >= 0),
    CONSTRAINT chk_allocated_lte_on_hand CHECK (quantity_allocated <= quantity_on_hand)
);

CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_location ON inventory(location_id);
CREATE INDEX idx_inventory_warehouse_product ON inventory(warehouse_id, product_id);
CREATE INDEX idx_inventory_lot ON inventory(lot_number) WHERE lot_number IS NOT NULL;
CREATE INDEX idx_inventory_serial ON inventory(serial_number) WHERE serial_number IS NOT NULL;
CREATE INDEX idx_inventory_lpn ON inventory(lpn) WHERE lpn IS NOT NULL;
CREATE INDEX idx_inventory_expiry ON inventory(expiry_date) WHERE expiry_date IS NOT NULL;

CREATE TABLE inventory_adjustment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inventory_id    UUID NOT NULL REFERENCES inventory(id),
    adjustment_type VARCHAR(30) NOT NULL,                 -- receipt, pick, putaway, cycle_count, damage, return, transfer
    quantity_change  NUMERIC(12,4) NOT NULL,              -- positive = increase, negative = decrease
    reason_code     VARCHAR(30),
    reference_type  VARCHAR(30),                          -- inbound_order, outbound_order, cycle_count, manual
    reference_id    UUID,
    performed_by    UUID NOT NULL,                        -- references user_account
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_adjustment_inventory ON inventory_adjustment(inventory_id);
CREATE INDEX idx_adjustment_performed_at ON inventory_adjustment(performed_at);
CREATE INDEX idx_adjustment_reference ON inventory_adjustment(reference_type, reference_id);
```

## Inbound (Receiving and Putaway)

```sql
CREATE TABLE trading_partner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(30) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    partner_type    VARCHAR(20) NOT NULL,                 -- supplier, customer, carrier, 3pl_client
    gln             VARCHAR(13),                          -- GS1 Global Location Number
    edi_qualifier   VARCHAR(4),                           -- EDI interchange qualifier
    edi_id          VARCHAR(30),                          -- EDI interchange ID
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE inbound_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    order_number    VARCHAR(50) NOT NULL,
    supplier_id     UUID REFERENCES trading_partner(id),
    client_id       UUID REFERENCES trading_partner(id), -- 3PL client if applicable
    order_type      VARCHAR(20) NOT NULL,                 -- purchase_order, return, transfer
    status          VARCHAR(20) NOT NULL DEFAULT 'expected', -- expected, in_progress, received, closed, cancelled
    expected_date   DATE,
    asn_number      VARCHAR(50),                          -- EDI 856 ASN reference
    po_number       VARCHAR(50),                          -- purchase order reference
    carrier_code    VARCHAR(20),
    trailer_number  VARCHAR(30),
    dock_door_id    UUID REFERENCES location(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, order_number)
);

CREATE TABLE inbound_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inbound_order_id UUID NOT NULL REFERENCES inbound_order(id),
    line_number     INTEGER NOT NULL,
    product_id      UUID NOT NULL REFERENCES product(id),
    expected_qty    NUMERIC(12,4) NOT NULL,
    received_qty    NUMERIC(12,4) NOT NULL DEFAULT 0,
    uom             VARCHAR(10) NOT NULL DEFAULT 'EA',
    lot_number      VARCHAR(50),
    status          VARCHAR(20) NOT NULL DEFAULT 'expected',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(inbound_order_id, line_number)
);

CREATE TABLE receipt (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inbound_order_id UUID NOT NULL REFERENCES inbound_order(id),
    inbound_line_id UUID NOT NULL REFERENCES inbound_order_line(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),  -- staging/receiving location
    quantity        NUMERIC(12,4) NOT NULL,
    uom             VARCHAR(10) NOT NULL DEFAULT 'EA',
    lot_number      VARCHAR(50),
    serial_number   VARCHAR(100),
    lpn             VARCHAR(50),
    expiry_date     DATE,
    quality_status  VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, passed, failed, waived
    received_by     UUID NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE putaway_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    receipt_id      UUID REFERENCES receipt(id),
    inventory_id    UUID REFERENCES inventory(id),
    from_location_id UUID NOT NULL REFERENCES location(id),
    to_location_id  UUID NOT NULL REFERENCES location(id),  -- AI-suggested or rule-based destination
    product_id      UUID NOT NULL REFERENCES product(id),
    quantity        NUMERIC(12,4) NOT NULL,
    uom             VARCHAR(10) NOT NULL DEFAULT 'EA',
    priority        INTEGER NOT NULL DEFAULT 50,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, assigned, in_progress, completed, cancelled
    assigned_to     UUID,
    suggestion_source VARCHAR(20) DEFAULT 'rule',            -- rule, ai_model
    ai_confidence   NUMERIC(5,4),                            -- AI model confidence score
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_putaway_status ON putaway_task(status) WHERE status IN ('pending', 'assigned', 'in_progress');
CREATE INDEX idx_putaway_assigned ON putaway_task(assigned_to) WHERE assigned_to IS NOT NULL;
```

## Outbound (Orders, Waves, Picking, Packing, Shipping)

```sql
CREATE TABLE outbound_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    order_number    VARCHAR(50) NOT NULL,
    customer_id     UUID REFERENCES trading_partner(id),
    client_id       UUID REFERENCES trading_partner(id),  -- 3PL client
    order_type      VARCHAR(20) NOT NULL,                  -- sales, transfer, replenishment
    priority        INTEGER NOT NULL DEFAULT 50,
    status          VARCHAR(20) NOT NULL DEFAULT 'new',    -- new, allocated, picking, packing, shipped, cancelled
    required_date   DATE,
    ship_by_date    DATE,
    carrier_code    VARCHAR(20),
    service_level   VARCHAR(20),                           -- ground, express, overnight
    ship_to_name    VARCHAR(200),
    ship_to_line_1  VARCHAR(200),
    ship_to_line_2  VARCHAR(200),
    ship_to_city    VARCHAR(100),
    ship_to_subdivision VARCHAR(6),
    ship_to_postal  VARCHAR(20),
    ship_to_country CHAR(2),                               -- ISO 3166-1
    edi_940_ref     VARCHAR(50),                           -- EDI 940 warehouse shipping order ref
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, order_number)
);

CREATE TABLE outbound_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outbound_order_id UUID NOT NULL REFERENCES outbound_order(id),
    line_number     INTEGER NOT NULL,
    product_id      UUID NOT NULL REFERENCES product(id),
    ordered_qty     NUMERIC(12,4) NOT NULL,
    allocated_qty   NUMERIC(12,4) NOT NULL DEFAULT 0,
    picked_qty      NUMERIC(12,4) NOT NULL DEFAULT 0,
    shipped_qty     NUMERIC(12,4) NOT NULL DEFAULT 0,
    uom             VARCHAR(10) NOT NULL DEFAULT 'EA',
    lot_number      VARCHAR(50),                           -- requested lot if specific
    status          VARCHAR(20) NOT NULL DEFAULT 'new',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(outbound_order_id, line_number)
);

CREATE TABLE wave (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    wave_number     VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'planned', -- planned, released, in_progress, completed
    planned_by      UUID,
    optimization_source VARCHAR(20) DEFAULT 'manual',       -- manual, rule, ai_model
    released_at     TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, wave_number)
);

CREATE TABLE wave_order (
    wave_id         UUID NOT NULL REFERENCES wave(id),
    outbound_order_id UUID NOT NULL REFERENCES outbound_order(id),
    PRIMARY KEY (wave_id, outbound_order_id)
);

CREATE TABLE pick_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    wave_id         UUID REFERENCES wave(id),
    outbound_order_id UUID NOT NULL REFERENCES outbound_order(id),
    outbound_line_id UUID NOT NULL REFERENCES outbound_order_line(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    from_location_id UUID NOT NULL REFERENCES location(id),
    inventory_id    UUID NOT NULL REFERENCES inventory(id),
    quantity        NUMERIC(12,4) NOT NULL,
    uom             VARCHAR(10) NOT NULL DEFAULT 'EA',
    pick_type       VARCHAR(20) NOT NULL DEFAULT 'single',  -- single, batch, cluster, zone
    priority        INTEGER NOT NULL DEFAULT 50,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, assigned, in_progress, completed, short_picked, cancelled
    assigned_to     UUID,
    pick_sequence   INTEGER,                                 -- optimised pick path sequence
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    picked_qty      NUMERIC(12,4) DEFAULT 0,
    to_location_id  UUID REFERENCES location(id),            -- staging/packing location
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pick_task_status ON pick_task(status) WHERE status IN ('pending', 'assigned', 'in_progress');
CREATE INDEX idx_pick_task_wave ON pick_task(wave_id);
CREATE INDEX idx_pick_task_assigned ON pick_task(assigned_to) WHERE assigned_to IS NOT NULL;
CREATE INDEX idx_pick_task_order ON pick_task(outbound_order_id);

CREATE TABLE pack_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    outbound_order_id UUID NOT NULL REFERENCES outbound_order(id),
    station_id      UUID REFERENCES location(id),           -- packing station location
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    assigned_to     UUID,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE shipping_container (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outbound_order_id UUID NOT NULL REFERENCES outbound_order(id),
    sscc            VARCHAR(18),                             -- GS1 Serial Shipping Container Code
    container_type  VARCHAR(20) NOT NULL,                    -- carton, pallet, envelope
    tracking_number VARCHAR(50),
    carrier_code    VARCHAR(20),
    weight_kg       NUMERIC(10,4),
    length_cm       NUMERIC(10,2),
    width_cm        NUMERIC(10,2),
    height_cm       NUMERIC(10,2),
    label_printed   BOOLEAN NOT NULL DEFAULT FALSE,
    shipped_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_shipping_container_sscc ON shipping_container(sscc) WHERE sscc IS NOT NULL;
CREATE INDEX idx_shipping_container_tracking ON shipping_container(tracking_number) WHERE tracking_number IS NOT NULL;

CREATE TABLE container_content (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    container_id    UUID NOT NULL REFERENCES shipping_container(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    quantity        NUMERIC(12,4) NOT NULL,
    lot_number      VARCHAR(50),
    serial_number   VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE shipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    shipment_number VARCHAR(50) NOT NULL,
    carrier_code    VARCHAR(20),
    trailer_number  VARCHAR(30),
    dock_door_id    UUID REFERENCES location(id),
    bill_of_lading  VARCHAR(50),
    status          VARCHAR(20) NOT NULL DEFAULT 'loading',  -- loading, shipped, delivered
    edi_945_ref     VARCHAR(50),                              -- EDI 945 shipping advice ref
    shipped_at      TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, shipment_number)
);

CREATE TABLE shipment_container (
    shipment_id     UUID NOT NULL REFERENCES shipment(id),
    container_id    UUID NOT NULL REFERENCES shipping_container(id),
    PRIMARY KEY (shipment_id, container_id)
);
```

## Cycle Counting and Quality

```sql
CREATE TABLE cycle_count (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    count_number    VARCHAR(30) NOT NULL,
    count_type      VARCHAR(20) NOT NULL,                    -- full, zone, abc, ai_prioritised
    status          VARCHAR(20) NOT NULL DEFAULT 'planned',
    priority_source VARCHAR(20) DEFAULT 'manual',            -- manual, schedule, ai_model
    planned_date    DATE,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, count_number)
);

CREATE TABLE cycle_count_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cycle_count_id  UUID NOT NULL REFERENCES cycle_count(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    product_id      UUID REFERENCES product(id),             -- NULL = blind count
    expected_qty    NUMERIC(12,4),
    counted_qty     NUMERIC(12,4),
    variance        NUMERIC(12,4),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    assigned_to     UUID,
    counted_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE quality_inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    receipt_id      UUID REFERENCES receipt(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    lot_number      VARCHAR(50),
    sample_size     INTEGER,
    pass_count      INTEGER,
    fail_count      INTEGER,
    result          VARCHAR(20) NOT NULL,                     -- passed, failed, conditional
    inspected_by    UUID NOT NULL,
    inspected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE non_conformance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID REFERENCES quality_inspection(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    ncr_number      VARCHAR(30) NOT NULL UNIQUE,
    category        VARCHAR(30) NOT NULL,                     -- damage, contamination, wrong_item, quantity_discrepancy
    severity        VARCHAR(10) NOT NULL,                     -- critical, major, minor
    disposition     VARCHAR(20),                              -- return_to_supplier, scrap, rework, accept_as_is
    quantity_affected NUMERIC(12,4),
    description     TEXT NOT NULL,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## EPCIS Visibility Events

```sql
CREATE TABLE epcis_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(30) NOT NULL,                     -- ObjectEvent, AggregationEvent, TransactionEvent, TransformationEvent
    event_time      TIMESTAMPTZ NOT NULL,
    event_timezone  VARCHAR(10) NOT NULL,
    action          VARCHAR(10) NOT NULL,                     -- ADD, OBSERVE, DELETE
    biz_step        VARCHAR(100),                             -- urn:epcglobal:cbv:bizstep:receiving, :shipping, etc.
    disposition     VARCHAR(100),                             -- urn:epcglobal:cbv:disp:in_transit, :sellable_accessible, etc.
    read_point_gln  VARCHAR(13),
    biz_location_gln VARCHAR(13),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE epcis_event_epc (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES epcis_event(id),
    epc_uri         VARCHAR(200) NOT NULL,                    -- urn:epc:id:sgtin:..., urn:epc:id:sscc:...
    epc_type        VARCHAR(20) NOT NULL                      -- sgtin, sscc, lgtin
);

CREATE INDEX idx_epcis_event_time ON epcis_event(event_time);
CREATE INDEX idx_epcis_event_type ON epcis_event(event_type);
CREATE INDEX idx_epcis_epc_uri ON epcis_event_epc(epc_uri);
```

## Labour and User Management

```sql
CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(50) NOT NULL UNIQUE,
    email           VARCHAR(200) NOT NULL UNIQUE,
    full_name       VARCHAR(200) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    auth_provider   VARCHAR(20) NOT NULL DEFAULT 'local',     -- local, oidc, saml
    external_id     VARCHAR(200),                              -- OIDC subject or SAML NameID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(50) NOT NULL UNIQUE,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id),
    permission_id   UUID NOT NULL REFERENCES permission(id),
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_warehouse_role (
    user_id         UUID NOT NULL REFERENCES user_account(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    PRIMARY KEY (user_id, warehouse_id, role_id)
);

CREATE TABLE labour_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES user_account(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    task_type       VARCHAR(30) NOT NULL,                      -- pick, putaway, pack, receive, count, replenish
    task_id         UUID,                                      -- references relevant task table
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    quantity_processed NUMERIC(12,4),
    zone_id         UUID REFERENCES zone(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_labour_log_user ON labour_log(user_id, started_at);
CREATE INDEX idx_labour_log_warehouse ON labour_log(warehouse_id, started_at);
```

## Slotting and AI Recommendations

```sql
CREATE TABLE slotting_profile (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    velocity_class  CHAR(1),                                   -- A, B, C, D
    avg_daily_picks NUMERIC(10,2),
    avg_daily_units NUMERIC(12,2),
    recommended_zone_id UUID REFERENCES zone(id),
    recommended_location_id UUID REFERENCES location(id),
    current_location_id UUID REFERENCES location(id),
    last_calculated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, product_id)
);

CREATE TABLE ai_recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    recommendation_type VARCHAR(30) NOT NULL,                  -- putaway, slotting, wave_plan, labour_schedule, cycle_count_priority
    model_version   VARCHAR(30),
    confidence      NUMERIC(5,4),
    payload_summary TEXT,                                       -- human-readable summary
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',    -- pending, accepted, rejected, expired
    reference_type  VARCHAR(30),
    reference_id    UUID,
    decided_by      UUID REFERENCES user_account(id),
    decided_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ai_rec_status ON ai_recommendation(status) WHERE status = 'pending';
CREATE INDEX idx_ai_rec_type ON ai_recommendation(recommendation_type, created_at);
```

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(50) NOT NULL,                      -- inventory, outbound_order, pick_task, etc.
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,                      -- create, update, delete, status_change
    changed_fields  TEXT[],                                     -- list of field names that changed
    old_values      TEXT,                                       -- serialised old values
    new_values      TEXT,                                       -- serialised new values
    performed_by    UUID NOT NULL,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ip_address      INET,
    user_agent      VARCHAR(300)
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_performed_at ON audit_log(performed_at);
CREATE INDEX idx_audit_user ON audit_log(performed_by);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Warehouse & Location | 6 | warehouse, address, zone, aisle, rack, location |
| Product & Item Master | 3 | product, UOM conversion, barcode |
| Inventory | 2 | inventory, adjustment |
| Inbound | 4 | trading_partner, inbound_order, inbound_line, receipt |
| Putaway | 1 | putaway_task |
| Outbound | 7 | outbound_order, outbound_line, wave, wave_order, pick_task, pack_task |
| Shipping | 4 | shipping_container, container_content, shipment, shipment_container |
| Cycle Count & Quality | 4 | cycle_count, cycle_count_task, quality_inspection, non_conformance |
| EPCIS | 2 | epcis_event, epcis_event_epc |
| Users & Labour | 6 | user_account, role, permission, role_permission, user_warehouse_role, labour_log |
| AI & Slotting | 2 | slotting_profile, ai_recommendation |
| Audit | 1 | audit_log |
| **Total** | **42** | |

---

## Key Design Decisions

1. **Full normalization over convenience** — every entity has its own table with proper foreign keys. This means more joins but ensures data integrity and prevents update anomalies.

2. **Location hierarchy (warehouse > zone > aisle > rack > location)** modeled as separate tables rather than a single self-referencing table, because the hierarchy is fixed in depth and each level has distinct attributes (temperature ranges on zones, weight limits on racks).

3. **Inventory table tracks position-level state** — one row per product-location-lot-serial combination, with computed `quantity_available`. The `inventory_adjustment` table provides a full transaction history.

4. **GS1 identifiers as first-class columns** — `gtin`, `sscc`, `gln` stored as typed VARCHAR columns with indexes, not buried in metadata, because they are queried constantly in receiving and shipping workflows.

5. **EPCIS events stored relationally** rather than as raw JSON, enabling SQL queries over visibility data for compliance reporting (FSMA 204, DSCSA).

6. **AI recommendations tracked explicitly** — the `ai_recommendation` table records what the AI suggested, the confidence level, and whether the operator accepted or rejected it, enabling model feedback loops.

7. **Per-warehouse RBAC** — users are assigned roles per warehouse via the `user_warehouse_role` junction table, supporting multi-site operations where a user may have different permissions at different facilities.

8. **Separate inbound and outbound order models** — rather than a generic "order" table with a type flag, inbound and outbound have distinct schemas reflecting their different workflows and field requirements.
