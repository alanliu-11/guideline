## 7. Use Case / Application Service Pattern

### 7.1 Command Use Case (Write Operation)
> Use cases orchestrate business operations.  
> Commands represent write operations that modify state.

```
java
// Located in: application/usecase/

@Service
@RequiredArgsConstructor
@Slf4j
public class SubmitOrderUseCase {
    
    private final OrderRepository orders;
    private final CustomerRepository customers;
    private final InventoryService inventory;
    private final OrderMapper mapper;
    
    @Transactional
    public OrderResponse execute(SubmitOrderCommand command) {
        log.info("Submitting order for customer: {}", command.customerId());
        
        // 1. Validate customer exists
        var customerId = CustomerId.of(command.customerId());
        var customer = customers.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
        
        // 2. Create order aggregate
        var order = Order.create(customer.getId());
        
        // 3. Add items with inventory validation
        for (var item : command.items()) {
            var productId = ProductId.of(item.productId());
            var quantity = new Quantity(item.quantity());
            
            // Validate stock availability
            if (!inventory.isAvailable(productId, quantity)) {
                throw new InsufficientStockException(productId, quantity, 
                    inventory.getAvailableQuantity(productId));
            }
            
            var product = inventory.getProduct(productId);
            order.addItem(product, quantity);
        }
        
        // 4. Submit order (triggers domain event)
        order.submit();
        
        // 5. Persist
        orders.save(order);
        
        log.info("Order submitted successfully: {}", order.getId());
        return mapper.toResponse(order);
    }
}

// Command DTO
public record SubmitOrderCommand(
    @NotNull UUID customerId,
    @NotEmpty List<@Valid OrderItemCommand> items
) {}

public record OrderItemCommand(
    @NotNull UUID productId,
    @Min(1) int quantity
) {}
```

### 7.2 Query Use Case (Read Operation)
> Query use cases are read-only operations.  
> Use @Transactional(readOnly = true) for better performance.

```
java
// Located in: application/usecase/

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class GetOrderDetailsUseCase {
    
    private final OrderRepository orders;
    private final OrderMapper mapper;
    
    public OrderResponse execute(UUID orderId) {
        var id = OrderId.of(orderId);
        
        return orders.findById(id)
            .map(mapper::toResponse)
            .orElseThrow(() -> new OrderNotFoundException(id));
    }
    
    public List<OrderSummaryResponse> executeByCustomer(UUID customerId) {
        var id = CustomerId.of(customerId);
        
        return orders.findByCustomerId(id).stream()
            .map(mapper::toSummaryResponse)
            .toList();
    }
}
```

### 7.3 Port Interfaces (Optional - Stricter Hexagonal)
> Port interfaces provide stricter hexagonal architecture boundaries.  
> Input ports define application entry points, output ports define external dependencies.

```
java
// Input Port (in application/port/input/)
public interface SubmitOrderPort {
    OrderResponse submitOrder(SubmitOrderCommand command);
}

// Output Port (in application/port/output/)
public interface PaymentGatewayPort {
    PaymentResult processPayment(OrderId orderId, Money amount, PaymentMethod method);
    PaymentStatus getPaymentStatus(PaymentId paymentId);
    void refundPayment(PaymentId paymentId, Money amount);
}

// Use Case implementing Input Port
@Service
@RequiredArgsConstructor
public class SubmitOrderUseCase implements SubmitOrderPort {
    
    private final OrderRepository orders;
    private final PaymentGatewayPort paymentGateway; // Uses output port
    
    @Override
    @Transactional
    public OrderResponse submitOrder(SubmitOrderCommand command) {
        // Implementation...
    }
}

// Infrastructure Adapter implementing Output Port
@Component
@RequiredArgsConstructor
class StripePaymentGatewayAdapter implements PaymentGatewayPort {
    
    private final StripeClient stripeClient;
    private final PaymentMapper mapper;
    
    @Override
    public PaymentResult processPayment(OrderId orderId, Money amount, PaymentMethod method) {
        var request = mapper.toStripeRequest(orderId, amount, method);
        var response = stripeClient.charge(request);
        return mapper.toPaymentResult(response);
    }
}
```
