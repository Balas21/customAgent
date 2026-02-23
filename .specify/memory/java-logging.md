# Java Logging Standards

## Logger Setup (SLF4J + Logback/Log4j2)
```java
// ✅ Always use SLF4J abstraction
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Slf4j  // Lombok shortcut (preferred)
public class UserService {
    // or manually:
    private static final Logger log = LoggerFactory.getLogger(UserService.class);
}
```

## Log Levels
| Level | When to Use |
|-------|-------------|
| `ERROR` | Unexpected failure, action required |
| `WARN` | Recoverable issue, degraded behavior |
| `INFO` | Key business events (startup, user actions) |
| `DEBUG` | Diagnostic info for development |
| `TRACE` | Very detailed, high-volume (disabled in prod) |

## Message Format
```java
// ✅ Use parameterized logging (never string concatenation)
log.info("Order created: orderId={}, userId={}", orderId, userId);
log.error("Payment failed: orderId={}", orderId, exception);

// ❌ Wrong — eager string evaluation
log.debug("User: " + user.toString());
```

## What to Log
- Service entry points with key inputs (at DEBUG)
- Business events: user login, order placed, payment processed (at INFO)
- All exceptions with context (at ERROR/WARN)
- Performance metrics for slow operations >500ms (at WARN)

## What NOT to Log
- Passwords, tokens, PII (email, phone, SSN)
- Full request/response bodies in production
- Redundant/noisy loops

## Structured Logging (Production)
- Use JSON format in production for log aggregation (ELK, Splunk)
- Include: `timestamp`, `level`, `service`, `traceId`, `message`

---

## MDC — traceId & correlationId Propagation (REQUIRED)

Every incoming request and Kafka event must populate MDC so all log lines in that operation carry `traceId`.

### REST — use a `OncePerRequestFilter`
```java
@Component
public class TraceIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String traceId = Optional.ofNullable(req.getHeader("X-Trace-Id"))
            .orElse(UUID.randomUUID().toString());
        MDC.put("traceId", traceId);
        res.setHeader("X-Trace-Id", traceId);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear(); // always clear after request to prevent thread-pool leaks
        }
    }
}
```

### Kafka Consumer — extract from CloudEvent extension
```java
@KafkaListener(topics = "${kafka.topics.order-created}")
public void handle(CloudEvent event, Acknowledgment ack) {
    String correlationId = (String) event.getExtension("correlationid");
    MDC.put("traceId", correlationId);
    try {
        // all log statements inside now carry traceId automatically
        log.info("Processing event type={} id={}", event.getType(), event.getId());
        ack.acknowledge();
    } finally {
        MDC.clear();
    }
}
```

### Logback config — include traceId in pattern
```xml
<pattern>%d{ISO8601} [%thread] %-5level %logger{36} traceId=%X{traceId} - %msg%n</pattern>
```

**Rules:**
- `MDC.clear()` must always be called in `finally` — never leave MDC populated after a unit of work
- `traceId` key in MDC must match the pattern key in `logback-spring.xml`
- In REST filters, propagate `X-Trace-Id` header outbound to downstream service calls (`RestTemplate`/`WebClient` interceptor)
