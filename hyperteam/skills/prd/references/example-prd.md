> **[Guidance]** Brief description of the feature and the problem it solves. One paragraph max.

# Notification Delivery

## 1. Introduction/Overview

> **[Guidance]** Describe the problem being solved and why it matters to the project. Keep it concise.

The system currently has no way to deliver notifications to users when important events occur. When a background process
completes or an error requires attention, users must manually poll for status. This feature adds a notification delivery
pipeline that routes messages through configurable channels (email, in-app, webhook).

## 2. Goals

> **[Guidance]** Bullet-list what this feature accomplishes. Keep goals measurable.

- Enable event producers to emit structured notifications without knowing the delivery channel.
- Support at least three delivery channels: email, in-app, and webhook.
- Ensure notification payloads are immutable once created (value objects, not mutable state).
- Provide a builder so callers can construct notification parameters without long argument lists.

## 3. Developer Stories

> **[Guidance]** One story = one autonomous agent iteration. Each must fit in a single context window. Use the
> scaffold-first / implement-second pattern for any new or changed API surface: the first story creates typed stubs
> (interfaces, API contracts) with placeholder bodies and skeleton tests; subsequent stories implement the business logic
> against those stable contracts via TDD. Story IDs follow format `FEAT-<slug>-NN` (two-digit zero-padded).

### FEAT-notification-delivery-01: Scaffold NotificationPayload and DeliveryService API

**As a** developer, **I want** typed stubs for `NotificationPayload` and `DeliveryService` **so that** I can validate the
API surface and write tests before implementing real logic.

**Acceptance Criteria:**

- [ ] `NotificationPayload` value object exists in `src/models/` with fields: `event_type`, `recipient_id`, `channel`,
  `body`, `created_at`
- [ ] `DeliveryService` exists in `src/services/` with `deliver()` stub raising `NotImplementedError`
- [ ] `PayloadBuilder` exists and allows chained construction
- [ ] Tests exist in `tests/` covering the expected API shape (tests will fail on `NotImplementedError`)
- [ ] Stub tests created for all new public API surfaces
- [ ] The project's verification command passes (see `CLAUDE.md`)
- [ ] Tests written in a clear, readable style with descriptive names

### FEAT-notification-delivery-02: Implement DeliveryService logic

**As a** developer, **I want** `DeliveryService.deliver()` to route notifications to the correct channel **so that**
event producers can send notifications without knowing delivery details.

**Acceptance Criteria:**

- [ ] `DeliveryService.deliver()` routes to the correct channel handler based on `NotificationPayload.channel`
- [ ] All scaffold-story tests now pass
- [ ] Edge cases covered: unknown channel raises explicit error, empty body rejected
- [ ] The project's verification command passes (see `CLAUDE.md`)
- [ ] TDD cycle followed: test first, implement, refactor

## 4. Functional Requirements

> **[Guidance]** Number every requirement. Each must be concrete, testable, and unmistakably in or out of scope.

- FR-1: `NotificationPayload` must be immutable (value object — no mutation after creation).
- FR-2: `DeliveryService.deliver()` must return a `DeliveryResult` object, not a raw dictionary or primitive.
- FR-3: An empty-body notification must be rejected with a descriptive error.
- FR-4: `PayloadBuilder` must provide a fluent interface (method chaining).

## 5. Non-Goals (Out of Scope)

> **[Guidance]** Be explicit about what this feature does NOT do. Prevents scope creep.

- No retry/queue infrastructure (out of scope for V1 — callers retry at their own layer).
- No admin UI for managing notification templates.
- No rate limiting or deduplication.

## 6. Design Considerations

> **[Guidance]** Architecture notes, relevant ADRs, known constraints.

- The scaffold-first / implement-second pattern is required for any new API surface.
- An ADR should be written if the channel-routing model differs from existing service patterns.
- See ADR index in `docs/adrs/README.md` for existing decisions that may constrain this design.

## 7. Technical Considerations

> **[Guidance]** Environment, language, tooling constraints. Adapt these to your project's stack — the examples below
> are illustrative, not prescriptive.

- Immutable value objects for all payloads
- Parameter encapsulation via Builder pattern
- The project's verification command must pass (see `CLAUDE.md` for the exact command)
- The project's test suite must pass with no skipped notification tests

## 8. Success Metrics

> **[Guidance]** How do we know this feature is done and correct?

- All acceptance criteria checked off.
- The project's verification command exits 0 on all new/modified files.
- The project's test suite exits 0 with no skipped notification tests.

## 9. Open Questions

> **[Guidance]** Unresolved items that need answers before or during implementation. Phase 2 and Phase 3 will resolve
> these.

- OQ-1: Should unknown-channel errors log a warning or raise immediately?
- OQ-2: Is there an existing `User` or `Recipient` model, or should `recipient_id` be a plain string for now?
