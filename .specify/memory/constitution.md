# Speckit AI Constitution

## Core Principles

### I. Microservice-First
Every feature is built as part of a focused, independently deployable microservice. Each service owns a single bounded context, must be self-contained, and independently testable. A clear domain purpose is required — no catch-all or infrastructure-only services.

### II. Test-First (NON-NEGOTIABLE)
TDD mandatory: Tests written → User approved → Tests fail → Then implement. Red-Green-Refactor cycle strictly enforced.

### III. Simplicity
Start simple. Apply YAGNI principles. Complexity must be justified and documented.

### IV. Observability
Structured logging required on all services. Text I/O ensures debuggability. Include `traceId` in all log lines.

### V. Versioning & Breaking Changes
Use `MAJOR.MINOR.PATCH` semantic versioning. Breaking changes require migration plan and documentation update.

---

## Spring Boot Microservice Standards

### VI. Service Boundaries
Each microservice owns exactly one bounded context. Cross-service communication must go through well-defined API contracts or events — never direct repository access or shared databases between services.

### VII. Configuration
All environment-specific values must be externalized using `${ENV_VAR:default_value}` syntax in `application.yml` / `application-{profile}.yml`. The `:default_value` provides a safe local-dev fallback; production injects real values via environment variables. Secrets (`DB_PASSWORD`, `KAFKA_PASSWORD`, JWT keys) must use `${ENV_VAR}` with **no default** to force explicit injection. Use `@ConfigurationProperties` over `@Value` for multi-property config groups. No hardcoded URLs, credentials, or infrastructure details in source code.

### VIII. Dependency Injection
Constructor injection is mandatory. Field injection (`@Autowired` on fields) is forbidden. All injected dependencies must be `final`. Circular dependencies are a design smell — resolve at the architecture level.

### IX. REST API Design
- Endpoints follow `noun-based`, plural, lowercase-kebab-case paths (e.g., `/orders`, `/order-items`)
- HTTP verbs map strictly: `GET` = read, `POST` = create, `PUT`/`PATCH` = update, `DELETE` = remove
- All responses use consistent envelope DTOs; never expose JPA entities directly from controllers
- Validation on all inbound DTOs using Bean Validation (`@Valid`, `@NotNull`, `@Size`, etc.)
- Global exception handling via `@RestControllerAdvice` — no try-catch inside controllers

### X. Persistence
- Repository interfaces extend `JpaRepository` or `CrudRepository` — no raw `EntityManager` unless absolutely required
- No business logic in repositories — queries only
- Named queries or `@Query` annotations are preferred over query method name chains longer than two predicates
- Transactions (`@Transactional`) belong on the service layer, never on the controller or repository layer

---

## Kafka Event Architecture Standards

> **Full specification**: [kafka-event-standards.md](./kafka-event-standards.md)

### Key Rules (non-negotiables)
- Events conform to **CloudEvents 1.0** (`io.cloudevents` Java SDK) — no custom envelope types
- Topic naming: `{domain}.{entity}.{event}` in `kebab-case`; DLQ = `{topic}.dlq`; all topic names use `${KAFKA_TOPIC_XXX:domain.entity.event}` in `application.yml`
- Producers publish via `@TransactionalEventListener(phase = AFTER_COMMIT)` — never inside the transaction
- Consumers use `@KafkaListener`; group ID externalized; manual `Acknowledgment.acknowledge()`; `enable.auto.commit=false`
- Consumers **must** be idempotent; errors route to DLQ via `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` with exponential backoff
- Integration tests use `@EmbeddedKafka`; every listener test verifies idempotency on duplicate delivery

---

## Java Coding Standards

All Java development in this project must follow the standards defined in the files below. Each file is concise and LLM-readable. When in doubt, refer to the specific standard file.

| Standard | File | Covers |
|----------|------|--------|
| Naming Conventions | [`java-naming.md`](./java-naming.md) | Classes, methods, variables, packages, Spring Boot layer suffixes |
| Code Formatting & Style | [`java-formatting.md`](./java-formatting.md) | Indentation, imports, class structure, braces |
| Exception Handling | [`java-exceptions.md`](./java-exceptions.md) | Try-catch scope, custom exceptions, anti-patterns |
| Constants & Enums | [`java-constants-enums.md`](./java-constants-enums.md) | No magic values, constants package, enum usage |
| Reusability & Structure | [`java-code-structure.md`](./java-code-structure.md) | Canonical package layout, Liquibase migration structure, Util/Mapper/constants |
| Testing Standards | [`java-testing.md`](./java-testing.md) | JUnit 5, 80% coverage minimum, all edge cases |
| Code Quality & Sonar | [`java-code-quality.md`](./java-code-quality.md) | SonarQube gate, SonarLint VS Code workflow |
| Logging Standards | [`java-logging.md`](./java-logging.md) | SLF4J, log levels, MDC traceId propagation, structured logging |
| Build Tools | [`java-build-tools.md`](./java-build-tools.md) | Maven, Gradle, versioning, required plugins, OpenAPI/Swagger |
| Documentation | [`java-documentation.md`](./java-documentation.md) | Javadoc templates, LLM-aware inline comments, package-info, event docs |
| Kafka & Events | [`kafka-event-standards.md`](./kafka-event-standards.md) | CloudEvents 1.0, topics, producer, consumer, DLQ, testing |
| JSON Serialization | [`json-serialization.md`](./json-serialization.md) | snake_case, ObjectMapper config, DTO rules |
| Security | [`java-security.md`](./java-security.md) | Entity exposure, validation, PII, passwords, CORS, Actuator, SQL injection |

### Quick Reference — Non-Negotiables
- Code must pass SonarQube Quality Gate before PR — check SonarLint in VS Code after every file edit
- 80% JUnit coverage minimum required — all edge cases (null, empty, boundary, exception paths) must be tested; coverage padding with trivial tests is prohibited
- No duplicate code — extract to Util, Mapper, or Helper class
- Mappers go in `mapper/` — use MapStruct with `@Mapper(componentModel = "spring")` so they are registered as Spring beans and injected via constructor. Pure stateless helpers (no dependencies) go in `util/` as final classes with private constructors, not as Spring beans
- No magic numbers or magic strings — use constants or enums (log message text is exempt)
- No granular try-catch per line — one block per logical operation
- Always check for existing enums before creating new ones
- No wildcard imports
- No `System.out.println` — use SLF4J logger
- Javadoc must be updated whenever a method is edited
- All JSON (REST + Kafka payloads) must use `snake_case` — configured globally via `spring.jackson.property-naming-strategy: SNAKE_CASE`; Java fields stay `camelCase`
- **NEVER** return a JPA `@Entity` from a controller — always map to a response DTO via `mapper/`
- **NEVER** log PII (email, phone, name, address) or credentials (passwords, tokens, API keys) — log only opaque IDs
- **NEVER** include password, token, or secret fields in any response DTO
- `@Valid` is mandatory on every controller request parameter and cascaded on nested DTOs
- No hardcoded secrets in any config file — use environment variables (`${ENV_VAR}`) in all profiles
- Swagger UI and `/v3/api-docs` must be disabled in `application-prod.yml`

---

## Development Workflow

1. Write failing tests first
2. Implement to make tests pass
3. Refactor and format code
4. Verify all standards via Checkstyle + SpotBugs
5. Submit PR with test coverage report

## Governance

This constitution supersedes all other practices. Amendments require documentation, team approval, and a migration plan. All PRs must verify compliance with the Java standards files listed above.

**Version**: 1.9.0 | **Ratified**: 2026-02-21 | **Last Amended**: 2026-02-21
