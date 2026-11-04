# ADR-001: Monorepo Architecture

## Status

Accepted

## Date

2026-11-04

## Context

StockFlow is a multi-tenant Inventory and Order Management SaaS platform built using multiple independent microservices.

The system contains:

- API Gateway
- Authentication Service
- Tenant Service
- Product Service
- Warehouse Service
- Inventory Service
- Order Service
- Notification Service
- React Frontend

Although these applications belong to the same product, each backend service has its own business logic, database ownership, deployment lifecycle, and responsibilities.

We need a repository structure that supports:

1. Independent development of each service.
2. Clear separation of service responsibilities.
3. Reuse of common technical packages.
4. Centralized documentation and infrastructure configuration.
5. Consistent tooling, linting, formatting, and testing.
6. A manageable development workflow for a small team.

We considered two primary repository strategies:

- Monorepo: Keep all applications and shared packages in one Git repository.
- Polyrepo: Maintain a separate Git repository for every application or service.

## Decision

We will use a **monorepo architecture** with pnpm workspaces.

All StockFlow applications, shared technical packages, infrastructure configuration, and documentation will live in one Git repository.

Each backend microservice will remain an independent NestJS application.

The monorepo is a code organization strategy. It does not mean that all services run as one application or share one deployment unit.

## Repository Structure

The planned repository structure is:

```text
stockflow/
│
├── apps/
│   ├── frontend/
│   ├── api-gateway/
│   ├── auth-service/
│   ├── tenant-service/
│   ├── product-service/
│   ├── warehouse-service/
│   ├── inventory-service/
│   ├── order-service/
│   └── notification-service/
│
├── packages/
│   ├── contracts/
│   ├── config/
│   └── logger/
│
├── infrastructure/
│   ├── docker/
│   ├── postgres/
│   ├── rabbitmq/
│   ├── redis/
│   └── monitoring/
│
├── docs/
│   ├── business-requirements/
│   └── architecture/
│       ├── decisions/
│       ├── diagrams/
│       └── workflows/
│
├── .github/
│   └── workflows/
│
├── docker-compose.yml
├── pnpm-workspace.yaml
├── package.json
├── .env.example
└── README.md
```

## Why pnpm Workspaces?

pnpm is selected as the package manager because it provides workspace support, efficient dependency management, and a structured approach to managing multiple applications in one repository.

Each application can define its own dependencies while sharing approved packages through workspace dependencies.

For example:

- Order Service can depend on the shared event contracts package.
- Inventory Service can depend on the same event contracts package.
- Frontend can have its own independent dependencies.
- Each NestJS application can be built and tested independently.

Workspace dependencies must be declared explicitly.

## Service Independence

Although applications share a repository, each backend service must preserve its independence.

The following rules apply:

1. Every microservice is an independent NestJS application.
2. Every business service owns its database.
3. Services communicate through REST or RabbitMQ.
4. No service directly accesses another service's database.
5. Each service owns its database entities, migrations, and business logic.
6. Services can be built, tested, containerized, and deployed independently.
7. Changes to one service should not unnecessarily require changes to unrelated services.

## Shared Packages

We will maintain a small number of shared technical packages.

### packages/contracts

Contains shared integration contracts such as:

- RabbitMQ event envelope definitions.
- Event payload types.
- Event version definitions.
- Common integration message types.

This package must not contain service-specific business logic or database entities.

### packages/config

Contains reusable technical configuration where appropriate.

Examples:

- TypeScript configuration.
- ESLint configuration.
- Prettier configuration.
- Shared environment validation utilities, if needed.

### packages/logger

Contains reusable logging utilities and standardized logging configuration.

Shared packages must remain small and should only be introduced when genuine reuse exists.

## Important Architectural Restrictions

A monorepo must not become a way to create a tightly coupled distributed monolith.

We will avoid:

- Sharing TypeORM entities between services.
- Sharing business domain models between services.
- Importing another service's internal modules.
- Accessing another service's database.
- Creating a large shared package containing unrelated business logic.
- Requiring all services to be deployed together.

Only explicitly approved technical utilities and integration contracts may be shared.

## Alternatives Considered

### 1. Polyrepo Architecture

Each microservice would have its own Git repository.

**Advantages:**

- Strong repository-level isolation.
- Independent access permissions.
- Independent repository configuration.
- Clear separation between teams.

**Disadvantages:**

- More repository management overhead.
- Shared contract changes require coordination across repositories.
- More complex local development.
- CI/CD and dependency updates must be maintained across repositories.

### 2. Monolithic Backend Repository

All backend modules would be implemented in one NestJS application.

**Advantages:**

- Simpler initial development.
- Easier in-process communication.
- Fewer deployment and infrastructure requirements.

**Disadvantages:**

- Weak service isolation.
- Shared application lifecycle.
- Difficult to establish independent database ownership.
- Does not provide the independent service boundaries required by StockFlow.

## Consequences

### Positive Consequences

- One repository provides a centralized view of the complete platform.
- Shared contracts can be updated and reviewed together.
- Local development is easier to coordinate.
- Documentation and architecture decisions remain centralized.
- Consistent tooling and coding standards are easier to maintain.
- Independent application deployment remains possible.

### Negative Consequences

- Repository size will increase as the platform grows.
- CI/CD must eventually identify which applications are affected by changes.
- Shared package changes may affect multiple applications.
- Repository organization and dependency boundaries require discipline.

## Implementation Notes

We will implement the monorepo using:

- pnpm workspaces.
- NestJS for backend services.
- React, TypeScript, and Vite for the frontend.
- Independent Dockerfiles for deployable applications where appropriate.
- GitHub Actions for automated validation and CI/CD.

Initially, we will prioritize a simple workspace configuration.

Advanced monorepo build tools such as Turborepo or Nx will only be considered when build orchestration or repository scale justifies their introduction.

Each service will maintain its own:

- Source code.
- TypeORM entities.
- Database migrations.
- Tests.
- Configuration.
- API documentation.

## References

- pnpm Workspaces: https://pnpm.io/workspaces
- NestJS Documentation: https://docs.nestjs.com/
- Turborepo Documentation: https://turborepo.com/docs
- Nx Documentation: https://nx.dev/