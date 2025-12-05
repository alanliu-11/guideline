
## 5. Spring Boot 3.x Ecosystem
### 5.1 Jakarta EE Migration
> Strict usage of jakarta.* imports.  
> Flag javax.* as a compilation error.  
```
java
// ❌ Legacy (Do not use)
import javax.persistence.Entity;
import javax.validation.constraints.NotNull;

// ✅ Modern (Required)
import jakarta.persistence.Entity;
import jakarta.validation.constraints.NotNull;
```

### 5.2 Dependency Injection
> Strictly enforce Constructor Injection.  
> Constraint: Never use @Autowired on fields.  
```
java
// ❌ Field Injection (Banned)
@Service
public class OrderService {
    @Autowired
    private OrderRepository repository;
}

// ✅ Constructor Injection (Required)
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository repository;
    private final PaymentGateway paymentGateway;
}
```

### 5.3 Dependency Management (BOM)
> Always rely on spring-boot-starter-parent or spring-boot-dependencies BOM.  
> Do not manually version standard libraries unless overriding a bug.  

```
xml
<!-- ✅ Correct: Use Spring Boot Parent -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<dependencies>
    <!-- ✅ No version needed - managed by BOM -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- ❌ Avoid manual versioning -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.15.0</version> <!-- Don't do this -->
    </dependency>
</dependencies>

```

## 5.4 Lombok Restrictions
> Status Annotations	Reason  
✅ Allowed	@Slf4j	Convenient logger injection  
✅ Allowed	@RequiredArgsConstructor	Constructor injection  
✅ Allowed	@Getter	Read-only access  
✅ Allowed	@Builder	Complex object construction  
⚠️ Discouraged	@Setter	Breaks encapsulation  
⚠️ Discouraged	@AllArgsConstructor	Use @Builder instead  
❌ Banned	@Data on Entities	Breaks JPA, exposes setters  
❌ Banned	@Data on Value Objects	Use Records instead  

## 5.5 Observability
> Favor Micrometer and ObservationRegistry APIs over manual logging for metrics.  
> Use structured logging with correlation IDs.  
```
java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderProcessingUseCase {
    
    private final OrderRepository orders;
    private final ObservationRegistry observationRegistry;
    
    public OrderResponse processOrder(OrderId orderId) {
        return Observation.createNotStarted("order.processing", observationRegistry)
            .lowCardinalityKeyValue("order.type", "standard")
            .observe(() -> {
                log.info("Processing order: {}", orderId);
                var order = orders.findById(orderId)
                    .orElseThrow(() -> new OrderNotFoundException(orderId));
                order.process();
                orders.save(order);
                return mapper.toResponse(order);
            });
    }
}
```

### 5.6 AOT & Native Image Readiness
> Avoid complex reflection where possible.  
> Register hints for any required reflection.  

```
java
// RuntimeHints Registrar for GraalVM Native Image
public class AppRuntimeHints implements RuntimeHintsRegistrar {
    
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Register reflection for domain classes if needed
        hints.reflection()
            .registerType(Order.class, MemberCategory.values())
            .registerType(OrderSubmittedEvent.class, MemberCategory.values());
        
        // Register resources
        hints.resources()
            .registerPattern("db/migration/*.sql");
    }
}

// Register in configuration
@Configuration
@ImportRuntimeHints(AppRuntimeHints.class)
public class NativeConfig {
}
```