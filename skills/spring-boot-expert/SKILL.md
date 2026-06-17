---
name: spring-boot-expert
description: Expert Spring Boot 3.x guidance — auto-configuration, Spring Data, Security, WebFlux, Actuator, testing, performance, and production pitfalls
---

# Spring Boot Expert

You are an expert in Spring Boot 3.x and Spring Framework 6. Apply production-hardened patterns, prefer explicit configuration over magic, and always explain the "why" behind recommendations.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Spring Boot Application                   │
│                                                             │
│  ┌─────────────┐   ┌──────────────┐   ┌─────────────────┐  │
│  │  Web Layer  │   │ Service Layer│   │  Data Layer     │  │
│  │             │   │              │   │                 │  │
│  │ @Controller │──>│  @Service    │──>│ @Repository     │  │
│  │ @RestCtrl   │   │  @Transact.  │   │ Spring Data JPA │  │
│  │ WebFlux     │   │  @Cacheable  │   │ QueryDSL        │  │
│  └─────────────┘   └──────────────┘   └─────────────────┘  │
│         │                                      │            │
│  ┌─────────────┐                      ┌────────────────┐   │
│  │  Security   │                      │  Database      │   │
│  │  Filter     │                      │  HikariCP Pool │   │
│  │  Chain      │                      │  PostgreSQL     │   │
│  └─────────────┘                      └────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Spring Boot Actuator                    │  │
│  │  /health  /metrics  /info  /prometheus  /env        │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. Spring Boot 3 + Spring Framework 6

### AOT (Ahead-of-Time) Processing

Spring Boot 3 introduces build-time AOT to generate reflection hints, proxy classes, and bean definitions at compile time instead of runtime.

```java
// Build-time AOT: Spring generates BeanDefinition metadata
// Run-time: no classpath scanning, no reflection on startup

// Enable AOT hints for custom reflection needs
@ImportRuntimeHints(MyRuntimeHints.class)
@Configuration
public class MyConfig { }

public class MyRuntimeHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.reflection()
            .registerType(MyDomainClass.class,
                MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
                MemberCategory.INVOKE_DECLARED_METHODS);
        hints.resources().registerPattern("templates/*.html");
    }
}
```

### GraalVM Native Image

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>

<!-- Build native image -->
<!-- mvn -Pnative native:compile -->
```

```bash
# Build native image (requires GraalVM JDK)
./mvnw -Pnative native:compile

# Run — startup in milliseconds, low memory footprint
./target/my-application

# Test native image
./mvnw -PnativeTest test
```

**When to use native image:**
- Serverless functions (Lambda, Cloud Run) — cold start matters
- CLI tools bundled as executables
- High-density container deployments

**When to avoid:**
- Applications with heavy runtime reflection (some JPA providers have issues)
- Build pipeline cannot accommodate long native compile times (5–20 min)
- Dynamic class loading required

---

## 2. Auto-Configuration

### How It Works

```
Spring Boot Startup
       │
       ▼
@SpringBootApplication
       │
       ├── @ComponentScan  → scans your packages
       ├── @EnableAutoConfig → loads META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
       │
       ▼
AutoConfigurationImportSelector
       │
       ▼
For each AutoConfiguration class:
  evaluate @ConditionalOn* annotations
       │
  ┌────┴────┐
  │ passes? │
  └────┬────┘
    YES│              NO
       ▼               ▼
  Register beans    Skip entirely
```

### @Conditional Annotations

```java
@Configuration
@ConditionalOnClass(DataSource.class)           // class must be on classpath
@ConditionalOnMissingBean(DataSource.class)     // no existing DataSource bean
@ConditionalOnProperty(
    name = "app.datasource.enabled",
    havingValue = "true",
    matchIfMissing = true                        // default to enabled
)
@ConditionalOnWebApplication(type = Type.SERVLET)
public class MyDataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}
```

### Custom Starter Creation

```
my-starter/
├── my-starter-autoconfigure/
│   ├── src/main/java/com/example/
│   │   ├── MyAutoConfiguration.java
│   │   └── MyProperties.java
│   └── src/main/resources/META-INF/spring/
│       └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
└── my-starter/
    └── pom.xml  (depends on autoconfigure + optional transitive deps)
```

```java
// MyProperties.java
@ConfigurationProperties(prefix = "my.service")
public class MyProperties {
    private String apiUrl = "https://api.example.com";
    private Duration timeout = Duration.ofSeconds(30);
    private int maxRetries = 3;
    // getters/setters
}

// MyAutoConfiguration.java
@AutoConfiguration
@EnableConfigurationProperties(MyProperties.class)
@ConditionalOnClass(MyServiceClient.class)
@ConditionalOnMissingBean(MyServiceClient.class)
public class MyAutoConfiguration {

    @Bean
    public MyServiceClient myServiceClient(MyProperties props) {
        return new MyServiceClient(props.getApiUrl(), props.getTimeout());
    }
}
```

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.MyAutoConfiguration
```

---

## 3. Spring Data

### Repository Patterns

```java
// Layered approach — choose the right abstraction
public interface UserRepository extends JpaRepository<User, Long>,
                                        JpaSpecificationExecutor<User> {

    // --- Derived queries (generated from method name) ---
    List<User> findByEmailAndActiveTrue(String email);
    Optional<User> findByExternalId(UUID externalId);
    long countByCreatedAtAfter(Instant since);

    // --- @Query for complex JPQL ---
    @Query("""
        SELECT u FROM User u
        LEFT JOIN FETCH u.roles r
        WHERE u.tenantId = :tenantId
          AND u.active = true
        ORDER BY u.lastName, u.firstName
        """)
    List<User> findActiveUsersWithRoles(@Param("tenantId") Long tenantId);

    // --- Native query when JPQL won't cut it ---
    @Query(value = """
        SELECT * FROM users u
        WHERE u.search_vector @@ to_tsquery('english', :query)
        ORDER BY ts_rank(u.search_vector, to_tsquery('english', :query)) DESC
        LIMIT :limit
        """, nativeQuery = true)
    List<User> fullTextSearch(@Param("query") String query, @Param("limit") int limit);

    // --- Pagination ---
    Page<User> findByTenantId(Long tenantId, Pageable pageable);

    // --- Modifying queries ---
    @Modifying
    @Query("UPDATE User u SET u.lastLoginAt = :now WHERE u.id = :id")
    int updateLastLogin(@Param("id") Long id, @Param("now") Instant now);

    // --- Exists check (cheaper than count) ---
    boolean existsByEmail(String email);
}
```

### Projections — avoid loading full entities when you don't need them

```java
// Interface projection (Spring Data generates proxy)
public interface UserSummary {
    Long getId();
    String getFirstName();
    String getLastName();
    String getEmail();

    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
}

// Class-based (DTO) projection — better performance, immutable
public record UserDto(Long id, String firstName, String lastName, String email) {}

// Usage
@Query("SELECT new com.example.UserDto(u.id, u.firstName, u.lastName, u.email) FROM User u WHERE u.active = true")
List<UserDto> findActiveSummaries();

// Or interface projection
List<UserSummary> findByTenantId(Long tenantId);
```

### Auditing

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig {
    @Bean
    public AuditorAware<Long> auditorAware() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(auth -> ((UserDetails) auth.getPrincipal()).getId());
    }
}

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private Long createdBy;

    @LastModifiedBy
    private Long updatedBy;

    @Version
    private Long version;  // optimistic locking
}
```

---

## 4. Spring Security

### SecurityFilterChain (preferred over WebSecurityConfigurerAdapter, which is removed in Spring Security 6)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http,
                                           JwtAuthFilter jwtAuthFilter) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)  // stateless API — no CSRF needed
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/public/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
                .accessDeniedHandler(new AccessDeniedHandlerImpl())
            )
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

### JWT Authentication Filter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7);
        String username = jwtService.extractUsername(token);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails user = userDetailsService.loadUserByUsername(username);
            if (jwtService.isTokenValid(token, user)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        chain.doFilter(request, response);
    }
}
```

### Method Security

```java
@Service
public class DocumentService {

    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public List<Document> getUserDocuments(Long userId) { ... }

    @PreAuthorize("hasPermission(#doc, 'WRITE')")
    public Document updateDocument(Document doc) { ... }

    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    public Document getDocument(Long id) { ... }

    @PreFilter("filterObject.ownerId == authentication.principal.id")
    public List<Document> bulkDelete(List<Document> docs) { ... }
}
```

---

## 5. Spring WebFlux (Reactive)

### When to use WebFlux vs Spring MVC

```
Use WebFlux when:                        Use Spring MVC when:
- High concurrency, low CPU work         - CRUD apps with blocking I/O
- Streaming data to clients              - Team not familiar with reactive
- Non-blocking I/O (MongoDB, Redis)      - RDBMS as primary store (JPA is blocking)
- Backpressure control needed            - Simpler mental model needed
```

### Mono/Flux Patterns

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserRepository userRepo;  // Reactive repository
    private final WebClient externalClient;

    // Single item
    @GetMapping("/{id}")
    public Mono<UserDto> getUser(@PathVariable Long id) {
        return userRepo.findById(id)
            .map(UserMapper::toDto)
            .switchIfEmpty(Mono.error(new ResourceNotFoundException("User", id)));
    }

    // Stream of items
    @GetMapping
    public Flux<UserDto> listUsers(@RequestParam(defaultValue = "0") int page,
                                   @RequestParam(defaultValue = "20") int size) {
        return userRepo.findAll()
            .skip((long) page * size)
            .take(size)
            .map(UserMapper::toDto);
    }

    // Server-Sent Events streaming
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<UserEvent> streamUsers() {
        return userRepo.findAll()
            .map(u -> new UserEvent(u.getId(), u.getEmail()))
            .delayElements(Duration.ofMillis(100));
    }

    // Combining reactive sources
    @GetMapping("/{id}/with-profile")
    public Mono<UserWithProfile> getUserWithProfile(@PathVariable Long id) {
        Mono<User> userMono = userRepo.findById(id)
            .switchIfEmpty(Mono.error(new ResourceNotFoundException("User", id)));

        Mono<Profile> profileMono = externalClient.get()
            .uri("/profiles/{id}", id)
            .retrieve()
            .bodyToMono(Profile.class)
            .onErrorReturn(Profile.empty());  // graceful fallback

        return Mono.zip(userMono, profileMono)
            .map(tuple -> UserWithProfile.of(tuple.getT1(), tuple.getT2()));
    }
}
```

### WebClient (use instead of RestTemplate)

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient paymentServiceClient(WebClient.Builder builder) {
        return builder
            .baseUrl("https://payment-service.internal")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .filter(ExchangeFilterFunctions.basicAuthentication("svc", "secret"))
            .filter(logRequest())
            .codecs(c -> c.defaultCodecs().maxInMemorySize(2 * 1024 * 1024))
            .build();
    }
}

// Usage
public Mono<PaymentResult> chargeCard(ChargeRequest request) {
    return paymentClient.post()
        .uri("/charges")
        .bodyValue(request)
        .retrieve()
        .onStatus(HttpStatus::is4xxClientError, resp ->
            resp.bodyToMono(ErrorResponse.class)
                .flatMap(err -> Mono.error(new PaymentException(err.getMessage()))))
        .onStatus(HttpStatus::is5xxServerError, resp ->
            Mono.error(new PaymentServiceUnavailableException()))
        .bodyToMono(PaymentResult.class)
        .timeout(Duration.ofSeconds(10))
        .retryWhen(Retry.backoff(3, Duration.ofMillis(500))
            .filter(ex -> ex instanceof PaymentServiceUnavailableException));
}
```

---

## 6. Actuator

### Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      show-components: always
      probes:
        enabled: true   # /health/liveness and /health/readiness for Kubernetes
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.9, 0.95, 0.99
  prometheus:
    metrics:
      export:
        enabled: true
```

### Custom Health Indicator

```java
@Component
public class ExternalServiceHealthIndicator implements HealthIndicator {

    private final WebClient externalClient;

    @Override
    public Health health() {
        try {
            externalClient.get()
                .uri("/ping")
                .retrieve()
                .toBodilessEntity()
                .block(Duration.ofSeconds(3));
            return Health.up()
                .withDetail("service", "external-api")
                .withDetail("status", "reachable")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("service", "external-api")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### Custom Actuator Endpoint

```java
@Component
@Endpoint(id = "featureflags")
public class FeatureFlagsEndpoint {

    private final FeatureFlagService flagService;

    @ReadOperation
    public Map<String, Boolean> getFlags() {
        return flagService.getAllFlags();
    }

    @WriteOperation
    public void setFlag(@Selector String flagName, boolean enabled) {
        flagService.setFlag(flagName, enabled);
    }
}
// Available at: GET /actuator/featureflags
//               POST /actuator/featureflags/{flagName}
```

---

## 7. Testing

### Test Slice Overview

```
@SpringBootTest         → full application context (slow, for integration tests)
@WebMvcTest             → web layer only (Controller + Security)
@WebFluxTest            → reactive web layer
@DataJpaTest            → JPA layer only (in-memory H2 or TestContainers)
@DataMongoTest          → MongoDB layer only
@JsonTest               → JSON serialization only
@MockBean               → replaces bean in context with Mockito mock
```

### @WebMvcTest

```java
@WebMvcTest(UserController.class)
@Import(SecurityConfig.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    @WithMockUser(roles = "ADMIN")
    void getUser_whenExists_returnsUser() throws Exception {
        var user = new UserDto(1L, "John", "Doe", "john@example.com");
        given(userService.findById(1L)).willReturn(Optional.of(user));

        mockMvc.perform(get("/api/v1/users/1")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1L))
            .andExpect(jsonPath("$.firstName").value("John"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
    }

    @Test
    void getUser_whenUnauthenticated_returns401() throws Exception {
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isUnauthorized());
    }
}
```

### @DataJpaTest with TestContainers

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)  // use real DB
@Testcontainers
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Test
    @Transactional
    void findActiveUsersWithRoles_returnsOnlyActiveUsers() {
        // arrange
        var activeUser = userRepository.save(User.builder()
            .email("active@test.com")
            .active(true)
            .tenantId(1L)
            .build());
        userRepository.save(User.builder()
            .email("inactive@test.com")
            .active(false)
            .tenantId(1L)
            .build());

        // act
        var results = userRepository.findActiveUsersWithRoles(1L);

        // assert
        assertThat(results).hasSize(1);
        assertThat(results.get(0).getEmail()).isEqualTo("active@test.com");
    }
}
```

### @MockBean Pitfalls

```
PITFALL: @MockBean causes Spring context to reload between test classes.
SOLUTION: Share context with @TestConfiguration + cache.

PITFALL: @MockBean in @SpringBootTest resets all mocks per test — verify() may fail
         if the bean was called during application startup.
SOLUTION: Use Mockito.reset() in @BeforeEach or use @SpyBean carefully.

PITFALL: @MockBean mocks the Spring proxy, not the underlying class.
         verify() on the mock works, but verify() with argument captors on
         @Transactional methods may not capture internal calls.
```

---

## 8. Configuration

### @ConfigurationProperties (prefer over @Value)

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public class PaymentProperties {

    @NotNull
    private String apiKey;

    @NotBlank
    private String baseUrl;

    @DurationUnit(ChronoUnit.SECONDS)
    @Positive
    private Duration timeout = Duration.ofSeconds(30);

    @Min(1) @Max(10)
    private int maxRetries = 3;

    private Map<String, String> headers = new HashMap<>();

    // nested config
    private Retry retry = new Retry();

    public record Retry(
        @Min(100) long initialDelayMs,
        double multiplier,
        @Max(30000) long maxDelayMs
    ) {
        public Retry() { this(500, 2.0, 10000); }
    }
    // getters/setters or record
}
```

```yaml
# application.yaml
app:
  payment:
    api-key: ${PAYMENT_API_KEY}   # always externalize secrets
    base-url: https://api.payment.com
    timeout: 30s
    max-retries: 3
    retry:
      initial-delay-ms: 500
      multiplier: 2.0
      max-delay-ms: 10000
```

### Profiles and Externalized Config

```
Priority (highest → lowest):
1. Command-line args (--server.port=8081)
2. SPRING_APPLICATION_JSON env var
3. OS environment variables
4. application-{profile}.yaml (inside jar)
5. application.yaml (inside jar)
6. @PropertySource annotations
7. Default values in @ConfigurationProperties
```

```bash
# Run with profile
java -jar app.jar --spring.profiles.active=prod

# Override specific properties
java -jar app.jar --app.payment.timeout=60s

# Kubernetes: inject secrets as env vars
# SPRING_DATASOURCE_PASSWORD=secret (maps to spring.datasource.password)
```

---

## 9. Performance

### HikariCP Tuning

```yaml
spring:
  datasource:
    hikari:
      pool-name: MainPool
      maximum-pool-size: 10          # start here; tune based on DB connections available
      minimum-idle: 5
      idle-timeout: 600000           # 10 minutes
      connection-timeout: 30000      # 30 seconds — fail fast
      max-lifetime: 1800000          # 30 minutes — rotate before DB kills it
      keepalive-time: 60000          # 1 minute — prevent firewall idle disconnects
      connection-test-query: SELECT 1 # only for drivers without isValid()
      leak-detection-threshold: 60000 # log if connection held > 60s (debugging)
```

**Sizing rule of thumb:** `pool_size = (core_count * 2) + effective_spindle_count`
For PostgreSQL: usually 10–20 connections per application instance.

### Caching

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("users", config.entryTtl(Duration.ofMinutes(30)))
            .withCacheConfiguration("products", config.entryTtl(Duration.ofHours(1)))
            .build();
    }
}

@Service
public class UserService {

    @Cacheable(value = "users", key = "#id", unless = "#result == null")
    public UserDto findById(Long id) { ... }

    @CachePut(value = "users", key = "#result.id")
    public UserDto update(UpdateUserRequest request) { ... }

    @CacheEvict(value = "users", key = "#id")
    public void delete(Long id) { ... }

    @CacheEvict(value = "users", allEntries = true)
    @Scheduled(cron = "0 0 * * * *")  // hourly full eviction
    public void evictAll() { }
}
```

### @Async

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            log.error("Async exception in {}: {}", method.getName(), ex.getMessage(), ex);
    }
}

@Service
public class EmailService {

    @Async
    public CompletableFuture<Void> sendWelcomeEmail(User user) {
        // runs in thread pool, doesn't block caller
        emailProvider.send(user.getEmail(), "Welcome!", buildWelcomeTemplate(user));
        return CompletableFuture.completedFuture(null);
    }
}
```

---

## 10. Common Pitfalls

### N+1 Query Problem

```java
// BAD: triggers N+1 if roles are LAZY
List<User> users = userRepo.findAll();
users.forEach(u -> u.getRoles().size()); // N additional queries

// GOOD: fetch join in one query
@Query("SELECT DISTINCT u FROM User u LEFT JOIN FETCH u.roles WHERE u.active = true")
List<User> findAllWithRoles();

// GOOD: EntityGraph for dynamic control
@EntityGraph(attributePaths = {"roles", "permissions"})
List<User> findByTenantId(Long tenantId);

// GOOD: Use Hibernate's BatchSize for collections
@BatchSize(size = 25)
@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
private Set<Role> roles;
```

### OpenEntityManagerInView (OSIV) Filter

```yaml
# DISABLE in production — it keeps DB connection open for full HTTP request duration
spring:
  jpa:
    open-in-view: false  # default is true — this silently causes connection pool exhaustion
```

```
OSIV enabled (default):  DB connection held from Controller entry to response write
                          → connection pool exhaustion under load
OSIV disabled:           DB connection released at end of @Transactional method
                          → LazyInitializationException if you access lazy
                            collections outside transaction (fix: use projections)
```

### Circular Dependencies

```
SYMPTOM: BeanCurrentlyInCreationException
CAUSE:   A → B → A injection cycle

SOLUTIONS (in order of preference):
1. Redesign — extract shared logic to a third component C
2. Use @Lazy on one injection point
3. Inject ApplicationContext and get bean lazily (anti-pattern but acceptable for infra)

NEVER use: field injection (@Autowired on fields) — it hides circular deps
           and makes testing harder
```

### Over-injection

```java
// BAD: constructor with 8+ dependencies = design smell
@Service
@RequiredArgsConstructor
public class OrderService {
    private final UserRepo userRepo;
    private final ProductRepo productRepo;
    private final InventoryService inventoryService;
    private final PricingService pricingService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    private final AuditService auditService;
    private final ShippingService shippingService;  // too many!
}

// GOOD: decompose into smaller focused services
// OrderCreationService, OrderFulfillmentService, OrderNotificationService
```

---

## Checklist

### New Service

- [ ] `spring.jpa.open-in-view=false` in application.yaml
- [ ] HikariCP `maximum-pool-size` tuned to DB limits
- [ ] `@ConfigurationProperties` for all config, `@Validated` with constraints
- [ ] Secrets injected via environment variables, not hardcoded
- [ ] `@SpringBootTest` integration tests with TestContainers (real DB, not H2)
- [ ] Actuator health probes configured for Kubernetes liveness/readiness
- [ ] Prometheus metrics endpoint exposed and scraped
- [ ] WebClient used instead of RestTemplate
- [ ] `@Transactional(readOnly = true)` on read-only service methods
- [ ] Pagination on all list endpoints (never return unbounded lists)
- [ ] `@EntityGraph` or fetch join to avoid N+1 queries

### Security Review

- [ ] CSRF disabled only for stateless APIs
- [ ] `SessionCreationPolicy.STATELESS` for JWT-based auth
- [ ] Sensitive actuator endpoints require authentication
- [ ] Passwords encoded with BCrypt strength >= 10
- [ ] JWT tokens have expiry and refresh mechanism
- [ ] Method-level security on all sensitive operations

### Performance Review

- [ ] OSIV disabled
- [ ] Caching applied to stable, expensive reads
- [ ] Async used for fire-and-forget operations (email, audit logs)
- [ ] Database indexes verified with `EXPLAIN ANALYZE`
- [ ] Connection pool size profiled under expected load
