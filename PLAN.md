# MES Execution Plan — Vertical Slicing (Feature by Feature)

## Guiding Principles

1. **Vertical Slicing:** Each feature is built as a complete slice (Domain → Application → Infrastructure → Tests) before moving to the next.
2. **Incremental Shared Kernel:** Value objects, DTOs, and events are added to `shared-kernel` only when a slice requires cross-module communication.
3. **Modular Monolith:** Both Edge and Cloud are structured as modular monoliths with strict module isolation — cross-module communication via Facades or Domain Events only. Whether to additionally adopt a Microkernel/plug-in architecture (e.g., for Edge machine-protocol adapters) is an open question — see [ADR-0003](../docs/adr/0003-microkernel-architecture-consideration.md); it may refine this principle later but does not change it yet.
4. **Domain-First:** Within each slice, pure Java domain logic is implemented before any infrastructure code.
5. **Monorepo:** All modules (`shared-kernel`, `edge-backend`, `cloud-backend`, `simulator`, `frontend`) live in a single repository.
6. **Helm Deployments:** Cloud on k8s, Edge on k3s — both managed via Helm charts.
7. **TDD:** For every numbered step below that produces code, the "Tests" item listed for that piece is written and run red *before* the corresponding Domain/Application/Infrastructure code, not after — the numbering reflects deliverables, not coding order. See `CLAUDE.md`.
8. **Shift Left:** Every vertical slice/phase gets an architecture and security evaluation both *before* implementation starts (design-time — does the planned approach fit the principles above, does it introduce a security concern) and *after* the code is written (a check, not a replacement). Phase 15 (E2E & Hardening) is a final, cross-cutting pass — it does not replace per-slice evaluation earlier. See `CLAUDE.md`.

## Repository Structure

```
mes/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle/libs.versions.toml
├── shared-kernel/
├── edge-backend/
│   └── src/main/java/com/mes/edge/
│       ├── production/       ← module (domain/ application/ infrastructure/)
│       ├── quality/          ← module
│       ├── logistics/        ← module
│       └── config/           ← module
├── cloud-backend/
│   └── src/main/java/com/mes/cloud/
│       ├── production/       ← module
│       ├── quality/          ← module
│       ├── logistics/        ← module
│       ├── config/           ← module
│       ├── fleet/            ← module (Edge fleet management)
│       └── user/             ← module
├── simulator/
├── frontend/
└── helm/
    ├── edge/
    └── cloud/
```

Each business module follows the hexagonal layering internally:
```
production/
├── domain/
│   ├── model/
│   ├── port/incoming/
│   ├── port/outgoing/
│   └── service/
├── application/
│   ├── service/
│   └── dto/
└── infrastructure/
    ├── persistence/
    ├── web/
    └── messaging/
```

---

## Phase 0: Project Skeleton

- [ ] **0.1** Create `settings.gradle.kts` with subprojects: `shared-kernel`, `edge-backend`, `cloud-backend`, `simulator`
- [ ] **0.2** Create root `build.gradle.kts` — Java 21 toolchain, common plugins, test dependencies (JUnit 5, AssertJ, Mockito)
- [ ] **0.3** Create `gradle/libs.versions.toml` — version catalog for Spring Boot 4, PostgreSQL, Flyway, Micrometer, Testcontainers, Eclipse Paho, Jackson, Resilience4j
- [ ] **0.4** Create `shared-kernel/build.gradle.kts` — plain `java-library`, no Spring
- [ ] **0.5** Create `edge-backend/build.gradle.kts` — Spring Boot plugin, depends on `shared-kernel`
- [ ] **0.6** Create `cloud-backend/build.gradle.kts` — Spring Boot plugin, depends on `shared-kernel`
- [ ] **0.7** Create `simulator/build.gradle.kts` — Spring Boot, depends on `shared-kernel`
- [ ] **0.8** Create minimal `Application.java` + `application.yml` for Edge (port 8081) and Cloud (port 8080) with `spring.threads.virtual.enabled=true`
- [ ] **0.9** Add `.gitignore` for Gradle/Java/Node
- [ ] **0.10** Add Gradle wrapper (`gradlew`, `gradle/wrapper/`)
- [ ] **0.11** Verify: `./gradlew build` passes, both apps respond on `/actuator/health`

---

## Phase 1: Vertical Slice — Operation Flow (Flow Management)

> *The foundational slice: start/end operations on a production line. Establishes the core Product aggregate, the hexagonal patterns, and the first Flyway migrations.*
>
> **PARTIALLY BLOCKED on [ADR-0002](../docs/adr/0002-error-handling-for-expected-business-outcomes.md):** the error-handling strategy for expected business outcomes (e.g., "product not found", invalid state transition) is not yet decided. Steps 1.1-1.6 (value objects, events, `Product`/`Operation` shape, state machine transitions themselves) can proceed since they don't hinge on it, but **1.7's use-case method signatures should be treated as provisional** until ADR-0002 is `Accepted` — they may need to change shape (e.g., return type instead of a thrown exception) once the decision lands.

### Shared Kernel (incremental)
- [ ] **1.1** Add `ProductId`, `StationId`, `LineId`, `OperationId` value objects (Java records, validated)
- [ ] **1.2** Add `ProductState` enum (`IN_PROGRESS`, `COMPLETED`)
- [ ] **1.3** Add `OperationStartedEvent`, `OperationCompletedEvent` domain events

### Edge — `production` module
- [ ] **1.4** **Domain:** `Product` aggregate root (id, serialNumber, state, list of operations)
- [ ] **1.5** **Domain:** `Operation` value object (stationId, startTime, endTime, parameters)
- [ ] **1.6** **Domain:** `ProductStateMachine` — transition logic with `EnumMap` (for now: `NEW → IN_PROGRESS → COMPLETED`)
- [ ] **1.7** **Domain ports (incoming):** `StartOperationUseCase`, `EndOperationUseCase` (return/error shape pending ADR-0002)
- [ ] **1.8** **Domain ports (outgoing):** `ProductRepository`, `EventPublisher`
- [ ] **1.9** **Domain service:** `ProductionService` (implements use cases, `@Service` allowed)
- [ ] **1.10** **Application:** `OperationAppService` — orchestrates domain calls, `@Transactional`
- [ ] **1.11** **Application:** `StartOperationCommand`, `EndOperationCommand` command DTOs
- [ ] **1.12** **Infrastructure/persistence:** `ProductEntity`, `OperationEntity` (JPA), `ProductMapper`, `ProductRepositoryAdapter`
- [ ] **1.13** **Infrastructure/web:** `OperationController` (`POST /api/operations/start`, `POST /api/operations/end`)
- [ ] **1.14** **Infrastructure:** Flyway `V1__create_product_table.sql`, `V2__create_operation_table.sql`
- [ ] **1.15** **Tests:** Unit tests for `ProductStateMachine`, `ProductionService`
- [ ] **1.16** **Tests:** Integration test with Testcontainers (PostgreSQL) for `ProductRepositoryAdapter`
- [ ] **1.17** **Tests:** `@WebMvcTest` for `OperationController`

### Cloud — `production` module
- [ ] **1.18** **Domain:** `HistoricalProduct`, `HistoricalOperation` models
- [ ] **1.19** **Application:** `ProductionSyncAppService` — receives and persists operation data from Edge
- [ ] **1.20** **Infrastructure/web:** `EdgeSyncController` (`POST /api/edge/sync/operations`)
- [ ] **1.21** **Infrastructure/persistence:** Entities, Flyway migrations, repository adapter
- [ ] **1.22** **Tests:** Unit + integration tests for Cloud production module

### Edge→Cloud Sync
- [ ] **1.23** **Edge infrastructure:** `CloudSyncAdapter` (implements `CloudSyncPort`) — REST client pushing operation events to Cloud
- [ ] **1.24** Add `ProductEventDto` to `shared-kernel/api/` for the Edge→Cloud contract

---

## Phase 2: Vertical Slice — Quality Control

> *The critical safety slice: validate process parameters, halt machines on defect. Introduces MQTT integration on Edge.*

### Shared Kernel (incremental)
- [ ] **2.1** Add `ConfigVersionId` value object
- [ ] **2.2** Add `QualityCheckFailedEvent`, `QualityCheckPassedEvent` domain events
- [ ] **2.3** Add `QualityResultDto` for Edge→Cloud sync

### Edge — `quality` module
- [ ] **2.4** **Domain:** `QualityCheck` value object (result, parameters map, timestamp, configVersionId)
- [ ] **2.5** **Domain:** `QualityThreshold` value object (parameter name, min, max)
- [ ] **2.6** **Domain ports (incoming):** `RecordQualityCheckUseCase`
- [ ] **2.7** **Domain ports (outgoing):** `MachineControlPort` (halt/resume), `QualityCheckRepository`
- [ ] **2.8** **Domain service:** `QualityService` — validates parameters against thresholds, triggers halt on failure
- [ ] **2.9** **Application:** `QualityAppService` — orchestrates: load product (via `production` module Facade), validate, halt if needed, save, sync to Cloud
- [ ] **2.10** **Infrastructure/persistence:** `QualityCheckEntity`, Flyway `V3__create_quality_check_table.sql` (parameters as JSONB)
- [ ] **2.11** **Infrastructure/mqtt:** `MqttMachineAdapter` (implements `MachineControlPort`), `MqttConfig`
- [ ] **2.12** **Infrastructure/web:** `QualityController` (`POST /api/quality/checks`)
- [ ] **2.13** **Cross-module:** Define `ProductFacade` interface in `production` module for `quality` module to query product data
- [ ] **2.14** **Tests:** Unit tests for `QualityService` — parameterized boundary tests, halt triggered on failure
- [ ] **2.15** **Tests:** Integration test with Testcontainers (Mosquitto) for `MqttMachineAdapter`
- [ ] **2.16** **Tests:** Integration test for quality check persistence

### Cloud — `quality` module
- [ ] **2.17** **Domain + Application + Infrastructure:** Historical quality data storage, sync endpoint
- [ ] **2.18** **Tests:** Cloud quality module tests

---

## Phase 3: Vertical Slice — Traceability

> *Recording serial/batch numbers of sub-components. Establishes the component linking model.*

### Shared Kernel (incremental)
- [ ] **3.1** Add `SerialNumber`, `BatchNumber` value objects
- [ ] **3.2** Add `ComponentLinkedEvent` domain event

### Edge — `production` module (extends existing)
- [ ] **3.3** **Domain:** `Component` model (serialNumber/batchNumber, linked to Product)
- [ ] **3.4** **Domain:** Extend `Product` aggregate with `List<Component>` and `linkComponent()` method
- [ ] **3.5** **Domain ports:** `LinkComponentUseCase`, `GetProductTraceabilityUseCase`
- [ ] **3.6** **Application:** `TraceabilityAppService`
- [ ] **3.7** **Infrastructure/persistence:** `ComponentEntity`, Flyway `V4__create_component_table.sql`
- [ ] **3.8** **Infrastructure/web:** `TraceabilityController` (`POST /api/products/{id}/components`, `GET /api/products/{id}/traceability`)
- [ ] **3.9** **Tests:** Unit tests for component linking; integration test for traceability query

### Cloud — `production` module (extends existing)
- [ ] **3.10** Historical traceability storage, query endpoints for UI
- [ ] **3.11** Tests

---

## Phase 4: Vertical Slice — Product Lifecycle (Rework/Repass & Scrap)

> *Extends the state machine with repair and scrap flows. Hard block on scrapped products.*

### Shared Kernel (incremental)
- [ ] **4.1** Extend `ProductState` enum: add `IN_REPAIR`, `SCRAPPED`
- [ ] **4.2** Add `ProductScrappedEvent`, `ProductSentToRepairEvent`, `ProductRepassedEvent`

### Edge — `production` module (extends existing)
- [ ] **4.3** **Domain:** Extend `ProductStateMachine` with full transitions:
  - `IN_PROGRESS → COMPLETED | IN_REPAIR | SCRAPPED`
  - `IN_REPAIR → IN_PROGRESS (repass) | SCRAPPED`
  - `SCRAPPED` = terminal (no outgoing transitions)
  - Signals an invalid move via `IllegalStateTransitionException` — placeholder pending [ADR-0002](../docs/adr/0002-error-handling-for-expected-business-outcomes.md); an invalid transition is an expected business outcome, not necessarily exception-worthy
- [ ] **4.4** **Domain ports:** `ScrapProductUseCase`, `RepairProductUseCase`, `RepassProductUseCase`
- [ ] **4.5** **Domain service:** Extend `ProductionService` with scrap/repair/repass logic
- [ ] **4.6** **Application:** `ProductLifecycleAppService` — orchestration + event publishing
- [ ] **4.7** **Infrastructure/web:** `ProductLifecycleController` (`POST /api/products/{id}/scrap`, `/repair`, `/repass`)
- [ ] **4.8** **Tests:** Exhaustive state machine tests — every valid transition, every invalid transition, terminal state hard block
- [ ] **4.9** **Tests:** Integration tests for lifecycle persistence

### Cloud
- [ ] **4.10** Historical lifecycle event storage, state transition audit trail
- [ ] **4.11** Tests

---

## Phase 5: Vertical Slice — Recipe Management (Configuration Versioning)

> *Immutable configuration versions. Each product hard-linked to its active recipe.*

### Shared Kernel (incremental)
- [ ] **5.1** Add `ConfigVersionDto` for Cloud→Edge recipe distribution

### Edge — `config` module (new)
- [ ] **5.2** **Domain:** `ConfigVersion` model (id, lineId, version number, parameters as Map, createdAt) — immutable, no update/delete
- [ ] **5.3** **Domain ports:** `GetActiveConfigUseCase`, `ReceiveConfigVersionUseCase`
- [ ] **5.4** **Domain ports (outgoing):** `ConfigVersionRepository`
- [ ] **5.5** **Domain service:** `ConfigService` — lookup active version for a line, link to product
- [ ] **5.6** **Application:** `ConfigAppService`
- [ ] **5.7** **Infrastructure/persistence:** `ConfigVersionEntity`, Flyway `V5__create_config_version_table.sql` (append-only)
- [ ] **5.8** **Infrastructure/web:** `ConfigController` (`GET /api/config/active/{lineId}`, `POST /api/config/receive`)
- [ ] **5.9** **Cross-module:** Wire `production` module to call `config` module Facade when starting an operation (to hard-link configVersionId to product)
- [ ] **5.10** **Tests:** Unit + integration tests

### Cloud — `config` module (new)
- [ ] **5.11** **Domain:** `ConfigVersion` + version creation logic (Process Engineer creates new versions)
- [ ] **5.12** **Application:** `RecipeManagementAppService` — create version, distribute to Edge instances
- [ ] **5.13** **Infrastructure/web:** `RecipeController` (CRUD-like but append-only), `RecipeDistributionController`
- [ ] **5.14** **Infrastructure/persistence:** Entities, Flyway migrations
- [ ] **5.15** **Tests:** Unit + integration tests, verify immutability (no update/delete)

---

## Phase 6: Vertical Slice — End-of-Line Logistics & Label Printing

> *Product → Box → Pallet aggregation with n-level hierarchy. ZPL label printing on Edge.*

### Shared Kernel (incremental)
- [ ] **6.1** Add `BoxId`, `PalletId` value objects
- [ ] **6.2** Add `ProductPackedEvent`, `LabelPrintedEvent`

### Edge — `logistics` module (new)
- [ ] **6.3** **Domain:** `Container` (abstract — id, type, children, parent), `Box extends Container`, `Pallet extends Container` — n-level nesting, traceability inheritance
- [ ] **6.4** **Domain:** `Label` value object (ZPL template + data bindings)
- [ ] **6.5** **Domain ports:** `PackProductUseCase`, `PrintLabelUseCase`, `GetContainerTraceabilityUseCase`
- [ ] **6.6** **Domain ports (outgoing):** `BoxRepository`, `PalletRepository`, `LabelPrinterPort`
- [ ] **6.7** **Domain service:** `LogisticsService` — packing logic, capacity validation
- [ ] **6.8** **Application:** `LogisticsAppService`
- [ ] **6.9** **Infrastructure/persistence:** `ContainerEntity` (self-referential `parent_container_id`), Flyway `V6__create_container_tables.sql`
- [ ] **6.10** **Infrastructure/printer:** `ZplPrinterAdapter` (implements `LabelPrinterPort` — TCP socket)
- [ ] **6.11** **Infrastructure/web:** `LogisticsController` (`POST /api/logistics/pack`, `POST /api/logistics/print-label`)
- [ ] **6.12** **Tests:** Unit tests for packing/nesting logic, traceability inheritance
- [ ] **6.13** **Tests:** Integration tests for container persistence

### Cloud — `logistics` module (new)
- [ ] **6.14** Historical logistics data, container query endpoints for UI
- [ ] **6.15** Tests

---

## Phase 7: Cloud — User Management & Authorization

> *Roles: Process Engineer, Production Manager, Logistics, Operator. Role-based access control.*
>
> **BLOCKED on [ADR-0001](../docs/adr/0001-authentication-and-authorization-approach.md):** the authentication/authorization mechanism has not been decided yet. Steps 7.1-7.6 (user/role modeling, independent of the auth mechanism) may proceed, but **7.7 onward must not start** until ADR-0001's Status is `Accepted`. Do not treat "JWT-based" below as a settled decision.

### Cloud — `user` module (new)
- [ ] **7.1** **Domain:** `User` aggregate (id, username, roles), `Role` enum (`PROCESS_ENGINEER`, `PRODUCTION_MANAGER`, `LOGISTICS`, `OPERATOR`)
- [ ] **7.2** **Domain ports:** `CreateUserUseCase`, `AssignRoleUseCase`, `AuthenticateUseCase`
- [ ] **7.3** **Domain service:** `UserService`
- [ ] **7.4** **Application:** `UserAppService`
- [ ] **7.5** **Infrastructure/persistence:** `UserEntity`, Flyway migrations
- [ ] **7.6** **Infrastructure/web:** `UserController` (CRUD)
- [ ] **7.7** **Infrastructure/security:** `SecurityConfig` (mechanism per ADR-0001, tentatively "JWT-based" — unconfirmed), `RoleBasedAccessConfig` (`@PreAuthorize` per role)
- [ ] **7.8** Apply role-based access to all existing Cloud controllers:
  - `PROCESS_ENGINEER`: recipes, quality params, line config
  - `PRODUCTION_MANAGER`: read all, manage lines, dashboards
  - `LOGISTICS`: products (read), containers, labels
  - `OPERATOR`: own station data (read-only)
- [ ] **7.9** **Tests:** Security tests per endpoint, role verification

---

## Phase 8: Cloud — Edge Fleet Management & OTA

### Cloud — `fleet` module (new)
- [ ] **8.1** **Domain:** `EdgeInstance` (id, name, version, status, lastHeartbeat), `ProductionLine` (linked Edge instances)
- [ ] **8.2** **Domain ports:** `RegisterEdgeUseCase`, `CheckUpdateAvailableUseCase`, `DistributeUpdateUseCase`
- [ ] **8.3** **Application:** `EdgeFleetAppService`, `OtaAppService`
- [ ] **8.4** **Infrastructure/web:** `EdgeRegistrationController`, `OtaController`
- [ ] **8.5** **Infrastructure/persistence:** Entities, Flyway migrations

### Edge — registration
- [ ] **8.6** Edge registers with Cloud on startup (`POST /api/fleet/register`), sends periodic heartbeats
- [ ] **8.7** Edge polls `GET /api/fleet/{id}/update-available` for OTA updates
- [ ] **8.8** **Tests:** Integration tests for registration and heartbeat flow

---

## Phase 9: Edge — MQTT Data Ingestion

> *Ingest real-time machine data via MQTT. Translates raw messages into domain commands.*

- [ ] **9.1** **Infrastructure/mqtt:** `MqttDataIngestionListener` — subscribes to `machines/{stationId}/data` and `machines/{stationId}/events`
- [ ] **9.2** Message deserialization → domain command translation (`StartOperationCommand`, `QualityCheckCommand`, etc.)
- [ ] **9.3** `MqttConfig` — connection factory, topic configuration via `application.yml`
- [ ] **9.4** **Tests:** Integration test with Testcontainers (Mosquitto) — publish MQTT message → verify domain command execution

---

## Phase 10: Observability & Distributed Tracing

> **BLOCKED on [ADR-0004](../docs/adr/0004-logging-and-tracing-standard.md):** the tracing standard/bridge (e.g., OpenTelemetry vs. Zipkin/B3) is not yet decided. Steps 10.3-10.6 (healthchecks, Prometheus metrics, Grafana dashboards) don't depend on it and may proceed, but **10.1-10.2 (tracer configuration and propagation) should not be implemented for real** until ADR-0004 is `Accepted` — the propagation format and exporter choice hinge on it, including the non-standard MQTT hop noted in the ADR.

- [ ] **10.1** Configure Micrometer Tracing — `TraceID` propagation via MDC in both Edge and Cloud (tracer/bridge per ADR-0004)
- [ ] **10.2** MDC filter injects `TraceID` into every log line; Edge events carry TraceID to Cloud
- [ ] **10.3** Spring Boot Actuator healthchecks — Edge monitors machine connectivity, Cloud monitors Edge uptime
- [ ] **10.4** Prometheus metrics endpoint (`/actuator/prometheus`) on both apps
- [ ] **10.5** Custom business metrics: throughput per line, defect rate, scrap count, MTBF
- [ ] **10.6** Grafana dashboard JSON provisioning files
- [ ] **10.7** **Tests:** Verify TraceID propagation in integration test (Edge → Cloud), verify metrics endpoint returns expected gauges/counters

---

## Phase 11: Resilience & Edge Autonomy

> *Edge must never fail or block due to Cloud unavailability.*

- [ ] **11.1** Resilience4j circuit breaker on `CloudSyncAdapter` — fallback to local buffering
- [ ] **11.2** Local event buffer: Edge stores unsent events in PostgreSQL when Cloud is unreachable
- [ ] **11.3** Replay mechanism: on Cloud reconnection, Edge replays buffered events (idempotent, UUID-based dedup)
- [ ] **11.4** Graceful shutdown: flush in-progress operations before container restart (OTA)
- [ ] **11.5** **Tests:** Integration test — simulate Cloud downtime, verify local buffering and replay

---

## Phase 12: Helm Charts & Containerization

### Edge (k3s)
- [ ] **12.1** `edge-backend/Dockerfile` — multi-stage (Gradle build → JRE 21 slim Alpine)
- [ ] **12.2** `helm/edge/Chart.yaml`, `values.yaml`
- [ ] **12.3** Helm templates: Deployment, Service, ConfigMap, PostgreSQL (or external), Mosquitto sidecar/dependency
- [ ] **12.4** Health/readiness probes pointing to Actuator endpoints
- [ ] **12.5** Rolling update strategy for zero-downtime OTA

### Cloud (k8s)
- [ ] **12.6** `cloud-backend/Dockerfile` — multi-stage
- [ ] **12.7** `helm/cloud/Chart.yaml`, `values.yaml`
- [ ] **12.8** Helm templates: Deployment, Service, Ingress, ConfigMap, PostgreSQL, Prometheus, Grafana
- [ ] **12.9** Health/readiness probes
- [ ] **12.10** HPA (Horizontal Pod Autoscaler) configuration

### Verify
- [ ] **12.11** `helm template` renders valid manifests for both charts
- [ ] **12.12** Local k3s deployment of Edge stack succeeds
- [ ] **12.13** Local k8s (minikube/kind) deployment of Cloud stack succeeds

---

## Phase 13: Simulator Module

> **Performance is a priority NFR for this module:** the simulator must scale to 1,000–10,000 concurrently running virtual machines on a single instance (see [SPEC.md](SPEC.md) §6). Design `VirtualMachine` state and scheduling to be lightweight per-instance from the start — this constraint should shape 13.1–13.5, not be retrofitted at 13.8.

- [ ] **13.1** `VirtualMachine` — runs on virtual thread, produces data at configurable intervals
- [ ] **13.2** `VirtualLine` — ordered set of virtual machines
- [ ] **13.3** `DataGenerator` — realistic process parameters (temperature, pressure, torque) with configurable mean/stddev
- [ ] **13.4** `DefectInjector` — out-of-spec values at configurable probability
- [ ] **13.5** `SimulatorMqttPublisher` — publishes to Edge-expected MQTT topics
- [ ] **13.6** REST API: `SimulatorController` (`POST /api/simulator/start`, `/stop`, `/configure`)
- [ ] **13.7** **Tests:** Unit tests for data generation; integration test: simulator → MQTT → Edge
- [ ] **13.8** **Performance:** Benchmark test spinning up 1,000–10,000 concurrent `VirtualMachine` instances on one simulator process; measure memory/CPU per virtual machine and scheduling overhead independent of downstream Edge/Cloud load. Tune virtual-thread pinning, GC settings, and per-VM allocations until the target range runs without degradation.

---

## Phase 14: React Frontend

- [ ] **14.1** Initialize Vite + React + TypeScript project in `frontend/`
- [ ] **14.2** API client layer (typed fetch wrapper, JWT handling)
- [ ] **14.3** Auth context, login page, role-based route guards (`ProtectedRoute`)
- [ ] **14.4** **Dashboard page:** real-time line status, throughput chart, defect rate chart (WebSocket/SSE)
- [ ] **14.5** **Production pages:** product list (state filters), product detail, traceability tree view
- [ ] **14.6** **Quality pages:** parameter trend charts, quality check history
- [ ] **14.7** **Logistics pages:** box/pallet management, label preview, print trigger
- [ ] **14.8** **Recipe pages:** version list, version history/diff, create new version
- [ ] **14.9** **Admin pages:** user management, Edge fleet status, OTA update panel
- [ ] **14.10** **Simulator page:** scenario builder, live simulation dashboard
- [ ] **14.11** Dockerfile for frontend (nginx-based)
- [ ] **14.12** Add to Cloud Helm chart as a sidecar or separate Deployment
- [ ] **14.13** **Tests:** Vitest + React Testing Library for components

---

## Phase 15: E2E Testing & Production Hardening

- [ ] **15.1** `docker-compose.full.yml` — entire stack (Edge + Cloud + PostgreSQL x2 + MQTT + Prometheus + Grafana + Frontend)
- [ ] **15.2** E2E: full product lifecycle (create → operations → quality → pack → ship)
- [ ] **15.3** E2E: scrap/rework flow (defect → scrap or repair → repass)
- [ ] **15.4** E2E: full traceability chain (product → components, container → products)
- [ ] **15.5** E2E: Edge→Cloud sync (data flows correctly, idempotent)
- [ ] **15.6** E2E: recipe versioning (new recipe → propagates to Edge → products linked to correct version)
- [ ] **15.7** Load test: simulator running 10K products, verify zero performance degradation
- [ ] **15.8** HikariCP connection pool tuning
- [ ] **15.9** PostgreSQL query optimization (EXPLAIN ANALYZE, index tuning)
- [ ] **15.10** PostgreSQL table partitioning by month on `operation` and `quality_check` tables
- [ ] **15.11** MQTT QoS: 1 for machine halt commands (at-least-once), 0 for telemetry
- [ ] **15.12** Final Helm chart review and values tuning for production

---

## Phase Dependency Graph

```
Phase 0  (Gradle Skeleton)
   │
   ├── Phase 1  (Operation Flow)        ← foundational slice
   │      │
   │      ├── Phase 2  (Quality Control)
   │      │      │
   │      │      └── Phase 9  (MQTT Ingestion)
   │      │
   │      ├── Phase 3  (Traceability)
   │      │
   │      ├── Phase 4  (Product Lifecycle)
   │      │
   │      ├── Phase 5  (Recipe Management)
   │      │
   │      └── Phase 6  (Logistics)
   │
   ├── Phase 7  (User Management)       ← Cloud-only, parallel after Phase 0
   │
   ├── Phase 8  (Fleet Management)      ← after Phase 7
   │
   ├── Phase 10 (Observability)         ← after Phase 1
   │
   ├── Phase 11 (Resilience)            ← after Phase 1
   │
   ├── Phase 12 (Helm Charts)           ← after Phases 1-11
   │
   ├── Phase 13 (Simulator)             ← after Phase 9
   │
   ├── Phase 14 (Frontend)              ← after Phase 7
   │
   └── Phase 15 (E2E & Hardening)       ← after all phases
```

**Parallel tracks after Phase 1:**
- Slices 2-6 can be developed sequentially or selectively (each adds to the system incrementally)
- Phase 7 (Users) is independent and can start after Phase 0
- Phase 10-11 (Observability, Resilience) can start after Phase 1
