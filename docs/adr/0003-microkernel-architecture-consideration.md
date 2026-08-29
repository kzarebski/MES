# 0003. Microkernel (Plug-in) Architecture Consideration

**Status:** Proposed — decision not yet made

## Context

The current baseline, already stated in `SPEC.md` §2 and `PLAN.md` Guiding Principle 3, is that both Edge and Cloud are **Modular Monoliths** with Hexagonal Architecture inside each vertical slice — strict module isolation, but all modules deployed and versioned together as one application.

A **Microkernel (plug-in) architecture** is a different pattern worth evaluating: a small, stable core (kernel) handles lifecycle/orchestration, while functional capabilities are implemented as plug-ins loaded into it — potentially added, updated, or removed independently of the core. This is being raised as a candidate, most plausibly for the **Edge** application, where:
- OTA updates already need to be zero-downtime (`SPEC.md` §2, `PLAN.md` Phase 12) — shipping a single updated plug-in could be lighter-weight than a full Edge redeploy.
- Machine integration is inherently variable (MQTT today, OPC UA and possibly others later, per `SPEC.md` §2) — a plug-in boundary around protocol adapters could make adding a new machine/protocol type an isolated change.
- Different production lines/machines may eventually need line-specific behavior (custom quality rules, custom data generators) without shipping a monolithic Edge build per site.

This is **not yet decided**, and it directly interacts with the already-stated Modular Monolith principle — resolving this ADR may mean refining that principle (e.g., "modular monolith core + selected plug-in extension points") rather than replacing it outright, or it may conclude the status quo is sufficient and this ADR is rejected.

## Decision Drivers

- OTA update granularity and zero-downtime deployment goals for Edge.
- Extensibility for new machine protocols / line-specific customization without full redeploys.
- Operational complexity: plug-in isolation (classloading, versioning, plug-in-to-core contract stability) is nontrivial in the JVM and could work against the project's stated preference for pragmatic simplicity (`CLAUDE.md`'s "pragmatic exception" tone).
- Consistency with the already-agreed Modular Monolith + Hexagonal baseline — avoid architecture churn without a clear payoff.
- Whether Cloud has any real need for this at all, versus it being an Edge-only concern.

## Options Considered

*(Not yet evaluated in detail — to be discussed and decided separately.)*

### Option A — Status quo: Modular Monolith only, no microkernel/plug-in layer
- Pros: no new complexity; matches every other architectural decision already made.
- Cons: OTA updates and new machine-protocol support still require shipping/redeploying the whole Edge application.

### Option B — Microkernel for Edge only, scoped to specific extension points (e.g., machine protocol adapters, quality rule plug-ins)
- Pros: targets the concrete pain points (OTA granularity, protocol variability) without rearchitecting the whole app; Cloud stays a plain Modular Monolith.
- Cons: two different architectural styles to reason about (Edge vs Cloud); plug-in contract design/versioning becomes its own ongoing concern.

### Option C — Microkernel across both Edge and Cloud
- Pros: architectural consistency across both applications.
- Cons: Cloud has no clearly stated driver for this (no OTA/protocol-variability pressure); likely unnecessary complexity there.

## Decision

**Not yet decided.** To be discussed and resolved explicitly. Until then, continue building strictly per the existing Modular Monolith / Hexagonal Architecture principles in `CLAUDE.md`, `SPEC.md`, and `PLAN.md` — do not introduce ad hoc plug-in/classloading machinery in anticipation of this decision.

## Consequences

Until this ADR is Accepted or Rejected:
- No plug-in loading mechanism, dynamic classloading, or "core + plug-ins" module split should be introduced into Edge or Cloud.
- If later Accepted (in whole or in part, e.g. Option B), it will require revisiting `PLAN.md` Guiding Principle 3 and potentially restructuring the `edge-backend` module layout described there.
- If Rejected, this ADR should be marked accordingly rather than left open indefinitely, so the question doesn't resurface without cause.
