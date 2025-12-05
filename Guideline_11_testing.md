## 11. Testing Guidelines

### 11.1 Unit Testing (Domain Layer)
> Domain tests should be framework-free.  
> Test business logic in isolation without Spring or JPA.

```
java
// Test Domain Logic without frameworks
class OrderTest {
    
    @Test
    void shouldAddItemToDraftOrder() {
        // Given
        var order = Order.create(CustomerId.generate());
        var product = new Product(ProductId.generate(), "Widget", Money.of(10, Currency.USD));
        var quantity = new Quantity(2);
        
        // When
        order.addItem(product, quantity);
        
        // Then
        assertThat(order.getItems()).hasSize(1);
        assertThat(order.getTotalAmount()).isEqualTo(Money.of(20, Currency.USD));
    }
    
    @Test
    void shouldNotAllowAddingItemToSubmittedOrder() {
        // Given
        var order = Order.create(CustomerId.generate());
        order.addItem(createSampleProduct(), new Quantity(1));
        order.submit();
        
        // When / Then
        assertThatThrownBy(() -> order.addItem(createSampleProduct(), new Quantity(1)))
            .isInstanceOf(OrderModificationException.class)
            .hasMessageContaining("Cannot modify a submitted order");
    }
    
    @Test
    void shouldNotSubmitEmptyOrder() {
        // Given
        var order = Order.create(CustomerId.generate());
        
        // When / Then
        assertThatThrownBy(order::submit)
            .isInstanceOf(OrderSubmissionException.class)
            .hasMessageContaining("Cannot submit empty order");
    }
}

class MoneyTest {
    
    @Test
    void shouldAddMoneyWithSameCurrency() {
        // Given
        var money1 = Money.of(10, Currency.USD);
        var money2 = Money.of(20, Currency.USD);
        
        // When
        var result = money1.add(money2);
        
        // Then
        assertThat(result).isEqualTo(Money.of(30, Currency.USD));
    }
    
    @Test
    void shouldNotAddMoneyWithDifferentCurrency() {
        // Given
        var usd = Money.of(10, Currency.USD);
        var eur = Money.of(20, Currency.EUR);
        
        // When / Then
        assertThatThrownBy(() -> usd.add(eur))
            .isInstanceOf(CurrencyMismatchException.class);
    }
    
    @Test
    void shouldNotAllowNegativeAmount() {
        assertThatThrownBy(() -> Money.of(-10, Currency.USD))
            .isInstanceOf(DomainRuleException.class)
            .hasMessageContaining("cannot be negative");
    }
}
```

### 11.2 Use Case Testing (Application Layer)
> Mock dependencies and test use case orchestration logic.  
> Verify interactions with repositories and external services.

```
java
@ExtendWith(MockitoExtension.class)
class SubmitOrderUseCaseTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @Mock
    private CustomerRepository customerRepository;
    
    @Mock
    private InventoryPort inventoryPort;
    
    @InjectMocks
    private SubmitOrderUseCase useCase;
    
    @Test
    void shouldSubmitOrderSuccessfully() {
        // Given
        var customerId = CustomerId.generate();
        var productId = ProductId.generate();
        var command = new SubmitOrderCommand(
            customerId.value(),
            List.of(new OrderItemCommand(productId.value(), 2))
        );
        
        var customer = CustomerMother.aValidCustomer(customerId);
        var product = ProductMother.aValidProduct(productId, Money.of(25, Currency.USD));
        
        when(customerRepository.findById(customerId)).thenReturn(Optional.of(customer));
        when(inventoryPort.isAvailable(productId, new Quantity(2))).thenReturn(true);
        when(inventoryPort.getProduct(productId)).thenReturn(product);
        
        // When
        var response = useCase.execute(command);
        
        // Then
        assertThat(response).isNotNull();
        assertThat(response.status()).isEqualTo("SUBMITTED");
        assertThat(response.totalAmount()).isEqualByComparingTo(BigDecimal.valueOf(50));
        
        verify(orderRepository).save(any(Order.class));
    }
    
    @Test
    void shouldThrowExceptionWhenCustomerNotFound() {
        // Given
        var command = new SubmitOrderCommand(
            UUID.randomUUID(),
            List.of(new OrderItemCommand(UUID.randomUUID(), 1))
        );
        
        when(customerRepository.findById(any())).thenReturn(Optional.empty());
        
        // When / Then
        assertThatThrownBy(() -> useCase.execute(command))
            .isInstanceOf(CustomerNotFoundException.class);
        
        verify(orderRepository, never()).save(any());
    }
    
    @Test
    void shouldThrowExceptionWhenInsufficientStock() {
        // Given
        var customerId = CustomerId.generate();
        var productId = ProductId.generate();
        var command = new SubmitOrderCommand(
            customerId.value(),
            List.of(new OrderItemCommand(productId.value(), 100))
        );
        
        when(customerRepository.findById(any())).thenReturn(Optional.of(CustomerMother.aValidCustomer(customerId)));
        when(inventoryPort.isAvailable(productId, new Quantity(100))).thenReturn(false);
        when(inventoryPort.getAvailableQuantity(productId)).thenReturn(new Quantity(10));
        
        // When / Then
        assertThatThrownBy(() -> useCase.execute(command))
            .isInstanceOf(InsufficientStockException.class)
            .hasMessageContaining("requested 100, available 10");
    }
}
```

### 11.3 Integration Testing with Testcontainers
> Use Testcontainers for real database integration tests.  
> Test repository adapters with actual database operations.

```
java
@SpringBootTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void shouldPersistAndRetrieveOrder() {
        // Given
        var order = Order.create(CustomerId.generate());
        order.addItem(ProductMother.aValidProduct(), new Quantity(2));
        order.submit();
        
        // When
        orderRepository.save(order);
        var retrieved = orderRepository.findById(order.getId());
        
        // Then
        assertThat(retrieved).isPresent();
        assertThat(retrieved.get().getStatus()).isEqualTo(OrderStatus.SUBMITTED);
        assertThat(retrieved.get().getItems()).hasSize(1);
    }
    
    @Test
    void shouldFindOrdersByCustomer() {
        // Given
        var customerId = CustomerId.generate();
        var order1 = createAndSaveOrder(customerId);
        var order2 = createAndSaveOrder(customerId);
        createAndSaveOrder(CustomerId.generate()); // Different customer
        
        // When
        var orders = orderRepository.findByCustomerId(customerId);
        
        // Then
        assertThat(orders).hasSize(2);
        assertThat(orders).extracting(Order::getId)
            .containsExactlyInAnyOrder(order1.getId(), order2.getId());
    }
}
```

### 11.4 REST Controller Testing
> Use @WebMvcTest for controller unit tests.  
> Mock use cases and verify HTTP responses.

```
java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @MockBean
    private SubmitOrderUseCase submitOrderUseCase;
    
    @MockBean
    private GetOrderDetailsUseCase getOrderDetailsUseCase;
    
    @Test
    void shouldSubmitOrderAndReturn201() throws Exception {
        // Given
        var command = new SubmitOrderCommand(
            UUID.randomUUID(),
            List.of(new OrderItemCommand(UUID.randomUUID(), 2))
        );
        
        var response = new OrderResponse(
            UUID.randomUUID(),
            "SUBMITTED",
            BigDecimal.valueOf(50),
            "USD",
            List.of(),
            Instant.now()
        );
        
        when(submitOrderUseCase.execute(any())).thenReturn(response);
        
        // When / Then
        mockMvc.perform(post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(command)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.status").value("SUBMITTED"))
            .andExpect(jsonPath("$.totalAmount").value(50));
    }
    
    @Test
    void shouldReturn400ForInvalidRequest() throws Exception {
        // Given - Empty items list (validation error)
        var invalidCommand = """
            {
                "customerId": "%s",
                "items": []
            }
            """.formatted(UUID.randomUUID());
        
        // When / Then
        mockMvc.perform(post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidCommand))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.title").value("Validation Error"))
            .andExpect(jsonPath("$.errors[0].field").value("items"));
    }
    
    @Test
    void shouldReturn404WhenOrderNotFound() throws Exception {
        // Given
        var orderId = UUID.randomUUID();
        when(getOrderDetailsUseCase.execute(orderId))
            .thenThrow(new OrderNotFoundException(OrderId.of(orderId)));
        
        // When / Then
        mockMvc.perform(get("/api/v1/orders/{orderId}", orderId))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.title").value("Resource Not Found"));
    }
    
    @Test
    void shouldReturn422ForDomainRuleViolation() throws Exception {
        // Given
        var command = new SubmitOrderCommand(
            UUID.randomUUID(),
            List.of(new OrderItemCommand(UUID.randomUUID(), 1))
        );
        
        when(submitOrderUseCase.execute(any()))
            .thenThrow(new DomainRuleException("Order cannot be submitted on weekends"));
        
        // When / Then
        mockMvc.perform(post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(command)))
            .andExpect(status().isUnprocessableEntity())
            .andExpect(jsonPath("$.title").value("Business Rule Violation"));
    }
}
```

### 11.5 Test Object Mothers / Fixtures
> Use Object Mother pattern for test data creation.  
> Provide fluent factory methods for common test scenarios.

```
java
// Located in: src/test/java/.../fixture/

public final class CustomerMother {
    
    private CustomerMother() {}
    
    public static Customer aValidCustomer() {
        return aValidCustomer(CustomerId.generate());
    }
    
    public static Customer aValidCustomer(CustomerId id) {
        return Customer.builder()
            .id(id)
            .email(new Email("customer@example.com"))
            .name("John Doe")
            .status(CustomerStatus.ACTIVE)
            .build();
    }
    
    public static Customer anInactiveCustomer() {
        return Customer.builder()
            .id(CustomerId.generate())
            .email(new Email("inactive@example.com"))
            .name("Jane Doe")
            .status(CustomerStatus.INACTIVE)
            .build();
    }
}

public final class ProductMother {
    
    private ProductMother() {}
    
    public static Product aValidProduct() {
        return aValidProduct(ProductId.generate(), Money.of(25, Currency.USD));
    }
    
    public static Product aValidProduct(ProductId id, Money price) {
        return new Product(id, "Sample Product", price);
    }
    
    public static Product anExpensiveProduct() {
        return new Product(
            ProductId.generate(), 
            "Expensive Product", 
            Money.of(1000, Currency.USD)
        );
    }
}

public final class OrderMother {
    
    private OrderMother() {}
    
    public static Order aDraftOrder() {
        return Order.create(CustomerId.generate());
    }
    
    public static Order aDraftOrderWithItems() {
        var order = Order.create(CustomerId.generate());
        order.addItem(ProductMother.aValidProduct(), new Quantity(2));
        order.addItem(ProductMother.anExpensiveProduct(), new Quantity(1));
        return order;
    }
    
    public static Order aSubmittedOrder() {
        var order = aDraftOrderWithItems();
        order.submit();
        return order;
    }
    
    public static Order aSubmittedOrder(CustomerId customerId) {
        var order = Order.create(customerId);
        order.addItem(ProductMother.aValidProduct(), new Quantity(1));
        order.submit();
        return order;
    }
    
    public static Order aShippedOrder() {
        var order = aSubmittedOrder();
        order.markAsPaid();
        order.ship();
        return order;
    }
    
    public static Order aCancelledOrder() {
        var order = aSubmittedOrder();
        order.cancel("Customer requested cancellation");
        return order;
    }
}
```

### 11.6 Architecture Testing with ArchUnit
> Use ArchUnit to enforce architectural rules automatically.  
> Prevent architectural violations at build time.

```
java
// Enforce architectural rules automatically
@AnalyzeClasses(packages = "com.company.orderservice")
class ArchitectureTest {
    
    // Layer Dependency Rules
    @ArchTest
    static final ArchRule domainShouldNotDependOnApplication =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..application..");
    
    @ArchTest
    static final ArchRule domainShouldNotDependOnInfrastructure =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..infrastructure..");
    
    @ArchTest
    static final ArchRule domainShouldNotDependOnInterfaces =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..interfaces..");
    
    @ArchTest
    static final ArchRule domainShouldNotUseSpringAnnotations =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("org.springframework..");
    
    // Naming Conventions
    @ArchTest
    static final ArchRule useCasesShouldBeSuffixed =
        classes()
            .that().resideInAPackage("..application.usecase..")
            .should().haveSimpleNameEndingWith("UseCase");
    
    @ArchTest
    static final ArchRule controllersShouldBeSuffixed =
        classes()
            .that().resideInAPackage("..interfaces.rest..")
            .and().areAnnotatedWith(RestController.class)
            .should().haveSimpleNameEndingWith("Controller");
    
    @ArchTest
    static final ArchRule repositoryAdaptersShouldBeSuffixed =
        classes()
            .that().resideInAPackage("..infrastructure.persistence..")
            .and().implement(simpleNameEndingWith("Repository"))
            .should().haveSimpleNameEndingWith("Adapter")
                .orShould().haveSimpleNameEndingWith("Repository");
    
    // Dependency Injection Rules
    @ArchTest
    static final ArchRule noFieldInjection =
        noFields()
            .should().beAnnotatedWith(Autowired.class)
            .because("Field injection is not allowed. Use constructor injection.");
    
    // Exception Rules
    @ArchTest
    static final ArchRule domainExceptionsShouldExtendBase =
        classes()
            .that().resideInAPackage("..domain.exception..")
            .and().areNotInterfaces()
            .and().doNotHaveSimpleName("DomainException")
            .should().beAssignableTo(DomainException.class);
    
    // Value Object Rules
    @ArchTest
    static final ArchRule valueObjectsShouldBeRecords =
        classes()
            .that().resideInAPackage("..domain.model..")
            .and().haveSimpleNameEndingWith("Id")
            .should().beRecords()
            .because("Typed IDs should be implemented as Records");
}
```
