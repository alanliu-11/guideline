## 9. Configuration Examples

### 9.1 Application Properties
> Always disable open-in-view for production.  
> Use Flyway for database migrations, never auto-create schema.

```
yaml
# application.yml
spring:
  application:
    name: order-service
  
  # Enable Virtual Threads
  threads:
    virtual:
      enabled: true
  
  # Database
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  
  # JPA
  jpa:
    open-in-view: false  # Always disable for production
    hibernate:
      ddl-auto: validate  # Never auto-create in production
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100
  
  # Flyway
  flyway:
    enabled: true
    locations: classpath:db/migration
  
  # Jackson
  jackson:
    default-property-inclusion: non_null
    serialization:
      write-dates-as-timestamps: false

# Observability
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  tracing:
    sampling:
      probability: 1.0

# Logging
logging:
  level:
    root: INFO
    com.company.project: DEBUG
    org.hibernate.SQL: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%X{traceId:-}] [%thread] %-5level %logger{36} - %msg%n"
```

### 9.2 Spring Configuration Classes
> Configuration classes should be focused and single-purpose.  
> Use @EnableJpaRepositories to control repository scanning.

```
java
// Located in: infrastructure/config/

@Configuration
@EnableJpaRepositories(basePackages = "com.company.project.infrastructure.persistence")
@EntityScan(basePackages = "com.company.project.domain.model")
public class JpaConfig {
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}

@Configuration
public class JacksonConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
            .addModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .enable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
            .build();
    }
}

@Configuration
public class ObservabilityConfig {
    
    @Bean
    public ObservationRegistry observationRegistry() {
        var registry = ObservationRegistry.create();
        registry.observationConfig()
            .observationHandler(new DefaultMeterObservationHandler(new SimpleMeterRegistry()));
        return registry;
    }
}
```
