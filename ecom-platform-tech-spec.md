# E-Commerce Microservices Platform — Technical Specification

## 1. Executive Summary

This document describes the complete technical architecture of an e-commerce microservices platform built with **Spring Boot 3.4.1**, **Spring Cloud 2024.0.0**, and **Java 21**. The platform consists of **9 independently deployable microservices** that communicate via REST APIs, use Eureka for service discovery, and are fully containerized with Docker. An observability stack (OpenTelemetry, Prometheus, Tempo, Loki, Grafana) provides end-to-end tracing, metrics, and logging.

---

## 2. Architecture Overview

### 2.1 Architecture Style

- **Pattern**: Microservices with Database-per-Service
- **Communication**: Synchronous REST (OpenFeign) with circuit breakers (Resilience4j) + asynchronous events via Apache Kafka
- **Service Discovery**: Netflix Eureka (client-side discovery)
- **API Gateway**: Spring Cloud Gateway (reactive, route-based)
- **Configuration**: Spring Cloud Config Server (native/classpath mode)
- **Resilience**: Resilience4j circuit breakers and retry with exponential backoff on critical inter-service calls
- **Messaging**: Apache Kafka for asynchronous order events and notification delivery
- **Security**: JWT-based authentication (issued by User Service)
- **Observability**: OpenTelemetry (traces), Prometheus (metrics), Loki (logs), Grafana (dashboards)

### 2.2 Service Topology

```
                         ┌─────────────────┐
                         │   API Gateway    │
                         │   (port 8080)    │
                         └────────┬─────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │           │           │           │            │
    ┌─────┴─────┐ ┌───┴───┐ ┌────┴────┐ ┌────┴────┐ ┌────┴────┐
    │   User    │ │Product│ │Inventory│ │  Order  │ │ Payment │
    │ Service   │ │Service│ │ Service │ │ Service │ │ Service │
    │ (8084)    │ │(8081) │ │ (8082)  │ │ (8083)  │ │ (8085)  │
    └───────────┘ └───────┘ └─────────┘ └────┬────┘ └─────────┘
                                              │
                                    ┌─────────┴─────────┐
                                    │   Apache Kafka     │
                                    │   (port 9092)      │
                                    └─────────┬──────────┘
                                              │ (async events)
                                        ┌─────┴──────┐
                                        │Notification│
                                        │  Service   │
                                        │  (8086)    │
                                        └────────────┘

    ┌────────────────┐     ┌──────────────────┐
    │ Discovery      │     │  Config Server   │
    │ Server (8761)  │     │  (8888)          │
    └────────────────┘     └──────────────────┘
```

### 2.3 Startup Order

Services must start in this sequence:

1. **Discovery Server** (Eureka) — all services register here
2. **Config Server** — serves configuration to all other services
3. **API Gateway** — entry point for all client traffic
4. **Business Services** (User, Product, Inventory, Order, Payment, Notification) — in any order

---

## 3. Infrastructure Services

### 3.1 Discovery Server (`ecom-discovery-server`)

| Property | Value |
|----------|-------|
| **Port** | 8761 |
| **Role** | Netflix Eureka Server for service registration and discovery |
| **Annotations** | `@EnableEurekaServer` |
| **Self-registration** | Disabled (`register-with-eureka: false`) |
| **Self-preservation** | Disabled (development mode) |
| **Dashboard** | `http://localhost:8761` |

All other services are Eureka clients and register themselves on startup. The discovery server does not register with itself and does not fetch the registry — it is a standalone registry.

### 3.2 Config Server (`ecom-config-server`)

| Property | Value |
|----------|-------|
| **Port** | 8888 |
| **Role** | Centralized configuration server for all microservices |
| **Annotations** | `@EnableConfigServer` |
| **Profile** | `native` (classpath-based, no Git required) |
| **Config Location** | `classpath:/configurations/` |

The config server stores YAML configuration files for all 7 downstream services in `src/main/resources/configurations/`. Each microservice fetches its configuration from `http://localhost:8888/<service-name>/default` on startup.

**Managed service configurations:**

| Service | Config File | Port |
|---------|-------------|------|
| api-gateway | `api-gateway.yml` | 8080 |
| product-service | `product-service.yml` | 8081 |
| inventory-service | `inventory-service.yml` | 8082 |
| order-service | `order-service.yml` | 8083 |
| user-service | `user-service.yml` | 8084 |
| payment-service | `payment-service.yml` | 8085 |
| notification-service | `notification-service.yml` | 8086 |

### 3.3 API Gateway (`ecom-api-gateway`)

| Property | Value |
|----------|-------|
| **Port** | 8080 |
| **Role** | Single entry point for all client requests |
| **Framework** | Spring Cloud Gateway (reactive) |
| **Load Balancing** | Client-side via `lb://` protocol with Eureka |
| **CORS** | Global CORS enabled for all origins |

**Route Table:**

| Route ID | Path Pattern | Target Service | Eureka Service Name |
|----------|-------------|----------------|---------------------|
| user-service | `/api/users/**` | User Service | `user-service` |
| product-service | `/api/products/**`, `/api/categories/**` | Product Service | `product-service` |
| inventory-service | `/api/inventory/**` | Inventory Service | `inventory-service` |
| order-service | `/api/orders/**`, `/api/cart/**` | Order Service | `order-service` |
| payment-service | `/api/payments/**` | Payment Service | `payment-service` |
| notification-service | `/api/notifications/**` | Notification Service | `notification-service` |

The gateway uses Eureka-based load balancing (`lb://service-name`) to route requests. No custom filters, rate limiting, or authentication filters are configured at the gateway level.

---

## 4. Business Services

### 4.1 User Service (`ecom-user-service`)

| Property | Value |
|----------|-------|
| **Port** | 8084 |
| **Database** | H2 in-memory (`jdbc:h2:mem:userdb`) |
| **Role** | User registration, authentication (JWT), and profile management |
| **Eureka Name** | `user-service` |

#### API Endpoints

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| POST | `/api/users/register` | Register a new user | No |
| POST | `/api/users/login` | Login (returns JWT token) | No |
| GET | `/api/users/{id}` | Get user by ID | No |
| GET | `/api/users/email/{email}` | Get user by email | No |
| PUT | `/api/users/{id}` | Update user profile | No |

#### Data Model

**User Entity:**
- `id` (Long, PK), `email` (String, unique), `password` (String, BCrypt-hashed)
- `firstName`, `lastName` (String)
- `role` (Enum: BUYER, SELLER, ADMIN — default BUYER)
- `createdAt`, `updatedAt` (LocalDateTime, auto-managed)

#### Security

- **JWT**: HMAC SHA-256, 24-hour expiration
- **Password Encoding**: BCrypt
- **Session Management**: Stateless
- **Spring Security**: Configured but all `/api/users/**` endpoints are currently permitted

#### Seed Data

10 pre-loaded users with roles BUYER, SELLER, ADMIN (password: `password123` for all).

#### Inter-Service Dependencies

None. This is a standalone service. Other services do not call it directly.

---

### 4.2 Product Service (`ecom-product-service`)

| Property | Value |
|----------|-------|
| **Port** | 8081 |
| **Database** | H2 in-memory (`jdbc:h2:mem:productdb`) |
| **Role** | Product catalog and category management |
| **Eureka Name** | `product-service` |

#### API Endpoints

**Products (`/api/products`):**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/products` | Get all products |
| GET | `/api/products/{id}` | Get product by ID |
| POST | `/api/products` | Create product |
| PUT | `/api/products/{id}` | Update product |
| DELETE | `/api/products/{id}` | Delete product |
| GET | `/api/products/category/{categoryId}` | Get products by category |
| GET | `/api/products/search?name={name}` | Search by name (case-insensitive) |
| GET | `/api/products/filter?minPrice=&maxPrice=&categoryId=` | Filter by price range |

**Categories (`/api/categories`):**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/categories` | Get all categories |
| GET | `/api/categories/{id}` | Get category by ID |
| POST | `/api/categories` | Create category |
| PUT | `/api/categories/{id}` | Update category |
| DELETE | `/api/categories/{id}` | Delete category |

#### Data Model

**Product Entity:**
- `id` (Long, PK), `name` (String, required), `description` (String)
- `price` (BigDecimal, required), `imageUrl` (String)
- `category` (ManyToOne → Category), `active` (boolean, default true)
- `createdAt`, `updatedAt` (auto-managed)

**Category Entity:**
- `id` (Long, PK), `name` (String, unique, required), `description` (String)
- `products` (OneToMany → Product)

#### Inter-Service Dependencies

None. Standalone service.

---

### 4.3 Inventory Service (`ecom-inventory-service`)

| Property | Value |
|----------|-------|
| **Port** | 8082 |
| **Database** | H2 in-memory (`jdbc:h2:mem:inventorydb`) |
| **Role** | Stock management, reservation, and release |
| **Eureka Name** | `inventory-service` |

#### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/inventory` | Get all inventory records |
| GET | `/api/inventory/{productId}` | Get inventory for a product |
| POST | `/api/inventory` | Create or update inventory |
| POST | `/api/inventory/reserve` | Reserve stock for an order |
| POST | `/api/inventory/release` | Release reserved stock |
| GET | `/api/inventory/low-stock` | Get items below reorder level |

#### Data Model

**Inventory Entity:**
- `id` (Long, PK), `productId` (Long, unique)
- `quantity` (Integer, total stock), `reservedQuantity` (Integer, default 0)
- `reorderLevel` (Integer, default 10)
- `lastUpdated` (auto-managed)
- Computed: `availableQuantity = quantity - reservedQuantity`

**InventoryEvent Entity (audit trail):**
- `id`, `productId`, `eventType` (STOCK_UPDATED, STOCK_RESERVED, STOCK_RELEASED, LOW_STOCK_ALERT)
- `quantity`, `timestamp`

#### Seed Data

50 products pre-loaded with varying stock levels.

#### Inter-Service Dependencies

None outbound. Called by Order Service via Feign.

---

### 4.4 Order Service (`ecom-order-service`)

| Property | Value |
|----------|-------|
| **Port** | 8083 |
| **Database** | H2 in-memory (`jdbc:h2:mem:orderdb`) |
| **Role** | Shopping cart, checkout, and order lifecycle management |
| **Eureka Name** | `order-service` |

**This is the central orchestrator of the e-commerce flow.** It is the only service that calls other business services.

#### API Endpoints

**Cart (`/api/cart`):**

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/cart?userId={userId}` | Add item to cart |
| GET | `/api/cart/{userId}` | Get cart items for user |
| PUT | `/api/cart/{cartItemId}?quantity={qty}` | Update cart item quantity |
| DELETE | `/api/cart/{cartItemId}` | Remove item from cart |
| DELETE | `/api/cart/user/{userId}` | Clear entire cart |

**Orders (`/api/orders`):**

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/orders/checkout/{userId}` | Checkout (cart → order) |
| GET | `/api/orders/{id}` | Get order by ID |
| GET | `/api/orders/user/{userId}` | Get orders for user |
| GET | `/api/orders/{id}/track` | **[NEW]** Track order with timeline and estimated delivery |
| PUT | `/api/orders/{id}/status?status={status}` | Update order status |
| PUT | `/api/orders/{id}/cancel` | Cancel order |

#### Data Model

**Order Entity:**
- `id` (Long, PK), `userId` (Long), `totalAmount` (BigDecimal)
- `status` (Enum: CREATED, CONFIRMED, **PROCESSING**, PAYMENT_PENDING, PAID, SHIPPED, DELIVERED, CANCELLED)
- `shippingAddress` (String), `createdAt`, `updatedAt`
- `items` (OneToMany → OrderItem)

**OrderItem Entity:**
- `id`, `order` (ManyToOne → Order), `productId`, `productName`, `quantity`, `price`

**CartItem Entity:**
- `id`, `userId`, `productId`, `productName`, `quantity`, `price`

#### Inter-Service Dependencies

**Synchronous (Feign Clients):**

| Target Service | Feign Client | Operations |
|----------------|-------------|------------|
| **Inventory Service** | `InventoryClient` | `GET /api/inventory/{productId}`, `POST /api/inventory/reserve`, `POST /api/inventory/release` |
| **Payment Service** | `PaymentClient` | `POST /api/payments/process` |

**Asynchronous (Kafka Producer):**

| Kafka Topic | Purpose | Consumer |
|-------------|---------|----------|
| `order-events` | Publishes all order lifecycle events (created, confirmed, cancelled, status changes) | Any service needing order state changes |
| `order-notifications` | Publishes notification events for order status changes | Notification Service |

The `OrderEventProducer` component publishes events to Kafka with the order ID as the message key (for partition ordering). This replaces the previous synchronous Feign call to the Notification Service, making notification delivery fully asynchronous and decoupled.

#### Circuit Breakers (Resilience4j)

Both the Inventory and Payment Feign calls are protected by circuit breakers:

| Parameter | Value |
|-----------|-------|
| Sliding window size | 10 |
| Failure rate threshold | 50% |
| Wait duration (open state) | 5 seconds |
| Permitted calls (half-open) | 3 |

Fallback methods return graceful error responses when circuits are open.

#### Retry with Exponential Backoff (Resilience4j)

**[NEW]** The Inventory Service Feign calls are additionally protected by a retry mechanism:

| Parameter | Value |
|-----------|-------|
| Max attempts | 3 |
| Initial wait duration | 1 second |
| Exponential backoff multiplier | 2x (1s → 2s → 4s) |
| Retried exceptions | `FeignException`, `IOException` |

Retry executes before the circuit breaker — if all retries fail, the circuit breaker records the failure.

#### Order Tracking (NEW)

The new `GET /api/orders/{id}/track` endpoint returns an `OrderTrackingResponse` with:
- Current order status
- A visual timeline of all order stages (CREATED → CONFIRMED → PROCESSING → SHIPPED → DELIVERED) with completion status
- Estimated delivery date (5 days from order creation)
- Shipping address and total amount

The `PROCESSING` status represents the stage where the order is being prepared for shipment, between payment confirmation and shipping.

#### Checkout Flow (Saga-like pattern)

```
1. Validate cart is not empty
2. FOR each cart item:
   └── Reserve inventory (Feign → Inventory Service, with retry + circuit breaker)
       └── On failure: Release all previously reserved items → throw InsufficientStockException
3. Calculate total amount
4. Create Order (status: CREATED)
5. Process payment (Feign → Payment Service)
   ├── On SUCCESS: Update order status → CONFIRMED
   └── On FAILURE: Update order status → CANCELLED, release all inventory
6. Clear user's cart
7. Publish order event to Kafka topic "order-events" (async)
8. Publish notification event to Kafka topic "order-notifications" (async)
```

#### Order Cancellation Flow

```
1. Validate order exists and status ≠ DELIVERED
2. Set order status → CANCELLED
3. Release inventory for all order items (Feign → Inventory Service)
4. Publish cancellation event to Kafka (async)
```

---

### 4.5 Payment Service (`ecom-payment-service`)

| Property | Value |
|----------|-------|
| **Port** | 8085 |
| **Database** | H2 in-memory (`jdbc:h2:mem:paymentdb`) |
| **Role** | Mock payment processing and refunds |
| **Eureka Name** | `payment-service` |

#### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/payments/process` | Process a payment |
| GET | `/api/payments/{id}` | Get payment by ID |
| GET | `/api/payments/order/{orderId}` | Get payment by order ID |
| GET | `/api/payments/user/{userId}` | Get all payments for user |
| POST | `/api/payments/refund` | Refund a payment |

#### Data Model

**Payment Entity:**
- `id` (Long, PK), `orderId` (Long), `userId` (Long)
- `amount` (BigDecimal), `paymentMethod` (String, default "MOCK_CARD")
- `status` (Enum: PENDING, SUCCESS, FAILED, REFUNDED)
- `transactionId` (String, UUID, unique), `createdAt`

#### Business Logic

- **Mock Payment**: 90% success rate, 10% random failure
- **Transaction ID**: UUID generated per payment
- **Refund Rules**: Only SUCCESS payments can be refunded; already-refunded payments are rejected

#### Inter-Service Dependencies

None outbound. Called by Order Service via Feign.

---

### 4.6 Notification Service (`ecom-notification-service`)

| Property | Value |
|----------|-------|
| **Port** | 8086 |
| **Database** | H2 in-memory (`jdbc:h2:mem:notificationdb`) |
| **Role** | Notification storage and delivery (mock) |
| **Eureka Name** | `notification-service` |

#### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/notifications` | Send a custom notification |
| POST | `/api/notifications/order-event` | Handle order event from Order Service |
| GET | `/api/notifications` | Get all notifications |
| GET | `/api/notifications/user/{userId}` | Get notifications for user |
| GET | `/api/notifications/order/{orderId}` | Get notifications for order |

#### Data Model

**Notification Entity:**
- `id` (Long, PK), `userId`, `orderId`
- `type` (Enum: ORDER_CONFIRMATION, ORDER_SHIPPED, ORDER_DELIVERED, ORDER_CANCELLED, PAYMENT_SUCCESS, PAYMENT_FAILED)
- `channel` (Enum: EMAIL, SMS)
- `subject`, `message` (max 2000 chars)
- `status` (Enum: PENDING, SENT, FAILED), `createdAt`

#### Order Status Mapping

| Order Status | Notification Type |
|-------------|-------------------|
| ORDER_CONFIRMED | ORDER_CONFIRMATION |
| PAID | PAYMENT_SUCCESS |
| SHIPPED | ORDER_SHIPPED |
| CANCELLED | ORDER_CANCELLED |

#### Inter-Service Dependencies

None outbound. Called by Order Service via Feign.

---

## 5. Inter-Service Communication Map

### 5.1 Service Dependency Graph

```
┌──────────┐   Feign (reserve/release stock)   ┌─────────────────┐
│          │ ─────────────────────────────────► │ Inventory       │
│  Order   │   + retry + circuit breaker        │ Service (8082)  │
│ Service  │                                    ├─────────────────┤
│ (8083)   │   Feign (process payment)          │ Payment         │
│          │ ─────────────────────────────────► │ Service (8085)  │
│          │   + circuit breaker                │                 │
│          │                                    └─────────────────┘
│          │
│          │   Kafka (async events)             ┌─────────────────┐
│          │ ═══════════════════════════════════►│  Apache Kafka   │
└──────────┘   order-events                     │  (port 9092)    │
               order-notifications              └────────┬────────┘
                                                         │ (consumed by)
                                                ┌────────┴────────┐
                                                │ Notification    │
                                                │ Service (8086)  │
                                                └─────────────────┘
```

**Key insight**: The Order Service uses a **hybrid communication pattern**: synchronous Feign for critical operations (inventory reservation, payment processing) that require immediate responses, and asynchronous Kafka events for non-critical operations (notifications, event broadcasting) that benefit from decoupling and resilience.

### 5.2 Communication Summary

| From | To | Protocol | Operations |
|------|----|----------|------------|
| API Gateway | All services | HTTP (lb://) | Route requests via Eureka |
| All services | Discovery Server | HTTP | Register & heartbeat |
| All services | Config Server | HTTP | Fetch configuration on startup |
| Order Service | Inventory Service | Feign (HTTP) + Retry + Circuit Breaker | Reserve stock, release stock, get inventory |
| Order Service | Payment Service | Feign (HTTP) + Circuit Breaker | Process payment |
| Order Service | Kafka (`order-events`) | Kafka Producer (async) | Publish order lifecycle events |
| Order Service | Kafka (`order-notifications`) | Kafka Producer (async) | Publish notification events |
| Notification Service | Kafka (`order-notifications`) | Kafka Consumer (async) | Consume and process notifications |

### 5.3 Hybrid Messaging Architecture

The platform uses a **hybrid synchronous/asynchronous communication pattern**:

- **Synchronous (Feign)**: Used for Inventory and Payment calls where an immediate response is required to continue the checkout flow. Protected by Resilience4j circuit breakers and retry mechanisms.
- **Asynchronous (Kafka)**: Used for order event broadcasting and notification delivery. The Order Service publishes events to two Kafka topics (`order-events` and `order-notifications`), decoupling the Notification Service and enabling future event consumers without modifying the Order Service.

**Kafka Configuration:**

| Property | Value |
|----------|-------|
| Bootstrap servers | `localhost:9092` |
| Key serializer | `StringSerializer` |
| Value serializer | `JsonSerializer` |
| Topics | `order-events` (3 partitions), `order-notifications` (3 partitions) |
| Message key | Order ID (ensures ordering per order) |

---

## 6. Data Architecture

### 6.1 Database-per-Service

Each service has its own isolated H2 in-memory database:

| Service | Database URL | Key Tables |
|---------|-------------|------------|
| User Service | `jdbc:h2:mem:userdb` | `users` |
| Product Service | `jdbc:h2:mem:productdb` | `products`, `categories` |
| Inventory Service | `jdbc:h2:mem:inventorydb` | `inventory`, `inventory_event` |
| Order Service | `jdbc:h2:mem:orderdb` | `orders`, `order_items`, `cart_items` |
| Payment Service | `jdbc:h2:mem:paymentdb` | `payments` |
| Notification Service | `jdbc:h2:mem:notificationdb` | `notifications` |

All databases use H2 in-memory (suitable for development/testing). For production, these should be replaced with persistent databases like PostgreSQL or MySQL.

### 6.2 JPA Configuration (All Services)

- `ddl-auto: update` (auto-create/update schema)
- `show-sql: true` (log SQL queries)
- H2 console enabled at `/h2-console` for each service

### 6.3 Data Consistency

The platform uses a **saga-like pattern** for data consistency during checkout:

1. Inventory is reserved first (compensating action: release)
2. Payment is processed next
3. If payment fails, inventory reservations are rolled back
4. Each step is in a separate transaction within its own database

There is no distributed transaction manager (no XA/2PC). Consistency is eventual and relies on the Order Service orchestrating compensating actions on failure.

---

## 7. Security Architecture

### 7.1 JWT Authentication

- **Issued by**: User Service on successful login
- **Algorithm**: HMAC SHA-256
- **Expiration**: 24 hours
- **Token contains**: User email, issued-at, expiration

### 7.2 Current State

- User Service has Spring Security configured (stateless, BCrypt password encoding)
- All other services have **no security configuration**
- API Gateway has **no authentication filters** — requests pass through unfiltered
- All endpoints across all services are currently publicly accessible

---

## 8. Observability Stack

### 8.1 Overview

Every service includes:
- **Spring Boot Actuator**: Health, info, and Prometheus endpoints
- **Micrometer + Prometheus**: Metrics collection and exposition
- **OpenTelemetry**: Distributed tracing with OTLP exporter
- **Correlated Logging**: Log pattern includes `[service-name, traceId, spanId]`

### 8.2 Observability Infrastructure (Docker Compose)

| Component | Port | Role |
|-----------|------|------|
| OpenTelemetry Collector | 4317 (gRPC), 4318 (HTTP) | Receives traces/metrics/logs from services |
| Prometheus | 9090 | Metrics storage and querying |
| Tempo | 3200 | Distributed trace storage |
| Loki | 3100 | Log aggregation |
| Grafana | 3000 | Unified dashboards for metrics, traces, and logs |

### 8.3 Tracing Configuration

- Sampling probability: `1.0` (100% of requests traced)
- All services export traces via OTLP to the OpenTelemetry Collector
- Docker containers include the OpenTelemetry Java Agent (v2.11.0) for automatic instrumentation

### 8.4 Actuator Endpoints (All Services)

| Endpoint | Description |
|----------|-------------|
| `/actuator/health` | Health status with details |
| `/actuator/info` | Application info |
| `/actuator/prometheus` | Prometheus metrics |

---

## 9. API Documentation

Every business service includes **SpringDoc OpenAPI (v2.8.0)** for automatic API documentation:

| Service | Swagger UI | OpenAPI JSON |
|---------|-----------|--------------|
| User Service | `http://localhost:8084/swagger-ui.html` | `http://localhost:8084/v3/api-docs` |
| Product Service | `http://localhost:8081/swagger-ui.html` | `http://localhost:8081/v3/api-docs` |
| Inventory Service | `http://localhost:8082/swagger-ui.html` | `http://localhost:8082/v3/api-docs` |
| Order Service | `http://localhost:8083/swagger-ui.html` | `http://localhost:8083/v3/api-docs` |
| Payment Service | `http://localhost:8085/swagger-ui.html` | `http://localhost:8085/v3/api-docs` |
| Notification Service | `http://localhost:8086/swagger-ui.html` | `http://localhost:8086/v3/api-docs` |

---

## 10. Containerization & Deployment

### 10.1 Docker Configuration

All services use the same Dockerfile pattern:

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v2.11.0/opentelemetry-javaagent.jar /app/opentelemetry-javaagent.jar
COPY target/*.jar app.jar
EXPOSE <port>
ENTRYPOINT ["java", "-javaagent:/app/opentelemetry-javaagent.jar", "-jar", "app.jar"]
```

### 10.2 Docker Compose

The `docker-compose.yml` orchestrates all 9 services plus the observability stack:

**Services (with health checks and dependency ordering):**

| Service | Container Port | Depends On |
|---------|---------------|------------|
| discovery-server | 8761 | — |
| config-server | 8888 | discovery-server |
| api-gateway | 8080 | discovery-server, config-server |
| user-service | 8084 | discovery-server, config-server |
| product-service | 8081 | discovery-server, config-server |
| inventory-service | 8082 | discovery-server, config-server |
| order-service | 8083 | discovery-server, config-server, inventory-service, payment-service, notification-service |
| payment-service | 8085 | discovery-server, config-server |
| notification-service | 8086 | discovery-server, config-server |

**Network**: All services are on a single Docker bridge network (`ecom-network`).

---

## 11. Technology Stack Summary

| Layer | Technology |
|-------|-----------|
| Language | Java 21 |
| Framework | Spring Boot 3.4.1 |
| Cloud | Spring Cloud 2024.0.0 |
| API Gateway | Spring Cloud Gateway |
| Service Discovery | Netflix Eureka |
| Configuration | Spring Cloud Config (native) |
| Inter-Service Calls | OpenFeign (sync) + Apache Kafka (async) |
| Resilience | Resilience4j (circuit breaker + retry with exponential backoff) |
| Messaging | Apache Kafka |
| Database | H2 (in-memory, dev) |
| ORM | Spring Data JPA / Hibernate |
| Security | Spring Security + JWT (jjwt 0.12.6) |
| API Docs | SpringDoc OpenAPI 2.8.0 |
| Metrics | Micrometer + Prometheus |
| Tracing | OpenTelemetry + Tempo |
| Logging | Logback + Loki |
| Dashboards | Grafana |
| Containerization | Docker + Docker Compose |
| Build Tool | Maven |
| Base Image | eclipse-temurin:21-jre-alpine |

---

## 12. End-to-End Purchase Flow

This is the complete flow for a customer making a purchase:

```
1. Customer registers    → POST /api/users/register → User Service
2. Customer logs in      → POST /api/users/login → User Service (returns JWT)
3. Browse products       → GET /api/products → Product Service
4. Add item to cart      → POST /api/cart?userId=1 → Order Service
5. Add another item      → POST /api/cart?userId=1 → Order Service
6. View cart             → GET /api/cart/1 → Order Service
7. Checkout              → POST /api/orders/checkout/1 → Order Service
   ├── 7a. Reserve inventory (Feign + retry + circuit breaker → Inventory Service)
   ├── 7b. Create order (status: CREATED)
   ├── 7c. Process payment (Feign + circuit breaker → Payment Service)
   │   ├── Success: Order status → CONFIRMED
   │   └── Failure: Order status → CANCELLED, release inventory
   ├── 7d. Clear cart
   ├── 7e. Publish order event → Kafka topic "order-events" (async)
   └── 7f. Publish notification event → Kafka topic "order-notifications" (async)
8. Track order           → GET /api/orders/{orderId}/track → Order Service (timeline + ETA)
9. View order            → GET /api/orders/{orderId} → Order Service
10. View notifications   → GET /api/notifications/user/{userId} → Notification Service
```

---

## 13. Port Reference

| Service | Port | Eureka Name |
|---------|------|-------------|
| Discovery Server | 8761 | — (server itself) |
| Config Server | 8888 | config-server |
| API Gateway | 8080 | api-gateway |
| Product Service | 8081 | product-service |
| Inventory Service | 8082 | inventory-service |
| Order Service | 8083 | order-service |
| User Service | 8084 | user-service |
| Payment Service | 8085 | payment-service |
| Notification Service | 8086 | notification-service |
| Apache Kafka | 9092 | — |
| Prometheus | 9090 | — |
| Grafana | 3000 | — |
| Tempo | 3200 | — |
| Loki | 3100 | — |
| OTel Collector | 4317/4318 | — |

---

## 14. Exception Handling Pattern

All business services follow a consistent exception handling pattern:

- **`@RestControllerAdvice`** with a `GlobalExceptionHandler` class
- Custom exceptions (e.g., `OrderNotFoundException`, `InsufficientStockException`) mapped to appropriate HTTP status codes
- Validation errors (400 Bad Request) with field-level error details
- Generic fallback handler (500 Internal Server Error)
- Structured JSON error responses with `timestamp`, `status`, `error`, and `message` fields

---

## 15. Testing

All services include:
- **Unit tests**: Service layer tests with Mockito
- **Controller tests**: MockMvc-based integration tests
- **Repository tests**: `@DataJpaTest` for JPA repositories (User Service)
- **Context tests**: Verify Spring application context loads

---

## 16. Known Gaps and Future Improvements

1. **No authentication at the Gateway level** — JWT validation should be added as a gateway filter
2. **H2 in-memory databases** — Replace with PostgreSQL/MySQL for production
3. ~~**No asynchronous messaging**~~ — **RESOLVED**: Kafka introduced for order events and notifications
4. **No rate limiting** — Should be added at the API Gateway
5. **No API versioning** — Routes should include version prefixes (e.g., `/api/v1/`)
6. **Mock payment processor** — Integrate with a real payment gateway (Stripe, etc.)
7. **Mock notification delivery** — Integrate with email/SMS providers
8. **No caching** — Consider Redis for product catalog and session management
9. **CORS wide open** — Restrict allowed origins for production
10. **Self-preservation disabled on Eureka** — Enable for production
11. **Kafka consumer not yet implemented in Notification Service** — Notification Service needs a Kafka consumer for the `order-notifications` topic to fully replace the synchronous Feign call
