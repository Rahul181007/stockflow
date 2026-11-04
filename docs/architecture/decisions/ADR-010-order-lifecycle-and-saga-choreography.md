# ADR-010: Order Lifecycle and Saga Choreography

- **Status:** Accepted
- **Date:** 2026-11-04

## 1. Context

StockFlow processes customer orders through the Order Service and reserves stock through the Inventory Service.

These services own separate databases. A customer order cannot be confirmed until inventory has been successfully reserved.

A distributed workflow creates challenges:
- Order creation and inventory reservation happen in different services.
- A service can become temporarily unavailable.
- Events can be delivered more than once.
- A reservation can succeed while the Order Service is temporarily unavailable.
- Order cancellation requires inventory to be released.
- Distributed database transactions are not appropriate across independent services.

StockFlow needs a reliable order lifecycle and a way to coordinate these operations without tightly coupling services.

## 2. Decision

StockFlow will use **Saga choreography** to coordinate the order and inventory workflow.

Each service owns its own state transitions and database transactions.

Services communicate through RabbitMQ events. Each service uses the Transactional Outbox pattern to publish outgoing events reliably and idempotent consumers to handle duplicate delivery.

The Order Service owns the order lifecycle. The Inventory Service owns inventory quantities and reservation lifecycle.

No service directly accesses another service's database.

## 3. Order Lifecycle

The Order Service owns the following order states:

| Status | Meaning |
|---|---|
| `PENDING_INVENTORY` | Order accepted for processing; inventory reservation is pending |
| `CONFIRMED` | All order items have been successfully reserved |
| `REJECTED` | Inventory reservation failed or order could not be fulfilled |
| `CANCELLATION_PENDING` | Cancellation requested; inventory release is pending |
| `CANCELLED` | Inventory has been successfully released |
 
### Valid state transitions

```text
                 Order Created
                       |
                       v
              PENDING_INVENTORY
                  /         \
                 /           \
        Reservation       Reservation
          succeeds          rejected
              |                 |
              v                 v
         CONFIRMED          REJECTED
              |
       Cancellation requested
              |
              v
      CANCELLATION_PENDING
              |
       Inventory released
              |
              v
          CANCELLED
```

### Transition rules

- A new order starts in `PENDING_INVENTORY`.
- An order becomes `CONFIRMED` only after successful reservation of all its items.
- An order becomes `REJECTED` when inventory cannot reserve the requested quantities.
- A confirmed order can move to `CANCELLATION_PENDING`.
- An order becomes `CANCELLED` only after Inventory confirms successful release.
- A rejected order cannot become confirmed through a late or duplicate reservation event.
- A cancelled order cannot be confirmed again.
- Invalid transitions must be rejected and logged.

Order state transitions must be performed transactionally in the Order Service database.

## 4. Saga Choreography

Saga choreography means that services coordinate a distributed business workflow by publishing and consuming events.

There is no central service controlling every step.

Each service reacts to events and publishes the result of its own operation.

### Services involved

| Service | Responsibility |
|---|---|
| Order Service | Create orders, manage order states, request cancellation |
| Inventory Service | Reserve and release stock, maintain inventory movements |
| Notification Service | Notify users of order outcomes |

### Order creation and confirmation

1. Customer submits an order through the API Gateway.
2. Order Service validates the request and checks product and warehouse information through their owning services.
3. Order Service saves the order with status `PENDING_INVENTORY`.
4. In the same database transaction, Order Service saves an `OrderCreated` event in its outbox.
5. The Outbox Publisher publishes `OrderCreated` to RabbitMQ.
6. Inventory Service consumes the event.
7. Inventory Service validates the requested items and attempts to reserve all quantities atomically.
8. If reservation succeeds, Inventory Service saves the reservation and publishes `InventoryReserved` through its outbox.
9. If reservation fails, Inventory Service publishes `InventoryReservationRejected` through its outbox.
10. Order Service consumes the result event.
11. Order Service updates the order to `CONFIRMED` or `REJECTED`.
12. Order Service publishes `OrderConfirmed` or `OrderRejected` through its outbox.
13. Notification Service consumes the outcome event and sends the appropriate notification.

The initial API response is `202 Accepted` because the order is accepted for asynchronous processing, not yet confirmed.

### Cancellation workflow

1. Customer requests cancellation of a confirmed order.
2. Order Service changes the status to `CANCELLATION_PENDING`.
3. Order Service saves an `OrderCancellationRequested` event in its outbox.
4. Inventory Service consumes the event.
5. Inventory Service releases the reservation transactionally.
6. Inventory Service publishes `InventoryReleased` through its outbox.
7. Order Service consumes `InventoryReleased`.
8. Order Service changes the status to `CANCELLED`.
9. Order Service publishes an order cancellation outcome event for Notification Service.

An order must not be marked `CANCELLED` before inventory release is confirmed.

## 5. Atomic Inventory Reservation

Inventory Service must reserve all requested order items or none of them.

For example, consider an order containing:

| Product | Requested quantity | Available quantity |
|---|---:|---:|
| Product A | 2 | 10 |
| Product B | 5 | 3 |

The reservation must fail because Product B does not have sufficient stock.

Product A must not remain partially reserved.

The Inventory Service must:
- Execute reservation checks and updates in one local database transaction.
- Prevent concurrent requests from reserving the same available stock.
- Record reservation details and inventory movements.
- Publish the reservation result through its outbox.
- Use idempotency to avoid duplicate reservations.

The database remains the source of truth for stock quantities.

## 6. Idempotency and Duplicate Events

RabbitMQ provides at-least-once delivery. A consumer may receive the same event more than once.

Every consumer must maintain processed-event records using a unique event ID.

For example:
- Inventory Service must not reserve stock twice for the same order event.
- Order Service must not confirm an order twice for duplicate reservation results.
- Inventory Service must not release the same reservation twice.
- Notification Service must avoid duplicate notifications where practical.

The processed-event record and the corresponding business changes must be saved in the same local database transaction.

## 7. Failure Handling

### Inventory Service is unavailable

- Order remains `PENDING_INVENTORY`.
- The Order Service outbox continues attempting publication.
- RabbitMQ retains messages in durable queues.
- Inventory processing resumes when the service recovers.

### Inventory reservation succeeds but Order Service is unavailable

- Inventory Service has already committed its reservation and result event to its database.
- The Inventory outbox publisher retries publication.
- Order Service eventually receives the result.
- Duplicate delivery is handled through idempotency.

### Inventory reservation fails

- Inventory Service publishes `InventoryReservationRejected`.
- Order Service changes the order to `REJECTED`.
- Notification Service can inform the customer.
- A business rejection is not a technical failure and must not enter an infinite retry loop.

### Inventory release fails during cancellation

- Order remains `CANCELLATION_PENDING`.
- Inventory release processing is retried according to the messaging retry policy.
- The order must not be marked `CANCELLED` until release is confirmed.
- Persistent failures require monitoring and operational reconciliation.

### Late or out-of-order events

- Order Service validates the current order state before applying an event.
- Duplicate and stale events must not cause invalid state transitions.
- Events that cannot be applied safely are logged and handled through the defined recovery process.
- Reconciliation must detect workflows that remain pending beyond an acceptable threshold.

## 8. Saga Recovery and Reconciliation

Saga choreography does not provide a single atomic transaction across services.

StockFlow must therefore support recovery.

A reconciliation process should identify:
- Orders stuck in `PENDING_INVENTORY`.
- Orders stuck in `CANCELLATION_PENDING`.
- Inventory reservations without corresponding completed order outcomes.
- Failed or repeatedly retried event processing.
- Inconsistent order and reservation states.

Reconciliation must use service APIs or events. It must never query another service's database directly.

Automated repair actions must be idempotent and auditable.

## 9. Payment Boundary

Payment processing is outside the MVP scope.

For the MVP:
- `CONFIRMED` means inventory has been successfully reserved.
- It does not mean payment has been completed.
- Payment gateway integration will be considered separately in a future architecture decision.

## 10. Alternatives Considered

### Centralized Saga Orchestration

A dedicated orchestrator controls each step and sends commands to participating services.

Advantages:
- Centralized workflow visibility.
- Explicit control over execution and compensation.
- Easier representation of complex branching workflows.

Disadvantages:
- Adds another component and responsibility.
- Can introduce tighter workflow coupling.
- Requires orchestrator persistence and recovery.

### Distributed Database Transaction

A distributed transaction attempts to make changes across multiple service databases atomic.

This is rejected because it tightly couples independent services and reduces availability and scalability.

### Saga Choreography

Selected for the MVP because the workflow is relatively straightforward and services can react to business events independently.

If the workflow becomes substantially more complex, orchestration can be reconsidered.

## 11. Consequences

### Positive

- Services retain independent database ownership.
- No distributed database transaction is required.
- Order and Inventory can scale independently.
- Temporary failures can be recovered through retries and outbox publishing.
- Business state transitions are explicit.
- The workflow supports future integrations.

### Trade-offs

- The workflow is eventually consistent.
- Debugging requires correlation IDs and distributed tracing.
- Duplicate and out-of-order events require careful handling.
- Reconciliation is necessary for long-running or inconsistent workflows.
- Customers may temporarily see a pending order status.

## 12. Implementation Notes

1. Implement the order state machine in the Order Service.
2. Define shared event contracts in the contracts package.
3. Implement the Order Service outbox publisher.
4. Implement idempotent Inventory Service consumers.
5. Implement atomic multi-item reservation.
6. Implement Inventory Service outbox publishing.
7. Implement idempotent Order Service result consumers.
8. Implement cancellation and inventory release.
9. Add status history and correlation IDs.
10. Implement reconciliation for long-running workflows.
11. Add integration and end-to-end tests for success and failure scenarios.

## 13. Testing Requirements

Test at least:

- Successful order creation and reservation.
- Insufficient stock rejection.
- Multi-item all-or-nothing reservation.
- Concurrent reservations for the final available stock.
- Duplicate `OrderCreated` event.
- Duplicate `InventoryReserved` event.
- Duplicate cancellation request.
- Inventory Service unavailable.
- Order Service unavailable after reservation.
- Inventory release failure.
- Late reservation result after cancellation has started.
- Invalid order state transition.
- Reconciliation of stuck workflows.

## 14. References

- Microservices.io — Saga Pattern: https://microservices.io/patterns/data/saga.html
- Microservices.io — Transactional Outbox: https://microservices.io/patterns/data/transactional-outbox.html
- RabbitMQ Documentation: https://www.rabbitmq.com/docs
