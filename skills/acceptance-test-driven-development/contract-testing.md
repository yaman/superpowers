# Contract Testing

## Purpose

Contract testing defines and verifies the boundary between a **consumer** and a **provider** without requiring both systems to be fully implemented at the same time.

In this ATDD workflow, contract testing is the only sanctioned form of service-level mocking. It exists to:

- Let the frontend define the API it needs.
- Let the backend implement exactly that API.
- Prevent drift between consumer expectations and provider behavior.
- Replace brittle hand-written mocks and ad-hoc stubs with an executable agreement.

Canonical examples include:

- Pact
- API Blueprint
- OpenAPI-based contract verification tools
- Equivalent consumer-driven contract tools in your stack

This workflow prefers **moving the generated Pact contract file from frontend to backend** instead of introducing a separate Pact Broker installation.

---

## Contract Testing in ATDD

Contract testing sits between the frontend and backend loops.

For a single Acceptance Criterion (AC):

1. The **frontend loop** reaches the point where it needs backend behavior.
2. The frontend defines a **consumer contract** and runs consumer tests against a generated mock provider.
3. The frontend produces a contract artifact, such as a Pact JSON file.
4. That contract file is **moved or copied to the backend project/workspace**.
5. The backend starts with a **provider verification test first**, using that contract file.
6. The backend then implements the minimum behavior required to satisfy the provider verification.
7. The outer acceptance test is then run against the real backend.

This means:

- Frontend drives the contract.
- Backend starts from provider verification, not guesswork.
- No Pact Broker is required.
- Acceptance tests remain the outer authority for behavior.

---

## Preferred Workflow Without Pact Broker

This skill assumes the following practical workflow:

### Frontend side

- Write consumer contract test.
- Run it against a Pact-generated mock provider.
- Produce the contract file, for example:
  - `pacts/frontend-backend.json`

### Transfer step

- Copy or move the contract artifact into the backend repository or test workspace.
- Store it in a stable location, for example:
  - `backend/tests/contracts/frontend-backend.json`

### Backend side

- Write the **provider verification test first** using that contract file.
- Run provider verification before starting backend implementation for that contract slice.
- Then continue with integration test, routing, domain unit tests, and minimal business logic.

This keeps the workflow simple and local:

- No broker setup
- No external service dependency
- Explicit contract handoff between consumer and provider

---

## Consumer-Driven Rule

Contracts in this workflow are **consumer-driven**.

This means:

- The consumer defines the API it needs to fulfill an AC.
- The provider implements and verifies against that need.
- The provider does not invent extra API structure and ask the consumer to adapt later.

When writing a consumer contract:

- Define the smallest valid interaction needed by the current AC.
- Avoid speculative fields or endpoints.
- Extend the contract only when a new AC or a new behavior slice requires it.

This keeps contracts:

- Minimal
- Focused
- Easier to verify
- Less likely to become accidental specifications for unrelated behavior

---

## What a Contract Should Describe

A good contract should describe only the interaction that matters to the current AC.

A contract should include:

- Request shape:
  - Method
  - Path
  - Query parameters if relevant
  - Headers if relevant
  - Request body if relevant

- Response shape:
  - Status code
  - Headers if relevant
  - Response body fields required by the consumer

- Interaction semantics relevant to the AC:
  - Example: creating a todo item returns an ID and the created text
  - Example: fetching a list returns items in the expected structure

A contract should **not** include:

- Internal provider implementation details
- DB schemas
- ORM behavior
- Unused fields "just in case"
- Behavior unrelated to the current AC

---

## Contract Testing Workflow

## Step 1 – Detect the Boundary

During the frontend loop, identify the moment where the current AC requires backend behavior.

Examples:

- Creating a todo item
- Loading a list of items
- Marking an item complete
- Authenticating a user
- Fetching pricing or configuration data

At this point, do not invent a hand-written mock service. Write a contract.

---

## Step 2 – Write the Consumer Contract

Create a consumer contract for the smallest interaction needed by the AC.

The contract should answer:

- What request does the frontend send?
- What response must the backend return for the frontend to continue?
- Which fields are required by the frontend domain model?
- Which fields are optional and therefore not required in this contract?

Keep it minimal. If the frontend only needs:

- `id`
- `text`
- `completed`

then the contract should not include:

- audit metadata
- timestamps
- internal enums
- debug fields
- unused nested objects

---

## Step 3 – Run Consumer CDC Test Against Mock Provider

Use the contract tool to run the frontend HTTP adapter against a mock provider generated from the contract.

This test verifies:

- The frontend sends the expected request.
- The frontend parses the response correctly.
- The frontend adapter conforms to the contract.

This test should exercise:

- The HTTP adapter
- Request mapping
- Response mapping
- Error handling relevant to the AC

This test should not exercise:

- Real backend infrastructure
- Backend business logic
- Database behavior

If the frontend uses ports & adapters correctly:

- The domain calls a port.
- The HTTP adapter implements that port.
- The consumer contract test validates the HTTP adapter at the boundary.

---

## Step 4 – Export and Move the Contract File

Once the consumer CDC test is green:

- Export the generated contract artifact.
- Move or copy it into the backend project in a known location.

Recommended rules:

- Use a predictable directory such as:
  - `tests/contracts/`
  - `spec/contracts/`
  - `contracts/consumer/`

- Keep filenames explicit:
  - `frontend-backend.json`
  - `todo-ui-todo-service.json`

- Commit the contract file if your workflow expects backend verification to run in CI from repository state.
- Otherwise, ensure your pipeline explicitly passes the artifact from frontend stage to backend verification stage.

The key rule is:

- Backend provider verification must run against the exact contract artifact generated by the consumer.

---

## Step 5 – Write Provider Verification Test First

After the contract file reaches the backend, the **first backend contract-testing step** is:

- Write the provider verification test using the moved contract file.

This is important.

Do **not** begin by implementing the endpoint and checking it manually.
Do **not** begin by writing backend business logic in isolation.

Begin by writing the provider verification test first.

That provider verification test should:

- Load the consumer contract file.
- Point verification at the real backend provider entrypoint.
- Fail because the provider does not yet satisfy the contract.

This is the backend RED state for contract testing.

This matches the ATDD rule:

- Frontend writes consumer contract first.
- Backend writes provider verification test first.

---

## Step 6 – Move to Backend Integration and Domain TDD

Once provider verification is red, continue backend development in the normal ATDD order:

1. Provider verification test is red.
2. Write integration test.
3. Integration test is red.
4. Create minimal routing/handler adapter.
5. Write backend domain unit test.
6. Backend domain unit test is red.
7. Implement minimal business logic.
8. Unit test becomes green.
9. Integration test becomes green.
10. Provider verification becomes green.

This matters because provider verification defines **what** must be satisfied, while integration and unit tests drive **how** it is implemented.

---

## What Contract Tests Replace

Contract tests replace:

- Manual mock servers that drift from reality
- Ad-hoc JSON fixtures with no verification
- Assumptions that "frontend and backend will line up later"
- Mocking cross-service behavior directly in unit tests

Contract tests do **not** replace:

- Acceptance tests
- Backend integration tests
- Domain unit tests
- Component tests

Each test type has its own job:

| Test type | Scope | Main purpose |
|---|---|---|
| Acceptance test | Whole user-facing flow | Prove the AC is satisfied |
| Component/widget test | UI adapter | Prove UI rendering/interaction behavior |
| Unit test | Domain logic | Prove core business rules |
| Contract test | Consumer/provider boundary | Prove both sides agree on the API |
| Integration test | Real backend entrypoint + infrastructure slice | Prove endpoint wiring and real interaction behavior |

---

## What Makes a Good Contract

A good contract is:

- **Consumer-relevant**: only includes what the consumer needs
- **Minimal**: no speculative fields or extra endpoints
- **Executable**: used in automated consumer and provider verification
- **Portable**: can be moved from consumer project to provider project as an artifact
- **Stable at the right level**: stable in meaning, not overfitted to formatting trivia
- **Focused on boundary behavior**: request/response semantics, not implementation

Examples of good contract design:

- Match on semantic fields required by the consumer
- Use flexible matching where appropriate for generated values like IDs or timestamps
- Assert important structure, not irrelevant serialization noise
- Define realistic examples representative of actual usage

---

## What Makes a Bad Contract

A bad contract:

- Over-specifies implementation details
- Includes fields the consumer does not actually use
- Becomes a snapshot of the provider's internal representation
- Breaks every time formatting changes but behavior does not
- Is not portable between frontend and backend projects
- Is so vague that incompatible implementations still pass

Common bad patterns:

### Over-contracting

The contract includes every field returned by the backend, even though the consumer only uses three.

### Under-contracting

The contract checks only status code and says nothing meaningful about response structure.

### Contracting Internal Details

The contract encodes internal enum names, ORM-specific shapes, or database IDs the consumer should never know about.

### One Giant Contract for Everything

A single oversized contract file tries to cover every endpoint and every edge case in one place.

### Hidden Broker Dependency

The team assumes broker-based distribution exists, but the local and CI workflow actually depends on manually shared files.

In this workflow, avoid that ambiguity. The contract file must be an explicit artifact.

---

## Matching Strategy

Use matching strategies that preserve meaning without creating brittleness.

Prefer:

- "String matching expected semantic pattern"
- "ID is present and of the expected type"
- "Array contains objects with required fields"
- "Timestamp matches expected format"

Avoid:

- Exact matching on values that are intentionally dynamic
- Exact matching on irrelevant field ordering
- Exact matching on serialization details unless they matter to the consumer

The rule is:

- Be strict about meaning.
- Be flexible about incidental formatting.

---

## Error Contracts

Do not write contracts only for happy paths.

If an AC depends on error behavior, write contracts for that too.

Examples:

- Validation error when text is empty
- Unauthorized response when token is missing
- Not found response for missing resource
- Conflict response for duplicate creation

Error contracts should specify:

- Relevant status code
- Response structure required by the consumer
- Error semantics needed by the UI or domain logic

Do not add every possible error case preemptively. Add only those relevant to the current AC or the next explicit AC.

---

## Versioning and Change Strategy

Change contracts incrementally.

When a new AC requires more fields or behavior:

- Extend the contract by adding only what the new AC requires.
- Regenerate the contract artifact on the consumer side.
- Move the updated contract artifact to the backend side.
- Re-run provider verification.
- Re-run acceptance tests if user-visible behavior changes.

Avoid "future-proofing" contracts by adding speculative fields.

If a contract must change incompatibly:

- Change the consumer expectation explicitly.
- Regenerate and move the contract file.
- Update provider verification.
- Update acceptance tests if behavior changes.
- Treat it as a real behavior change, not a hidden implementation tweak.

---

## Relationship to Ports & Adapters

Contract tests fit naturally into ports & adapters architecture.

Recommended shape:

- Domain layer depends on a port such as `TodoRepository` or `TodoService`.
- Frontend HTTP adapter implements that port by calling backend APIs.
- Consumer contract tests verify that HTTP adapter at the boundary.
- Backend controller/handler adapter exposes the provider boundary.
- Provider verification confirms the controller/handler and response shape satisfy the contract.
- Backend domain logic remains behind ports and is tested independently.

This is important because contract tests should validate the **adapter boundary**, not replace domain tests.

---

## Contract Testing Rules

Follow these rules:

1. Write contracts from the consumer side.
2. Define only the minimal interaction needed for the current AC.
3. Run consumer tests against a generated/mock provider.
4. Export the generated contract artifact.
5. Move or copy that contract artifact to the backend project or backend CI stage.
6. Write provider verification test first on the backend.
7. Run provider verification against the real backend provider.
8. Keep contracts focused on boundary behavior, not provider internals.
9. Do not use contract tests as a substitute for acceptance, integration, or domain tests.
10. Do not write giant shared contracts covering unrelated behaviors.

---

## Warning Signs

Stop and reconsider the contract workflow if:

- The contract file is generated but never used by backend verification.
- Backend implements endpoints before provider verification exists.
- The contract contains many fields the consumer never reads.
- It mirrors the database schema.
- It breaks whenever harmless provider refactors occur.
- It is impossible to tell which AC introduced which interaction.
- It is being used to avoid writing proper integration tests.
- It is being used to avoid writing acceptance tests.

These indicate that the contract is either too broad, too brittle, or not being used at the correct architectural boundary.

---

## Practical Guidance for Agents

When writing contract tests as an agent:

- Start from the current AC, not from the entire API surface.
- Inspect the frontend domain use-case and determine the exact provider interaction required.
- Encode only that interaction.
- Run consumer verification and generate the contract artifact.
- Ensure the contract artifact is placed where backend verification can consume it.
- Write provider verification test first on the backend.
- When provider verification fails, ask:
  - Is the contract wrong?
  - Is the provider wrong?
  - Is the adapter mapping wrong?
- Fix the smallest incorrect layer and rerun all relevant tests.

Do not silently widen or loosen a contract just to make tests pass. That hides real incompatibilities.

---

## Final Rule

A service boundary is not complete just because one side "seems to work."

It is complete when:

- The consumer contract is executable,
- The consumer passes against the mock provider,
- The contract artifact is transferred to the backend,
- The provider verification test runs against that exact artifact,
- The provider passes verification,
- And the outer acceptance test still passes against the real system.
