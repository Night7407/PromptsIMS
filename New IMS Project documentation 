# Inventory Management System (IMS)
## Technical Architecture & Design Documentation

---

## 1. Purpose & Scope

The Inventory Management System (IMS) is a Spring Boot microservices platform built for internal use by **Employees, Managers, and Admins**. It is not customer-facing — there is no storefront, cart, or public catalog. The system exists to manage stock levels, procurement, and supplier relationships for a single organization.

---

## 2. Technology Stack

| Layer | Technology |
|---|---|
| Language / Runtime | Java 25 (virtual threads enabled) |
| Framework | Spring Boot 3.5.x |
| Cloud / Discovery | Spring Cloud 2025.0.3, Netflix Eureka |
| AI | Spring AI 1.1.x, Anthropic Claude (tool-calling) |
| Database | PostgreSQL (one instance per service) |
| Inter-service calls | RestTemplate with `@LoadBalanced` (synchronous only — no message broker) |
| Frontend | Thymeleaf (BFF pattern) |

**Deliberately avoided:** Spring Boot 4.0 and Spring AI 2.0 — ruled out for ecosystem immaturity at time of build, not preference.

---

## 3. Service Inventory

| # | Service | Port | Responsibility |
|---|---|---|---|
| 1 | Eureka Server | 8761 | Service discovery |
| 2 | API Gateway | 8080 | Entry point, JWT validation, header injection |
| 3 | Identity Service | 8081 | Authentication, user/role management |
| 4 | Catalog Service | 8082 | Product master data |
| 5 | Supplier Service | 8083 | Supplier records |
| 6 | Inventory Service | 8084 | Stock levels, stock adjustments |
| 7 | Procurement Service | 8085 | Purchase orders, approvals, payments |
| 8 | Chatbot Service | 8086 | AI assistant (Spring AI + Claude) |
| 9 | Thymeleaf BFF | 8090 | Web frontend |

Each service owns a dedicated PostgreSQL database — no shared schemas.

```
                        ┌─────────────────┐
                        │  Eureka Server   │
                        │      :8761       │
                        └────────▲─────────┘
                                 │ registers
        ┌────────────────────────────────────────────┐
        │                                              │
┌───────┴───────┐                              ┌───────┴───────┐
│  API Gateway  │◄──── JWT validated here ─────►│  Thymeleaf BFF │
│     :8080     │                               │     :8090      │
└───────┬───────┘                               └────────────────┘
        │  injects X-User-Id / X-User-Role
        ▼
┌──────────────────────────────────────────────────────────────┐
│  Identity  │  Catalog  │  Supplier  │  Inventory  │ Procurement│
│   :8081    │   :8082   │   :8083    │    :8084    │   :8085    │
└──────────────────────────────────────────────────────────────┘
        ▲
        │  X-Internal-Service-Token (bypasses Gateway)
        │
┌───────┴───────┐
│   Chatbot     │
│     :8086     │
└───────────────┘
```

---

## 4. Security Model

**JWT is validated at the Gateway only.** Downstream services never see or validate a JWT — they trust two headers injected by the Gateway after successful validation:

- `X-User-Id`
- `X-User-Role`

Each downstream service enforces authorization via `HeaderAuthenticationFilter`, which reads these headers and applies role checks per endpoint.

**Service-to-service calls bypass the Gateway** and authenticate using a shared `X-Internal-Service-Token`, presenting as `ROLE_SERVICE`.

### 4.1 Role Model

| Role | Description |
|---|---|
| `ROLE_EMPLOYEE` | Day-to-day operational user |
| `ROLE_MANAGER` | Operational + approval authority |
| `ROLE_ADMIN` | Full administrative authority |
| `ROLE_SERVICE` | Internal service identity for service-to-service calls |

Manager and Admin permissions are additive — each higher role inherits everything below it.

### 4.2 Permission Matrix

| Capability | Employee | Manager | Admin | Service |
|---|:---:|:---:|:---:|:---:|
| View inventory / catalog | ✅ | ✅ | ✅ | ✅ |
| Record stock adjustments | ✅ | ✅ | ✅ | — |
| Create purchase order | ✅ | ✅ | ✅ | — |
| **Approve purchase order** | ❌ | ✅ | ✅ | ❌ |
| Manage suppliers | ❌ | ✅ | ✅ | — |
| Manage supplier payment terms | ❌ | ❌ | ✅ | — |
| Manage user accounts | ❌ | ❌ | ✅ | — |
| System configuration | ❌ | ❌ | ✅ | — |

### 4.3 Hard Security Invariants

- `ROLE_SERVICE` can **never** approve a purchase order.
- `ROLE_EMPLOYEE` can **never** approve a purchase order — only `ROLE_MANAGER` or `ROLE_ADMIN` may.
- These checks are enforced in Procurement Service business logic, not just at the Gateway, so a compromised or misconfigured header still cannot authorize an approval outside policy.

---

## 5. Entity Design Highlights

- **Product** — one supplier per product (FK, not many-to-many); `ProductType` enum replaces a separate Category entity; stock quantity is the single source of truth (no redundant `InStock` boolean).
- **StockAdjustment** — records manager/employee-initiated stock changes with a reason-code enum and a `manager_id` audit FK.
- **PurchaseOrder** — includes `approved_by` and `approved_at`; lifecycle includes a `PENDING_APPROVAL` stage prior to `SENT`.
- **SupplierPayment** — dedicated entity supporting partial B2B payments against a purchase order.

---

## 6. Inter-Service Communication

All service-to-service communication is **synchronous**, via `RestTemplate` with `@LoadBalanced` and Eureka-based discovery. There is no message broker or event bus in this system — this was a deliberate simplicity choice for a manager-centric internal tool of this scale.

Example: Inventory Service calls Procurement Service directly when stock falls below a reorder threshold, using the internal service token.

---

## 7. AI Chatbot Feature

The Chatbot Service uses **Spring AI 1.1.x** with **Anthropic Claude** via tool-calling, rather than a standalone ML forecasting model. This was chosen because:

- It works without historical sales data (no cold-start problem).
- It demonstrates the security architecture end-to-end (JWT → Gateway → header propagation → internal service token) via live tool calls into other services.

---

## 8. Build Roadmap

| Phase | Scope |
|---|---|
| Phase 1 | Architecture lock-in (complete) |
| Phase 2 | Identity, Catalog, Supplier services |
| Phase 3 | Inventory Service (StockAdjustment, RestTemplate client to Procurement) |
| Phase 4 | Procurement Service (approval workflow, SupplierPayment) |
| Phase 5 | Hardening, Docker, integration tests |

---

## 9. Design History Notes

- The system was originally scoped as customer-facing (with a Sales Service) and was deliberately pivoted mid-project to a manager-centric internal tool; the Sales Service was removed entirely.
- The role model evolved from a single Inventory Manager role → a proposal to remove Admin entirely → the current three-tier **Employee / Manager / Admin** model with `ROLE_SERVICE` for internal calls.
