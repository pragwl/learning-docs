Here is a detailed explanation of each annotation you mentioned, with examples relevant to Spring and Spring Boot applications. This should cover the purpose, usage, and implications of each annotation, ensuring that they work seamlessly in your projects.

### Core Spring & Spring Boot Annotations

#### `@SpringBootApplication`
- **Purpose**: Marks the main class of a Spring Boot application. It is a combination of `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`.
- **Example**:
  ```java
  @SpringBootApplication
  public class MyApp {
      public static void main(String[] args) {
          SpringApplication.run(MyApp.class, args);
      }
  }
  ```

#### `@EnableAutoConfiguration`
- **Purpose**: Enables Spring Boot’s auto-configuration mechanism, which automatically configures your Spring application based on the dependencies you have.
- **Example**:
  ```java
  @SpringBootApplication
  @EnableAutoConfiguration
  public class MyApp {
      // Main method
  }
  ```

#### `@ComponentScan`
- **Purpose**: Tells Spring to scan the specified packages for annotated components (like `@Component`, `@Service`, `@Repository`, `@Controller`).
- **Example**:
  ```java
  @ComponentScan(basePackages = "com.example")
  public class MyAppConfig {
      // Configuration methods
  }
  ```

### Spring Context Annotations

#### `@Component`
- **Purpose**: Indicates that a class is a Spring component, making it eligible for component scanning and dependency injection.
- **Example**:
  ```java
  @Component
  public class MyComponent {
      // Business logic
  }
  ```

#### `@Service`
- **Purpose**: Specialization of `@Component` to indicate that a class is a service layer component.
- **Example**:
  ```java
  @Service
  public class MyService {
      // Service methods
  }
  ```

#### `@Repository`
- **Purpose**: Indicates that a class is a Data Access Object (DAO) and provides exception translation.
- **Example**:
  ```java
  @Repository
  public class MyRepository {
      // Data access methods
  }
  ```

#### `@Controller`
- **Purpose**: Marks a class as a Spring MVC controller.
- **Example**:
  ```java
  @Controller
  public class MyController {
      // Handler methods
  }
  ```

#### `@RestController`
- **Purpose**: Combines `@Controller` and `@ResponseBody`, making it suitable for RESTful web services.
- **Example**:
  ```java
  @RestController
  public class MyRestController {
      @GetMapping("/hello")
      public String sayHello() {
          return "Hello, World!";
      }
  }
  ```

#### `@Configuration`
- **Purpose**: Indicates that a class declares one or more `@Bean` methods and is used by Spring IoC container as a source of bean definitions.
- **Example**:
  ```java
  @Configuration
  public class MyAppConfig {
      @Bean
      public MyService myService() {
          return new MyService();
      }
  }
  ```

#### `@Bean`
- **Purpose**: Marks a method as a bean producer. The returned object will be registered as a Spring bean.
- **Example**:
  ```java
  @Configuration
  public class MyAppConfig {
      @Bean
      public MyService myService() {
          return new MyService();
      }
  }
  ```

#### `@Autowired`
- **Purpose**: Enables automatic injection of a dependency by type.
- **Example**:
  ```java
  @Service
  public class MyService {
      @Autowired
      private MyRepository myRepository;
  }
  ```

#### `@Qualifier`
- **Purpose**: Used in conjunction with `@Autowired` to resolve the ambiguity when multiple beans of the same type exist.
- **Example**:
  ```java
  @Service
  public class MyService {
      @Autowired
      @Qualifier("mySpecificRepository")
      private MyRepository myRepository;
  }
  ```

#### `@Primary`
- **Purpose**: Indicates that a bean should be given preference when multiple candidates are qualified to be autowired.
- **Example**:
  ```java
  @Service
  @Primary
  public class PrimaryService implements MyService {
      // Implementation
  }
  ```

#### `@Lazy`
- **Purpose**: Marks a bean to be initialized lazily, i.e., only when it is first needed.
- **Example**:
  ```java
  @Service
  @Lazy
  public class LazyService {
      // Implementation
  }
  ```

#### `@Value`
- **Purpose**: Injects a property or expression into a bean.
- **Example**:
  ```java
  @Component
  public class MyComponent {
      @Value("${my.property}")
      private String myProperty;
  }
  ```

#### `@PropertySource`
- **Purpose**: Specifies the properties file to be loaded into the Spring environment.
- **Example**:
  ```java
  @Configuration
  @PropertySource("classpath:application.properties")
  public class AppConfig {
      // Configuration methods
  }
  ```

#### `@Scope`
- **Purpose**: Defines the scope of a bean (`singleton`, `prototype`, etc.).
- **Example**:
  ```java
  @Service
  @Scope("prototype")
  public class PrototypeService {
      // Implementation
  }
  ```

#### `@Profile`
- **Purpose**: Specifies the profile(s) in which a bean should be active.
- **Example**:
  ```java
  @Service
  @Profile("dev")
  public class DevService {
      // Development-specific implementation
  }
  ```

#### `@DependsOn`
- **Purpose**: Indicates that a bean should be initialized after the specified beans.
- **Example**:
  ```java
  @Service
  @DependsOn("anotherBean")
  public class DependentService {
      // Implementation
  }
  ```

### Spring Boot Testing Annotations

#### `@SpringBootTest`
- **Purpose**: Used for integration tests to load the full Spring application context.
- **Example**:
  ```java
  @SpringBootTest
  public class MyAppTests {
      @Test
      public void contextLoads() {
      }
  }
  ```

#### `@MockBean`
- **Purpose**: Used to mock a bean within the Spring application context during tests.
- **Example**:
  ```java
  @SpringBootTest
  public class MyServiceTests {
      @MockBean
      private MyRepository myRepository;

      @Autowired
      private MyService myService;
  }
  ```

#### `@SpyBean`
- **Purpose**: Used to spy on a real bean in the Spring context, allowing you to stub methods.
- **Example**:
  ```java
  @SpringBootTest
  public class MyServiceTests {
      @SpyBean
      private MyService myService;
  }
  ```

#### `@WebMvcTest`
- **Purpose**: Focuses only on Spring MVC components, providing a Spring context with only MVC-related beans.
- **Example**:
  ```java
  @WebMvcTest(MyController.class)
  public class MyControllerTests {
      @Autowired
      private MockMvc mockMvc;
  }
  ```

#### `@DataJpaTest`
- **Purpose**: Configures an in-memory database, scans for `@Entity` classes, and configures Spring Data JPA repositories.
- **Example**:
  ```java
  @DataJpaTest
  public class MyRepositoryTests {
      @Autowired
      private MyRepository myRepository;
  }
  ```

#### `@ContextConfiguration`
- **Purpose**: Loads an ApplicationContext for tests with specific configurations.
- **Example**:
  ```java
  @ContextConfiguration(classes = MyAppConfig.class)
  public class MyTests {
      @Autowired
      private MyService myService;
  }
  ```

#### `@TestPropertySource`
- **Purpose**: Specifies a properties file or inlined properties to be added to the Spring Environment for tests.
- **Example**:
  ```java
  @TestPropertySource(locations = "classpath:test.properties")
  public class MyTests {
      @Value("${test.property}")
      private String testProperty;
  }
  ```

### Spring Security Annotations

#### `@EnableWebSecurity`
- **Purpose**: Enables Spring Security’s web security support.
- **Example**:
  ```java
  @EnableWebSecurity
  public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
      @Override
      protected void configure(HttpSecurity http) throws Exception {
          http.authorizeRequests().anyRequest().authenticated();
      }
  }
  ```

#### `@EnableGlobalMethodSecurity`
- **Purpose**: Enables method-level security with annotations like `@Secured`, `@PreAuthorize`, and `@PostAuthorize`.
- **Example**:
  ```java
  @EnableGlobalMethodSecurity(prePostEnabled = true)
  public class MethodSecurityConfig {
      // Configuration methods
  }
  ```

#### `@Secured`
- **Purpose**: Secures methods with roles.
- **Example**:
  ```java
  @Service
  public class MyService {
      @Secured("ROLE_ADMIN")
      public void adminMethod() {
      }
  }
  ```

#### `@PreAuthorize`
- **Purpose**: Secures methods with expressions, allowing for complex security checks.
- **Example

**:
  ```java
  @Service
  public class MyService {
      @PreAuthorize("hasRole('ROLE_USER')")
      public void userMethod() {
      }
  }
  ```

#### `@PostAuthorize`
- **Purpose**: Secures methods with expressions after the method execution.
- **Example**:
  ```java
  @Service
  public class MyService {
      @PostAuthorize("returnObject.owner == authentication.name")
      public MyObject findObject() {
          // Logic
      }
  }
  ```

#### `@RolesAllowed`
- **Purpose**: Specifies the roles that are allowed to invoke a method.
- **Example**:
  ```java
  @Service
  public class MyService {
      @RolesAllowed({"ROLE_USER", "ROLE_ADMIN"})
      public void roleMethod() {
      }
  }
  ```

#### `@WithMockUser`
- **Purpose**: Mocks a user in Spring Security tests.
- **Example**:
  ```java
  @Test
  @WithMockUser(username = "testUser", roles = {"USER"})
  public void testMethod() {
      // Test logic
  }
  ```

### Spring AOP Annotations

#### `@Aspect`
- **Purpose**: Indicates that a class is an aspect, defining cross-cutting concerns like logging, security, etc.
- **Example**:
  ```java
  @Aspect
  @Component
  public class LoggingAspect {
      @Before("execution(* com.example.service.*.*(..))")
      public void logBefore() {
          // Logging logic
      }
  }
  ```

#### `@Before`
- **Purpose**: Runs advice before the join point.
- **Example**:
  ```java
  @Before("execution(* com.example.service.*.*(..))")
  public void logBefore() {
      // Logging logic
  }
  ```

#### `@After`
- **Purpose**: Runs advice after the join point.
- **Example**:
  ```java
  @After("execution(* com.example.service.*.*(..))")
  public void logAfter() {
      // Logging logic
  }
  ```

#### `@AfterReturning`
- **Purpose**: Runs advice after a join point completes normally.
- **Example**:
  ```java
  @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
  public void logAfterReturning(Object result) {
      // Logging logic
  }
  ```

#### `@AfterThrowing`
- **Purpose**: Runs advice after a join point throws an exception.
- **Example**:
  ```java
  @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "error")
  public void logAfterThrowing(Throwable error) {
      // Logging logic
  }
  ```

#### `@Around`
- **Purpose**: Surrounds a join point, controlling when and if the join point should be executed.
- **Example**:
  ```java
  @Around("execution(* com.example.service.*.*(..))")
  public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
      // Before logic
      Object result = joinPoint.proceed();
      // After logic
      return result;
  }
  ```

#### `@Pointcut`
- **Purpose**: Declares a reusable pointcut expression.
- **Example**:
  ```java
  @Pointcut("execution(* com.example.service.*.*(..))")
  public void serviceMethods() {
      // Pointcut logic
  }
  ```

### JPA Annotations

#### `@Entity`
- **Purpose**: Specifies that the class is an entity mapped to a database table.
- **Example**:
  ```java
  @Entity
  public class User {
      @Id
      @GeneratedValue(strategy = GenerationType.IDENTITY)
      private Long id;

      @Column(name = "username")
      private String username;
  }
  ```

#### `@Table`
- **Purpose**: Specifies the table name for an entity.
- **Example**:
  ```java
  @Entity
  @Table(name = "users")
  public class User {
      // Entity fields
  }
  ```

#### `@Id`
- **Purpose**: Specifies the primary key of an entity.
- **Example**:
  ```java
  @Entity
  public class User {
      @Id
      @GeneratedValue(strategy = GenerationType.IDENTITY)
      private Long id;
  }
  ```

#### `@GeneratedValue`
- **Purpose**: Specifies the generation strategy for the primary key.
- **Example**:
  ```java
  @Entity
  public class User {
      @Id
      @GeneratedValue(strategy = GenerationType.IDENTITY)
      private Long id;
  }
  ```

#### `@Column`
- **Purpose**: Specifies a column for an entity.
- **Example**:
  ```java
  @Column(name = "username", nullable = false)
  private String username;
  ```

#### `@OneToOne`
- **Purpose**: Defines a one-to-one relationship between two entities.
- **Example**:
  ```java
  @OneToOne
  @JoinColumn(name = "profile_id")
  private Profile profile;
  ```

#### `@OneToMany`
- **Purpose**: Defines a one-to-many relationship between two entities.
- **Example**:
  ```java
  @OneToMany(mappedBy = "user")
  private List<Order> orders;
  ```

#### `@ManyToOne`
- **Purpose**: Defines a many-to-one relationship between two entities.
- **Example**:
  ```java
  @ManyToOne
  @JoinColumn(name = "user_id")
  private User user;
  ```

#### `@ManyToMany`
- **Purpose**: Defines a many-to-many relationship between two entities.
- **Example**:
  ```java
  @ManyToMany
  @JoinTable(name = "user_roles",
             joinColumns = @JoinColumn(name = "user_id"),
             inverseJoinColumns = @JoinColumn(name = "role_id"))
  private Set<Role> roles;
  ```

#### `@JoinColumn`
- **Purpose**: Specifies the foreign key column.
- **Example**:
  ```java
  @JoinColumn(name = "user_id")
  private User user;
  ```

#### `@JoinTable`
- **Purpose**: Specifies the join table for many-to-many relationships.
- **Example**:
  ```java
  @JoinTable(name = "user_roles",
             joinColumns = @JoinColumn(name = "user_id"),
             inverseJoinColumns = @JoinColumn(name = "role_id"))
  private Set<Role> roles;
  ```

#### `@Transient`
- **Purpose**: Marks a field to be ignored by JPA.
- **Example**:
  ```java
  @Transient
  private String nonPersistentField;
  ```

#### `@Lob`
- **Purpose**: Marks a field to be persisted as a large object (LOB).
- **Example**:
  ```java
  @Lob
  private byte[] fileData;
  ```

#### `@Enumerated`
- **Purpose**: Specifies how an enum should be persisted.
- **Example**:
  ```java
  @Enumerated(EnumType.STRING)
  private Status status;
  ```

#### `@Version`
- **Purpose**: Adds optimistic locking to an entity.
- **Example**:
  ```java
  @Version
  private int version;
  ```

#### `@Query`
- **Purpose**: Defines a custom JPQL or SQL query.
- **Example**:
  ```java
  @Query("SELECT u FROM User u WHERE u.username = :username")
  User findByUsername(@Param("username") String username);
  ```

#### `@Modifying`
- **Purpose**: Indicates that a `@Query` method is a modifying query.
- **Example**:
  ```java
  @Modifying
  @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
  void updateUserStatus(@Param("id") Long id, @Param("status") Status status);
  ```

#### `@Transactional`
- **Purpose**: Manages transaction boundaries.
- **Example**:
  ```java
  @Transactional
  public void performTransaction() {
      // Transactional logic
  }
  ```

### Spring MVC Annotations

#### `@RequestMapping`
- **Purpose**: Maps HTTP requests to handler methods.
- **Example**:
  ```java
  @RequestMapping("/users")
  public String getUsers() {
      return "users";
  }
  ```

#### `@GetMapping`
- **Purpose**: Shortcut for `@RequestMapping` with `method = RequestMethod.GET`.
- **Example**:
  ```java
  @GetMapping("/users")
  public String getUsers() {
      return "users";
  }
  ```

#### `@PostMapping`
- **Purpose**: Shortcut for `@RequestMapping` with `method = RequestMethod.POST`.
- **Example**:
  ```java
  @PostMapping("/users")
  public String createUser() {
      return "user";
  }
  ```

#### `@PutMapping`
- **Purpose**: Shortcut for `@RequestMapping` with `method = RequestMethod.PUT`.
- **Example**:
  ```java
  @PutMapping("/users/{id}")
  public String updateUser(@PathVariable Long id) {
      return "user";
  }
  ```

#### `@DeleteMapping`
- **Purpose**: Shortcut for `@RequestMapping

` with `method = RequestMethod.DELETE`.
- **Example**:
  ```java
  @DeleteMapping("/users/{id}")
  public void deleteUser(@PathVariable Long id) {
      // Delete logic
  }
  ```

#### `@PatchMapping`
- **Purpose**: Shortcut for `@RequestMapping` with `method = RequestMethod.PATCH`.
- **Example**:
  ```java
  @PatchMapping("/users/{id}")
  public String patchUser(@PathVariable Long id) {
      return "user";
  }
  ```

#### `@RequestParam`
- **Purpose**: Binds a request parameter to a method argument.
- **Example**:
  ```java
  @GetMapping("/users")
  public String getUser(@RequestParam String name) {
      return "user";
  }
  ```

#### `@PathVariable`
- **Purpose**: Binds a URI template variable to a method argument.
- **Example**:
  ```java
  @GetMapping("/users/{id}")
  public String getUser(@PathVariable Long id) {
      return "user";
  }
  ```

#### `@RequestBody`
- **Purpose**: Binds the body of a request to a method argument.
- **Example**:
  ```java
  @PostMapping("/users")
  public String createUser(@RequestBody User user) {
      return "user";
  }
  ```

#### `@ResponseBody`
- **Purpose**: Binds the return value of a method to the response body.
- **Example**:
  ```java
  @GetMapping("/users/{id}")
  @ResponseBody
  public User getUser(@PathVariable Long id) {
      return new User();
  }
  ```

#### `@ModelAttribute`
- **Purpose**: Binds a model attribute to a method argument.
- **Example**:
  ```java
  @ModelAttribute("user")
  public User getUser() {
      return new User();
  }
  ```

#### `@SessionAttributes`
- **Purpose**: Specifies the model attributes that should be stored in the session.
- **Example**:
  ```java
  @Controller
  @SessionAttributes("user")
  public class UserController {
      // Controller logic
  }
  ```

#### `@CrossOrigin`
- **Purpose**: Enables Cross-Origin Resource Sharing (CORS) on methods or classes.
- **Example**:
  ```java
  @CrossOrigin(origins = "http://example.com")
  @GetMapping("/users")
  public String getUsers() {
      return "users";
  }
  ```

#### `@ExceptionHandler`
- **Purpose**: Handles exceptions thrown by handler methods.
- **Example**:
  ```java
  @ExceptionHandler(Exception.class)
  public String handleException() {
      return "error";
  }
  ```

#### `@ControllerAdvice`
- **Purpose**: Applies `@ExceptionHandler` methods globally.
- **Example**:
  ```java
  @ControllerAdvice
  public class GlobalExceptionHandler {
      @ExceptionHandler(Exception.class)
      public String handleException() {
          return "error";
      }
  }
  ```

#### `@ResponseStatus`
- **Purpose**: Marks a method or exception class with a response status code.
- **Example**:
  ```java
  @ResponseStatus(HttpStatus.NOT_FOUND)
  public class ResourceNotFoundException extends RuntimeException {
      // Exception logic
  }
  ```

#### `@RestControllerAdvice`
- **Purpose**: Combines `@ControllerAdvice` and `@ResponseBody`.
- **Example**:
  ```java
  @RestControllerAdvice
  public class RestExceptionHandler {
      @ExceptionHandler(Exception.class)
      public ResponseEntity<String> handleException() {
          return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Error");
      }
  }
  ```

### Spring Boot Annotations

#### `@SpringBootApplication`
- **Purpose**: Combines `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan` for Spring Boot applications.
- **Example**:
  ```java
  @SpringBootApplication
  public class MyApplication {
      public static void main(String[] args) {
          SpringApplication.run(MyApplication.class, args);
      }
  }
  ```

#### `@EnableAutoConfiguration`
- **Purpose**: Enables Spring Boot’s auto-configuration mechanism.
- **Example**:
  ```java
  @EnableAutoConfiguration
  public class MyApplication {
      // Application logic
  }
  ```

#### `@ComponentScan`
- **Purpose**: Configures component scanning directives for Spring.
- **Example**:
  ```java
  @ComponentScan(basePackages = "com.example")
  public class MyApplication {
      // Application logic
  }
  ```

#### `@ConfigurationProperties`
- **Purpose**: Binds externalized configuration properties to a Java bean.
- **Example**:
  ```java
  @ConfigurationProperties(prefix = "app")
  public class AppProperties {
      private String name;
      // Getters and setters
  }
  ```

#### `@ConditionalOnProperty`
- **Purpose**: Configures a bean to be created based on the presence of a property.
- **Example**:
  ```java
  @ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
  public class FeatureBean {
      // Bean logic
  }
  ```

#### `@ConditionalOnClass`
- **Purpose**: Configures a bean to be created based on the presence of a class.
- **Example**:
  ```java
  @ConditionalOnClass(name = "com.example.MyClass")
  public class MyConditionalBean {
      // Bean logic
  }
  ```

#### `@ConditionalOnMissingBean`
- **Purpose**: Configures a bean to be created if a bean of the same type is missing.
- **Example**:
  ```java
  @ConditionalOnMissingBean(MyService.class)
  public class MyFallbackService implements MyService {
      // Fallback logic
  }
  ```

#### `@ConditionalOnBean`
- **Purpose**: Configures a bean to be created if a bean of the specified type is present.
- **Example**:
  ```java
  @ConditionalOnBean(MyService.class)
  public class MyConditionalBean {
      // Bean logic
  }
  ```

#### `@ConditionalOnExpression`
- **Purpose**: Configures a bean to be created based on a SpEL expression.
- **Example**:
  ```java
  @ConditionalOnExpression("${feature.enabled}")
  public class MyConditionalBean {
      // Bean logic
  }
  ```

#### `@ConditionalOnWebApplication`
- **Purpose**: Configures a bean to be created only in a web application.
- **Example**:
  ```java
  @ConditionalOnWebApplication
  public class MyWebBean {
      // Bean logic
  }
  ```

#### `@ConditionalOnNotWebApplication`
- **Purpose**: Configures a bean to be created only in a non-web application.
- **Example**:
  ```java
  @ConditionalOnNotWebApplication
  public class MyNonWebBean {
      // Bean logic
  }
  ```
