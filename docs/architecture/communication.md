# StockFlow — Inter-Service Communication Design

## 1. Purpose

This document defines how StockFlow microservices communicate with each other.

The system uses:

- REST APIs for synchronous communication.
- RabbitMQ for asynchronous communication.
- Transactional Outbox for reliable event publishing.
- Idempotent consumers for duplicate message protection.

The objective is to reduce service coupling while maintaining reliable business workflows.

## 2. Communication Patterns

### 2.1 Synchronous Communication — REST

REST is used when a service requires an immediate response from another service.

Examples:

- Order Service requests product information from Product Service.
- Order Service validates warehouse information through Warehouse Service.
- API Gateway forwards requests to backend services.

Rules:

- Every internal API call must have a timeout.
- Services must handle unavailable dependencies.
- Avoid unnecessary chains of synchronous service calls.
- Never access another service's database directly.
- Validate tenant authorization at the receiving service.

### 2.2 Asynchronous Communication — RabbitMQ

RabbitMQ is used when a business operation can continue without waiting for an immediate response.

Examples:

- OrderCreated
- InventoryReserved
- InventoryReservationRejected
- OrderConfirmed
- OrderRejected
- OrderCancellationRequested
- InventoryReleased

Rules:

- Messages must have a well-defined contract.
- Consumers must support duplicate delivery.
- Important messages must be durable.
- Consumers acknowledge messages only after successful processing.
- Failed messages must follow a controlled retry strategy.

## 3. Command vs Event

### Commands

A command requests that a service perform an operation.

Example:

`ReserveInventory`

Meaning: Inventory Service is requested to reserve stock for an order.

### Events

An event communicates that something has already happened.

Examples:

- `OrderCreated`
- `InventoryReserved`
- `InventoryReservationRejected`
- `OrderConfirmed`

Meaning: These events describe completed business facts.

Commands and events must have clear ownership and naming.

## 4. Service Communication Matrix

| Producer | Consumer | Communication | Purpose |
|---|---|---|---|
| Frontend | API Gateway | REST | External requests |
| API Gateway | Auth Service | REST | Authentication |
| API Gateway | Tenant Service | REST | Tenant authorization/context |
| Order Service | Product Service | REST | Product validation and snapshots |
| Order Service | Warehouse Service | REST | Warehouse validation |
| Order Service | Inventory Service | RabbitMQ | Inventory reservation workflow |
| Inventory Service | Order Service | RabbitMQ | Reservation result |
| Order Service | Notification Service | RabbitMQ | Order notifications |
| Product Service | Other services | Events when needed | Product lifecycle changes |

This matrix describes the initial MVP design. Additional communication should be introduced only when a real business requirement needs it.

## 5. Event Envelope Structure

Every business event should use a consistent envelope.

Required metadata:

| Field | Purpose |
|---|---|
| eventId | Globally unique event identifier |
| eventType | Event name |
| eventVersion | Event schema version |
| occurredAt | Event creation timestamp |
| producer | Publishing service |
| tenantId | Tenant associated with the event |
| correlationId | Connects related operations |
| causationId | Identifies the event or command that caused this event |
| payload | Event-specific business data |

Example:

```json
{
  "eventId": "8a6c5f30-9d9b-4d51-9c7d-2d6c0a5c1001",
  "eventType": "OrderCreated",
  "eventVersion": 1,
  "occurredAt": "2026-11-04T10:30:00.000Z",
  "producer": "order-service",
  "tenantId": "tenant-123",
  "correlationId": "correlation-456",
  "causationId": null,
  "payload": {
    "orderId": "order-789",
    "warehouseId": "warehouse-101",
    "items": [
      {
        "productId": "product-201",
        "quantity": 2
      }
    ]
  }
}
```

Do not include passwords, access tokens, refresh tokens, or other secrets in events.

## 6. Transactional Outbox Pattern

### Problem

A service may successfully commit a database transaction but fail to publish the corresponding RabbitMQ event.

This creates inconsistent system state.

### Solution

Each publishing service maintains its own `outbox_events` table.

Within one local database transaction:

1. Save the business data.
2. Save the corresponding outbox event.
3. Commit both changes together.

A separate publisher process:

1. Reads unpublished outbox records.
2. Publishes events to RabbitMQ.
3. Waits for publisher confirmation.
4. Marks successfully published records as published.

If RabbitMQ is unavailable, the outbox event remains available for later publishing.

The Order Service and Inventory Service each own their own outbox table.

## 7. Idempotent Message Consumers

RabbitMQ provides at-least-once delivery in our design.

A consumer may receive the same message more than once.

Each consumer must therefore process messages idempotently.

Initial approach:

1. Receive the event.
2. Start a local database transaction.
3. Check whether the event ID has already been processed.
4. If already processed, skip the business operation.
5. Otherwise, perform the business operation.
6. Store the processed event ID.
7. Save any outgoing events in the local outbox.
8. Commit the transaction.
9. Acknowledge the RabbitMQ message.

The business operation and processed-event record must be committed atomically.

## 8. RabbitMQ Reliability Strategy

We will use:

- Durable exchanges and queues.
- Persistent messages.
- Publisher confirms.
- Manual consumer acknowledgements.
- Bounded retry attempts.
- Dead-letter queues.
- Monitoring for failed and delayed messages.

Repeated failures must not cause infinite immediate retries.

Poison messages should eventually move to a dead-letter queue for investigation and controlled replay.

## 9. Saga Pattern

StockFlow uses event-driven choreography for the initial order and inventory workflow.

No dedicated orchestration service is required for the MVP.

Each service owns its local transaction and responds through events.

Example:

1. Order Service creates an order.
2. Order Service publishes `OrderCreated`.
3. Inventory Service attempts reservation.
4. Inventory Service publishes `InventoryReserved` or `InventoryReservationRejected`.
5. Order Service updates the order status.
6. Order Service publishes `OrderConfirmed` or `OrderRejected`.
7. Notification Service processes the notification event.

This is a distributed workflow, not a single ACID transaction across multiple databases.

## 10. Failure Handling

| Failure | Expected behavior |
|---|---|
| RabbitMQ unavailable | Outbox records remain pending |
| Consumer crashes | Message can be redelivered |
| Duplicate event | Consumer detects processed event ID |
| Insufficient inventory | Reservation is rejected |
| Concurrent reservations | Inventory database concurrency control prevents overselling |
| Order update fails | Result event is retried safely |
| Notification Service unavailable | Notification event is retried without blocking order processing |
| Repeated message failure | Message moves to a dead-letter queue |

## 11. Observability

All service-to-service operations should carry:

- Correlation ID.
- Causation ID where applicable.
- Event ID.
- Tenant ID, handled carefully in logs.
- Structured logs.
- Processing duration and failure metrics.

Secrets and sensitive personal data must not be logged.

Distributed tracing will use OpenTelemetry.

## 12. Architecture Rules

- REST for immediate request-response operations.
- RabbitMQ for asynchronous business workflows.
- Transactional Outbox for reliable event publishing.
- Idempotent consumers for duplicate protection.
- No cross-service database access.
- Explicit event contracts and versioning.
- Bounded retries and dead-letter handling.
- Tenant context must be validated.
- No sensitive credentials in messages.

---

**Document Status:** Draft  
**Project:** StockFlow  
**Milestone:** 1 — Architecture and Foundation