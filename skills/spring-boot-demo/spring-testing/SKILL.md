---
name: spring-testing
description: >-
  JUnit 5, MockMvc, @DataJpaTest, Testcontainers, and slice-test strategy
  for spring-boot-demo. Use when writing or fixing Java tests, adding
  Spring Boot endpoint tests, or choosing @SpringBootTest vs slice tests.
---

# Spring Testing (spring-boot-demo)

TDD for this Spring Boot service. Shared `tdd-workflow` is Node-oriented; use
this skill for JUnit 5, MockMvc, and Testcontainers.

## When to Activate

- Adding a feature, bug fix, or refactor in Java
- Writing controller, service, or repository tests
- Deciding between slice tests and full `@SpringBootTest`

## Commands

```bash
mvn test
mvn -Dtest=UserControllerTest test
mvn -Dtest=UserControllerTest#createUserReturns201 test
mvn jacoco:report
```

If the project uses Gradle: `./gradlew test` and `--tests UserControllerTest`.
Do not edit production code until the new test has compiled and failed for the
intended reason.

## Which Test Slice

| Need | Annotation | Spring context |
|------|------------|----------------|
| JSON + HTTP mapping | `@WebMvcTest(UserController.class)` | web slice, mock the service |
| Service rules | Plain JUnit + Mockito | none |
| Queries / constraints | `@DataJpaTest` | JPA slice |
| Full wiring, listeners | `@SpringBootTest` | full (use sparingly) |
| Real Postgres | `@SpringBootTest` + Testcontainers | full + container |

Default to the narrowest slice that proves the behavior.

## RED → GREEN

1. Write a failing test for the observable behavior (status, JSON, DB row).
2. Run the focused test; confirm RED.
3. Implement the minimum production code.
4. Re-run the same test; confirm GREEN.
5. Refactor only while GREEN. Keep JaCoCo on changed types ≥ 80%.

## WebMvcTest

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired MockMvc mvc;
    @Autowired ObjectMapper objectMapper;
    @MockitoBean UserService users;

    @Test
    void createUserReturns201() throws Exception {
        UUID id = UUID.fromString("11111111-1111-1111-1111-111111111111");
        given(users.create(any())).willReturn(new UserResponse(id, "ada@example.com", "Ada"));

        mvc.perform(post("/api/v1/users")
                        .contentType(APPLICATION_JSON)
                        .content("""
                                {"email":"ada@example.com","name":"Ada"}
                                """))
                .andExpect(status().isCreated())
                .andExpect(header().string("Location", "/api/v1/users/" + id))
                .andExpect(jsonPath("$.email").value("ada@example.com"));
    }

    @Test
    void createUserRejectsInvalidEmail() throws Exception {
        mvc.perform(post("/api/v1/users")
                        .contentType(APPLICATION_JSON)
                        .content("""
                                {"email":"not-an-email","name":"Ada"}
                                """))
                .andExpect(status().isUnprocessableEntity())
                .andExpect(jsonPath("$.code").value("VALIDATION_ERROR"));
    }
}
```

Assert the project's error document (ProblemDetail `code` / `details`). Do not
mock `UserController`; mock its collaborators.

## Service Unit Test

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock UserRepository users;
    @InjectMocks UserService service;

    @Test
    void createRejectsDuplicateEmail() {
        given(users.existsByEmail("ada@example.com")).willReturn(true);

        assertThatThrownBy(() -> service.create(
                new CreateUserRequest("ada@example.com", "Ada")))
                .isInstanceOf(ConflictException.class);
    }
}
```

## DataJpaTest

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired UserRepository users;

    @Test
    void existsByEmailIsTrueAfterSave() {
        users.save(new User("ada@example.com", "Ada"));

        assertThat(users.existsByEmail("ada@example.com")).isTrue();
    }
}
```

Use an in-memory DB only when the SQL dialect matches production well enough.
Prefer Testcontainers Postgres for anything involving constraints, JSONB, or
partial indexes.

## Testcontainers

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
class UserApiIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void datasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

Reuse a single container per test class. Do not start Docker for pure unit tests.

## Anti-Patterns

| Avoid | Do instead |
|-------|------------|
| `@SpringBootTest` for every class | Slice test unless full wiring is the point |
| `Thread.sleep` | Awaitility or MockMvc's built-in wait |
| Asserting Hibernate internals | Assert HTTP/JSON or repository results |
| Shared mutable `@BeforeAll` data | Per-test setup; `@Transactional` rollback on slices |

## Checklist

- [ ] New behavior has a failing test before production edits
- [ ] Narrowest slice that still proves the contract
- [ ] Validation and 404/409 paths are tested, not only 200
- [ ] No real network in unit tests
- [ ] Coverage on touched types stays ≥ 80% unless the gap is documented
