# 0002. Error Handling for Expected (Non-Exceptional) Business Outcomes

**Status:** Proposed — decision not yet made

## Context

Across the domain/application layers (Edge and Cloud alike), many "failure" outcomes are not bugs or infrastructure faults but normal, expected results of doing business — e.g., a product is not found, a state transition is invalid (`PLAN.md` 4.3 currently sketches `IllegalStateTransitionException`), a quality check fails thresholds, a config version doesn't exist, a container is already full. Using Java exceptions (and their stack-trace cost, `try/catch`-driven control flow, and unclear method signatures) for this class of outcome is a common anti-pattern: exceptions should be reserved for truly exceptional, unrecoverable, or programmer-error conditions (infra failures, bugs), not for outcomes a caller is expected to handle as part of normal flow.

This affects the shape of use case ports across essentially every vertical slice (`StartOperationUseCase`, `EndOperationUseCase`, `ScrapProductUseCase`, etc.), so it needs to be settled early — ideally before or alongside Phase 1 (the foundational slice) — rather than discovered piecemeal per module.

## Decision Drivers

- Consistency: one convention for "expected failure" across all Domain ports (incoming), not ad-hoc per module.
- Idiomatic Java: prefer solutions that fit Java 21+ (sealed interfaces, records, pattern matching) over importing a large functional library for one concept.
- Readability at call sites (Application layer orchestration, Infrastructure/web controllers translating outcomes to HTTP status codes).
- Testability: expected-failure paths should be trivial to unit test without asserting on exception types/messages.
- Must not block or complicate the mandatory TDD workflow (write test first, red, then green).

## Options Considered

*(Not yet evaluated in detail — first thought below, others welcome; to be discussed and decided separately.)*

### Option A — `Either<Error, Success>` (e.g., via a library like Vavr, or a small hand-rolled sealed `Either`)
- Pros: explicit, composable, familiar functional pattern for representing "one of two outcomes."
- Cons: if using a library (Vavr), adds an external dependency across Domain code, which otherwise stays close to plain Java; a hand-rolled version needs upkeep in `shared-kernel`.

### Option B — Custom `Result<T>` / `Outcome<T>` sealed interface (pure Java 21+, no library)
- Pros: no external dependency, fits naturally with Java sealed interfaces + records + pattern matching (`switch` on `Success`/`Failure`).
- Cons: yet another abstraction to design and get right once, shared across `shared-kernel`.

### Option C — Per-use-case explicit return types (e.g., `Optional<Product>` for "not found", dedicated result records per use case rather than one generic wrapper)
- Pros: minimal, no generic wrapper type to introduce; each use case's contract is self-describing.
- Cons: less uniform — different modules may converge on different conventions without a shared pattern to point to.

### Option D — Keep unchecked exceptions, but restrict them by policy to truly exceptional cases only
- Not a real alternative to A/B/C so much as a scoping rule that should hold regardless of which of A/B/C is picked: whatever mechanism is chosen for expected outcomes, exceptions remain reserved for bugs/infra failures.

## Decision

**Not yet decided.** To be discussed and resolved explicitly (this ADR exists specifically to hold that discussion) before Domain ports for Phase 1 (`StartOperationUseCase`, `EndOperationUseCase`, etc.) are implemented, since their signatures depend on the outcome.

## Consequences

Until this ADR is Accepted:
- Do not introduce a generic `Either`/`Result` type into `shared-kernel`, and do not standardize on plain exceptions for expected outcomes either — both are pending this decision.
- `PLAN.md` step 4.3's `IllegalStateTransitionException` mention should be read as an unconfirmed placeholder, same caveat as ADR-0001 gave step 7.7.
- New Domain port method signatures written before this ADR is accepted should be treated as provisional and may need to change once the decision lands.
