---
name: spring-boot-conventions
description: >-
  Spring Boot 3 layered architecture, REST controllers, constructor
  injection, DTO mapping, and configuration for spring-boot-demo. Use when
  adding APIs, services, configuration, or structuring Java packages in
  this project.
---

# Spring Boot Conventions (spring-boot-demo)

Application structure for this Spring Boot 3 / Java 21 service. Use
`api-design` for HTTP contracts; this skill covers Spring wiring.

## When to Activate

- Adding a REST endpoint, service, or configuration class
- Choosing package layout or injection style
- Mapping entities to API DTOs
- Introducing profiles or `application.yml` settings

## Package Layout

```
com.example.demo
├── DemoApplication.java
├── api/                    # @RestController, request/response records
├── service/                # @Service business logic
├── domain/                 # JPA entities (not returned from controllers)
├── repository/             # Spring Data interfaces
├── config/                 # @Configuration
└── error/                  # ProblemDetail / @RestControllerAdvice
```

Controller → Service → Repository. Controllers never call `EntityManager` or
repositories directly.

## Constructor Injection

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;

    @PostMapping
    public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest request) {
        UserResponse created = userService.create(request);
        URI location = URI.create("/api/v1/users/" + created.id());
        return ResponseEntity.created(location).body(created);
    }

    @GetMapping("/{id}")
    public UserResponse get(@PathVariable UUID id) {
        return userService.get(id);
    }
}
```

- Inject with a constructor (`@RequiredArgsConstructor` or an explicit ctor).
- Do not use `@Autowired` on fields.
- Do not use `@Data` on entities; prefer records for DTOs.

## DTOs vs Entities

```java
public record CreateUserRequest(
        @Email @NotBlank String email,
        @NotBlank @Size(max = 100) String name
) {}

public record UserResponse(UUID id, String email, String name) {
    public static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getEmail(), user.getName());
    }
}
```

Never serialize a JPA entity (lazy proxies, infinite graphs, leaked columns).

## Service Layer

```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository users;

    @Transactional
    public UserResponse create(CreateUserRequest request) {
        if (users.existsByEmail(request.email())) {
            throw new ConflictException("email_taken", "Email already registered");
        }
        User saved = users.save(new User(request.email(), request.name()));
        return UserResponse.from(saved);
    }

    @Transactional(readOnly = true)
    public UserResponse get(UUID id) {
        return users.findById(id)
                .map(UserResponse::from)
                .orElseThrow(() -> new NotFoundException("User", id));
    }
}
```

`@Transactional` belongs on the service, not the controller.

## Errors

```java
@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    public ProblemDetail notFound(NotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(NOT_FOUND, ex.getMessage());
        problem.setProperty("code", ex.getCode());
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail validation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(UNPROCESSABLE_ENTITY);
        problem.setProperty("code", "VALIDATION_ERROR");
        problem.setProperty("details", ex.getBindingResult().getFieldErrors().stream()
                .map(err -> Map.of("field", err.getField(), "message", err.getDefaultMessage()))
                .toList());
        return problem;
    }
}
```

Keep 500 responses generic. Log the cause; do not put stack traces in JSON.

## Configuration

```yaml
# application.yml
spring:
  application:
    name: spring-boot-demo
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
```

- `open-in-view: false` is required.
- Schema changes go through Flyway/Liquibase (`ddl-auto: validate` in all
  shared environments).
- Secrets from env vars, never committed YAML.

## Checklist

- [ ] URL is `/api/v1/` + plural nouns
- [ ] Constructor injection only
- [ ] Controller returns DTOs, not entities
- [ ] `@Valid` on write request bodies
- [ ] `@Transactional` on the service method that needs it
- [ ] `spring.jpa.open-in-view` is false
