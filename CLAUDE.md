## Hexagonal Architecture Rules (For Edge/Cloud Modules)
1. **Domain Layer (`domain` package):**
   - Core business logic resides here.
   - PRAGMATIC EXCEPTION: You are ALLOWED to use Spring Dependency Injection annotations (e.g., `@Service`, `@Component`) on domain services to avoid manual `@Bean` configuration.
   - STRICTLY PROHIBITED IN DOMAIN: Web/API annotations (e.g., `@RestController`, `@RequestMapping`) and Database/JPA annotations (e.g., `@Entity`, `@Table`, `@Column`).
2. **Application Layer (`application` package):** - Orchestrates domain objects and handles transactions.
3. **Infrastructure Layer (`infrastructure` package):** - Contains all external Adapters (PostgreSQL entities/repositories, REST Controllers, MQTT clients) and external API configurations.

## Architectural & Development Approach
- **Modular Monolith:** Both Edge and Cloud applications must be structured as Modular Monoliths. Strictly separate business domains into distinct top-level packages (e.g., `com.mes.edge.production`, `com.mes.edge.quality`, `com.mes.edge.logistics`).
- **Module Isolation:** Cross-module communication must happen via public interfaces (Facades) or Domain Events. A domain in one module CANNOT directly access the database repository of another module.
- **Vertical Slicing (Feature by Feature):** Develop the system one complete vertical slice at a time (e.g., Domain -> App -> Infra -> Tests for "Operation Flow", then move to "Quality Control").
- **Incremental Shared Kernel:** DO NOT build the entire `shared-kernel` upfront. Add DTOs, Value Objects, and Domain Events to the shared kernel incrementally ONLY when a new vertical slice requires them to communicate between Edge, Cloud, or different modules.
