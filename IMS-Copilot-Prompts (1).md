# Inventory Management System — GitHub Copilot Prompt Set

**Stack:** Spring Boot 3.5.16 · Spring Cloud 2025.0.3 · Java 25 (virtual threads enabled) · PostgreSQL · Netflix Eureka · Spring Cloud Gateway (WebFlux) · JJWT · Spring AI (Anthropic) · Thymeleaf

**Ports:** Eureka 8761 · Gateway 8080 · Identity 8081 · Catalog 8082 · Supplier 8083 · Inventory 8084 · Procurement 8085 · Chatbot 8086 · WebApp 8090

**Databases (PostgreSQL, one instance, separate DBs):** `identity_db`, `catalog_db`, `supplier_db`, `inventory_db`, `procurement_db`

**Core security model:** JWT is validated **once, at the Gateway**. Downstream services trust two headers the Gateway injects after validation (`X-User-Id`, `X-User-Role`), plus a shared `X-Internal-Service-Token` header for service-to-service calls. Identity is the exception — it keeps its own `permitAll` config for `/auth/**` since login happens before a token exists. Approval of purchase orders must always require a real human JWT (MANAGER or ADMIN) — `ROLE_SERVICE` is explicitly blocked from that one endpoint.

**Roles:** The `Role` enum has exactly three user-facing values — `EMPLOYEE`, `MANAGER`, `ADMIN`. `EMPLOYEE` is read-only everywhere plus can log stock adjustments. `MANAGER` has all application access except system configuration (product/supplier CRUD, PO approval/receipt/cancellation, adjustments, alerts). `ADMIN` has everything `MANAGER` has, plus system configuration. `ROLE_SERVICE` is a separate, internal-token-only mechanism — it is **not** in the `Role` enum and is never stored on the `User` entity.

**Build order:** Identity → Catalog/Supplier → Inventory → Procurement → Eureka Server → API Gateway (routing) → API Gateway (security/JWT) → Chatbot → Frontend. Test each service fully before adding the next layer — never debug business logic and security at the same time.

---

---

## 0. Project Overview & Folder Structure

Give this to Copilot as the very first prompt in the workspace, before any service-specific prompt, so it understands the overall shape of the project and doesn't try to generate a monolith or guess at a different layout.

```
ROLE: Act as a senior Java architect setting up a multi-module Spring Boot
microservices project.

TASK: Do NOT generate any code yet. Just confirm your understanding of the
project structure below, and use it as fixed context for every prompt that
follows in this workspace.

CONTEXT:
- This is a microservices architecture, NOT a monolith. It is a single Git
  repository (monorepo) containing nine independent Maven projects, one per
  service. Each service has its own pom.xml, its own Spring Boot
  application, its own PostgreSQL database, and is independently runnable
  and independently deployable. There is no shared database and no shared
  Spring context between services.
- Root folder structure:

  ims/
  ├── eureka-server/
  │   └── src/main/java/com/ims/eureka/...
  ├── api-gateway/
  │   └── src/main/java/com/ims/gateway/...
  ├── identity-service/
  │   └── src/main/java/com/ims/identity/...
  ├── catalog-service/
  │   └── src/main/java/com/ims/catalog/...
  ├── supplier-service/
  │   └── src/main/java/com/ims/supplier/...
  ├── inventory-service/
  │   └── src/main/java/com/ims/inventory/...
  ├── procurement-service/
  │   └── src/main/java/com/ims/procurement/...
  ├── chatbot-service/
  │   └── src/main/java/com/ims/chatbot/...
  └── web-app/
      └── src/main/java/com/ims/webapp/...

- Each service's Java source follows the same layered package structure
  underneath its base package (e.g. com.ims.catalog):
    - entity/       — JPA entities
    - repository/    — Spring Data JPA repositories
    - dto/           — request/response records
    - service/       — business logic
    - controller/     — REST controllers
    - client/        — RestTemplate-based clients to other services (only
                        in services that call another service directly,
                        e.g. inventory-service, chatbot-service)
    - config/        — security filters, RestTemplate beans, etc.
    - exception/      — custom exceptions + @RestControllerAdvice handler
  Each service also has src/main/resources/application.yml, and
  src/test/java mirroring the same package structure for tests.
  web-app additionally has src/main/resources/templates/ for Thymeleaf HTML.
- These are NOT Maven modules of a single parent reactor POM — each service
  has an independent parent (spring-boot-starter-parent, per the shared
  version block below), so any one service can be built, run, and deployed
  on its own with `mvn spring-boot:run` from inside its own folder.
- Ports and databases per service are fixed and listed at the top of this
  document — use those exact values in every application.yml you generate.

OUTPUT FORMAT: Just confirm you understand this structure in one sentence.
Do not generate any files yet — wait for the next prompt.
```

---

## 0.1 Shared `pom.xml` version block

Paste this into every service's Maven setup, or give it to Copilot as the first prompt of each service so the BOM versions don't drift:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.16</version>
</parent>

<properties>
    <java.version>25</java.version>
    <spring-cloud.version>2025.0.3</spring-cloud.version>
</properties>
```

> Note: If any dependency pins an old `log4j2.version` explicitly, remove the override and let Spring Boot's BOM manage it — a hardcoded pin causes `IncompatibleClassChangeError` on newer Boot versions.

---

## 1. Identity Service

### Prompt 1 — Scaffold & entities
```
ROLE: Act as a senior Java developer building a Spring Boot 3.5.16 microservice.

TASK: Generate the complete identity-service (Java 25, Maven, base package
com.ims.identity), including entity, repository, service, DTOs, controller,
exception handling, and PostgreSQL configuration. Do NOT implement JWT
generation yet — that comes in a separate prompt.

CONTEXT:
- Stack: Spring Boot 3.5.16, Spring Cloud 2025.0.3, Java 25.
- This service is the sole authority for user accounts and roles in an
  Inventory Management System.
- Entity: User
    - id: Long (PK, auto-generated)
    - username: String (unique, not null)
    - password: String (BCrypt-hashed, not null)
    - email: String (unique, not null)
    - role: Role enum (EMPLOYEE, MANAGER, ADMIN) — stored as STRING, not ORDINAL
    - createdAt: LocalDateTime (set on creation)
- Database: PostgreSQL, database name identity_db, running on localhost:5432.
  Use spring.jpa.hibernate.ddl-auto=update for now.
- DTOs (Java records): RegisterRequestDto(username, password, email, role),
  UserResponseDto(id, username, email, role, createdAt),
  LoginRequestDto(username, password)
- Endpoints:
    - POST /auth/register -> validates uniqueness of username/email, hashes
      password with BCryptPasswordEncoder, saves User, returns UserResponseDto
    - POST /auth/login -> validates credentials against stored hash, returns
      UserResponseDto (no token yet)
- Custom exceptions: UserAlreadyExistsException, InvalidCredentialsException,
  UserNotFoundException, each mapped to appropriate HTTP status via a
  @RestControllerAdvice GlobalExceptionHandler returning a consistent
  ErrorResponseDto(timestamp, status, message, path).
- Enable virtual threads via spring.threads.virtual.enabled=true in
  application.yml.
- Use spring-boot-properties-migrator warnings as a guide if any
  application.yml property shows as deprecated — prefer current
  (non-deprecated) property names Copilot generates.

OUTPUT FORMAT: Provide complete, ready-to-paste Java files grouped by layer
(entity, repository, dto, service, controller, exception), each preceded by
its exact file path. Include the application.yml with datasource config,
server.port=8081, and spring.application.name=identity-service. Do not
include explanations between files, only brief file-path comments.
```

### Prompt 2 — JWT issuance (after Prompt 1 is tested and working)
```
ROLE: Act as a senior Java developer.

TASK: Add JWT token generation to the already-built identity-service.
Do not modify User entity or existing register/login business logic.

CONTEXT:
- Use JJWT (io.jsonwebtoken) 0.12.x.
- JwtUtil class: generateToken(username, role) using an HMAC secret from
  application.yml property jwt.secret, 1 hour expiry, claims include
  "role"; also provide extractUsername, extractRole, isTokenValid.
- Update POST /auth/login to return JwtResponseDto(token, username, role)
  instead of UserResponseDto.
- SecurityConfig: permitAll on /auth/**, sessionCreationPolicy STATELESS,
  csrf disabled, no other endpoints exist on this service so no further
  authorization rules needed.
- Add jwt.secret and jwt.expiration-ms as externalized properties in
  application.yml (placeholder secret value is fine).

OUTPUT FORMAT: Complete files only — JwtUtil, SecurityConfig, updated
AuthController and JwtResponseDto, updated application.yml snippet. No
narrative explanation.
```

### Prompt 3 — Tests
```
ROLE: Act as a senior Java developer writing tests.

TASK: Generate unit and integration tests for identity-service.

CONTEXT:
- Use JUnit 5, Mockito, and Spring Boot Test.
- Unit tests: UserServiceTest — mock UserRepository, cover
  registerUser (success + UserAlreadyExistsException for duplicate
  username/email) and login (success + InvalidCredentialsException).
- Integration test: AuthControllerIntegrationTest using
  @SpringBootTest(webEnvironment = RANDOM_PORT) + Testcontainers PostgreSQL
  module (org.testcontainers:postgresql), testing the full
  register -> login -> JWT returned flow via TestRestTemplate/MockMvc.
- JwtUtilTest: verify token generation, extraction of username/role, and
  that isTokenValid returns false for an expired or tampered token.

OUTPUT FORMAT: Complete test files with full imports, using
@DynamicPropertySource to point Testcontainers' Postgres at the datasource.
No prose, code only.
```

---

## 2. Catalog Service

### Prompt 1 — Entities, service, controller, unsecured
```
ROLE: Act as a senior Java developer building a Spring Boot 3.5.16 microservice.

TASK: Generate the complete catalog-service (Java 25, Maven, base package
com.ims.catalog) — entity, repository, DTOs, service, controller, exception
handling, PostgreSQL config, and a header-based security filter (no JWT
validation here — that's the Gateway's job).

CONTEXT:
- Stack: Spring Boot 3.5.16, Spring Cloud 2025.0.3, Java 25.
- Entity: Product
    - id: Long (PK)
    - name: String (not null)
    - sku: String (unique, not null)
    - productType: ProductType enum (RAW_MATERIAL, FINISHED_GOOD,
      PACKAGING, CONSUMABLE) — stored as STRING
    - price: BigDecimal
    - quantity: int (current stock level — this service is the source of
      truth for quantity)
    - supplierId: Long (plain FK-style field referencing Supplier in
      supplier-service — NOT a JPA @ManyToOne relationship, no junction
      table, cross-service reference only)
    - createdAt, updatedAt: LocalDateTime
- Database: PostgreSQL, database name catalog_db, localhost:5432.
- DTOs (records): ProductRequestDto(name, sku, productType, price, quantity,
  supplierId), ProductResponseDto(id, name, sku, productType, price,
  quantity, supplierId, createdAt, updatedAt), StockUpdateRequestDto(delta)
- Endpoints under /api/products:
    - POST / -> create (role MANAGER or ADMIN)
    - GET / -> list all (any authenticated role, including EMPLOYEE)
    - GET /{id} -> get one, throws ProductNotFoundException if missing (any
      authenticated role)
    - PUT /{id} -> update (MANAGER or ADMIN)
    - DELETE /{id} -> delete (MANAGER or ADMIN)
    - PATCH /{id}/quantity -> internal endpoint used by inventory-service to
      adjust quantity by a delta (positive or negative); accessible only to
      ROLE_SERVICE, MANAGER, or ADMIN
- Security: Do NOT implement JWT parsing. Instead add a
  HeaderAuthenticationFilter (a plain OncePerRequestFilter) that reads
  X-User-Id and X-User-Role headers (set upstream by the Gateway after it
  validated the JWT) and populates the SecurityContext with a
  UsernamePasswordAuthenticationToken carrying that role as a
  GrantedAuthority. If both headers are absent, also check for header
  X-Internal-Service-Token matching property internal.service.token from
  application.yml — if it matches, authenticate as ROLE_SERVICE. If neither
  is present, leave SecurityContext empty (request will be rejected by
  method-level @PreAuthorize / SecurityConfig rules).
- Use @PreAuthorize annotations reflecting the role rules above.
- Custom exception ProductNotFoundException mapped via
  @RestControllerAdvice to 404 with a consistent ErrorResponseDto.
- Enable virtual threads via spring.threads.virtual.enabled=true — beneficial
  here since services make blocking RestTemplate calls to each other.

OUTPUT FORMAT: Complete files grouped by layer with file paths as comments.
Include application.yml with server.port=8082,
spring.application.name=catalog-service, datasource config, and
internal.service.token placeholder property. No explanatory prose.
```

### Prompt 2 — Eureka registration
```
ROLE: Act as a senior Java developer.

TASK: Configure catalog-service as a Eureka client.

CONTEXT:
- Eureka server running at localhost:8761.
- server.port=8082, spring.application.name=catalog-service.
- Confirm the correct spring-cloud-starter-netflix-eureka-client dependency
  and version for Spring Cloud 2025.0.3.

OUTPUT FORMAT: Updated application.yml snippet and pom.xml dependency
entry. No prose.
```

### Prompt 3 — Tests
```
ROLE: Act as a senior Java developer writing tests.

TASK: Generate tests for catalog-service.

CONTEXT:
- ProductServiceTest (JUnit5 + Mockito): CRUD success paths,
  ProductNotFoundException on missing id, quantity-adjustment logic
  (delta applied correctly, rejects negative resulting quantity).
- ProductControllerIntegrationTest (@SpringBootTest RANDOM_PORT +
  Testcontainers PostgreSQL): verify that requests with X-User-Role=ADMIN and
  X-User-Role=MANAGER can both create a product (200/201 on POST), a request
  with X-User-Role=EMPLOYEE gets 403 on POST but 200 on GET, and a request
  with no headers gets 401/403 on every endpoint.
- HeaderAuthenticationFilterTest: verify SecurityContext is populated
  correctly for valid headers, for internal service token, and remains
  empty when neither is present.

OUTPUT FORMAT: Complete test files, imports included, no prose.
```

---

## 3. Supplier Service

### Prompt 1 — Entities, service, controller, unsecured
```
ROLE: Act as a senior Java developer building a Spring Boot 3.5.16 microservice.

TASK: Generate the complete supplier-service (Java 25, Maven, base package
com.ims.supplier) — entity, repository, DTOs, service, controller, exception
handling, PostgreSQL config, and the same HeaderAuthenticationFilter pattern
used in catalog-service.

CONTEXT:
- Stack: Spring Boot 3.5.16, Spring Cloud 2025.0.3, Java 25.
- Entity: Supplier
    - id: Long (PK)
    - name: String (not null)
    - contactEmail: String (unique, not null)
    - phone: String
    - address: String
    - createdAt: LocalDateTime
- Database: PostgreSQL, database name supplier_db, localhost:5432.
- DTOs (records): SupplierRequestDto(name, contactEmail, phone, address),
  SupplierResponseDto(id, name, contactEmail, phone, address, createdAt)
- Endpoints under /api/suppliers:
    - POST / -> create (MANAGER, ADMIN)
    - GET / -> list all (EMPLOYEE, MANAGER, ADMIN)
    - GET /{id} -> get one, throws SupplierNotFoundException if missing
      (EMPLOYEE, MANAGER, ADMIN)
    - PUT /{id} -> update (MANAGER, ADMIN)
    - DELETE /{id} -> delete (MANAGER, ADMIN)
- Reuse the identical HeaderAuthenticationFilter + SecurityConfig approach
  from catalog-service: trust X-User-Id/X-User-Role headers, fall back to
  X-Internal-Service-Token for ROLE_SERVICE.
- Custom exception SupplierNotFoundException -> 404 via
  @RestControllerAdvice with ErrorResponseDto.
- Enable virtual threads via spring.threads.virtual.enabled=true.

OUTPUT FORMAT: Complete files with file-path comments. application.yml with
server.port=8083, spring.application.name=supplier-service, datasource
config, internal.service.token property. No prose.
```

### Prompt 2 — Eureka registration
```
ROLE: Act as a senior Java developer.

TASK: Configure supplier-service as a Eureka client.

CONTEXT:
- Eureka server running at localhost:8761.
- server.port=8083, spring.application.name=supplier-service.

OUTPUT FORMAT: Updated application.yml snippet and pom.xml dependency
entry. No prose.
```

### Prompt 3 — Tests
```
ROLE: Act as a senior Java developer writing tests.

TASK: Generate tests for supplier-service, mirroring the structure used for
catalog-service.

CONTEXT:
- SupplierServiceTest (JUnit5 + Mockito): CRUD paths +
  SupplierNotFoundException case.
- SupplierControllerIntegrationTest (@SpringBootTest RANDOM_PORT +
  Testcontainers PostgreSQL): role-based access exactly as in
  catalog-service's tests (MANAGER and ADMIN can mutate, EMPLOYEE can only
  read, no headers = rejected).

OUTPUT FORMAT: Complete test files, imports included, no prose.
```

---

## 4. Inventory Service

### Prompt 1 — Entities, clients, service, controller
```
ROLE: Act as a senior Java developer building a Spring Boot 3.5.16 microservice.

TASK: Generate the complete inventory-service (Java 25, Maven, base package
com.ims.inventory) — entities, repositories, DTOs, service layer, controller,
a RestTemplate-based client to catalog-service and procurement-service,
exception handling, PostgreSQL config, and the HeaderAuthenticationFilter
pattern.

CONTEXT:
- Stack: Spring Boot 3.5.16, Spring Cloud 2025.0.3, Java 25.
- This service owns no Product data itself — quantity lives in
  catalog-service. It only tracks alerts and adjustment history, referencing
  productId as a plain Long (no JPA relationship, cross-service reference).
- Entity: StockAlert
    - id: Long (PK)
    - productId: Long (not null)
    - currentQuantity: int
    - threshold: int
    - status: AlertStatus enum (OPEN, RESOLVED) — stored as STRING
    - createdAt: LocalDateTime
    - resolvedAt: LocalDateTime (nullable)
- Entity: StockAdjustment
    - id: Long (PK)
    - productId: Long (not null)
    - adjustmentType: AdjustmentType enum (RESTOCK_RECEIVED,
      MANUAL_CORRECTION, DAMAGE) — stored as STRING
    - quantityChanged: int (can be negative)
    - previousQuantity: int
    - newQuantity: int
    - adjustedBy: String (username, taken from X-User-Id header)
    - adjustedAt: LocalDateTime
    - notes: String (nullable)
- Database: PostgreSQL, database name inventory_db, localhost:5432.
- RestTemplate config: a @LoadBalanced RestTemplate bean, plus a
  CatalogClient with getProduct(productId) and adjustQuantity(productId,
  delta) calling catalog-service's PATCH /api/products/{id}/quantity, and a
  ProcurementClient with createDraftPurchaseOrder(productId, supplierId,
  quantity) calling procurement-service's POST /api/purchase-orders/draft.
  Both clients must attach header X-Internal-Service-Token (value from
  application.yml property internal.service.token) via a
  ClientHttpRequestInterceptor named ServiceAuthInterceptor.
- Business logic: StockAlertService.checkAndCreateAlertIfNeeded(productId)
  fetches the product via CatalogClient, and if quantity < a configurable
  threshold and no OPEN alert already exists for that productId, creates a
  StockAlert and calls ProcurementClient.createDraftPurchaseOrder. On
  failure of the Procurement call (connection error or 5xx), log the error
  and leave the alert OPEN for retry rather than throwing to the caller.
- StockAdjustmentService.recordAdjustment(dto) saves a StockAdjustment,
  calls CatalogClient.adjustQuantity with the delta, and if the adjustment
  type is RESTOCK_RECEIVED, resolves any OPEN StockAlert for that productId.
- Endpoints under /api/inventory:
    - POST /adjustments -> record a StockAdjustment (EMPLOYEE, MANAGER,
      ADMIN — EMPLOYEE is explicitly allowed to log adjustments even though
      it can't do product/supplier CRUD or approve POs)
    - GET /adjustments/{productId} -> adjustment history for a product (any
      authenticated role)
    - GET /alerts?status=OPEN -> list alerts (EMPLOYEE, MANAGER, ADMIN)
    - POST /alerts/check/{productId} -> manually trigger
      checkAndCreateAlertIfNeeded (mainly for testing/demo; MANAGER, ADMIN)
- Reuse the HeaderAuthenticationFilter/SecurityConfig pattern from
  catalog-service.
- Custom exceptions: ProductFetchException (wraps failures calling
  catalog-service), AlertNotFoundException.
- Enable virtual threads via spring.threads.virtual.enabled=true — this
  service is heavy on blocking outbound calls (Catalog + Procurement), so
  virtual threads meaningfully help thread scalability here.

OUTPUT FORMAT: Complete files with file-path comments, grouped by layer
(entity, repository, dto, client, service, controller, config, exception).
application.yml with server.port=8084,
spring.application.name=inventory-service, datasource config,
internal.service.token property, and a configurable
inventory.low-stock-threshold property (default 10). No prose.
```

### Prompt 2 — Eureka registration
```
ROLE: Act as a senior Java developer.

TASK: Configure inventory-service as a Eureka client.

CONTEXT:
- Eureka server running at localhost:8761.
- server.port=8084, spring.application.name=inventory-service.

OUTPUT FORMAT: Updated application.yml snippet and pom.xml dependency
entry. No prose.
```

### Prompt 3 — Tests
```
ROLE: Act as a senior Java developer writing tests.

TASK: Generate tests for inventory-service, including client interaction
tests using WireMock.

CONTEXT:
- StockAlertServiceTest (JUnit5 + Mockito): mock CatalogClient and
  ProcurementClient — cover: alert created when quantity below threshold,
  no duplicate alert if one is already OPEN, alert stays OPEN (not thrown)
  if ProcurementClient call fails.
- StockAdjustmentServiceTest: mock CatalogClient — verify quantity delta is
  sent correctly, RESTOCK_RECEIVED resolves an existing OPEN alert,
  non-restock types do not touch alerts.
- CatalogClientIntegrationTest and ProcurementClientIntegrationTest using
  WireMock (com.github.tomakehurst:wiremock-jre8) to stub the downstream
  HTTP responses and verify the X-Internal-Service-Token header is sent.
- InventoryControllerIntegrationTest (@SpringBootTest RANDOM_PORT +
  Testcontainers PostgreSQL): role-based access checks as in prior services.

OUTPUT FORMAT: Complete test files, imports included, no prose.
```

---

## 5. Procurement Service

### Prompt 1 — Entities, service, controller
```
ROLE: Act as a senior Java developer building a Spring Boot 3.5.16 microservice.

TASK: Generate the complete procurement-service (Java 25, Maven, base
package com.ims.procurement) — entity, repository, DTOs, service,
controller, exception handling, PostgreSQL config, and the
HeaderAuthenticationFilter pattern with an explicit rule that ROLE_SERVICE
can never approve a purchase order.

CONTEXT:
- Stack: Spring Boot 3.5.16, Spring Cloud 2025.0.3, Java 25.
- Entity: PurchaseOrder
    - id: Long (PK)
    - productId: Long (not null, cross-service reference, no JPA relation)
    - supplierId: Long (not null, cross-service reference, no JPA relation)
    - quantity: int
    - status: POStatus enum (PENDING_APPROVAL, SENT, RECEIVED, CANCELLED)
      — stored as STRING
    - approvedBy: String (nullable, username)
    - approvedAt: LocalDateTime (nullable)
    - createdAt: LocalDateTime
    - expectedDeliveryDate: LocalDate (nullable)
- Database: PostgreSQL, database name procurement_db, localhost:5432.
- DTOs (records): DraftPurchaseOrderRequestDto(productId, supplierId,
  quantity), PurchaseOrderResponseDto(id, productId, supplierId, quantity,
  status, approvedBy, approvedAt, createdAt, expectedDeliveryDate),
  ApprovePurchaseOrderRequestDto(expectedDeliveryDate)
- Endpoints under /api/purchase-orders:
    - POST /draft -> creates a PurchaseOrder with status PENDING_APPROVAL.
      Accessible to ROLE_SERVICE, MANAGER, or ADMIN. (Not EMPLOYEE — drafting
      a PO is not part of EMPLOYEE's "log a stock adjustment" permission.)
    - PUT /{id}/approve -> sets status SENT, approvedBy (from X-User-Id
      header), approvedAt=now, expectedDeliveryDate. MANAGER or ADMIN — and
      must explicitly reject any request authenticated as ROLE_SERVICE
      (return 403) even if the internal token is valid, since approval must
      always be a real human action.
    - PUT /{id}/receive -> sets status RECEIVED. MANAGER, ADMIN. (Not
      EMPLOYEE — marking a PO RECEIVED is distinct from EMPLOYEE's stock
      adjustment permission on the inventory-service side.)
    - PUT /{id}/cancel -> sets status CANCELLED. MANAGER, ADMIN.
    - GET / -> list all, optional ?status= filter. EMPLOYEE, MANAGER, ADMIN.
    - GET /{id} -> get one, throws PurchaseOrderNotFoundException if
      missing. EMPLOYEE, MANAGER, ADMIN.
- Reuse the HeaderAuthenticationFilter pattern from catalog-service, but add
  an explicit @PreAuthorize or manual check in the approve endpoint that
  rejects an authentication whose authority is ROLE_SERVICE, returning 403
  with a clear message ("service accounts cannot approve purchase orders").
- Custom exceptions: PurchaseOrderNotFoundException,
  InvalidStatusTransitionException (e.g. approving an already-SENT order).
- Enable virtual threads via spring.threads.virtual.enabled=true.

OUTPUT FORMAT: Complete files with file-path comments. application.yml with
server.port=8085, spring.application.name=procurement-service, datasource
config, internal.service.token property. No prose.
```

### Prompt 2 — Eureka registration
```
ROLE: Act as a senior Java developer.

TASK: Configure procurement-service as a Eureka client.

CONTEXT:
- Eureka server running at localhost:8761.
- server.port=8085, spring.application.name=procurement-service.

OUTPUT FORMAT: Updated application.yml snippet and pom.xml dependency
entry. No prose.
```

### Prompt 3 — Tests
```
ROLE: Act as a senior Java developer writing tests.

TASK: Generate tests for procurement-service, with specific focus on the
ROLE_SERVICE-cannot-approve rule.

CONTEXT:
- PurchaseOrderServiceTest (JUnit5 + Mockito): status transition logic
  (draft -> approve -> receive), InvalidStatusTransitionException when
  approving a non-PENDING_APPROVAL order.
- PurchaseOrderControllerIntegrationTest (@SpringBootTest RANDOM_PORT +
  Testcontainers PostgreSQL):
    - X-Internal-Service-Token can hit POST /draft successfully
    - X-Internal-Service-Token gets 403 on PUT /{id}/approve
    - X-User-Role=ADMIN succeeds on PUT /{id}/approve
    - X-User-Role=MANAGER also succeeds on PUT /{id}/approve and on
      PUT /{id}/receive
    - X-User-Role=EMPLOYEE gets 403 on PUT /{id}/approve, 403 on
      PUT /{id}/receive, but 200 on GET /

OUTPUT FORMAT: Complete test files, imports included, no prose.
```

---

## 6. Eureka Server

```
ROLE: Act as a senior Java developer setting up service discovery.

TASK: Generate a complete Spring Cloud Netflix Eureka Server module
(eureka-server, Java 25, Maven, Spring Boot 3.5.16, Spring Cloud 2025.0.3).

CONTEXT:
- Standalone discovery server, not a client of itself.
- Main application class annotated @EnableEurekaServer.
- application.yml: server.port=8761,
  eureka.client.register-with-eureka=false,
  eureka.client.fetch-registry=false, spring.application.name=eureka-server.
- Add basic dashboard security note: for a portfolio project, leave the
  dashboard unauthenticated (do not add Spring Security to this module).

OUTPUT FORMAT: pom.xml, main application class, application.yml. No prose.
```

---

## 7. API Gateway — routing only

```
ROLE: Act as a senior Java developer setting up API routing.

TASK: Generate a complete Spring Cloud Gateway module (api-gateway, Java 25,
Maven, WebFlux-based, Spring Boot 3.5.16, Spring Cloud 2025.0.3) with routes
to all five backend services. Do NOT add security yet — routing only.

CONTEXT:
- Eureka client, registers with eureka-server at localhost:8761.
- application.yml routes (all via lb:// + Eureka service discovery):
    - Path /auth/** -> lb://identity-service
    - Path /api/products/** -> lb://catalog-service
    - Path /api/suppliers/** -> lb://supplier-service
    - Path /api/inventory/** -> lb://inventory-service
    - Path /api/purchase-orders/** -> lb://procurement-service
- server.port=8080, spring.application.name=api-gateway.
- Add a global CORS config permitting the future Thymeleaf frontend
  (localhost:8090) to call through the gateway.
- Note: Spring Cloud Gateway in the 2025.0.x train disables
  X-Forwarded-*/Forwarded header trust by default. Not needed for local
  development, but if added later, use property
  spring.cloud.gateway.server.webflux.trusted-proxies.

OUTPUT FORMAT: pom.xml, main application class, application.yml with full
route definitions, GlobalCorsConfig class. No prose.
```

## 8. API Gateway — Spring Security + JWT (the critical piece)

```
ROLE: Act as a senior Java developer implementing centralized authentication.

TASK: Add JWT validation to the already-built api-gateway as a global
WebFlux security filter. This is the ONLY place JWT signature/expiry gets
validated in the whole system. On success, forward the request downstream
with X-User-Id and X-User-Role headers added; on failure, short-circuit
with 401 before the request reaches any backend service.

CONTEXT:
- Gateway is WebFlux-based (reactive) — do NOT use servlet-based Spring
  Security classes (no OncePerRequestFilter). Use
  org.springframework.cloud.gateway.filter.GlobalFilter or a
  ReactiveSecurityContext-based approach.
- JwtValidator component: parses the token using JJWT with the same
  jwt.secret used by identity-service (externalize as application.yml
  property jwt.secret — must match identity-service's value exactly),
  extracts username and role, returns false/throws on invalid signature or
  expiry.
- AuthenticationGlobalFilter (implements GlobalFilter, Ordered):
    - Skip validation entirely for paths matching /auth/** (login/register
      are public).
    - For all other paths, read Authorization: Bearer <token> header. If
      missing or invalid -> respond 401 with a JSON body
      {"error": "Unauthorized"} and complete the exchange, do not proceed
      to the route.
    - If valid, mutate the request to add headers X-User-Id=<username> and
      X-User-Role=<role>, then continue the filter chain.
    - Also strip the original Authorization header before forwarding, since
      downstream services no longer need it (they trust the injected
      headers instead).
- Add a second, separate concern: internal service-to-service calls
  (inventory-service calling procurement-service directly, not through the
  gateway) do NOT go through this filter at all, since they bypass the
  gateway entirely via Eureka load-balanced RestTemplate. Confirm in a code
  comment that this filter only applies to gateway-routed traffic.
- application.yml: add jwt.secret property (same value as identity-service).

OUTPUT FORMAT: Complete files — JwtValidator, AuthenticationGlobalFilter,
updated application.yml snippet. Include a short code comment above the
filter class explaining the header-trust model for future reference. No
additional prose outside code comments.
```

---

## 9. Chatbot Service (Spring AI)

```
ROLE: Act as a senior Java developer integrating an LLM assistant.

TASK: Generate a complete Spring Boot 3.5.16 microservice (chatbot-service,
Java 25, Maven, base package com.ims.chatbot) using Spring AI's Anthropic
Claude integration with function/tool calling, exposing a chat endpoint that
the frontend can call on behalf of any logged-in user (EMPLOYEE, MANAGER, or
ADMIN).

CONTEXT:
- Dependency: spring-ai-anthropic-spring-boot-starter (Spring AI Anthropic
  starter, matching a Spring AI release compatible with Spring Boot 3.5.16).
- application.yml: spring.ai.anthropic.api-key from environment variable
  ANTHROPIC_API_KEY, spring.ai.anthropic.chat.options.model=claude-sonnet-4-6.
  server.port=8086, spring.application.name=chatbot-service, Eureka client.
- Define Spring AI @Tool-annotated methods (or FunctionCallback beans,
  whichever is current for the Spring AI version) that map to existing
  backend REST calls via a @LoadBalanced RestTemplate:
    - getOpenPurchaseOrders(): calls GET lb://procurement-service/api/purchase-orders?status=PENDING_APPROVAL
    - getLowStockAlerts(): calls GET lb://inventory-service/api/inventory/alerts?status=OPEN
    - getSupplierDetails(supplierId): calls GET lb://supplier-service/api/suppliers/{id}
    - getProductDetails(productId): calls GET lb://catalog-service/api/products/{id}
  Each tool method must forward the caller's identity by attaching
  X-User-Id and X-User-Role headers taken from the incoming chat request
  (passed in from the frontend, which stores them from the caller's
  session) — the chatbot must never have broader access than the user
  asking the question.
- Endpoint: POST /api/chat with ChatRequestDto(message, userId, userRole) ->
  ChatResponseDto(reply). Inside the controller, build a ChatClient request
  with the available tools registered, execute, and return the model's
  final natural-language answer.
- Do not persist chat history for this version — stateless per-request only.
- Basic exception handling: if a downstream tool call fails (5xx or timeout),
  the tool method should return a short error string to the model rather
  than throwing, so Claude can gracefully tell the manager the data wasn't
  available.
- Enable virtual threads via spring.threads.virtual.enabled=true.

OUTPUT FORMAT: Complete files — ChatController, ChatToolService (or
FunctionCallback beans per current Spring AI API), DTOs, RestTemplate config,
application.yml. Add file-path comments. No prose outside brief code
comments explaining the identity-forwarding requirement.
```

---

## 10. Frontend Service (Thymeleaf BFF)

```
ROLE: Act as a senior Java developer building a server-rendered frontend.

TASK: Generate a complete Spring Boot 3.5.16 MVC (Servlet-based, NOT
WebFlux) microservice (web-app, Java 25, Maven, base package com.ims.webapp)
using Thymeleaf, acting as a Backend-for-Frontend that authenticates users
(EMPLOYEE, MANAGER, or ADMIN) via session and calls backend services through
the api-gateway.

CONTEXT:
- Eureka client, but calls other services THROUGH api-gateway
  (lb://api-gateway) rather than directly, so it exercises the same JWT
  validation path as any other client.
- Session-based auth: LoginController with GET /login (renders form) and
  POST /login (calls api-gateway's /auth/login, on success stores the
  returned JWT and role in HttpSession — never in a cookie or JS-visible
  storage). LogoutController invalidates the session.
- SessionAuthInterceptor (HandlerInterceptor): for any request to a page
  requiring auth, checks HttpSession for a valid JWT; if absent, redirects
  to /login.
- RestTemplateConfig: a @LoadBalanced RestTemplate with a
  ClientHttpRequestInterceptor that pulls the JWT out of the current
  request's HttpSession and attaches it as an Authorization: Bearer header
  on every outgoing call to api-gateway.
- Pages/controllers (Thymeleaf templates, minimal styling is fine — focus on
  functionality):
    - DashboardController: GET /dashboard -> shows counts of open alerts
      and pending POs (calls inventory-service and procurement-service
      through the gateway)
    - ProductController: GET /products -> list, GET /products/new + POST
      /products -> create (MANAGER or ADMIN — hide the "new" button in the
      template unless session role is MANAGER or ADMIN)
    - SupplierController: GET /suppliers -> list
    - PurchaseOrderController: GET /purchase-orders -> list with an
      "Approve" button visible for MANAGER or ADMIN role, POST
      /purchase-orders/{id}/approve
    - ChatController: GET /chat (renders a simple chat UI), POST
      /chat/send -> forwards message + session userId/role to
      chatbot-service's /api/chat, renders the reply
- Templates: use Thymeleaf fragments for a shared layout (header/nav showing
  the logged-in username and role, logout link).
- Enable virtual threads via spring.threads.virtual.enabled=true.

OUTPUT FORMAT: Complete files — controllers, config, DTOs, and .html
templates under src/main/resources/templates. File-path comments before
each file. application.yml with server.port=8090,
spring.application.name=web-app. No prose outside code comments.
```

---

## Reminders before you start

1. **Build and test strictly top-to-bottom** — Identity → Catalog/Supplier → Inventory → Procurement → Eureka → Gateway (routing) → Gateway (security) → Chatbot → Frontend.
2. **`jwt.secret` must be identical** across identity-service and api-gateway — the single most common source of "valid login but every downstream call gets 401."
3. **Test the ROLE_SERVICE-can't-approve rule explicitly** the moment procurement-service is built — it's the one hard security invariant in the system.
4. **Watch your GitHub Copilot AI Credit usage** as you go (VS Code Status Bar → Copilot icon) — pick the default/included model for repetitive CRUD scaffolding, and reserve a stronger model for the Gateway JWT filter and the Chatbot service, where mistakes are more expensive to debug.
5. **Lombok, if used, must be 1.18.36+** for JDK 21+/25 compatibility.
