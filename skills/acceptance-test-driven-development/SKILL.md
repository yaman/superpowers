---
name: acceptance-test-driven-development
description: >
  Use when implementing any user-facing feature or user story.
  For each Acceptance Criterion (AC), write one failing outer acceptance test first,
  then drive it green through sequential frontend and backend TDD inner loops
  (component, unit, and CDC tests) — each loop red/green/refactor — until the
  outer acceptance test passes. Never write implementation code before both
  the outer acceptance test and the relevant inner test are red.
references:
  - scenario-writing-guide.md
  - contract-testing.md
---

# Acceptance Test Driven Development (ATDD)

## Purpose

This skill defines how to implement user-facing features using **Acceptance Test Driven Development (ATDD)** with **ports & adapters (hexagonal) architecture** on both frontend and backend.

The core structure is a **double loop**:

- **Outer loop** — one failing acceptance test per AC. This test is the sole authority on whether the AC is done. It stays red until all inner loops complete.
- **Inner loops** — sequential frontend and backend TDD loops, each following red → green → refactor, that incrementally build up the behavior required to turn the outer acceptance test green.

Goals:

- Start from clearly defined Acceptance Criteria (ACs).
- Express each AC as a single automated outer acceptance test.
- Drive frontend and backend design from that outer test via paired inner loops.
- Keep domain logic pure and decoupled.
- Use consumer-driven contracts for service integration.

> **Sub-files for this skill:**
> - [`scenario-writing-guide.md`](scenario-writing-guide.md) — how to write correct, automatable Given/When/Then scenarios.
> - [`contract-testing.md`](contract-testing.md) — consumer-driven contract workflow between frontend and backend.

---

## Preconditions

Before using this skill:

- A user story or feature request is available.
- You can collaborate with a human to confirm Acceptance Criteria (ACs) or infer them from existing documentation and ask for confirmation.

---

## Concepts

- **Acceptance Criterion (AC)**: A single behavior expressed in domain language (Given / When / Then).

- **Acceptance test**: Automated end-to-end test for one AC using tools such as:
  - Playwright, Cypress, Selenium.
  - Cucumber/Gherkin or similar BDD frameworks.

- **Outer loop**: The acceptance test for an AC. Stays red while inner loops execute. Turning it green is the definition of done for that AC.

- **Inner loop**: A frontend or backend TDD loop — component, unit, or CDC tests — that drives a slice of behavior red → green → refactor.

- **Ports & adapters / hexagonal architecture**:
  - Domain core (pure logic) depends on abstract ports.
  - UI, HTTP, DB, and other infrastructure are adapters implementing those ports.

- **Consumer-driven contract (CDC)**:
  - A contract that defines expectations of an API from the consumer side.
  - Implemented with tools like Pact, Pactflow, or API Blueprint.
  - Acts as the handoff artifact between the frontend inner loop and the backend inner loop.

---

## Architecture Constraints

### Domain

Domain logic must be implemented as pure functions or services that:

- Take domain data and port interfaces as arguments.
- Return domain data.
- Do not access framework APIs or global state directly.
- Do not directly perform I/O (HTTP, DB, filesystem, browser APIs).

### Ports

Define ports for external dependencies:

- Examples: `TodoRepository`, `AuthService`, `Clock`, `UuidGenerator`, `PaymentGateway`.
- Domain logic uses these ports instead of concrete implementations.

### Adapters

- **Frontend adapters**:
  - UI components and hooks (React, Vue, Svelte, etc.).
  - HTTP clients, storage clients, and other browser integrations.
- **Backend adapters**:
  - HTTP controllers/routers.
  - DB/ORM adapters.
  - Messaging clients, cache clients, etc.

Adapters implement ports and translate between external formats (HTTP, JSON, HTML) and domain types.

### Mocking

- Do not use mocking libraries to intercept:
  - Private methods.
  - Global singletons.
  - Framework internals (React internals, ORM internals, etc.).

- For cross-service communication, the only sanctioned "mock" at the service boundary is a **consumer-driven contract**:
  - Consumer defines expected interactions.
  - Provider verifies implementation against that contract.

If a test requires mocking private internals, refactor the design:

- Extract pure domain logic out of components/controllers.
- Introduce ports for external dependencies.
- Test against the domain and ports instead of internals.

---

## Outer Loop: Per Acceptance Criterion

You work **AC by AC**. Each AC has its own outer loop.

### Step 1 - Define the Acceptance Criterion (AC)

Express each AC in domain language, ideally in Given / When / Then form:

- Given [initial state]
- When [user action]
- Then [observable outcome]

Example:

- Given an empty todo list
- When I add "They came to test box"
- Then I see that item in the list

Confirm each AC with the human partner. See [`scenario-writing-guide.md`](scenario-writing-guide.md) for rules and anti-patterns.

### Step 2 - Create Executable Acceptance Test (RED)

For each AC:

1. Implement an automated acceptance test for this AC using suitable tools:
   - Playwright, Cypress, Selenium, or similar.
   - Optionally Cucumber/Gherkin or another DSL for readability.

2. Ensure the test:
   - Follows the AC's Given / When / Then structure as closely as possible.
   - Uses user-level interactions (clicking, typing, navigation).
   - Locates elements by semantics (labels, roles, visible text) rather than brittle selectors when possible.

3. Run the acceptance test and confirm:
   - It fails.
   - It fails due to missing or incorrect behavior, not due to environment or setup errors.

If the test passes immediately, it is not a valid ATDD starting point. Fix the test so that it fails for the right reason.

This failing acceptance test is the **outer loop**. It must remain red until all inner loops for this AC are complete.

### Step 3 - Run Paired Frontend and Backend Loops

For this AC, you will run **paired inner loops**:

- For loop index `n` (starting from 1):
  - Run **Frontend Loop `n`** (component test → domain unit test → CDC consumer test).
  - Then run **Backend Loop `n`** (CDC provider test → integration test → domain unit test).
- Repeat for `n+1` as needed while the outer acceptance test still fails.

Each inner loop follows its own red → green → refactor cycle. The outer acceptance test is the only authority on completion for this AC.

### Step 4 - Check AC Completion

An AC is considered complete when:

- The outer acceptance test passes against the real frontend and real backend implementation in a test environment.
- All unit/component/CDC tests related to this AC are also green.

Only then mark the AC as done.

---

## Frontend Loop (Per AC, Loop n)

The frontend loop is responsible for:

- UI behavior.
- Frontend domain logic.
- Consumer contract(s) for backend APIs required by this AC.

### FE1 - Identify Acceptance Test Failure

- Run the acceptance test.
- Confirm that the failure for this AC is caused by missing or incorrect frontend behavior (UI, frontend domain, or frontend-side contract).

### FE2 - Component / Widget Test (RED)

Create a component/widget test that verifies:

- The required UI elements for this AC exist and are rendered correctly.
- Given specific props/state, the UI behaves as expected from the user's perspective.

Use tools appropriate for your framework:

- React Testing Library, Vue Test Utils, Svelte Testing Library, etc.

Rules:

- Assert on rendered output and user-observable behavior (DOM text, attributes, events).
- Do not:
  - Inspect private component methods or internal state.
  - Depend on framework internals.

Run the component test and confirm it fails.

### FE3 - Component Adapter Implementation (GREEN)

Implement or adjust UI components so that:

- The component test passes.
- Components act as thin adapters:
  - They receive data and callbacks via props or hooks.
  - They render based on those inputs.
- They do not contain business rules or direct network calls.

Run the component test and ensure it passes. The outer acceptance test will likely still fail at this stage.

### FE4 - Domain Unit Test (RED)

Identify the next piece of frontend domain logic required for this AC.

Create a unit test for a frontend domain use-case:

- Example: `addTodo(text, { todoRepository, clock })`.

Characteristics:

- Pure data in, pure data out.
- Dependencies passed as ports/interfaces, e.g. `TodoRepository`, `Clock`.
- Use in-memory or simple fake implementations of ports in tests.

Run the unit test and confirm it fails.

### FE5 - Domain Implementation (GREEN)

Implement minimal domain logic to satisfy the unit test:

- Logic resides in a framework-agnostic domain layer (no direct React/Vue/Svelte imports).
- Domain logic uses ports/interfaces for any external interactions.

Run the unit test until it passes.

### FE6 - Consumer Contract (CDC) for Backend (Optional but Required if Backend Is Involved)

If this AC requires backend communication:

1. Define or extend a consumer contract using Pact, Pactflow, API Blueprint, or a similar tool.
2. Write a consumer CDC test that:
   - Uses your frontend HTTP client adapter implementing the relevant ports.
   - Runs against a mock provider supplied by the CDC tool.

3. Run the CDC test and confirm it fails initially (no backend yet).
4. Update the HTTP adapter to conform to the contract:
   - Map domain calls to HTTP requests.
   - Map HTTP responses back to domain models.

5. Run the CDC test until it passes.

The generated contract artifact is the handoff to Backend Loop `n`. See [`contract-testing.md`](contract-testing.md) for the full workflow.

### FE7 - Acceptance Test Against Mock Backend

Run the outer acceptance test for this AC using:

- Real frontend implementation.
- Mock backend based on the consumer contract (Pact stub or equivalent).

If the acceptance test passes against the mock backend:

- Frontend Loop `n` for this AC is complete.

If it fails:

- Repeat FE2-FE6 within this same loop index `n`, adding the minimal behavior needed to move closer to green.

### Frontend Design Feedback Rule

At any point during FE2-FE5:

- If writing tests appears to require:
  - Mocking private component internals.
  - Accessing global singletons.
  - Coupling tightly to framework internals.

Then:

1. Pause implementation.
2. Refactor to extract domain logic into pure functions/services behind ports.
3. Adjust tests to target the refactored design.
4. Continue the ATDD cycle after refactoring.

---

## Backend Loop (Per AC, Loop n)

The backend loop begins after Frontend Loop `n` has produced or updated the consumer contract for the AC.

This loop is responsible for:

- Backend domain logic.
- Backend adapters (HTTP, DB, etc.).
- Provider-side CDC tests for the AC.

### BE1 - Provider CDC Test (RED)

Implement a provider CDC test based on the consumer contract:

- Load the consumer contract artifact transferred from the frontend loop.
- Verify that backend responses will match the contract.

Run the provider CDC test and confirm it fails initially.

See [`contract-testing.md`](contract-testing.md) for the transfer and verification workflow.

### BE2 - Integration Test (RED)

Create an integration test for the backend endpoint(s) associated with this AC:

- Sends HTTP requests (or equivalent in your transport).
- Validates:
  - Status codes.
  - Response body structure.
  - Key semantics required by this AC.

Run the integration test and confirm it fails.

### BE3 - Routing / Handler Adapter

Implement minimal routing and handler adapters:

- Define routes (URL + method) mapping to controllers/handlers.
- Controllers/handlers:
  - Parse incoming HTTP requests into domain requests.
  - Call domain use-cases via ports.
  - Map domain responses back to HTTP responses.

No domain rules should live in controllers/handlers; they are adapters only.

### BE4 - Domain Unit Test (RED)

Identify the next piece of backend domain logic required for this AC.

Write a unit test for a backend domain use-case:

- Pure domain types as input and output.
- Ports/interfaces for persistence and communication (e.g. `TodoRepository`, `EmailSender`).
- Use in-memory or test doubles implementing ports.

Run the unit test and confirm it fails.

### BE5 - Domain Implementation (GREEN)

Implement minimal backend domain logic to satisfy the unit test:

- Keep domain logic independent from HTTP, DB, and frameworks.
- Use ports for all external dependencies.

Run the unit test until it passes.

### BE6 - Integration & Provider CDC (GREEN)

- Run the integration test:
  - If it fails, adjust routing/handlers or domain logic as needed while keeping domain unit tests green.
- Once integration passes:
  - Run the provider CDC test.
  - Adjust adapter implementations (serialization, field names, status codes, etc.) until the provider CDC test passes.

When both integration and provider CDC tests are green, Backend Loop `n` is complete for this AC.

### BE7 - Acceptance Test Against Real Backend

Run the outer acceptance test for this AC using:

- Real frontend implementation.
- Real backend implementation in a test environment.

If the outer acceptance test still fails:

- Start Frontend Loop `n+1` for this AC.

If it passes:

- This AC moves closer to completion. If this is the final required behavior for the AC, proceed to mark it as done after all other checks are green.

### Backend Design Feedback Rule

At any point during BE2-BE5:

- If writing tests appears to require:
  - Mocking DB drivers or ORM internals.
  - Mocking HTTP clients directly instead of working through ports.
  - Accessing global singletons.

Then:

1. Pause implementation.
2. Refactor:
   - Extract domain logic into core services with ports.
   - Keep controllers/handlers and infrastructure thin.
3. Adjust tests to target ports and domain logic.
4. Continue the ATDD cycle after refactoring.

---

## Loop Sequencing Constraint

For a given AC and loop index `n`:

- Complete **Frontend Loop `n`** (including consumer CDC tests) before starting **Backend Loop `n`**.
- Complete **Backend Loop `n`** (including provider CDC tests) before starting **Frontend Loop `n+1`**.

You may have multiple FE/BE loop pairs (`n = 1, 2, 3, ...`) for a single AC, but each pair must be sequential.

---

## Local Pre-Push Checks

Before pushing a branch or opening a pull/merge request:

1. Run lint/static analysis.
2. Run all unit tests (frontend and backend).
3. Run all component/widget tests.
4. Run all CDC tests (consumer and provider).
5. Run all acceptance tests for ACs affected by your changes (or the entire acceptance suite if no optimized subset is defined).

Only push if all checks pass.

If any check fails, fix the issue and re-run the full set before pushing.

---

## CI/CD Gates

In CI/CD pipelines:

- On each push:
  - Run unit tests.
  - Run component/widget tests.
  - Run linters.

- Before deploying to a test or staging environment:
  - Run all CDC tests for services involved in the release.
  - Block deployment if any CDC test fails.

- Before deploying to production:
  - Run all acceptance tests for ACs included in the release.
  - Block deployment if any acceptance test fails.

---

## Summary

ATDD in this skill is a **double loop** per AC:

```
┌─────────────────────────────────────────────────────┐
│  OUTER LOOP: Acceptance Test (RED → stays red)      │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  INNER LOOP n: Frontend TDD                   │  │
│  │  Component test RED → GREEN → Refactor        │  │
│  │  Domain unit test RED → GREEN → Refactor      │  │
│  │  CDC consumer test RED → GREEN                │  │
│  └───────────────────────────────────────────────┘  │
│            ↓ contract artifact handoff              │
│  ┌───────────────────────────────────────────────┐  │
│  │  INNER LOOP n: Backend TDD                    │  │
│  │  CDC provider test RED → GREEN                │  │
│  │  Integration test RED → GREEN → Refactor      │  │
│  │  Domain unit test RED → GREEN → Refactor      │  │
│  └───────────────────────────────────────────────┘  │
│            ↓ if still red: n+1                      │
│  OUTER LOOP: Acceptance Test → GREEN ✓              │
└─────────────────────────────────────────────────────┘
```

Key rules:

- One outer acceptance test per AC. It is the sole definition of done.
- Never write implementation code before the outer acceptance test and the relevant inner test are both red.
- Frontend loop always precedes backend loop for the same `n`.
- The CDC contract artifact is the only sanctioned handoff between frontend and backend loops.
- All tests must be green locally before pushing; all gates must pass in CI before deploying.
- See [`scenario-writing-guide.md`](scenario-writing-guide.md) for how to write correct ACs.
- See [`contract-testing.md`](contract-testing.md) for the full CDC workflow.
