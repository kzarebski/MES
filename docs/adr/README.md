# Architecture Decision Records (ADR)

This directory records significant, cross-cutting architectural decisions for the MES project — the kind that affect multiple modules, are expensive to reverse, or need explicit sign-off before implementation starts.

## When to write one

Write an ADR before implementing anything that:
- Affects both Edge and Cloud, or crosses module boundaries in a way not already settled by `CLAUDE.md` / `SPEC.md`.
- Chooses between fundamentally different technical approaches (e.g., authentication mechanism, messaging pattern, storage strategy).
- Would be costly or disruptive to reverse once built.

Routine implementation choices within an already-agreed slice do not need one.

## Process

1. Copy `0000-template.md` to `NNNN-short-title.md` (next sequential number).
2. Fill in Context, Decision Drivers, and Options Considered. Set **Status: Proposed**.
3. A task or `PLAN.md` step that depends on this decision must not start implementation until the ADR's Status is **Accepted**.
4. Once a decision is made, update Status to **Accepted**, fill in the Decision and Consequences sections, and reference the ADR number from the relevant `PLAN.md` step(s).
5. If a later decision reverses an earlier one, mark the old ADR **Superseded by ADR-NNNN** rather than deleting it.

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [0001](0001-authentication-and-authorization-approach.md) | Authentication & Authorization Approach | Proposed — decision pending |
| [0002](0002-error-handling-for-expected-business-outcomes.md) | Error Handling for Expected (Non-Exceptional) Business Outcomes | Proposed — decision pending |
| [0003](0003-microkernel-architecture-consideration.md) | Microkernel (Plug-in) Architecture Consideration | Proposed — decision pending |
