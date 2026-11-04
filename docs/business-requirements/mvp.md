# StockFlow — MVP Business Requirements

## 1. Product Overview

StockFlow is a multi-tenant Inventory and Order Management SaaS platform designed for businesses selling products through multiple sales channels.

It provides centralized management of products, SKUs, warehouses, inventory, and customer orders.

The primary goal is to prevent inventory inconsistencies and overselling while providing a reliable order processing workflow.

## 2. Target Users

- Platform Administrator
- Company Owner
- Company Administrator
- Inventory Manager
- Order Manager
- Warehouse Operator

## 3. MVP Scope

### 3.1 Authentication and Tenant Management
- User registration and login.
- Company registration.
- Tenant-aware user memberships.
- Role-based access control.
- Users may belong to multiple companies.

### 3.2 Product Management
- Create, update, and deactivate products.
- Assign unique SKUs within each tenant.
- Store product names, descriptions, and attributes.

### 3.3 Warehouse Management
- Create and manage multiple warehouses.
- Store warehouse details and status.
- Associate warehouses with a tenant.

### 3.4 Inventory Management
- Track stock per product and warehouse.
- Record stock receiving and adjustments.
- Reserve inventory for orders.
- Release reservations when orders are cancelled.
- Maintain inventory movement history.
- Prevent overselling.

### 3.5 Order Management
- Create orders manually or through a generic API.
- Associate every order with an explicit warehouse.
- Support multiple items per order.
- Track order status and status history.
- Support order cancellation.
- Protect order creation with idempotency keys.

### 3.6 Notifications
- Notify users about order confirmation and rejection.
- Track notification delivery status.

## 4. Out of Scope for MVP

- Amazon and Meesho integrations.
- Shopify and other marketplace integrations.
- Payment gateway integration.
- Shipping provider integrations.
- Returns and refunds.
- Advanced analytics and forecasting.
- Automated multi-warehouse allocation.
- Purchase order management.

## 5. Core Business Rules

1. Every business-owned record must be associated with a tenant.
2. Tenant data must remain isolated.
3. A SKU must be unique within a tenant.
4. A tenant can have multiple warehouses.
5. Inventory is tracked per product and warehouse.
6. Inventory cannot be reserved beyond available stock.
7. An order must contain at least one item.
8. Every order must specify a warehouse.
9. Repeated requests with the same idempotency key must not create duplicate orders.
10. Cancelling an order must release its inventory reservation.
11. Inventory changes must be auditable.
12. Order confirmation means inventory has been successfully reserved; it does not mean payment has been completed.
13. Reservation of all order items must be atomic: either all items are reserved or none are reserved.
14. Order items must preserve product and price snapshots at the time of order creation.

## 6. Inventory Calculation

Available inventory is calculated as:

`Available Quantity = On-Hand Quantity - Reserved Quantity`

Example:

- On-hand quantity: 100
- Reserved quantity: 20
- Available quantity: 80

A new order can reserve no more than 80 units.

## 7. Order Lifecycle

The initial order lifecycle is:

`PENDING_INVENTORY → CONFIRMED`

If stock is insufficient:

`PENDING_INVENTORY → REJECTED`

For cancellation after confirmation:

`CONFIRMED → CANCELLATION_PENDING → CANCELLED`

An order becomes cancelled only after its inventory reservation has been successfully released.

## 8. MVP Order Workflow

1. Customer submits an order.
2. API Gateway authenticates the request and establishes tenant context.
3. Order Service validates product and warehouse information.
4. Order Service creates the order with `PENDING_INVENTORY` status.
5. Order Service publishes an `OrderCreated` event through its transactional outbox.
6. Inventory Service consumes the event.
7. Inventory Service checks stock availability.
8. Inventory Service reserves all requested items atomically, or rejects the reservation.
9. Inventory Service publishes the reservation result.
10. Order Service updates the order to `CONFIRMED` or `REJECTED`.
11. Notification Service processes the resulting notification event.

## 9. Sales Channels

The MVP supports:

- `MANUAL`
- `API`

The design should allow additional sales channels in the future.

## 10. Important Technical Requirements

- Multi-tenant data isolation.
- Independent service database ownership.
- REST communication for synchronous operations.
- RabbitMQ for asynchronous events.
- Transactional Outbox pattern.
- Idempotent message consumers.
- Atomic inventory reservation.
- Reliable retries and dead-letter handling.
- Structured logging and error handling.
- Automated unit and integration tests.

## 11. MVP Success Criteria

The MVP is successful when:

- A user can register and create a company.
- Company users can manage products and warehouses.
- Inventory can be received and adjusted.
- Customers can submit manual or API orders.
- Orders can reserve inventory without overselling.
- Insufficient stock causes a clear order rejection.
- Duplicate requests do not create duplicate orders.
- Cancellation releases reserved stock.
- Tenant data remains isolated.
- The complete order-to-inventory workflow is covered by automated tests.

---

**Document Status:** Draft  
**Project:** StockFlow  
**Milestone:** 1 — Architecture and Foundation