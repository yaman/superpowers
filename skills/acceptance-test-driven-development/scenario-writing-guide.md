# Scenario Writing Guide

This guide defines how to write correct, well-scoped, and automatable Acceptance Criteria (AC) scenarios for use in the ATDD skill.

Its purpose is to answer one question: **Given a user story, how do I produce scenarios that are clear, testable, and meaningful to both humans and agents?**

---

## The Given / When / Then Structure

Every AC scenario must follow the Given / When / Then format:

- **Given**: The state of the system before the action takes place.
- **When**: The single user action that triggers the behavior.
- **Then**: The observable outcome from the user's perspective.

### Canonical Example

```
Given an empty todo list
When I type "Buy milk" and click Add
Then I see "Buy milk" in the list
```

### Rules for Given

- Describe system state in **domain language**, not technical language.
- Use named, meaningful states that can map to reusable test fixtures.
- Do not describe database rows, API payloads, or internal data structures.

| Bad | Good |
|-----|------|
| `Given user_id=5 exists in the users table` | `Given a registered user "Alice"` |
| `Given the DB has 0 rows in todos` | `Given an empty todo list` |
| `Given the Redux store has { items: [] }` | `Given an empty todo list` |

### Rules for When

- Describe **one user action only**.
- Use active voice: the user is always the actor.
- If you need two actions, split into two scenarios or reconsider the story's AC scope.

| Bad | Good |
|-----|------|
| `When I log in, go to todos, and add an item` | Split into separate ACs |
| `When the item gets added` (passive) | `When I click Add` |
| `When I fill in the form and submit and wait` | `When I submit the form` |

### Rules for Then

- Describe what the **user sees or experiences**, not what happened internally.
- Must be verifiable at the UI/browser level by an acceptance test runner.
- Never assert on database state, API responses, function calls, or internal events.

| Bad | Good |
|-----|------|
| `Then the todo record exists in the DB` | `Then I see "Buy milk" in the list` |
| `Then the API returns 201` | `Then I see a success confirmation message` |
| `Then the addTodo function is called` | `Then I see the new item appear in the list` |
| `Then the state is updated` | `Then I see the updated total on screen` |

---

## Scope: How Many ACs Per Story

- A single user story typically produces **2 to 5 ACs**.
- Each AC covers **one specific behavior under one set of conditions**.
- Together, the ACs for a story should cover:
  - At least one happy path.
  - Key validation and error cases.
  - Relevant edge cases that matter to the user.

### Example

User story: *As a user I want to add todos so I can track my tasks.*

- AC1 (happy path): Add item to an empty list.
- AC2 (happy path variant): Add item to a non-empty list — item appears at the end.
- AC3 (validation): Cannot add an item with empty text — error message is shown.
- AC4 (validation): Cannot add an item over the maximum character limit — error message is shown.

Each AC runs as a separate ATDD outer loop.

### When to Split an AC

Split an AC into two if:

- It contains more than one When clause.
- The Then clause describes two independent outcomes.
- The scenario would require two separate acceptance test flows to cover fully.

### When Not to Split

Do not create micro-ACs that test a single pixel, label, or style detail. Those belong in component tests, not in acceptance criteria.

---

## Named Fixtures

For complex domains, define reusable named states for Given clauses rather than spelling out setup details in every scenario.

Examples:

- `Given an empty todo list`
- `Given a todo list with 3 items`
- `Given a registered user "Alice" with an active subscription`
- `Given an expired session`

Named fixtures:

- Keep scenarios readable and stable.
- Map to reusable test data setup functions shared across acceptance, component, and unit tests.
- Should be defined in a fixtures or test helpers module and reused consistently.

If a Given clause requires more than one sentence of explanation, it is a sign that a named fixture is needed.

---

## Anti-Patterns Reference

### 1. Technical Given

Describing system state using implementation details instead of domain concepts.

```
# Bad
Given user_id=5 exists in the database with role="admin"

# Good
Given a logged-in admin user
```

### 2. Multi-Action When

Packing more than one user action into a single When clause.

```
# Bad
When I fill in the title, set the due date, assign a user, and click Save

# Good
When I submit the task creation form with valid details
```

Or split into multiple ACs if each action has its own observable outcome.

### 3. Internal Then

Asserting on system internals instead of user-observable outcomes.

```
# Bad
Then the POST /todos endpoint is called with body { text: "Buy milk" }

# Good
Then I see "Buy milk" in my todo list
```

### 4. Vague Scenario

Using non-specific language that cannot be directly translated into an automated test.

```
# Bad
Given the system is ready
When I do the thing
Then it works

# Good
Given an empty todo list
When I type "Buy milk" and click Add
Then I see "Buy milk" in the list
```

Always use concrete, specific values in scenarios.

### 5. Testing Implementation

Referring to code-level constructs in the scenario.

```
# Bad
Then the addTodo() function is called once
Then the Redux store contains the new item

# Good
Then I see the new item in the list
```

### 6. Happy Path Only

Writing only one AC per story and skipping validation, error, and edge cases.

Always ask:

- What happens if the user submits an empty or invalid input?
- What happens at the boundary (maximum length, empty state, full state)?
- What happens if a required dependency (network, session) is unavailable?

### 7. Overlapping ACs

Two ACs that always pass or fail together because they test the same behavior under the same conditions. Each AC must be independently satisfiable.

If removing one AC would not reduce your test coverage, merge them.

### 8. Passive Voice in When

The user is always the actor in an AC. Passive constructions hide who is doing what.

```
# Bad
When the item is submitted
When the form gets filled in

# Good
When I submit the item
When I fill in the form and click Save
```

### 9. Non-Observable Then

Describing internal state changes instead of what the user perceives.

```
# Bad
Then the component re-renders
Then the state is updated
Then the event is dispatched

# Good
Then I see the updated count on screen
Then I see a confirmation message
```

### 10. Over-Specified Given

Writing a long Given clause that duplicates another scenario's setup, instead of using a named fixture.

```
# Bad
Given I have registered with email alice@example.com, confirmed my email,
logged in, created a project called "Work", and added 3 tasks

# Good
Given a project "Work" with 3 tasks
```

---

## Agent-Specific Rules

When an agent is deriving or writing scenarios:

- Do not silently change the meaning of an AC while translating it into an executable test.
  - The executable test must faithfully represent the scenario as written.
  - If the scenario is ambiguous, flag it to the human before proceeding.

- Do not add extra Then clauses that were not in the original AC specification.
  - Adding undiscussed assertions changes the definition of done without human approval.

- Do not invent Given conditions that were not part of the story or confirmed by the human.
  - Undiscussed preconditions introduce hidden assumptions.

- If a scenario cannot be automated as written (e.g. it is too vague or technically impossible to observe at the UI level):
  1. Stop.
  2. Identify the specific problem.
  3. Propose a refined scenario to the human.
  4. Wait for confirmation before writing the executable test.

- Use named fixtures from the project's fixture library when they exist.
  - Do not duplicate fixture setup inline if a named fixture already covers it.

---

## Quick Reference Checklist

Before finalising any scenario, verify:

- [ ] Given uses domain language, not technical implementation details.
- [ ] When contains exactly one user action.
- [ ] Then describes only what the user can observe in the UI.
- [ ] The scenario uses concrete, specific values (not placeholders like "something" or "the item").
- [ ] The scenario is independently testable (does not always pass/fail with another AC).
- [ ] Validation and error paths are covered by at least one AC in the story.
- [ ] Named fixtures are used for complex or reusable Given states.
- [ ] No code-level constructs (function names, store shape, API routes) appear in the scenario text.
