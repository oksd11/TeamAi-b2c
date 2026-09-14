---
name: spring-data-jpa
description: >-
  Spring Data JPA entities, repositories, transactions, and N+1 prevention
  for spring-boot-demo. Use when adding entities, query methods, lazy
  loading, or Flyway schema changes in this Spring Boot project.
---

# Spring Data JPA (spring-boot-demo)

Persistence conventions for this service. Pair with `database-migrations` for
zero-downtime DDL.

## When to Activate

- Adding or changing JPA entities and relationships
- Writing repository query methods
- Debugging lazy loading, N+1, or transaction boundaries
- Adding Flyway/Liquibase migrations

## Entity Rules

```java
@Entity
@Table(name = "users", indexes = @Index(name = "uk_users_email", columnList = "email", unique = true))
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, unique = true, length = 255)
    private String email;

    @Column(nullable = false, length = 100)
    private String name;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    protected User() {} // JPA

    public User(String email, String name) {
        this.email = email;
        this.name = name;
    }
    // getters; no Lombok @Data on entities
}
```

- Explicit table/column names; do not rely on implicit naming in production.
- No Lombok `@Data` / `@EqualsAndHashCode` on entities (hashCode on proxies).
- Relationships: `LAZY` by default. Set `FetchType.EAGER` only with a written reason.

```java
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY, cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
```

## Repositories

```java
public interface UserRepository extends JpaRepository<User, UUID> {
    Optional<User> findByEmail(String email);

    boolean existsByEmail(String email);

    @Query("select u from User u where u.createdAt >= :since")
    List<User> findCreatedSince(@Param("since") Instant since);
}

public interface OrderRepository extends JpaRepository<Order, UUID> {
    @EntityGraph(attributePaths = "items")
    Optional<Order> findWithItemsById(UUID id);
}
```

- Derived query names for simple lookups.
- `@Query` when the method name would lie or join.
- `@EntityGraph` or `join fetch` when the caller needs a graph — do not
  initialize lazy collections in a loop.

## Transactions

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orders;

    @Transactional
    public OrderResponse place(PlaceOrderRequest request) {
        Order order = new Order(request.userId());
        request.items().forEach(order::addItem);
        return OrderResponse.from(orders.save(order));
    }

    @Transactional(readOnly = true)
    public OrderResponse get(UUID id) {
        Order order = orders.findWithItemsById(id)
                .orElseThrow(() -> new NotFoundException("Order", id));
        return OrderResponse.from(order);
    }
}
```

- Write methods: `@Transactional` on the service.
- Read methods: `@Transactional(readOnly = true)`.
- Do not catch, log, and swallow `DataIntegrityViolationException` without
  mapping it to a domain conflict.

## N+1

```java
// FAIL: one query per order
List<Order> found = orders.findAll();
found.forEach(o -> o.getItems().size());

// PASS: load the graph in the repository query
List<Order> found = orders.findAllWithItems();
```

```java
@EntityGraph(attributePaths = "items")
@Query("select o from Order o")
List<Order> findAllWithItems();
```

Prefer `selectin` / `@EntityGraph` / DTO projections over Open Session in View.
`spring.jpa.open-in-view` must stay `false`.

## Migrations

Hibernate `ddl-auto` is `validate` outside local experiments. Schema changes:

```sql
-- db/migration/V20260914__add_users_avatar_url.sql
ALTER TABLE users ADD COLUMN avatar_url TEXT;
```

```bash
# Flyway
mvn flyway:migrate
```

Add nullable columns first; backfill; then constrain. Do not edit applied
migration checksums.

## Checklist

- [ ] Entity has a protected no-arg constructor and a real domain constructor
- [ ] Associations are `LAZY`; graphs loaded explicitly
- [ ] Unique columns have a DB constraint, not only a Java check
- [ ] Service owns `@Transactional`
- [ ] Migration file added; `ddl-auto` is not `update` in shared envs
