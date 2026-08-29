# Manufacturing Execution System (MES) Specification

## 1. Project Goal
Develop a distributed application for monitoring and managing production lines. The system must operate in real-time, guarantee quality control (stopping defective products at any stage/machine), ensure full traceability of components and process parameters, and guarantee zero performance degradation to production machines.

## 2. System Architecture
The system consists of two main environments communicating via REST API and async events:
* **Edge (Local Instance):** A high-performance server deployed at the machines.
    * **Architecture:** Modular Monolith incorporating Hexagonal Architecture (Ports & Adapters) within each vertical slice.
    * **Responsibilities:** Low-latency machine control (MQTT/OPC UA), local data buffering, and immediate quality evaluations.
    * **Deployment & OTA:** Containerized and deployed using **k3s (Lightweight Kubernetes)**. Deployments and Over-The-Air (OTA) updates are strictly managed via **Helm** charts to ensure zero-downtime rolling updates.
* **Cloud:** The centralized management system.
    * **Architecture:** Modular Monolith (ready to be extracted into microservices if scaling requires it).
    * **Responsibilities:** Aggregates historical data, serves UI, manages users, and distributes recipes/OTA updates.
    * **Open Decision:** The authentication/authorization mechanism (human user RBAC and Edge↔Cloud machine auth) has not been decided yet — pending [ADR-0001](docs/adr/0001-authentication-and-authorization-approach.md).
    * **Deployment:** Deployed on standard **Kubernetes (k8s)** using **Helm** charts.

## 3. Technology Stack
* **Build Tool:** Gradle with Kotlin DSL (multi-module backend project).
* **Backend (Edge & Cloud):** Java 21+ (leveraging Virtual Threads), Spring Boot 4.
* **Database:** PostgreSQL (optimized for infinite retention of historical parameters).
* **Frontend (UI):** React, TypeScript.

## 4. Main Business Features
* **Flow Management:** Registration of start and end times for every operation.
* **Traceability:** Recording serial/batch numbers of sub-components for every finished product.
* **Quality Control:** Validation against quality requirements. Immediate process halt if a product is defective. Logging all process parameters.
* **Product Lifecycle (Rework/Repass & Scrap):**
    * State Machine in the Domain layer (e.g., `IN_PROGRESS`, `IN_REPAIR`, `SCRAPPED`).
    * Support for product repair (Repass) and return to the line, or permanent scrapping (Scrap) with hard blocks on further processing.
* **End-of-Line Logistics & Printers (Batching):**
    * Aggregation of products into boxes and pallets (n-level hierarchy) inheriting full traceability.
    * Integration with industrial network printers on the Edge (e.g., ZPL commands) for automatic or on-demand label printing.
* **Configuration Versioning (Recipe Management):**
    * Changes to line/machine parameters follow the Immutability principle—creating a new version rather than overwriting. Each produced item is hard-linked to the specific version ID of the configuration active during its processing.

## 5. Observability & Distributed Tracing
* **Distributed Tracing:** Every request carries a unique `TraceID` passed between Edge and Cloud logs (e.g., using MDC, Micrometer Tracing). **Open Decision:** the specific tracing standard/bridge to target (e.g., OpenTelemetry vs. Zipkin/B3) has not been decided — pending [ADR-0004](docs/adr/0004-logging-and-tracing-standard.md).
* **Healthchecks:** Edge monitors machine connectivity; Cloud monitors Edge instance uptime (Spring Boot Actuator).
* **Metrics (Prometheus/Grafana):** * Technical: Request rates, communication errors. 
    * Business: Throughput, defect rates, scrap counts, and machine error frequencies (for Preventive Maintenance).

## 6. QA & Testing Strategy
* **Unit Tests:** Primary focus on the Domain layer (Hexagonal architecture).
* **Integration Tests:** Utilizing Testcontainers for database and adapter verification.
* **E2E Tests:** Verifying critical data flow paths.
* **Traffic Simulator:** A separate module with a UI to configure virtual machines and generate realistic test traffic for Edge/Cloud systems.
    * **Performance Requirement:** The simulator must scale to run a large number of concurrent virtual production machines on a single instance — target range 1,000–10,000 — by leveraging Java Virtual Threads and lightweight per-machine state. This is a priority NFR: simulator throughput must not become the bottleneck when load-testing Edge/Cloud.

## 7. AI Agent Guidelines (Specification Driven Development)
* **Modularity:** Maintain strict separation between `cloud-backend`, `edge-backend`, and `simulator` modules.
* **Domain-First Approach:** For the Edge module, always begin by implementing pure Java domain logic before writing infrastructure or Spring-specific code.
* **Step-by-Step Execution:** Always explain planned changes before writing code. Provide complete, runnable code and document core business methods.
