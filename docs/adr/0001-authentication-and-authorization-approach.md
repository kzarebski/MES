# 0001. Authentication & Authorization Approach

**Status:** Proposed — decision not yet made

## Context

`PLAN.md` Phase 7 (Cloud — User Management & Authorization) and Phase 8 (Edge Fleet Management) both need an authentication/authorization mechanism:
- Cloud exposes role-based access for `PROCESS_ENGINEER`, `PRODUCTION_MANAGER`, `LOGISTICS`, and `OPERATOR` roles across its REST API and the React frontend.
- Edge instances register with Cloud and must be authenticated as fleet members, independent of any human user session.
- Edge must remain operational (per the resilience requirements in `SPEC.md` §5 and `PLAN.md` Phase 11) even when disconnected from Cloud — so any auth mechanism that assumes constant connectivity to a central identity provider needs a defined offline/degraded behavior for Edge.

No decision has been made yet on the concrete mechanism. `PLAN.md` step 7.7 currently mentions "JWT-based" only as a placeholder assumption, not a settled decision.

## Decision Drivers

- Edge autonomy: machine-facing auth at Edge cannot hard-depend on live connectivity to Cloud.
- Human user RBAC on Cloud (4 roles, `@PreAuthorize`-style enforcement).
- Machine-to-machine auth for Edge↔Cloud sync and fleet registration/OTA, separate from human user auth.
- Operational complexity of running/maintaining an external Identity Provider (e.g., Keycloak) vs. self-issued tokens in `cloud-backend`.
- Long-term needs: SSO, multi-tenant/multi-site deployments, audit requirements for a manufacturing/quality-regulated environment.

## Options Considered

*(Not yet evaluated in detail — to be filled in once this ADR is picked up.)*

### Option A — Self-issued JWT (Spring Security, no external IdP)
- Pros: simple to stand up, no extra infrastructure component.
- Cons: Cloud owns token issuance/rotation/revocation directly; scaling to SSO/multi-tenant later means migrating later.

### Option B — External OIDC provider (e.g., Keycloak)
- Pros: standard protocol, SSO-ready, centralized user/role management, better fit for audit requirements.
- Cons: another deployed component (Helm chart, HA, backups) on both the Cloud side and potentially Edge if machine identity is also delegated to it.

### Option C — Hybrid: OIDC for human users on Cloud, separate lightweight credential/mTLS scheme for Edge↔Cloud machine auth
- Pros: decouples human RBAC from fleet/machine auth, lets Edge auth degrade gracefully offline.
- Cons: two mechanisms to build and maintain.

## Decision

**Not yet decided.** This ADR exists to track that the decision is open and must be resolved — through an explicit architectural review — before implementation of `PLAN.md` Phase 7 (`SecurityConfig`, step 7.7 onward) or Phase 8 fleet registration auth begins.

## Consequences

Until this ADR is Accepted:
- `PLAN.md` step 7.7's "JWT-based" wording should be read as an unconfirmed placeholder, not a decision.
- No code in `cloud-backend` or `edge-backend` should hard-code a specific auth mechanism (issuer, token format, IdP client) beyond what's needed for Phases 0-6, which do not require auth.
- Phase 7 and Phase 8 auth-related steps are blocked pending this ADR's acceptance (see `PLAN.md`).
