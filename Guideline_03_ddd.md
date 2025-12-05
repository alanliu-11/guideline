
## 3. Domain-Driven Design (DDD) Principles
### 3.1 Rich Domain Models
> Reject "Anemic Domain Models." Entities must encapsulate business logic.  
> State changes must happen via named business methods.  
> Constraint: Do not use Lombok @Data on @Entity classes.  
```
java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {
    
    @EmbeddedId
    private OrderId id;
    
    @Embedded
    private Money totalAmount;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    // ✅ Business method with invariant enforcement
    public void addItem(Product product, Quantity quantity) {
        if (this.status != OrderStatus.DRAFT) {
            throw new OrderModificationException("Cannot modify a submitted order");
        }
        var item = new OrderItem(product, quantity);
        this.items.add(item);
        recalculateTotal();
    }

    // ✅ Named business method (not setStatus!)
    public void submit() {
        if (this.items.isEmpty()) {
            throw new OrderSubmissionException("Cannot submit empty order");
        }
        this.status = OrderStatus.SUBMITTED;
        registerEvent(new OrderSubmittedEvent(this.id, Instant.now()));
    }

    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

### 3.2 Typed Entity IDs
> Use Value Objects for entity identifiers instead of primitives.  
> Provides type safety and prevents ID mixups between entities.  
```
java
// Value Object for Order ID
public record OrderId(UUID value) {
    public OrderId {
        Objects.requireNonNull(value, "OrderId cannot be null");
    }
    
    public static OrderId generate() {
        return new OrderId(UUID.randomUUID());
    }
    
    public static OrderId of(String value) {
        return new OrderId(UUID.fromString(value));
    }
}

// JPA Converter for persistence
@Converter(autoApply = true)
public class OrderIdConverter implements AttributeConverter<OrderId, UUID> {
    @Override
    public UUID convertToDatabaseColumn(OrderId id) {
        return id != null ? id.value() : null;
    }
    
    @Override
    public OrderId convertToEntityAttribute(UUID uuid) {
        return uuid != null ? new OrderId(uuid) : null;
    }
}
```

### 3.3 Value Objects
> Identify concepts defined by their attributes.  
> Implement as immutable records with validation in compact constructors.  

```
java
public record Money(BigDecimal amount, Currency currency) {
    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.USD);
    
    public Money {
        Objects.requireNonNull(amount, "Amount cannot be null");
        Objects.requireNonNull(currency, "Currency cannot be null");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new DomainRuleException("Money cannot be negative");
        }
    }
    
    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    public Money multiply(int quantity) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(quantity)), this.currency);
    }
    
    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException(this.currency, other.currency);
        }
    }
}

public record Email(String value) {
    private static final Pattern EMAIL_PATTERN = 
        Pattern.compile("^[A-Za-z0-9+_.-]+@(.+)$");
    
    public Email {
        Objects.requireNonNull(value, "Email cannot be null");
        if (!EMAIL_PATTERN.matcher(value).matches()) {
            throw new InvalidEmailException(value);
        }
        value = value.toLowerCase().trim();
    }
}
```

### 3.4 Aggregate Root Pattern
> Only Aggregate Roots have repositories.  
> External entities reference aggregates by ID only.  
> All modifications go through the Aggregate Root.  

```
java
// Order is the Aggregate Root
// OrderItem is an entity within the Order aggregate
// External aggregates reference Order by OrderId, not direct reference

public class Shipment {
    private ShipmentId id;
    private OrderId orderId;  // ✅ Reference by ID
    // private Order order;   // ❌ Direct reference violates aggregate boundary
}
```

### 3.5 Domain Events
> Define events as records in the domain layer.  
> Use Spring's ApplicationEventPublisher for decoupling.  

```
java
// Domain Event Definition
public record OrderSubmittedEvent(
    OrderId orderId,
    CustomerId customerId,
    Money totalAmount,
    Instant occurredAt
) {
    public OrderSubmittedEvent {
        Objects.requireNonNull(orderId);
        Objects.requireNonNull(customerId);
        Objects.requireNonNull(totalAmount);
        occurredAt = occurredAt != null ? occurredAt : Instant.now();
    }
}

// Publishing from Entity (collect events)
@Entity
public class Order extends AbstractAggregateRoot<Order> {
    
    public void submit() {
        // ... validation logic
        this.status = OrderStatus.SUBMITTED;
        registerEvent(new OrderSubmittedEvent(
            this.id, 
            this.customerId, 
            this.totalAmount, 
            Instant.now()
        ));
    }
}

// Event Handler in Application Layer
@Component
@RequiredArgsConstructor
public class OrderSubmittedEventHandler {
    
    private final NotificationService notifications;
    private final InventoryService inventory;
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handle(OrderSubmittedEvent event) {
        notifications.sendOrderConfirmation(event.orderId());
        inventory.reserveStock(event.orderId());
    }
}
```

### 3.6 Domain Exception Strategy
> Define a hierarchy of domain-specific exceptions.  
> Exceptions should be meaningful to the business domain.  
```
java
// Base Domain Exception
public abstract class DomainException extends RuntimeException {
    protected DomainException(String message) {
        super(message);
    }
}

// Specific Domain Exceptions
public class DomainRuleException extends DomainException {
    public DomainRuleException(String rule) {
        super("Domain rule violated: " + rule);
    }
}

public class EntityNotFoundException extends DomainException {
    public EntityNotFoundException(String entityType, Object id) {
        super("%s with id '%s' not found".formatted(entityType, id));
    }
}

public class OrderNotFoundException extends EntityNotFoundException {
    public OrderNotFoundException(OrderId id) {
        super("Order", id.value());
    }
}

public class InsufficientStockException extends DomainException {
    public InsufficientStockException(ProductId productId, Quantity requested, Quantity available) {
        super("Insufficient stock for product '%s': requested %d, available %d"
            .formatted(productId.value(), requested.value(), available.value()));
    }
}

public class CurrencyMismatchException extends DomainException {
    public CurrencyMismatchException(Currency expected, Currency actual) {
        super("Currency mismatch: expected %s but got %s".formatted(expected, actual));
    }
}

public class InvalidEmailException extends DomainException {
    public InvalidEmailException(String email) {
        super("Invalid email format: " + email);
    }
}

```

### 3.7 Validation Strategy
> Domain Layer: Validate invariants inside entities and value objects (self-validating).  
> Application Layer: Validate input DTOs using Bean Validation (jakarta.validation).  
> Constraint: Domain objects should never be in an invalid state.  

```
java
// Application Layer: Input validation with Bean Validation
public record CreateOrderRequest(
    @NotNull(message = "Customer ID is required")
    UUID customerId,
    
    @NotEmpty(message = "Order must have at least one item")
    List<@Valid OrderItemRequest> items
) {}

public record OrderItemRequest(
    @NotNull UUID productId,
    @Min(value = 1, message = "Quantity must be at least 1") int quantity
) {}

// Domain Layer: Invariant enforcement (self-validating)
public record Quantity(int value) {
    public Quantity {
        if (value < 0) {
            throw new DomainRuleException("Quantity cannot be negative");
        }
    }
    
    public Quantity add(Quantity other) {
        return new Quantity(this.value + other.value);
    }
    
    public Quantity subtract(Quantity other) {
        int result = this.value - other.value;
        if (result < 0) {
            throw new DomainRuleException("Insufficient quantity");
        }
        return new Quantity(result);
    }
}
```

### 3.8 Mapping Guidelines
> Direction: Map at the boundaries (Controllers, Repositories).  
> Location: Mappers belong in the Application or Infrastructure layer.  
> Approach: Use dedicated Mapper classes or MapStruct.  

```
java
// Mapper in Application Layer
@Component
public class OrderMapper {
    
    public OrderResponse toResponse(Order order) {
        return new OrderResponse(
            order.getId().value(),
            order.getStatus().name(),
            order.getTotalAmount().amount(),
            order.getTotalAmount().currency().name(),
            order.getItems().stream()
                .map(this::toItemResponse)
                .toList(),
            order.getCreatedAt()
        );
    }
    
    private OrderItemResponse toItemResponse(OrderItem item) {
        return new OrderItemResponse(
            item.getProductId().value(),
            item.getProductName(),
            item.getQuantity().value(),
            item.getUnitPrice().amount()
        );
    }
    
    public Order toDomain(CreateOrderRequest request, CustomerRepository customers) {
        var customerId = CustomerId.of(request.customerId());
        var customer = customers.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
        return Order.create(customer.getId());
    }
}

// Response DTO (Immutable Record)
public record OrderResponse(
    UUID id,
    String status,
    BigDecimal totalAmount,
    String currency,
    List<OrderItemResponse> items,
    Instant createdAt
) {}
```

### 3.9 DDD principle: clear boundaries
> Each domain/bounded context owns its model and should not directly reach into another’s internal classes, repositories, or database.​  
> Interaction should go through an integration boundary: REST/gRPC API, domain events, or an application service façade that exposes only what the other domain needs.​  

#### In a Spring Boot monolith
> You can keep domains as separate modules/packages and expose interfaces (application services, ports) that other domains call, instead of letting Domain A use Domain B’s entities or repositories directly.​  
> Technically this is still an in-process call, but conceptually it is similar to calling another service’s API: the interface is the contract, and each domain hides its internals behind that contract.​  

### 3.10 Ubiquitous Language & Naming Conventions

> Code naming must match business terminology.  
> Avoid technical suffixes in domain concepts.


| ❌ Technical Naming	| ✅ Ubiquitous Language |
|---------|----------|
| UserEntity	Customer | Member, Account |
| OrderDAO	| OrderRepository |
| ProductDTO	ProductDetails | ProductSummary|
| insertUser()	| registerCustomer()|
| updateOrderStatus()	order.ship() | order.cancel()|
| isDeleted	isArchived | isCancelled|
| createdAt	registeredAt | placedAt|
| processOrder()	fulfillOrder() | submitOrder()|
