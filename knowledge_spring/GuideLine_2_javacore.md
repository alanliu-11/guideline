
## 1. Java 21 Language Standards
### 1.1 Version Baseline
Assume Java 21 (LTS) is the environment. Do not offer Java 8 or 11 syntax.

### 2.2 Records
> Strictly enforce the use of Java records for:
  > DDD Value Objects (immutable data holders).
  > DTOs (Data Transfer Objects).
  > Domain Events.
> Configuration Properties.
> Constraint: Do not use Lombok @Data or standard Classes for immutable data.

### 2.3 Sealed Classes/Interfaces
> Use sealed interfaces/classes to represent finite domain states or business rule variants.
> Enables exhaustive pattern matching in switch expressions.

```
java
public sealed interface PaymentStatus 
    permits Pending, Completed, Failed, Refunded {
}

public record Pending(Instant initiatedAt) implements PaymentStatus {}
public record Completed(Instant completedAt, String transactionId) implements PaymentStatus {}
public record Failed(Instant failedAt, String reason) implements PaymentStatus {}
public record Refunded(Instant refundedAt, Money amount) implements PaymentStatus {}

```

### 2.4 Pattern Matching
> Use Pattern Matching for switch and instanceof to reduce boilerplate.
```
java
// Pattern Matching for switch (exhaustive with sealed types)
public String describeStatus(PaymentStatus status) {
    return switch (status) {
        case Pending p    -> "Payment initiated at " + p.initiatedAt();
        case Completed c  -> "Payment completed: " + c.transactionId();
        case Failed f     -> "Payment failed: " + f.reason();
        case Refunded r   -> "Refunded amount: " + r.amount();
    };
}

// Pattern Matching for instanceof
if (obj instanceof Customer customer && customer.isActive()) {
    processActiveCustomer(customer);
}

```

### 2.5 Virtual Threads (Project Loom)
> Assume the application runs on Virtual Threads (spring.threads.virtual.enabled=true).
> Constraint: Prefer standard blocking I/O over Reactive (WebFlux) complexity.

### 2.6 Text Blocks
> Use Text Blocks (""") for multi-line strings: JSON, SQL, XML, HTML.
```
java
String query = """
    SELECT o.id, o.status, o.total_amount
    FROM orders o
    WHERE o.customer_id = :customerId
      AND o.status = 'ACTIVE'
    ORDER BY o.created_at DESC
    """;
```    

### 2.7 Sequenced Collections
> Use Java 21 SequencedCollection interfaces.
```
java
// Instead of: list.get(0) and list.get(list.size() - 1)
var first = list.getFirst();
var last = list.getLast();
var reversed = list.reversed();
```
