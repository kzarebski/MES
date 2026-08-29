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
- **Test-Driven Development (TDD) — MANDATORY for every task/step:**
  1. Write the test(s) for the behavior first (unit test for Domain logic, integration test for Infrastructure adapters, etc.), before writing any production code.
  2. Run the test suite and confirm the new test(s) FAIL (red) — this proves the test actually exercises the intended behavior.
  3. Write the minimum production code needed to make the test(s) PASS (green).
  4. Refactor if needed, keeping tests green.
  - Do not write production code for a step before its corresponding test exists and has been confirmed red.
- **Architecture Decision Records (ADR):** Significant, cross-cutting, or hard-to-reverse architectural decisions (e.g., authentication/authorization mechanism, messaging patterns, storage strategy) MUST be recorded as an ADR in `docs/adr/` before implementation starts — see `docs/adr/README.md` for the process and template. Do not start implementing a `PLAN.md` step that depends on such a decision while its ADR is still `Status: Proposed`.
- **No Exceptions for Expected Business Outcomes:** Do not use Java exceptions to represent normal, expected business results (e.g., "not found", a failed validation, an invalid state transition) — reserve exceptions for truly exceptional, unrecoverable, or programmer-error conditions (infra failures, bugs). The concrete mechanism for expected outcomes (e.g., an `Either`/`Result` type, or another approach) is not yet decided — see [ADR-0002](docs/adr/0002-error-handling-for-expected-business-outcomes.md). Do not implement Domain ports whose contracts depend on this decision until ADR-0002 is `Accepted`.

## Git Workflow & Pull Requests (Strict Rules)
- **NEVER** commit or push directly to the `main` or `master` branch.
- Before starting any new task, feature, or phase from `PLAN.md`, always create and checkout a new branch from `main`.
- **Strict Branch Naming Convention:** The branch name MUST start with the task type (`feature/`, `chore/`, `fix/`, `refactor/`), followed immediately by the exact phase and step number from `PLAN.md`, and a short description.
- Format: `<type>/<phase>.<step>-<description>`
- Examples based on PLAN.md: `chore/0.1-gradle-skeleton`, `feature/1.2-product-state-machine`, `feature/2.1-operation-flow`.
- When the code for the task is ready and tests pass, commit using Conventional Commits containing the task number (e.g., `feat(1.2): implement product state machine`).
- Push the branch to the remote repository: `git push -u origin <branch-name>`.
- After pushing, use the GitHub CLI to create a Pull Request: `gh pr create --title "<PR Title>" --body "<PR Description>"`. If GitHub CLI is not available, provide the user with the exact GitHub web link to open the PR.
