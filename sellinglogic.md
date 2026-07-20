# Sales Module — Copilot Prompts

Two prompts: (1) backend module inside the existing **Procurement Service**, (2) frontend pages inside the existing **Thymeleaf BFF**. Deliver these to Copilot in order — backend first, since the BFF pages call its endpoints.

---

## Prompt 1: Sales Module (Procurement Service, port 8085)

**Role:** You are a senior Spring Boot engineer working on the IMS Procurement Service, a Spring Boot 3.5.x microservice using Java 25, Spring Security + JJWT, PostgreSQL with Flyway (`ddl-auto: validate`), and synchronous REST via `RestTemplate` with `@LoadBalanced`.

**Task:** Add a new **Sales** module to the existing Procurement Service, structurally parallel to the existing Purchase Order module but for outbound sales to customer companies. Do not create a new microservice, new port, or new database instance — this lives in the existing `ims_procurement` database as new tables.

**Context:**
- Existing Procurement Service already has a `purchaseorder` package (entity, repository, service, controller) at `/api/procurement/purchase-orders`, following the project's `buildForwardedHeaders` helper pattern, `RestTemplate.exchange()` for any downstream calls, and session-based role validation on write operations.
- Role model (locked, do not modify): `EMPLOYEE`, `MANAGER`, `ADMIN`. `ROLE_SERVICE` is not a Role enum value — it is asserted only via the `X-Internal-Service-Token` header for service-to-service calls.
- Role rules for this module (exact mirror of Purchase Order rules):
  - `EMPLOYEE`: can create sales orders (selects customer, product, quantity only). New orders go straight to `PENDING_APPROVAL`. Cannot approve, reject, send, or cancel any sales order.
  - `MANAGER` / `ADMIN`: full access — approve, reject, send, cancel sales orders; full CRUD on Customer records.
  - `EMPLOYEE`: read-only on Customer records (list/view only, no create/update/delete).
  - **Hard invariant**: `ROLE_SERVICE` can never approve a sales order. Enforce this directly in the service-layer business logic (not solely via annotations or gateway rules) — mirror exactly how PO-approval already blocks `ROLE_SERVICE` in this codebase.

**Entities:**
1. `Customer` — mirrors the existing `Supplier` entity's shape (id, name, contact fields, address, active flag, created/updated timestamps).
2. `SalesOrder` — mirrors the existing `PurchaseOrder` entity's shape: id, `customer_id` (FK), status enum (`PENDING_APPROVAL`, `APPROVED`, `REJECTED`, `SENT`, `CANCELLED`), `created_by`, `approved_by`, timestamps.
3. `SalesOrderItem` — mirrors `PurchaseOrderItem`: id, `sales_order_id` (FK), `product_id`, quantity, unit price at time of order.

**Endpoints:**
- `POST /api/procurement/customers` — ADMIN/MANAGER only
- `PUT /api/procurement/customers/{id}` — ADMIN/MANAGER only
- `DELETE /api/procurement/customers/{id}` — ADMIN/MANAGER only
- `GET /api/procurement/customers` — all roles; supports `?q=` forwarded/filtered as `?search=` per project convention
- `GET /api/procurement/customers/{id}` — all roles
- `POST /api/procurement/sales-orders` — all roles (EMPLOYEE and above); body takes `customerId`, `items[]` (productId, quantity); status forced to `PENDING_APPROVAL` server-side regardless of request body
- `GET /api/procurement/sales-orders` — all roles; supports `?status=` filter
- `GET /api/procurement/sales-orders/{id}` — all roles
- `PUT /api/procurement/sales-orders/{id}/approve` — MANAGER/ADMIN only; reject if actor is `ROLE_SERVICE` even if header spoofed
- `PUT /api/procurement/sales-orders/{id}/reject` — MANAGER/ADMIN only
- `PUT /api/procurement/sales-orders/{id}/send` — MANAGER/ADMIN only
- `PUT /api/procurement/sales-orders/{id}/cancel` — MANAGER/ADMIN only

**Database:**
- Provide Flyway migration SQL (new version file, e.g. `V{next}__create_sales_tables.sql`) for `customers`, `sales_orders`, `sales_order_items` tables, with appropriate FKs, NOT NULL constraints, and a status CHECK constraint on `sales_orders.status`.
- Do not modify `application.yml` datasource config — reuse the existing Procurement Service datasource; these are new tables in the same schema.
- Follow the same `ddl-auto: validate` convention: migrations are the source of truth, entities must match exactly.

**Output:** Full Java source for `Customer`, `SalesOrder`, `SalesOrderItem` entities; their repositories; `CustomerService`/`SalesOrderService` with role-validation and the `ROLE_SERVICE`-cannot-approve invariant; `CustomerController`/`SalesOrderController`; and the Flyway migration SQL file. Match existing package structure and naming conventions used by the `purchaseorder` package in this service.

---

## Prompt 2: Sales UI (Thymeleaf BFF, port 8090)

**Role:** You are a senior Spring Boot engineer working on the IMS Thymeleaf BFF, which holds JWTs server-side in `HttpSession` only and proxies to backend services via `RestTemplate` with forwarded headers.

**Task:** Add frontend pages for the new Sales module (Customers + Sales Orders), calling the Procurement Service's new `/api/procurement/customers` and `/api/procurement/sales-orders` endpoints through the Gateway, following the exact patterns already used for the existing Purchase Order pages in this BFF.

**Context:**
- Existing BFF has Purchase Order pages (list, detail, create form, approve/reject/send/cancel actions) that call Procurement Service via the Gateway using `buildForwardedHeaders`, session-based role checks to show/hide action buttons, and Thymeleaf templates under the project's existing `templates/` structure.
- Same role visibility rules apply: EMPLOYEE sees "Create Sales Order" but not approve/reject/send/cancel buttons; MANAGER/ADMIN see everything including Customer management.
- Customer CRUD forms are only rendered/reachable for MANAGER/ADMIN sessions — mirror how Supplier CRUD pages are currently gated in this BFF.

**Pages to add:**
1. `GET /sales-orders` — list view, status filter dropdown, "Create Sales Order" button (EMPLOYEE+)
2. `GET /sales-orders/{id}` — detail view with line items; approve/reject/send/cancel buttons conditionally rendered for MANAGER/ADMIN only
3. `GET /sales-orders/new` and `POST /sales-orders` — create form: customer dropdown (populated from `GET /api/procurement/customers`), product + quantity line items; EMPLOYEE+
4. `GET /customers` — list view, MANAGER/ADMIN only (redirect/403 page for EMPLOYEE)
5. `GET /customers/new`, `POST /customers`, `GET /customers/{id}/edit`, `POST /customers/{id}` — Customer CRUD forms, MANAGER/ADMIN only

**Navigation:** Add "Sales Orders" and "Customers" links to the existing nav/sidebar fragment, with the Customers link hidden for EMPLOYEE sessions (same pattern as any existing MANAGER/ADMIN-only nav items).

**Output:** Thymeleaf templates for all five page groups above, the corresponding `@Controller` (or additions to an existing controller) using `RestTemplate.exchange()` calls to the new Procurement Service endpoints, and the nav fragment update. Match existing Purchase Order page styling/layout conventions in this project exactly — same CSS classes, same page structure, same button/badge treatment for status values (extend the status badge logic to include Sales Order statuses).
