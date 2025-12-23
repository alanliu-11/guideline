

## 14. Quick Reference Cheat Sheet

### 14.1 Do's and Don'ts Summary
> Quick reference for common patterns and anti-patterns.

| Category | ❌ Don't | ✅ Do |
|----------|----------|-------|
| DTOs | `@Data class UserDTO` | `record UserResponse(...)` |
| Value Objects | Mutable class with setters | Immutable record with validation |
| Entity IDs | `Long id` or `UUID id` | `OrderId id` (typed wrapper) |
| Injection | `@Autowired private Repo repo` | `@RequiredArgsConstructor + final` |
| State Changes | `order.setStatus("SHIPPED")` | `order.ship()` |
| Nulls | Return `null` | Return `Optional<T>` |
| Packages | `javax.persistence.*` | `jakarta.persistence.*` |
| Collections | `list.get(0)` | `list.getFirst()` |
| Threads | WebFlux for simple I/O | Virtual Threads + blocking |
| Exceptions | Generic `RuntimeException` | Domain-specific exceptions |
| Validation | Scattered null checks | Self-validating Value Objects |
| Naming | `UserEntity`, `OrderDAO` | `Customer`, `OrderRepository` |

### 14.2 Layer Responsibilities
> Clear separation of concerns across architectural layers.

| Layer | Responsibility | Spring Annotations Allowed |
|-------|----------------|---------------------------|
| Domain | Business logic, Entities, Value Objects, Domain Events | None (Pure Java) |
| Application | Use cases, Orchestration, DTOs, Mappers | `@Service`, `@Transactional` |
| Infrastructure | Persistence, External APIs, Messaging | `@Repository`, `@Component`, `@Configuration` |
| Interfaces | REST Controllers, Message Listeners | `@RestController`, `@ControllerAdvice` |

### 14.3 Record Templates
> Reusable templates for common record patterns.

```
java
// Value Object Template
public record {{Name}}({{Type}} value) {
    public {{Name}} {
        Objects.requireNonNull(value, "{{Name}} cannot be null");
        // Add validation rules here
    }
    
    public static {{Name}} of({{Type}} value) {
        return new {{Name}}(value);
    }
}

// Typed ID Template
public record {{Entity}}Id(UUID value) {
    public {{Entity}}Id {
        Objects.requireNonNull(value, "{{Entity}}Id cannot be null");
    }
    
    public static {{Entity}}Id generate() {
        return new {{Entity}}Id(UUID.randomUUID());
    }
    
    public static {{Entity}}Id of(UUID value) {
        return new {{Entity}}Id(value);
    }
    
    public static {{Entity}}Id of(String value) {
        return new {{Entity}}Id(UUID.fromString(value));
    }
    
    @Override
    public String toString() {
        return value.toString();
    }
}

// Domain Event Template
public record {{Event}}Event(
    {{AggregateId}} aggregateId,
    // Event-specific fields
    Instant occurredAt
) {
    public {{Event}}Event {
        Objects.requireNonNull(aggregateId);
        occurredAt = occurredAt != null ? occurredAt : Instant.now();
    }
}

// Command DTO Template
public record {{Action}}Command(
    @NotNull UUID targetId,
    @NotBlank String requiredField,
    @Valid List<{{Nested}}Command> nestedItems
) {}

// Response DTO Template
public record {{Entity}}Response(
    UUID id,
    String status,
    // Response-specific fields
    Instant createdAt,
    Instant updatedAt
) {}
```

### 14.4 Common Patterns Quick Reference
> Common DDD and design patterns implementation examples.

```
java
// Pattern 1: Factory Method in Aggregate
public class Order {
    public static Order create(CustomerId customerId) {
        var order = new Order();
        order.id = OrderId.generate();
        order.customerId = customerId;
        order.status = OrderStatus.DRAFT;
        order.items = new ArrayList<>();
        order.createdAt = Instant.now();
        return order;
    }
    
    // For reconstituting from persistence
    public static Order reconstitute(
            OrderId id,
            CustomerId customerId,
            OrderStatus status,
            List<OrderItem> items,
            Instant createdAt) {
        var order = new Order();
        order.id = id;
        order.customerId = customerId;
        order.status = status;
        order.items = new ArrayList<>(items);
        order.createdAt = createdAt;
        return order;
    }
}

// Pattern 2: Specification Pattern for Complex Queries
public interface Specification<T> {
    boolean isSatisfiedBy(T entity);
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
}

public class ActiveOrderSpecification implements Specification<Order> {
    
    @Override
    public boolean isSatisfiedBy(Order order) {
        return order.getStatus() != OrderStatus.CANCELLED 
            && order.getStatus() != OrderStatus.DELIVERED;
    }
    
    @Override
    public Predicate toPredicate(Root<Order> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
        return cb.not(root.get("status").in(OrderStatus.CANCELLED, OrderStatus.DELIVERED));
    }
}

// Pattern 3: Domain Service for Cross-Aggregate Logic
public class PricingService {
    
    public Money calculateTotal(List<OrderItem> items, DiscountPolicy discountPolicy) {
        var subtotal = items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
        
        return discountPolicy.apply(subtotal);
    }
}

// Pattern 4: Policy Pattern for Business Rules
public sealed interface DiscountPolicy permits NoDiscount, PercentageDiscount, FixedDiscount {
    Money apply(Money originalAmount);
}

public record NoDiscount() implements DiscountPolicy {
    @Override
    public Money apply(Money originalAmount) {
        return originalAmount;
    }
}

public record PercentageDiscount(int percentage) implements DiscountPolicy {
    public PercentageDiscount {
        if (percentage < 0 || percentage > 100) {
            throw new DomainRuleException("Percentage must be between 0 and 100");
        }
    }
    
    @Override
    public Money apply(Money originalAmount) {
        var multiplier = BigDecimal.valueOf(100 - percentage).divide(BigDecimal.valueOf(100));
        return originalAmount.multiply(multiplier);
    }
}

public record FixedDiscount(Money discountAmount) implements DiscountPolicy {
    @Override
    public Money apply(Money originalAmount) {
        var result = originalAmount.subtract(discountAmount);
        return result.isNegative() ? Money.ZERO : result;
    }
}
```

## 15. Checklist for Code Review
> Use this checklist when reviewing Spring Boot code.

### 15.1 Domain Layer Checklist
- [ ] No framework dependencies (no Spring, no JPA annotations in pure domain)
- [ ] Entities are rich (contain business logic, not just getters/setters)
- [ ] Value Objects are Records (immutable, self-validating)
- [ ] Typed IDs used (OrderId instead of UUID/Long)
- [ ] No public setters on entities (state changes via business methods)
- [ ] Domain events defined as Records
- [ ] Ubiquitous language used in naming
- [ ] Repository interfaces defined (not implementations)
- [ ] Domain exceptions used (not generic RuntimeException)

### 15.2 Application Layer Checklist
- [ ] Use cases are single-purpose (one public method per use case)
- [ ] Constructor injection used (no @Autowired on fields)
- [ ] All fields are final
- [ ] DTOs are Records (immutable)
- [ ] Input validation via Bean Validation annotations
- [ ] Transactions managed at use case level
- [ ] Mappers used for Entity <-> DTO conversion
- [ ] No business logic (orchestration only)

### 15.3 Infrastructure Layer Checklist
- [ ] Implements domain interfaces (Repository pattern)
- [ ] Spring Data repositories are internal (not exposed)
- [ ] AttributeConverters defined for Value Objects
- [ ] External clients wrapped in adapters
- [ ] Configuration classes are focused (single responsibility)
- [ ] No business logic (pure technical implementation)

### 15.4 Interface Layer Checklist
- [ ] Controllers are thin (delegate to use cases)
- [ ] Global exception handler defined
- [ ] ProblemDetail used for error responses (RFC 7807)
- [ ] OpenAPI annotations present
- [ ] Input validation via @Valid
- [ ] Proper HTTP status codes returned
- [ ] No business logic in controllers

### 15.5 General Checklist
- [ ] Java 21 features used (Records, Pattern Matching, Sealed Classes)
- [ ] jakarta.* packages used (not javax.*)
- [ ] Optional used only for return types
- [ ] Stream API used appropriately
- [ ] No Lombok @Data on entities or value objects
- [ ] Virtual Threads enabled (not WebFlux for simple I/O)
- [ ] Tests follow the testing pyramid
- [ ] ArchUnit tests enforce architecture rules
