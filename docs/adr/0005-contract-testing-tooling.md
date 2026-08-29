# 0005. Contract Testing Tooling

**Status:** Proposed — decision not yet made

## Context

MES has more than one process/deployment boundary where one side's REST contract can silently drift from the other's expectations without either side's own unit/integration test suite catching it: Edge→Cloud sync (`PLAN.md` Phase 1's `POST /api/edge/sync/operations` and later phases' equivalents), Cloud's user-management API (Phase 7), and the Frontend↔Cloud API (Phase 14). Ordinary unit tests verify each side in isolation; integration tests (Testcontainers) verify a side against real infrastructure (DB, broker) but not against the *other service's actual expectations*. Contract testing closes that gap by verifying, per-boundary, that a consumer's expectations match what a provider actually delivers — and that has been decided as a required practice for MES's REST boundaries.

What has **not** been decided is the concrete tooling:

- **Pact:** the most widely used consumer-driven contract testing tool — consumer defines expectations as a "pact" (contract file), provider replays it in CI to verify compliance; typically coordinated through a Pact Broker for contract sharing/versioning between Edge, Cloud, and Frontend repos/modules (here, monorepo modules).
- **Spring Cloud Contract:** provider-driven alternative, tightly integrated with Spring Boot; contracts (Groovy/YAML) live with the provider and generate both provider-side tests and consumer-side stubs.
- **Schema-based validation (OpenAPI/JSON Schema diffing):** a lighter-weight approach — validate requests/responses against a shared OpenAPI spec and diff spec versions for breaking changes, without a dedicated consumer-driven contract framework.

## Decision Drivers

- MES is a monorepo (`PLAN.md` Guiding Principle 5) with Edge, Cloud, and Frontend as separate deployable modules but co-located source — this removes some of Pact's cross-repo coordination overhead (a Pact Broker's main value proposition) but the consumer/provider verification model itself is still useful.
- Both Java sides (Edge, Cloud) are Spring Boot 4 — Spring Cloud Contract would integrate natively; Pact's Spring/JUnit integration is also mature.
- The Frontend (React/TypeScript) is also a consumer of the Cloud API — whichever tool is chosen should have a workable TS/JS consumer story too (Pact has first-class JS support; Spring Cloud Contract's consumer-side story is Java/stub-server oriented and weaker for non-JVM consumers).
- Per [[feedback-design-for-testability]] / `CLAUDE.md`'s Design for Testability principle, this needs a workable seam decided before the relevant slice's implementation, not discovered once Phase 15 (E2E) arrives.
- Per [[feedback-no-single-point-of-assumption-failure]], contract verification should run in CI on both sides independently, so a breaking change is caught at the PR that introduces it rather than only during a manual E2E pass.

## Options Considered

*(Not yet evaluated in detail — to be discussed and decided separately.)*

### Option A — Pact (consumer-driven, with a Pact Broker)
- Pros: consumer-driven contracts fit the Edge/Cloud/Frontend triangle well; strong JS/TS consumer support covers the Frontend case; contracts are executable and versioned.
- Cons: extra moving part to run/operate (Pact Broker), even in a monorepo; steeper initial setup than a same-framework alternative.

### Option B — Spring Cloud Contract (provider-driven)
- Pros: native Spring Boot integration on both Edge and Cloud; contracts live with the provider, generating stubs consumers can test against locally.
- Cons: weaker/less idiomatic story for a non-JVM consumer (the React Frontend); provider-driven model fits Edge→Cloud less naturally if Cloud's contract needs to reflect multiple Edge-fleet versions (ties to OTA/fleet versioning, Phase 8).

### Option C — Shared OpenAPI spec + schema validation/diffing
- Pros: lowest operational overhead; a single source-of-truth spec also documents the API; breaking-change detection via spec diffing (e.g., `openapi-diff`) is straightforward to add to CI.
- Cons: not true consumer-driven contract testing — it verifies structural compatibility, not that a specific consumer's actual usage pattern still works; weaker guarantee than A or B.

## Decision

**Not yet decided.** To be discussed and resolved explicitly before the first Edge→Cloud sync contract (`PLAN.md` step 1.24, `ProductEventDto`) is treated as stable, and in any case before Phase 15 E2E work begins.

## Consequences

Until this ADR is Accepted:
- Do not adopt a specific contract-testing framework/dependency in `libs.versions.toml` or `package.json` beyond what's needed for Phases 0-1.
- Treat every new REST boundary (Phase 1 Edge→Cloud sync, Phase 7 user API, Phase 14 Frontend↔Cloud API) as needing a contract test once this ADR is Accepted — retrofit is expected for boundaries built beforehand, so keep DTOs/controllers in `shared-kernel`/`infrastructure/web` reasonably small and well-isolated to make that retrofit cheap.
- `PLAN.md` steps that define a new cross-boundary DTO or controller should note that a contract test is pending this decision, per the Design for Testability principle (`CLAUDE.md`) — this is not deferred to Phase 15.
