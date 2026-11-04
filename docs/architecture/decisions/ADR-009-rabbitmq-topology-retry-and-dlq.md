# ADR-009: RabbitMQ Topology, Retry and Dead Letter Queues

- **Status:** Accepted
- **Date:** 2026-11-04

## 1. Context

StockFlow uses RabbitMQ for asynchronous communication between independently deployed microservices.

The Order and Inventory services exchange events to coordinate order creation, inventory reservation, confirmation, rejection and cancellation. The Notification service consumes order outcome events.

A production messaging system must handle:
- Temporary service and network failures.
- Duplicate message delivery.
- Consumer crashes.
- Invalid or unprocessable messages.
- Retry without overwhelming services.
- Messages that repeatedly fail processing.
- Monitoring and operational recovery.

We need a consistent RabbitMQ topology and failure-handling strategy.

## 2. Decision

StockFlow will use a **topic exchange**, durable queues, persistent messages, publisher confirms, manual consumer acknowledgements, bounded retries and dead letter queues.

The messaging system will provide **at-least-once delivery**, not exactly-once delivery.

Consumers must be idempotent because a message can be delivered more than once.

## 3. Exchange and Queue Topology

### Main exchange

Use the following exchange:

`stockflow.events`

- Type: Topic
- Durable: Yes
- Responsibility: Route published events to queues using routing keys.

A topic exchange allows routing keys to be matched against queue bindings. It does not store messages itself; queues hold messages until consumers process them.

### Main queues

| Queue | Owning consumer | Purpose |
|---|---|---|
| `stockflow.order.inventory` | Inventory Service | Consume order events requiring inventory processing |
| `stockflow.inventory.order` | Order Service | Consume inventory reservation and release results |
| `stockflow.order.notification` | Notification Service | Consume order outcome events |

### Routing keys

| Event | Routing key |
|---|---|
| Order created | `order.created` |
| Inventory reserved | `inventory.reserved` |
| Inventory reservation rejected | `inventory.reservation_rejected` |
| Order confirmed | `order.confirmed` |
| Order rejected | `order.rejected` |
| Order cancellation requested | `order.cancellation_requested` |
| Inventory released | `inventory.released` |

Each service owns its consumer queues and bindings. Shared exchange topology is managed as infrastructure configuration.

## 4. Publisher Responsibilities

A publishing service must:

1. Save its business changes and outgoing event in its local database transaction using the Transactional Outbox pattern.
2. Publish the event to `stockflow.events` using the appropriate routing key.
3. Use persistent messages.
4. Enable publisher confirms.
5. Mark the outbox event as published only after receiving broker confirmation.
6. Retry publishing when confirmation fails or the connection is interrupted.

An event must not be considered safely published merely because the application attempted to send it.

## 5. Consumer Responsibilities

Every consumer must:

1. Receive the message.
2. Validate the event envelope and payload.
3. Check whether the event has already been processed.
4. Execute the business operation in a local database transaction.
5. Record the processed event and any outgoing events in that same transaction.
6. Acknowledge the RabbitMQ message only after the transaction commits.

If processing fails before the database transaction commits, the consumer must not acknowledge the message as successfully processed.

If the consumer crashes after committing but before acknowledging, RabbitMQ may redeliver the message. The idempotency check prevents the business operation from being applied twice.

## 6. Event Envelope

All events should follow a consistent envelope:

```json
{
  "eventId": "unique-event-id",
  "eventType": "OrderCreated",
  "eventVersion": 1,
  "occurredAt": "2026-11-04T10:00:00.000Z",
  "producer": "order-service",
  "tenantId": "tenant-id",
  "correlationId": "workflow-correlation-id",
  "causationId": "causing-event-id",
  "payload": {}
}
```

Rules:
- `eventId` uniquely identifies the event.
- `eventType` identifies the business event.
- `eventVersion` supports contract evolution.
- `tenantId` identifies the tenant context.
- `correlationId` connects events belonging to one workflow.
- `causationId` identifies the event or command that caused this event.
- Payloads must not contain passwords, tokens, secrets or unnecessary sensitive information.

Consumers must validate tenant context and authorization assumptions. They must not trust tenant information as permission to access another tenant's data.

## 7. Retry Strategy

Temporary technical failures may be retried.

Examples:
- Temporary database connectivity failure.
- Temporary dependency outage.
- Transient network failure.
- Temporary resource exhaustion.

Use bounded retries with delayed retry queues. A retry queue can use message TTL and dead-letter routing to return messages to the main processing queue after the delay.

Initial retry policy:

| Attempt | Suggested delay |
|---|---:|
| First retry | 5 seconds |
| Second retry | 30 seconds |
| Third retry | 2 minutes |

These are initial defaults, not permanent requirements. Add random jitter and make retry settings configurable.

After the retry limit is exhausted, route the message to the appropriate dead letter queue.

Retries must be bounded. Infinite immediate retries can create message storms and overload an already unhealthy service.

## 8. Dead Letter Queue (DLQ)

A Dead Letter Queue stores messages that cannot be processed successfully after the configured retry policy or that are rejected as permanently invalid.

DLQs are used for:
- Invalid message formats.
- Unsupported event versions.
- Repeated technical processing failures.
- Messages that exceed the retry limit.

A DLQ message must retain enough metadata for investigation, including the original event ID, routing key, failure reason and retry history.

DLQ replay must be a controlled and auditable operational action. Replayed messages must still pass through normal consumer validation and idempotency checks.

### Business rejection is not a DLQ failure

Insufficient stock is a valid business outcome, not a technical processing failure.

For example:
1. Inventory Service receives an order event.
2. It checks stock.
3. Stock is insufficient.
4. It records the rejection result.
5. It publishes `InventoryReservationRejected`.
6. It acknowledges the original message after the transaction commits.

The message was processed successfully. It must not be retried or sent to the DLQ merely because the business operation was rejected.

## 9. Delivery Guarantees and Ordering

RabbitMQ provides at-least-once delivery in this architecture.

Consequences:
- Duplicate delivery is possible.
- Consumers must be idempotent.
- Publisher confirms and consumer acknowledgements serve different purposes.
- Acknowledging a message before committing the database transaction risks losing work.

StockFlow does not assume global message ordering.

Multiple consumers, retries and redelivery can affect processing order. Where ordering matters:
- Include aggregate identifiers such as `orderId` or `reservationId`.
- Validate the current aggregate state before applying a transition.
- Use event versions or sequence numbers where required.
- Make state transitions safe against stale or duplicate events.

## 10. Consumer Scaling and Prefetch

Multiple consumer instances may consume from the same queue for horizontal scaling.

RabbitMQ distributes messages among available consumers. Configure prefetch to limit the number of unacknowledged messages assigned to a consumer.

Start with a conservative prefetch value and tune it using throughput, processing latency and memory measurements.

Avoid excessively high prefetch values because they can increase memory usage and leave messages unevenly distributed among consumers.

## 11. Queue Durability and High Availability

- Exchanges and queues must be durable.
- Published messages must be persistent.
- Publisher confirms must be enabled.
- Consumer acknowledgements must be manual.
- Critical production queues should be evaluated for RabbitMQ quorum queues.

Durable queues and persistent messages alone do not guarantee high availability. Production deployment must also consider broker replication, node failures, storage, network partitions and recovery procedures.

## 12. Tenant Isolation

StockFlow will not create a separate RabbitMQ queue for every tenant.

Instead:
- Events carry `tenantId`.
- Consumers validate tenant context.
- Database queries always include tenant scope.
- Authorization rules remain enforced by the consuming service.
- No service accesses another service's database.

This keeps broker topology manageable while preserving tenant isolation.

## 13. Monitoring and Operations

Monitor at least:

- Queue depth.
- Age of the oldest queued message.
- Number of unacknowledged messages.
- Redelivery count.
- Publish confirmation failures.
- Consumer processing failures.
- Retry queue depth.
- DLQ message count.
- Consumer throughput and processing latency.

Alert on sustained queue growth, old messages, repeated consumer failures and unexpected DLQ accumulation.

DLQs must be inspected and resolved through an operational process rather than silently ignored.

## 14. Alternatives Considered

### Direct exchange

A direct exchange routes messages using exact routing-key matches. It is simpler but less flexible for future event subscriptions.

### Fanout exchange

A fanout exchange broadcasts messages to every bound queue. This is useful for broadcast events but does not provide the routing flexibility needed for StockFlow's event types.

### Kafka

Kafka is suitable for high-throughput event streaming, durable event logs and replay-oriented architectures. RabbitMQ is selected for the current MVP because the workflow primarily requires reliable asynchronous service messaging and routing.

Kafka may be reconsidered if future requirements justify a streaming platform.

## 15. Consequences

### Positive

- Consistent messaging conventions across services.
- Controlled retries and failure handling.
- Better recovery from temporary outages.
- Duplicate delivery is handled safely.
- Poison messages can be isolated for investigation.
- Services can scale consumers independently.

### Trade-offs

- More infrastructure configuration.
- Additional retry and DLQ queues.
- Operational responsibility for monitoring and replay.
- At-least-once delivery requires idempotent consumers.
- Ordering guarantees require application-level design.

## 16. Implementation Notes

1. Define exchange, queue, binding and retry topology as version-controlled infrastructure configuration.
2. Create service-owned queue bindings.
3. Implement the outbox publisher in Order and Inventory services.
4. Implement idempotent consumers.
5. Configure manual acknowledgements and publisher confirms.
6. Add bounded retry queues and DLQs.
7. Add structured logs with event ID and correlation ID.
8. Add integration tests for duplicate delivery, retries, DLQ routing and broker outages.
9. Evaluate quorum queues and broker high availability before production deployment.
10. Document and test the DLQ replay procedure.

## 17. References

- RabbitMQ Exchanges: https://www.rabbitmq.com/docs/exchanges
- RabbitMQ Queues: https://www.rabbitmq.com/docs/queues
- RabbitMQ Dead Letter Exchanges: https://www.rabbitmq.com/docs/dlx
- RabbitMQ Publisher Confirms: https://www.rabbitmq.com/docs/confirms
- RabbitMQ Consumer Prefetch: https://www.rabbitmq.com/docs/confirms#channel-qos
- RabbitMQ Quorum Queues: https://www.rabbitmq.com/docs/quorum-queues
