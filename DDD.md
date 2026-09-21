# DDD

Use DDD as the organizing principle for this repository. Keep the domain at the center of the system and let the outer layers adapt to it.

## Core Idea

This repository should not be organized around technical details like controllers, endpoints, or database tables. It should be organized around the problem domain.

The domain contains the business rules, domain types, and ports (interfaces the domain uses). Everything else — API, adapters, infrastructure — exists to serve the domain, not the other way around.

## Layering

The repository uses the following layers:

- Domain Layer (core)
- API Layer
- Adapter Layer
- Infra Layer

### Domain Module

The domain module is the core: it owns business rules, domain types, and ports (interfaces the domain uses).

- Exposes:
  - `domain.model`
  - `domain.service`
  - `domain.ports`
- Never depends on the API layer, adapters, infra, or external runtime concerns.
- Must stay framework-agnostic and easy to test.

Example:

```text
domain/
  model/
  service/
  ports/
```

### API Layer

The API layer owns request handling, validation, authentication, response shaping, and transport concerns.

- Depends on:
  - `domain.service` (the only domain entry point for mutations)
  - `domain.model` (read-only, for shaping responses)
- No business logic in the API layer.
- Controllers and handlers only translate between the wire format and the domain; entity mutation happens only through `domain.service`.

Example:

```text
api/
  routes/
  controllers/
  middleware/
```

### Adapter Layer

The Adapter layer owns integrations, third-party clients, messaging, and external system translation.

- Depends on:
  - `domain.service`
  - `domain.model`
- No direct references to the API layer or other adapters.
- No domain rules in adapters — only protocol and transport translation; business logic stays in the domain.

Example:

```text
adapter/
  github/
  payments/
  messaging/
```

### Infrastructure Module

The infra module owns persistence, configuration, and environment-specific adapters.

- Depends on:
  - `domain.ports`
  - `domain.model`
- No domain rules in infra.
- Never referenced by the domain module (ports flow the other way: infra implements them, domain consumes them).

Example:

```text
infra/
  config/
  repository/
  logger/
```

## Ownership

- Every file or package has exactly one owning layer.
- Every module maps to one layer.
- No shared catch-all layer for domain logic.
- If a type or function contains business rules, it belongs in the domain.
- The composition root (e.g. root `app.ts`) is the only code allowed to import across layers; it wires port implementations into domain services and holds no business logic of its own.

## Ports

Ports are interfaces owned by the domain. They describe the capabilities the domain needs without specifying how they are implemented.

- The domain module defines the port.
- Port implementations live in exactly one layer:
  - `infra/` — persistence ports (databases, filesystem, object storage).
  - `adapter/` — external-system clients (third-party APIs, messaging, webhooks).
- Port implementations are tested separately from domain logic.

Example:

```ts
// domain/ports
export interface Repository {
  save(entity: DomainEntity): Promise<void>;
}
```

## Dependency Rules

- The domain never depends on the API, adapters, or infra.
- The API layer depends on the domain and never on adapters or infra directly.
- Adapters and infra depend on the domain and may implement ports.
- Framework code stays outside the domain.

## Testing

- Domain services and domain models are tested directly, without infrastructure.
- Port implementations are tested against real (or stubbed) external systems, separately from domain tests.
- The API layer is the seam: testing starts at the API boundary and stops at the domain service.
- Seam testing: domain is tested with fakes implementing `domain.ports`; API and adapter/infra layers are tested through their seams, not through the full stack.
- Tests must mirror the production directory structure.

## Practical Guidance

- Start from the domain model, not the endpoints.
- Identify the business capabilities first, then expose them through the API or adapters.
- Keep the domain small, explicit, and independent.
- Prefer moving logic into the domain when a controller, adapter, or infra module starts containing business rules.
- Every public domain type should be testable without network, database, or framework setup.
