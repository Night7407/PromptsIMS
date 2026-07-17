# Inventory Management System (IMS)

*Technical Documentation*

Spring Boot Microservices Architecture

## 1. Overview

The Inventory Management System is a manager-centric warehouse operations platform built on a Spring Boot microservices architecture. Unlike a customer-facing storefront, IMS is designed exclusively for Inventory Managers and Admins to control stock levels, approve procurement, and manage supplier relationships. There is no customer/sales-facing surface — stock changes are deliberate actions logged by managers, not the byproduct of orders.

### 1.1 Design Philosophy

- **Manager-owned stock changes.** Every quantity change traces back to a named adjustment logged by a manager (restock, correction, damage) — never an implicit side effect.
- **Simplicity over premature scale.** Synchronous REST (via Eureka + RestTemplate) was chosen over an async event bus. This is a conscious tradeoff, documented as a candidate for revisiting if throughput demands grow.
- **No redundant/derivable state.** Fields like a boolean `inStock` flag were deliberately excluded — they're derivable from quantity and would risk drifting out of sync.
- **Minimal relational complexity.** One supplier per product is modeled as a simple foreign key, not a many-to-many junction table, because the business rule doesn't require it.

## 2. Architecture

### 2.1 Service Map

```
                              +------------------+
                              |  Eureka Server   |
                              |   (port 8761)    |
                              +--------^---------+
                                       | service registry
              +------------------------+------------------------+
              |                        |                        |
    +---------+---------+    +---------+---------+    +---------+---------+
    |     web-app        |    |   API Gateway     |    |  chatbot-service   |
    |  (Thymeleaf BFF)   |--->|   (port 8080)      |    |   (port 8086)      |
    |    port 8090       |    |  JWT validation    |<---|  Spring AI (tools) |
    +--------------------+    |  X-User-Id/-Role   |    +---------+---------+
                              |  header injection  |              |
                              +----------+----------+              |
                                         | routes                  | read-only
        +--------------+------------------+------------------+     | tool calls
        v              v                  v                  v     |
 +--------------+ +--------------+ +--------------+ +------------------+
 |  identity-   | |  catalog-    | |  supplier-   | |  inventory- &    |
 |  service     | |  service     | |  service     | |  procurement-    |
 |  port 8081   | |  port 8082   | |  port 8083   | |  service          |
 |              | |              | |              | |  ports 8084/8085 |
 +------+-------+ +------+-------+ +------+-------+ +--------+---------+
        |                |                |                   |
   identity_db      catalog_db      supplier_db        inventory_db
                                                       procurement_db
                                              (PostgreSQL, one instance,
                                                separate databases)

  Inventory --RestTemplate + Eureka--> Catalog      (adjust quantity)
  Inventory --RestTemplate + Eureka--> Procurement  (draft PO on low stock)

  These two calls bypass the Gateway entirely -- service-to-service traffic
  is authenticated via a shared X-Internal-Service-Token, not a user JWT.
```

### 2.2 Tech Stack

| **Layer** | **Technology** |
|---|---|
| Language / Runtime | Java 25 (virtual threads enabled) |
| Framework | Spring Boot 3.5.16 |
| Cloud / Discovery | Spring Cloud 2025.0.3, Netflix Eureka |
| Gateway | Spring Cloud Gateway (WebFlux / reactive) |
| Security | Spring Security, JJWT (JSON Web Tokens) |
| Database | PostgreSQL (one instance, one database per service) |
| Inter-service comms | Synchronous REST via `@LoadBalanced` RestTemplate + Eureka |
| AI | Spring AI 1.1.x (Anthropic Claude provider) |
| Frontend | Thymeleaf (server-rendered, session-based auth) |
| Build tool | Maven |
| Testing | JUnit 5, Mockito, Testcontainers (PostgreSQL), WireMock |

### 2.3 Why Synchronous REST, Not Messaging

Kafka/event-bus architecture was evaluated and explicitly rejected for this version. The stock-alert-to-procurement flow is low-volume and needs a direct, traceable request/response (did the draft PO get created or not?) rather than eventual consistency. This is documented as a deliberate scoping decision, not an oversight — worth revisiting only if the system needs to handle high-frequency stock events across many warehouses.

## 3. Security Model

Authentication and authorization follow a single validation point design:

- The API Gateway is the only place a JWT signature is ever checked. It validates the token on every request except `/auth/**`, then injects `X-User-Id` and `X-User-Role` headers before forwarding downstream, and strips the original `Authorization` header.
- Downstream services trust those injected headers, via a lightweight `HeaderAuthenticationFilter` — they never re-parse a JWT themselves. This avoids duplicating JWT logic (and JWT bugs) across five services.
- Service-to-service calls (Inventory → Catalog, Inventory → Procurement) bypass the Gateway and carry a static `X-Internal-Service-Token` header instead, authenticating as a synthetic `ROLE_SERVICE`.
- One hard invariant: `ROLE_SERVICE` can never approve a purchase order. The `PUT /api/purchase-orders/{id}/approve` endpoint explicitly rejects it, even with a valid internal token — approval must always be a real user's action, tied to their JWT-derived identity.

### 3.1 Roles

| **Role** | **Description** |
|---|---|
| EMPLOYEE | Read-only access across products, suppliers, purchase orders, and inventory; can log stock adjustments. Cannot approve/cancel POs or perform product/supplier CRUD. |
| MANAGER | All application access except system configuration: product/supplier CRUD, PO approval/receipt/cancellation, adjustments, alerts. |
| ADMIN | Full access, including system configuration. Everything MANAGER can do, plus system-level configuration. |
| ROLE_SERVICE (synthetic, internal only) | Used only for service-to-service calls; never issued to a human user; never stored on the User entity. |

The `Role` enum has exactly three user-facing values — `EMPLOYEE`, `MANAGER`, `ADMIN`. `ROLE_SERVICE` is deliberately **not** part of that enum; it is a separate, internal-token-only mechanism.

### 3.2 Why This Design

Validating JWTs once at the Gateway, rather than in every service, means a signing-key rotation or a security bug fix happens in exactly one place. The tradeoff is that downstream services must fully trust the Gateway's network position — acceptable here since all internal traffic stays inside the same trusted network/cluster.

## 4. Services

### 4.1 Identity Service — port 8081, `identity_db`

Responsibility: sole authority for user accounts, credentials, and role assignment. The only service that issues JWTs.

**Entity: User**

| **Field** | **Description** |
|---|---|
| id, username | Primary key; unique username |
| password | BCrypt hash |
| email | Unique |
| role | `EMPLOYEE \| MANAGER \| ADMIN` |
| createdAt | Timestamp |

**Endpoints**

| **Endpoint** | **Access** | **Purpose** |
|---|---|---|
| `POST /auth/register` | Public | Create account, hash password, reject duplicate username/email |
| `POST /auth/login` | Public | Validate credentials, return JWT + role |

### 4.2 Catalog Service — port 8082, `catalog_db`

Responsibility: source of truth for product data and current stock quantity.

**Entity: Product**

| **Field** | **Description** |
|---|---|
| id, name, sku | sku is unique |
| productType | Enum: `RAW_MATERIAL`, `FINISHED_GOOD`, `PACKAGING`, `CONSUMABLE` |
| price, quantity | Current price and stock level |
| supplierId | Plain FK, cross-service reference — no JPA relation |
| createdAt, updatedAt | Timestamps |

**Endpoints**

| **Endpoint** | **Access** | **Purpose** |
|---|---|---|
| `POST /api/products` | MANAGER, ADMIN | Create product |
| `GET /api/products` | Any authenticated | List all |
| `GET /api/products/{id}` | Any authenticated | Get one (404 if missing) |
| `PUT /api/products/{id}` | MANAGER, ADMIN | Update |
| `DELETE /api/products/{id}` | MANAGER, ADMIN | Delete |
| `PATCH /api/products/{id}/quantity` | ROLE_SERVICE, MANAGER, ADMIN | Adjust quantity by a delta — called internally by Inventory |

*Design note: productType is a plain enum, not a separate Category entity — a junction table/relationship was evaluated and rejected as unnecessary complexity for a fixed, small set of types.*

### 4.3 Supplier Service — port 8083, `supplier_db`

Responsibility: supplier contact and relationship data.

**Entity: Supplier**

| **Field** | **Description** |
|---|---|
| id, name | Primary key, supplier name |
| contactEmail | Unique |
| phone, address | Contact details |
| createdAt | Timestamp |

**Endpoints**

| **Endpoint** | **Access** | **Purpose** |
|---|---|---|
| `POST /api/suppliers` | MANAGER, ADMIN | Create supplier |
| `GET /api/suppliers` | EMPLOYEE, MANAGER, ADMIN | List all |
| `GET /api/suppliers/{id}` | EMPLOYEE, MANAGER, ADMIN | Get one (404 if missing) |
| `PUT /api/suppliers/{id}` | MANAGER, ADMIN | Update |
| `DELETE /api/suppliers/{id}` | MANAGER, ADMIN | Delete |

*Design note: one supplier per product, modeled as a foreign key on Product — no junction table, since the business rule doesn't require many-to-many.*

### 4.4 Inventory Service — port 8084, `inventory_db`

Responsibility: owns stock alerts and the audit trail of stock adjustments. Holds no product data itself — quantity lives in Catalog.

**Entity: StockAlert**

| **Field** | **Description** |
|---|---|
| id, productId | Cross-service reference to Catalog's Product |
| currentQuantity, threshold | Snapshot values at alert creation |
| status | `OPEN \| RESOLVED` |
| createdAt, resolvedAt | Timestamps |

**Entity: StockAdjustment**

| **Field** | **Description** |
|---|---|
| id, productId | Cross-service reference |
| adjustmentType | `RESTOCK_RECEIVED`, `MANUAL_CORRECTION`, `DAMAGE` |
| quantityChanged | Can be negative |
| previousQuantity, newQuantity | Before/after snapshot |
| adjustedBy, adjustedAt, notes | Audit fields |

**Endpoints**

| **Endpoint** | **Access** | **Purpose** |
|---|---|---|
| `POST /api/inventory/adjustments` | EMPLOYEE, MANAGER, ADMIN | Log an adjustment; calls Catalog to update quantity; resolves any OPEN alert if RESTOCK_RECEIVED |
| `GET /api/inventory/adjustments/{productId}` | Any authenticated | Adjustment history for a product |
| `GET /api/inventory/alerts?status=OPEN` | EMPLOYEE, MANAGER, ADMIN | List alerts |
| `POST /api/inventory/alerts/check/{productId}` | MANAGER, ADMIN | Manually trigger the low-stock check (demo/testing) |

**The Stock-Alert Loop**

```
Inventory detects quantity < threshold (via CatalogClient)
        |
        v
Creates StockAlert (OPEN)
        |
        v
Calls Procurement: POST /api/purchase-orders/draft  (as ROLE_SERVICE)
        |
        v
Manager reviews & approves PO  (MANAGER or ADMIN, via own JWT -- never ROLE_SERVICE)
        |
        v
Goods received -> Manager logs StockAdjustment (RESTOCK_RECEIVED)
        |
        v
Quantity updated in Catalog, StockAlert resolved
```

*Failure handling: if the call to Procurement fails (network error or 5xx), the alert stays OPEN for retry rather than surfacing an error to the caller — a transient Procurement outage shouldn't block the manager's other work.*

### 4.5 Procurement Service — port 8085, `procurement_db`

Responsibility: purchase order lifecycle, from draft through approval to receipt.

**Entity: PurchaseOrder**

| **Field** | **Description** |
|---|---|
| id, productId, supplierId | Both plain FK, cross-service references |
| quantity | Order quantity |
| status | `PENDING_APPROVAL -> SENT -> RECEIVED / CANCELLED` |
| approvedBy, approvedAt | Set on approval |
| createdAt, expectedDeliveryDate | Timestamps |

**Endpoints**

| **Endpoint** | **Access** | **Purpose** |
|---|---|---|
| `POST /api/purchase-orders/draft` | ROLE_SERVICE, MANAGER, ADMIN | Create PO in `PENDING_APPROVAL` |
| `PUT /api/purchase-orders/{id}/approve` | MANAGER, ADMIN (blocks ROLE_SERVICE) | Approve → `SENT`, records approver + delivery date |
| `PUT /api/purchase-orders/{id}/receive` | MANAGER, ADMIN | Mark `RECEIVED` |
| `PUT /api/purchase-orders/{id}/cancel` | MANAGER, ADMIN | Mark `CANCELLED` |
| `GET /api/purchase-orders` | EMPLOYEE, MANAGER, ADMIN | List, optional status filter |
| `GET /api/purchase-orders/{id}` | EMPLOYEE, MANAGER, ADMIN | Get one (404 if missing) |

**Status Flow**

```
PENDING_APPROVAL --approve--> SENT --receive--> RECEIVED
        |
        +--cancel--> CANCELLED
```

### 4.6 Chatbot Service — port 8086

Responsibility: natural-language assistant over the system's live data, using Spring AI's tool-calling with the Claude API.

| **Endpoint** | **Access** | **Purpose** |
|---|---|---|
| `POST /api/chat` | Forwarded from the caller's web-app session | Accepts a question, lets Claude call read-only tools scoped to the caller's own role, returns a natural-language answer |

**Tools Exposed to the Model**

- `getOpenPurchaseOrders()` — Procurement
- `getLowStockAlerts()` — Inventory
- `getSupplierDetails(supplierId)` — Supplier
- `getProductDetails(productId)` — Catalog

*Key design principle: the chatbot never has broader access than the user asking the question. Every tool call forwards the same X-User-Id/X-User-Role headers the caller's own session carries — it's a convenience layer over the existing API surface, not a privileged backend.*

### 4.7 Web-App (Frontend) — port 8090

Responsibility: Thymeleaf-rendered Backend-for-Frontend (BFF). The only component that touches the browser directly; all other services are pure REST APIs.

| **Route** | **Purpose** |
|---|---|
| `GET/POST /login` | Login form → calls Identity via the Gateway → stores JWT server-side in HttpSession |
| `GET /logout` | Invalidates session |
| `GET /dashboard` | Summary counts: open alerts, pending POs |
| `GET /products`, `GET/POST /products/new` | List/create products (create hidden unless role is MANAGER or ADMIN) |
| `GET /suppliers` | List suppliers |
| `GET /purchase-orders`, `POST /purchase-orders/{id}/approve` | List POs; approve button shown to MANAGER or ADMIN |
| `GET /chat`, `POST /chat/send` | Chat UI, forwards to chatbot-service |

*Critical design point: the JWT is held server-side in the session, never in a browser-visible cookie or localStorage. The browser only ever holds a session cookie; web-app is the sole component that reads/attaches the JWT on outbound calls.*

## 5. Data Ownership

Each service owns exactly one database and is the sole writer to it. Cross-service references (supplierId on Product, productId/supplierId on PurchaseOrder, productId on StockAlert/StockAdjustment) are plain foreign-key-style fields, not JPA relationships — there is no cross-database join. Any need for related data crosses a service boundary via a REST call, not a database query.

| **Service** | **Database** | **Owns** |
|---|---|---|
| Identity | identity_db | Users, roles |
| Catalog | catalog_db | Products, stock quantity |
| Supplier | supplier_db | Suppliers |
| Inventory | inventory_db | Stock alerts, adjustment history |
| Procurement | procurement_db | Purchase orders |

## 6. Build & Deployment Order

Services must be built and verified in this order to avoid debugging multiple concerns (business logic, discovery, security) simultaneously:

1. **Eureka Server** — discovery backbone, no dependents
2. **Identity Service** — build and test register/login unsecured, then add JWT issuance
3. **Catalog Service and Supplier Service** — build unsecured, register with Eureka, then add header-based auth
4. **Inventory Service** — depends on Catalog + Procurement clients being reachable
5. **Procurement Service** — the ROLE_SERVICE-cannot-approve rule is tested explicitly here
6. **API Gateway** — routing first, JWT validation filter second (never both in the same pass)
7. **Chatbot Service** — requires all backend APIs already secured, since its tools call through the same auth model
8. **Web-App (Thymeleaf)** — requires Gateway's JWT layer fully working, since login depends on it

## 7. Testing Strategy

| **Test Type** | **Tooling** | **Scope** |
|---|---|---|
| Unit tests | JUnit 5 + Mockito | Service-layer logic, exception paths, mocked repositories/clients |
| Integration tests | `@SpringBootTest` + Testcontainers (PostgreSQL) | Full request → DB round trip per service, including role-based access checks via injected headers |
| Client/contract tests | WireMock | Inventory's calls to Catalog/Procurement, verifying the internal service-token header is sent correctly |
| Security-specific tests | Integration tests with header variations | Confirms ROLE_SERVICE is rejected on PO approval; confirms EMPLOYEE is blocked from write mutations; confirms MANAGER is blocked from ADMIN-only system-configuration actions |

*Open question (unresolved, tracked for future work): validation and load-testing strategy for synchronous inter-service calls at scale — relevant if the system ever needs to handle high request volume across the Inventory → Procurement → Catalog chain.*

## 8. Known Tradeoffs & Future Considerations

- **Synchronous REST over messaging** — simpler to build and debug now; would need re-evaluation if stock-alert volume grows significantly across many warehouses.
- **Static shared secret for service-to-service auth** (X-Internal-Service-Token) — adequate for a single-deployment/portfolio context; a production system would likely move to per-service credentials or mTLS.
- **No chat history persistence** in the chatbot service — each question is stateless; adding conversation memory is a natural v2 extension.
- **Spring AI 1.1.x / Spring Boot 3.5.16** were chosen deliberately over the newer Spring Boot 4.0 / Spring AI 2.0 pairing, since the latter reached GA only in mid-2026 and carries more ecosystem instability risk for this build. Revisit once that stack matures.
