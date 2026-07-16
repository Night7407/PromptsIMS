ROLE: Act as a senior technical writer documenting a Spring Boot microservices project.

TASK: Generate a complete technical documentation file named
PROJECT-DOCUMENTATION.md for an Inventory Management System (IMS). Follow
the exact section structure and content below — do not omit any section,
table, or diagram.

CONTEXT:

Project summary: IMS is a manager-centric warehouse operations platform
(not a customer storefront) built on Spring Boot microservices. Only
Inventory Managers and Admins use it — there is no customer-facing surface.
Stock changes are always deliberate, logged manager actions.

Tech stack: Java 25 (virtual threads enabled), Spring Boot 3.5.16, Spring
Cloud 2025.0.3, Netflix Eureka, Spring Cloud Gateway (WebFlux/reactive),
Spring Security + JJWT, PostgreSQL (one instance, one database per
service), synchronous REST via @LoadBalanced RestTemplate + Eureka, Spring
AI 1.1.x (Anthropic Claude provider), Thymeleaf frontend, Maven, JUnit 5 +
Mockito + Testcontainers + WireMock for testing.

Services and ports: Eureka Server (8761), API Gateway (8080), Identity
(8081), Catalog (8082), Supplier (8083), Inventory (8084), Procurement
(8085), Chatbot (8086), Web-App/Thymeleaf (8090).

Databases: identity_db, catalog_db, supplier_db, inventory_db,
procurement_db — each owned by exactly one service, no cross-database
joins; cross-service references are plain FK-style fields, not JPA
relationships.

Security model: JWT is validated ONLY at the API Gateway. The Gateway
injects X-User-Id and X-User-Role headers downstream and strips the
original Authorization header. Downstream services trust those headers via
a HeaderAuthenticationFilter rather than re-parsing JWTs. Service-to-service
calls (Inventory -> Catalog, Inventory -> Procurement) bypass the Gateway
entirely and use a shared X-Internal-Service-Token header, authenticating
as a synthetic ROLE_SERVICE. Hard invariant: ROLE_SERVICE can never approve
a purchase order (PUT /api/purchase-orders/{id}/approve) — that endpoint
must always require a real ADMIN JWT.

Roles: INVENTORY_MANAGER (day-to-day operations), ADMIN (full control),
ROLE_SERVICE (synthetic, internal-only, never issued to a human).

Generate the document with these exact top-level sections, in this order:

1. Overview — project summary, and a "Design Philosophy" subsection
   covering: manager-owned stock changes, simplicity over premature scale
   (why sync REST not Kafka), no redundant/derivable state (e.g. no
   boolean inStock field), minimal relational complexity (FK not junction
   table for supplier-product).

2. Architecture —
   2.1 Service Map: an ASCII-art diagram (in a fenced code block) showing
   Eureka at the top, Gateway routing to the 5 backend services plus
   web-app and chatbot-service, each service's port and database, and a
   note that Inventory's calls to Catalog/Procurement bypass the Gateway
   and use the internal service token.
   2.2 Tech Stack: a markdown table (Layer | Technology) listing every
   item from the tech stack above.
   2.3 "Why Synchronous REST, Not Messaging" — explain Kafka was evaluated
   and rejected because the stock-alert-to-procurement flow is low-volume
   and needs traceable request/response, not eventual consistency.

3. Security Model — explain the single-validation-point design (bullet
   list of the 4 points in the security model context above), then a "3.1
   Roles" table (Role | Description) for all three roles, then a "3.2 Why
   This Design" paragraph on why centralizing JWT validation at the
   Gateway avoids duplicated security logic.

4. Services — one subsection per service (4.1 through 4.7), each
   following this exact pattern: a one-line "Responsibility" statement, an
   entity table (Field | Description) for every entity in that service, an
   endpoint table (Endpoint | Access | Purpose) for every REST endpoint,
   and any design notes as an italicized paragraph. Use these entities and
   endpoints exactly:

   4.1 Identity Service (8081, identity_db):
   Entity User: id, username (unique), password (BCrypt hash), email
   (unique), role (INVENTORY_MANAGER|ADMIN), createdAt.
   Endpoints: POST /auth/register (public), POST /auth/login (public,
   returns JWT + role).

   4.2 Catalog Service (8082, catalog_db):
   Entity Product: id, name, sku (unique), productType (enum:
   RAW_MATERIAL, FINISHED_GOOD, PACKAGING, CONSUMABLE), price, quantity,
   supplierId (plain FK, cross-service), createdAt, updatedAt.
   Endpoints: POST /api/products (ADMIN), GET /api/products (any auth),
   GET /api/products/{id} (any auth), PUT /api/products/{id} (ADMIN),
   DELETE /api/products/{id} (ADMIN), PATCH /api/products/{id}/quantity
   (ROLE_SERVICE, ADMIN).
   Design note: productType is a plain enum, not a Category entity —
   junction table rejected as unnecessary complexity.

   4.3 Supplier Service (8083, supplier_db):
   Entity Supplier: id, name, contactEmail (unique), phone, address,
   createdAt.
   Endpoints: POST /api/suppliers (ADMIN), GET /api/suppliers (ADMIN,
   INVENTORY_MANAGER), GET /api/suppliers/{id} (ADMIN,
   INVENTORY_MANAGER), PUT /api/suppliers/{id} (ADMIN), DELETE
   /api/suppliers/{id} (ADMIN).
   Design note: one supplier per product via FK on Product, no junction
   table.

   4.4 Inventory Service (8084, inventory_db):
   Entity StockAlert: id, productId, currentQuantity, threshold, status
   (OPEN|RESOLVED), createdAt, resolvedAt.
   Entity StockAdjustment: id, productId, adjustmentType
   (RESTOCK_RECEIVED, MANUAL_CORRECTION, DAMAGE), quantityChanged,
   previousQuantity, newQuantity, adjustedBy, adjustedAt, notes.
   Endpoints: POST /api/inventory/adjustments (INVENTORY_MANAGER, ADMIN),
   GET /api/inventory/adjustments/{productId} (any auth), GET
   /api/inventory/alerts?status=OPEN (INVENTORY_MANAGER, ADMIN), POST
   /api/inventory/alerts/check/{productId} (ADMIN).
   Include a subsection "The Stock-Alert Loop" with an ASCII flow diagram:
   Inventory detects low quantity via CatalogClient -> creates StockAlert
   (OPEN) -> calls Procurement POST /api/purchase-orders/draft as
   ROLE_SERVICE -> Manager approves PO as ADMIN (never ROLE_SERVICE) ->
   goods received -> Manager logs StockAdjustment (RESTOCK_RECEIVED) ->
   quantity updated in Catalog, alert resolved.
   Design note: if the Procurement call fails, the alert stays OPEN for
   retry rather than throwing to the caller.

   4.5 Procurement Service (8085, procurement_db):
   Entity PurchaseOrder: id, productId, supplierId (both plain FK),
   quantity, status (PENDING_APPROVAL, SENT, RECEIVED, CANCELLED),
   approvedBy, approvedAt, createdAt, expectedDeliveryDate.
   Endpoints: POST /api/purchase-orders/draft (ROLE_SERVICE,
   INVENTORY_MANAGER, ADMIN), PUT /api/purchase-orders/{id}/approve
   (ADMIN only, explicitly blocks ROLE_SERVICE), PUT
   /api/purchase-orders/{id}/receive (INVENTORY_MANAGER, ADMIN), PUT
   /api/purchase-orders/{id}/cancel (ADMIN), GET /api/purchase-orders
   (INVENTORY_MANAGER, ADMIN), GET /api/purchase-orders/{id}
   (INVENTORY_MANAGER, ADMIN).
   Include a small ASCII status-flow diagram: PENDING_APPROVAL --approve-->
   SENT --receive--> RECEIVED, with a branch --cancel--> CANCELLED.

   4.6 Chatbot Service (8086):
   Endpoint: POST /api/chat — accepts a question from the manager's
   session, lets Claude call read-only tools scoped to the caller's role,
   returns a natural-language answer.
   Tools exposed to the model: getOpenPurchaseOrders() [Procurement],
   getLowStockAlerts() [Inventory], getSupplierDetails(supplierId)
   [Supplier], getProductDetails(productId) [Catalog].
   Design note: the chatbot forwards the same X-User-Id/X-User-Role
   headers as the manager's session — it never has broader access than the
   manager asking.

   4.7 Web-App / Frontend (8090):
   Routes: GET/POST /login (stores JWT server-side in HttpSession, never
   in a cookie), GET /logout, GET /dashboard (alert/PO counts), GET
   /products + GET/POST /products/new (create hidden unless ADMIN), GET
   /suppliers, GET /purchase-orders + POST
   /purchase-orders/{id}/approve (approve button ADMIN-only), GET /chat +
   POST /chat/send.
   Design note: JWT lives only server-side in session; the browser never
   sees it.

5. Data Ownership — a paragraph on the one-database-per-service rule and
   no cross-database joins, followed by a table (Service | Database |
   Owns) for all five backend services.

6. Build & Deployment Order — a numbered list in this exact order: Eureka
   Server, Identity Service, Catalog + Supplier Service, Inventory
   Service, Procurement Service, API Gateway (routing then JWT security as
   separate passes), Chatbot Service, Web-App. Include a one-line reason
   next to each item for why it's positioned there.

7. Testing Strategy — a table (Test Type | Tooling | Scope) covering unit
   tests (JUnit5+Mockito), integration tests (@SpringBootTest +
   Testcontainers Postgres), client/contract tests (WireMock), and
   security-specific tests (role/header variation checks). End with a note
   that load-testing strategy for synchronous inter-service calls at scale
   is an open question for future work.

8. Known Tradeoffs & Future Considerations — a bullet list covering: sync
   REST over messaging as a scoping decision, static shared secret for
   service-to-service auth as adequate for now but not production-grade,
   no chat history persistence in the chatbot (stateless per request), and
   the deliberate choice of Spring AI 1.1.x/Spring Boot 3.5.16 over the
   newer 4.0/2.0 pairing due to ecosystem immaturity at time of writing.

OUTPUT FORMAT: A single valid Markdown file, using proper heading levels
(#, ##, ###), GitHub-flavored markdown tables, and fenced code blocks for
all ASCII diagrams. No content outside the file itself — no preamble, no
explanation of what you generated.