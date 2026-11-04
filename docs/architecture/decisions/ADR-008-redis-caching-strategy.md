# ADR-008: Redis Caching Strategy

## Status
Accepted

## Date
2026-11-04

## Context

StockFlow consists of multiple independently deployed microservices.

As the number of companies, products, warehouses, and orders increases, frequently requested data may create unnecessary database load and increase response latency.

StockFlow needs a caching strategy that improves performance without compromising data correctness or tenant isolation.

Redis is considered for:

- Frequently accessed product information.
- Warehouse metadata.
- User session-related temporary data.
- Rate limiting.
- Temporary application data.
- Distributed coordination where justified.

However, inventory and order processing require strong correctness guarantees.

Cached inventory values must never become the authoritative source for stock reservations or availability decisions.

## Decision

StockFlow will use Redis as a distributed cache and temporary-data store.

PostgreSQL remains the source of truth for all persistent business data.

Redis will not replace PostgreSQL or become the authoritative inventory database.

The initial caching strategy will use the **Cache-Aside pattern**.

Redis usage must be tenant-aware, service-owned, and designed with explicit expiration and invalidation policies.

## 1. Redis Responsibilities

Redis may be used for:

| Use case | Redis usage |
|---|---|
| Product information | Cache frequently accessed product details |
| Warehouse information | Cache warehouse metadata |
| Tenant configuration | Cache frequently accessed configuration |
| Rate limiting | Track request counts and windows |
| Temporary data | Store short-lived application data |
| Distributed locks | Only for specific justified coordination needs |
| Session-related data | Temporary session or revocation information where required |

Redis must not be used as the primary store for:

- Orders.
- Product records.
- Warehouse records.
- Inventory quantities.
- Inventory reservations.
- Financial or transactional records.

These records remain owned by their respective PostgreSQL databases.

## 2. Cache-Aside Pattern

StockFlow will initially use Cache-Aside.

In this pattern, the application checks Redis before querying PostgreSQL.

### Cache Read Flow

1. Client requests a product.
2. Product Service generates a tenant-aware cache key.
3. Product Service checks Redis.
4. If the value exists, return the cached result.
5. If the value does not exist, query product_db.
6. Store the result in Redis with a TTL.
7. Return the result.

This is known as a cache hit or cache miss.

### Cache Update Flow

When a product is updated:

1. Product Service updates PostgreSQL.
2. The database transaction commits.
3. Product Service invalidates the corresponding Redis key.
4. The next read loads fresh data from PostgreSQL.

Database persistence must happen before cache invalidation.

Cache invalidation failures must be observable and handled safely.

## 3. Tenant-Aware Cache Keys

Because StockFlow is multi-tenant, cache keys must include tenant context whenever the cached data is tenant-specific.

Example:

```text
tenant:{tenantId}:product:{productId}
tenant:{tenantId}:warehouse:{warehouseId}
tenant:{tenantId}:product-list:{filterHash}
tenant:{tenantId}:configuration
```

The actual implementation should use a consistent key naming convention.

Cache keys must not rely only on a product ID or warehouse ID when tenant-specific access or visibility applies.

The application must validate tenant authorization independently. Knowing a cache key must never grant access to the underlying data.

## 4. TTL and Expiration Strategy

Every cache entry must have an explicit expiration policy.

Example initial TTL guidelines:

| Data | Initial TTL |
|---|---:|
| Product details | 5 minutes |
| Warehouse metadata | 5 minutes |
| Tenant configuration | 5 minutes |
| Temporary verification data | Based on business requirements |
| Rate-limiting counters | Based on configured time window |

These are starting values, not permanent requirements.

TTL values should be adjusted based on access patterns, freshness requirements, and production measurements.

Randomized TTL jitter may be used to reduce synchronized cache expiration.

Sensitive temporary data should have short expiration periods and must not be stored unnecessarily.

## 5. Cache Invalidation

Cache invalidation is necessary when underlying database records change.

Examples:

- Product updated: invalidate product details and affected product-list entries.
- Warehouse updated: invalidate warehouse metadata.
- Tenant configuration updated: invalidate the affected tenant configuration.

Invalidation should happen after the corresponding database transaction commits.

For operations where invalidation reliability is important, an event-driven invalidation mechanism may be introduced using the service's transactional outbox.

If Redis is unavailable, services should generally fall back to PostgreSQL for cacheable reads, subject to normal database availability and load protection.

A cache failure must not cause incorrect business decisions.

## 6. Inventory and Redis

Inventory correctness is a critical requirement.

Redis must not be the source of truth for:

- On-hand quantity.
- Reserved quantity.
- Available quantity.
- Inventory reservations.
- Inventory movements.

Inventory Service must use PostgreSQL transactions and appropriate concurrency control when reserving stock.

Example:

```text
available = on_hand - reserved
```

This value must be evaluated against authoritative database state during reservation.

Redis may later hold approximate or read-optimized inventory views for dashboards, but those values must be clearly treated as potentially stale.

An inventory reservation must never succeed based only on a Redis value.

## 7. Redis for Rate Limiting

Redis may support distributed rate limiting across multiple API Gateway instances.

A centralized counter is necessary because in-memory counters on separate gateway instances would not share state.

Initial rate-limiting policies may include:

- Login attempts per IP address.
- Requests per authenticated user.
- Requests per tenant.
- Sensitive endpoint limits.

Rate-limit keys must be designed to avoid accidental cross-tenant interference.

The implementation should use atomic Redis operations or an appropriate Redis-based rate-limiting algorithm.

Rate-limit behavior during Redis failure must be explicitly configured. Sensitive authentication endpoints should fail safely rather than silently becoming unlimited.

## 8. Alternatives Considered

### 1. PostgreSQL Only

Advantages:
- Simpler architecture.
- No cache invalidation complexity.
- Strong transactional consistency.

Disadvantages:
- Frequently accessed data repeatedly hits PostgreSQL.
- May increase latency and database load as traffic grows.

Can remain sufficient for services and endpoints that do not need caching.

### 2. Redis

Advantages:
- Fast in-memory data access.
- Supports expiration.
- Supports atomic operations.
- Useful for distributed rate limiting and temporary data.

Disadvantages:
- Introduces additional infrastructure.
- Requires cache invalidation.
- Cached data may become stale.
- Memory usage requires monitoring.

Selected for caching and temporary application needs.

### 3. In-Memory Application Cache

Advantages:
- Very simple.
- No network call to Redis.

Disadvantages:
- Each service instance has independent cache state.
- Cache consistency becomes difficult.
- Data is lost when instances restart.
- Does not support shared distributed rate limiting.

Not selected as the primary distributed caching strategy.

## 9. Consequences

### Positive

- Reduced database load for frequently accessed data.
- Lower response latency for cacheable reads.
- Shared caching across service instances.
- Centralized rate-limiting support.
- Useful temporary data storage.

### Negative

- Additional infrastructure and operational monitoring.
- Cache invalidation complexity.
- Possibility of stale data.
- Memory sizing and eviction policies require attention.
- Redis outages must be handled without compromising business correctness.

## 10. Implementation Notes

1. Deploy Redis using Docker Compose for local development.
2. Define Redis connection configuration separately for each service.
3. Implement Cache-Aside for selected read-heavy endpoints.
4. Use tenant-aware cache keys.
5. Define explicit TTL values.
6. Invalidate cache entries after successful database commits.
7. Monitor cache hits, misses, memory usage, and eviction.
8. Implement safe fallback behavior for cache failures.
9. Implement distributed rate limiting where required.
10. Add tests for cache hits, misses, expiration, invalidation, and Redis failures.
11. Never use Redis as the authoritative store for inventory reservations.
12. Define suitable Redis authentication, network restrictions, and TLS settings for production.

Services should use Redis only for data they are authorized to access and should avoid coupling their business logic to Redis-specific storage details.

## 11. Testing Requirements

The implementation must test:

- Cache hit.
- Cache miss.
- Cache expiration.
- Cache invalidation after product update.
- Cache invalidation after warehouse update.
- Tenant-aware cache key isolation.
- Redis connection failure.
- PostgreSQL fallback behavior.
- Rate limiting across multiple gateway instances.
- Inventory reservation while cached inventory data is stale.
- Redis restart and cache repopulation.

## References

- [Redis Documentation](https://redis.io/docs/latest/)
- [Redis Cache-Aside Pattern](https://redis.io/docs/latest/develop/use/client-side-caching/)
- [Microsoft: Cache-Aside Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [Redis: Rate Limiting](https://redis.io/docs/latest/develop/use/patterns/rate-limiting/)