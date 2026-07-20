Role: You are a Spring Boot + Thymeleaf developer working on the BFF frontend service (port 8090) of the IMS project.
Task: Add a search bar and delete functionality to the Products and Suppliers list pages.
Context:
BFF service uses Thymeleaf templates, calling downstream Catalog Service (products) and Supplier Service (suppliers) via RestTemplate with @LoadBalanced.
Current pages (products.html, suppliers.html) render a table with columns as shown (ID, Name, SKU, Quantity, Price for Products; ID, Name, Email, Phone, Address for Suppliers).
Logged-in role shown in this screenshot is MANAGER. Delete for Products/Suppliers should be MANAGER or ADMIN only — EMPLOYEE must not see or be able to trigger delete (read-only per finalized role rules).
Search bar should filter client-side by Name (and SKU for Products) OR server-side via a ?q= query param — pick server-side since the BFF already talks to backend services, and product/supplier lists could grow beyond a single page.
Delete should call DELETE /api/catalog/products/{id} and DELETE /api/supplier/suppliers/{id} respectively on the downstream services, propagating the X-User-Id/X-User-Role headers as usual.
Add a confirmation step before delete (simple confirm dialog is fine) to avoid accidental deletion.
Output format:
Updated Thymeleaf controller methods for ProductController and SupplierController in the BFF, adding a search endpoint (GET /products?q=..., GET /suppliers?q=...) and delete endpoints (POST /products/{id}/delete, POST /suppliers/{id}/delete).
Updated products.html and suppliers.html fragments: a search input + submit button above the table, and a "Delete" button/icon in each row, wrapped in sec:authorize (or your role-check mechanism) so it only renders for MANAGER/ADMIN.
Note any corresponding endpoints needed on the Catalog/Supplier services if search-by-name isn't already supported there (e.g., GET /api/catalog/products?search=).