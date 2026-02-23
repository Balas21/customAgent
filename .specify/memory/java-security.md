# Java Security Standards

> **For AI/Copilot:** Every rule in this file is NON-NEGOTIABLE.
> Security constraints must be applied when generating or editing ANY class — not only when explicitly asked.
> If a generated class would violate any rule below, fix it before presenting it.

---

## I. Never Expose JPA Entities from Controllers — EVER

Controllers must NEVER return `@Entity` classes directly. Always map to a response DTO.

```java
// ❌ FORBIDDEN — exposes internal DB structure, hibernate proxies, lazy-load issues
@GetMapping("/{id}")
public Order getOrder(@PathVariable UUID id) {
    return orderRepository.findById(id).orElseThrow();
}

// ✅ CORRECT — map to response DTO before returning
@GetMapping("/{id}")
public ResponseEntity<OrderResponseDto> getOrder(@PathVariable UUID id) {
    return ResponseEntity.ok(orderService.getOrderById(id));
}
```

**Why:** Entities expose internal DB column names, can trigger lazy-load exceptions, and may include fields (audit columns, internal flags) that must not be visible to clients.

---

## II. Bean Validation on ALL Inbound DTOs — Mandatory

Every request DTO field must have an explicit validation annotation. No field may arrive unvalidated.

```java
// ❌ FORBIDDEN — no validation, anything can enter the service layer
public record OrderRequestDto(UUID customerId, List<OrderItemDto> items) {}

// ✅ CORRECT — every field validated
public record OrderRequestDto(

    @NotNull(message = "customer_id is required")
    UUID customerId,

    @NotEmpty(message = "Order must contain at least one item")
    @Valid
    List<@Valid OrderItemDto> items,

    @Size(max = 500, message = "Delivery address must not exceed 500 characters")
    @NotBlank(message = "delivery_address is required")
    String deliveryAddress

) {}
```

**Rules:**
- `@Valid` on the controller method parameter — always
- `@Valid` cascaded on nested DTO collections
- Validation messages must be human-readable and match the JSON field name (snake_case)
- Custom validators (`@Constraint`) for domain rules not covered by standard annotations (e.g., UUID format, enum value)
- Validation failure handled globally by `GlobalExceptionHandler` — never catch `MethodArgumentNotValidException` inside a controller

---

## III. PII & Sensitive Data — Must Never Appear in Logs or Responses

**PII includes:** name, email, phone number, address, date of birth, national ID, payment card data, IP address when linked to identity.

**Sensitive credentials:** passwords, tokens, API keys, secrets, PINs.

### Logging Rules

```java
// ❌ FORBIDDEN
log.info("Processing order for customer email={} phone={}", email, phone);
log.debug("User login attempt: username={} password={}", username, password);

// ✅ CORRECT — log only non-sensitive identifiers
log.info("Processing order: orderId={} customerId={}", orderId, customerId);
log.info("User login attempt: userId={}", userId);
```

**Hard rules:**
- Never log: passwords, tokens, API keys, full card numbers, CVV, email addresses, phone numbers, national IDs
- Log only opaque identifiers (`customerId`, `orderId`, `userId`) — never the values they point to
- Use `// SECURITY:` inline comment on any log line that is intentionally near sensitive data to signal it has been reviewed

### Response DTO Rules

```java
// ❌ FORBIDDEN — exposing password hash in response
public record UserResponseDto(UUID id, String username, String passwordHash) {}

// ✅ CORRECT — sensitive fields excluded entirely
public record UserResponseDto(UUID id, String username) {}
```

- Password fields (`password`, `passwordHash`, `passwordSalt`) must NEVER appear in any response DTO
- Token/secret fields (`accessToken`, `refreshToken`, `apiKey`) must only appear in dedicated auth response DTOs — never in general-purpose entity response DTOs
- Use `@JsonIgnore` as a last-resort safety net on `@Entity` fields, but the primary guard is DTO separation — do not rely on `@JsonIgnore` alone

---

## IV. Password Handling

```java
// ❌ FORBIDDEN — plaintext, MD5, SHA-1
user.setPassword(plainPassword);
user.setPassword(md5Hash(plainPassword));

// ✅ CORRECT — BCrypt via Spring Security
user.setPassword(passwordEncoder.encode(plainPassword));
```

**Rules:**
- Passwords stored using `BCryptPasswordEncoder` with strength ≥ 12
- Never store plaintext, MD5, or SHA-1 hashed passwords
- Never compare passwords with `equals()` — use `passwordEncoder.matches(raw, encoded)`
- Never return a password (even hashed) in any API response or log
- Password fields in request DTOs must be annotated `@NotBlank` + `@Size(min=8, max=128)` + a custom `@StrongPassword` validator if complexity rules apply

---

## V. No Sensitive Values in Code or Config Files

```yaml
# ❌ FORBIDDEN — credentials in application.yml committed to git
spring:
  datasource:
    password: mySecretPassword123

# ✅ CORRECT — reference environment variable
spring:
  datasource:
    password: ${DB_PASSWORD}
```

**Rules:**
- All secrets (DB passwords, Kafka credentials, API keys, JWT secrets) must come from environment variables or a secrets manager (e.g., AWS Secrets Manager, HashiCorp Vault)
- `application-prod.yml` must contain NO literal secret values — only `${ENV_VAR}` references
- `.gitignore` must exclude any local `.env` files
- Never hardcode `localhost` URLs, ports, or credentials in any non-test profile

---

## VI. Actuator Endpoint Security

Spring Boot Actuator exposes internal metrics and health data — must be locked down in production.

```yaml
# application.yml — safe defaults
management:
  endpoints:
    web:
      exposure:
        include: ${ACTUATOR_ENDPOINTS:health,info}   # override to add more in internal profiles only
  endpoint:
    health:
      show-details: when-authorized  # never "always" in production
```

**Rules:**
- Never expose `env`, `beans`, `mappings`, `threaddump`, `heapdump`, or `shutdown` actuator endpoints in production
- `health` and `info` endpoints are the only ones permitted publicly
- All other actuator endpoints must be behind authentication (role `ACTUATOR_ADMIN`) or on a separate management port not reachable from the public network

---

## VII. SQL Injection Prevention

```java
// ❌ FORBIDDEN — string concatenation in query
@Query("SELECT o FROM Order o WHERE o.status = '" + status + "'")

// ✅ CORRECT — parameterised query
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatus(@Param("status") OrderStatus status);
```

**Rules:**
- All queries use Spring Data method names, `@Query` with `:named` parameters, or `JdbcTemplate` with `?` placeholders
- String concatenation into any query is forbidden — no exceptions
- Raw `EntityManager.createNativeQuery(string)` with unbound user input is forbidden

---

## VIII. CORS Configuration

```java
// ✅ CORRECT — explicit, restrictive CORS config in @Configuration class
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("${security.cors.allowed-origins}"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Trace-Id"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

**Rules:**
- `allowedOrigins` must never be `*` in non-local profiles — use `${security.cors.allowed-origins}` externalized in `application.yml`
- Allowed methods must be an explicit list — never `*`
- CORS config lives in `config/SecurityConfig.java` — never scattered across individual controllers via `@CrossOrigin`

`application.yml` entry:
```yaml
security:
  cors:
    allowed-origins: ${CORS_ALLOWED_ORIGINS:http://localhost:3000,http://localhost:4200}
```

---

## IX. HTTP Security Headers

All responses must include the following security headers, configured in `SecurityConfig`:

| Header | Required Value |
|--------|---------------|
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `X-XSS-Protection` | `1; mode=block` |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |
| `Content-Security-Policy` | `default-src 'self'` (adjust for Swagger UI in dev) |

```java
http.headers(headers -> headers
    .contentTypeOptions(withDefaults())
    .frameOptions(frame -> frame.deny())
    .httpStrictTransportSecurity(hsts -> hsts.maxAgeInSeconds(31536000))
);
```

---

## X. Swagger / OpenAPI Access Control

```yaml
# application-prod.yml
springdoc:
  swagger-ui:
    enabled: false      # MANDATORY in production
  api-docs:
    enabled: false      # MANDATORY in production
```

**Rules:**
- Swagger UI and `/v3/api-docs` must be disabled in `application-prod.yml` — no exceptions
- In `dev` and `test` profiles they may be enabled
- If Swagger must be accessible in a staging environment, it must be behind authentication

---

## XI. Input Size & Rate Limit Guards

- All string fields in request DTOs must have a `@Size(max = N)` constraint — no unbounded strings
- File upload endpoints (if any) must enforce `spring.servlet.multipart.max-file-size` and `max-request-size` in config
- Rate limiting is enforced at the API gateway level; the service must not assume uncapped throughput

---

## Non-Negotiables Summary

| Rule | Enforcement point |
|------|-------------------|
| No `@Entity` in controller responses | `mapper/` + response DTO always |
| `@Valid` on all controller request params | Controller method signature |
| No PII in logs | All `log.*` calls |
| No passwords/tokens in response DTOs | DTO field list |
| BCrypt strength ≥ 12 for passwords | `PasswordEncoder` bean config |
| No hardcoded secrets in config files | `application-prod.yml` — env vars only |
| Actuator: only `health` + `info` exposed | `application.yml` |
| No string concatenation in queries | All `@Query` usages |
| CORS: no wildcard origins in non-local | `SecurityConfig` |
| Swagger disabled in production | `application-prod.yml` |
