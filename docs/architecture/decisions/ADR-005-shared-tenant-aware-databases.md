# ADR-005: Shared Tenant-Aware Databases

## Status
Accepted

## Date
2026-11-04

## Context

StockFlow is a multi-tenant SaaS platform. Multiple companies use the same application while their business data must remain isolated.

We need a database strategy that provides:

- Strong tenant data isolation.
- Clear database ownership between microservices.
- Cost-effective infrastructure for the MVP.
- Maintainability as the platform grows.
- A migration path toward greater isolation if required.

There are multiple ways to implement multi-tenancy:

1. Separate database per tenant.
2. Separate schema per tenant.
3. Shared database with tenant identifiers.

We must also maintain the microservices principle that each service owns its data.

## Decision

StockFlow will use a **database-per-service architecture with shared tenant-aware databases**.

Each business microservice owns its own database. Multiple tenants share that service's database, and tenant-owned records contain a `tenant_id` column.

For example:

- Product Service owns `product_db`.
- Inventory Service owns `inventory_db`.
- Order Service owns `order_db`.
- Multiple companies share each respective database.
- Tenant-owned records are isolated using `tenant_id`.

A service must never directly access another service's database.

The API Gateway does not own a business database.

## Database Ownership

| Service | Database | Tenant isolation |
|---|---|---|
| Auth Service | auth_db | Identity and credential ownership |
| Tenant Service | tenant_db | Tenant and membership relationships |
| Product Service | product_db | tenant_id |
| Warehouse Service | warehouse_db | tenant_id |
| Inventory Service | inventory_db | tenant_id |
| Order Service | order_db | tenant_id |
| Notification Service | notification_db | tenant_id |

Auth records are primarily user identities and credentials. Tenant memberships and roles are owned by Tenant Service.

## Tenant Identification

Every authenticated business request must have a trusted tenant context.

The request flow is:

1. User authenticates through Auth Service.
2. The user selects an active company (tenant).
3. The application obtains a trusted tenant context.
4. API Gateway validates the authentication context and forwards the tenant context to the target service.
5. The receiving service verifies that the authenticated user has access to that tenant.
6. The service applies tenant filtering to all tenant-owned data operations.

Services must not blindly trust a `tenant_id` supplied in a request body or query parameter.

The tenant context must originate from trusted authentication and authorization mechanisms.

## Data Isolation Rules

The following rules are mandatory:

1. Every tenant-owned business record must contain `tenant_id`.
2. Every database query involving tenant-owned data must apply tenant filtering.
3. Database constraints must include tenant identifiers where appropriate.
4. Unique business identifiers must generally be scoped to a tenant.
5. Services must verify tenant ownership before reading or modifying records.
6. Cross-tenant access must be rejected.
7. Background jobs and RabbitMQ consumers must preserve and validate tenant context.
8. Administrative access across tenants must require explicit platform-level authorization and auditing.

Example:

A product SKU must be unique within a tenant, not necessarily across the entire platform.

Database constraint:

`UNIQUE(tenant_id, sku)`

Similarly, a warehouse code can use:

`UNIQUE(tenant_id, code)`

Inventory uniqueness:

`UNIQUE(tenant_id, product_id, warehouse_id)`

These constraints provide an additional layer of protection beyond application-level validation.

## Database Security

Tenant isolation must not depend exclusively on developers remembering to add filters.

The implementation should use multiple protection layers:

- Tenant-aware repository and query patterns.
- Database foreign keys and unique constraints.
- PostgreSQL Row-Level Security (RLS) evaluated as an additional defense.
- Restricted database credentials for each service.
- Automated cross-tenant access tests.
- Audit logging for sensitive operations.

PostgreSQL RLS will be evaluated and introduced with a carefully tested tenant-context mechanism.

Application-level tenant filtering remains mandatory even when RLS is enabled.

## Alternatives Considered

### 1. Separate Database Per Tenant

Advantages:
- Stronger physical data isolation.
- Easier tenant-specific backup and restoration.
- Greater flexibility for dedicated enterprise customers.

Disadvantages:
- Increased infrastructure and operational costs.
- More complicated database provisioning.
- More complex migrations across many tenant databases.
- Greater connection management overhead.

Not selected for the MVP.

### 2. Separate Schema Per Tenant

Advantages:
- Logical separation between tenants.
- Can support tenant-specific schema management.

Disadvantages:
- Migration management becomes more complicated.
- Schema provisioning and maintenance add operational overhead.
- Cross-tenant administration becomes more difficult.

Not selected for the MVP.

### 3. Shared Database With tenant_id

Advantages:
- Lower infrastructure cost.
- Simpler database provisioning.
- Easier centralized migrations.
- Efficient resource utilization.
- Suitable for a large number of small and medium tenants.

Disadvantages:
- Application queries must enforce tenant isolation consistently.
- A missing tenant filter can expose data.
- Large tenants may create resource contention.
- Tenant-specific backup and restoration are more complicated.

Selected for the MVP with strict isolation controls.

## Consequences

### Positive

- Cost-effective infrastructure.
- Clear ownership of data by microservice.
- Simpler tenant onboarding.
- Easier schema migration management.
- Supports multiple companies without provisioning a database for each company.

### Negative

- Tenant isolation requires careful implementation and testing.
- Shared infrastructure can create resource contention.
- Tenant-specific database operations may require additional tooling.
- Noisy-neighbor problems may need attention as usage grows.

## Implementation Notes

1. Add `tenant_id` to tenant-owned business tables.
2. Define tenant-aware database constraints and indexes.
3. Implement trusted tenant-context propagation.
4. Enforce tenant filtering in repository and application operations.
5. Configure separate database credentials for each service.
6. Never share TypeORM entities or repositories between services.
7. Evaluate PostgreSQL Row-Level Security.
8. Add automated tests proving that one tenant cannot access another tenant's records.
9. Include tenant context in asynchronous event envelopes.
10. Ensure logs and audit records contain tenant identifiers where appropriate without exposing sensitive data.

## Future Considerations

If StockFlow serves large enterprise customers with stronger isolation requirements, we may introduce dedicated databases for selected tenants.

This should be an explicit architectural evolution rather than the default MVP strategy.

## References

- [Microsoft: Multitenancy and SaaS architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview)
- [PostgreSQL: Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [AWS: SaaS Tenant Isolation Strategies](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/tenant-isolation.html)