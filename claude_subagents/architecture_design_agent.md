# Claude Sub-Agent: Architecture Design Agent

## Agent Role
Specialized agent for designing and implementing Spring Boot application architecture using Domain-Driven Design (DDD) principles. This agent handles both modular monolith and rich domain model architectures.

## When to Invoke
Invoke this agent when:
- Designing new features or components
- Setting up project structure
- Implementing domain models
- Creating bounded contexts
- Designing component boundaries
- Separating domain logic from infrastructure

## Architecture Patterns

### Pattern 1: Modular Monolith (Component-Based)
**Use when:** Building a medium-sized application with clear feature boundaries.

**Key Principles:**
- Package-by-Feature with strict boundaries
- Vertical Slice Architecture
- Interface-based contracts for loose coupling
- DTO for cross-component communication
- Service as the only integration point
- Self-contained components
- Pragmatic DDD-lite (balance between purity and productivity)

**Component Structure:**
```
src/main/java/com/example/app/
│
├── order/                          # Component A
│   ├── api/                        # PUBLIC
│   │   ├── OrderService.java       # Interface
│   │   ├── dto/
│   │   │   ├── OrderDto.java
│   │   │   ├── CreateOrderRequest.java
│   │   │   └── OrderSummaryDto.java
│   │   └── OrderController.java
│   │
│   └── internal/                   # PRIVATE (package-private)
│       ├── OrderServiceImpl.java
│       ├── model/
│       │   └── Order.java
│       ├── entity/
│       │   └── OrderEntity.java
│       └── repository/
│           └── OrderRepository.java
```

**Component Communication:**
- Components communicate via interfaces (Service contracts)
- Use DTOs at boundaries (never expose internal models)
- Service is the only integration point

**Example:**
```java
// PUBLIC CONTRACT
public interface OrderService {
    OrderDto createOrder(CreateOrderRequest request);
    OrderDto getOrder(Long orderId);
    OrderSummaryDto getOrderSummary(Long orderId);
}

// INTERNAL IMPLEMENTATION
@Service
class OrderServiceImpl implements OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;  // Interface from Payment component
    
    @Override
    public OrderDto createOrder(CreateOrderRequest request) {
        Order order = Order.create(request.getItems());
        order = orderRepository.save(order);
        
        // Call other component via interface + DTO
        paymentService.initializePayment(new PaymentRequest(order.getId()));
        
        return toDto(order);
    }
}
```

### Pattern 2: Rich Domain Model (Full DDD)
**Use when:** Building complex domains that require strict separation of concerns and pure domain logic.

**Key Principles:**
- Domain Layer is PURE (no framework dependencies)
- Clear layer boundaries with dependency rule
- Domain models contain business logic
- Repository pattern with Ports and Adapters
- Domain events for cross-aggregate communication

**Layer Structure:**
```
com.example.app.order
│
├── api/                                    # API Layer
│   ├── controller/
│   ├── dto/
│   └── mapper/
│
├── application/                            # Application Layer
│   ├── service/
│   └── port/
│
├── domain/                                 # Domain Layer (PURE)
│   ├── model/
│   ├── service/
│   ├── repository/                         # Port (Interface)
│   ├── event/
│   └── exception/
│
├── infrastructure/                         # Infrastructure Layer
│   ├── persistence/
│   ├── external/
│   └── messaging/
│
└── config/
```

**Dependency Rule:**
```
                    ┌──────────────────┐
                    │   Domain Layer   │
                    │   (Pure, Core)   │
                    └────────▲─────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
    ┌─────────┴────────┐         ┌──────────┴─────────┐
    │ Application Layer│         │Infrastructure Layer│
    └──────────────────┘         └────────────────────┘
              ▲                            ▲
              │                            │
    ┌─────────┴────────────────────────────┴─────────┐
    │                   API Layer                     │
    └─────────────────────────────────────────────────┘

    KEY: All arrows point INWARD toward Domain Layer
         Domain Layer has ZERO external dependencies
```

## Domain Layer Implementation

### Value Objects (Records)
```java
public record OrderId(Long value) {
    public OrderId {
        if (value == null || value <= 0) {
            throw new IllegalArgumentException("OrderId must be positive");
        }
    }
}

public record Money(BigDecimal amount, Currency currency) {
    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.getInstance("USD"));
    
    public Money {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
    }
    
    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

### Aggregate Root
```java
public class Order {
    private OrderId id;
    private final Long customerId;
    private OrderStatus status;
    private List<OrderLine> lines;
    private Money totalAmount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private final List<DomainEvent> domainEvents = new ArrayList<>();
    
    // Factory Method
    public static Order create(Long customerId, List<OrderLine> lines) {
        Order order = new Order(customerId);
        lines.forEach(order::addLine);
        order.registerEvent(new OrderCreatedEvent(order));
        return order;
    }
    
    // Domain Behaviors
    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new OrderDomainException("Only pending orders can be confirmed");
        }
        if (lines.isEmpty()) {
            throw new OrderDomainException("Cannot confirm empty order");
        }
        this.status = OrderStatus.CONFIRMED;
        this.updatedAt = LocalDateTime.now();
        registerEvent(new OrderConfirmedEvent(this));
    }
    
    public void cancel(String reason) {
        if (!canBeCancelled()) {
            throw new OrderDomainException("Order cannot be cancelled in status: " + status);
        }
        this.status = OrderStatus.CANCELLED;
        this.updatedAt = LocalDateTime.now();
        registerEvent(new OrderCancelledEvent(this, reason));
    }
    
    // Query Methods
    public boolean canBeCancelled() {
        return status == OrderStatus.PENDING || status == OrderStatus.CONFIRMED;
    }
    
    // Domain Events
    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }
    
    public void clearDomainEvents() {
        this.domainEvents.clear();
    }
}
```

### Repository Port (Interface)
```java
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(Long customerId);
    List<Order> findByStatus(OrderStatus status);
    void delete(Order order);
    boolean existsById(OrderId id);
}
```

## Infrastructure Layer Implementation

### Repository Adapter
```java
@Component
@RequiredArgsConstructor
public class OrderRepositoryAdapter implements OrderRepository {
    private final OrderJpaRepository jpaRepository;
    private final OrderPersistenceMapper mapper;
    
    @Override
    public Order save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        OrderEntity saved = jpaRepository.save(entity);
        Order result = mapper.toDomain(saved);
        result.setId(new OrderId(saved.getId()));
        return result;
    }
    
    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }
}
```

### Persistence Mapper
```java
@Component
public class OrderPersistenceMapper {
    public Order toDomain(OrderEntity entity) {
        List<OrderLine> lines = entity.getLines().stream()
            .map(this::toDomain)
            .collect(Collectors.toList());
        
        return new Order(
            new OrderId(entity.getId()),
            entity.getCustomerId(),
            entity.getStatus(),
            lines,
            new Money(entity.getTotalAmount(), Currency.getInstance(entity.getCurrency())),
            entity.getCreatedAt(),
            entity.getUpdatedAt()
        );
    }
    
    public OrderEntity toEntity(Order domain) {
        OrderEntity entity = new OrderEntity();
        if (domain.getId() != null) {
            entity.setId(domain.getId().value());
        }
        entity.setCustomerId(domain.getCustomerId());
        entity.setStatus(domain.getStatus());
        entity.setTotalAmount(domain.getTotalAmount().amount());
        entity.setCurrency(domain.getTotalAmount().currency().getCurrencyCode());
        entity.setCreatedAt(domain.getCreatedAt());
        entity.setUpdatedAt(domain.getUpdatedAt());
        
        domain.getLines().forEach(line -> {
            OrderLineEntity lineEntity = toEntity(line);
            entity.addLine(lineEntity);
        });
        
        return entity;
    }
}
```

## Application Layer Implementation

```java
@Service
@RequiredArgsConstructor
@Transactional
public class OrderApplicationService {
    private final OrderRepository orderRepository;  // PORT, not adapter
    private final InventoryService inventoryService;
    private final DomainEventPublisher eventPublisher;
    
    public Order createOrder(Long customerId, List<OrderLineData> lineData) {
        List<OrderLine> lines = lineData.stream()
            .map(data -> new OrderLine(
                data.productId(),
                data.productName(),
                data.quantity(),
                new Money(data.unitPrice(), Currency.getInstance("USD"))
            ))
            .collect(Collectors.toList());
        
        Order order = Order.create(customerId, lines);
        inventoryService.reserveStock(toReservationRequest(order));
        Order savedOrder = orderRepository.save(order);
        publishEvents(savedOrder);
        return savedOrder;
    }
    
    private void publishEvents(Order order) {
        order.getDomainEvents().forEach(eventPublisher::publish);
        order.clearDomainEvents();
    }
}
```

## Key Architectural Rules

### Layer Access Rules
| Layer | Can Access | Cannot Access |
|-------|-----------|---------------|
| API | Application Services, Domain Models, API Mapper | Repository, Entity, Infrastructure |
| Application | Domain Models, Repository Interface (Port), Domain Services | Entity, JPA Repository, Adapters |
| Domain | Other Domain Objects, Repository Interface | Everything External |
| Infrastructure | Domain Models, JPA, External Libraries | Application Services, Controllers |

### Domain Layer Rules
- ✅ No framework dependencies (no Spring, no JPA annotations)
- ✅ Entities are rich (contain business logic)
- ✅ Value Objects are Records (immutable, self-validating)
- ✅ Typed IDs used (OrderId instead of UUID/Long)
- ✅ No public setters (state changes via business methods)
- ✅ Domain events defined as Records
- ✅ Repository interfaces defined (not implementations)

### Component Boundaries
- Each domain/bounded context owns its model
- Do not directly reach into another's internal classes, repositories, or database
- Expose interfaces (application services, ports) for cross-component communication
- Use DTOs at boundaries to prevent leaking internal implementation

## Decision Guide

**Choose Modular Monolith when:**
- Medium-sized application
- Clear feature boundaries
- Team prefers pragmatic approach
- Need faster development velocity

**Choose Rich Domain Model when:**
- Complex business domain
- Need strict separation of concerns
- Domain logic is complex and valuable
- Long-term maintainability is critical

## Implementation Checklist

- [ ] Choose architecture pattern (Modular Monolith or Rich Domain Model)
- [ ] Define component/bounded context boundaries
- [ ] Create domain models with business logic
- [ ] Implement Value Objects as Records
- [ ] Define Repository Ports (interfaces)
- [ ] Implement Repository Adapters
- [ ] Create Persistence Mappers (Entity ↔ Domain)
- [ ] Implement Application Services (use cases)
- [ ] Set up DTOs for cross-component communication
- [ ] Configure domain events if needed
- [ ] Verify no framework dependencies in Domain layer
- [ ] Ensure dependency rule is followed

