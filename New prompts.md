# Copilot Prompts — PO Creation Workflow + Action Buttons Fix

Paste each prompt into GitHub Copilot Chat inside the relevant service/module.

---

## Prompt 1 — Procurement Service: Allow EMPLOYEE to create a PO with derived supplier

**Role:** You are a senior Spring Boot backend engineer working on the Procurement microservice of a Spring Boot 3.5.x / Spring Cloud 2025.0.3 / Java 25 microservices project.

**Task:** Update the Purchase Order creation endpoint so that EMPLOYEE, MANAGER, and ADMIN can all create a PO, and the supplier is derived automatically rather than supplied by the client.

**Context:**
- `PurchaseOrder` entity has fields: `productId`, `supplierId`, `quantity`, `status` (enum: `PENDING_APPROVAL`, `SENT`, `RECEIVED`, `CANCELLED`), `approvedBy`, `approvedAt`.
- Each `Product` has exactly one `supplierId` (FK), fetched via a RestTemplate `@LoadBalanced` call to the Catalog Service (`GET /api/catalog/products/{id}`).
- Identity is available via injected headers: `X-User-Id`, `X-User-Role`. Do not re-validate JWT here — the Gateway already did that.
- `HeaderAuthenticationFilter` already restricts endpoints by role; just ensure this endpoint's config allows `EMPLOYEE`, `MANAGER`, `ADMIN`.

**Requirements:**
1. `POST /api/procurement/purchase-orders` accepts only `{ "productId": Long, "quantity": Integer }` in the request body — no `supplierId` field.
2. In the service layer, call the Catalog Service client to fetch the product and extract its `supplierId`. If the product doesn't exist, return 404.
3. Persist the new `PurchaseOrder` with `status = PENDING_APPROVAL`, `supplierId` set from the fetched product, `approvedBy`/`approvedAt` left null.
4. Update the security config / method-level `@PreAuthorize` (or equivalent header-role check) to allow `EMPLOYEE`, `MANAGER`, `ADMIN` on this endpoint specifically — approval/reject/send/cancel endpoints remain `MANAGER`/`ADMIN` only, and must never allow `ROLE_SERVICE`.
5. Add a unit test verifying that a product's supplier is correctly propagated to the created PO, and that the request is rejected if `supplierId` is present in the payload (ignore it silently or reject with 400 — pick one and document it in a comment).

**Output format:** Full updated controller method, service method, and one unit test class. Include package declarations.

---

## Prompt 2 — Thymeleaf BFF: Fix missing Action buttons + add "Create PO" form

**Role:** You are a senior Spring Boot engineer working on the Thymeleaf BFF (port 8090) of a Spring Boot microservices project.

**Task:** Two fixes in the Purchase Orders view:
1. The Action column currently renders empty for `PENDING_APPROVAL` rows — Manager/Admin have no way to approve or reject from the UI.
2. Add a "+ New Purchase Order" form/button visible to EMPLOYEE, MANAGER, and ADMIN that lets them submit `productId` + `quantity`.

**Context:**
- The BFF calls the Procurement Service via RestTemplate, forwarding `X-User-Id`/`X-User-Role` headers extracted from the logged-in session.
- The current `purchase-orders.html` Thymeleaf template has an `Action` column defined in the table header but no buttons rendered per row.
- User's role is available in the model as `userRole` (or equivalent — check the existing controller for the exact attribute name).

**Requirements:**
1. In `purchase-orders.html`, for rows where `status == 'PENDING_APPROVAL'` and `userRole` is `MANAGER` or `ADMIN`, render **Approve** and **Reject** buttons that POST to the BFF controller, which forwards to `PUT /api/procurement/purchase-orders/{id}/approve` and `/reject` on the Procurement Service.
2. For rows where `status == 'SENT'` and `userRole` is `MANAGER`/`ADMIN`, render a **Mark Received** button.
3. Add a "+ New Purchase Order" button/link visible to `EMPLOYEE`, `MANAGER`, `ADMIN` (same pattern as the existing "+ New Product" button on the Products page) that opens a small form with a Product dropdown (populated from `GET /api/catalog/products`) and a Quantity input, submitting to a new BFF endpoint that forwards to `POST /api/procurement/purchase-orders`.
4. Update the BFF `PurchaseOrderController` (or equivalent) with the new POST-create endpoint and the approve/reject/receive action endpoints if they don't already exist, each forwarding the appropriate headers.
5. Use conditional rendering (`th:if`) based on `userRole` — do not hide buttons with CSS only, since that's not a real access boundary (the Procurement Service still enforces role checks server-side regardless).

**Output format:** Updated Thymeleaf template snippet for the table + new form, and the updated/new controller methods. Include package declarations for controller code.
