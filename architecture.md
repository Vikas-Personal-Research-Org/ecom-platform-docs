# Architecture Document — E-Commerce Research Platform

## BMAD Phase 3A: Architect Agent Output

---

## 1. System Overview

The E-Commerce Research Platform is a microservice-based distributed system built with Spring Boot and Spring Cloud. It demonstrates modern distributed systems patterns including service discovery, centralized configuration, API gateway routing, inter-service communication, circuit breakers, and database-per-service isolation.

---

## 2. Architecture Style

- **Pattern**: Microservices Architecture
- **Communication**: Synchronous REST (OpenFeign) for request/response flows; REST-based event posting for notifications
- **Discovery**: Client-side discovery via Netflix Eureka
- **Gateway**: Spring Cloud Gateway (reactive) as the single entry point
- **Configuration**: Spring Cloud Config Server with native (classpath) profiles
- **Resilience**: Resilience4j circuit breakers on inter-service Feign calls
- **Data**: Database-per-service with H2 in-memory (development); each service owns its schema

---

## 3. Service Topology

```
                        ┌─────────────────┐
                        │  Discovery Server│
                        │   (Eureka)       │
                        │   Port: 8761     │
                        └────────┬─────────┘
                                 │ registers/discovers
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
┌───────▼──────┐   ┌────────────▼──────────┐   ┌─────────▼──────────┐
│ Config Server│   │    API Gateway         │   │  Business Services │
│ Port: 8888   │   │    Port: 8080          │   │  (see below)       │
└──────────────┘   └────────────┬───────────┘   └────────────────────┘
                                │ routes to
          ┌──────────┬──────────┼──────────┬──────────┬──────────┐
          ▼          ▼          ▼          ▼          ▼          ▼
     User Service  Product  Inventory   Order     Payment  Notification
     Port: 8084    Service  Service     Service   Service  Service
                   Port:8081 Port:8082  Port:8083 Port:8085 Port:8086
```

### Service Registry

| Service              | Spring Name           | Port | Type           |
|----------------------|-----------------------|------|----------------|
| Discovery Server     | discovery-server      | 8761 | Infrastructure |
| Config Server        | config-server         | 8888 | Infrastructure |
| API Gateway          | api-gateway           | 8080 | Infrastructure |
| User Service         | user-service          | 8084 | Business       |
| Product Service      | product-service       | 8081 | Business       |
| Inventory Service    | inventory-service     | 8082 | Business       |
| Order Service        | order-service         | 8083 | Business       |
| Payment Service      | payment-service       | 8085 | Business       |
| Notification Service | notification-service  | 8086 | Business       |

---

## 4. Startup Order

Services must start in this order to ensure proper registration and configuration:

1. **Discovery Server** (Eureka) — must be available before any service registers
2. **Config Server** — registers with Eureka; serves centralized configuration
3. **API Gateway** — registers with Eureka; routes traffic to downstream services
4. **Business Services** (User, Product, Inventory, Order, Payment, Notification) — register with Eureka; can start in parallel

---

## 5. Inter-Service Communication

### Synchronous (OpenFeign)

| Caller         | Callee           | Endpoint                    | Purpose                           |
|----------------|------------------|-----------------------------|-----------------------------------|
| Order Service  | Inventory Service| `GET /api/inventory/{id}`   | Check stock before order          |
| Order Service  | Inventory Service| `POST /api/inventory/reserve`| Reserve stock on checkout        |
| Order Service  | Payment Service  | `POST /api/payments/process`| Process payment on checkout       |

### REST Event Posting

| Publisher      | Consumer              | Endpoint                              | Purpose                     |
|----------------|-----------------------|---------------------------------------|-----------------------------|
| Order Service  | Notification Service  | `POST /api/notifications/order-event` | Notify on order status changes |

### Circuit Breaker Configuration

Resilience4j circuit breakers protect the Order Service Feign clients:
- **Failure rate threshold**: 50%
- **Wait duration in open state**: 5 seconds
- **Sliding window size**: 10 calls
- **Fallback**: Return error response without cascading failure

---

## 6. Data Architecture

Each service maintains its own H2 in-memory database, enforcing the database-per-service pattern:

| Service              | Database Name  | Key Entities                              |
|----------------------|----------------|-------------------------------------------|
| User Service         | userdb         | User (id, name, email, password, role)    |
| Product Service      | productdb      | Product, Category                         |
| Inventory Service    | inventorydb    | Inventory (productId, quantity, reserved)  |
| Order Service        | orderdb        | Order, OrderItem, CartItem                |
| Payment Service      | paymentdb      | Payment (orderId, amount, status, txnId)  |
| Notification Service | notificationdb | Notification (type, channel, status)      |

**Data Consistency**: Eventual consistency using a saga-like checkout flow:
1. Reserve inventory
2. Create order (CREATED status)
3. Process payment
4. If payment SUCCESS -> confirm order (CONFIRMED)
5. If payment FAILED -> release inventory, cancel order

---

## 7. API Gateway Routing

The API Gateway (port 8080) routes all external traffic using path-based predicates with Eureka load balancing (`lb://`):

| Route Pattern              | Target Service       |
|----------------------------|----------------------|
| `/api/users/**`            | user-service         |
| `/api/products/**`         | product-service      |
| `/api/categories/**`       | product-service      |
| `/api/inventory/**`        | inventory-service    |
| `/api/orders/**`           | order-service        |
| `/api/cart/**`             | order-service        |
| `/api/payments/**`         | payment-service      |
| `/api/notifications/**`    | notification-service |

Global CORS is enabled for all origins, methods, and headers (development configuration).

---

## 8. Security Architecture

- **Authentication**: JWT-based, handled by User Service
- **Token generation**: User Service issues JWT on successful login (`POST /api/users/login`)
- **Token propagation**: Client includes JWT in `Authorization: Bearer <token>` header
- **Token validation**: Each secured service validates the JWT independently
- **Roles**: BUYER, SELLER, ADMIN (defined in User entity)
- **Secret**: Shared HMAC-SHA256 key configured per service

---

## 9. Technology Stack

| Layer              | Technology                              | Version    |
|--------------------|-----------------------------------------|------------|
| Language           | Java                                    | 21         |
| Framework          | Spring Boot                             | 3.4.1      |
| Cloud              | Spring Cloud                            | 2024.0.0   |
| Service Discovery  | Netflix Eureka                          | via Spring Cloud |
| API Gateway        | Spring Cloud Gateway (reactive)         | via Spring Cloud |
| Config Management  | Spring Cloud Config Server              | via Spring Cloud |
| REST Client        | Spring Cloud OpenFeign                  | via Spring Cloud |
| Resilience         | Resilience4j                            | via Spring Cloud |
| ORM                | Spring Data JPA / Hibernate             | via Spring Boot  |
| Database           | H2 (in-memory, development)             | via Spring Boot  |
| Security           | Spring Security + JJWT                  | 0.12.6     |
| API Documentation  | SpringDoc OpenAPI (Swagger)             | via Spring Boot  |
| Build Tool         | Maven                                   | 3.x        |
| Containerization   | Docker (eclipse-temurin:21-jre-alpine)  | -          |
| Orchestration      | Docker Compose                          | -          |

---

## 10. End-to-End Purchase Flow

```
Client                Gateway(8080)   UserSvc    ProductSvc   InventorySvc   OrderSvc    PaymentSvc   NotifSvc
  │                       │              │            │             │            │             │           │
  │──POST /api/users/register──>│──────>│            │             │            │             │           │
  │<────── 201 Created ──│<─────│            │             │            │             │           │
  │                       │              │            │             │            │             │           │
  │──POST /api/users/login──>│──────────>│            │             │            │             │           │
  │<────── JWT Token ────│<──────────────│            │             │            │             │           │
  │                       │              │            │             │            │             │           │
  │──GET /api/products───>│──────────────────────────>│             │            │             │           │
  │<────── Product List──│<──────────────────────────│             │            │             │           │
  │                       │              │            │             │            │             │           │
  │──POST /api/cart──────>│─────────────────────────────────────────>│             │           │
  │<────── Cart Item ────│<────────────────────────────────────────│             │           │
  │                       │              │            │             │            │             │           │
  │──POST /api/orders/checkout/{userId}──>│──────────────────────>│             │           │
  │                       │              │            │             │<──check stock─│             │           │
  │                       │              │            │             │──reserve────>│             │           │
  │                       │              │            │             │            │──process pay─>│           │
  │                       │              │            │             │            │<──pay result──│           │
  │                       │              │            │             │            │──order event─────────────>│
  │<────── Order ────────│<────────────────────────────────────────│             │           │
```

---

## 11. Containerization Strategy

- Each service has its own `Dockerfile` using `eclipse-temurin:21-jre-alpine` as the base image
- A root-level `docker-compose.yml` orchestrates all 9 services
- Service startup ordering enforced via `depends_on` with health checks
- All ports mapped 1:1 from container to host for development simplicity

---

## 12. Key Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| H2 in-memory databases | Simplicity for research platform; no external DB setup needed |
| Synchronous Feign over async messaging | Simpler to implement for MVP; messaging can be added later |
| Database-per-service | Enforces domain boundaries; each service owns its data |
| JWT shared secret | Simpler than OAuth2/OIDC for research purposes |
| Spring Cloud ecosystem | Mature, well-documented, cohesive set of tools |
| Eureka over Consul/K8s | Native Spring Cloud integration; standalone setup |
