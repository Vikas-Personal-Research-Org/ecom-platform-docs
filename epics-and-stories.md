# Epics & User Stories — E-Commerce Research Platform

## BMAD Phase 3B: Architect / Product Manager Agent Output

---

## Epic 1: User Management

**Goal**: Enable user registration, authentication, and profile management.

### Story 1.1 — User Registration (US-01) [P0]
**As a** buyer or seller, **I want to** register with my name, email, and password, **so that** I can access the platform.

**Acceptance Criteria**:
- POST `/api/users/register` accepts name, email, password, and role
- Password is hashed before storage (BCrypt)
- Duplicate email returns 409 Conflict
- Successful registration returns 201 with user details (no password)

**Dependencies**: None

### Story 1.2 — JWT Authentication (US-02) [P0]
**As a** registered user, **I want to** log in and receive a JWT token, **so that** I can authenticate subsequent requests.

**Acceptance Criteria**:
- POST `/api/users/login` accepts email and password
- Returns JWT token on valid credentials
- Returns 401 Unauthorized on invalid credentials
- Token expires after configured duration (24h)

**Dependencies**: Story 1.1

### Story 1.3 — User Profile (US-03) [P0]
**As a** user, **I want to** view and update my profile, **so that** I can manage my account information.

**Acceptance Criteria**:
- GET `/api/users/{id}` returns user profile
- GET `/api/users/email/{email}` returns user by email
- PUT `/api/users/{id}` updates name and email
- Returns 404 if user not found

**Dependencies**: Story 1.1

### Story 1.4 — Role-Based Access (US-04) [P1]
**As an** admin, **I want** role-based access control, **so that** buyers, sellers, and admins have appropriate permissions.

**Acceptance Criteria**:
- User entity stores role (BUYER, SELLER, ADMIN)
- Role is included in JWT claims
- Endpoints can be restricted by role

**Dependencies**: Story 1.2

---

## Epic 2: Product Catalog

**Goal**: Provide a searchable product catalog with categories.

### Story 2.1 — Product CRUD (PS-01) [P0]
**As a** seller, **I want to** create, read, update, and delete products, **so that** I can manage my product listings.

**Acceptance Criteria**:
- POST `/api/products` creates a product with name, description, price, categoryId
- GET `/api/products/{id}` returns a single product
- GET `/api/products` returns all products
- PUT `/api/products/{id}` updates product details
- DELETE `/api/products/{id}` removes a product
- Validation: name required, price > 0

**Dependencies**: Story 2.2 (categories must exist)

### Story 2.2 — Category Management (PS-02) [P0]
**As a** seller, **I want to** manage product categories, **so that** products are organized for buyers.

**Acceptance Criteria**:
- POST `/api/categories` creates a category (name, description)
- GET `/api/categories` returns all categories
- GET `/api/categories/{id}` returns a single category
- PUT `/api/categories/{id}` updates a category
- DELETE `/api/categories/{id}` removes a category
- Products reference a category via foreign key

**Dependencies**: None

### Story 2.3 — Product Search (PS-03) [P0]
**As a** buyer, **I want to** search products by name, category, and price range, **so that** I can find what I need.

**Acceptance Criteria**:
- GET `/api/products/search?name={name}` returns matching products
- GET `/api/products/category/{categoryId}` returns products in category
- GET `/api/products/filter?minPrice={}&maxPrice={}&categoryId={}` filters by price and category

**Dependencies**: Stories 2.1, 2.2

### Story 2.4 — Paginated Listing (PS-04) [P1]
**As a** buyer, **I want** paginated product listings, **so that** browsing is performant with large catalogs.

**Acceptance Criteria**:
- GET `/api/products` supports `page` and `size` query parameters
- Response includes pagination metadata (totalPages, totalElements)

**Dependencies**: Story 2.1

### Story 2.5 — Seed Data (PS-05) [P0]
**As a** developer, **I want** 50+ dummy products across 5 categories, **so that** the platform has realistic test data.

**Acceptance Criteria**:
- `data.sql` seeds 5 categories (Electronics, Books, Clothing, Home, Sports)
- 50+ products distributed across all categories
- Seed runs on application startup

**Dependencies**: Stories 2.1, 2.2

---

## Epic 3: Inventory Management

**Goal**: Track and manage product stock levels.

### Story 3.1 — Stock Level Management (IS-01) [P0]
**As a** seller, **I want to** set and query stock levels, **so that** inventory is accurately tracked.

**Acceptance Criteria**:
- POST `/api/inventory` creates/updates inventory for a product (productId, quantity, reorderLevel)
- GET `/api/inventory/{productId}` returns stock for a product
- GET `/api/inventory` returns all inventory records

**Dependencies**: None (uses productId reference, no Feign dependency)

### Story 3.2 — Stock Reservation (IS-02) [P0]
**As the** order service, **I want to** reserve stock during checkout, **so that** sold items are not over-committed.

**Acceptance Criteria**:
- POST `/api/inventory/reserve` accepts productId and quantity
- Decrements available stock and increments reserved quantity
- Returns 400 if insufficient stock
- Operation is atomic

**Dependencies**: Story 3.1

### Story 3.3 — Stock Release (IS-03) [P1]
**As the** order service, **I want to** release reserved stock on cancellation, **so that** inventory returns to available pool.

**Acceptance Criteria**:
- POST `/api/inventory/release` accepts productId and quantity
- Increments available stock and decrements reserved quantity
- Returns 400 if release quantity exceeds reserved

**Dependencies**: Story 3.2

### Story 3.4 — Low Stock Alerts (IS-04) [P2]
**As a** seller, **I want to** see low-stock items, **so that** I can replenish before stockout.

**Acceptance Criteria**:
- GET `/api/inventory/low-stock` returns items where quantity <= reorderLevel

**Dependencies**: Story 3.1

---

## Epic 4: Order & Cart Management

**Goal**: Enable shopping cart and order placement with full lifecycle tracking.

### Story 4.1 — Cart Management (OS-01) [P0]
**As a** buyer, **I want to** add, update, and remove items in my cart, **so that** I can prepare my purchase.

**Acceptance Criteria**:
- POST `/api/cart` adds an item (userId, productId, productName, quantity, price)
- GET `/api/cart/{userId}` returns all cart items for a user
- PUT `/api/cart/{cartItemId}` updates quantity
- DELETE `/api/cart/{cartItemId}` removes an item
- DELETE `/api/cart/user/{userId}` clears entire cart

**Dependencies**: None (uses productId reference)

### Story 4.2 — Checkout / Place Order (OS-02) [P0]
**As a** buyer, **I want to** place an order from my cart, **so that** I can complete my purchase.

**Acceptance Criteria**:
- POST `/api/orders/checkout/{userId}` creates an order from the user's cart
- Cart must not be empty (returns 400)
- Order created with CREATED status, items copied from cart
- Cart is cleared after successful order creation

**Dependencies**: Story 4.1

### Story 4.3 — Order Status Lifecycle (OS-03) [P0]
**As a** buyer, **I want to** track my order status, **so that** I know where my order is.

**Acceptance Criteria**:
- GET `/api/orders/{id}` returns order with current status
- GET `/api/orders/user/{userId}` returns all orders for a user
- PUT `/api/orders/{id}/status` updates order status
- PUT `/api/orders/{id}/cancel` cancels an order
- Valid status transitions: CREATED -> CONFIRMED -> SHIPPED -> DELIVERED; any -> CANCELLED

**Dependencies**: Story 4.2

### Story 4.4 — Inventory Check on Order (OS-04) [P0]
**As the** system, **I want to** check and reserve inventory during checkout, **so that** orders are only placed for available products.

**Acceptance Criteria**:
- Checkout calls Inventory Service to check stock for each item
- If stock available, reserves quantity via `POST /api/inventory/reserve`
- If stock insufficient for any item, checkout fails with 400
- Circuit breaker protects against Inventory Service failures

**Dependencies**: Stories 4.2, 3.2

### Story 4.5 — Payment on Checkout (OS-05) [P0]
**As the** system, **I want to** trigger payment during checkout, **so that** orders are paid for.

**Acceptance Criteria**:
- After inventory reserved, calls Payment Service `POST /api/payments/process`
- If payment SUCCESS: order status -> CONFIRMED
- If payment FAILED: order status -> CANCELLED (inventory should be released)
- Circuit breaker protects against Payment Service failures

**Dependencies**: Stories 4.2, 5.1

---

## Epic 5: Payment Processing

**Goal**: Provide mock payment processing for orders.

### Story 5.1 — Process Payment (PM-01) [P0]
**As the** order service, **I want to** submit a payment request, **so that** orders are financially processed.

**Acceptance Criteria**:
- POST `/api/payments/process` accepts orderId, userId, amount, paymentMethod
- Mock processing: 90% success, 10% failure
- Returns payment record with status and transactionId

**Dependencies**: None

### Story 5.2 — Payment Status (PM-02) [P0]
**As a** user, **I want to** query payment status, **so that** I can verify my payment.

**Acceptance Criteria**:
- GET `/api/payments/{id}` returns payment by ID
- GET `/api/payments/order/{orderId}` returns payment by order
- GET `/api/payments/user/{userId}` returns all payments for a user
- Returns 404 if payment not found

**Dependencies**: Story 5.1

### Story 5.3 — Payment Refund (PM-03) [P1]
**As a** buyer, **I want to** get a refund for a cancelled order, **so that** my money is returned.

**Acceptance Criteria**:
- POST `/api/payments/refund` accepts paymentId and reason
- Only SUCCESS payments can be refunded
- Status changes to REFUNDED

**Dependencies**: Story 5.1

---

## Epic 6: Notifications

**Goal**: Notify users of order lifecycle events.

### Story 6.1 — Order Event Handling (NS-01) [P0]
**As the** system, **I want to** receive order events, **so that** users are notified of order updates.

**Acceptance Criteria**:
- POST `/api/notifications/order-event` accepts orderId, userId, orderStatus, totalAmount
- Maps order status to notification type (e.g., CONFIRMED -> ORDER_CONFIRMATION)
- Creates notification record in database

**Dependencies**: None

### Story 6.2 — Email Notification Stub (NS-02) [P0]
**As a** buyer, **I want to** receive order confirmation, **so that** I know my order was placed.

**Acceptance Criteria**:
- Order confirmation creates notification with channel=EMAIL
- Notification is logged (stub, no actual email sent)
- Status marked as SENT after logging

**Dependencies**: Story 6.1

### Story 6.3 — SMS Notification Stub (NS-03) [P2]
**As a** buyer, **I want to** receive shipping notifications via SMS, **so that** I know when my order ships.

**Acceptance Criteria**:
- Shipping status creates notification with channel=SMS
- Notification is logged (stub, no actual SMS sent)
- Status marked as SENT after logging

**Dependencies**: Story 6.1

---

## Epic 7: Infrastructure Services

**Goal**: Provide service discovery, centralized configuration, and API gateway.

### Story 7.1 — Service Discovery (IF-01) [P0]
**As a** developer, **I want** automatic service discovery, **so that** services find each other without hardcoded URLs.

**Acceptance Criteria**:
- Eureka Server runs on port 8761
- All services register with Eureka on startup
- Eureka dashboard shows registered services
- Services use `lb://` URIs for load-balanced calls

**Dependencies**: None

### Story 7.2 — Centralized Configuration (IF-02) [P0]
**As a** developer, **I want** centralized config management, **so that** configuration is managed in one place.

**Acceptance Criteria**:
- Config Server runs on port 8888
- Config Server registers with Eureka
- Native profile serves configuration from classpath

**Dependencies**: Story 7.1

### Story 7.3 — API Gateway (IF-03) [P0]
**As a** client, **I want** a single entry point, **so that** I don't need to know individual service URLs.

**Acceptance Criteria**:
- Gateway runs on port 8080
- Routes defined for all 6 business services
- Uses Eureka for service resolution
- CORS enabled for development
- Actuator health endpoint available

**Dependencies**: Story 7.1

---

## Epic 8: Containerization & DevOps

**Goal**: Containerize all services and enable single-command local deployment.

### Story 8.1 — Service Dockerfiles [P0]
**As a** developer, **I want** each service containerized, **so that** deployment is consistent.

**Acceptance Criteria**:
- Each of the 9 services has a Dockerfile
- Base image: `eclipse-temurin:21-jre-alpine`
- Correct port exposed per service
- JAR copied from `target/` directory

**Dependencies**: All services must be buildable

### Story 8.2 — Docker Compose (Root) [P0]
**As a** developer, **I want to** start the entire platform with `docker-compose up`, **so that** local development is easy.

**Acceptance Criteria**:
- `docker-compose.yml` at repository root
- All 9 services defined
- Startup ordering: discovery -> config -> gateway -> business services
- Health checks for dependency ordering
- All ports mapped to host

**Dependencies**: Story 8.1

---

## Story Summary by Priority

| Priority | Count | Stories |
|----------|-------|---------|
| P0 (Must)  | 22 | 1.1, 1.2, 1.3, 2.1, 2.2, 2.3, 2.5, 3.1, 3.2, 4.1, 4.2, 4.3, 4.4, 4.5, 5.1, 5.2, 6.1, 6.2, 7.1, 7.2, 7.3, 8.1, 8.2 |
| P1 (Should)| 4  | 1.4, 2.4, 3.3, 5.3 |
| P2 (Nice)  | 2  | 3.4, 6.3 |
| **Total**  | **28** | |
