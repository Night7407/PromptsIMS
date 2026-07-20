# IMS Sales Module — Seed Data

Target: `ims_procurement` database, new tables (`customers`, `sales_orders`, `sales_order_items`).
Run **after** the Flyway migration (`V{next}__create_sales_tables.sql`) has been applied.

> **Placeholders to replace before running:**
> - `<PRODUCT_ID_1>`–`<PRODUCT_ID_4>` — real IDs from the `products` table (Catalog Service)
> - `<EMPLOYEE_USER_ID>` / `<MANAGER_USER_ID>` — real IDs from the `users` table (Identity Service), one EMPLOYEE and one MANAGER/ADMIN
> - IDs below assume UUID primary keys to match the rest of the project's convention. Swap to numeric IDs if your schema uses `BIGSERIAL`.

---

## Customers

| id | name | contact_name | email | phone | address | active |
|---|---|---|---|---|---|---|
| c1111111-1111-1111-1111-111111111111 | Acme Retail Group | Priya Nair | priya.nair@acmeretail.com | +1-555-0101 | 221 Market St, Springfield, IL | true |
| c2222222-2222-2222-2222-222222222222 | Blue Harbor Logistics | Tom Whitfield | tom.whitfield@blueharbor.com | +1-555-0102 | 48 Dockside Ave, Portland, OR | true |
| c3333333-3333-3333-3333-333333333333 | Crestline Manufacturing | Angela Ruiz | angela.ruiz@crestlinemfg.com | +1-555-0103 | 900 Industrial Pkwy, Dayton, OH | true |
| c4444444-4444-4444-4444-444444444444 | Dunwoody Hardware Co. | Marcus Lee | marcus.lee@dunwoodyhw.com | +1-555-0104 | 77 Elm St, Dunwoody, GA | true |
| c5555555-5555-5555-5555-555555555555 | Everline Foods | Sandra Okafor | sandra.okafor@everlinefoods.com | +1-555-0105 | 15 Harvest Rd, Fresno, CA | false |

```sql
INSERT INTO customers (id, name, contact_name, email, phone, address, active, created_at, updated_at) VALUES
('c1111111-1111-1111-1111-111111111111', 'Acme Retail Group',       'Priya Nair',    'priya.nair@acmeretail.com',       '+1-555-0101', '221 Market St, Springfield, IL',  true,  now(), now()),
('c2222222-2222-2222-2222-222222222222', 'Blue Harbor Logistics',   'Tom Whitfield', 'tom.whitfield@blueharbor.com',    '+1-555-0102', '48 Dockside Ave, Portland, OR',   true,  now(), now()),
('c3333333-3333-3333-3333-333333333333', 'Crestline Manufacturing', 'Angela Ruiz',   'angela.ruiz@crestlinemfg.com',    '+1-555-0103', '900 Industrial Pkwy, Dayton, OH', true,  now(), now()),
('c4444444-4444-4444-4444-444444444444', 'Dunwoody Hardware Co.',   'Marcus Lee',    'marcus.lee@dunwoodyhw.com',       '+1-555-0104', '77 Elm St, Dunwoody, GA',         true,  now(), now()),
('c5555555-5555-5555-5555-555555555555', 'Everline Foods',          'Sandra Okafor', 'sandra.okafor@everlinefoods.com', '+1-555-0105', '15 Harvest Rd, Fresno, CA',       false, now(), now());
```

---

## Sales Orders

| id | customer_id | status | created_by | approved_by |
|---|---|---|---|---|
| s1111111-1111-1111-1111-111111111111 | c1111111... (Acme Retail Group) | PENDING_APPROVAL | `<EMPLOYEE_USER_ID>` | NULL |
| s2222222-2222-2222-2222-222222222222 | c2222222... (Blue Harbor Logistics) | APPROVED | `<EMPLOYEE_USER_ID>` | `<MANAGER_USER_ID>` |
| s3333333-3333-3333-3333-333333333333 | c3333333... (Crestline Manufacturing) | SENT | `<EMPLOYEE_USER_ID>` | `<MANAGER_USER_ID>` |
| s4444444-4444-4444-4444-444444444444 | c4444444... (Dunwoody Hardware Co.) | REJECTED | `<EMPLOYEE_USER_ID>` | `<MANAGER_USER_ID>` |
| s5555555-5555-5555-5555-555555555555 | c1111111... (Acme Retail Group) | CANCELLED | `<EMPLOYEE_USER_ID>` | `<MANAGER_USER_ID>` |

```sql
INSERT INTO sales_orders (id, customer_id, status, created_by, approved_by, created_at, updated_at) VALUES
('s1111111-1111-1111-1111-111111111111', 'c1111111-1111-1111-1111-111111111111', 'PENDING_APPROVAL', '<EMPLOYEE_USER_ID>', NULL,                 now(), now()),
('s2222222-2222-2222-2222-222222222222', 'c2222222-2222-2222-2222-222222222222', 'APPROVED',          '<EMPLOYEE_USER_ID>', '<MANAGER_USER_ID>', now(), now()),
('s3333333-3333-3333-3333-333333333333', 'c3333333-3333-3333-3333-333333333333', 'SENT',              '<EMPLOYEE_USER_ID>', '<MANAGER_USER_ID>', now(), now()),
('s4444444-4444-4444-4444-444444444444', 'c4444444-4444-4444-4444-444444444444', 'REJECTED',          '<EMPLOYEE_USER_ID>', '<MANAGER_USER_ID>', now(), now()),
('s5555555-5555-5555-5555-555555555555', 'c1111111-1111-1111-1111-111111111111', 'CANCELLED',         '<EMPLOYEE_USER_ID>', '<MANAGER_USER_ID>', now(), now());
```

---

## Sales Order Items

| id | sales_order_id | product_id | quantity | unit_price |
|---|---|---|---|---|
| i1111111-... | s1111111... | `<PRODUCT_ID_1>` | 50 | 12.99 |
| i1111112-... | s1111111... | `<PRODUCT_ID_2>` | 20 | 34.50 |
| i2222221-... | s2222222... | `<PRODUCT_ID_1>` | 100 | 12.99 |
| i3333331-... | s3333333... | `<PRODUCT_ID_3>` | 30 | 8.25 |
| i3333332-... | s3333333... | `<PRODUCT_ID_4>` | 15 | 45.00 |
| i4444441-... | s4444444... | `<PRODUCT_ID_2>` | 10 | 34.50 |
| i5555551-... | s5555555... | `<PRODUCT_ID_1>` | 25 | 12.99 |

```sql
INSERT INTO sales_order_items (id, sales_order_id, product_id, quantity, unit_price) VALUES
('i1111111-1111-1111-1111-111111111111', 's1111111-1111-1111-1111-111111111111', '<PRODUCT_ID_1>', 50,  12.99),
('i1111112-1111-1111-1111-111111111112', 's1111111-1111-1111-1111-111111111111', '<PRODUCT_ID_2>', 20,  34.50),

('i2222221-2222-2222-2222-222222222221', 's2222222-2222-2222-2222-222222222222', '<PRODUCT_ID_1>', 100, 12.99),

('i3333331-3333-3333-3333-333333333331', 's3333333-3333-3333-3333-333333333333', '<PRODUCT_ID_3>', 30,  8.25),
('i3333332-3333-3333-3333-333333333332', 's3333333-3333-3333-3333-333333333333', '<PRODUCT_ID_4>', 15,  45.00),

('i4444441-4444-4444-4444-444444444441', 's4444444-4444-4444-4444-444444444444', '<PRODUCT_ID_2>', 10,  34.50),

('i5555551-5555-5555-5555-555555555551', 's5555555-5555-5555-5555-555555555555', '<PRODUCT_ID_1>', 25,  12.99);
```
