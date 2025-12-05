## 6. Repository Pattern Implementation

### 6.1 Domain Layer Interface (Port)
> Repository interfaces are defined in the domain layer.  
> Infrastructure implements these interfaces as adapters.

```
java
// Located in: domain/repository/
public interface OrderRepository {
    
    void save(Order order);
    
    Optional<Order> findById(OrderId id);
    
    List<Order> findByCustomerId(CustomerId customerId);
    
    void delete(Order order);
    
    boolean existsById(OrderId id);
}
```

### 6.2 Infrastructure Layer Implementation (Adapter)
> Spring Data JPA interfaces are internal to infrastructure.  
> Adapters bridge between domain interfaces and JPA repositories.

```
java
// Located in: infrastructure/persistence/

// Spring Data JPA Interface (internal)
interface JpaOrderRepository extends JpaRepository<Order, UUID> {
    
    @Query("SELECT o FROM Order o WHERE o.customerId = :customerId")
    List<Order> findByCustomerId(@Param("customerId") CustomerId customerId);
}

// Adapter implementing Domain Interface
@Repository
@RequiredArgsConstructor
class OrderRepositoryAdapter implements OrderRepository {
    
    private final JpaOrderRepository jpaRepository;
    
    @Override
    public void save(Order order) {
        jpaRepository.save(order);
    }
    
    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value());
    }
    
    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return jpaRepository.findByCustomerId(customerId);
    }
    
    @Override
    public void delete(Order order) {
        jpaRepository.delete(order);
    }
    
    @Override
    public boolean existsById(OrderId id) {
        return jpaRepository.existsById(id.value());
    }
}
```

### 6.3 Separated Persistence Model (Optional Advanced Pattern)
> For complex domains, consider separating Domain Entities from JPA Entities.  
> Use mappers to convert between domain and persistence models.

```
java
// Domain Entity (Pure - in domain/model/)
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private Money totalAmount;
    private OrderStatus status;
    private final List<OrderItem> items;
    
    // Business logic methods...
}

// JPA Entity (Infrastructure - in infrastructure/persistence/)
@Entity
@Table(name = "orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
class OrderJpaEntity {
    
    @Id
    private UUID id;
    
    @Column(name = "customer_id", nullable = false)
    private UUID customerId;
    
    @Column(name = "total_amount", nullable = false)
    private BigDecimal totalAmount;
    
    @Column(name = "currency", nullable = false)
    @Enumerated(EnumType.STRING)
    private Currency currency;
    
    @Column(name = "status", nullable = false)
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderItemJpaEntity> items = new ArrayList<>();
}

// Mapper between Domain and JPA Entity
@Component
class OrderPersistenceMapper {
    
    Order toDomain(OrderJpaEntity entity) {
        return Order.reconstitute(
            new OrderId(entity.getId()),
            new CustomerId(entity.getCustomerId()),
            new Money(entity.getTotalAmount(), entity.getCurrency()),
            entity.getStatus(),
            entity.getItems().stream()
                .map(this::toItemDomain)
                .toList()
        );
    }
    
    OrderJpaEntity toJpaEntity(Order order) {
        var entity = new OrderJpaEntity();
        entity.setId(order.getId().value());
        entity.setCustomerId(order.getCustomerId().value());
        entity.setTotalAmount(order.getTotalAmount().amount());
        entity.setCurrency(order.getTotalAmount().currency());
        entity.setStatus(order.getStatus());
        entity.setItems(order.getItems().stream()
            .map(this::toItemJpaEntity)
            .toList());
        return entity;
    }
}

// Repository Adapter with Mapping
@Repository
@RequiredArgsConstructor
class OrderRepositoryAdapter implements OrderRepository {
    
    private final JpaOrderRepository jpaRepository;
    private final OrderPersistenceMapper mapper;
    
    @Override
    public void save(Order order) {
        var entity = mapper.toJpaEntity(order);
        jpaRepository.save(entity);
    }
    
    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }
}
```
