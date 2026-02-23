# Java Testing Standards (JUnit 5)

## Coverage Requirements — NON-NEGOTIABLE

| Scope | Required Coverage |
|-------|------------------|
| Service / business logic | **80% minimum** |
| Util classes | **100%** |
| Mapper classes | **100%** |
| Controller layer | **80% minimum** |
| Repository layer | Integration tests required |

**80% is the Quality Gate threshold. PRs below this will not be merged.**
Coverage padding with trivial/empty tests is prohibited — every test must assert meaningful behaviour.

---

## Edge Cases — All Must Be Covered

Every method's test class must explicitly cover:

| Category | Examples |
|----------|---------|
| **Null inputs** | `null` passed for any parameter |
| **Empty inputs** | empty string `""`, empty list `[]`, zero |
| **Boundary values** | min/max allowed values, first/last page |
| **Invalid inputs** | negative IDs, malformed strings, wrong types |
| **Not found** | entity doesn't exist → `NotFoundException` thrown |
| **Duplicate / conflict** | creating an already-existing resource |
| **Unauthorized** | user lacks permission |
| **Happy path** | normal valid input → expected output |
| **Partial data** | optional fields absent |
| **Exception propagation** | dependency throws → correct exception rethrown |

```java
// Example: full edge case coverage for getUser()
@Test void shouldReturnUserWhenValidId() { ... }             // happy path
@Test void shouldThrowNotFoundWhenUserDoesNotExist() { ... } // not found
@Test void shouldThrowWhenUserIdIsNull() { ... }             // null input
@Test void shouldThrowWhenUserIdIsNegative() { ... }         // invalid input
@Test void shouldThrowWhenRepositoryFails() { ... }          // exception propagation
```

---

## Test Structure — AAA Pattern

```java
@Test
void shouldReturnUserWhenValidId() {
    // Arrange
    var user = new User(1L, "Alice");
    when(repo.findById(1L)).thenReturn(Optional.of(user));

    // Act
    var result = service.getUser(1L);

    // Assert
    assertThat(result.getName()).isEqualTo("Alice");
}
```

---

## Naming Convention
- Method: `should[ExpectedBehavior]When[Condition]`
- Test class: `[ClassName]Test`

## Annotations
- `@Test` — standard test
- `@ParameterizedTest` + `@MethodSource` — data-driven tests (use for boundary/invalid input variants)
- `@BeforeEach` / `@AfterEach` — setup/teardown per test
- `@ExtendWith(MockitoExtension.class)` — Mockito integration

## Assertions
- Prefer **AssertJ** (`assertThat`) over JUnit's built-in assertions
- One logical assertion per test (chain with AssertJ where needed)
- For exceptions: use `assertThatThrownBy` or `assertThrows`

```java
@Test
void shouldThrowNotFoundWhenUserDoesNotExist() {
    when(repo.findById(99L)).thenReturn(Optional.empty());

    assertThatThrownBy(() -> service.getUser(99L))
        .isInstanceOf(NotFoundException.class)
        .hasMessageContaining("99");
}
```

## Test Types

| Type | Annotation | Use For |
|------|------------|---------|
| Unit | `@ExtendWith(MockitoExtension.class)` | Service, Util, Mapper — no Spring context |
| Controller (slice) | `@WebMvcTest(XController.class)` | Controller layer only; mocks all services |
| Data (slice) | `@DataJpaTest` | Repository queries against H2/embedded DB |
| Full integration | `@SpringBootTest` | Cross-layer flows; use sparingly |
| Kafka integration | `@SpringBootTest` + `@EmbeddedKafka` | Producer/consumer flows — see `kafka-event-standards.md` |

**Prefer slice tests** (`@WebMvcTest`, `@DataJpaTest`) over `@SpringBootTest` for speed. Use `@SpringBootTest` only when the full application context is required.

## Anti-Patterns
- No logic (`if`/`for`) in test methods
- No shared mutable state between tests
- No `Thread.sleep()` — use Awaitility
- No skipping edge cases because "it's unlikely" — all edges must be covered
- No test that passes vacuously (empty arrange/assert)
- No multiple unrelated assertions in one test — split into separate tests
