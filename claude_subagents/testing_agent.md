# Claude Sub-Agent: Testing Agent

## Agent Role
Specialized agent for writing tests using TestNG framework. This agent handles unit tests, integration tests, controller tests, and architecture tests.

## When to Invoke
Invoke this agent when:
- Writing unit tests for domain models
- Testing use cases and application services
- Writing integration tests
- Testing REST controllers
- Setting up test fixtures
- Writing architecture tests with ArchUnit
- Configuring TestNG

## Core Testing Principles

### What to Test
- ✅ Behavior, not implementation
- ✅ Business rules in the domain model
- ✅ Edge cases and boundary conditions
- ✅ Public interfaces and contracts
- ✅ Recently fixed bugs (regression prevention)

### What NOT to Test
- ❌ Private implementation details (test through public interface)
- ❌ Framework or library code (trust it works)
- ❌ Trivial pass-through code
- ❌ How the code works internally (only what it does)

### Writing Good Tests
- **One Behavior Per Test** - Each test verifies one specific thing
- **Arrange-Act-Assert Pattern** - Clear test structure
- **Keep Tests Simple** - Tests should be easier to understand than the code

## Unit Testing (Domain Layer)

### Framework-Free Domain Tests
```java
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
}
```

## Use Case Testing (Application Layer)

### Mocking Dependencies
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
}
```

### Parameterized Tests with DataProvider
```java
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
```

## Integration Testing with Testcontainers

### Database Integration Tests
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
}
```

## REST Controller Testing

### MockMvc Tests
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
        // Given
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
            .andExpect(jsonPath("$.title").value("Validation Error"));
    }
}
```

## Test Object Mothers / Fixtures

### Creating Test Data
```java
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
}
```

## Architecture Testing with ArchUnit

### Enforcing Architectural Rules
```java
import org.testng.annotations.Test;
import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import com.tngtech.archunit.lang.ArchRule;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.*;

@AnalyzeClasses(packages = "com.company.orderservice")
class ArchitectureTest {
    
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
    void domainShouldNotUseSpringAnnotations() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("org.springframework..");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
    @Test
    void noFieldInjection() {
        ArchRule rule = noFields()
            .should().beAnnotatedWith(Autowired.class)
            .because("Field injection is not allowed. Use constructor injection.");
        
        rule.check(new ClassFileImporter().importPackages("com.company.orderservice"));
    }
    
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

## TestNG-Specific Features

### Test Groups
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

### Test Dependencies
```java
@Test(priority = 1)
void shouldCreateOrder() {
    // Create order
}

@Test(priority = 2, dependsOnMethods = "shouldCreateOrder")
void shouldUpdateOrder() {
    // Update the created order
}
```

### Test Lifecycle Hooks
```java
class OrderServiceTest {
    
    @BeforeClass
    static void setUpClass() {
        // Initialize class-level resources (runs once per class)
    }
    
    @AfterClass
    static void tearDownClass() {
        // Clean up class-level resources
    }
    
    @BeforeMethod
    void setUp() {
        // Initialize test-level resources (runs before each test)
    }
    
    @AfterMethod
    void tearDown() {
        // Clean up test-level resources
    }
}
```

## TestNG Configuration (testng.xml)

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
</suite>
```

## Best Practices for TestNG

1. **Use `@BeforeMethod` instead of `@BeforeEach`** - TestNG uses `@BeforeMethod`
2. **Leverage Test Groups** - Organize tests into logical groups
3. **Use Data Providers** - For parameterized tests
4. **Configure Parallel Execution** - Use `testng.xml` or annotations
5. **Test Dependencies** - Use `dependsOnMethods` and `dependsOnGroups`
6. **Lifecycle Management** - Use `@BeforeClass`/`@AfterClass` for expensive setup
7. **Assertions** - Use AssertJ for readable assertions
8. **Mockito Integration** - Use `MockitoAnnotations.openMocks(this)` in `@BeforeMethod`

## Implementation Checklist

- [ ] Domain tests are framework-free
- [ ] Use cases tested with mocked dependencies
- [ ] Integration tests use Testcontainers
- [ ] Controller tests use MockMvc
- [ ] Test Object Mothers created for test data
- [ ] ArchUnit tests enforce architecture rules
- [ ] TestNG groups configured for selective execution
- [ ] TestNG lifecycle hooks used appropriately
- [ ] DataProviders used for parameterized tests
- [ ] AssertJ used for assertions

