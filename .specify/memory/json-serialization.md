# JSON Serialization Standards

## Global Rule: snake_case for All JSON

All JSON produced and consumed by this service uses `snake_case` field names.  
Java fields remain `camelCase` — the ObjectMapper handles the translation automatically.

**Example:**
```java
// Java field
private String customerId;

// JSON wire format
{ "customer_id": "abc-123" }
```

---

## ObjectMapper Configuration

Configure globally via `application.yml` — do **not** redefine `ObjectMapper` as a custom `@Bean` unless additional modules are required. Spring Boot's auto-configuration applies this to all REST endpoints, `RestTemplate`, `WebClient`, and manual `objectMapper.writeValueAsString(...)` calls.

```yaml
spring:
  jackson:
    property-naming-strategy: SNAKE_CASE
    default-property-inclusion: NON_NULL
    serialization:
      write-dates-as-timestamps: false
    deserialization:
      fail-on-unknown-properties: false
```

**What each setting does:**

| Setting | Value | Effect |
|---------|-------|--------|
| `property-naming-strategy` | `SNAKE_CASE` | `customerId` → `customer_id` globally |
| `default-property-inclusion` | `NON_NULL` | Omits `null` fields from output |
| `write-dates-as-timestamps` | `false` | Dates as ISO-8601 strings, not epoch millis |
| `fail-on-unknown-properties` | `false` | Tolerant deserialization — ignores extra fields |

---

## If a Custom `@Bean` is Required

Only define a custom `ObjectMapper` bean when additional modules (e.g., `JavaTimeModule`, `ParameterNamesModule`) or custom serializers are needed. Always include the base settings:

```java
@Bean
@Primary
public ObjectMapper objectMapper() {
    return new ObjectMapper()
        .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)
        .setSerializationInclusion(JsonInclude.Include.NON_NULL)
        .registerModule(new JavaTimeModule())
        .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
}
```

---

## DTO Rules

- Java field names are **always** `camelCase` — the ObjectMapper converts to `snake_case` at the boundary
- Do **not** annotate every field with `@JsonProperty("snake_case_name")` — that defeats the global config
- Use `@JsonProperty` **only** when the JSON name must differ from the automatic snake_case conversion
- Use `@JsonIgnore` to exclude fields from serialization (e.g., internal audit fields)
- Do **not** use `@JsonNaming` on individual classes — the global strategy covers all DTOs

```java
// Correct — rely on global snake_case config
public record OrderRequestDto(
    UUID customerId,        // serializes as "customer_id"
    List<OrderItemDto> items,  // serializes as "items"
    String deliveryAddress  // serializes as "delivery_address"
) {}

// Wrong — redundant, conflicts with global config
public record OrderRequestDto(
    @JsonProperty("customer_id") UUID customerId
) {}
```

---

## Kafka Event Payloads

CloudEvent `data` bytes are serialized/deserialized using the same `ObjectMapper` bean — snake_case applies to all event payload fields automatically.

```java
// Serialize payload into CloudEvent data
byte[] data = objectMapper.writeValueAsBytes(orderPayload);

// Deserialize from CloudEvent data
OrderEventPayload payload = objectMapper.readValue(event.getData().toBytes(), OrderEventPayload.class);
```

---

## Non-Negotiables

- `snake_case` is the canonical JSON format for REST responses, request bodies, and Kafka event payloads
- Never mix `camelCase` and `snake_case` in the same API surface
- `null` fields must not appear in JSON output (`NON_NULL` inclusion)
- Dates and timestamps must be ISO-8601 strings — never epoch milliseconds
- `fail-on-unknown-properties=false` is required for forward-compatible deserialization
