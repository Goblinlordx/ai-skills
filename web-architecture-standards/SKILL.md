---
name: web-architecture-standards
description: Enforce strict web architecture standards for React-based applications, including Hexagonal Architecture, Separation of Concerns, and Type Safety.
metadata:
  version: "1.0"
---

# Web Architecture Standards

This skill defines the mandatory standard operating procedures for designing and implementing React-based web applications. You MUST adhere to these practices to ensure maintainability, scalability, and testability.

## 1. Core Philosophy: Strict Separation of Concerns

**Rule:** Decouple business logic, state management, and data fetching from the visual representation.

- **Logic-Pure Views**: View components (pages/sections) should be primarily responsible for layout and composing other components. They should contain minimal business logic.
- **Custom Hooks for Logic**: All complex state logic, side effects, and data transformations MUST be extracted into custom hooks.
- **Pure UI Components**: Reusable UI components (buttons, inputs, etc.) must be pure and driven entirely by props. They should have no knowledge of the application's global state or API structure.
- **Type Safety at Boundaries**: Ensure all data entering the application (API responses, user input) is strictly typed and validated at the boundary.

## 2. Project Structure: Feature-Based Design

**Rule:** Organize code by business features, not by technical role.

- **`src/features/`**: Group components, hooks, types, and logic related to a specific feature (e.g., `auth`, `products`, `cart`) here.
- **`src/components/`**: Only for truly global, domain-agnostic UI components (e.g., a generic `Button` or `Modal`).
- **`src/views/` (or `src/pages/`)**: Use this for high-level page layouts that compose features.
- **Explicit Component/View Separation**:
  - **Components**: Reusable, atomic pieces of UI.
  - **Views/Pages**: Orchestrators that pull in multiple features and components to form a screen.

## 3. Styling & Foundations

**Rule:** Leverage standardized UI kits before building custom styles.

- **Foundational UI**: Use **shadcn/ui** for all standard UI components (Dialogs, Buttons, Selects, etc.). Do not reinvent these unless a custom design is strictly required.
- **Custom Styling**: Use **Tailwind CSS** for layout and custom styling needs. Follow utility-first patterns.
- **Animations**: Use **Framer Motion** for all interactive animations and state transitions to ensure a premium feel.

## 4. Technical Standards: Type Safety & Validation

**Rule:** Ensure end-to-end type safety from the API boundary to the UI.

- **Mandatory TypeScript**: Use strict-mode TypeScript for all application code. Avoid `any` at all costs.
- **Runtime Validation with Typia**: Use **typia** for ultra-fast runtime type validation and JSON serialization.
  - **API Boundary**: Validate all incoming API data using `typia.assert<T>()` or `typia.validate<T>()` at the earliest possible point.
  - **Persistence**: If using local storage or indexedDB, validate retrieved data before use.
- **Zod Fallback**: If `typia` is not suitable for a specific environment (e.g., restricted build pipes), use **Zod** as the secondary standard for schema validation.

## 5. Data Layer: Server & Client State

**Rule:** Categorize and manage state based on its source and lifecycle.

### Server State (Async Data)

- **TanStack Query**: Mandatory for all server-side data fetching, caching, and synchronization.
- **Prefer Fresh Data**: To minimize stale data, strictly prefer fetching the latest data over relying on long-lived cache.
  - **Refetch on Mount**: Ensure queries refetch on component mount or view change (default behavior of TanStack Query should typically be preserved or made more aggressive).
  - **Stale Time**: Keep `staleTime` low (or zero) for critical data to ensure a "fetch-then-cache" or "stale-while-revalidate" behavior that favors the network.
- **Hook-Based Fetching**: Wrap all `useQuery` and `useMutation` calls in custom hooks (e.g., `useUser(id)`) to keep logic out of components.
- **Automatic Invalidation**: Leverage query keys and automatic invalidation to keep the UI in sync with the server.

### API Integration (Schema-First)

- **Schema-Driven**: API integration MUST be based on a formal schema (OpenAPI, GraphQL, gRPC).
- **Code Generation**: Use tools like `rtk-query` code-gen, `openapi-typescript`, or `graphql-codegen` to generate type-safe API clients or hooks directly from the schema.
- **Dynamic Usage**: If code-gen is not possible, ensure all API response types are manually mapped and validated at the boundary using `typia`.
- **API Flexibility**: The web application MUST be easily configurable to point to different API server deployments.
  - **Environment Variables**: Use environment variables (e.g., `VITE_API_URL`) to inject the API base URL.
  - **Dev Proxying**: The development server (Vite) MUST be configured to proxy requests to the target API server. This enables developing against different environments (dev, staging, prod) without CORS issues or manual URL changes in code.
  - **Default to Host**: In production, the default behavior should ideally be to call the origin host, unless the API is hosted on a separate cross-origin domain.

## 6. Architecture: Ports & Adapters (Hexagonal)

**Rule:** Treat API services as external dependencies and inject them at the application level.

- **API Services as Adapters**: Treat the generated or manual API client as an **Egress Adapter** (Driven).
- **Abstract Interfaces (Ports)**: Define interfaces for your data services in the application layer. The UI/Hooks should depend on these interfaces, not the concrete API implementation (where practical).
- **Dependency Injection**: Inject services/clients at the top level of the application (e.g., via React Context or as arguments to custom hooks). This makes it easy to swap implementations or point to different mock servers during testing.
- **Benefits**: Enhances testability, allows for easy mocking, and decouples the UI from the specific network implementation.

### Client State (Sync Data)

- **Local State**: Use standard `useState` and `useReducer` for isolation within a single component.
- **Global State with Zustand**: Use **Zustand** for cross-component or global state (e.g., theme preferences, session info, or complex multi-step forms).
  - **Minimalism**: Only put state in Zustand if it truly needs to be shared or persists across views.

## 7. Logic Isolation: The Hook Pattern

**Rule:** Isolated business logic must be testable without the DOM.

- **Custom Hooks as Controllers**: Use custom hooks to manage feature-specific logic. A component should ideally only call a hook and map its return values to UI elements.
- **Pure Functional Logic**: Extract complex calculations or data transformations into pure functions that are called by the custom hooks.
- **Independence**: Logic hooks should not depend on specific UI components. This allows them to be tested in isolation using `@testing-library/react-hooks` or Vitest.

## 8. Form Handling

**Rule:** Use standardized form libraries to handle validation and submission state.

- **React Hook Form**: Mandatory for all forms.
- **Typed Form Schemas**: Use `typia` or Zod to define form schemas and validate input. This ensures the form data matches the expected API structure exactly.
- **Feedback**: Ensure forms provide immediate validation feedback and clear submission/loading states.

## 9. Development & Testing Strategy

**Rule:** Every layer of the application must be verifiable.

- **UI Development in Storybook**: Build and document all custom/shared components in **Storybook**. Ensure components are tested in isolation against various prop states.
- **Unit Testing (Vitest)**: Focus on testing custom hooks, pure functions, and domain logic.
- **End-to-End & Component Testing (Playwright)**:
  - **E2E**: Verify critical user flows (e.g., login, checkout) across multiple browsers.
  - **Component Testing**: Use Playwright's component testing feature for complex interactive components that require a real browser environment.
- **Build Tooling (Vite)**: Leverage Vite for lightning-fast development and optimized production builds.

---

### Execution Example: Building a Product Dashboard

If a user asks to "Build a product dashboard with a searchable list":

1. **Schema First**: Define the Product type and API response structure.
2. **Logic Isolation**: Create `useProducts(searchQuery)` custom hook.
   - Uses TanStack Query to fetch data.
   - Validates API response with `typia.validate<Product[]>()`.
   - Handles searching/filtering logic.
3. **UI Foundations**:
   - Use `shadcn/ui` for the search Input, Table, and Skeleton loaders.
   - Apply **Tailwind CSS** for the grid layout.
4. **View Composition**:
   - Create `ProductPage.tsx` (the View).
   - It calls `useProducts` and maps the results to `ProductTable` and `SearchBar` components.
   - Adds **Framer Motion** for list item entry animations.
5. **Testing**:
   - Write a Vitest unit test for the `useProducts` hook (mocking the API).
   - Write a Playwright E2E test to verify the search flow in the browser.
