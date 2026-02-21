---
name: application-architecture-standards
description: Enforce strict application architecture standards including Schema-First design, Clean Architecture, DDD, and TDD for core applications (API, CLI, TUI). Not intended for primarily presentation-only frontend web applications.
metadata:
  version: "1.0"
---

# Application Architecture Standards

This skill defines the mandatory standard operating procedures for designing and implementing application core logic and adapters. You MUST adhere to these practices when generating, refactoring, or architecting application code, whether it is an API, CLI, or TUI.

## 1. Schema-First API & Persistence Design

**Rule:** Always define the contract before writing implementation code. The schema is the single source of truth.

- **API Layer**: Define APIs using OpenAPI (Swagger) for REST or GraphQL SDL for GraphQL.
- **Persistence Layer**: Define database schemas (e.g. SQL migrations, Prisma schema, or equivalent) before writing database access code. You are not strictly required to use SQL, but the schema must be explicitly defined.
- **Migrations**: Ensure that proper schema and migration support tooling (e.g. `golang-migrate`, Prisma Migrations, Alembic) is integrated into the project _early_ so that the persistence layer can evolve safely over time.
- **Code Generation**: Aggressively leverage code generation tools to produce type-safe code directly from the schemas.
  - For APIs: Use tools like `oapi-codegen` for Go, or OpenAPI Generator for TypeScript/Python.
  - For Persistence: Use `sqlc` for Go, `Prisma` for TypeScript, and `prisma-client-py` for Python to generate strict types and query functions from your schemas.
- **Benefit**: Ensures parallel development, guarantees type safety across boundaries, and reduces boilerplate.

## 2. Domain-Driven Design (DDD)

**Rule:** Structure the application to reflect the business domain closely.

- **Ubiquitous Language**: Use domain terminology for all class, function, and variable names.
- **Bounded Contexts**: Keep domain models isolated. Do not share database models directly across bounded contexts without deliberate contracts.
- **Entities & Value Objects**: Group behavior and data. Value objects must be immutable. Entities are defined by a unique identity.
- **Aggregate Roots**: External interactions with a domain must go through its Aggregate Root to guarantee consistency.

## 3. Clean / Hexagonal Architecture (Ports and Adapters)

**Rule:** Isolate core business logic from all external concerns (UI, databases, message brokers, external APIs).

- **Dependencies point inward**:
  - `Domain` layer (Entities, Value Objects) depends on nothing.
  - `Application` layer (Use Cases, `Ports`/Interfaces) depends only on `Domain`.
  - `Presentation` / **Ingress Adapters** (commonly known as Driving): The layer that _drives_ the application (e.g., REST/GraphQL APIs, CLI commands, TUIs, Pub/Sub message listeners). This layer translates external input into calls to the `Application` layer.
  - `Infrastructure` / **Egress Adapters** (commonly known as Driven): The layer that is _driven by_ the application (e.g., Databases, Object Storage, Pub/Sub publishers, external 3rd-party APIs). This layer implements the output `Ports` defined by the `Application`.
- **Treat all external systems equally**: A database is just one type of external infrastructure. Message brokers (Pub/Sub) and Object Storage (S3, etc.) must also be treated as infrastructure and tightly hidden behind Ports (interfaces) defined by the Application layer.
- **Prevent Domain Leakage via Adapters**: Infrastructure layers must NEVER leak into the domain. Do not let ORM-specific models, cloud SDK-specific types (e.g., AWS S3 objects), annotations, or query objects (e.g., `IQueryable`) enter the domain logic.
  - The `Application` or `Domain` layer defines a pure interface (Port).
  - The `Infrastructure` layer implements the adapter. The adapter is strictly responsible for mapping generated persistence or vendor types into pure Domain Entities.
- **Dependency Injection (Compile-Time Preferred)**: Always inject dependencies (Repositories, Object Storage clients, external services) via interfaces into the Application/Use Case layer. Avoid hardcoding infrastructure implementations inside business logic.
  - **Crucial Rule**: Strongly prefer compile-time, transpile-time, or strict static-analysis-time DI mechanisms over runtime reflection. Fail builds early instead of encountering runtime resolution faults.
  - **Go**: Use compile-time DI generation tools like `goforj/wire` (favor over the unmaintained `google/wire`).
  - **TypeScript**: Use compile-time DI frameworks like `Clawject`, or fall back to native Constructor/Pure DI. Avoid decorators that rely heavily on runtime `reflect-metadata`.
  - **Python**: Use Constructor/Pure DI (manual wiring at the composition root), or carefully configured statically-analyzable containers (e.g., `Dependency Injector` rigidly checked by `pyright`) avoiding dangerous dynamic runtime overrides.

## 4. Strict Typing and Boundary Validation

**Rule:** Catch errors at compile-time whenever possible; catch runtime errors at the system boundaries.

- **Compile-Time Checks**: Use strong typing (TypeScript, Go, Python with `pyright`/type hints). Avoid `any`, `interface{}`, or dynamic typing unless absolutely impossible.
- **Boundary Validation**: Validate all incoming data at the earliest point of entry (API controllers/handlers). If data passes the boundary, internal layers should trust it and rely on the strong typing system. Utilize runtime schema validators to guarantee this (e.g., `typia` for TS, `pydantic` for Python).

## 5. Test-Driven Development (TDD)

**Rule:** Write failing tests before writing production code.

- Focus primarily on testing the `Application` (Use Case) and `Domain` layers.
- Because of Dependency Injection and Clean Architecture, mock out the `Infrastructure` layer (databases, external APIs) easily using generated mocks or lightweight stubs.

## 6. Structured & Leveled Logging

**Rule:** Implement logging as a structured, leveled service injected as a dependency.

- **Leveled Logging**: Use appropriate log levels to balance observability and noise.
  - `INFO`: Records high-level operational flow (e.g., "User registered", "Payment processed"). It must be concise and NOT verbose.
  - `DEBUG`: Records detailed internal state for troubleshooting. Can be highly verbose.
- **Structured Format**: Always output logs in a structured format (e.g., JSON) to ensure they are easily searchable in log management systems.
- **Logging as a Service**: Manage logging as an injected dependency. Define a logging interface (Port) in the Application layer and implement it in the Infrastructure layer. Avoid using global logging state or direct console/vendor imports inside core business logic.
- **Language Recommendations**:
  - **Go**: Use `zap` (Uber) for high-performance structured logging.
  - **TypeScript**: Use `pino` for high-performance structured logging.
  - **Python**: Use `structlog` for easy structured logging.

---

### Execution Example

If a user asks to "Build a user registration endpoint":

1. Extract the Ubiquitous Language: "User", "Registration".
2. Create/Update the OpenAPI schema for `/register`.
3. Generate the boundary models and handler stubs from the schema.
4. Write a failing test for the `RegisterUserUseCase`.
5. Implement the `RegisterUserUseCase` (Application Layer).
6. Implement the `UserRepository` interface (Port) for the database (Adapter).
7. Inject the repository into the Use Case and connect the generated HTTP handler to the Use Case.
