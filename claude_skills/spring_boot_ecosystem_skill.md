# Claude Skill: Spring Boot 3.x Ecosystem

## Purpose
This skill provides guidelines for using Spring Boot 3.x and the Spring ecosystem. Use this when working with Spring Boot, dependency injection, or Jakarta EE features.

## Jakarta EE Migration

- **Strict usage of `jakarta.*` imports**
- **Flag `javax.*` as a compilation error**

## Dependency Injection

**Strictly enforce Constructor Injection.**

**Constraint:** Never use `@Autowired` on fields.

**Correct Example:**
```java
// ✅ Constructor Injection (Required)
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository repository;
    private final PaymentGateway paymentGateway;
}
```

## Dependency Management (BOM)

- Always rely on `spring-boot-starter-parent` or `spring-boot-dependencies` BOM
- Do not manually version standard libraries unless overriding a bug

## Lombok Restrictions

| Status | Annotation | Reason |
|--------|------------|--------|
| ✅ Allowed | `@Slf4j` | Convenient logger injection |
| ✅ Allowed | `@RequiredArgsConstructor` | Constructor injection |
| ✅ Allowed | `@Getter` | Read-only access |
| ✅ Allowed | `@Builder` | Complex object construction |
| ⚠️ Discouraged | `@Setter` | Breaks encapsulation |
| ⚠️ Discouraged | `@AllArgsConstructor` | Use `@Builder` instead |
| ❌ Banned | `@Data` on Entities | Breaks JPA, exposes setters |
| ❌ Banned | `@Data` on Value Objects | Use Records instead |

## Observability

- Favor Micrometer and ObservationRegistry APIs over manual logging for metrics
- Use structured logging with correlation IDs

## Key Takeaways

1. **Always use `jakarta.*`** - Never use `javax.*`
2. **Constructor Injection Only** - Never use field injection with `@Autowired`
3. **Use Lombok Carefully** - Only allowed annotations, never `@Data` on domain objects
4. **BOM for Dependencies** - Rely on Spring Boot BOM for version management
5. **Observability First** - Use Micrometer for metrics, structured logging

