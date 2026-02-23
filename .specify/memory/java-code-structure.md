# Java Code Structure — Package & Folder Layout

> **For AI/Copilot:** Every new class MUST be placed in the exact package defined in this file.
> Do NOT create packages that are not listed here. Do NOT place a class in a convenient package — place it in the correct one.
> When in doubt about placement, consult the decision tree below.

---

## Canonical Package Structure

```
com.foodmela.orderservice/
│
├── FoodMelaOrderServiceApplication.java   ← @SpringBootApplication entry point ONLY
│
├── config/                                ← Spring configuration classes
│   ├── KafkaConfig.java                   ← Kafka producer/consumer beans, error handler
│   ├── JacksonConfig.java                 ← ObjectMapper bean (only if custom modules needed)
│   ├── OpenApiConfig.java                 ← @OpenAPIDefinition, Swagger bean config
│   └── SecurityConfig.java                ← Spring Security config (if applicable)
│
├── controller/                            ← REST layer ONLY — no business logic
│   └── OrderController.java               ← @RestController, @RequestMapping
│
├── service/                               ← Business logic interfaces + implementations
│   ├── OrderService.java                  ← Interface (defines contract)
│   └── impl/
│       └── OrderServiceImpl.java          ← @Service implementation
│
├── repository/                            ← Spring Data JPA interfaces ONLY
│   └── OrderRepository.java               ← extends JpaRepository<Order, UUID>
│
├── model/                                 ← JPA entity classes (@Entity)
│   ├── Order.java
│   ├── OrderItem.java
│   └── OrderStatus.java                   ← Enum for entity state
│
├── dto/                                   ← Data Transfer Objects (no JPA annotations)
│   ├── request/
│   │   └── OrderRequestDto.java           ← Inbound payload (Bean Validation annotations here)
│   └── response/
│       └── OrderResponseDto.java          ← Outbound payload (never expose @Entity directly)
│
├── event/                                 ← Kafka CloudEvent payload classes
│   ├── OrderCreatedEvent.java             ← record / immutable class
│   ├── OrderCancelledEvent.java
│   └── OrderStatusUpdatedEvent.java
│
├── exception/                             ← Custom exception classes + global handler
│   ├── OrderNotFoundException.java        ← extends RuntimeException
│   ├── DuplicateOrderException.java       ← extends RuntimeException
│   ├── InvalidOrderStateException.java    ← extends RuntimeException
│   └── GlobalExceptionHandler.java        ← @RestControllerAdvice
│
├── mapper/                                ← MapStruct interfaces for entity ↔ DTO conversion
│   └── OrderMapper.java                   ← @Mapper(componentModel = "spring")
│
├── constants/                             ← Named constants (no magic numbers/strings)
│   ├── OrderConstants.java                ← order-domain limits, defaults
│   └── KafkaTopics.java                   ← topic name constants (mirrors application.yml keys)
│
├── util/                                  ← Stateless helper methods (final class, private ctor)
│   └── OrderValidationUtil.java
│
└── filter/                                ← Servlet filters (e.g., traceId MDC population)
    └── TraceIdFilter.java                 ← OncePerRequestFilter for MDC traceId
```

---

## Resources Folder Structure

> **For AI/Copilot:** Database migration files MUST follow this layout exactly.
> Never place SQL or XML changelog files inside `src/main/java` or outside `db/changelog/`.

```
src/
├── main/
│   ├── java/                              ← Java source (see package tree above)
│   └── resources/
│       ├── application.yml                ← base config (no environment-specific values)
│       ├── application-dev.yml            ← dev overrides (local DB, debug logging)
│       ├── application-test.yml           ← test overrides (H2, embedded Kafka)
│       ├── application-prod.yml           ← prod overrides (Swagger disabled, prod DB)
│       └── db/
│           └── changelog/
│               ├── db.changelog-master.xml   ← Liquibase master file (includes all others)
│               ├── v1.0.0/
│               │   ├── 001-create-order-table.sql
│               │   └── 002-create-order-item-table.sql
│               └── v1.1.0/
│                   └── 001-add-duplicate-check-index.sql
└── test/
    └── resources/
        ├── application-test.yml           ← test profile config
        └── db/
            └── changelog/
                └── db.changelog-test.xml  ← test-specific data fixtures (if needed)
```

---

## Liquibase Rules — Non-Negotiable

**Master file:** `db/changelog/db.changelog-master.xml` — the only file Liquibase reads directly. It `<include>`s all versioned changelogs.

```xml
<!-- db.changelog-master.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.0.xsd">

    <include file="db/changelog/v1.0.0/001-create-order-table.sql"
             relativeToChangelogFile="false"/>
    <include file="db/changelog/v1.0.0/002-create-order-item-table.sql"
             relativeToChangelogFile="false"/>
    <include file="db/changelog/v1.1.0/001-add-duplicate-check-index.sql"
             relativeToChangelogFile="false"/>
</databaseChangeLog>
```

**Changelog file naming:** `{sequence}-{description-in-kebab-case}.sql`
- Sequence is zero-padded 3 digits: `001`, `002`, `003`
- Description is lowercase kebab-case: `create-order-table`, `add-status-index`

**Versioned directories:** One directory per application version (`v1.0.0/`, `v1.1.0/`). New migrations for a feature go in a new version folder — never add files into an already-released version directory.

**Changesets:**
- Every SQL file must have a Liquibase changeset header comment with `id`, `author`, and `description`
- `id` format: `{version}-{sequence}` (e.g., `v1.0.0-001`)
- Never modify or delete an existing changeset — always add a new one
- Destructive changes (drop column, rename) require a rollback block

```sql
-- liquibase formatted sql

-- changeset foodmela:v1.0.0-001 labels:v1.0.0
-- comment: Creates the order table
CREATE TABLE orders (
    id            UUID         PRIMARY KEY,
    customer_id   UUID         NOT NULL,
    status        VARCHAR(50)  NOT NULL,
    created_at    TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- rollback DROP TABLE orders;
```

**`application.yml` Liquibase config:**
```yaml
spring:
  liquibase:
    change-log: ${LIQUIBASE_CHANGELOG:classpath:db/changelog/db.changelog-master.xml}
    enabled: ${LIQUIBASE_ENABLED:true}   # override to false in test profile if using H2 schema-auto
```

**Rules summary:**
- Never write raw DDL/DML directly in the application — all schema changes go through Liquibase
- Never set `spring.jpa.hibernate.ddl-auto` to `create`, `create-drop`, or `update` in any profile — use `validate` (prod) or `none` (test with Liquibase)
- Every migration must be reviewed in the PR alongside the code change that requires it
- Column and table names use `snake_case` — mirrors the Jackson `snake_case` JSON convention

| Class / File type | Location | Annotation / Pattern |
|-------------------|----------|----------------------|
| Spring Boot entry point | `src/main/java/.../` root | `@SpringBootApplication` |
| REST endpoint | `controller/` | `@RestController` |
| Business logic interface | `service/` | — |
| Business logic implementation | `service/impl/` | `@Service` |
| JPA repository | `repository/` | — (extends `JpaRepository`) |
| JPA entity | `model/` | `@Entity` |
| Request DTO | `dto/request/` | Bean Validation annotations |
| Response DTO | `dto/response/` | `@Schema` (OpenAPI) |
| Kafka event payload | `event/` | `record` or immutable class |
| Custom exception | `exception/` | extends `RuntimeException` |
| Global error handler | `exception/` | `@RestControllerAdvice` |
| MapStruct mapper | `mapper/` | `@Mapper(componentModel = "spring")` |
| Named constants | `constants/` | `public final class`, `private` ctor |
| Stateless util methods | `util/` | `public final class`, `private` ctor |
| Spring config | `config/` | `@Configuration` |
| Servlet filter | `filter/` | extends `OncePerRequestFilter` |
| Liquibase master file | `src/main/resources/db/changelog/` | `db.changelog-master.xml` |
| Liquibase migration | `src/main/resources/db/changelog/v{x.y.z}/` | `{seq}-{description}.sql` |

---

## Package Rules

**`controller/`**
- One controller class per top-level resource (`OrderController`, not `GetOrderController`)
- No business logic — delegate entirely to `service/`
- No `try-catch` — exceptions bubble to `GlobalExceptionHandler`
- No direct `repository/` access from controllers — ever

**`service/`**
- One interface + one `impl/` class per domain aggregate
- `@Transactional` annotations go here — never on controller or repository
- All Kafka event publishing goes here via `@TransactionalEventListener(phase = AFTER_COMMIT)`
- Never inject `HttpServletRequest` or Spring MVC types — services must be web-layer agnostic

**`repository/`**
- Spring Data interfaces only — no `@Service` or `@Component` classes here
- Custom queries use `@Query` — no business logic
- No `@Transactional` — managed by the service layer

**`model/`**
- `@Entity` classes only — no DTOs, no logic classes
- Enums that represent entity state live here (e.g., `OrderStatus`)
- No Jackson annotations (`@JsonProperty`, etc.) on entities — use DTOs for serialization

**`dto/`**
- Split into `request/` and `response/` sub-packages
- Request DTOs carry Bean Validation annotations (`@NotNull`, `@Size`, `@Valid`)
- Response DTOs carry `@Schema` for OpenAPI documentation
- No JPA annotations on DTOs — never
- DTOs are Java `record` types wherever possible (immutable, concise)

**`event/`**
- Kafka CloudEvent payload classes — one class per event type
- Always `record` or explicitly immutable (`final` fields, no setters)
- No Spring annotations — plain Java
- Named in past tense: `OrderCreatedEvent`, not `CreateOrderEvent`

**`exception/`**
- All custom exceptions are unchecked (`extends RuntimeException`)
- Exception class name must end with `Exception`
- `GlobalExceptionHandler` is the only `@RestControllerAdvice` in the project
- Error response shape is defined here (e.g., `ErrorResponse` record)

**`mapper/`**
- MapStruct `@Mapper(componentModel = "spring")` interfaces only
- Method names: `toDto(Entity)` → DTO, `toEntity(Dto)` → Entity
- No business logic — field mapping only
- Hand-written mappers follow the same `final class / private constructor / static methods` pattern

**`constants/`**
- One `final` class per domain area — never a single `Constants.java` dumping ground
- `KafkaTopics.java` holds `public static final String` constants mirroring `application.yml` kafka topic keys
- Never use raw string literals for topic names anywhere else in the codebase

**`util/`**
- `final` class + `private` constructor — cannot be instantiated
- All methods `static` — no instance state
- No `@Component` / `@Service` — not a Spring bean
- One util class per concern (`OrderValidationUtil`, not `Util`)

**`config/`**
- One `@Configuration` class per infrastructure concern (Kafka, Jackson, Security, OpenAPI)
- No business logic — bean definitions only
- `KafkaConfig` defines: producer factory, consumer factory, `KafkaTemplate`, `DefaultErrorHandler` with `DeadLetterPublishingRecoverer`

**`filter/`**
- `TraceIdFilter` is the only filter required — reads `X-Trace-Id` header (or generates UUID), sets `MDC.put("traceId", ...)`, clears MDC in `finally`

---

## What Does NOT Get Its Own Package

| Temptation | Rule |
|------------|------|
| `api/` or `web/` alongside `controller/` | Use `controller/` only |
| `models/` or `entities/` | Use `model/` only |
| `services/` (plural) | Use `service/` (singular) |
| `dtos/` or `data/` | Use `dto/` only |
| `events/` (plural) | Use `event/` (singular) |
| `exceptions/` (plural) | Use `exception/` (singular) |
| `utils/` (plural) | Use `util/` (singular) |
| `helpers/` | Use `util/` for stateless, `service/impl/` for stateful |
| `common/` or `shared/` | Extract to a shared library if needed across services — not a package in this service |
| `base/` | Abstract base classes go in the package of the layer they belong to |

---

## Class Placement Decision Tree

```
New class needed?
  │
  ├─ Receives HTTP request? ──────────────────────────────── controller/
  │
  ├─ Contains business logic?
  │     ├─ Interface ──────────────────────────────────────── service/
  │     └─ Implementation ───────────────────────────────── service/impl/
  │
  ├─ Talks to the database? ────────────────────────────── repository/
  │
  ├─ Is a JPA entity / enum for entity state? ──────────── model/
  │
  ├─ Carries data between layers?
  │     ├─ Inbound (request) ──────────────────────────── dto/request/
  │     └─ Outbound (response) ──────────────────────────── dto/response/
  │
  ├─ Is a Kafka event payload? ─────────────────────────── event/
  │
  ├─ Is an exception or error handler? ─────────────────── exception/
  │
  ├─ Converts entity ↔ DTO? ────────────────────────────── mapper/
  │
  ├─ Holds named constants? ────────────────────────────── constants/
  │
  ├─ Stateless pure-logic helper? ──────────────────────── util/
  │
  ├─ Spring bean / infrastructure config? ──────────────── config/
  │
  ├─ Servlet filter? ───────────────────────────────────── filter/
  │
  └─ Database schema / migration change?
        ├─ Master include file ──────── src/main/resources/db/changelog/db.changelog-master.xml
        └─ New migration SQL ──────── src/main/resources/db/changelog/v{x.y.z}/{seq}-{desc}.sql
```

---

## Reusability Rules

| What it holds | Where it goes |
|---------------|---------------|
| Reusable **logic** (methods) | `util/` |
| Fixed **values** (numbers, strings) | `constants/` |
| Object **conversion** (entity ↔ DTO) | `mapper/` |
| Shared **business logic** with dependencies | `service/` |

- No `Utils.java` dumping ground — one util class per concern
- No copying method bodies between classes — always extract
- No static methods on `@Service` classes — move them to `util/`
- No business logic inside `mapper/` classes
- A method exceeding 20 lines must be broken into private named methods

