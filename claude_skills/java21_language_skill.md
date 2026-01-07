# Claude Skill: Java 21 Language Standards

## Purpose
This skill provides Java 21 (LTS) language feature guidelines and standards. Use this when writing Java code to ensure modern, idiomatic Java 21 usage.

## Version Baseline
- Assume Java 21 (LTS) is the environment
- Do not offer Java 8 or 11 syntax
- Always use Java 21 features when appropriate

## Records

**Strictly enforce the use of Java records for:**
- DDD Value Objects (immutable data holders)
- DTOs (Data Transfer Objects)
- Domain Events
- Configuration Properties

**Constraint:** Do not use Lombok @Data or standard Classes for immutable data.

## Sealed Classes/Interfaces

**Use sealed interfaces/classes to represent:**
- Finite domain states
- Business rule variants
- Enables exhaustive pattern matching in switch expressions

**Example:**
```java
public sealed interface PaymentStatus 
    permits Pending, Completed, Failed, Refunded {
}

public record Pending(Instant initiatedAt) implements PaymentStatus {}
public record Completed(Instant completedAt, String transactionId) implements PaymentStatus {}
public record Failed(Instant failedAt, String reason) implements PaymentStatus {}
public record Refunded(Instant refundedAt, Money amount) implements PaymentStatus {}
```

## Pattern Matching

**Use Pattern Matching for switch and instanceof to reduce boilerplate.**

**Pattern Matching for switch (exhaustive with sealed types):**
```java
public String describeStatus(PaymentStatus status) {
    return switch (status) {
        case Pending p    -> "Payment initiated at " + p.initiatedAt();
        case Completed c  -> "Payment completed: " + c.transactionId();
        case Failed f     -> "Payment failed: " + f.reason();
        case Refunded r   -> "Refunded amount: " + r.amount();
    };
}
```

**Pattern Matching for instanceof:**
```java
if (obj instanceof Customer customer && customer.isActive()) {
    processActiveCustomer(customer);
}
```

## Virtual Threads (Project Loom)

- Assume the application runs on Virtual Threads (`spring.threads.virtual.enabled=true`)
- **Constraint:** Prefer standard blocking I/O over Reactive (WebFlux) complexity

## Text Blocks

**Use Text Blocks (""") for multi-line strings:**
- JSON
- SQL
- XML
- HTML

**Example:**
```java
String query = """
    SELECT o.id, o.status, o.total_amount
    FROM orders o
    WHERE o.customer_id = :customerId
      AND o.status = 'ACTIVE'
    ORDER BY o.created_at DESC
    """;
```

## Sequenced Collections

**Use Java 21 SequencedCollection interfaces.**

**Instead of:**
```java
list.get(0)
list.get(list.size() - 1)
```

**Use:**
```java
var first = list.getFirst();
var last = list.getLast();
var reversed = list.reversed();
```

## Key Takeaways

1. **Always use Records** for immutable data structures
2. **Use Sealed Classes** for finite state representations
3. **Leverage Pattern Matching** to reduce boilerplate
4. **Prefer Virtual Threads** over reactive programming for simple I/O
5. **Use Text Blocks** for multi-line strings
6. **Use Sequenced Collections** for better collection APIs

