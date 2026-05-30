# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Warehouse Management System · Created: 2026-05-20

## Philosophy

This model uses a normalized relational core for the entities and relationships that are universal across all warehouse operations (locations, products, inventory, orders, tasks), but stores variable, domain-specific, and extensible attributes in JSONB columns. The key insight is that a WMS serving diverse verticals — food/beverage, pharma, electronics, fashion, 3PL multi-client — cannot anticipate every field a specific deployment needs.

Rather than adding dozens of nullable columns for vertical-specific data (catch weight, temperature readings, FDA lot genealogy, fashion size/colour, hazmat classifications) or maintaining a complex EAV (entity-attribute-value) pattern, this model puts a typed JSONB column on the tables that need extensibility. Core fields that are always present and always queried (SKU, quantity, location, status) remain as proper relational columns with types and constraints. Variable fields go into `attributes JSONB` with partial GIN indexes for the attributes that matter.

This is the pattern used by modern SaaS platforms that serve multiple verticals from a single codebase (Shopify's metafields, Stripe's metadata, Deposco's configurable workflows). It enables rapid feature development — a new vertical requirement means adding JSON keys to the attribute column, not running ALTER TABLE migrations across a production database.

**Best for:** Teams building a multi-vertical or 3PL-focused WMS where different clients or industries need different data fields, and rapid iteration on schema is more important than maximum query performance on every attribute.

**Trade-offs:**
- Pro: New attributes can be added without database migrations
- Pro: Vertical-specific fields (pharma, food, fashion) coexist without sparse columns
- Pro: 3PL multi-client: each client can have custom product attributes and order fields
- Pro: Faster MVP development — core schema is smaller and simpler
- Pro: JSONB GIN indexes provide good query performance on frequently-accessed attributes
- Con: JSONB fields lack database-level type enforcement — validation moves to the application layer
- Con: Complex queries on JSONB attributes are slower than on typed columns
- Con: Reporting and BI tools may not handle JSONB columns natively
- Con: Risk of "schema drift" if JSONB attribute structures are not governed
- Con: Cannot use foreign keys inside JSONB — referential integrity is application-enforced for JSONB references

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | Relational column `product.gtin` — always present, always indexed |
| GS1 SSCC | Relational column `container.sscc` — always present for shipping containers |
| GS1 GLN | Relational column on warehouse and trading partner |
| GS1 EPCIS 2.0 | EPCIS events stored as JSONB documents in `epcis_event.event_data`, matching the EPCIS 2.0 JSON-LD schema directly |
| ISO 3166-1/2 | Relational columns for country/subdivision on address records |
| ANSI X12 856/940/945 | EDI references stored as relational columns; EDI segment-specific data in `edi_attributes JSONB` |
| FDA FSMA 204 | CTE/KDE data stored in `inventory.compliance_attributes` JSONB — varies by commodity |
| DSCSA | Serialisation chain data in `product.pharma_attributes` JSONB |
| ISO 9001 | Quality inspection results with flexible `findings JSONB` for varying inspection criteria |

---

## Warehouse and Location

```sql
CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    gln             VARCHAR(13),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Extensible: operating hours, contact info, certifications
    attributes      JSONB NOT NULL DEFAULT '{}',
    /*  attributes example:
    {
        "operating_hours": {"mon_fri": "06:00-22:00", "sat": "06:00-14:00"},
        "certifications": ["ISO9001", "FSSC22000", "GDP"],
        "contact": {"name": "John Smith", "phone": "+1-555-0100"},
        "cold_chain_capable": true,
        "hazmat_certified": true
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    zone_code       VARCHAR(20) NOT NULL,
    zone_type       VARCHAR(30) NOT NULL,                      -- storage, staging, receiving, shipping, quarantine
    barcode         VARCHAR(50) NOT NULL,
    location_type   VARCHAR(30) NOT NULL,                      -- bin, floor, pallet, bulk, pick_face
    aisle           VARCHAR(10),
    rack            VARCHAR(10),
    level           INTEGER,
    position        INTEGER,
    max_weight_kg   NUMERIC(10,2),
    max_volume_m3   NUMERIC(10,4),
    is_pickable     BOOLEAN NOT NULL DEFAULT TRUE,
    is_receivable   BOOLEAN NOT NULL DEFAULT TRUE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Extensible: temperature range, humidity, special handling
    attributes      JSONB NOT NULL DEFAULT '{}',
    /*  attributes example:
    {
        "temperature_min_c": -18,
        "temperature_max_c": -15,
        "humidity_max_pct": 40,
        "requires_forklift": true,
        "pallet_positions": 2,
        "pick_sequence": 142
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, barcode)
);

CREATE INDEX idx_location_zone ON location(warehouse_id, zone_code);
CREATE INDEX idx_location_type ON location(location_type);
CREATE INDEX idx_location_attrs ON location USING GIN (attributes);
```

## Product Master

```sql
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
    -- Extensible: vertical-specific product attributes
    attributes      JSONB NOT NULL DEFAULT '{}',
    /*  attributes example (food):
    {
        "allergens": ["peanuts", "soy"],
        "organic_certified": true,
        "country_of_origin": "US",
        "catch_weight": true,
        "min_shelf_life_days": 90,
        "fda_food_category": "Leafy Greens"
    }
    */
    /*  attributes example (pharma):
    {
        "ndc": "12345-678-90",
        "dea_schedule": "II",
        "cold_chain_required": true,
        "dscsa_serialised": true,
        "storage_temp_range": {"min_c": 2, "max_c": 8}
    }
    */
    /*  attributes example (fashion):
    {
        "colour": "Navy Blue",
        "size": "L",
        "style_number": "FW26-1234",
        "season": "FW2026",
        "material": "100% Cotton"
    }
    */
    -- UOM conversions stored inline for simplicity
    uom_conversions JSONB NOT NULL DEFAULT '[]',
    /*  uom_conversions example:
    [
        {"from": "CS", "to": "EA", "factor": 24},
        {"from": "PL", "to": "CS", "factor": 80}
    ]
    */
    -- Barcodes stored inline
    barcodes        JSONB NOT NULL DEFAULT '[]',
    /*  barcodes example:
    [
        {"value": "00012345678905", "type": "EAN-13", "uom": "EA", "primary": true},
        {"value": "10012345678912", "type": "GS1-128", "uom": "CS", "primary": false}
    ]
    */
    -- 3PL: which clients own this product
    client_id       UUID,                                      -- owning client for 3PL
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_client ON product(client_id) WHERE client_id IS NOT NULL;
CREATE INDEX idx_product_attrs ON product USING GIN (attributes);
CREATE INDEX idx_product_barcodes ON product USING GIN (barcodes);
```

## Inventory

```sql
CREATE TABLE inventory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    lot_number      VARCHAR(50),
    serial_number   VARCHAR(100),
    lpn             VARCHAR(50),
    quantity_on_hand NUMERIC(12,4) NOT NULL DEFAULT 0,
    quantity_allocated NUMERIC(12,4) NOT NULL DEFAULT 0,
    quantity_available NUMERIC(12,4) GENERATED ALWAYS AS (quantity_on_hand - quantity_allocated) STORED,
    uom             VARCHAR(10) NOT NULL DEFAULT 'EA',
    status          VARCHAR(20) NOT NULL DEFAULT 'available',
    expiry_date     DATE,
    received_at     TIMESTAMPTZ,
    -- Extensible: catch weight, temperature log, compliance data
    attributes      JSONB NOT NULL DEFAULT '{}',
    /*  attributes example (food/catch weight):
    {
        "actual_weight_kg": 12.45,
        "temperature_at_receipt_c": 3.2,
        "country_of_origin": "MX",
        "fsma_204_cte": {
            "critical_tracking_event": "receiving",
            "traceability_lot_code": "TLC-2026-05-20-001",
            "reference_document": "PO-12345"
        }
    }
    */
    /*  attributes example (pharma):
    {
        "serial_number_chain": ["SN-001", "SN-002"],
        "dscsa_transaction_id": "TI-2026-0520-789",
        "temperature_excursion": false,
        "last_verification_date": "2026-05-15"
    }
    */
    -- 3PL client ownership
    client_id       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_qty_non_negative CHECK (quantity_on_hand >= 0),
    CONSTRAINT chk_allocated_lte_on_hand CHECK (quantity_allocated <= quantity_on_hand)
);

CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_location ON inventory(location_id);
CREATE INDEX idx_inventory_warehouse_product ON inventory(warehouse_id, product_id);
CREATE INDEX idx_inventory_lot ON inventory(lot_number) WHERE lot_number IS NOT NULL;
CREATE INDEX idx_inventory_serial ON inventory(serial_number) WHERE serial_number IS NOT NULL;
CREATE INDEX idx_inventory_lpn ON inventory(lpn) WHERE lpn IS NOT NULL;
CREATE INDEX idx_inventory_expiry ON inventory(expiry_date) WHERE expiry_date IS NOT NULL;
CREATE INDEX idx_inventory_client ON inventory(client_id) WHERE client_id IS NOT NULL;
CREATE INDEX idx_inventory_attrs ON inventory USING GIN (attributes);

CREATE TABLE inventory_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inventory_id    UUID NOT NULL REFERENCES inventory(id),
    transaction_type VARCHAR(30) NOT NULL,
    quantity_change NUMERIC(12,4) NOT NULL,
    reason_code     VARCHAR(30),
    reference_type  VARCHAR(30),
    reference_id    UUID,
    performed_by    UUID NOT NULL,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Extensible: transaction-specific metadata
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_inv_tx_inventory ON inventory_transaction(inventory_id);
CREATE INDEX idx_inv_tx_time ON inventory_transaction(performed_at);
```

## Trading Partners and 3PL Clients

```sql
CREATE TABLE trading_partner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(30) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    partner_type    VARCHAR(20) NOT NULL,                      -- supplier, customer, carrier, 3pl_client
    gln             VARCHAR(13),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Extensible: billing config, EDI settings, SLAs, contact info
    attributes      JSONB NOT NULL DEFAULT '{}',
    /*  attributes example (3PL client):
    {
        "billing_model": "per_transaction",
        "rate_card": {
            "receiving_per_pallet": 8.50,
            "storage_per_pallet_day": 0.75,
            "pick_per_line": 1.25,
            "pack_per_order": 2.00
        },
        "sla": {
            "same_day_cutoff": "14:00",
            "order_accuracy_target_pct": 99.5
        },
        "edi_config": {
            "qualifier": "ZZ",
            "interchange_id": "ACME001",
            "document_types": ["856", "940", "945"]
        },
        "contacts": [
            {"role": "operations", "name": "Jane Doe", "email": "jane@acme.com"}
        ]
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Inbound Orders

```sql
CREATE TABLE inbound_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    order_number    VARCHAR(50) NOT NULL,
    supplier_id     UUID REFERENCES trading_partner(id),
    client_id       UUID REFERENCES trading_partner(id),
    order_type      VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'expected',
    expected_date   DATE,
    -- Extensible: ASN data, appointment, carrier details
    attributes      JSONB NOT NULL DEFAULT '{}',
    /*  attributes example:
    {
        "asn_number": "ASN-2026-1234",
        "po_number": "PO-56789",
        "carrier": {"code": "FEDX", "trailer": "TRL-4567", "seal": "SEAL-890"},
        "dock_appointment": {"door": "DOCK-03", "time": "2026-05-20T08:00:00-05:00"},
        "customs_clearance": {"entry_number": "ENT-2026-001", "cleared": true}
    }
    */
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
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(inbound_order_id, line_number)
);
```

## Outbound Orders and Tasks

```sql
CREATE TABLE outbound_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    order_number    VARCHAR(50) NOT NULL,
    customer_id     UUID REFERENCES trading_partner(id),
    client_id       UUID REFERENCES trading_partner(id),
    order_type      VARCHAR(20) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 50,
    status          VARCHAR(20) NOT NULL DEFAULT 'new',
    required_date   DATE,
    ship_by_date    DATE,
    carrier_code    VARCHAR(20),
    service_level   VARCHAR(20),
    -- Ship-to as JSONB: address structures vary by country
    ship_to         JSONB NOT NULL DEFAULT '{}',
    /*  ship_to example:
    {
        "name": "Acme Corp",
        "line_1": "123 Main St",
        "line_2": "Suite 400",
        "city": "Chicago",
        "subdivision": "US-IL",
        "postal_code": "60601",
        "country": "US",
        "phone": "+1-555-0100"
    }
    */
    -- Extensible: channel info, gift options, special instructions
    attributes      JSONB NOT NULL DEFAULT '{}',
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
    status          VARCHAR(20) NOT NULL DEFAULT 'new',
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(outbound_order_id, line_number)
);

-- Unified task table: pick, putaway, pack, count, replenish
CREATE TABLE warehouse_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    task_type       VARCHAR(20) NOT NULL,                      -- pick, putaway, pack, count, replenish
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    priority        INTEGER NOT NULL DEFAULT 50,
    assigned_to     UUID,
    product_id      UUID REFERENCES product(id),
    from_location_id UUID REFERENCES location(id),
    to_location_id  UUID REFERENCES location(id),
    quantity        NUMERIC(12,4),
    uom             VARCHAR(10) DEFAULT 'EA',
    -- Reference to parent entity
    reference_type  VARCHAR(30),                               -- inbound_order, outbound_order, cycle_count, wave
    reference_id    UUID,
    -- Extensible: task-specific fields
    attributes      JSONB NOT NULL DEFAULT '{}',
    /*  attributes example (pick):
    {
        "wave_id": "uuid-here",
        "order_number": "ORD-789",
        "pick_type": "batch",
        "pick_sequence": 14,
        "lot_number": "LOT-2026-0520",
        "serial_numbers_required": 3,
        "ai_optimised_path": true
    }
    */
    /*  attributes example (putaway):
    {
        "receipt_id": "uuid-here",
        "suggestion_source": "ai_model",
        "ai_confidence": 0.94,
        "alternative_locations": ["B2-03-01", "B2-03-02"]
    }
    */
    /*  attributes example (count):
    {
        "count_type": "cycle",
        "expected_qty": 150,
        "counted_qty": 148,
        "variance": -2,
        "variance_pct": -1.33,
        "priority_source": "ai_model"
    }
    */
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_task_status ON warehouse_task(warehouse_id, status)
    WHERE status IN ('pending', 'assigned', 'in_progress');
CREATE INDEX idx_task_assigned ON warehouse_task(assigned_to) WHERE assigned_to IS NOT NULL;
CREATE INDEX idx_task_type ON warehouse_task(task_type, status);
CREATE INDEX idx_task_reference ON warehouse_task(reference_type, reference_id);
CREATE INDEX idx_task_attrs ON warehouse_task USING GIN (attributes);
```

## Waves

```sql
CREATE TABLE wave (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    wave_number     VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'planned',
    optimization_source VARCHAR(20) DEFAULT 'manual',
    -- Extensible: wave parameters, AI optimization metadata
    attributes      JSONB NOT NULL DEFAULT '{}',
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
```

## Shipping

```sql
CREATE TABLE shipping_container (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outbound_order_id UUID NOT NULL REFERENCES outbound_order(id),
    sscc            VARCHAR(18),
    container_type  VARCHAR(20) NOT NULL,
    tracking_number VARCHAR(50),
    carrier_code    VARCHAR(20),
    weight_kg       NUMERIC(10,4),
    length_cm       NUMERIC(10,2),
    width_cm        NUMERIC(10,2),
    height_cm       NUMERIC(10,2),
    shipped_at      TIMESTAMPTZ,
    -- Extensible: label data, customs, dangerous goods declarations
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_container_sscc ON shipping_container(sscc) WHERE sscc IS NOT NULL;
CREATE INDEX idx_container_tracking ON shipping_container(tracking_number) WHERE tracking_number IS NOT NULL;

CREATE TABLE container_content (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    container_id    UUID NOT NULL REFERENCES shipping_container(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    quantity        NUMERIC(12,4) NOT NULL,
    lot_number      VARCHAR(50),
    serial_number   VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## EPCIS Events (JSON-LD Native)

```sql
CREATE TABLE epcis_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(30) NOT NULL,                      -- ObjectEvent, AggregationEvent, TransactionEvent
    event_time      TIMESTAMPTZ NOT NULL,
    -- Full EPCIS 2.0 JSON-LD document stored directly
    event_data      JSONB NOT NULL,
    /*  event_data example (EPCIS 2.0 ObjectEvent):
    {
        "@context": "https://ref.gs1.org/standards/epcis/2.0.0/epcis-context.jsonld",
        "type": "ObjectEvent",
        "eventTime": "2026-05-20T14:30:00.000-05:00",
        "eventTimeZoneOffset": "-05:00",
        "epcList": ["urn:epc:id:sgtin:0614141.107346.2017"],
        "action": "OBSERVE",
        "bizStep": "urn:epcglobal:cbv:bizstep:receiving",
        "disposition": "urn:epcglobal:cbv:disp:in_progress",
        "readPoint": {"id": "urn:epc:id:sgln:0614141.07346.1234"},
        "bizLocation": {"id": "urn:epc:id:sgln:0614141.07346.0"}
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_epcis_event_time ON epcis_event(event_time);
CREATE INDEX idx_epcis_event_type ON epcis_event(event_type);
CREATE INDEX idx_epcis_event_data ON epcis_event USING GIN (event_data);
```

## Users, Roles, and Audit

```sql
CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(50) NOT NULL UNIQUE,
    email           VARCHAR(200) NOT NULL UNIQUE,
    full_name       VARCHAR(200) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    attributes      JSONB NOT NULL DEFAULT '{}',               -- certifications, preferences
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(50) NOT NULL UNIQUE,
    permissions     JSONB NOT NULL DEFAULT '[]',               -- ["inventory.read", "inventory.write", "orders.manage"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_warehouse_role (
    user_id         UUID NOT NULL REFERENCES user_account(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    PRIMARY KEY (user_id, warehouse_id, role_id)
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changes         JSONB NOT NULL DEFAULT '{}',               -- {"field": {"old": x, "new": y}}
    performed_by    UUID NOT NULL,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata        JSONB NOT NULL DEFAULT '{}'                -- ip_address, user_agent, etc.
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(performed_at);
```

## AI and Slotting

```sql
CREATE TABLE ai_recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    recommendation_type VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    confidence      NUMERIC(5,4),
    -- Full recommendation payload as JSONB
    payload         JSONB NOT NULL,
    /*  payload example (slotting):
    {
        "product_sku": "WIDGET-001",
        "current_location": "C2-08-03-A",
        "recommended_location": "A1-02-01-B",
        "reason": "velocity_increase",
        "metrics": {
            "avg_daily_picks_30d": 45.2,
            "velocity_class_current": "B",
            "velocity_class_new": "A",
            "estimated_travel_reduction_pct": 32
        },
        "model_version": "slot-v3.2",
        "alternatives": [
            {"location": "A1-02-02-A", "confidence": 0.87},
            {"location": "A1-03-01-B", "confidence": 0.81}
        ]
    }
    */
    decided_by      UUID REFERENCES user_account(id),
    decided_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ai_rec_status ON ai_recommendation(status) WHERE status = 'pending';
CREATE INDEX idx_ai_rec_type ON ai_recommendation(recommendation_type, created_at);
```

## Example Queries

### Query JSONB product attributes for pharma compliance

```sql
-- Find all DSCSA-serialised products in cold chain
SELECT sku, name, attributes->'ndc' AS ndc, attributes->'storage_temp_range' AS temp_range
FROM product
WHERE attributes @> '{"dscsa_serialised": true, "cold_chain_required": true}'
  AND is_active = TRUE;
```

### Query inventory with FSMA 204 compliance data

```sql
-- Find all inventory with FSMA 204 CTEs for a specific traceability lot code
SELECT i.lot_number, p.sku, p.name, l.barcode AS location,
       i.quantity_on_hand, i.attributes->'fsma_204_cte' AS cte_data
FROM inventory i
JOIN product p ON p.id = i.product_id
JOIN location l ON l.id = i.location_id
WHERE i.attributes @> '{"fsma_204_cte": {"critical_tracking_event": "receiving"}}'
  AND i.warehouse_id = 'warehouse-uuid';
```

### Query tasks with AI metadata

```sql
-- Find all AI-suggested putaway tasks with confidence > 0.9
SELECT wt.id, p.sku, l_from.barcode AS from_loc, l_to.barcode AS to_loc,
       wt.attributes->>'ai_confidence' AS confidence,
       wt.attributes->'alternative_locations' AS alternatives
FROM warehouse_task wt
JOIN product p ON p.id = wt.product_id
JOIN location l_from ON l_from.id = wt.from_location_id
JOIN location l_to ON l_to.id = wt.to_location_id
WHERE wt.task_type = 'putaway'
  AND wt.status = 'pending'
  AND (wt.attributes->>'ai_confidence')::NUMERIC > 0.9;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Warehouse & Location | 2 | Flattened: zone is a column, aisle/rack are location attributes |
| Product | 1 | Barcodes and UOM conversions inline as JSONB arrays |
| Inventory | 2 | inventory + inventory_transaction |
| Inbound | 2 | inbound_order + lines |
| Outbound | 2 | outbound_order + lines |
| Tasks | 1 | Unified warehouse_task (pick, putaway, pack, count) |
| Waves | 2 | wave + wave_order |
| Shipping | 2 | shipping_container + container_content |
| EPCIS | 1 | EPCIS events as JSON-LD documents |
| Users & Auth | 3 | user_account, role, user_warehouse_role |
| AI | 1 | ai_recommendation |
| Audit | 1 | audit_log |
| **Total** | **20** | ~50% fewer tables than fully normalized model |

---

## Key Design Decisions

1. **JSONB `attributes` on core entities** — product, inventory, location, order, task, and trading_partner all carry an `attributes JSONB` column for vertical-specific data. The application layer validates attribute schemas using JSON Schema definitions per vertical.

2. **Unified `warehouse_task` table** — instead of separate pick_task, putaway_task, pack_task, cycle_count_task tables, a single task table with `task_type` discrimination and task-specific details in `attributes`. This simplifies the task assignment and management APIs while preserving flexibility.

3. **Ship-to address as JSONB** — international address formats vary significantly (Japanese addresses have no street names, UK addresses have post towns). JSONB avoids forcing all addresses into a US-centric column structure.

4. **Permissions stored as JSONB array on role** — instead of a permission table and junction table, permissions are a JSON array of string codes. This is simpler for most deployments and easy to query with `@>` containment.

5. **EPCIS events stored as native JSON-LD** — rather than decomposing EPCIS events into relational tables, they are stored as the original JSON-LD documents matching the GS1 EPCIS 2.0 specification. GIN indexes enable queries. This means EPCIS export is a simple SELECT, not a complex join and transformation.

6. **Location hierarchy flattened** — zone_code, aisle, rack, level, and position are columns on the location table rather than separate zone/aisle/rack tables. Additional location metadata goes in `attributes`. This reduces joins for the most common query pattern (find inventory at location).

7. **3PL multi-client via client_id** — products and inventory carry an optional `client_id` foreign key. Client-specific configuration (billing rates, SLAs, EDI settings) lives in the trading_partner `attributes` JSONB. No separate per-client schema needed.

8. **JSON Schema governance** — the model assumes that the application layer maintains JSON Schema definitions for each `attributes` JSONB column, per vertical. These schemas are not enforced by the database but by the API validation layer, enabling rapid evolution without migrations.
