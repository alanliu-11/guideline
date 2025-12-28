# Architecture Overview

```
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Application                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         API Layer                                      │  │
│  │                  (Controllers, Request/Response DTOs)                  │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
│                                  │                                           │
│  ┌───────────────────────────────▼───────────────────────────────────────┐  │
│  │                      Application Layer                                 │  │
│  │                (Application Services / Use Cases)                      │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
│                                  │                                           │
│  ┌───────────────────────────────▼───────────────────────────────────────┐  │
│  │                         Domain Layer                                   │  │
│  │    ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐      │  │
│  │    │   Domain    │    │   Domain    │    │   Repository Port   │      │  │
│  │    │   Models    │    │  Services   │    │    (Interface)      │      │  │
│  │    └─────────────┘    └─────────────┘    └──────────┬──────────┘      │  │
│  └─────────────────────────────────────────────────────┼─────────────────┘  │
│                                                        │                     │
│  ┌─────────────────────────────────────────────────────▼─────────────────┐  │
│  │                      Infrastructure Layer                              │  │
│  │    ┌─────────────────────┐    ┌─────────────┐    ┌─────────────┐      │  │
│  │    │  Repository Adapter │    │   Entity    │    │   External  │      │  │
│  │    │  (Implementation)   │    │   Models    │    │   Services  │      │  │
│  │    └─────────────────────┘    └─────────────┘    └─────────────┘      │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Dependency Rule
```
text
                    ┌──────────────────┐
                    │   Domain Layer   │
                    │   (Pure, Core)   │
                    │                  │
                    │  - Domain Models │
                    │  - Domain Services│
                    │  - Repository    │
                    │    Interfaces    │
                    └────────▲─────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
    ┌─────────┴────────┐         ┌──────────┴─────────┐
    │ Application Layer│         │Infrastructure Layer│
    │                  │         │                    │
    │ - App Services   │         │ - Repo Impls       │
    │ - Use Cases      │         │ - Entities         │
    │ - Orchestration  │         │ - Adapters         │
    └──────────────────┘         └────────────────────┘
              ▲                            ▲
              │                            │
    ┌─────────┴────────────────────────────┴─────────┐
    │                   API Layer                     │
    │         (Controllers, DTOs, Mappers)           │
    └─────────────────────────────────────────────────┘

    ════════════════════════════════════════════════
    KEY: All arrows point INWARD toward Domain Layer
         Domain Layer has ZERO external dependencies
    ════════════════════════════════════════════════

```    

## Complete Package Structure
```
text
com.example.app.order
│
├── api/                                    # API Layer
│   ├── controller/
│   │   └── OrderController.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateOrderRequest.java
│   │   │   └── CancelOrderRequest.java
│   │   └── response/
│   │       ├── OrderResponse.java
│   │       └── OrderSummaryResponse.java
│   └── mapper/
│       └── OrderApiMapper.java            # DTO ↔ Domain
│
├── application/                            # Application Layer
│   ├── service/
│   │   ├── OrderApplicationService.java   # Use case orchestration
│   │   └── OrderQueryService.java         # Read-only queries
│   └── port/
│       └── out/                            # Output ports (optional)
│           └── PaymentGatewayPort.java
│
├── domain/                                 # Domain Layer (PURE)
│   ├── model/
│   │   ├── Order.java                     # Aggregate Root
│   │   ├── OrderLine.java                 # Entity
│   │   ├── OrderId.java                   # Value Object
│   │   ├── Money.java                     # Value Object
│   │   └── OrderStatus.java               # Enum
│   ├── service/
│   │   └── OrderPricingService.java       # Domain Service
│   ├── repository/
│   │   └── OrderRepository.java           # Port (Interface)
│   ├── event/
│   │   ├── OrderCreatedEvent.java
│   │   └── OrderCancelledEvent.java
│   └── exception/
│       ├── OrderNotFoundException.java
│       └── OrderDomainException.java
│
├── infrastructure/                         # Infrastructure Layer
│   ├── persistence/
│   │   ├── entity/
│   │   │   ├── OrderEntity.java
│   │   │   └── OrderLineEntity.java
│   │   ├── repository/
│   │   │   ├── OrderJpaRepository.java    # Spring Data JPA
│   │   │   └── OrderRepositoryAdapter.java # Adapter (implements Port)
│   │   └── mapper/
│   │       └── OrderPersistenceMapper.java # Entity ↔ Domain
│   ├── external/
│   │   └── PaymentGatewayAdapter.java
│   └── messaging/
│       ├── OrderEventPublisher.java
│       └── OrderEventListener.java
│
└── config/
    └── OrderDomainConfig.java
```

## Complete Code Implementation
1. Domain Layer (Pure - No Dependencies)
```
java
// =============== domain/model/OrderId.java ===============
public record OrderId(Long value) {
    public OrderId {
        if (value == null || value <= 0) {
            throw new IllegalArgumentException("OrderId must be positive");
        }
    }
}

// =============== domain/model/Money.java ===============
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
    
    public Money multiply(int quantity) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(quantity)), this.currency);
    }
    
    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
    }
}

// =============== domain/model/OrderLine.java ===============
public class OrderLine {
    private Long id;
    private final Long productId;
    private final String productName;
    private final int quantity;
    private final Money unitPrice;
    
    public OrderLine(Long productId, String productName, int quantity, Money unitPrice) {
        if (quantity <= 0) {
            throw new OrderDomainException("Quantity must be positive");
        }
        this.productId = productId;
        this.productName = productName;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }
    
    public Money calculateLineTotal() {
        return unitPrice.multiply(quantity);
    }
    
    // Getters only - immutable
    public Long getId() { return id; }
    public Long getProductId() { return productId; }
    public String getProductName() { return productName; }
    public int getQuantity() { return quantity; }
    public Money getUnitPrice() { return unitPrice; }
}

// =============== domain/model/Order.java (Aggregate Root) ===============
public class Order {
    private OrderId id;
    private final Long customerId;
    private OrderStatus status;
    private List<OrderLine> lines;
    private Money totalAmount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // Domain events to be published
    private final List<DomainEvent> domainEvents = new ArrayList<>();
    
    // ===== Factory Method =====
    public static Order create(Long customerId, List<OrderLine> lines) {
        Order order = new Order(customerId);
        lines.forEach(order::addLine);
        order.registerEvent(new OrderCreatedEvent(order));
        return order;
    }
    
    // Private constructor - use factory method
    private Order(Long customerId) {
        if (customerId == null) {
            throw new OrderDomainException("Customer ID is required");
        }
        this.customerId = customerId;
        this.status = OrderStatus.PENDING;
        this.lines = new ArrayList<>();
        this.totalAmount = Money.ZERO;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }
    
    // Reconstitution constructor (for repository)
    public Order(OrderId id, Long customerId, OrderStatus status, 
                 List<OrderLine> lines, Money totalAmount,
                 LocalDateTime createdAt, LocalDateTime updatedAt) {
        this.id = id;
        this.customerId = customerId;
        this.status = status;
        this.lines = new ArrayList<>(lines);
        this.totalAmount = totalAmount;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }
    
    // ===== Domain Behaviors =====
    
    public void addLine(OrderLine line) {
        if (status != OrderStatus.PENDING) {
            throw new OrderDomainException("Cannot modify order in status: " + status);
        }
        this.lines.add(line);
        recalculateTotal();
    }
    
    public void removeLine(Long productId) {
        if (status != OrderStatus.PENDING) {
            throw new OrderDomainException("Cannot modify order in status: " + status);
        }
        this.lines.removeIf(line -> line.getProductId().equals(productId));
        recalculateTotal();
    }
    
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
    
    public void markAsPaid() {
        if (status != OrderStatus.CONFIRMED) {
            throw new OrderDomainException("Only confirmed orders can be paid");
        }
        this.status = OrderStatus.PAID;
        this.updatedAt = LocalDateTime.now();
        registerEvent(new OrderPaidEvent(this));
    }
    
    public void ship() {
        if (status != OrderStatus.PAID) {
            throw new OrderDomainException("Only paid orders can be shipped");
        }
        this.status = OrderStatus.SHIPPED;
        this.updatedAt = LocalDateTime.now();
    }
    
    // ===== Query Methods =====
    
    public boolean canBeCancelled() {
        return status == OrderStatus.PENDING || status == OrderStatus.CONFIRMED;
    }
    
    public boolean isPaid() {
        return status == OrderStatus.PAID || status == OrderStatus.SHIPPED;
    }
    
    public int getTotalItems() {
        return lines.stream().mapToInt(OrderLine::getQuantity).sum();
    }
    
    // ===== Private Methods =====
    
    private void recalculateTotal() {
        this.totalAmount = lines.stream()
            .map(OrderLine::calculateLineTotal)
            .reduce(Money.ZERO, Money::add);
    }
    
    private void registerEvent(DomainEvent event) {
        this.domainEvents.add(event);
    }
    
    // ===== Domain Events =====
    
    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }
    
    public void clearDomainEvents() {
        this.domainEvents.clear();
    }
    
    // ===== Getters =====
    public OrderId getId() { return id; }
    public Long getCustomerId() { return customerId; }
    public OrderStatus getStatus() { return status; }
    public List<OrderLine> getLines() { return Collections.unmodifiableList(lines); }
    public Money getTotalAmount() { return totalAmount; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    
    // Used by repository adapter
    public void setId(OrderId id) { this.id = id; }
}

// =============== domain/repository/OrderRepository.java (PORT) ===============
public interface OrderRepository {
    
    Order save(Order order);
    
    Optional<Order> findById(OrderId id);
    
    List<Order> findByCustomerId(Long customerId);
    
    List<Order> findByStatus(OrderStatus status);
    
    void delete(Order order);
    
    boolean existsById(OrderId id);
}

// =============== domain/service/OrderPricingService.java ===============
public class OrderPricingService {
    
    private final DiscountPolicy discountPolicy;
    private final TaxCalculator taxCalculator;
    
    public OrderPricingService(DiscountPolicy discountPolicy, TaxCalculator taxCalculator) {
        this.discountPolicy = discountPolicy;
        this.taxCalculator = taxCalculator;
    }
    
    public Money calculateFinalPrice(Order order, Customer customer) {
        Money subtotal = order.getTotalAmount();
        Money discount = discountPolicy.calculateDiscount(order, customer);
        Money tax = taxCalculator.calculateTax(subtotal.subtract(discount));
        return subtotal.subtract(discount).add(tax);
    }
}
```

2. Infrastructure Layer (Adapter Implementation)
```
java
// =============== infrastructure/persistence/entity/OrderEntity.java ===============
@Entity
@Table(name = "orders")
@Getter
@Setter
@NoArgsConstructor
public class OrderEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private Long customerId;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;
    
    @Column(precision = 19, scale = 2)
    private BigDecimal totalAmount;
    
    @Column(length = 3)
    private String currency;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    private LocalDateTime updatedAt;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLineEntity> lines = new ArrayList<>();
    
    public void addLine(OrderLineEntity line) {
        lines.add(line);
        line.setOrder(this);
    }
}

// =============== infrastructure/persistence/entity/OrderLineEntity.java ===============
@Entity
@Table(name = "order_lines")
@Getter
@Setter
@NoArgsConstructor
public class OrderLineEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private OrderEntity order;
    
    @Column(nullable = false)
    private Long productId;
    
    private String productName;
    
    @Column(nullable = false)
    private Integer quantity;
    
    @Column(precision = 19, scale = 2)
    private BigDecimal unitPrice;
    
    @Column(length = 3)
    private String currency;
}

// =============== infrastructure/persistence/repository/OrderJpaRepository.java ===============
@Repository
public interface OrderJpaRepository extends JpaRepository<OrderEntity, Long> {
    
    List<OrderEntity> findByCustomerId(Long customerId);
    
    List<OrderEntity> findByStatus(OrderStatus status);
}

// =============== infrastructure/persistence/mapper/OrderPersistenceMapper.java ===============
@Component
public class OrderPersistenceMapper {
    
    // ===== Entity → Domain =====
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
    
    public OrderLine toDomain(OrderLineEntity entity) {
        OrderLine line = new OrderLine(
            entity.getProductId(),
            entity.getProductName(),
            entity.getQuantity(),
            new Money(entity.getUnitPrice(), Currency.getInstance(entity.getCurrency()))
        );
        // Set ID via reflection or package-private setter
        setId(line, entity.getId());
        return line;
    }
    
    // ===== Domain → Entity =====
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
    
    public OrderLineEntity toEntity(OrderLine domain) {
        OrderLineEntity entity = new OrderLineEntity();
        entity.setId(domain.getId());
        entity.setProductId(domain.getProductId());
        entity.setProductName(domain.getProductName());
        entity.setQuantity(domain.getQuantity());
        entity.setUnitPrice(domain.getUnitPrice().amount());
        entity.setCurrency(domain.getUnitPrice().currency().getCurrencyCode());
        return entity;
    }
    
    private void setId(OrderLine line, Long id) {
        try {
            Field field = OrderLine.class.getDeclaredField("id");
            field.setAccessible(true);
            field.set(line, id);
        } catch (Exception e) {
            throw new RuntimeException("Failed to set OrderLine id", e);
        }
    }
}

// =============== infrastructure/persistence/repository/OrderRepositoryAdapter.java ===============
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
        result.setId(new OrderId(saved.getId()));  // Ensure ID is set
        return result;
    }
    
    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }
    
    @Override
    public List<Order> findByCustomerId(Long customerId) {
        return jpaRepository.findByCustomerId(customerId).stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
    }
    
    @Override
    public List<Order> findByStatus(OrderStatus status) {
        return jpaRepository.findByStatus(status).stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
    }
    
    @Override
    public void delete(Order order) {
        jpaRepository.deleteById(order.getId().value());
    }
    
    @Override
    public boolean existsById(OrderId id) {
        return jpaRepository.existsById(id.value());
    }
}
```

3. Application Layer (Use Cases)
```
java
// =============== application/service/OrderApplicationService.java ===============
@Service
@RequiredArgsConstructor
@Transactional
public class OrderApplicationService {
    
    // Depends on PORT (interface), NOT adapter
    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;  // Cross-domain interface
    private final DomainEventPublisher eventPublisher;
    
    public Order createOrder(Long customerId, List<OrderLineData> lineData) {
        // Convert input to domain objects
        List<OrderLine> lines = lineData.stream()
            .map(data -> new OrderLine(
                data.productId(),
                data.productName(),
                data.quantity(),
                new Money(data.unitPrice(), Currency.getInstance("USD"))
            ))
            .collect(Collectors.toList());
        
        // Create domain object via factory method
        Order order = Order.create(customerId, lines);
        
        // Cross-domain call (same transaction)
        inventoryService.reserveStock(toReservationRequest(order));
        
        // Save via repository (uses adapter internally)
        Order savedOrder = orderRepository.save(order);
        
        // Publish domain events
        publishEvents(savedOrder);
        
        return savedOrder;
    }
    
    public Order confirmOrder(Long orderId) {
        Order order = findOrderOrThrow(orderId);
        
        // Domain logic in aggregate
        order.confirm();
        
        Order savedOrder = orderRepository.save(order);
        publishEvents(savedOrder);
        
        return savedOrder;
    }
    
    public Order cancelOrder(Long orderId, String reason) {
        Order order = findOrderOrThrow(orderId);
        
        // Domain logic
        order.cancel(reason);
        
        // Cross-domain call
        inventoryService.releaseReservation(orderId);
        
        Order savedOrder = orderRepository.save(order);
        publishEvents(savedOrder);
        
        return savedOrder;
    }
    
    public void processPayment(Long orderId, PaymentDetails paymentDetails) {
        Order order = findOrderOrThrow(orderId);
        
        // Domain logic
        order.markAsPaid();
        
        orderRepository.save(order);
        publishEvents(order);
    }
    
    // ===== Private Helpers =====
    
    private Order findOrderOrThrow(Long orderId) {
        return orderRepository.findById(new OrderId(orderId))
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
    
    private void publishEvents(Order order) {
        order.getDomainEvents().forEach(eventPublisher::publish);
        order.clearDomainEvents();
    }
    
    private ReservationRequest toReservationRequest(Order order) {
        // Convert to cross-domain DTO
        return new ReservationRequest(
            order.getId() != null ? order.getId().value() : null,
            order.getLines().stream()
                .map(line -> new ReservationItem(line.getProductId(), line.getQuantity()))
                .collect(Collectors.toList())
        );
    }
}

// =============== application/service/OrderQueryService.java ===============
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OrderQueryService {
    
    private final OrderRepository orderRepository;
    
    public Optional<Order> findById(Long orderId) {
        return orderRepository.findById(new OrderId(orderId));
    }
    
    public List<Order> findByCustomer(Long customerId) {
        return orderRepository.findByCustomerId(customerId);
    }
    
    public List<Order> findPendingOrders() {
        return orderRepository.findByStatus(OrderStatus.PENDING);
    }
}
4. API Layer (Controllers + DTOs)
java
// =============== api/dto/request/CreateOrderRequest.java ===============
public record CreateOrderRequest(
    @NotNull Long customerId,
    @NotEmpty List<OrderLineRequest> lines
) {}

public record OrderLineRequest(
    @NotNull Long productId,
    @NotBlank String productName,
    @Min(1) int quantity,
    @NotNull @Positive BigDecimal unitPrice
) {}

// =============== api/dto/response/OrderResponse.java ===============
public record OrderResponse(
    @JsonProperty("order_id") Long id,
    Long customerId,
    String status,
    List<OrderLineResponse> lines,
    BigDecimal totalAmount,
    String currency,
    LocalDateTime createdAt,
    LocalDateTime updatedAt,
    
    // Computed fields
    int totalItems,
    boolean canBeCancelled
) {}

public record OrderLineResponse(
    Long id,
    Long productId,
    String productName,
    int quantity,
    BigDecimal unitPrice,
    BigDecimal lineTotal
) {}

// =============== api/mapper/OrderApiMapper.java ===============
@Component
public class OrderApiMapper {
    
    public OrderResponse toResponse(Order domain) {
        return new OrderResponse(
            domain.getId().value(),
            domain.getCustomerId(),
            domain.getStatus().name(),
            toLineResponses(domain.getLines()),
            domain.getTotalAmount().amount(),
            domain.getTotalAmount().currency().getCurrencyCode(),
            domain.getCreatedAt(),
            domain.getUpdatedAt(),
            domain.getTotalItems(),          // From domain logic
            domain.canBeCancelled()          // From domain logic
        );
    }
    
    public List<OrderLineResponse> toLineResponses(List<OrderLine> lines) {
        return lines.stream()
            .map(this::toLineResponse)
            .collect(Collectors.toList());
    }
    
    public OrderLineResponse toLineResponse(OrderLine line) {
        return new OrderLineResponse(
            line.getId(),
            line.getProductId(),
            line.getProductName(),
            line.getQuantity(),
            line.getUnitPrice().amount(),
            line.calculateLineTotal().amount()
        );
    }
    
    public List<OrderLineData> toLineData(List<OrderLineRequest> requests) {
        return requests.stream()
            .map(r -> new OrderLineData(r.productId(), r.productName(), 
                                         r.quantity(), r.unitPrice()))
            .collect(Collectors.toList());
    }
}

// =============== api/controller/OrderController.java ===============
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {
    
    private final OrderApplicationService orderService;
    private final OrderQueryService queryService;
    private final OrderApiMapper mapper;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {
        
        Order order = orderService.createOrder(
            request.customerId(),
            mapper.toLineData(request.lines())
        );
        
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(mapper.toResponse(order));
    }
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable Long orderId) {
        return queryService.findById(orderId)
            .map(mapper::toResponse)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping("/{orderId}/confirm")
    public ResponseEntity<OrderResponse> confirmOrder(@PathVariable Long orderId) {
        Order order = orderService.confirmOrder(orderId);
        return ResponseEntity.ok(mapper.toResponse(order));
    }
    
    @PostMapping("/{orderId}/cancel")
    public ResponseEntity<OrderResponse> cancelOrder(
            @PathVariable Long orderId,
            @RequestBody CancelOrderRequest request) {
        
        Order order = orderService.cancelOrder(orderId, request.reason());
        return ResponseEntity.ok(mapper.toResponse(order));
    }
}
```

## Layer Dependency Diagram
```
text
┌─────────────────────────────────────────────────────────────────────────┐
│                           DEPENDENCY FLOW                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────┐         ┌─────────────────────┐                       │
│   │ Controller  │────────►│ ApplicationService  │                       │
│   │  (API)      │         │   (Use Cases)       │                       │
│   └──────┬──────┘         └──────────┬──────────┘                       │
│          │                           │                                   │
│          │                           │ depends on                        │
│          ▼                           ▼                                   │
│   ┌─────────────┐         ┌─────────────────────┐                       │
│   │ ApiMapper   │         │ OrderRepository     │◄─────┐                │
│   │ (DTO↔Domain)│         │ (PORT/Interface)    │      │                │
│   └──────┬──────┘         └─────────────────────┘      │                │
│          │                           ▲                  │                │
│          │                           │                  │                │
│          ▼                  ┌────────┴────────┐        │                │
│   ┌─────────────┐          │   DOMAIN LAYER   │        │ implements     │
│   │   Domain    │          │   (Pure Core)    │        │                │
│   │   Model     │◄─────────┤                  │        │                │
│   │  (Order)    │          │  - Order         │        │                │
│   └─────────────┘          │  - OrderLine     │        │                │
│                            │  - Money (VO)    │        │                │
│                            └──────────────────┘        │                │
│                                                        │                │
│   ┌────────────────────────────────────────────────────┴──────────┐    │
│   │                    INFRASTRUCTURE LAYER                        │    │
│   │  ┌─────────────────────┐    ┌─────────────────────────────┐   │    │
│   │  │ RepositoryAdapter   │───►│ JpaRepository + Entity      │   │    │
│   │  │ (implements PORT)   │    │ (Spring Data JPA)           │   │    │
│   │  └─────────────────────┘    └─────────────────────────────┘   │    │
│   │              │                                                 │    │
│   │              ▼                                                 │    │
│   │  ┌─────────────────────┐                                      │    │
│   │  │ PersistenceMapper   │                                      │    │
│   │  │ (Entity ↔ Domain)   │                                      │    │
│   │  └─────────────────────┘                                      │    │
│   └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## What Each Layer Can Access
 - Layer	Can Access	Cannot Access
 - API	Application Services, Domain Models, API Mapper	Repository, Entity, Infrastructure
 - Application	Domain Models, Repository Interface (Port), Domain Services	Entity, JPA Repository, Adapters
 - Domain	Other Domain Objects, Repository Interface	Everything External
 - Infrastructure	Domain Models, JPA, External Libraries	Application Services, Controllers


## Key Benefits of This Architecture
```
text
✅ Domain Model is PURE
   - No JPA annotations
   - No Jackson annotations  
   - No framework dependencies
   - Easy to unit test

✅ Clear Boundaries
   - Service layer never sees Entity
   - Service layer never sees JPA Repository
   - Domain logic stays in Domain layer

✅ Flexible Persistence
   - Can switch from JPA to MongoDB easily
   - Just create new Adapter implementing same Port

✅ Testable
   - Mock Repository Port for unit tests
   - Domain model tested in complete isolation
```   
## Testing Example
```
java
// Unit test - Domain model (no mocks needed)
class OrderTest {
    
    @Test
    void shouldCalculateTotalCorrectly() {
        List<OrderLine> lines = List.of(
            new OrderLine(1L, "Product A", 2, new Money(new BigDecimal("10.00"), USD)),
            new OrderLine(2L, "Product B", 3, new Money(new BigDecimal("5.00"), USD))
        );
        
        Order order = Order.create(1L, lines);
        
        assertThat(order.getTotalAmount())
            .isEqualTo(new Money(new BigDecimal("35.00"), USD));
    }
    
    @Test
    void shouldNotAllowCancellationOfShippedOrder() {
        Order order = createShippedOrder();
        
        assertThatThrownBy(() -> order.cancel("Customer request"))
            .isInstanceOf(OrderDomainException.class)
            .hasMessageContaining("cannot be cancelled");
    }
}

// Integration test - Application service with mock repository
@ExtendWith(MockitoExtension.class)
class OrderApplicationServiceTest {
    
    @Mock
    private OrderRepository orderRepository;  // Mock the PORT
    
    @InjectMocks
    private OrderApplicationService service;
    
    @Test
    void shouldCreateOrderSuccessfully() {
        // Given
        when(orderRepository.save(any(Order.class)))
            .thenAnswer(inv -> {
                Order order = inv.getArgument(0);
                order.setId(new OrderId(1L));
                return order;
            });
        
        // When
        Order result = service.createOrder(1L, createLineData());
        
        // Then
        assertThat(result.getId()).isNotNull();
        verify(orderRepository).save(any(Order.class));
    }
}
```