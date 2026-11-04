# StockFlow — System Architecture Overview

## 1. Purpose

This document describes the high-level architecture of StockFlow, including its microservices, technology stack, database ownership, and communication patterns.

The architecture is designed to support multiple companies (tenants), independent service development, reliable asynchronous processing, and future scalability.

## 2. Architecture Style

StockFlow follows a microservices architecture.

Each business service:
- Owns its business logic.
- Owns its database.
- Exposes APIs or consumes events.
- Can be developed and deployed independently.
- Must not directly access another service's database.

We will use a monorepo to manage the source code while keeping services logically independent.

## 3. High-Level Architecture

The system contains three main communication layers:

1. Client layer — React frontend and external API clients.
2. API layer — NestJS API Gateway and internal service APIs.
3. Business service layer — independent NestJS microservices communicating through REST and RabbitMQ.

### Main Request Flow

Client → API Gateway → Business Services

### Asynchronous Communication

Business Services → Transactional Outbox → RabbitMQ → Consumer Services

## 4. Microservices and Responsibilities

| Service | Responsibility | Database |
|---|---|---|
| API Gateway | Routing, authentication context, rate limiting | None |
| Authentication | User identity, credentials, tokens | auth_db |
| Tenant | Companies, memberships, tenant roles | tenant_db |
| Product | Product catalog and SKUs | product_db |
| Warehouse | Warehouse information and metadata | warehouse_db |
| Inventory | Stock, reservations, movements | inventory_db |
| Order | Orders, order items, lifecycle | order_db |
| Notification | Notifications and delivery history | notification_db |

## 5. Database Architecture

Each business microservice owns a separate PostgreSQL database.

For the MVP, multiple logical databases may run inside one PostgreSQL container to simplify local development.

Important rules:

- A service can access only its own database.
- Every tenant-owned record must include `tenant_id`.
- Tenant isolation must be enforced in application logic and supported by database constraints and indexes where appropriate.
- Services exchange data through APIs and events, not direct database queries.

This is a shared-database-per-service tenant model, not a separate database for every tenant.

## 6. Technology Stack

| Component | Technology |
|---|---|
| Frontend | React, TypeScript, Vite |
| State Management | Redux Toolkit |
| Styling | Tailwind CSS |
| Backend | NestJS, TypeScript |
| Database | PostgreSQL |
| ORM | TypeORM |
| Cache | Redis |
| Message Broker | RabbitMQ |
| Background Jobs | BullMQ |
| API Documentation | Swagger |
| Testing | Jest |
| Containerization | Docker, Docker Compose |
| CI/CD | GitHub Actions |
| Cloud | AWS |
| Monitoring | Prometheus, Grafana |
| Distributed Tracing | OpenTelemetry |

## 7. Communication Between Services

### Synchronous Communication

REST APIs will be used when a service needs an immediate response.

Examples:
- Order Service validates product information through Product Service.
- Order Service validates warehouse information through Warehouse Service.
- Frontend requests order details through API Gateway.

### Asynchronous Communication

RabbitMQ will be used for events and asynchronous business workflows.

Examples:
- OrderCreated
- InventoryReserved
- InventoryReservationRejected
- OrderConfirmed
- OrderRejected
- OrderCancellationRequested
- InventoryReleased

### Reliability

We will implement:

- Transactional Outbox pattern.
- Idempotent event consumers.
- Durable RabbitMQ queues.
- Publisher confirmations.
- Retry and dead-letter handling.

RabbitMQ provides at-least-once delivery in our design. Consumers must safely handle duplicate messages.

## 8. Authentication and Multi-Tenancy

Authentication Service owns user identity and credentials.

Tenant Service owns company memberships and tenant-specific roles.

The API Gateway validates authentication and establishes trusted tenant context.

Every business service must independently enforce tenant authorization for its operations.

A user may belong to multiple companies and must operate within an authorized tenant context.

## 9. Order and Inventory Workflow

The initial end-to-end workflow is:

1. Customer submits an order.
2. API Gateway authenticates the request.
3. Order Service validates product and warehouse information.
4. Order Service saves the order as `PENDING_INVENTORY`.
5. Order Service publishes `OrderCreated` through its transactional outbox.
6. Inventory Service receives the event.
7. Inventory Service checks available stock.
8. Inventory Service reserves all items atomically or rejects the reservation.
9. Inventory Service publishes the reservation result.
10. Order Service updates the order to `CONFIRMED` or `REJECTED`.
11. Notification Service processes the resulting notification event.

The order is confirmed only after successful inventory reservation.

## 10. Important Architecture Principles

- Database ownership per service.
- Tenant isolation.
- Clean Architecture within each service.
- Domain-driven design where appropriate.
- Event-driven communication.
- Transactional Outbox.
- Saga choreography for distributed workflows.
- Idempotency.
- Reliable error handling and retries.
- Automated testing.
- Observability through logs, metrics, and distributed tracing.

## 11. Future Considerations

The architecture should allow future additions such as:

- Amazon and Meesho integrations.
- Shopify integration.
- Payment and shipping integrations.
- Advanced inventory forecasting.
- Automated warehouse allocation.
- Horizontal service scaling.

These features are not part of the initial MVP.

---

**Document Status:** Draft  
**Project:** StockFlow  
**Milestone:** 1 — Architecture and Foundation