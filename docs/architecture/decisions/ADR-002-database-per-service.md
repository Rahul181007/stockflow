# ADR-002: Database per Service

## Status

Accepted

## Date

2026-11-04

## Context

StockFlow is designed as a distributed system composed of independently deployable microservices.

Each service owns a specific business capability:

- Authentication Service owns user identity and credentials.
- Tenant Service owns companies, memberships, and roles.
- Product Service owns product catalog information.
- Warehouse Service owns warehouse metadata.
- Inventory Service owns stock quantities and reservations.
- Order Service owns orders and order lifecycle.
- Notification Service owns notification records and delivery history.

If all services access the same database tables, the services become tightly coupled.

For example:

- Order Service could directly query Inventory Service tables.
- Product Service could modify data owned by Warehouse Service.
- A schema change in one service could unexpectedly break another service.

This would undermine service independence and make the system difficult to maintain and deploy.

We need a database ownership strategy that preserves service boundaries while allowing services to exchange information.

## Decision

We will adopt the **Database per Service** pattern.

Each business microservice owns its own database and is solely responsible for its schema, migrations, queries, and business data.

The API Gateway does not require a business database.

The initial database allocation will be:

| Service | Database |
|---|---|
| Authentication Service | `auth_db` |
| Tenant Service | `tenant_db` |
| Product Service | `product_db` |
| Warehouse Service | `warehouse_db` |
| Inventory Service | `inventory_db` |
| Order Service | `order_db` |
| Notification Service | `notification_db` |
| API Gateway | No business database |

These are logically separate PostgreSQL databases.

During local development, they may initially run on the same PostgreSQL server or Docker container to reduce infrastructure complexity.

Logical database separation does not mean that every service needs a separate PostgreSQL server.

## Database Ownership Rules

### 1. Single Service Ownership

Every database has exactly one owning service.

For example:

- Only Order Service can create or modify orders.
- Only Inventory Service can modify stock quantities and reservations.
- Only Product Service can create or modify product catalog records.
- Only Warehouse Service can create or modify warehouse metadata.

Other services must use the owning service's APIs or integration events to obtain information or request changes.

### 2. No Cross-Service Database Access

Services must never directly access another service's database.

For example, Order Service must not:

- Connect to `inventory_db`.
- Query Inventory Service tables.
- Modify Inventory Service reservations.
- Depend on Inventory Service's internal TypeORM entities.

Instead, Order Service communicates with Inventory Service through RabbitMQ events for asynchronous workflows.

REST APIs are used when synchronous validation or data retrieval is required.

### 3. Independent Database Schemas

Each service maintains its own:

- TypeORM entities.
- Database migrations.
- Repository implementations.
- Database configuration.
- Database credentials.
- Data validation rules.

A service can change its internal schema without requiring other services to know about the change, provided its external API and event contracts remain compatible.

## Tenant Data Isolation

StockFlow uses a shared database per service with tenant-aware tables.

We are not creating a separate database for every tenant in the MVP.

Most business tables will contain a `tenant_id` column.

Example:

```text id="5zq5f3"
products
--------------------------------
id
tenant_id
sku
name
description
status

warehouses
--------------------------------
id
tenant_id
name
code
address
status

inventory_items
--------------------------------
id
tenant_id
product_id
warehouse_id
on_hand
reserved

orders
--------------------------------
id
tenant_id
order_number
warehouse_id
status
sales_channel
```

### Tenant Isolation Rules

1. Every business record must belong to a tenant where applicable.
2. Every tenant-scoped query must enforce the correct tenant context.
3. Tenant context must come from authenticated and authorized request context, not an untrusted client-supplied value.
4. Unique business constraints must include `tenant_id` where uniqueness is tenant-scoped.
5. Relationships must be validated within the same tenant.
6. Database permissions and application-level authorization must prevent unauthorized cross-tenant access.

For example, SKU uniqueness will be enforced using:

```sql
UNIQUE (tenant_id, sku)
```

Warehouse code uniqueness will be enforced using:

```sql
UNIQUE (tenant_id, code)
```

Inventory uniqueness will be enforced using:

```sql
UNIQUE (tenant_id, product_id, warehouse_id)
```

Where appropriate, composite foreign keys can help enforce tenant consistency at the database level.

## How Services Exchange Data

Database separation means services cannot use database joins across service boundaries.

Instead, they exchange information through REST APIs and RabbitMQ.

### Synchronous Communication: REST

REST is used when a service needs an immediate response.

Examples:

- Order Service requests product information from Product Service.
- Order Service validates warehouse information through Warehouse Service.
- API Gateway forwards authenticated requests to backend services.

### Asynchronous Communication: RabbitMQ

RabbitMQ is used for event-driven workflows.

Examples:

- Order Service publishes `OrderCreated`.
- Inventory Service publishes `InventoryReserved`.
- Inventory Service publishes `InventoryReservationRejected`.
- Product Service publishes product lifecycle events where needed.

Each service processes messages and updates only its own database.

## PostgreSQL Infrastructure Strategy

For local development, we will initially use one PostgreSQL Docker container hosting multiple logical databases.

For example:

```text id="g5n1zt"
PostgreSQL Server
│
├── auth_db
├── tenant_db
├── product_db
├── warehouse_db
├── inventory_db
├── order_db
└── notification_db
```

Each service will have its own database credentials and connection configuration.

We will avoid using a single shared application database user with unrestricted access to all databases.

Production infrastructure may use separate PostgreSQL instances, managed database services, or other deployment arrangements depending on availability, scaling, security, and operational requirements.

## Database Transactions

Each service uses local ACID transactions within its own database.

For example, when Order Service creates an order, it can atomically save:

- Order record.
- Order item records.
- Idempotency record.
- Outbox event.

However, a single ACID transaction cannot span Order Service and Inventory Service databases.

The distributed order and inventory workflow therefore uses the Saga pattern and asynchronous events.

Each service commits its local transaction independently.

The Transactional Outbox pattern ensures that business changes and outgoing events are saved atomically within the owning service's database.

## Database Migrations

Each service owns and executes its own database migrations.

For example:

```text id="o3zq4w"
apps/
├── order-service/
│   └── src/
│       └── infrastructure/
│           └── database/
│               └── migrations/
│
└── inventory-service/
    └── src/
        └── infrastructure/
            └── database/
                └── migrations/
```

Migration rules:

1. Never modify another service's schema through a migration.
2. Use version-controlled migrations.
3. Do not use TypeORM `synchronize: true` in production.
4. Test migrations before deployment.
5. Plan backward-compatible schema changes when services communicate through shared contracts.

## Alternatives Considered

### 1. Shared Database for All Services

All microservices would use the same database and tables.

**Advantages:**

- Simple initial setup.
- Easy cross-module joins.
- Fewer database instances to manage.

**Disadvantages:**

- Strong coupling between services.
- Difficult independent schema evolution.
- Risk of one service modifying another service's data.
- Database changes can break multiple services.
- Weakens microservice boundaries.

This option is rejected.

### 2. Separate Database Server for Every Service

Every service would run on an entirely separate PostgreSQL server.

**Advantages:**

- Strong infrastructure isolation.
- Independent resource allocation.
- Independent scaling and backup strategies.
- Better control over service-level database availability.

**Disadvantages:**

- Higher infrastructure cost.
- More operational complexity.
- More resource consumption during local development.
- Unnecessary complexity for the initial MVP.

We are not choosing this as the initial infrastructure strategy.

### 3. Shared Database Server with Separate Logical Databases

Each service gets its own logical PostgreSQL database on a shared PostgreSQL server.

**Advantages:**

- Clear database ownership.
- Lower infrastructure complexity.
- Easier local development.
- Independent database schemas and migrations.

**Disadvantages:**

- Services still share some server-level resources.
- A PostgreSQL server outage can affect multiple services.
- Resource contention may occur.

This is the selected initial infrastructure strategy.

### 4. Database per Tenant

Each company would receive a separate database.

**Advantages:**

- Stronger tenant-level database isolation.
- Easier tenant-specific backups and restores.
- Potentially useful for large enterprise customers.

**Disadvantages:**

- Increased database provisioning and management complexity.
- More complicated migrations.
- Higher infrastructure overhead.
- Difficult to manage when tenant counts become large.

This may be considered in the future for enterprise isolation requirements, but it is outside the MVP.

## Consequences

### Positive Consequences

- Stronger service boundaries.
- Independent schema evolution.
- Clear data ownership.
- Reduced accidental coupling.
- Independent database migrations.
- Better alignment with microservices architecture.

### Negative Consequences

- Cross-service joins are not possible.
- Data consistency across services requires asynchronous workflows.
- Some information may need to be duplicated as snapshots or projections.
- Distributed transactions are not available.
- Debugging distributed data flows requires correlation and tracing.

## Implementation Notes

We will implement this decision using:

- PostgreSQL.
- TypeORM.
- One logical database per business service.
- Service-specific database credentials.
- Tenant-aware tables using `tenant_id`.
- Version-controlled migrations.
- REST and RabbitMQ for inter-service communication.
- Transactional Outbox for reliable event publishing.
- Saga choreography for distributed workflows.

Redis will be used for caching and temporary data where appropriate. PostgreSQL remains the source of truth for business records and inventory quantities.

## References

- Microservices.io: Database per Service Pattern — https://microservices.io/patterns/data/database-per-service.html
- PostgreSQL Documentation — https://www.postgresql.org/docs/
- TypeORM Documentation — https://typeorm.io/
- NestJS Documentation — https://docs.nestjs.com/