## 3. Modular Monolith / Component-Based Architecture
- Package-by-Feature with strict boundaries
- Vertical Slice Architecture (partial implementation)

- Interface-based contracts	Loose coupling between components
- DTO for cross-component communication	Prevents leaking internal implementation
- Service as the only integration point	Single entry point, easier to control
- Self-contained components	Clear ownership and boundaries
- Pragmatic DDD-lite	Balance between purity and productivity

### 3.1 Your Architecture Summary
```
┌─────────────────────────────────────────────────────────────────────┐
│                         SPRING BOOT APPLICATION                     │
│                                                                     │
│  ┌─────────────────────┐         ┌─────────────────────┐            │
│  │    Component A      │         │    Component B      │            │
│  │  ┌───────────────┐  │         │  ┌───────────────┐  │            │
│  │  │  Controller   │  │         │  │  Controller   │  │            │
│  │  └───────┬───────┘  │         │  └───────┬───────┘  │            │
│  │          │          │         │          │          │            │
│  │  ┌───────▼───────┐  │   DTO   │  ┌───────▼───────┐  │            │
│  │  │   Interface   │◄─┼─────────┼──┤   Interface   │  │            │
│  │  └───────┬───────┘  │         │  └───────┬───────┘  │            │
│  │          │          │         │          │          │            │
│  │  ┌───────▼───────┐  │         │  ┌───────▼───────┐  │            │
│  │  │    Service    │  │         │  │    Service    │  │            │
│  │  └───┬───────┬───┘  │         │  └───┬───────┬───┘  │            │
│  │      │       │      │         │      │       │      │            │
│  │  ┌───▼───┐ ┌─▼────┐ │         │  ┌───▼───┐ ┌─▼────┐ │            │
│  │  │ Model │ │Entity│ │         │  │ Model │ │Entity│ │            │
│  │  └───────┘ └──┬───┘ │         │  └───────┘ └──┬───┘ │            │
│  │               │     │         │               │     │            │
│  │         ┌─────▼───┐ │         │         ┌─────▼───┐ │            │
│  │         │Repository│         │         │Repository│ │            │
│  │         └─────────┘ │         │         └─────────┘ │            │
│  └─────────────────────┘         └─────────────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Using DTO as at Boundaries

```
✅ GOOD: Using DTOs at Boundaries
┌─────────────────┐         ┌─────────────────┐
│   Component A   │         │   Component B   │
│                 │         │                 │
│  ServiceA       │         │  ServiceB       │
│     │           │         │     │           │
│     ▼           │         │     │           │
│  ModelA         │         │  Uses           │
│     │           │         │  ComponentADto  │
│     ▼           │   DTO   │     ▲           │
│  Interface ─────┼────────►│─────┘           │
└─────────────────┘         └─────────────────┘

Benefits:
• Contract is explicit and stable
• Internal changes don't leak
• Easy to version/evolve
```

### 3.3 Recommended Architecture Pattern
```
// ═══════════════════════════════════════════════════════════
// COMPONENT A: ORDER
// ═══════════════════════════════════════════════════════════

// 1. PUBLIC CONTRACT (what others see)
// ───────────────────────────────────
public interface OrderService {
    OrderDto createOrder(CreateOrderRequest request);
    OrderDto getOrder(Long orderId);
    OrderSummaryDto getOrderSummary(Long orderId);
}

// DTOs - Stable contract
public record OrderDto(
    Long id,
    String status,
    BigDecimal totalAmount,
    LocalDateTime createdAt
) {}

public record OrderSummaryDto(
    Long id,
    String status,
    int itemCount
) {}

// 2. INTERNAL IMPLEMENTATION (hidden from others)
// ───────────────────────────────────────────────
@Service
class OrderServiceImpl implements OrderService {
    
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;  // Interface from Payment component
    
    @Override
    public OrderDto createOrder(CreateOrderRequest request) {
        // Use internal model
        Order order = Order.create(request.getItems());
        order = orderRepository.save(order);
        
        // Call other component via interface + DTO
        paymentService.initializePayment(new PaymentRequest(order.getId()));
        
        // Return DTO, not model
        return toDto(order);
    }
    
    private OrderDto toDto(Order order) {
        return new OrderDto(
            order.getId(),
            order.getStatus().name(),
            order.calculateTotal(),
            order.getCreatedAt()
        );
    }
}

// Internal Model - Rich domain logic
class Order {
    private Long id;
    private OrderStatus status;
    private List<OrderItem> items;
    private LocalDateTime createdAt;
    
    // Factory method
    public static Order create(List<ItemRequest> items) {
        Order order = new Order();
        order.status = OrderStatus.PENDING;
        order.items = items.stream()
            .map(OrderItem::fromRequest)
            .toList();
        order.createdAt = LocalDateTime.now();
        return order;
    }
    
    // Business logic stays internal
    public void confirm() {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("Can only confirm pending orders");
        }
        this.status = OrderStatus.CONFIRMED;
    }
    
    public BigDecimal calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// Entity - Persistence concern
@Entity
@Table(name = "orders")
class OrderEntity {
    @Id @GeneratedValue
    private Long id;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItemEntity> items;
    
    private LocalDateTime createdAt;
}

```

### 3.4 Project Structure

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
│
├── payment/                        # Component B
│   ├── api/
│   │   ├── PaymentService.java
│   │   └── dto/
│   │       └── PaymentRequest.java
│   └── internal/
│       └── ...
│
└── shipping/                       # Component C
    ├── api/
    └── internal/
```


### 3.5 Others

- Constraint: Do not use Lombok @Data on @Entity classes.  
- Each domain/bounded context owns its model and should not directly reach into another’s internal classes, repositories, or database.​  
- You can keep domains as separate modules/packages and expose interfaces (application services, ports) that other domains call, instead of letting Domain A use Domain B’s entities or repositories directly.​  
- Technically this is still an in-process call, but conceptually it is similar to calling another service’s API: the interface is the contract, and each domain hides its internals behind that contract.​  

