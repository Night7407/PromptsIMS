# IMS Seed Data

> Table/column names below are best-guess defaults based on your entity definitions (Hibernate's default naming strategy converts camelCase fields to snake_case columns, and uses the entity name lowercased as the table name unless `@Table(name=...)` overrides it).
>
> **Before running:** connect to each database and run `\dt` (list tables) and `\d <table>` (columns) to confirm actual names, then adjust anything that doesn't match. The `users` table name is already confirmed working from your admin insert earlier — the rest are educated guesses.
>
> Run each block against its own database:
> ```
> psql -h localhost -U <your_pg_user> -d <database_name>
> ```
> then paste the relevant section below.

---

## 1. `identity_db` — additional users (manager + employees)

You already have an admin. These add one manager and two employees so you can test role-based screens (Employees list, PO approval, adjustments).

| Username | Password | Role |
|---|---|---|
| `manager1` | `ManagerPass123!` | MANAGER |
| `employee1` | `EmployeePass123!` | EMPLOYEE |
| `employee2` | `EmployeePass456!` | EMPLOYEE |

```sql
-- manager1 / ManagerPass123!
INSERT INTO users (username, password, email, role, created_at)
VALUES (
  'manager1',
  '$2b$10$gJHvnMWAAMahKIh.jcHA8uk.Vnuy7buJZ89b5rcqLigCvF0vlnOEe',
  'manager1@ims.local',
  'MANAGER',
  now()
);

-- employee1 / EmployeePass123!
INSERT INTO users (username, password, email, role, created_at)
VALUES (
  'employee1',
  '$2b$10$/00x9oMmDXTvy.zZJ1IVme37fbbLU.hyIkifSIaGZSnGq8hAOZdg2',
  'employee1@ims.local',
  'EMPLOYEE',
  now()
);

-- employee2 / EmployeePass456!
INSERT INTO users (username, password, email, role, created_at)
VALUES (
  'employee2',
  '$2b$10$QkiovcBXCWoE4xy22yX7Ve0nYM2lnzeEcZk/4pmmOOYDXN0J2GpKy',
  'employee2@ims.local',
  'EMPLOYEE',
  now()
);
```

---

## 2. `supplier_db` — suppliers

Table guess: `supplier` (columns: `name`, `contact_email`, `phone`, `address`, `created_at`)

```sql
INSERT INTO supplier (name, contact_email, phone, address, created_at) VALUES
('Acme Raw Materials Co.',   'sales@acmeraw.com',      '+1-555-0101', '100 Industrial Way, Newark, NJ',   now()),
('Bluepeak Packaging Ltd.',  'orders@bluepeak.com',    '+1-555-0102', '55 Harbor Rd, Long Beach, CA',      now()),
('Crestline Components',     'contact@crestline.com',  '+1-555-0103', '900 Foundry St, Pittsburgh, PA',    now()),
('Delta Consumables Inc.',   'info@deltaconsume.com',  '+1-555-0104', '12 Commerce Dr, Austin, TX',        now()),
('Everline Finished Goods',  'hello@everline.com',     '+1-555-0105', '77 Market Ave, Columbus, OH',       now());
```

---

## 3. `catalog_db` — products

Table guess: `product` (columns: `name`, `sku`, `product_type`, `price`, `quantity`, `supplier_id`, `created_at`, `updated_at`)

`supplier_id` values below assume the suppliers above got ids 1-5 in insertion order — check `supplier_db`'s actual ids first if this isn't a fresh table.

Quantities are deliberately mixed: some below the inventory-service's low-stock threshold (default 10) so you can test alert generation once a `StockAdjustment` or manual check runs.

```sql
INSERT INTO product (name, sku, product_type, price, quantity, supplier_id, created_at, updated_at) VALUES
('Steel Sheet 2mm',            'RM-STL-2MM',   'RAW_MATERIAL',  45.50,  120, 1, now(), now()),
('Aluminum Rod 10mm',          'RM-ALU-10MM',  'RAW_MATERIAL',  12.75,   8,  1, now(), now()),
('Corrugated Box - Medium',    'PKG-BOX-MED',  'PACKAGING',      1.20,  500, 2, now(), now()),
('Bubble Wrap Roll 50m',       'PKG-BUB-50M',  'PACKAGING',      8.99,    6,  2, now(), now()),
('Precision Bearing 8mm',      'CMP-BRG-8MM',  'RAW_MATERIAL',   3.40,   200, 3, now(), now()),
('Hex Bolt M6x20 (100pk)',     'CMP-BLT-M6',    'CONSUMABLE',     4.10,   15,  4, now(), now()),
('Industrial Lubricant 1L',    'CNS-LUB-1L',   'CONSUMABLE',     9.25,    5,  4, now(), now()),
('Assembled Widget A100',      'FG-WID-A100',  'FINISHED_GOOD', 89.00,   40, 5, now(), now()),
('Assembled Widget B200',      'FG-WID-B200',  'FINISHED_GOOD', 129.00,   3, 5, now(), now()),
('Shipping Pallet Standard',   'PKG-PAL-STD',  'PACKAGING',     18.00,   75, 2, now(), now());
```

---

## 4. `procurement_db` — purchase orders

Table guess: `purchase_order` (columns: `product_id`, `supplier_id`, `quantity`, `status`, `approved_by`, `approved_at`, `created_at`, `expected_delivery_date`)

`product_id` / `supplier_id` below assume the ids from the inserts above (products 1-10 in order, suppliers 1-5 in order) — verify before running.

```sql
-- Pending approval (test the approve/reject flow as MANAGER or ADMIN)
INSERT INTO purchase_order (product_id, supplier_id, quantity, status, approved_by, approved_at, created_at, expected_delivery_date)
VALUES (2, 1, 200, 'PENDING_APPROVAL', NULL, NULL, now(), NULL);

INSERT INTO purchase_order (product_id, supplier_id, quantity, status, approved_by, approved_at, created_at, expected_delivery_date)
VALUES (9, 5, 100, 'PENDING_APPROVAL', NULL, NULL, now(), NULL);

-- Already approved / sent
INSERT INTO purchase_order (product_id, supplier_id, quantity, status, approved_by, approved_at, created_at, expected_delivery_date)
VALUES (7, 4, 50, 'SENT', 'admin', now(), now(), CURRENT_DATE + INTERVAL '7 days');

-- Received (completed) order
INSERT INTO purchase_order (product_id, supplier_id, quantity, status, approved_by, approved_at, created_at, expected_delivery_date)
VALUES (4, 2, 300, 'RECEIVED', 'admin', now() - INTERVAL '5 days', now() - INTERVAL '10 days', CURRENT_DATE - INTERVAL '3 days');

-- Cancelled order
INSERT INTO purchase_order (product_id, supplier_id, quantity, status, approved_by, approved_at, created_at, expected_delivery_date)
VALUES (6, 4, 500, 'CANCELLED', 'admin', now() - INTERVAL '2 days', now() - INTERVAL '4 days', NULL);
```

---

## 5. `inventory_db` — stock alerts + adjustment history (optional)

Table guesses: `stock_alert` (`product_id`, `current_quantity`, `threshold`, `status`, `created_at`, `resolved_at`), `stock_adjustment` (`product_id`, `adjustment_type`, `quantity_changed`, `previous_quantity`, `new_quantity`, `adjusted_by`, `adjusted_at`, `notes`)

These mirror the low-stock products above (ids 2, 4, 7, 9).

```sql
INSERT INTO stock_alert (product_id, current_quantity, threshold, status, created_at, resolved_at)
VALUES
(2, 8, 10, 'OPEN', now(), NULL),
(4, 6, 10, 'OPEN', now(), NULL),
(7, 5, 10, 'OPEN', now(), NULL),
(9, 3, 10, 'OPEN', now(), NULL);

INSERT INTO stock_adjustment (product_id, adjustment_type, quantity_changed, previous_quantity, new_quantity, adjusted_by, adjusted_at, notes)
VALUES
(1, 'RESTOCK_RECEIVED', 50, 70, 120, 'employee1', now() - INTERVAL '1 day', 'Restock from supplier delivery'),
(3, 'MANUAL_CORRECTION', -20, 520, 500, 'manager1', now() - INTERVAL '2 days', 'Cycle count correction'),
(5, 'DAMAGE', -10, 210, 200, 'employee2', now() - INTERVAL '1 day', 'Water damage in storage');
```
