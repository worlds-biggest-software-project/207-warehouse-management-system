# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Warehouse Management System · Created: 2026-05-20

## Philosophy

This model combines a relational foundation for transactional CRUD operations with a property graph layer for relationship-heavy queries. The relational tables handle day-to-day warehouse operations (receiving, inventory, picking, shipping). The graph layer models the relationships that are hard to query relationally: product co-pick affinity networks, location adjacency for path optimization, supply chain custody chains, and hierarchical warehouse structures.

Warehouses are fundamentally spatial and relational: products are related to locations, locations are adjacent to other locations, products are frequently picked together, suppliers connect to products which connect to lots which connect to locations which connect to zones. Traditional relational models handle individual lookups well but struggle with graph traversal questions like "what is the shortest pick path through these 15 locations?" or "which products are most frequently co-picked with WIDGET-001 and should be slotted nearby?" or "trace this serial number through every hand it has passed through."

The graph layer is implemented using PostgreSQL `graph_node` and `graph_edge` tables with a lightweight adjacency model, keeping the entire system in a single database. For deployments that need more advanced graph capabilities, the edge table design is compatible with Neo4j or Apache AGE migration.

**Best for:** Operations where pick path optimization, product affinity analysis, supply chain traceability, or spatial warehouse layout reasoning are primary concerns. Especially valuable when the AI layer needs to reason about relationships between entities (slotting based on co-pick affinity, layout optimization, conflict-of-interest in 3PL client segregation).

**Trade-offs:**
- Pro: Graph queries for path optimization and affinity analysis are natural and efficient
- Pro: Product co-pick network enables AI slotting based on real relationship data
- Pro: Supply chain custody chain traversal for DSCSA/FSMA 204 compliance
- Pro: Location adjacency graph enables pick path optimization without external tools
- Pro: Single PostgreSQL database — no separate graph database to operate
- Con: Graph query patterns (recursive CTEs, multi-hop traversals) are less familiar to most developers
- Con: Edge table grows large with high-volume operations (millions of pick events generate affinity edges)
- Con: Graph maintenance (updating edge weights, pruning stale edges) adds operational complexity
- Con: If graph queries are not a primary use case, the added complexity is not justified
- Con: PostgreSQL recursive CTEs are slower than native graph databases for deep traversals (6+ hops)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN / SSCC / GLN | Relational columns on product, container, warehouse tables |
| GS1 EPCIS 2.0 | EPCIS custody chain modeled as graph edges between EPCs and locations, enabling traversal queries |
| ISO 3166-1/2 | Country and subdivision codes on address records |
| ANSI X12 856/940/945 | EDI references on inbound/outbound order tables |
| FDA FSMA 204 | Traceability chain modeled as a directed graph: supplier -> receipt -> location -> pick -> shipment -> customer |
| DSCSA | Serial number custody chain as graph path: manufacturer -> distributor -> warehouse -> pharmacy |
| GS1 General Specs | Barcode symbologies and identifier structures |

---

## Relational Core (Transactional CRUD)

```sql
-- ============================================================
-- WAREHOUSE AND LOCATION
-- ============================================================

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
    aisle           VARCHAR(10),
    rack            VARCHAR(10),
    level           INTEGER,
    position        INTEGER,
    max_weight_kg   NUMERIC(10,2),
    max_volume_m3   NUMERIC(10,4),
    -- Coordinates for spatial graph (aisle/bay grid position)
    x_coord         NUMERIC(10,2),
    y_coord         NUMERIC(10,2),
    z_coord         NUMERIC(10,2),                             -- floor level for multi-level
    is_pickable     BOOLEAN NOT NULL DEFAULT TRUE,
    is_receivable   BOOLEAN NOT NULL DEFAULT TRUE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(warehouse_id, barcode)
);

CREATE INDEX idx_location_zone ON location(zone_id);
CREATE INDEX idx_location_coords ON location(warehouse_id, x_coord, y_coord);

-- ============================================================
-- PRODUCT
-- ============================================================

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

CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;

-- ============================================================
-- INVENTORY
-- ============================================================

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_qty CHECK (quantity_on_hand >= 0 AND quantity_allocated >= 0 AND quantity_allocated <= quantity_on_hand)
);

CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_location ON inventory(location_id);
CREATE INDEX idx_inventory_warehouse_product ON inventory(warehouse_id, product_id);
CREATE INDEX idx_inventory_lot ON inventory(lot_number) WHERE lot_number IS NOT NULL;
CREATE INDEX idx_inventory_serial ON inventory(serial_number) WHERE serial_number IS NOT NULL;

-- ============================================================
-- ORDERS AND TASKS (abbreviated — same structure as Model 1)
-- ============================================================

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

CREATE TABLE inbound_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    order_number    VARCHAR(50) NOT NULL,
    supplier_id     UUID REFERENCES trading_partner(id),
    order_type      VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'expected',
    expected_date   DATE,
    asn_number      VARCHAR(50),
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
    status          VARCHAR(20) NOT NULL DEFAULT 'expected',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(inbound_order_id, line_number)
);

CREATE TABLE outbound_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    order_number    VARCHAR(50) NOT NULL,
    customer_id     UUID REFERENCES trading_partner(id),
    order_type      VARCHAR(20) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 50,
    status          VARCHAR(20) NOT NULL DEFAULT 'new',
    required_date   DATE,
    carrier_code    VARCHAR(20),
    service_level   VARCHAR(20),
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(outbound_order_id, line_number)
);

CREATE TABLE pick_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    wave_id         UUID,
    outbound_order_id UUID NOT NULL REFERENCES outbound_order(id),
    outbound_line_id UUID NOT NULL REFERENCES outbound_order_line(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    from_location_id UUID NOT NULL REFERENCES location(id),
    inventory_id    UUID NOT NULL REFERENCES inventory(id),
    quantity        NUMERIC(12,4) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 50,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    assigned_to     UUID,
    pick_sequence   INTEGER,                                   -- from graph-optimised path
    picked_qty      NUMERIC(12,4) DEFAULT 0,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pick_task_status ON pick_task(status) WHERE status IN ('pending', 'assigned', 'in_progress');

CREATE TABLE shipping_container (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outbound_order_id UUID NOT NULL REFERENCES outbound_order(id),
    sscc            VARCHAR(18),
    container_type  VARCHAR(20) NOT NULL,
    tracking_number VARCHAR(50),
    carrier_code    VARCHAR(20),
    weight_kg       NUMERIC(10,4),
    shipped_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- USER AND RBAC
-- ============================================================

CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(50) NOT NULL UNIQUE,
    email           VARCHAR(200) NOT NULL UNIQUE,
    full_name       VARCHAR(200) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(50) NOT NULL UNIQUE,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_warehouse_role (
    user_id         UUID NOT NULL REFERENCES user_account(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    PRIMARY KEY (user_id, warehouse_id, role_id)
);
```

## Graph Layer

The graph layer uses two tables — `graph_node` and `graph_edge` — that form a property graph within PostgreSQL. Every entity that participates in relationship analysis gets a corresponding graph node. Edges encode the relationships with typed labels, directional semantics, and weight/metadata.

```sql
-- ============================================================
-- GRAPH NODES
-- ============================================================

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(30) NOT NULL,                      -- product, location, zone, warehouse, supplier, customer, lot, serial, epc
    entity_id       UUID NOT NULL,                             -- FK to the relational entity
    label           VARCHAR(200) NOT NULL,                     -- human-readable label (SKU, barcode, partner code)
    warehouse_id    UUID,                                      -- partition key for warehouse-scoped queries
    properties      JSONB NOT NULL DEFAULT '{}',               -- node-specific properties
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(node_type, entity_id)
);

CREATE INDEX idx_graph_node_type ON graph_node(node_type);
CREATE INDEX idx_graph_node_entity ON graph_node(entity_id);
CREATE INDEX idx_graph_node_warehouse ON graph_node(warehouse_id) WHERE warehouse_id IS NOT NULL;
CREATE INDEX idx_graph_node_props ON graph_node USING GIN (properties);

-- ============================================================
-- GRAPH EDGES
-- ============================================================

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_type       VARCHAR(40) NOT NULL,                      -- see edge type taxonomy below
    source_node_id  UUID NOT NULL REFERENCES graph_node(id),
    target_node_id  UUID NOT NULL REFERENCES graph_node(id),
    weight          NUMERIC(12,4) DEFAULT 1.0,                 -- edge weight (co-pick count, distance, affinity score)
    direction       VARCHAR(10) NOT NULL DEFAULT 'directed',   -- directed, undirected
    warehouse_id    UUID,
    properties      JSONB NOT NULL DEFAULT '{}',               -- edge-specific metadata
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    valid_to        TIMESTAMPTZ,                               -- NULL = currently valid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_graph_edge_type ON graph_edge(edge_type);
CREATE INDEX idx_graph_edge_source ON graph_edge(source_node_id);
CREATE INDEX idx_graph_edge_target ON graph_edge(target_node_id);
CREATE INDEX idx_graph_edge_warehouse ON graph_edge(warehouse_id) WHERE warehouse_id IS NOT NULL;
CREATE INDEX idx_graph_edge_valid ON graph_edge(valid_from, valid_to);
CREATE INDEX idx_graph_edge_weight ON graph_edge(edge_type, weight DESC);
CREATE INDEX idx_graph_edge_props ON graph_edge USING GIN (properties);
```

### Edge Type Taxonomy

```
-- Spatial edges (warehouse layout)
ADJACENT_TO          -- location -> location: physical adjacency (weight = distance in meters)
BELONGS_TO_ZONE      -- location -> zone
ZONE_IN_WAREHOUSE    -- zone -> warehouse

-- Product affinity edges (built from pick history)
CO_PICKED_WITH       -- product -> product: weight = co-pick frequency in trailing 30 days
SUBSTITUTE_FOR       -- product -> product: interchangeable products
COMPONENT_OF         -- product -> product: kit/BOM relationship

-- Inventory edges (current state)
STORED_AT            -- product -> location: weight = quantity on hand
SUPPLIED_BY          -- product -> supplier: active supply relationship

-- Supply chain custody edges (EPCIS/traceability)
RECEIVED_FROM        -- lot/serial -> supplier: custody chain link
STORED_IN            -- lot/serial -> location: with timestamps
PICKED_FOR           -- lot/serial -> outbound_order
SHIPPED_TO           -- lot/serial -> customer
CONTAINED_IN         -- epc -> epc: EPCIS aggregation (item in case, case on pallet)

-- Operational edges
ASSIGNED_TO          -- task -> user: current task assignment
PART_OF_WAVE         -- order -> wave
```

### Example Graph Node and Edge Records

```sql
-- Product nodes
INSERT INTO graph_node (node_type, entity_id, label, warehouse_id, properties) VALUES
('product', 'prod-uuid-001', 'WIDGET-001', 'wh-uuid', '{"gtin": "00012345678905", "velocity_class": "A"}'),
('product', 'prod-uuid-002', 'GADGET-042', 'wh-uuid', '{"gtin": "00012345678912", "velocity_class": "A"}');

-- Location nodes
INSERT INTO graph_node (node_type, entity_id, label, warehouse_id, properties) VALUES
('location', 'loc-uuid-001', 'A1-02-01-B', 'wh-uuid', '{"x": 10.5, "y": 3.0, "zone": "PICK-A"}'),
('location', 'loc-uuid-002', 'A1-02-02-A', 'wh-uuid', '{"x": 10.5, "y": 4.5, "zone": "PICK-A"}');

-- Co-pick affinity edge (WIDGET-001 and GADGET-042 are picked together 142 times in 30 days)
INSERT INTO graph_edge (edge_type, source_node_id, target_node_id, weight, direction, warehouse_id, properties) VALUES
('CO_PICKED_WITH', 'node-uuid-prod-001', 'node-uuid-prod-002', 142, 'undirected', 'wh-uuid',
 '{"period_days": 30, "co_pick_pct": 0.34, "last_co_pick": "2026-05-19"}');

-- Location adjacency edge (1.5 meters apart)
INSERT INTO graph_edge (edge_type, source_node_id, target_node_id, weight, direction, warehouse_id, properties) VALUES
('ADJACENT_TO', 'node-uuid-loc-001', 'node-uuid-loc-002', 1.5, 'undirected', 'wh-uuid',
 '{"same_aisle": true, "same_level": false}');

-- Custody chain edge (lot received from supplier)
INSERT INTO graph_edge (edge_type, source_node_id, target_node_id, weight, direction, warehouse_id, properties) VALUES
('RECEIVED_FROM', 'node-uuid-lot-001', 'node-uuid-supplier-001', 1, 'directed', 'wh-uuid',
 '{"receipt_date": "2026-05-20", "po_number": "PO-12345", "asn": "ASN-2026-1234", "quantity": 500}');
```

## Example Graph Queries

### Find top 10 co-pick affinities for a product (AI slotting input)

```sql
-- "Which products are most frequently co-picked with WIDGET-001?"
SELECT
    target.label AS co_picked_product,
    e.weight AS co_pick_count,
    (e.properties->>'co_pick_pct')::NUMERIC AS co_pick_percentage
FROM graph_edge e
JOIN graph_node source ON source.id = e.source_node_id
JOIN graph_node target ON target.id = e.target_node_id
WHERE source.label = 'WIDGET-001'
  AND source.node_type = 'product'
  AND e.edge_type = 'CO_PICKED_WITH'
  AND e.valid_to IS NULL
ORDER BY e.weight DESC
LIMIT 10;
```

### Optimise pick path using location adjacency graph

```sql
-- Given a set of pick locations, find the shortest traversal path
-- using recursive CTE on the adjacency graph
WITH RECURSIVE pick_locations AS (
    -- The set of locations we need to visit for this wave
    SELECT gn.id AS node_id, gn.label, gn.properties->>'x' AS x, gn.properties->>'y' AS y
    FROM graph_node gn
    WHERE gn.entity_id IN ('loc-uuid-001', 'loc-uuid-002', 'loc-uuid-003', 'loc-uuid-004')
      AND gn.node_type = 'location'
),
nearest_neighbor AS (
    -- Simple nearest-neighbor heuristic starting from dock
    SELECT
        pl.node_id,
        pl.label,
        ROW_NUMBER() OVER (ORDER BY (pl.x::NUMERIC) + (pl.y::NUMERIC)) AS visit_order
    FROM pick_locations pl
)
SELECT visit_order, label
FROM nearest_neighbor
ORDER BY visit_order;
```

### Trace custody chain for a serial number (DSCSA/FSMA 204)

```sql
-- "Show the full chain of custody for serial number SN-12345"
WITH RECURSIVE custody_chain AS (
    -- Start from the serial number node
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        e.edge_type,
        e.properties AS edge_props,
        1 AS depth,
        ARRAY[gn.id] AS visited
    FROM graph_node gn
    LEFT JOIN graph_edge e ON e.source_node_id = gn.id
        AND e.edge_type IN ('RECEIVED_FROM', 'STORED_IN', 'PICKED_FOR', 'SHIPPED_TO')
    WHERE gn.label = 'SN-12345'
      AND gn.node_type = 'serial'

    UNION ALL

    -- Traverse outward
    SELECT
        target.id,
        target.node_type,
        target.label,
        e2.edge_type,
        e2.properties,
        cc.depth + 1,
        cc.visited || target.id
    FROM custody_chain cc
    JOIN graph_edge e2 ON e2.source_node_id = cc.node_id
        AND e2.edge_type IN ('RECEIVED_FROM', 'STORED_IN', 'PICKED_FOR', 'SHIPPED_TO')
    JOIN graph_node target ON target.id = e2.target_node_id
    WHERE target.id != ALL(cc.visited)
      AND cc.depth < 10
)
SELECT depth, node_type, label, edge_type,
       edge_props->>'receipt_date' AS event_date
FROM custody_chain
ORDER BY depth;
```

### AI slotting: find optimal location for a product based on co-pick neighbours

```sql
-- "Where should WIDGET-001 be slotted based on where its co-picked products are stored?"
WITH co_picked AS (
    SELECT target.entity_id AS product_id, e.weight AS affinity
    FROM graph_edge e
    JOIN graph_node source ON source.id = e.source_node_id
    JOIN graph_node target ON target.id = e.target_node_id
    WHERE source.label = 'WIDGET-001' AND source.node_type = 'product'
      AND e.edge_type = 'CO_PICKED_WITH' AND e.valid_to IS NULL
    ORDER BY e.weight DESC LIMIT 5
),
co_picked_locations AS (
    SELECT i.location_id, SUM(cp.affinity) AS total_affinity
    FROM co_picked cp
    JOIN inventory i ON i.product_id = cp.product_id AND i.status = 'available'
    GROUP BY i.location_id
)
SELECT l.barcode, l.zone_id, cpl.total_affinity,
       l.x_coord, l.y_coord
FROM co_picked_locations cpl
JOIN location l ON l.id = cpl.location_id
WHERE l.is_pickable = TRUE
ORDER BY cpl.total_affinity DESC
LIMIT 5;
```

## Graph Maintenance Jobs

```sql
-- Periodic job: rebuild co-pick affinity edges from pick history (run nightly)
-- This replaces old CO_PICKED_WITH edges with fresh ones from the trailing 30 days

-- Step 1: Expire old edges
UPDATE graph_edge
SET valid_to = NOW()
WHERE edge_type = 'CO_PICKED_WITH'
  AND valid_to IS NULL;

-- Step 2: Build new edges from pick history
INSERT INTO graph_edge (edge_type, source_node_id, target_node_id, weight, direction, warehouse_id, properties)
SELECT
    'CO_PICKED_WITH',
    gn1.id,
    gn2.id,
    COUNT(*),
    'undirected',
    pt1.warehouse_id,
    jsonb_build_object(
        'period_days', 30,
        'co_pick_pct', ROUND(COUNT(*)::NUMERIC / NULLIF(total_picks.cnt, 0), 4),
        'last_co_pick', MAX(pt1.completed_at)::TEXT
    )
FROM pick_task pt1
JOIN pick_task pt2
    ON pt1.outbound_order_id = pt2.outbound_order_id
    AND pt1.product_id < pt2.product_id  -- avoid duplicates
    AND pt1.status = 'completed'
    AND pt2.status = 'completed'
    AND pt1.completed_at >= NOW() - INTERVAL '30 days'
JOIN graph_node gn1 ON gn1.entity_id = pt1.product_id AND gn1.node_type = 'product'
JOIN graph_node gn2 ON gn2.entity_id = pt2.product_id AND gn2.node_type = 'product'
CROSS JOIN LATERAL (
    SELECT COUNT(*) AS cnt FROM pick_task
    WHERE product_id = pt1.product_id AND status = 'completed'
      AND completed_at >= NOW() - INTERVAL '30 days'
) total_picks
GROUP BY gn1.id, gn2.id, pt1.warehouse_id, total_picks.cnt
HAVING COUNT(*) >= 3;  -- minimum co-pick threshold
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Warehouse & Location | 3 | warehouse, zone, location (with x/y/z coordinates) |
| Product | 1 | product |
| Inventory | 1 | inventory |
| Inbound | 2 | inbound_order, inbound_order_line |
| Outbound | 2 | outbound_order, outbound_order_line |
| Tasks | 1 | pick_task |
| Shipping | 1 | shipping_container |
| Trading Partners | 1 | trading_partner |
| Users & RBAC | 3 | user_account, role, user_warehouse_role |
| **Graph Layer** | **2** | **graph_node, graph_edge** |
| **Total** | **17** | Compact relational core + 2-table graph layer |

---

## Key Design Decisions

1. **Two-table property graph** — `graph_node` and `graph_edge` implement a full property graph within PostgreSQL. Every entity that needs relationship analysis gets a node; every relationship gets a typed, weighted, temporal edge. This avoids adding a separate graph database (Neo4j) to the infrastructure.

2. **Temporal edges with `valid_from` / `valid_to`** — edges are versioned. When co-pick affinities are recalculated, old edges are expired (set `valid_to`) and new ones created. This preserves historical relationship data for AI training while keeping current queries fast.

3. **Co-pick affinity as a graph** — rather than a simple junction table or materialized view, co-pick relationships are modeled as weighted graph edges. This enables multi-hop reasoning: "if A is co-picked with B, and B is co-picked with C, then A and C might benefit from proximity even if they are rarely directly co-picked."

4. **Location coordinates (x, y, z)** — each location has physical coordinates enabling distance calculations and adjacency graph construction. The `ADJACENT_TO` edges encode the actual walking distances between locations, which the pick path optimizer traverses.

5. **EPCIS custody chain as graph** — traceability queries (required by FSMA 204 and DSCSA) become graph traversals. "Show me the full chain of custody for this lot" is a recursive traversal through `RECEIVED_FROM`, `STORED_IN`, `PICKED_FOR`, `SHIPPED_TO` edges, which is natural in a graph model and awkward in a pure relational model.

6. **Relational core kept lean** — the relational tables focus on transactional CRUD. The graph layer adds relationship intelligence on top. This means standard CRUD operations (create order, confirm pick, adjust inventory) are straightforward relational operations; only relationship-heavy queries hit the graph.

7. **Graph maintenance is a batch job** — co-pick affinity edges are rebuilt nightly from pick task history. Location adjacency edges are rebuilt when the warehouse layout changes. This keeps the graph fresh without adding overhead to every transaction.

8. **Migration path to native graph** — the `graph_node` / `graph_edge` schema is compatible with Apache AGE (PostgreSQL graph extension) or Neo4j import. If graph query performance becomes a bottleneck, the graph layer can be migrated to a native graph database without changing the relational core.
