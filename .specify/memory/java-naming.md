# Java & Spring Boot Naming Conventions

## Classes & Interfaces
- Classes: `PascalCase` → `UserService`, `OrderRepository`
- Interfaces: `PascalCase` — **no `I` prefix** (not idiomatic in Spring Boot) → `OrderRepository`, `PaymentGateway`
- Abstract classes: prefix `Abstract` → `AbstractBaseEntity`
- Enums: `PascalCase` → `OrderStatus`

## Spring Boot Layer Naming

| Layer | Suffix | Example |
|-------|--------|---------|
| REST controller | `Controller` | `OrderController` |
| Service interface | `Service` | `OrderService` |
| Service implementation | `ServiceImpl` | `OrderServiceImpl` |
| Repository | `Repository` | `OrderRepository` |
| Configuration | `Config` | `KafkaConfig`, `SecurityConfig` |
| Exception | `Exception` | `OrderNotFoundException`, `DuplicateOrderException` |
| DTO (request) | `RequestDto` | `OrderRequestDto` |
| DTO (response) | `ResponseDto` | `OrderResponseDto` |
| Kafka event | `Event` | `OrderCreatedEvent`, `OrderCancelledEvent` |
| Mapper | `Mapper` | `OrderMapper` |
| Util (stateless helper) | `Util` | `DateUtil`, `ValidationUtil` |

## Methods & Variables
- Methods/variables: `camelCase` → `getUserById()`, `totalAmount`
- Boolean variables/methods: prefix `is`, `has`, `can` → `isActive()`, `hasPermission()`
- Constants: `UPPER_SNAKE_CASE` → `MAX_RETRY_COUNT`

## Packages
- All lowercase, dot-separated → `com.company.project.service`

## Generic Types
- Single uppercase letter → `T` (type), `E` (element), `K`/`V` (key/value)

## Anti-Patterns to Avoid
- No abbreviations unless universally known (`url`, `id`, `dto`)
- No Hungarian notation (`strName`, `intCount`)
- No single-letter names except loop counters (`i`, `j`)
