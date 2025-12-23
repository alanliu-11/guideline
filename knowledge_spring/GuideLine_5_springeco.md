## 5. Spring Boot 3.x Ecosystem
### 5.1 Jakarta EE Migration
> Strict usage of jakarta.* imports.  
> Flag javax.* as a compilation error.  

### 5.2 Dependency Injection
> Strictly enforce Constructor Injection.  
> Constraint: Never use @Autowired on fields.  
```
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


