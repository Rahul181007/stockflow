# ADR-004: REST for Synchronous Inter-Service Communication

## Status
Accepted

## Date
2026-11-04

## Context

StockFlow consists of independently deployable microservices. Some operations require an immediate response from another service before the current operation can continue.

Examples include:

- Order Service validating product information with Product Service.
- Order Service validating warehouse information with Warehouse Service.
- API Gateway forwarding incoming HTTP requests to the appropriate service.
- Frontend retrieving order, product, warehouse, and inventory information.

These interactions need request-response communication.

RabbitMQ is suitable for asynchronous communication, but it is not the best choice for every interaction. Some operations require an immediate success or failure response.

## Decision

StockFlow will use REST APIs over HTTP for synchronous communication between services.

RabbitMQ will remain the primary mechanism for asynchronous communication and event-driven workflows.

### Responsibilities

**REST will be used for:**

- Immediate data retrieval.
- Resource validation.
- Operations requiring an immediate response.
- API Gateway to backend service communication.

**RabbitMQ will be used for:**

- Order and inventory workflows.
- Event notifications.
- Background processing.
- Communication that should not block the original request.

### Example: Order Creation

When a customer submits an order:

1. API Gateway forwards the request to Order Service.
2. Order Service calls Product Service to validate products and retrieve product information and price snapshots.
3. Order Service calls Warehouse Service to validate the selected warehouse.
4. Order Service saves the order with `PENDING_INVENTORY` status.
5. Order Service stores an `OrderCreated` event in its transactional outbox.
6. Order Service returns HTTP `202 Accepted`.
7. The outbox publisher eventually publishes the event to RabbitMQ.
8. Inventory Service processes the event asynchronously.
9. Order Service receives the inventory result through RabbitMQ and updates the order status.

REST is used for immediate validation, while RabbitMQ handles the asynchronous reservation workflow.

## API Design Principles

Internal service APIs should follow these principles:

- Use resource-oriented REST endpoints.
- Use appropriate HTTP methods and status codes.
- Validate all incoming requests.
- Apply authentication and authorization.
- Propagate tenant context securely.
- Include request correlation identifiers.
- Define request and response DTOs.
- Document APIs using Swagger/OpenAPI.
- Apply API versioning where appropriate.

Example internal endpoints:

| Service | Method | Endpoint | Purpose |
|---|---|---|---|
| Product | GET | `/internal/products/:id` | Retrieve product details |
| Warehouse | GET | `/internal/warehouses/:id` | Retrieve warehouse details |
| Order | POST | `/internal/orders` | Create an order |
| Inventory | GET | `/internal/inventory` | Retrieve inventory information |

These are initial endpoint examples; exact contracts will be defined during service implementation.

## Reliability and Failure Handling

Synchronous communication introduces runtime dependencies between services.

For example, if Product Service is unavailable, Order Service may be unable to validate a product.

To reduce the impact of failures:

- Configure connection and request timeouts.
- Use bounded retries only where appropriate.
- Retry transient failures, not validation errors.
- Use exponential backoff with jitter for retryable failures.
- Avoid unlimited retry loops.
- Apply circuit breakers when justified by observed failure patterns.
- Log failures with correlation IDs.
- Return meaningful errors without exposing internal details.

Order creation must fail safely if required product or warehouse validation cannot be completed.

A timeout must not be interpreted as proof that a remote operation failed. Where a request may have been processed despite a lost response, use idempotency or a safe read-after-timeout strategy.

## Security

Internal REST APIs must not be treated as automatically trusted.

The implementation must:

- Authenticate service-to-service requests.
- Authorize access to internal endpoints.
- Derive tenant context from trusted authentication information.
- Verify that requested resources belong to the expected tenant.
- Avoid trusting arbitrary tenant IDs supplied by clients.
- Use TLS in production environments.
- Never expose credentials or tokens in logs.

## Alternatives Considered

### 1. RabbitMQ for all communication

Advantages:
- Loose coupling.
- Asynchronous processing.
- Better tolerance of temporary service unavailability.

Disadvantages:
- Immediate request-response interactions become more complicated.
- Request-reply messaging requires correlation and timeout handling.
- Simple validation and retrieval operations become unnecessarily complex.

Not selected as the only communication mechanism.

### 2. REST for all communication

Advantages:
- Simple request-response model.
- Familiar HTTP tooling.
- Easier synchronous debugging.

Disadvantages:
- Creates runtime dependencies between services.
- A downstream failure can affect the original request.
- Long-running workflows become tightly coupled.
- Does not naturally support asynchronous event-driven processing.

Not selected as the only communication mechanism.

### 3. gRPC

Advantages:
- Strongly typed service contracts.
- Efficient binary communication.
- Supports streaming.

Disadvantages:
- Adds protocol and tooling complexity.
- Less convenient for browser-facing APIs and general REST tooling.
- Not necessary for the current MVP requirements.

May be evaluated later if performance or communication requirements justify it.

## Consequences

### Positive

- Simple request-response communication.
- Clear separation between synchronous queries and asynchronous workflows.
- Easier debugging and API documentation.
- Existing HTTP tooling can be reused.
- RabbitMQ remains focused on event-driven communication.

### Negative

- Services can become temporarily dependent on one another.
- Network failures and timeouts must be handled carefully.
- Excessive synchronous calls can increase latency.
- Service contracts require versioning and backward compatibility planning.

## Implementation Notes

1. Implement REST APIs within each service using NestJS controllers.
2. Keep business logic in application/domain layers rather than controllers.
3. Use DTOs and validation pipes.
4. Use typed HTTP clients for internal service communication.
5. Configure timeouts and bounded retry policies.
6. Propagate correlation IDs and trusted tenant context.
7. Document internal APIs with Swagger.
8. Add integration tests for successful responses, validation failures, timeouts, and downstream service failures.
9. Keep asynchronous workflows on RabbitMQ rather than introducing synchronous chains unnecessarily.

## References

- [Microsoft: Microservices Architecture - Communication in a microservice architecture](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/interservice-communication)
- [NestJS Documentation: HTTP Module](https://docs.nestjs.com/techniques/http-module)
- [REST Architectural Style](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)