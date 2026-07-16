# Inventory Management System
### Microservices Architecture & Entity Design Document — Manager-Centric Edition

---

## 1. Overview

The Inventory Management System (IMS) is a microservices-based backend built around an **Inventory Manager**, not a customer storefront. The Manager monitors stock levels, makes manual stock adjustments (damage, correction, goods received), and orders directly from Suppliers. There is no customer-facing sales flow — stock changes are driven by the Manager rather than by customer purchases.

When stock for a product falls below its defined reorder level, the system generates a `StockAlert` and Procurement auto-drafts a `PurchaseOrder`. The Manager reviews and approves the `PurchaseOrder` before it is sent to the Supplier. On delivery, the Manager logs a `StockAdjustment` to bring stock back up, and separately logs `SupplierPayment`s as the PO gets paid.

The system is composed of **7 services**: 2 infrastructure services (Eureka Server, API Gateway) and 5 domain services, communicating synchronously over REST using Spring Cloud Eureka for service discovery and RestTemplate (`@LoadBalanced`) for inter-service calls. There is no message broker / event bus in this design — all coordination is synchronous. The customer-facing Sales service from the earlier storefront-oriented design has been dropped.

---

## 2. Tech Stack

- Java 17, Spring Boot 3.2.5, Spring Cloud 2023.0.1
- Spring Cloud Gateway (WebFlux) — API Gateway, single entry point
- Netflix Eureka — service discovery / registry
- RestTemplate (`@LoadBalanced`) — synchronous inter-service communication
- JJWT 0.12.5 — centralized JWT authentication at the Gateway
- Spring Data JPA + MySQL/PostgreSQL — database-per-service pattern

---

## 3. Architecture Diagram

The Inventory Manager (via web/mobile client) authenticates through the API Gateway and interacts directly with Inventory, Catalog, and Procurement services. There is no customer traffic. Inventory calls Procurement synchronously when a stock alert is triggered; the Manager approves purchase orders before they go to the Supplier.

```
                    +-----------------+
                    |  Eureka Server  |  (8761)
                    +--------^--------+
        +--------------------+--------------------+
+-------v------+    +--------v--------+   +-------v------+
| API Gateway   |--->|  Identity Svc   |   | Catalog Svc  |
| (JWT auth)    |    | (Manager/Admin) |   |  (Product)   |
+-------^-------+    +-----------------+   +--------------+
        |
  Inventory Manager
   (Web/Mobile)
        |
        |        +------------------+
        +------->|  Inventory Svc   |
                 | (Stock, Alert,   |
                 |  Adjustment)     |
                 +--------+---------+
                          | RestTemplate call (on alert)
                 +--------v---------+   +--------------+
                 | Procurement Svc  |-->| Supplier Svc |
                 | (PO + approval + |   |              |
                 |  SupplierPayment)|   |              |
                 +------------------+   +--------------+
```

**Stock alert & approval flow:** Inventory detects `quantity < reorder_level` → creates a `StockAlert` (status `OPEN`) → calls Procurement to auto-draft a `PurchaseOrder` (status `PENDING_APPROVAL`) → Manager reviews and approves it (status `SENT`) → goods arrive from Supplier → Manager logs a `StockAdjustment` (reason `RESTOCK_RECEIVED`) which updates `Stock.quantity` and resolves the `StockAlert`. Payments against the PO are tracked separately via `SupplierPayment`.

---

## 4. Services & Entities

### 4.1 Eureka Server (Infrastructure)
**Port:** 8761 | No business entities

Central service registry. All domain services and the Gateway register here on startup; enables service-name-based lookups instead of hardcoded host:port.

### 4.2 API Gateway (Infrastructure)
**Port:** 8080 (typical) | No business entities

Single entry point for all client traffic. Performs centralized JWT authentication and routes requests to the appropriate downstream service via Eureka-based load balancing. Built with Spring Cloud Gateway (WebFlux).

### 4.3 Identity Service
**Owns:** User authentication & accounts

Handles registration, login, and JWT issuance. The Gateway's `JwtAuthenticationFilter` validates tokens issued by this service.

**Entity: User**

| Field | Type | Notes |
|---|---|---|
| user_id | Long (PK) | Auto-generated |
| username | String | Unique |
| email | String | Unique |
| password_hash | String | BCrypt hashed, never returned in responses |
| role | Enum | `INVENTORY_MANAGER`, `ADMIN` |
| created_at | Timestamp | |

### 4.4 Catalog Service
**Owns:** Product catalog

Manages product listings used by Inventory and Procurement for reference.

**Entity: Product**

| Field | Type | Notes |
|---|---|---|
| product_id | Long (PK) | Auto-generated |
| name | String | |
| description | String | |
| product_type | Enum | e.g. ELECTRONICS, GROCERY, APPAREL |
| price | BigDecimal | Selling price |
| supplier_id | Long (FK) | References Supplier — one supplier per product |
| created_at | Timestamp | |

### 4.5 Supplier Service
**Owns:** Supplier directory

Manages supplier contact and sourcing information referenced by Catalog and Procurement.

**Entity: Supplier**

| Field | Type | Notes |
|---|---|---|
| supplier_id | Long (PK) | Auto-generated |
| name | String | |
| contact_email | String | |
| phone | String | |
| address | String | |

### 4.6 Inventory Service
**Owns:** Stock levels, manual adjustments & reorder alerts — core of the system

Tracks quantity per product and exposes manager-facing endpoints to manually adjust stock (damage, correction, goods received) rather than reacting to customer checkouts. Detects when stock falls below the reorder level; on breach, creates a `StockAlert` and synchronously calls Procurement to draft a purchase order.

**Entity: Stock**

| Field | Type | Notes |
|---|---|---|
| stock_id | Long (PK) | Auto-generated |
| product_id | Long (FK) | References Product in Catalog Service |
| quantity | Integer | Current on-hand quantity |
| reorder_level | Integer | Threshold that triggers a restock alert |
| warehouse_location | String | |
| last_updated | Timestamp | |

**Entity: StockAlert**

| Field | Type | Notes |
|---|---|---|
| alert_id | Long (PK) | Auto-generated |
| product_id | Long (FK) | References Product |
| triggered_at | Timestamp | |
| status | Enum | `OPEN`, `RESOLVED` |

**Entity: StockAdjustment**

| Field | Type | Notes |
|---|---|---|
| adjustment_id | Long (PK) | Auto-generated |
| product_id | Long (FK) | References Product |
| manager_id | Long (FK) | References User — who made the change |
| quantity_change | Integer | Positive or negative delta |
| reason | Enum | `DAMAGE`, `RESTOCK_RECEIVED`, `CORRECTION`, `RETURN` |
| timestamp | Timestamp | |

### 4.7 Procurement Service
**Owns:** Purchase orders to suppliers, manager approval, and supplier payments

Receives draft-purchase-order calls from Inventory when a `StockAlert` is triggered. The Inventory Manager reviews and approves each `PurchaseOrder` before it is sent to the Supplier, manages the lifecycle through to fulfillment, and logs `SupplierPayment`s made against each `PurchaseOrder` (bank transfer, cheque, etc. — no payment gateway integration required, as this is B2B/manager-logged, not a customer checkout).

**Entity: PurchaseOrder**

| Field | Type | Notes |
|---|---|---|
| po_id | Long (PK) | Auto-generated |
| supplier_id | Long (FK) | References Supplier |
| status | Enum | `PENDING_APPROVAL`, `SENT`, `RECEIVED`, `CANCELLED` |
| ordered_at | Timestamp | |
| expected_delivery | Timestamp | |
| approved_by | Long (FK) | References User — manager who approved |
| approved_at | Timestamp | |

**Entity: PurchaseOrderItem**

| Field | Type | Notes |
|---|---|---|
| po_item_id | Long (PK) | Auto-generated |
| po_id | Long (FK) | References PurchaseOrder |
| product_id | Long (FK) | References Product |
| quantity | Integer | |
| cost_price | BigDecimal | Price paid to supplier |

**Entity: SupplierPayment**

| Field | Type | Notes |
|---|---|---|
| payment_id | Long (PK) | Auto-generated |
| po_id | Long (FK) | References PurchaseOrder |
| amount | BigDecimal | Supports partial payments against a PO |
| payment_method | Enum | `BANK_TRANSFER`, `CHEQUE`, `UPI`, `CASH` |
| status | Enum | `PENDING`, `COMPLETED`, `FAILED` |
| paid_at | Timestamp | |

---

## 5. Build Order

### Phase 1 — Infrastructure (Completed)
- Eureka Server
- API Gateway + JWT auth (`JwtUtil`, `AuthController`, `JwtAuthenticationFilter`)

### Phase 2 — Core domain services
- Identity Service — Manager/Admin accounts
- Catalog Service
- Supplier Service

### Phase 3 — Inventory logic (core of the system)
- Inventory Service — Stock CRUD, reorder-level check, StockAlert creation
- StockAdjustment endpoints — manual increase/decrease with reason codes, audit history per product
- RestTemplate client in Inventory → calls Procurement when alert triggers

### Phase 4 — Procurement & approval
- Procurement Service — receives draft-PO calls from Inventory, manages PurchaseOrder lifecycle
- Manager approval endpoint (`PENDING_APPROVAL` → `SENT`), `approved_by`/`approved_at` tracking
- SupplierPayment endpoints — log payments against a PO, supports partial payments

### Phase 5 — Hardening
- Centralized exception handling per service
- Dockerize each service + docker-compose for local orchestration
- Integration tests (Inventory → Procurement → approval → StockAdjustment chain)
