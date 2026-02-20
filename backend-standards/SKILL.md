---
name: backend-standards
description: Enforce strict backend development standards including Schema-First design, Clean Architecture, Domain-Driven Design (DDD), and Test-Driven Development (TDD) for Go, TypeScript, and Python projects.
metadata:
  version: "1.0"
---

# Backend Standards

This skill defines the mandatory standard operating procedures for designing and implementing backend services. You MUST adhere to these practices when generating, refactoring, or architecting backend code.

## 1. Schema-First API & Persistence Design

**Rule:** Always define the contract before writing implementation code. The schema is the single source of truth.

- **API Layer**: Define APIs using OpenAPI (Swagger) for REST or GraphQL SDL for GraphQL.
- **Persistence Layer**: Define database schemas or migration standards before writing repository code.
- **Code Generation**: Aggressively leverage code generation tools (e.g., `oapi-codegen` for Go, OpenAPI Generator for TypeScript/Python) to produce server stubs, models, and client SDKs directly from the schema.
- **Benefit**: Ensures parallel development, guarantees type safety across boundaries, and reduces boilerplate.

## 2. Domain-Driven Design (DDD)

**Rule:** Structure the application to reflect the business domain closely.

- **Ubiquitous Language**: Use domain terminology for all class, function, and variable names.
- **Bounded Contexts**: Keep domain models isolated. Do not share database models directly across bounded contexts without deliberate contracts.
- **Entities & Value Objects**: Group behavior and data. Value objects must be immutable. Entities are defined by a unique identity.
- **Aggregate Roots**: External interactions with a domain must go through its Aggregate Root to guarantee consistency.

## 3. Clean / Hexagonal Architecture (Ports and Adapters)

**Rule:** Isolate core business logic from infrastructure, frameworks, and UI.

- **Dependencies point inward**:
  - `Domain` layer (Entities, Value Objects) depends on nothing.
  - `Application` layer (Use Cases, `Ports`/Interfaces) depends only on `Domain`.
  - `Infrastructure` layer (Adapters, Databases, Web Frameworks) depends on `Application`.
- **Dependency Injection**: Always inject dependencies (Repositories, external services) via interfaces into the Application/Use Case layer. Avoid hardcoding infrastructure implementations inside business logic.

## 4. Strict Typing and Boundary Validation

**Rule:** Catch errors at compile-time whenever possible; catch runtime errors at the system boundaries.

- **Compile-Time Checks**: Use strong typing (TypeScript, Go, Python with `mypy`/type hints). Avoid `any`, `interface{}`, or dynamic typing unless absolutely impossible.
- **Boundary Validation**: Validate all incoming data at the earliest point of entry (API controllers/handlers). If data passes the boundary, internal layers should trust it and rely on the strong typing system. Utilize runtime schema validators where appropriate (e.g., `zod` for TS, `pydantic` for Python).

## 5. Test-Driven Development (TDD)

**Rule:** Write failing tests before writing production code.

- Focus primarily on testing the `Application` (Use Case) and `Domain` layers.
- Because of Dependency Injection and Clean Architecture, mock out the `Infrastructure` layer (databases, external APIs) easily using generated mocks or lightweight stubs.

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
