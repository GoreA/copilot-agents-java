# Spring Boot 4 Best Practices (Spring Boot 4.0.x + Java 25)

> **Note**: Spring Boot 4 is built on Spring Framework 7 and Jakarta EE 11.
> Java 25 is the latest LTS release (September 2025) and the recommended target for Spring Boot 4.
> This file serves as a technical reference for patterns and best practices.

## Prerequisites
- **Java**: 25 (LTS)
- **Maven**: 3.9.0+
- **Spring Boot**: 4.0.x (latest: 4.0.5)

## Project Setup

### Verify Java Version
```bash
java -version  # Should be 25.x.x
mvn --version  # Should be 3.9.0+
```

### Spring Boot CLI (Optional)
```bash
# Install Spring Boot CLI via SDKMAN
sdk install springboot

# Verify
spring --version
```

### Spring Initializr
```bash
# Generate a new project via curl
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,validation,postgresql \
  -d type=maven-project \
  -d javaVersion=25 \
  -d bootVersion=4.0.5 \
  -d groupId=com.example \
  -d artifactId=myapp \
  -o myapp.zip
```

---

## Java 25 Features to Use

### 1. Records (Data Classes)
Use records for DTOs and immutable data.

```java
// ✅ GOOD - Record for DTO
public record UserDto(Long id, String name, String email) {}

// ✅ GOOD - Record with validation
public record CreateUserRequest(
    @NotBlank String name,
    @Email String email
) {}

// ❌ BAD - Traditional class for simple DTO
public class UserDto {
    private Long id;
    private String name;
    // getters, setters, equals, hashCode...
}
```

### 2. Pattern Matching with Primitives (JEP 507)
Java 25 extends pattern matching to all primitive types in `instanceof` and `switch`.

```java
// ✅ GOOD - Primitive pattern matching (new in Java 25)
static String describeValue(Object obj) {
    return switch (obj) {
        case int i when i > 0    -> "Positive int: " + i;
        case int i               -> "Non-positive int: " + i;
        case double d            -> "Double: " + d;
        case String s            -> "String: " + s;
        case null                -> "Unknown";
        default                  -> "Other: " + obj;
    };
}

// ✅ GOOD - Primitive instanceof
static void process(Object obj) {
    if (obj instanceof int i) {
        System.out.println("It's an int: " + i);
    }
}

// ✅ GOOD - Record patterns
public double calculateDiscount(Customer customer) {
    return switch (customer) {
        case PremiumCustomer(var name, var level) when level > 5 -> 0.20;
        case PremiumCustomer(var name, var level) -> 0.10;
        case RegularCustomer(var name) -> 0.05;
    };
}
```

### 3. Flexible Constructor Bodies (JEP 513)
Run validation and logic before `super()` or `this()` calls.

```java
// ✅ GOOD - Flexible constructor (new in Java 25)
public class VerifiedUser extends User {

    public VerifiedUser(String name, String email) {
        // Logic BEFORE super() — previously illegal
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name must not be blank");
        }
        String normalized = email.strip().toLowerCase();
        super(name, normalized);
    }
}

// ❌ BAD - Pre-Java 25: had to use static factory methods to validate before construction
```

### 4. Module Import Declarations (JEP 511)
Import all exported packages of a module in one line.

```java
// ✅ GOOD - Module import (new in Java 25)
import module java.base;

// Replaces dozens of individual imports for java.util.*, java.io.*, etc.
List<String> items = Stream.of("a", "b", "c")
    .collect(Collectors.toList());
```

### 5. Compact Source Files & Instance Main Methods (JEP 512)
Simpler entry points — useful for quick prototypes and scripts.

```java
// ✅ GOOD - Compact source file (new in Java 25)
// No class declaration needed, instance main method
void main() {
    System.out.println("Hello, Spring Boot 4!");
}
```

### 6. Scoped Values (JEP 501 — Finalized)
Share immutable data within a thread scope — a modern replacement for `ThreadLocal`.

```java
// ✅ GOOD - Scoped values (finalized in Java 25)
private static final ScopedValue<String> CURRENT_USER = ScopedValue.newInstance();

public void handleRequest(String userId) {
    ScopedValue.runWhere(CURRENT_USER, userId, () -> {
        processOrder();  // CURRENT_USER is available throughout this scope
    });
}

private void processOrder() {
    String user = CURRENT_USER.get();  // No need to pass as parameter
    log.info("Processing order for user: {}", user);
}
```

### 7. Stable Values (JEP 502 — Preview)
Deferred immutability — lazily initialized constants optimized by the JVM.

```java
// ✅ GOOD - Stable values for lazy initialization (preview in Java 25)
@Service
public class OrderService {

    private final StableValue<Logger> logger = StableValue.of();

    Logger getLogger() {
        return logger.orElseSet(() -> Logger.create(OrderService.class));
    }
}
```

### 8. Text Blocks
Use text blocks for multi-line strings.

```java
// ✅ GOOD - Text block
String query = """
    SELECT u.id, u.name, u.email
    FROM users u
    WHERE u.active = true
    ORDER BY u.name
    """;

// ❌ BAD - String concatenation
String query = "SELECT u.id, u.name, u.email " +
               "FROM users u " +
               "WHERE u.active = true " +
               "ORDER BY u.name";
```

### 9. Sequenced Collections
Use new collection methods.

```java
// ✅ GOOD - Sequenced collections
List<String> items = new ArrayList<>();
items.addFirst("first");
items.addLast("last");
String first = items.getFirst();
String last = items.getLast();

// Reversed iteration
for (String item : items.reversed()) {
    // process in reverse
}
```

### 10. Compact Object Headers (JEP 519)
Enabled as a product feature in Java 25 — reduces object header size from 96–128 bits to 64 bits. No code changes needed; improves memory footprint automatically.

```bash
# Compact object headers are a product feature — enable via JVM flag
java -XX:+UseCompactObjectHeaders -jar myapp.jar
```

---

## Spring Boot 4 Best Practices

### 1. REST Controllers

#### Modern Controller Structure
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public List<UserDto> list() {
        return userService.findAll();
    }

    @GetMapping("/{id}")
    public UserDto getById(@PathVariable Long id) {
        return userService.findById(id)
            .orElseThrow(() -> new ResponseStatusException(
                HttpStatus.NOT_FOUND, "User not found"));
    }

    @PostMapping
    public ResponseEntity<UserDto> create(@Valid @RequestBody CreateUserRequest request) {
        UserDto created = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    @PutMapping("/{id}")
    public UserDto update(@PathVariable Long id,
                          @Valid @RequestBody UpdateUserRequest request) {
        return userService.update(id, request);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

#### API Versioning (New in Spring Boot 4 / Spring Framework 7)
```java
// ✅ NEW - First-class API versioning support
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping(value = "/search", version = "1")
    public List<ProductV1> searchV1(@RequestParam String query) {
        // v1 implementation
    }

    @GetMapping(value = "/search", version = "2")
    public ProductSearchResponseV2 searchV2(@RequestParam String query,
                                            @RequestParam(defaultValue = "10") int limit) {
        // v2 implementation with pagination
    }
}
```

### 2. Dependency Injection

#### Use Constructor Injection (Preferred)
```java
// ✅ GOOD - Constructor injection (no @Autowired needed with single constructor)
@Service
public class UserService {

    private final UserRepository repository;
    private final EmailService emailService;

    public UserService(UserRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
}

// ❌ BAD - Field injection
@Service
public class UserService {
    @Autowired
    UserRepository repository;  // Harder to test
}
```

#### Bean Scopes
```java
// ✅ Service layer - singleton (default in Spring)
@Service
public class UserService { }

// ✅ Request-scoped data
@Component
@RequestScope
public class RequestContext { }

// ✅ Prototype scope (new instance per injection)
@Component
@Scope("prototype")
public class Helper { }
```

### 3. Spring Data JPA

#### Repository Pattern
```java
// ✅ GOOD - JPA Entity
@Entity
@Table(name = "users")
public class UserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    private boolean active = true;

    // Getters and setters (required in Spring Boot 4 — public field binding removed)
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public boolean isActive() { return active; }
    public void setActive(boolean active) { this.active = active; }
}
```

```java
// ✅ GOOD - Spring Data Repository
public interface UserRepository extends JpaRepository<UserEntity, Long> {

    List<UserEntity> findByActiveTrue();

    Optional<UserEntity> findByEmail(String email);

    @Query("SELECT u FROM UserEntity u WHERE u.name LIKE %:name%")
    List<UserEntity> searchByName(@Param("name") String name);
}
```

### 4. Transactions

#### Declarative Transactions
```java
// ✅ GOOD - Declarative @Transactional
@Service
public class UserService {

    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public UserDto create(CreateUserRequest request) {
        UserEntity user = new UserEntity();
        user.setName(request.name());
        user.setEmail(request.email());
        repository.save(user);
        return toDto(user);
    }

    @Transactional
    public void delete(Long id) {
        repository.deleteById(id);
    }

    // Read-only (optimized — no dirty checking)
    @Transactional(readOnly = true)
    public List<UserDto> findAll() {
        return repository.findAll().stream()
            .map(this::toDto)
            .toList();
    }
}
```

### 5. Configuration

#### Use `@ConfigurationProperties` (Preferred)
```java
// ✅ GOOD - Type-safe configuration (public field binding removed in Boot 4)
@ConfigurationProperties(prefix = "email")
public class EmailProperties {

    private String from;
    private boolean enabled = true;
    private int retryAttempts = 3;

    // Getters and setters required
    public String getFrom() { return from; }
    public void setFrom(String from) { this.from = from; }
    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    public int getRetryAttempts() { return retryAttempts; }
    public void setRetryAttempts(int retryAttempts) { this.retryAttempts = retryAttempts; }
}
```

```java
// Enable in your main class or a config class
@SpringBootApplication
@EnableConfigurationProperties(EmailProperties.class)
public class MyApplication { }
```

#### Use `@Value` for Simple Cases
```java
// ✅ OK for simple values
@Service
public class EmailService {

    @Value("${email.from}")
    private String fromAddress;

    @Value("${email.enabled:true}")
    private boolean enabled;
}
```

#### application.yaml
```yaml
spring:
  # Database
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: postgres
    password: postgres

  # JPA / Hibernate
  jpa:
    open-in-view: false  # Best practice: disable OSIV
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true

  # Jackson (3.x in Boot 4)
  jackson:
    serialization:
      write-dates-as-timestamps: false

  # Virtual threads (enabled by default in Boot 4)
  threads:
    virtual:
      enabled: true

# Server
server:
  port: 8080

# CORS
spring:
  web:
    cors:
      allowed-origins: http://localhost:4200

# Dev profile
---
spring:
  config:
    activate:
      on-profile: dev
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
```

### 6. Exception Handling

#### Global Exception Handler with `@ControllerAdvice`
```java
// ✅ GOOD - Global exception handling (uses Java 25 pattern matching in switch)
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResponseStatusException.class)
    public ResponseEntity<ErrorResponse> handleResponseStatus(ResponseStatusException ex) {
        return ResponseEntity.status(ex.getStatusCode())
            .body(new ErrorResponse(ex.getStatusCode().value(), ex.getReason()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
            .body(new ErrorResponse(400, message));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        return switch (ex) {
            case IllegalArgumentException e ->
                ResponseEntity.badRequest()
                    .body(new ErrorResponse(400, e.getMessage()));
            case IllegalStateException e ->
                ResponseEntity.status(HttpStatus.CONFLICT)
                    .body(new ErrorResponse(409, e.getMessage()));
            default ->
                ResponseEntity.internalServerError()
                    .body(new ErrorResponse(500, "Internal server error"));
        };
    }
}

public record ErrorResponse(int code, String message) {}
```

### 7. Validation

#### Bean Validation (Jakarta Validation 3.1)
```java
// ✅ GOOD - Use Jakarta Bean Validation
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100)
    String name,

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email
) {}

// In controller — @Valid triggers automatic validation
@PostMapping
public ResponseEntity<UserDto> create(@Valid @RequestBody CreateUserRequest request) {
    UserDto created = userService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(created);
}
```

### 8. HTTP Interface Client (Declarative REST Client)

#### Declarative HTTP Client (Spring Framework 7)
```java
// ✅ GOOD - Declarative HTTP interface client
public interface UserApiClient {

    @GetExchange("/api/users")
    List<UserDto> getUsers();

    @GetExchange("/api/users/{id}")
    UserDto getUser(@PathVariable Long id);

    @PostExchange("/api/users")
    UserDto createUser(@RequestBody CreateUserRequest request);
}
```

```java
// Configuration
@Configuration
public class ApiClientConfig {

    @Bean
    public UserApiClient userApiClient(RestClient.Builder builder) {
        RestClient restClient = builder
            .baseUrl("http://localhost:8081")
            .build();

        return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(restClient))
            .build()
            .createClient(UserApiClient.class);
    }
}
```

```java
// Usage
@Service
public class UserSyncService {

    private final UserApiClient userApiClient;

    public UserSyncService(UserApiClient userApiClient) {
        this.userApiClient = userApiClient;
    }

    public void syncUsers() {
        List<UserDto> users = userApiClient.getUsers();
        // process users
    }
}
```

### 9. Health Checks (Actuator)

#### Custom Health Indicators
```java
// ✅ GOOD - Custom health indicator
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;

    public DatabaseHealthIndicator(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            return Health.up()
                .withDetail("database", "PostgreSQL")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// Access at: /actuator/health
```

#### Actuator Configuration
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized
```

### 10. Logging

#### Use SLF4J (with Logback)
```java
// ✅ GOOD - SLF4J logging
@Service
public class UserService {

    private static final Logger log = LoggerFactory.getLogger(UserService.class);

    public UserDto create(CreateUserRequest request) {
        log.info("Creating user: {}", request.email());

        try {
            UserEntity user = new UserEntity();
            user.setName(request.name());
            user.setEmail(request.email());
            repository.save(user);

            log.info("User created with id: {}", user.getId());
            return toDto(user);
        } catch (Exception e) {
            log.error("Failed to create user: {}", request.email(), e);
            throw e;
        }
    }
}
```

---

## Testing Best Practices

### 1. Unit Tests with Mockito
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository userRepository;

    @Mock
    EmailService emailService;

    @InjectMocks
    UserService userService;

    @Test
    void givenValidRequest_whenCreate_thenUserCreated() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest("John", "john@example.com");
        UserEntity user = new UserEntity();
        user.setId(1L);
        user.setName(request.name());

        when(userRepository.save(any(UserEntity.class))).thenReturn(user);

        // Act
        UserDto result = userService.create(request);

        // Assert
        assertThat(result).isNotNull();
        assertThat(result.name()).isEqualTo("John");
        verify(emailService).sendWelcomeEmail(any());
    }
}
```

### 2. Integration Tests with `@SpringBootTest`
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Autowired
    ObjectMapper objectMapper;

    @Test
    void testGetUsers() throws Exception {
        mockMvc.perform(get("/api/users"))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON));
    }

    @Test
    void testCreateUser() throws Exception {
        CreateUserRequest request = new CreateUserRequest("John", "john@example.com");

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

### 3. Slice Tests
```java
// ✅ GOOD - Test only the web layer
@WebMvcTest(UserController.class)
class UserControllerSliceTest {

    @Autowired
    MockMvc mockMvc;

    @MockitoBean  // Replaces deprecated @MockBean in Boot 4
    UserService userService;

    @Test
    void testGetById() throws Exception {
        when(userService.findById(1L))
            .thenReturn(Optional.of(new UserDto(1L, "John", "john@example.com")));

        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}

// ✅ GOOD - Test only the data layer
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    UserRepository userRepository;

    @Test
    void testFindByEmail() {
        UserEntity user = new UserEntity();
        user.setName("John");
        user.setEmail("john@example.com");
        userRepository.save(user);

        Optional<UserEntity> found = userRepository.findByEmail("john@example.com");
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John");
    }
}
```

---

## Performance Best Practices

### 1. Use Projections
```java
// ✅ GOOD - Interface-based projection for specific fields
public interface UserSummary {
    Long getId();
    String getName();
}

public interface UserRepository extends JpaRepository<UserEntity, Long> {

    List<UserSummary> findByActiveTrue();
}
```

```java
// ✅ GOOD - Record-based projection with @Query
public record UserSummaryDto(Long id, String name) {}

@Query("SELECT new com.example.dto.UserSummaryDto(u.id, u.name) FROM UserEntity u")
List<UserSummaryDto> findSummaries();
```

### 2. Use Pagination
```java
// ✅ GOOD - Paginated results
@GetMapping
public Page<UserDto> list(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size) {

    return repository.findAll(PageRequest.of(page, size))
        .map(this::toDto);
}
```

### 3. Virtual Threads (Enabled by Default in Boot 4)
```yaml
# Virtual threads are enabled by default in Spring Boot 4
# To explicitly configure:
spring:
  threads:
    virtual:
      enabled: true  # default in Boot 4
```

### 4. Scoped Values over ThreadLocal
```java
// ✅ GOOD - Use ScopedValue (Java 25, finalized) instead of ThreadLocal
// Especially important with virtual threads where ThreadLocal is expensive
private static final ScopedValue<RequestContext> REQUEST_CTX = ScopedValue.newInstance();

public void handle(RequestContext ctx) {
    ScopedValue.runWhere(REQUEST_CTX, ctx, () -> {
        service.process();  // ctx available via REQUEST_CTX.get()
    });
}

// ❌ BAD - ThreadLocal (expensive with virtual threads, risk of memory leaks)
private static final ThreadLocal<RequestContext> REQUEST_CTX = new ThreadLocal<>();
```

### 5. Compact Object Headers (Java 25)
```bash
# Reduce memory footprint — object headers shrink from 96-128 bits to 64 bits
java -XX:+UseCompactObjectHeaders -jar myapp.jar
```

---

## Spring Boot 4 Migration Notes

### Key Changes from Spring Boot 3.x
- **Jakarta EE 11** — Servlet 6.1, JPA 3.2, Validation 3.1
- **Spring Framework 7** — new API versioning, resilience annotations, JSpecify null safety
- **Jackson 3.x** — breaking changes in serialization, package renames
- **JUnit Jupiter 6** — JUnit 4 support fully removed
- **`@MockBean` → `@MockitoBean`** — deprecated annotation replaced
- **Public field binding removed** — `@ConfigurationProperties` requires getters/setters
- **Undertow dropped** — only Tomcat 11+ and Jetty 12.1+ supported
- **Virtual threads by default** — web thread pool uses virtual threads

### Java 25 Advantages over Java 21
- **Scoped Values (finalized)** — safer alternative to ThreadLocal, especially with virtual threads
- **Stable Values (preview)** — lazy initialization optimized by the JVM
- **Flexible Constructor Bodies** — validation before `super()` calls
- **Module Import Declarations** — cleaner imports with `import module`
- **Compact Object Headers** — reduced memory footprint (product feature)
- **Primitive Pattern Matching** — `instanceof` and `switch` work with all primitive types
- **Compact Source Files** — instance `main()` methods for simpler entry points
- **AOT Profiling (JEP 515)** — method-level profiling for ahead-of-time optimization
- **Generational Shenandoah GC** — improved low-latency garbage collection

---

## Do's and Don'ts

### ✅ DO:
- Use Java 25 records for DTOs
- Use pattern matching in switch statements (including primitives)
- Use text blocks for multi-line strings
- Use flexible constructor bodies for validation before `super()`
- Use `ScopedValue` instead of `ThreadLocal` (especially with virtual threads)
- Use constructor injection (single constructor = no `@Autowired` needed)
- Use Spring Data JPA repositories
- Use `@Transactional` for write operations, `@Transactional(readOnly = true)` for reads
- Use `@ConfigurationProperties` for type-safe configuration
- Use declarative HTTP interface clients (`@GetExchange`, `@PostExchange`)
- Use `@RestControllerAdvice` for global exception handling
- Implement health indicators with Actuator
- Use SLF4J for logging
- Use `@MockitoBean` (not `@MockBean`) in tests
- Enable compact object headers for reduced memory (`-XX:+UseCompactObjectHeaders`)

### ❌ DON'T:
- Use traditional getter/setter classes for simple DTOs (use records)
- Use field injection (prefer constructor injection)
- Use `ThreadLocal` when `ScopedValue` works (finalized in Java 25)
- Use raw JDBC when Spring Data is available
- Forget `@Transactional` on write operations
- Hardcode configuration values
- Mix business logic in controllers
- Forget validation on inputs with `@Valid`
- Ignore exception handling
- Use `System.out.println` (use Logger)
- Use `javax.*` imports (must be `jakarta.*`)
- Use `@MockBean` / `@SpyBean` (removed in Boot 4)
- Rely on public field binding in `@ConfigurationProperties`

---

## Quick Command Reference

### Development Mode (with DevTools)
```bash
./mvnw spring-boot:run
```

### Build
```bash
./mvnw clean package
```

### Run Tests
```bash
./mvnw test
```

### Build Native Image (GraalVM)
```bash
./mvnw -Pnative native:compile
```

### Run as JAR
```bash
java -jar target/myapp-0.0.1-SNAPSHOT.jar
```

### Run with Profile and Java 25 Optimizations
```bash
java -XX:+UseCompactObjectHeaders \
     --spring.profiles.active=dev \
     -jar target/myapp.jar
```

### Enable Preview Features (for Stable Values, etc.)
```bash
java --enable-preview -jar target/myapp.jar
```

---

## Common Dependencies

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.5</version>
</parent>

<properties>
    <java.version>25</java.version>
</properties>

<dependencies>
    <!-- Web (includes Tomcat, Jackson 3) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Data JPA (includes Hibernate, Spring Data) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- PostgreSQL Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Validation (Jakarta Validation 3.1) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Actuator (Health & Metrics) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Prometheus Metrics -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- DevTools (live reload in dev) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>

    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## Summary Checklist

- [ ] Java 25 (LTS) installed and configured
- [ ] Spring Boot 4.0.x with `spring-boot-starter-parent`
- [ ] `<java.version>25</java.version>` in pom.xml
- [ ] Use records for DTOs
- [ ] Use pattern matching (including primitives) where applicable
- [ ] Use text blocks for multi-line strings
- [ ] Use flexible constructor bodies for pre-super validation
- [ ] Use `ScopedValue` instead of `ThreadLocal`
- [ ] Constructor injection for dependencies
- [ ] Spring Data JPA for database operations
- [ ] `@Transactional` on write operations
- [ ] Bean validation on inputs with `@Valid`
- [ ] Global exception handling with `@RestControllerAdvice`
- [ ] Declarative HTTP clients for external APIs
- [ ] Health indicators implemented with Actuator
- [ ] Proper logging with SLF4J
- [ ] Tests with `@SpringBootTest` and slice tests
- [ ] Configuration externalized with `@ConfigurationProperties`
- [ ] All imports use `jakarta.*` (not `javax.*`)
- [ ] Using `@MockitoBean` instead of removed `@MockBean`
- [ ] Compact object headers enabled for production

---

## Evaluating New Libraries & Approaches

When considering a new library, dependency, or implementation approach, follow this checklist before adopting:

### Compatibility Check
- Verify compatibility with Spring Boot 4.0.x, Spring Framework 7, Java 25, and Jakarta EE 11
- Check that the library uses `jakarta.*` (not `javax.*`)
- Confirm Jackson 3.x compatibility if JSON serialization is involved
- Test with JUnit Jupiter 6 (JUnit 4 is removed)

### Library Health Check
- **Last release date** — avoid unmaintained libraries (no release in 12+ months)
- **Open issue count** — high count with no triage is a red flag
- **Contributor activity** — single-maintainer projects carry bus-factor risk
- **Breaking changes history** — frequent breaking changes signal instability

### Decision Framework
When comparing approaches, evaluate each option on:
- **Repo fit** (High/Medium/Low) — does it match existing patterns in the codebase?
- **Estimate** (S/M/L) — implementation effort
- **Confidence** (High/Medium/Low) — how certain are you this will work?
- **Risks** — what could go wrong?

### Research Storage
Save all research to `.claude/research/YYYY-MM-DD-<feature-name>/research-<feature-name>.md` using the canonical layout defined in the agent file. This creates a decision log that future developers can reference.

### Cross-Reference
For the full evaluate-options workflow, structured templates, and response format, see `springboot-dev-agent.md`.
