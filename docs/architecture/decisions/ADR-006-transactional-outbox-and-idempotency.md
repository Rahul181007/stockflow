# ADR-006: Transactional Outbox and Idempotent Consumers

## Status
Accepted

## Date
2026-11-04

## Context

StockFlow uses RabbitMQ for asynchronous communication between microservices.

A critical problem occurs when a service needs to update its database and publish an event.

Consider the Order Service:

1. A customer creates an order.
2. Order Service saves the order in PostgreSQL.
3. Order Service publishes an `OrderCreated` event to RabbitMQ.

These are two independent operations involving two different systems.

If the database transaction succeeds but RabbitMQ is unavailable, the order exists but the event may never be published.

If the event is published but the database transaction fails, other services may process an event describing an order that does not exist.

This is known as the dual-write problem.

Another challenge is duplicate message delivery.

RabbitMQ provides at-least-once delivery when configured appropriately. A consumer may receive the same event more than once due to retries, acknowledgements, or application failures.

StockFlow must handle both problems reliably.

## Decision

StockFlow will implement:

1. The Transactional Outbox pattern for reliable event publishing.
2. Idempotent consumers for duplicate message protection.
3. Publisher confirms to verify broker acceptance.
4. Manual consumer acknowledgements after successful database processing.
5. Bounded retries and Dead Letter Queues (DLQs) for persistent technical failures.

These mechanisms will be implemented independently within each service that publishes or consumes events.

## 1. Transactional Outbox Pattern

The Transactional Outbox pattern ensures that a business database change and the intention to publish an event are committed in the same local database transaction.

Each publishing service owns its own outbox table.

Example outbox structure:

| Column | Purpose |
|---|---|
| id | Unique outbox record ID |
| event_id | Globally unique event identifier |
| event_type | Event name |
| event_version | Event schema version |
| aggregate_id | Related business entity ID |
| payload | Event data |
| occurred_at | Event creation timestamp |
| published_at | Successful publication timestamp |
| created_at | Record creation timestamp |

### Order Service Example

When a customer creates an order:

1. Begin a PostgreSQL transaction.
2. Insert the order.
3. Insert order items.
4. Insert the `OrderCreated` event into the outbox table.
5. Commit the transaction.

Both the order and outbox record are saved atomically.

If the transaction fails, neither is committed.

If the transaction succeeds, the event remains available for publishing even if RabbitMQ is temporarily unavailable.

### Outbox Publisher

A background publisher runs independently within the service.

Its responsibilities are:

1. Find unpublished outbox records.
2. Publish events to RabbitMQ.
3. Wait for publisher confirmation.
4. Mark successfully published records.
5. Retry failed publications using a controlled policy.

The publisher must not mark an event as published before receiving broker confirmation.

If the broker accepts an event but the publisher crashes before updating `published_at`, the event may be published again.

This is expected and is why consumers must be idempotent.

## 2. Idempotent Consumers

An idempotent consumer can process the same event multiple times without applying the same business operation more than once.

Every event must have a unique `eventId`.

Each consuming service maintains a record of processed event IDs.

Example table:

### processed_events

| Column | Purpose |
|---|---|
| event_id | Unique event identifier |
| event_type | Event name |
| processed_at | Processing timestamp |

The database must enforce:

`UNIQUE(event_id)`

### Consumer Processing Flow

When Inventory Service receives `OrderCreated`:

1. Receive the RabbitMQ message.
2. Begin a database transaction.
3. Check whether the event ID has already been processed.
4. If already processed, do not repeat the business operation.
5. If not processed, validate the order items and stock.
6. Reserve inventory atomically.
7. Save inventory reservation and movement records.
8. Record the event ID in `processed_events`.
9. Save the resulting event in Inventory Service's outbox.
10. Commit the transaction.
11. Acknowledge the RabbitMQ message.

The inventory operation, processed-event record, and resulting outbox event must be committed together.

If the transaction fails, none of those changes should be committed.

The message must remain eligible for retry.

## 3. Acknowledgement Strategy

Consumers will use manual acknowledgements.

A message must be acknowledged only after its database transaction commits successfully.

Expected behavior:

| Situation | Consumer behavior |
|---|---|
| Business operation succeeds | Commit and acknowledge |
| Duplicate event | Skip business operation and acknowledge |
| Temporary database failure | Do not acknowledge; retry through configured failure handling |
| Invalid or permanently unprocessable message | Route through the configured DLQ policy |
| Consumer crashes before commit | Message can be redelivered |
| Consumer crashes after commit but before acknowledgement | Message can be redelivered; idempotency prevents duplicate effects |

Acknowledgements must never happen before successful database processing.

## 4. Retry and Dead Letter Queue Strategy

Temporary technical failures should be retried.

Examples:

- Temporary database connectivity failure.
- RabbitMQ connectivity failure.
- Transient infrastructure errors.
- Temporary downstream dependency failures.

Retries must be bounded and use exponential backoff with jitter.

After the configured retry limit, the message should be routed to a Dead Letter Queue for investigation and controlled replay.

Business rejections must be treated differently.

For example, insufficient inventory is not a technical failure.

Inventory Service should successfully process the `OrderCreated` event as a business decision, record the rejection result, and publish `InventoryReservationRejected`.

It should not repeatedly retry the event because stock is insufficient.

## 5. Event Envelope

All integration events should follow a consistent envelope.

Required fields:

| Field | Purpose |
|---|---|
| eventId | Unique event identifier |
| eventType | Event name |
| eventVersion | Schema version |
| occurredAt | Event creation time |
| producer | Publishing service |
| tenantId | Tenant context |
| correlationId | Identifies the overall workflow |
| causationId | Identifies the event or command that caused this event |
| payload | Business event data |

Example event:

```json
{
  "eventId": "event-uuid",
  "eventType": "OrderCreated",
  "eventVersion": 1,
  "occurredAt": "2026-11-04T10:00:00Z",
  "producer": "order-service",
  "tenantId": "tenant-uuid",
  "correlationId": "correlation-uuid",
  "causationId": "request-uuid",
  "payload": {
    "orderId": "order-uuid",
    "warehouseId": "warehouse-uuid",
    "items": [
      {
        "productId": "product-uuid",
        "quantity": 2
      }
    ]
  }
}
```

The example identifiers are illustrative.

Events must not contain passwords, authentication tokens, or other unnecessary sensitive information.

## 6. Service Ownership

Every service owns its own outbox and processed-event records.

| Service | Outbox responsibility |
|---|---|
| Order Service | Publishes order lifecycle events |
| Inventory Service | Publishes inventory reservation and release results |
| Product Service | Publishes product lifecycle events when required |
| Warehouse Service | Publishes warehouse lifecycle events when required |

Consumers maintain their own processed-event records.

No service may read or modify another service's outbox or processed-event table.

## 7. Alternatives Considered

### 1. Direct Database Write Followed by RabbitMQ Publish

Advantages:
- Simple initial implementation.
- Fewer database tables and background processes.

Disadvantages:
- Database and broker operations are not atomic.
- Events can be lost if the broker is unavailable.
- Database and event state can become inconsistent.

Not selected.

### 2. Distributed Transactions / Two-Phase Commit

Advantages:
- Can coordinate multiple transactional resources.

Disadvantages:
- Adds significant infrastructure and operational complexity.
- Reduces availability and increases coupling.
- Not appropriate for the asynchronous microservice workflow.

Not selected.

### 3. Transactional Outbox

Advantages:
- Atomic business update and event persistence.
- Supports recovery after broker outages.
- Works with local service-owned databases.
- Fits event-driven microservice architecture.

Disadvantages:
- Requires an outbox publisher.
- Published records require cleanup or archival.
- Duplicate publication remains possible.
- Monitoring and retry management are required.

Selected.

## 8. Consequences

### Positive

- Reliable event persistence.
- Reduced risk of lost integration events.
- Safe handling of duplicate message delivery.
- Better recovery from temporary infrastructure failures.
- No distributed database transaction required.
- Clear ownership of event publishing and processing.

### Negative

- Additional tables and background publisher processes.
- Event delivery may be delayed.
- Duplicate delivery must still be handled.
- Operational monitoring and DLQ management are required.

## 9. Implementation Notes

1. Create an `outbox_events` table in each publishing service.
2. Save business changes and outbox records in one TypeORM transaction.
3. Implement a background outbox publisher.
4. Use RabbitMQ publisher confirms.
5. Mark records published only after broker confirmation.
6. Create a `processed_events` table for consumers.
7. Add a unique constraint on `event_id`.
8. Process business changes and processed-event records in one local transaction.
9. Acknowledge messages only after transaction commit.
10. Configure bounded retry policies and Dead Letter Queues.
11. Add monitoring for outbox backlog, publication failures, consumer failures, and DLQ messages.
12. Implement safe DLQ inspection and replay procedures.

Outbox publishers must also handle concurrent workers safely, for example through database row locking and `FOR UPDATE SKIP LOCKED`, or an equivalent claiming mechanism.

Published outbox records should be retained for an appropriate period and then archived or cleaned up according to an operational policy.

## 10. Testing Requirements

The implementation must test:

- Database transaction rollback.
- Database commit while RabbitMQ is unavailable.
- Publisher crash after broker confirmation but before marking the record published.
- Duplicate event delivery.
- Consumer crash before database commit.
- Consumer crash after commit but before acknowledgement.
- Concurrent processing of the same event.
- Insufficient inventory as a business rejection.
- Retry exhaustion and DLQ routing.
- Safe replay of a DLQ message.

## References

- [Microservices.io: Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [Microservices.io: Idempotent Consumer Pattern](https://microservices.io/patterns/communication-style/idempotent-consumer.html)
- [RabbitMQ: Publisher Confirms](https://www.rabbitmq.com/docs/confirms)
- [RabbitMQ: Consumer Acknowledgements](https://www.rabbitmq.com/docs/confirms)