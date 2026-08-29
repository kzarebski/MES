# Architecture Decision Records (ADR)

This directory records significant, cross-cutting architectural decisions for the MES project — the kind that affect multiple modules, are expensive to reverse, or need explicit sign-off before implementation starts.

## When to write one

Write an ADR before implementing anything that:
- Affects both Edge and Cloud, or crosses module boundaries in a way not already settled by `CLAUDE.md` / `SPEC.md`.
- Chooses between fundamentally different technical approaches (e.g., authentication mechanism, messaging pattern, storage strategy).
- Would be costly or disruptive to reverse once built.

Routine implementation choices within an already-agreed slice do not need one.

## Agent Tooling & Guardrails Prerequisites (before any design or implementation work)

Before the open ADRs above are pushed toward a decision, any new architecture-design ADR is started, or implementation work begins (`PLAN.md` Phase 0 onward), set up the agent tooling and constraints meant to support that work:

- [ ] Install the [Product-Manager-Skills](https://github.com/deanpeters/Product-Manager-Skills) Claude Code plugin (`claude /plugin marketplace add deanpeters/Product-Manager-Skills`) — covers business-analyst work (requirements, PRDs, user stories/acceptance criteria) and product-manager work (roadmap sequencing, prioritization) with battle-tested frameworks rather than ad hoc prompting. User action — third-party plugin, not installed by the agent.
- [ ] Build a custom `architect` skill (via `skill-creator`) scoped to architecture-*design* — evaluating structural options, drafting/updating ADRs, checking a direction against Hexagonal/DDD/Modular Monolith principles — distinct from the `Plan` subagent, which handles step-by-step implementation planning, not design.
- [ ] Build a custom `mentor` skill (via `skill-creator`) scoped to strategic technical sanity-checking across the whole project — is the overall direction still sound, is debt accumulating across phases, are earlier ADRs still holding up — distinct from `/code-review`, which operates at the PR/branch level.
- [ ] Define guardrails for AI agents working on MES — the boundaries of what an agent may do autonomously versus what needs explicit human sign-off, for both design work (e.g., committing to an ADR decision) and implementation work (e.g., what an agent may change, run, or deploy unsupervised). **Not yet researched or decided** — flagged here as a pending action point only, per explicit instruction not to research it now.

No off-the-shelf skill exists yet for `architect` or `mentor` (checked 2026-08-29); `skill-creator` is the path for both.

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
| [0004](0004-logging-and-tracing-standard.md) | Logging & Distributed Tracing Standard | Proposed — decision pending |
| [0005](0005-contract-testing-tooling.md) | Contract Testing Tooling | Proposed — decision pending |
