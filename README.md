# E-Commerce Research Platform — Documentation & Orchestration

This repository contains all BMAD (Breakthrough Method for Agile AI-Driven Development) artifacts and the master `docker-compose.yml` for the e-commerce research platform.

## BMAD Artifacts

| Document | BMAD Phase | Description |
|----------|-----------|-------------|
| [product-brief.md](product-brief.md) | Phase 1: Analyst | Problem statement, users, MVP boundaries |
| [prd.md](prd.md) | Phase 2: Product Manager | Requirements, personas, feature matrix |
| [architecture.md](architecture.md) | Phase 3: Architect | System design, tech stack, patterns |
| [epics-and-stories.md](epics-and-stories.md) | Phase 4: Scrum Master | Epics and user stories |

## Microservices

| Service | Port | Repository |
|---------|------|------------|
| Discovery Server (Eureka) | 8761 | [ecom-discovery-server](https://github.com/Vikas-Personal-Research-Org/ecom-discovery-server) |
| Config Server | 8888 | [ecom-config-server](https://github.com/Vikas-Personal-Research-Org/ecom-config-server) |
| API Gateway | 8080 | [ecom-api-gateway](https://github.com/Vikas-Personal-Research-Org/ecom-api-gateway) |
| Product Service | 8081 | [ecom-product-service](https://github.com/Vikas-Personal-Research-Org/ecom-product-service) |
| Inventory Service | 8082 | [ecom-inventory-service](https://github.com/Vikas-Personal-Research-Org/ecom-inventory-service) |
| Order Service | 8083 | [ecom-order-service](https://github.com/Vikas-Personal-Research-Org/ecom-order-service) |
| User Service | 8084 | [ecom-user-service](https://github.com/Vikas-Personal-Research-Org/ecom-user-service) |
| Payment Service | 8085 | [ecom-payment-service](https://github.com/Vikas-Personal-Research-Org/ecom-payment-service) |
| Notification Service | 8086 | [ecom-notification-service](https://github.com/Vikas-Personal-Research-Org/ecom-notification-service) |

## Quick Start

### Prerequisites
- Java 21+
- Maven 3.9+
- Docker & Docker Compose

### Option 1: Docker Compose (All Services)

Clone all repos into the same parent directory, then:

```bash
cd ecom-platform-docs

# Build all service JARs first
for dir in ../ecom-discovery-server ../ecom-config-server ../ecom-api-gateway ../ecom-user-service ../ecom-product-service ../ecom-inventory-service ../ecom-order-service ../ecom-payment-service ../ecom-notification-service; do
  (cd "$dir" && mvn clean package -DskipTests)
done

# Start all services
docker-compose up --build
```

### Option 2: Run Individual Services

```bash
cd ../ecom-discovery-server && mvn spring-boot:run   # Start first
cd ../ecom-config-server && mvn spring-boot:run       # Start second
cd ../ecom-api-gateway && mvn spring-boot:run         # Start third
# Then start business services in any order
```

## Verification

1. **Eureka Dashboard**: http://localhost:8761
2. **Swagger UI** (per service): http://localhost:{port}/swagger-ui.html
3. **H2 Console** (per service): http://localhost:{port}/h2-console

## End-to-End Test Flow

```bash
# 1. Register a user
curl -X POST http://localhost:8080/api/users/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123","firstName":"Test","lastName":"User"}'

# 2. Login
curl -X POST http://localhost:8080/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# 3. Browse products
curl http://localhost:8080/api/products

# 4. Add to cart
curl -X POST http://localhost:8080/api/cart/1/items \
  -H "Content-Type: application/json" \
  -d '{"productId":1,"productName":"Wireless Headphones","quantity":1,"price":79.99}'

# 5. Place order
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"shippingAddress":"123 Main St"}'
```

## Tech Stack

- **Java 21** + **Spring Boot 3.4.1** + **Spring Cloud 2024.0.0**
- **H2** in-memory databases (per service)
- **Eureka** service discovery
- **Spring Cloud Gateway** API gateway
- **OpenFeign** + **Resilience4j** for inter-service calls
- **JWT** authentication
- **Docker** containerization
