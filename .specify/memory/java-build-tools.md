# Java Build Tools (Maven / Gradle)

## Project Choice
- **Maven**: Preferred for enterprise/legacy projects
- **Gradle (Kotlin DSL)**: Preferred for new projects (faster, flexible)

## Maven â€” Key Rules
```xml
<!-- pom.xml must define -->
<groupId>com.company</groupId>
<artifactId>service-name</artifactId>
<version>1.0.0-SNAPSHOT</version>

<!-- Always pin dependency versions via <dependencyManagement> -->
<!-- Use BOM imports for frameworks (Spring Boot, etc.) -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>${spring-boot.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

## Gradle â€” Key Rules
```kotlin
// build.gradle.kts
plugins {
    id("java")
    id("org.springframework.boot") version "3.2.0"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

## Versioning
- Use **Semantic Versioning**: `MAJOR.MINOR.PATCH`
- Snapshots: `1.0.0-SNAPSHOT` (dev); Release: `1.0.0`

## Dependency Rules
- No unused dependencies
- Scope correctly: `provided`, `test`, `runtime` as needed
- Regularly audit with `mvn dependency:analyze` or `gradle dependencies`

## Standard Plugins Required
| Plugin | Purpose |
|--------|---------|
| `checkstyle` | Code style enforcement |
| `jacoco` | Code coverage |
| `spotbugs` | Static analysis |
| `surefire`/`failsafe` | Unit/integration test separation |

---

## OpenAPI Specification & Swagger UI

### Library
Use **`springdoc-openapi`** (not the legacy `springfox`). It integrates natively with Spring Boot 3.x and auto-generates an OpenAPI 3.0 spec from annotations.

**Maven dependency:**
```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.3.0</version>
</dependency>
```

**Gradle (Kotlin DSL):**
```kotlin
implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")
```

### application.yml Configuration
```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    enabled: true          # set false in production
  show-actuator: false
```

### Required `@OpenAPIDefinition` — place on the main application class
```java
@OpenAPIDefinition(
    info = @Info(
        title = "Food Mela Order Service API",
        version = "1.0.0",
        description = "REST API for managing food orders",
        contact = @Contact(name = "Food Mela Team", email = "api@foodmela.com")
    ),
    servers = @Server(url = "http://localhost:8080", description = "Local")
)
@SpringBootApplication
public class FoodMelaOrderServiceApplication { ... }
```

### Controller & DTO Annotation Rules
- Every controller class must have `@Tag(name = "...", description = "...")`
- Every endpoint must have `@Operation(summary = "...", description = "...")`
- Every endpoint must declare `@ApiResponse` for all returned HTTP status codes
- Every request/response DTO must have `@Schema` on the class and on fields that need description or example values
- `@Parameter` must be used on all path/query parameters with `description` and `example`

```java
@Tag(name = "Orders", description = "Order lifecycle management")
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Operation(summary = "Create a new order")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Order created"),
        @ApiResponse(responseCode = "400", description = "Invalid request payload"),
        @ApiResponse(responseCode = "409", description = "Duplicate order detected")
    })
    @PostMapping
    public ResponseEntity<OrderResponseDto> createOrder(@Valid @RequestBody OrderRequestDto request) { ... }
}
```

### Non-Negotiables
- Swagger UI **must be disabled** in production profiles (`springdoc.swagger-ui.enabled=false` in `application-prod.yml`)
- The generated `/v3/api-docs` endpoint must remain accessible in `dev` and `test` profiles for contract testing
- OpenAPI annotations are part of the contract — update them whenever an endpoint or DTO changes
- Do **not** duplicate documentation in Javadoc and `@Operation`; prefer `@Operation` for HTTP-facing behaviour, Javadoc for internal logic
- Commit the static spec file and review it in PRs; use `springdoc-openapi-maven-plugin` to generate `openapi.json` at build time:

```xml
<plugin>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-maven-plugin</artifactId>
  <version>1.4</version>
  <executions>
    <execution>
      <id>generate-openapi</id>
      <goals><goal>generate</goal></goals>
    </execution>
  </executions>
  <configuration>
    <apiDocsUrl>http://localhost:8080/v3/api-docs</apiDocsUrl>
    <outputFileName>openapi.json</outputFileName>
    <outputDir>${project.build.directory}</outputDir>
  </configuration>
</plugin>
```
---

## OpenAPI Specification & Swagger UI

### Library
Use **`springdoc-openapi`** (not the legacy `springfox`). It integrates natively with Spring Boot 3.x and auto-generates an OpenAPI 3.0 spec from annotations.

**Maven dependency:**
```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.3.0</version>
</dependency>
```

**Gradle (Kotlin DSL):**
```kotlin
implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")
```

### application.yml Configuration
```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    enabled: true          # set false in production
  show-actuator: false
```

### Required `@OpenAPIDefinition` - place on the main application class
```java
@OpenAPIDefinition(
    info = @Info(
        title = "Food Mela Order Service API",
        version = "1.0.0",
        description = "REST API for managing food orders",
        contact = @Contact(name = "Food Mela Team", email = "api@foodmela.com")
    ),
    servers = @Server(url = "http://localhost:8080", description = "Local")
)
@SpringBootApplication
public class FoodMelaOrderServiceApplication { ... }
```

### Controller & DTO Annotation Rules
- Every controller class must have `@Tag(name = "...", description = "...")`
- Every endpoint must have `@Operation(summary = "...", description = "...")`
- Every endpoint must declare `@ApiResponse` for all returned HTTP status codes
- Every request/response DTO must have `@Schema` on the class and on fields that need description or example values
- `@Parameter` must be used on all path/query parameters with `description` and `example`

```java
@Tag(name = "Orders", description = "Order lifecycle management")
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Operation(summary = "Create a new order")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Order created"),
        @ApiResponse(responseCode = "400", description = "Invalid request payload"),
        @ApiResponse(responseCode = "409", description = "Duplicate order detected")
    })
    @PostMapping
    public ResponseEntity<OrderResponseDto> createOrder(@Valid @RequestBody OrderRequestDto request) { ... }
}
```

### Non-Negotiables
- Swagger UI **must be disabled** in production profiles (`springdoc.swagger-ui.enabled=false` in `application-prod.yml`)
- The generated `/v3/api-docs` endpoint must remain accessible in `dev` and `test` profiles for contract testing
- OpenAPI annotations are part of the contract - update them whenever an endpoint or DTO changes
- Do **not** duplicate documentation in Javadoc and `@Operation`; prefer `@Operation` for HTTP-facing behaviour, Javadoc for internal logic
- Commit the static spec file and review it in PRs; use `springdoc-openapi-maven-plugin` to generate `openapi.json` at build time:

```xml
<plugin>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-maven-plugin</artifactId>
  <version>1.4</version>
  <executions>
    <execution>
      <id>generate-openapi</id>
      <goals><goal>generate</goal></goals>
    </execution>
  </executions>
  <configuration>
    <apiDocsUrl>http://localhost:8080/v3/api-docs</apiDocsUrl>
    <outputFileName>openapi.json</outputFileName>
    <outputDir>${project.build.directory}</outputDir>
  </configuration>
</plugin>
```