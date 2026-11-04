# StockFlow — Microservice Boundaries and Ownership

## 1. Purpose

This document defines the responsibilities, data ownership, and boundaries of every microservice in StockFlow.

The objective is to maintain clear service ownership, reduce coupling, and prevent services from depending directly on another service's internal implementation or database.

## 2. Service Ownership Principles

Every business microservice must follow these rules:

1. Own its business logic.
2. Own its database and database schema.
3. Expose functionality through APIs or asynchronous messages.
4. Never directly query or modify another service's database.
5. Validate tenant authorization for its operations.
6. Publish business events when important state changes occur.
7. Handle duplicate messages safely.
8. Keep its internal domain model independent of other services.

The API Gateway is an entry point, not a place for business logic.

## 3. Service Responsibilities

### 3.1 API Gateway

**Owns:**
- External API entry point.
- Request routing.
- Authentication token validation.
- Trusted tenant context propagation.
- Rate limiting.
- API-level request handling.

**Does not own:**
- Product or inventory business logic.
- Order lifecycle.
- User credentials.
- Business databases.

**Database:** None.

### 3.2 Authentication Service

**Owns:**
- User identity.
- Email and password credentials.
- Password hashing.
- Login and logout.
- Access token and refresh token management.
- User account status.

**Database:** `auth_db`

**Does not own:**
- Company membership.
- Tenant-specific roles.
- Product or order permissions as business data.

### 3.3 Tenant Service

**Owns:**
- Company registration and management.
- Tenant information.
- User-company memberships.
- Tenant-specific roles.
- Membership status.

**Database:** `tenant_db`

**Does not own:**
- User passwords.
- Authentication credentials.
- Products, inventory, or orders.

### 3.4 Product Service

**Owns:**
- Product catalog.
- SKU management.
- Product descriptions and attributes.
- Product activation and deactivation.
- Product-related metadata.

**Database:** `product_db`

**Business rule:**
- SKU must be unique within a tenant.

**Does not own:**
- Warehouse stock quantities.
- Inventory reservations.
- Order status.

### 3.5 Warehouse Service

**Owns:**
- Warehouse creation and management.
- Warehouse names and codes.
- Warehouse addresses.
- Warehouse operational status.

**Database:** `warehouse_db`

**Does not own:**
- Stock quantities.
- Inventory reservations.
- Order lifecycle.

### 3.6 Inventory Service

**Owns:**
- On-hand stock quantities.
- Reserved stock quantities.
- Available stock calculation.
- Inventory reservations and releases.
- Stock receiving and adjustments.
- Inventory movements and audit history.

**Database:** `inventory_db`

**Business rule:**

`Available = On-Hand - Reserved`

Inventory reservation must be atomic across all items in an order.

**Does not own:**
- Product descriptions or SKU definitions.
- Warehouse addresses and metadata.
- Order status and customer order details.

Inventory stores the identifiers needed to associate stock with products, warehouses, tenants, and orders. It does not directly access their databases.

### 3.7 Order Service

**Owns:**
- Order creation.
- Order items.
- Order status and lifecycle.
- Order status history.
- Sales channel information.
- Idempotency records.
- Product and price snapshots captured when an order is created.

**Database:** `order_db`

**Does not own:**
- Inventory quantities.
- Inventory reservations.
- Product catalog definitions.
- Warehouse metadata.

Order Service requests inventory reservation through asynchronous messaging. It does not reserve stock by modifying Inventory Service data.

### 3.8 Notification Service

**Owns:**
- Notification creation.
- Notification delivery status.
- Notification attempts and history.
- Notification processing and retries.

**Database:** `notification_db`

**Does not own:**
- Order lifecycle.
- Inventory operations.
- User authentication credentials.

## 4. Database Ownership Matrix

| Service | Database | Data owned |
|---|---|---|
| Authentication | auth_db | Users and credentials |
| Tenant | tenant_db | Companies and memberships |
| Product | product_db | Products and SKUs |
| Warehouse | warehouse_db | Warehouse metadata |
| Inventory | inventory_db | Stock and reservations |
| Order | order_db | Orders and order items |
| Notification | notification_db | Notifications and delivery history |
| API Gateway | None | No business data |

For local development, these logical databases may run inside one PostgreSQL container. They remain separate databases with service-specific ownership and access credentials.

## 5. Cross-Service Communication Rules

Services communicate using two mechanisms.

### REST APIs

Used when an immediate response is required.

Examples:
- Order Service requests product details from Product Service.
- Order Service validates warehouse information through Warehouse Service.
- API Gateway routes requests to backend services.

### RabbitMQ Events

Used for asynchronous workflows and business state changes.

Examples:
- OrderCreated
- InventoryReserved
- InventoryReservationRejected
- OrderConfirmed
- OrderRejected
- OrderCancellationRequested
- InventoryReleased

Services must not use another service's database as a shortcut for communication.

## 6. Tenant Isolation

Every tenant-owned business record must be associated with a `tenant_id`.

Each service must enforce tenant isolation independently.

The trusted tenant context must be established by the authenticated request flow and validated by the receiving service.

A tenant ID supplied by an untrusted client must never be accepted as proof of authorization.

## 7. Data Ownership Examples

### Example 1: Product and Inventory

Product Service creates a product.

Inventory Service stores the product ID when stock is received.

Inventory Service does not copy or manage the product's full catalog definition.

### Example 2: Warehouse and Inventory

Warehouse Service creates a warehouse.

Inventory Service tracks quantities associated with that warehouse ID.

Inventory Service does not manage the warehouse address or metadata.

### Example 3: Order and Inventory

Order Service creates an order.

Inventory Service processes the reservation request and owns the resulting reservation.

Order Service owns the order status and updates it after receiving the reservation result event.

## 8. Architecture Rules to Enforce

- No cross-service database access.
- No shared business entities or domain models.
- No shared TypeORM entity package.
- Shared technical utilities are allowed when genuinely reusable.
- Shared event contracts are allowed.
- Each service owns its migrations and database configuration.
- Each service validates tenant authorization.
- All event consumers must be idempotent.
- Cross-service workflows must tolerate partial failures.

## 9. Future Evolution

New capabilities should be assigned to an existing service only when they fit its domain responsibility.

If a capability develops a distinct business responsibility, we may evaluate whether it deserves a separate service.

We will avoid creating additional microservices prematurely.

---

**Document Status:** Draft  
**Project:** StockFlow  
**Milestone:** 1 — Architecture and Foundation