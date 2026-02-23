# Kafka Event Standards

## Event Spec: CloudEvents 1.0

All events **must** conform to the [CloudEvents 1.0](https://cloudevents.io) spec using the official Java SDK.

**Maven dependencies:**
```xml
<dependency>
  <groupId>io.cloudevents</groupId>
  <artifactId>cloudevents-spring</artifactId>
  <version>3.0.0</version>
</dependency>
<dependency>
  <groupId>io.cloudevents</groupId>
  <artifactId>cloudevents-kafka</artifactId>
  <version>3.0.0</version>
</dependency>
```

### Required CloudEvent Attributes

| Attribute | Type | Rule |
|-----------|------|------|
| `id` | UUID string | Unique per event — use `UUID.randomUUID().toString()` |
| `source` | URI | `/food-mela/{service-name}` (e.g., `/food-mela/order-service`) |
| `type` | string | Matches topic name: `food-mela.order.created` |
| `specversion` | string | Always `"1.0"` (set by SDK) |
| `time` | RFC 3339 | `OffsetDateTime.now()` |
| `datacontenttype` | string | Always `"application/json"` |
| `correlationid` | extension | Propagate from inbound request `traceId` |

---

## Topic Naming

Pattern: `{domain}.{entity}.{event}` in `kebab-case`

| Topic | Meaning |
|-------|---------|
| `food-mela.order.created` | Order created |
| `food-mela.order.status-updated` | Order status changed |
| `food-mela.order.cancelled` | Order cancelled |
| `food-mela.order.created.dlq` | Dead-letter for created |
| `food-mela.order.created.retry` | Retry topic for created |

Externalize all topic names in `application.yml` under `kafka.topics.*` — never hardcode in annotations.
---

## application.yml — Canonical Template

> **Rule:** Every value that can differ per environment MUST use `${ENV_VAR:default_value}` syntax.
> The `:default_value` is the local-dev safe default. Production injects the real value via environment variable.
> Never write a bare value without a `${...}` wrapper for anything environment-specific.

```yaml
spring:
  application:
    name: food-mela-order-service

  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/foodmela}
    username: ${DB_USERNAME:foodmela}
    password: ${DB_PASSWORD:foodmela}       # overridden by env var in all non-local profiles
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: validate                   # NEVER create/update — Liquibase owns the schema
    show-sql: false

  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.xml
    enabled: true

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: io.cloudevents.kafka.CloudEventSerializer
      acks: all
      properties:
        enable.idempotence: true
    consumer:
      group-id: ${KAFKA_CONSUMER_GROUP_ID:food-mela-order-service-group}
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: io.cloudevents.kafka.CloudEventDeserializer

kafka:
  topics:
    order-created:        ${KAFKA_TOPIC_ORDER_CREATED:food-mela.order.created}
    order-cancelled:      ${KAFKA_TOPIC_ORDER_CANCELLED:food-mela.order.cancelled}
    order-status-updated: ${KAFKA_TOPIC_ORDER_STATUS_UPDATED:food-mela.order.status-updated}
    order-created-dlq:    ${KAFKA_TOPIC_ORDER_CREATED_DLQ:food-mela.order.created.dlq}
    order-created-retry:  ${KAFKA_TOPIC_ORDER_CREATED_RETRY:food-mela.order.created.retry}

springdoc:
  swagger-ui:
    enabled: ${SWAGGER_ENABLED:true}       # override to false in application-prod.yml
    path: /swagger-ui.html
  api-docs:
    path: /v3/api-docs
    enabled: ${SWAGGER_ENABLED:true}

management:
  endpoints:
    web:
      exposure:
        include: ${ACTUATOR_ENDPOINTS:health,info}
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    root: ${LOG_LEVEL_ROOT:INFO}
    com.foodmela: ${LOG_LEVEL_APP:DEBUG}
```

### Rules for `${ENV_VAR:default_value}`

| Element | Rule |
|---------|------|
| `ENV_VAR` name | `UPPER_SNAKE_CASE` — matches exact OS/container environment variable name |
| `:default_value` | Always present for non-secret values; safe local-dev default |
| Secret values | No `:default_value` for secrets — e.g., `${DB_PASSWORD}` with no fallback forces explicit injection |
| Topic names | Always `${KAFKA_TOPIC_XXX:domain.entity.event}` — default is the canonical topic name |
| Booleans | `${SWAGGER_ENABLED:true}` — default `true` in base, overridden to `false` in prod profile |

**Binding to Java:** Use `@ConfigurationProperties` to bind the `kafka.topics.*` block — never `@Value` for multi-property groups.

```java
@ConfigurationProperties(prefix = "kafka.topics")
public record KafkaTopicsProperties(
    String orderCreated,
    String orderCancelled,
    String orderStatusUpdated,
    String orderCreatedDlq,
    String orderCreatedRetry
) {}
```

Register with `@EnableConfigurationProperties(KafkaTopicsProperties.class)` in `KafkaConfig`.
---

## Producer

```java
// Build and publish via @TransactionalEventListener(phase = AFTER_COMMIT)
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderCreated(OrderCreatedDomainEvent domainEvent) {
    CloudEvent event = CloudEventBuilder.v1()
        .withId(UUID.randomUUID().toString())
        .withSource(URI.create("/food-mela/order-service"))
        .withType("food-mela.order.created")
        .withTime(OffsetDateTime.now())
        .withDataContentType("application/json")
        .withExtension("correlationid", domainEvent.correlationId())
        .withData(objectMapper.writeValueAsBytes(domainEvent.payload()))
        .build();

    kafkaTemplate.send(topic, domainEvent.orderId().toString(), event);
}
```

**Rules:**
- `KafkaTemplate<String, CloudEvent>` — key is always the aggregate root ID
- Use `@TransactionalEventListener(phase = AFTER_COMMIT)` — never publish inside the transaction
- `acks=all` and `enable.idempotence=true` required in producer config
- Log failures with `correlationId` and event `type` — never swallow `KafkaProducerException`

---

## Consumer

```java
@KafkaListener(
    topics = "${kafka.topics.order-created}",
    groupId = "${spring.kafka.consumer.group-id}"
)
public void handleOrderCreated(CloudEvent event, Acknowledgment ack) {
    String correlationId = (String) event.getExtension("correlationid");
    log.info("Received event type={} id={} correlationId={}", event.getType(), event.getId(), correlationId);

    OrderEventPayload payload = objectMapper.readValue(event.getData().toBytes(), OrderEventPayload.class);
    // process — must be idempotent
    ack.acknowledge();
}
```

**Rules:**
- Always use `@KafkaListener` — no manual consumer polling
- Group ID externalized in `application.yml` — never hardcoded in the annotation
- `enable.auto.commit=false` + manual `Acknowledgment.acknowledge()` after successful processing
- `auto.offset.reset=earliest` for new consumer groups
- Consumers **must** be idempotent — duplicate delivery must not cause side effects
- Log `event.getId()`, `event.getType()`, and `correlationId` at `INFO` on receipt and completion

---

## Error Handling

```java
@Bean
public DefaultErrorHandler kafkaErrorHandler(KafkaTemplate<String, CloudEvent> kafkaTemplate) {
    var recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate,
        (record, ex) -> new TopicPartition(record.topic() + ".dlq", record.partition()));

    var backoff = new ExponentialBackOff(1000L, 2.0);
    backoff.setMaxElapsedTime(30_000L);

    return new DefaultErrorHandler(recoverer, backoff);
}
```

**Rules:**
- Exponential backoff before DLQ routing — minimum 2 retries
- DLQ message headers must include: original topic, partition, offset, exception class, exception message
- Poison-pill (undeserializable) messages caught at deserializer level → DLQ immediately, never block consumer group
- Circuit breakers (Resilience4j) required on any synchronous calls made inside a consumer

---

## Testing

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"food-mela.order.created"})
class OrderCreatedConsumerTest {

    @Autowired KafkaTemplate<String, CloudEvent> kafkaTemplate;

    @Test
    void shouldProcessOrderCreatedEventIdempotently() {
        // Publish once, assert side effects
        // Publish same event again, assert no duplicate side effect
    }
}
```

**Rules:**
- `@EmbeddedKafka` for all integration tests — no `KafkaTemplate` mocks when the publish path is under test
- Every `@KafkaListener` method must have an integration test covering: happy path, idempotency, error → DLQ routing
- Producer tests must assert: message key = aggregate root ID, topic, CloudEvent `type` and `id`
