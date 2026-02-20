# Diagram Templates

Templates for standalone diagram files generated in `docs/diagrams/`. All diagrams use Mermaid syntax.

---

## Full Project Class Diagram (CLASS_DIAGRAM.md)
Generate a COMPLETE class diagram of the entire project showing every class, interface, enum, and their relationships:
```mermaid
classDiagram
    %% Controllers
    class {ControllerName} {
        -{ServiceType} {service}
        +ResponseEntity~ApiResponse~ {method}({params})
    }

    %% Services
    class {ServiceName} {
        -{RepositoryType} {repo}
        +{ReturnType} {method}({params})
    }

    %% Repositories
    class {RepositoryName} {
        <<interface>>
        +Optional~{Entity}~ {findMethod}({params})
        +boolean {existsMethod}({params})
        +List~{Entity}~ {listMethod}({params})
        +long {countMethod}({params})
    }

    %% Entities
    class {EntityName} {
        -Long id
        -{type} {field}
        +{getter}()
        +{setter}()
    }

    %% DTOs
    class {RequestDTO} {
        -{type} {field}
        +{EntityName} toEntity()
    }
    class {ResponseDTO} {
        -{type} {field}
        +{ResponseDTO} fromEntity({EntityName})
    }
    class ApiResponse~T~ {
        -boolean success
        -String message
        -T data
        -int statusCode
        +ApiResponse~T~ success()
        +ApiResponse~T~ created()
        +ApiResponse~T~ error()
    }

    %% Enums
    class {EnumName} {
        <<enumeration>>
        {VALUE_1}
        {VALUE_2}
        {VALUE_3}
    }

    %% Exceptions
    class {ExceptionName} {
        +{ExceptionName}(String message)
    }
    class GlobalExceptionHandler {
        +ResponseEntity handle{Exception}({ExceptionType} ex)
    }

    %% Relationships
    {ControllerName} --> {ServiceName} : uses
    {ServiceName} --> {RepositoryName} : uses
    {RepositoryName} ..> {EntityName} : manages
    {EntityName} --> {EnumName} : has
    {ControllerName} ..> {RequestDTO} : receives
    {ControllerName} ..> {ResponseDTO} : returns
    {ControllerName} ..> ApiResponse~T~ : wraps response
    {ServiceName} ..> {RequestDTO} : consumes
    {ServiceName} ..> {ResponseDTO} : produces
    {ServiceName} ..> {ExceptionName} : throws
    GlobalExceptionHandler ..> {ExceptionName} : handles
    RuntimeException <|-- {ExceptionName}
```

---

## Entity Relationship Diagram (ENTITY_RELATIONSHIP.md)
Generate a database-level ER diagram:
```mermaid
erDiagram
    {TABLE_NAME} {
        {col_type} {col_name} PK "auto-generated"
        {col_type} {col_name} UK "unique"
        {col_type} {col_name} "not null"
        {col_type} {col_name} "nullable"
    }
    {PARENT_TABLE} ||--o{ {CHILD_TABLE} : "{relationship}"
```

---

## Dependency Graph (DEPENDENCY_GRAPH.md)
```mermaid
graph TD
    Controller --> Service
    Service --> Repository
    Repository --> Entity
```

---

## Sequence Diagrams (SEQUENCE_DIAGRAMS.md)
For each major endpoint, generate:
```mermaid
sequenceDiagram
    Client->>Controller: POST /register
    Controller->>Service: registerUser(request)
    Service->>Repository: existsByEmail(email)
    Repository-->>Service: boolean
    Service->>Repository: save(user)
    Repository-->>Service: User
    Service-->>Controller: UserResponse
    Controller-->>Client: 201 Created
```

---

## Complete Request Flow (FLOW_DIAGRAMS.md)
Generate end-to-end request processing flow:
```mermaid
flowchart TD
    A["HTTP Request"] --> B["DispatcherServlet"]
    B --> C["@RequestMapping Resolution"]
    C --> D{"@Valid Annotation?"}
    D -->|"Yes"| E["Bean Validation"]
    D -->|"No"| F["Controller Method"]
    E -->|"Valid"| F
    E -->|"Invalid"| G["MethodArgumentNotValidException"]
    F --> H["Service Layer"]
    H --> I{"Business Validation"}
    I -->|"Pass"| J["Repository Layer"]
    I -->|"Fail"| K["Custom Exception"]
    J --> L["JPA/Hibernate"]
    L --> M[("Database")]
    M --> L
    L --> J
    J --> H
    H --> N["DTO Transformation"]
    N --> O["ApiResponse Wrapper"]
    O --> P["ResponseEntity"]
    P --> Q["HTTP Response + JSON"]
    G --> R["GlobalExceptionHandler"]
    K --> R
    R --> S["ApiResponse.error()"]
    S --> P

    style A fill:#4CAF50,color:#fff
    style Q fill:#4CAF50,color:#fff
    style G fill:#f44336,color:#fff
    style K fill:#f44336,color:#fff
    style R fill:#FF9800,color:#fff
    style M fill:#2196F3,color:#fff
```

---

## Validation Decision Tree (FLOW_DIAGRAMS.md)
Generate decision flow for all validation rules:
```mermaid
flowchart TD
    A["Incoming Request"] --> B{"fullName present?"}
    B -->|"No"| X1["400: Full name is required"]
    B -->|"Yes"| C{"fullName 2-100 chars?"}
    C -->|"No"| X2["400: Name length invalid"]
    C -->|"Yes"| D{"email valid format?"}
    D -->|"No"| X3["400: Invalid email"]
    D -->|"Yes"| E{"email unique in DB?"}
    E -->|"No"| X4["409: Email already registered"]
    E -->|"Yes"| F{"phone 10-15 digits?"}
    F -->|"No"| X5["400: Invalid phone"]
    F -->|"Yes"| G{"phone unique in DB?"}
    G -->|"No"| X6["409: Phone already registered"]
    G -->|"Yes"| H{"password >= 8 chars?"}
    H -->|"No"| X7["400: Password too short"]
    H -->|"Yes"| I["✅ Create User"]

    style I fill:#4CAF50,color:#fff
    style X1 fill:#f44336,color:#fff
    style X2 fill:#f44336,color:#fff
    style X3 fill:#f44336,color:#fff
    style X4 fill:#FF9800,color:#fff
    style X5 fill:#f44336,color:#fff
    style X6 fill:#FF9800,color:#fff
    style X7 fill:#f44336,color:#fff
```

---

## Coverage Matrix
Generate a cross-reference matrix:

| Component | Has Tests | Test File | Methods Tested |
|-----------|-----------|-----------|----------------|
| UserService | ✅ | UserServiceTest.java | 6/10 |
