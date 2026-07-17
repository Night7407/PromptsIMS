# IMS — Authentication Flow & System Workflow

## 1. JWT Authentication Flow

### Step 1: Login (Identity Service)
- User hits `/auth/login` on the **Identity Service** (routed via Gateway).
- Identity Service validates credentials and issues a **signed JWT** containing claims:
  - `userId`
  - `role` (e.g. `ROLE_MANAGER`, `ROLE_ADMIN`)
- JWT is returned to the client.

### Step 2: Subsequent Requests (API Gateway)
- Client sends the JWT as `Authorization: Bearer <token>` to the **API Gateway**.
- The Gateway is the **only** component that validates the JWT (signature + expiry).
- On successful validation, the Gateway extracts `userId` and `role` from the claims and injects them as new headers:
  - `X-User-Id: <userId>`
  - `X-User-Role: <role>`
- The original `Authorization` header is **stripped** before forwarding downstream — raw JWTs never reach internal services.

### Step 3: Downstream Services (Trust Boundary)
- Each service (Catalog, Supplier, Inventory, Procurement, Chatbot, etc.) has a `HeaderAuthenticationFilter`.
- This filter reads `X-User-Id` / `X-User-Role` and populates the Spring Security context.
- Downstream services **trust these headers implicitly** — no re-verification of a JWT signature.
- Standard method security works as normal, e.g. `@PreAuthorize("hasRole('ADMIN')")`.

### Step 4: Service-to-Service Calls (Bypassing the Gateway)
- Internal calls (e.g. Inventory → Procurement via `RestTemplate` with `@LoadBalanced`) don't carry a user JWT.
- Instead, the calling service attaches a shared secret header:
  - `X-Internal-Service-Token: <shared-secret>`
- The receiving service's filter validates this token and sets the security context to `ROLE_SERVICE`.

### Hard Security Invariant
> **`ROLE_SERVICE` can never approve a Purchase Order.**

This is enforced at the business logic / method-security layer in the Procurement Service, not just at the Gateway — ensuring that even a compromised or misused internal token cannot trigger a PO approval.

### Why This Design
- Single point of JWT validation (Gateway) — no duplicated JWT parsing/secret management across 9 services.
- Explicit trust boundary: only the Gateway is internet-facing and performs cryptographic verification; everything inside the Docker network trusts propagated headers.

---

## 2. Overall System Workflow

### Startup & Service Registration
- **Eureka Server (8761)** starts first.
- All 8 other services register with Eureka on boot.
- **API Gateway (8080)** uses Eureka for service discovery — no hardcoded service URLs.

### Typical Manager Session
- **Login:** Manager → Gateway → Identity Service → JWT issued.
- **Browsing UI:** Manager accesses the Thymeleaf BFF (8090), which calls the Gateway (using the manager's JWT) to fetch catalog, stock, and supplier data, and renders HTML server-side.
- **Checking stock:** BFF → Gateway → Inventory Service → returns current quantities.

### Core Business Flow — Low Stock → Purchase Order
1. Manager notices low stock or creates a `StockAdjustment` (Inventory Service).
2. Manager initiates a Purchase Order (Procurement Service). Procurement calls:
   - Inventory Service — confirm product/supplier stock context
   - Catalog Service — product details
   - Supplier Service — supplier details
   (all synchronous, via `RestTemplate`)
3. PO enters `PENDING_APPROVAL` stage.
4. An **Admin** (never `ROLE_SERVICE`) approves → `approved_by` / `approved_at` set → status moves to `SENT`.
5. Supplier fulfills the order → goods received → Inventory Service stock quantity updated via a new `StockAdjustment`.
6. `SupplierPayment` entries are recorded against the PO as partial or full payments are made.

### Chatbot Flow
- Manager asks a question via BFF → Gateway → Chatbot Service (8086).
- Chatbot Service (Spring AI + Claude, tool-calling) queries other services (e.g. "what's low on stock?") using the same internal-service authenticated pattern.
- Response is synthesized and returned through the BFF.

### Cross-Cutting Principles
- Every hop after the Gateway relies on `X-User-Id` / `X-User-Role` (user-initiated) or `X-Internal-Service-Token` (service-initiated) — **raw JWTs never travel past the Gateway**.
- Each service owns its own PostgreSQL database — no shared schema, no cross-service DB joins; all cross-service data access goes through REST calls.

---

## 3. Summary Diagram (ASCII)

```
                        ┌─────────────────┐
                        │  Eureka Server   │  (8761)
                        └────────▲─────────┘
                                 │ registers
        ┌────────────────────────┼─────────────────────────┐
        │                        │                          │
   ┌─────────┐             ┌──────────┐              ┌────────────┐
   │ Identity │             │ Catalog  │   ...        │  Chatbot   │
   │  (8081)  │             │  (8082)  │              │   (8086)   │
   └────▲─────┘             └────▲─────┘              └─────▲──────┘
        │  X-User-Id/Role        │ X-User-Id/Role           │
        │  or X-Internal-Token   │                          │
   ┌────┴─────────────────────────────────────────────────┐
   │                  API Gateway (8080)                    │
   │      - Validates JWT signature/expiry                  │
   │      - Injects X-User-Id / X-User-Role headers          │
   │      - Strips Authorization header                     │
   └────▲─────────────────────────────────────────────────┘
        │  Authorization: Bearer <JWT>
   ┌────┴─────┐
   │  Manager  │  (via Thymeleaf BFF, 8090)
   └───────────┘
```
