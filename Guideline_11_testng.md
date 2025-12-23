## 11. Testing Guidelines (TestNG Framework)

### 11.1 Unit Testing (Domain Layer)
> Domain tests should be framework-free.  
> Test business logic in isolation without Spring or JPA.

```java
// Test Domain Logic without frameworks
import org.testng.annotations.Test;
import static org.assertj.core.api.Assertions.*;

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

```java
import org.testng.annotations.BeforeMethod;
import org.testng.annotations.Test;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

class SubmitOrderUseCaseTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @Mock
    private CustomerRepository customerRepository;
    
    @Mock
    private InventoryPort inventoryPort;
    
    private SubmitOrderUseCase useCase;
    
    @BeforeMethod
    void setUp() {
        MockitoAnnotations.openMocks(this);
        useCase = new SubmitOrderUseCase(orderRepository, customerRepository, inventoryPort);
    }
    
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
    
    // TestNG DataProvider for parameterized tests
    @Test(dataProvider = "orderQuantityProvider")
    void shouldHandleDifferentOrderQuantities(int quantity, boolean expectedSuccess) {
        // Given
        var customerId = CustomerId.generate();
        var productId = ProductId.generate();
        var command = new SubmitOrderCommand(
            customerId.value(),
            List.of(new OrderItemCommand(productId.value(), quantity))
        );
        
        when(customerRepository.findById(any()))
            .thenReturn(Optional.of(CustomerMother.aValidCustomer(customerId)));
        when(inventoryPort.isAvailable(productId, new Quantity(quantity)))
            .thenReturn(expectedSuccess);
        
        if (expectedSuccess) {
            when(inventoryPort.getProduct(productId))
                .thenReturn(ProductMother.aValidProduct(productId, Money.of(25, Currency.USD)));
        }
        
        // When / Then
        if (expectedSuccess) {
            var response = useCase.execute(command);
            assertThat(response).isNotNull();
        } else {
            assertThatThrownBy(() -> useCase.execute(command))
                .isInstanceOf(InsufficientStockException.class);
        }
    }
    
    @DataProvider(name = "orderQuantityProvider")
    Object[][] orderQuantityProvider() {
        return new Object[][] {
            {1, true},
            {10, true},
            {100, false},
            {1000, false}
        };
    }
}
```

### 11.3 Integration Testing with Testcontainers
> Use Testcontainers for real database integration tests.  
> Test repository adapters with actual database operations.

```java
import org.testng.annotations.BeforeClass;
import org.testng.annotations.AfterClass;
import org.testng.annotations.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.beans.factory.annotation.Autowired;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.utility.DockerImageName;
import static org.assertj.core.api.Assertions.*;

@SpringBootTest
class OrderRepositoryIntegrationTest {
    
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>(
        DockerImageName.parse("postgres:16-alpine")
    )
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @BeforeClass
    static void startContainer() {
        postgres.start();
    }
    
    @AfterClass
    static void stopContainer() {
        postgres.stop();
    }
    
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
    
    @Test(groups = "integration", priority = 1)
    void shouldHandleConcurrentOrderCreation() {
        // Test concurrent order creation scenarios
        // Implementation details...
    }
}
```

### 11.4 REST Controller Testing
> Use Spring TestNG integration for controller tests.  
> Mock use cases and verify HTTP responses.

```java
import org.testng.annotations.BeforeMethod;
import org.testng.annotations.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import com.fasterxml.jackson.databind.ObjectMapper;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
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
    
    // TestNG test groups for organizing tests
    @Test(groups = "controller", priority = 1)
    void shouldHandleMultipleOrderSubmissions() throws Exception {
        // Test multiple order submissions
        // Implementation details...
    }
    
    // TestNG test dependencies
    @Test(dependsOnMethods = "shouldSubmitOrderAndReturn201")
    void shouldRetrieveSubmittedOrder() throws Exception {
        // This test depends on order submission
        // Implementation details...
    }
}
```

### 11.5 Test Object Mothers / Fixtures
> Use Object Mother pattern for test data creation.  
> Provide fluent factory methods for common test scenarios.

```java
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

```java
// Enforce architectural rules automatically
import org.testng.annotations.Test;
import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import com.tngtech.archunit.lang.ArchRule;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.*;

@AnalyzeClasses(packages = "com.company.orderservice")
class ArchitectureTest {
    
    // Layer Dependency Rules
    @Test
    void domainShouldNotDependOnApplication() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..application..");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
    @Test
    void domainShouldNotDependOnInfrastructure() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..infrastructure..");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
    @Test
    void domainShouldNotDependOnInterfaces() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..interfaces..");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
    @Test
    void domainShouldNotUseSpringAnnotations() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("org.springframework..");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
    // Naming Conventions
    @Test
    void useCasesShouldBeSuffixed() {
        ArchRule rule = classes()
            .that().resideInAPackage("..application.usecase..")
            .should().haveSimpleNameEndingWith("UseCase");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
    @Test
    void controllersShouldBeSuffixed() {
        ArchRule rule = classes()
            .that().resideInAPackage("..interfaces.rest..")
            .and().areAnnotatedWith(RestController.class)
            .should().haveSimpleNameEndingWith("Controller");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
    @Test
    void repositoryAdaptersShouldBeSuffixed() {
        ArchRule rule = classes()
            .that().resideInAPackage("..infrastructure.persistence..")
            .and().implement(simpleNameEndingWith("Repository"))
            .should().haveSimpleNameEndingWith("Adapter")
                .orShould().haveSimpleNameEndingWith("Repository");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
    // Dependency Injection Rules
    @Test
    void noFieldInjection() {
        ArchRule rule = noFields()
            .should().beAnnotatedWith(Autowired.class)
            .because("Field injection is not allowed. Use constructor injection.");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
    // Exception Rules
    @Test
    void domainExceptionsShouldExtendBase() {
        ArchRule rule = classes()
            .that().resideInAPackage("..domain.exception..")
            .and().areNotInterfaces()
            .and().doNotHaveSimpleName("DomainException")
            .should().beAssignableTo(DomainException.class);
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
    // Value Object Rules
    @Test
    void valueObjectsShouldBeRecords() {
        ArchRule rule = classes()
            .that().resideInAPackage("..domain.model..")
            .and().haveSimpleNameEndingWith("Id")
            .should().beRecords()
            .because("Typed IDs should be implemented as Records");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
}
```

### 11.7 TestNG-Specific Features

#### 11.7.1 Test Groups
> Organize tests into groups for selective execution.

```java
@Test(groups = {"unit", "fast"})
void shouldCalculateOrderTotal() {
    // Fast unit test
}

@Test(groups = {"integration", "slow"})
void shouldProcessOrderThroughFullStack() {
    // Slow integration test
}

// Run only fast tests: mvn test -Dgroups=fast
// Run only unit tests: mvn test -Dgroups=unit
```

#### 11.7.2 Test Dependencies
> Define test execution order using dependencies.

```java
@Test(priority = 1)
void shouldCreateOrder() {
    // Create order
}

@Test(priority = 2, dependsOnMethods = "shouldCreateOrder")
void shouldUpdateOrder() {
    // Update the created order
}

@Test(priority = 3, dependsOnMethods = {"shouldCreateOrder", "shouldUpdateOrder"})
void shouldDeleteOrder() {
    // Delete the order
}
```

#### 11.7.3 Data Providers
> Parameterize tests with multiple data sets.

```java
@Test(dataProvider = "customerDataProvider")
void shouldValidateCustomerEmail(String email, boolean expectedValid) {
    // Given
    var customer = Customer.builder()
        .email(new Email(email))
        .build();
    
    // When / Then
    if (expectedValid) {
        assertThat(customer.getEmail()).isNotNull();
    } else {
        assertThatThrownBy(() -> customer.getEmail())
            .isInstanceOf(InvalidEmailException.class);
    }
}

@DataProvider(name = "customerDataProvider")
Object[][] customerDataProvider() {
    return new Object[][] {
        {"valid@example.com", true},
        {"another.valid@example.com", true},
        {"invalid-email", false},
        {"@example.com", false},
        {"", false}
    };
}
```

#### 11.7.4 Test Lifecycle Hooks
> Use TestNG lifecycle methods for setup and teardown.

```java
class OrderServiceTest {
    
    @BeforeClass
    static void setUpClass() {
        // Initialize class-level resources (runs once per class)
        System.out.println("Setting up test class");
    }
    
    @AfterClass
    static void tearDownClass() {
        // Clean up class-level resources
        System.out.println("Tearing down test class");
    }
    
    @BeforeMethod
    void setUp() {
        // Initialize test-level resources (runs before each test)
        System.out.println("Setting up test method");
    }
    
    @AfterMethod
    void tearDown() {
        // Clean up test-level resources
        System.out.println("Tearing down test method");
    }
    
    @BeforeGroups("integration")
    void setUpIntegrationTests() {
        // Setup for integration test group
    }
    
    @AfterGroups("integration")
    void tearDownIntegrationTests() {
        // Cleanup for integration test group
    }
}
```

#### 11.7.5 Parallel Test Execution
> Configure parallel test execution for faster test runs.

```java
// In testng.xml
<suite name="TestSuite" parallel="methods" thread-count="4">
    <test name="AllTests">
        <packages>
            <package name="com.company.orderservice.*"/>
        </packages>
    </test>
</suite>

// Or via annotation
@Test(threadPoolSize = 3, invocationCount = 10, timeOut = 10000)
void shouldHandleConcurrentRequests() {
    // Test concurrent execution
}
```

### 11.8 TestNG Configuration (testng.xml)

```xml
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="OrderServiceTestSuite" verbose="1" parallel="methods" thread-count="4">
    
    <listeners>
        <listener class-name="com.company.orderservice.test.TestListener"/>
    </listeners>
    
    <test name="UnitTests">
        <groups>
            <run>
                <include name="unit"/>
                <exclude name="slow"/>
            </run>
        </groups>
        <packages>
            <package name="com.company.orderservice.domain.*"/>
            <package name="com.company.orderservice.application.*"/>
        </packages>
    </test>
    
    <test name="IntegrationTests">
        <groups>
            <run>
                <include name="integration"/>
            </run>
        </groups>
        <packages>
            <package name="com.company.orderservice.infrastructure.*"/>
        </packages>
    </test>
    
    <test name="ControllerTests">
        <groups>
            <run>
                <include name="controller"/>
            </run>
        </groups>
        <packages>
            <package name="com.company.orderservice.interfaces.rest.*"/>
        </packages>
    </test>
</suite>
```

### 11.9 Best Practices for TestNG

1. **Use `@BeforeMethod` instead of `@BeforeEach`**: TestNG uses `@BeforeMethod` for setup before each test method.

2. **Leverage Test Groups**: Organize tests into logical groups (unit, integration, slow, fast) for selective execution.

3. **Use Data Providers**: For parameterized tests, use `@DataProvider` instead of JUnit's `@ParameterizedTest`.

4. **Configure Parallel Execution**: Use `testng.xml` or annotations to run tests in parallel for faster execution.

5. **Test Dependencies**: Use `dependsOnMethods` and `dependsOnGroups` to define test execution order when needed.

6. **Lifecycle Management**: Use `@BeforeClass`/`@AfterClass` for expensive setup/teardown that should run once per test class.

7. **Assertions**: Continue using AssertJ for readable assertions, as it works seamlessly with TestNG.

8. **Mockito Integration**: Use `MockitoAnnotations.openMocks(this)` in `@BeforeMethod` instead of `@ExtendWith(MockitoExtension.class)`.
