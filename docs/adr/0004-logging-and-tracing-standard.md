# 0004. Logging & Distributed Tracing Standard

**Status:** Proposed — decision not yet made

## Context

`SPEC.md` §5 and `PLAN.md` Phase 10 already sketch observability: a `TraceID` propagated between Edge and Cloud logs via MDC, "Micrometer Tracing" as the mechanism, and Prometheus/Grafana for metrics. What has **not** been decided is which underlying distributed-tracing *standard* the implementation should be compatible with, and Micrometer Tracing itself is only a facade — it still needs a concrete tracer/bridge and propagation format underneath it. There is more than one live standard/ecosystem to choose between, and picking one affects context-propagation format, exporter choice, and how the Grafana stack (SPEC.md §5) receives trace data:

- **OpenTelemetry (OTel):** the current vendor-neutral standard — W3C Trace Context propagation, OTLP export protocol, and an emerging OTel Log Data Model for correlating structured logs with traces/spans.
- **Zipkin/B3 (via Micrometer's Brave bridge):** an older, well-established combination Micrometer supports directly; uses the B3 propagation header format rather than W3C Trace Context.

This also has an MES-specific wrinkle beyond typical HTTP-only microservice tracing: Edge↔Cloud communication isn't purely REST — Edge also talks to machines over MQTT (`PLAN.md` Phase 9) — and neither W3C Trace Context nor B3 has a built-in MQTT binding, so trace propagation across the MQTT hop needs an explicit (likely custom header/property) design regardless of which standard is chosen upstream.

## Decision Drivers

- Compatibility with the already-planned Prometheus/Grafana observability stack (`SPEC.md` §5, `PLAN.md` Phase 10) — Grafana Tempo and most current backends are OTLP-native.
- Context propagation across Edge→Cloud REST calls, and the non-standard MQTT hop (Edge→machine and machine→Edge).
- Structured log correlation format (log lines carrying traceId/spanId in a way that's queryable alongside traces).
- Vendor neutrality and long-term ecosystem support (OpenTelemetry has broadly become the industry default over Zipkin/B3).
- Spring Boot 4 / Micrometer Tracing's own default integration path, to avoid fighting the framework.

## Options Considered

*(Not yet evaluated in detail — to be discussed and decided separately.)*

### Option A — Micrometer Tracing + OpenTelemetry bridge, OTLP export
- Pros: aligns with the current industry-standard tracing ecosystem; W3C Trace Context propagation; OTLP integrates cleanly with modern Grafana/Tempo-style backends.
- Cons: requires deciding on and operating an OTLP-compatible backend/collector; still need a custom solution for the MQTT propagation gap.

### Option B — Micrometer Tracing + Brave/Zipkin bridge
- Pros: simplest path with Micrometer's original default bridge; mature, low-friction Spring Boot integration.
- Cons: B3 propagation format is legacy relative to W3C Trace Context; potential rework later if the ecosystem (or a future integration partner) expects OTel/OTLP natively.

### Option C — Direct OpenTelemetry SDK (bypass the Micrometer Tracing facade)
- Pros: full native access to OTel semantic conventions for traces, metrics, and logs together.
- Cons: steps outside Spring Boot's Micrometer auto-configuration, more manual wiring.

## Decision

**Not yet decided.** To be discussed and resolved explicitly before Phase 10 tracing configuration (`10.1`, `10.2`) is implemented for real. Until then, `SPEC.md` §5's and `PLAN.md` Phase 10's mentions of "Micrometer Tracing" describe the facade only, not the underlying standard/bridge, which remains open.

## Consequences

Until this ADR is Accepted:
- Do not hard-code a specific tracer bridge (Brave/Zipkin vs OpenTelemetry) or exporter/collector dependency in `libs.versions.toml` or application config beyond what's needed for Phases 0-9, which don't require tracing.
- The MQTT trace-propagation gap (no standard binding) needs its own explicit design once the base standard is chosen — do not invent an ad hoc propagation scheme before that.
- `PLAN.md` step 10.1 ("Configure Micrometer Tracing") should be read as provisional pending this decision.
- The blocking-vs-non-blocking error message convention (`CLAUDE.md`) applies regardless of this decision's outcome — it's a message-content/log-level rule, orthogonal to which tracing standard/bridge is chosen.
