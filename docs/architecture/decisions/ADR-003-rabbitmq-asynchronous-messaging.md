# ADR-003: RabbitMQ for Asynchronous Messaging

## Status

Accepted

## Date

2026-11-04

## Context

StockFlow consists of independently deployable microservices. These services need to communicate without becoming tightly coupled to each other's availability or internal implementation.

Consider our first end-to-end workflow:

1. A customer creates an order.
2. Order Service saves the order.
3. Inventory Service checks and reserves stock.
4. Inventory Service reports the reservation result.
5. Order Service confirms or rejects the order.
6. Notification Service informs the customer.

If every operation uses synchronous REST calls, Order Service must wait for Inventory Service to finish processing before responding.

This introduces several problems:

- Services become dependent on each other's availability.
- Slow downstream operations increase request latency.
- Temporary failures can interrupt the entire workflow.
- Long-running operations become difficult to manage.
- Services become more tightly coupled.

We need a messaging system that allows services to publish events and process them independently.

## Decision

We will use **RabbitMQ as the primary message broker for asynchronous communication between StockFlow microservices**.

Services will publish messages to RabbitMQ and consume messages from their own queues.

The broker will provide message routing, buffering, acknowledgements, and redelivery.

Services remain responsible for their own business logic, database transactions, idempotency, and failure recovery.

## Why RabbitMQ?

RabbitMQ is a message broker that allows applications to exchange messages through exchanges and queues.

A producer publishes a message to an exchange. RabbitMQ routes that message to one or more queues based on the exchange type, routing key, and bindings.

A consumer receives messages from its queue and processes them.

This allows producers and consumers to operate independently.

### Benefits for StockFlow

1. **Loose coupling:** Producers do not need to know the internal implementation of consumers.
2. **Asynchronous processing:** A service can continue its work without waiting for another service to finish.
3. **Message buffering:** Messages can wait in queues while consumers are temporarily unavailable.
4. **Acknowledgements:** Consumers can acknowledge messages after successful processing.
5. **Redelivery:** Messages can be delivered again when processing fails.
6. **Routing:** Exchanges can distribute messages to appropriate queues.
7. **Independent scaling:** Consumer instances can be increased when processing demand grows.

## Communication Strategy

StockFlow will use two primary communication mechanisms.

### 1. Synchronous REST Communication

REST is used when a service needs an immediate response.

Examples:

- Order Service validates products through Product Service.
- Order Service validates a warehouse through Warehouse Service.
- API Gateway forwards frontend requests to backend services.

### 2. Asynchronous RabbitMQ Communication

RabbitMQ is used for workflows that can be processed independently.

Examples:

- Order creation triggers inventory reservation.
- Inventory reservation results update order status.
- Order confirmation triggers customer notification.
- Order cancellation triggers inventory release.

We will not replace every REST call with RabbitMQ. Each communication mechanism has a different purpose.

## RabbitMQ Core Concepts

### 1. Producer

A producer is an application or service that publishes messages.

Examples:

- Order Service publishes `OrderCreated`.
- Inventory Service publishes `InventoryReserved`.
- Order Service publishes `OrderConfirmed`.

### 2. Exchange

An exchange receives messages from producers and routes them to queues.

We will initially use a durable topic exchange named:

`stockflow.events`

A topic exchange routes messages based on routing-key patterns.

### 3. Queue

A queue stores messages until consumers process them.

Each consumer group will have its own queue.

For example:

```text
stockflow.order.inventory
stockflow.inventory.order
stockflow.order.notification
```

These queues allow each service to process relevant messages independently.

### 4. Routing Key

A routing key identifies the type of message being published.

Examples:

```text
order.created
inventory.reserved
inventory.reservation_rejected
order.confirmed
order.rejected
order.cancellation_requested
inventory.released
```

### 5. Binding

A binding connects a queue to an exchange using a routing pattern.

For example:

```text
Exchange: stockflow.events

Queue: stockflow.order.inventory
Binding: order.created

Queue: stockflow.inventory.order
Binding: inventory.reserved
Binding: inventory.reservation_rejected

Queue: stockflow.order.notification
Binding: order.confirmed
Binding: order.rejected
```

The exchange routes each message to queues whose bindings match its routing key.

## Initial RabbitMQ Topology

```text
                         stockflow.events
                         (Topic Exchange)
                                |
              +-----------------+------------------+
              |                 |                  |
              v                 v                  v
     order.inventory     inventory.order    order.notification
          Queue                Queue               Queue
              |                 |                  |
              v                 v                  v
       Inventory Service   Order Service    Notification Service
```

The topology may grow as additional services and workflows are introduced.

## Event Naming Convention

Events represent facts that have already happened.

Examples:

| Event | Producer | Consumer |
|---|---|---|
| `OrderCreated` | Order Service | Inventory Service |
| `InventoryReserved` | Inventory Service | Order Service |
| `InventoryReservationRejected` | Inventory Service | Order Service |
| `OrderConfirmed` | Order Service | Notification Service |
| `OrderRejected` | Order Service | Notification Service |
| `OrderCancellationRequested` | Order Service | Inventory Service |
| `InventoryReleased` | Inventory Service | Order Service |

The routing key uses lowercase dot-separated names.

For example:

```text
Event: OrderCreated
Routing key: order.created
```

We will keep event names consistent across producers, consumers, documentation, and shared contracts.

## Event Envelope

All integration events should follow a common envelope structure.

```json
{
  "eventId": "unique-event-id",
  "eventType": "OrderCreated",
  "eventVersion": 1,
  "occurredAt": "2026-11-04T10:00:00.000Z",
  "producer": "order-service",
  "tenantId": "tenant-id",
  "correlationId": "workflow-id",
  "causationId": "causing-event-id",
  "payload": {
    "orderId": "order-id",
    "warehouseId": "warehouse-id",
    "items": [
      {
        "productId": "product-id",
        "quantity": 2
      }
    ]
  }
}
```

### Envelope Fields

- `eventId`: Globally unique event identifier.
- `eventType`: Name of the event.
- `eventVersion`: Version of the event contract.
- `occurredAt`: Time the event was created.
- `producer`: Service that published the event.
- `tenantId`: Tenant associated with the business operation.
- `correlationId`: Connects messages belonging to the same workflow.
- `causationId`: Identifies the event that caused the current event.
- `payload`: Event-specific business information.

The envelope is an integration contract, not a shared database model.

It must not contain passwords, access tokens, refresh tokens, or other secrets.

## Transactional Outbox Integration

A critical problem exists when a service updates its database and publishes an event.

Consider:

1. Order Service saves an order in PostgreSQL.
2. Order Service publishes `OrderCreated` to RabbitMQ.
3. RabbitMQ is unavailable.

The order is saved, but the event is not published.

This can leave the workflow stuck.

To solve this, we will use the **Transactional Outbox pattern**.

### How It Works

Within one local database transaction, Order Service:

1. Creates the order.
2. Creates order items.
3. Saves the idempotency record.
4. Inserts an `OrderCreated` record into its outbox table.
5. Commits the transaction.

A separate publisher process reads unpublished outbox records and publishes them to RabbitMQ.

After receiving a publisher confirmation, it marks the outbox record as published.

Inventory Service will use the same pattern when publishing reservation results.

### Important Rule

The outbox record and the business database changes must be committed in the same local transaction.

We must never depend on a separate database transaction followed by an unreliable broker publish.

Publisher confirmations reduce the risk of losing messages during publishing, but they do not guarantee exactly-once delivery.

If a publisher crashes after RabbitMQ accepts a message but before the outbox record is marked published, it may publish that message again.

Consumers must handle duplicates.

## Message Delivery Guarantees

We will design for **at-least-once message delivery**.

This means a message may be delivered more than once.

We will not assume exactly-once processing.

### Why Duplicate Delivery Happens

For example:

1. Inventory Service receives `OrderCreated`.
2. It reserves inventory and commits its database transaction.
3. The service crashes before acknowledging the RabbitMQ message.
4. RabbitMQ redelivers the message.

Without idempotency, Inventory Service could reserve the same stock twice.

Therefore, every consumer must be idempotent.

### Consumer Idempotency

Each consuming service maintains a processed-event record in its own database.

Example:

```text
processed_events
--------------------------------
event_id
processed_at
```

A consumer:

1. Receives the message.
2. Starts a local database transaction.
3. Checks whether the event ID was already processed.
4. If already processed, skips the business operation.
5. Otherwise, performs the business operation.
6. Saves the processed-event record.
7. Saves any outgoing events to its outbox.
8. Commits the transaction.
9. Acknowledges the RabbitMQ message.

The processed-event record and business changes must be committed together.

## Message Acknowledgement

We will use manual acknowledgements for important business consumers.

The consumer acknowledges a message only after its local database transaction commits successfully.

### Successful Processing

```text
Receive message
      |
      v
Process message
      |
      v
Commit database transaction
      |
      v
Acknowledge message
```

### Failed Processing

If processing fails before the database transaction commits, the consumer must not acknowledge the message as successfully processed.

RabbitMQ can then redeliver it according to the configured retry strategy.

Acknowledging before committing the database transaction is not allowed for these business workflows.

## Retry and Dead-Letter Strategy

Not every failure is permanent.

For example:

- A temporary database outage may recover.
- A service may restart.
- A downstream dependency may become available again.

We will implement bounded retries.

### Retry Strategy

Initially, we will use retry queues with message TTL and dead-letter routing.

A simplified flow:

```text
Main Queue
    |
    v
Consumer processing fails
    |
    v
Retry Queue
    |
    v
Wait for configured TTL
    |
    v
Return message to Main Queue
    |
    v
Retry processing
    |
    v
Retry limit exceeded
    |
    v
Dead-Letter Queue
```

The actual topology will be configured per consumer group.

Retry attempts must be bounded to avoid infinite retry loops.

### Dead-Letter Queue

A dead-letter queue stores messages that cannot be processed successfully after the configured retry policy.

Examples:

- Invalid event payload.
- Unsupported event version.
- Persistent application failure.
- Repeated database errors.

Messages in the dead-letter queue must be monitored and investigated.

They must not be silently discarded.

Replaying a dead-letter message requires understanding whether its original business operation was partially completed and whether replay is safe.

### Business Rejection Is Not a Technical Failure

Insufficient inventory is a valid business outcome.

It must produce `InventoryReservationRejected`.

It should not be retried as though it were a temporary infrastructure failure.

## Reliability Configuration

We will initially apply the following configuration principles:

| Component | Configuration |
|---|---|
| Exchange | Durable topic exchange |
| Business queues | Durable queues |
| Messages | Persistent delivery mode |
| Publisher | Publisher confirmations |
| Consumer | Manual acknowledgements |
| Processing | Idempotent consumers |
| Retry | Bounded retries |
| Permanent failures | Dead-letter queues |
| Outgoing events | Transactional Outbox |
| Monitoring | Queue depth, unacknowledged messages, failures |

We will evaluate RabbitMQ quorum queues for important production workloads based on availability requirements and operational capacity.

The initial implementation must not claim high availability merely because queues are durable.

Durability and high availability are different concerns.

## Ordering and Concurrency

We must not assume that all messages are globally ordered.

Multiple consumers, retries, and concurrent processing can change the order in which messages are completed.

Important rules:

1. Use order IDs, reservation IDs, and event IDs to identify business operations.
2. Validate the current business state before applying a transition.
3. Make event consumers idempotent.
4. Prevent invalid state transitions.
5. Use database concurrency controls for inventory operations.
6. Use correlation IDs to trace distributed workflows.

If strict ordering is required for a specific business entity in the future, we will design an explicit ordering strategy rather than relying on global broker ordering.

## Event Contract Management

Shared event contracts will live in:

`packages/contracts/`

This package may contain:

- Event envelope types.
- Event payload interfaces.
- Event version definitions.
- Routing key constants.

It must not contain:

- TypeORM entities.
- Database repository implementations.
- Service-specific business logic.
- Cross-service domain models.

Event contracts must evolve carefully.

Consumers should reject or safely handle unsupported versions rather than interpreting incompatible payloads silently.

Backward-compatible contract changes are preferred.

## Security Considerations

RabbitMQ must be treated as an internal infrastructure component.

We will:

1. Use service-specific RabbitMQ credentials where appropriate.
2. Apply permissions to exchanges and queues.
3. Avoid exposing RabbitMQ management interfaces publicly.
4. Use TLS for broker connections in production.
5. Validate incoming event payloads.
6. Validate tenant context before processing business operations.
7. Never treat a message's `tenantId` alone as proof of authorization.
8. Avoid logging secrets or sensitive payload information.

## Alternatives Considered

### 1. Synchronous REST for Every Interaction

**Advantages:**

- Simple request-response model.
- Immediate success or failure response.
- Fewer messaging concepts to learn initially.

**Disadvantages:**

- Strong runtime dependency between services.
- Increased request latency.
- Downstream failures can interrupt upstream operations.
- Poor fit for long-running workflows.
- More difficult buffering and retry handling.

Rejected as the only communication strategy.

REST remains appropriate for immediate queries and validation.

### 2. RabbitMQ

**Advantages:**

- Flexible routing.
- Acknowledgements and redelivery.
- Durable queues and messages.
- Supports asynchronous workflows.
- Mature ecosystem and tooling.

**Disadvantages:**

- Additional infrastructure to operate.
- Requires careful retry and dead-letter configuration.
- At-least-once delivery requires idempotency.
- Eventual consistency makes workflows more complex to debug.

Selected for StockFlow.

### 3. Apache Kafka

**Advantages:**

- Distributed event streaming.
- Partition-based scalability.
- Event retention and replay capabilities.
- Strong fit for high-volume streaming and analytics workloads.

**Disadvantages:**

- Additional operational complexity for our initial requirements.
- Different messaging and consumer-group model.
- More infrastructure than the initial MVP needs.

We may evaluate Kafka in the future if event-streaming, replay, or throughput requirements justify it.

## Consequences

### Positive Consequences

- Services become less dependent on each other's immediate availability.
- Long-running workflows can execute asynchronously.
- Messages can be buffered during temporary service outages.
- Producers and consumers can scale independently.
- Events provide a foundation for future integrations.

### Negative Consequences

- The platform gains additional infrastructure complexity.
- Eventual consistency must be handled explicitly.
- Duplicate delivery must be expected.
- Retry and dead-letter operations require monitoring.
- Distributed debugging requires correlation IDs and tracing.
- Event contracts must be maintained carefully.

## Implementation Notes

We will implement RabbitMQ integration in stages:

1. Add RabbitMQ to the local Docker Compose environment.
2. Configure the topic exchange and durable queues.
3. Define shared event contracts.
4. Implement the Order Service transactional outbox.
5. Implement the outbox publisher with publisher confirmations.
6. Implement Inventory Service event consumption.
7. Add manual acknowledgement and idempotent processing.
8. Implement Inventory Service outbox publishing.
9. Implement Order Service result-event consumption.
10. Add retry queues and dead-letter handling.
11. Add integration tests for duplicate delivery and service failures.
12. Add monitoring for queue depth, processing failures, and dead-letter messages.

We will first implement the `OrderCreated` to `InventoryReserved` workflow and verify it before expanding messaging across all services.

## References

- RabbitMQ Documentation: https://www.rabbitmq.com/docs
- RabbitMQ Exchanges: https://www.rabbitmq.com/docs/exchanges
- RabbitMQ Publisher Confirms: https://www.rabbitmq.com/docs/confirms
- RabbitMQ Dead Letter Exchanges: https://www.rabbitmq.com/docs/dlx
- Microservices.io Transactional Outbox: https://microservices.io/patterns/data/transactional-outbox.html
- Microservices.io Idempotent Consumer: https://microservices.io/patterns/communication-style/idempotent-consumer.html