## 16. IntelliJ IDEA Live Templates
> Add these live templates for faster development.

### 16.1 Value Object Record (vor)
```
java
public record $NAME$($TYPE$ value) {
    public $NAME$ {
        java.util.Objects.requireNonNull(value, "$NAME$ cannot be null");
        $VALIDATION$
    }
    
    public static $NAME$ of($TYPE$ value) {
        return new $NAME$(value);
    }
}
```

### 16.2 Typed ID Record (tid)
```
java
public record $ENTITY$Id(java.util.UUID value) {
    public $ENTITY$Id {
        java.util.Objects.requireNonNull(value, "$ENTITY$Id cannot be null");
    }
    
    public static $ENTITY$Id generate() {
        return new $ENTITY$Id(java.util.UUID.randomUUID());
    }
    
    public static $ENTITY$Id of(java.util.UUID value) {
        return new $ENTITY$Id(value);
    }
    
    public static $ENTITY$Id of(String value) {
        return new $ENTITY$Id(java.util.UUID.fromString(value));
    }
}
```

### 16.3 Domain Event Record (dev)
```
java
public record $NAME$Event(
    $AGGREGATE_ID$ aggregateId,
    $END$
    java.time.Instant occurredAt
) {
    public $NAME$Event {
        java.util.Objects.requireNonNull(aggregateId);
        occurredAt = occurredAt != null ? occurredAt : java.time.Instant.now();
    }
}
```

### 16.4 Use Case Class (ucs)
```
java
@org.springframework.stereotype.Service
@lombok.RequiredArgsConstructor
@lombok.extern.slf4j.Slf4j
public class $NAME$UseCase {
    
    private final $REPOSITORY$ repository;
    
    @org.springframework.transaction.annotation.Transactional
    public $RETURN_TYPE$ execute($COMMAND$ command) {
        log.info("Executing $NAME$: {}", command);
        $END$
    }
}
```

### 16.5 Repository Interface (rpi)
```
java
public interface $ENTITY$Repository {
    
    void save($ENTITY$ entity);
    
    java.util.Optional<$ENTITY$> findById($ENTITY$Id id);
    
    void delete($ENTITY$ entity);
    
    boolean existsById($ENTITY$Id id);
}
```

### 16.6 REST Controller (rct)
```
java
@org.springframework.web.bind.annotation.RestController
@org.springframework.web.bind.annotation.RequestMapping("/api/v1/$PATH$")
@lombok.RequiredArgsConstructor
@io.swagger.v3.oas.annotations.tags.Tag(name = "$TAG$", description = "$DESCRIPTION$")
public class $NAME$Controller {
    
    private final $USE_CASE$ useCase;
    
    $END$
}
```

## 17. Conclusion & Usage Instructions

### 17.1 How to Use This Configuration
> Copy the entire configuration into your AI tool's System Prompt or Custom Instructions.

**For Claude Code / Claude Projects:**
- Add to Project Instructions or System Prompt
- Reference when starting new conversations

**For Google Gemini:**
- Use as System Instructions in AI Studio
- Or paste at the beginning of conversations

**For GitHub Copilot:**
- Create a `.github/copilot-instructions.md` file
- Paste relevant sections

**For Cursor:**
- Add to `.cursorrules` file in project root

### 17.2 Customization Points
> Adjust these sections based on your project needs.

| Section | Customization |
|---------|---------------|
| Java Version | Change from 21 to 17 if needed (remove Virtual Threads) |
| Package Structure | Adjust base package name |
| Database | Modify for MySQL, Oracle, etc. |
| Messaging | Adjust for your broker (Kafka, RabbitMQ, SQS) |
| Security | Add OAuth2/OIDC specifics |
| Testing | Add contract testing if needed |

### 17.3 Version Information
> Current technology stack versions.

| Component | Version |
|-----------|---------|
| Java | 21 (LTS) |
| Spring Boot | 3.3.x |
| Spring Framework | 6.1.x |
| Hibernate | 6.4.x |
| JUnit | 5.10.x |
| Testcontainers | 1.19.x |
| ArchUnit | 1.2.x |

This completes the comprehensive Sub-Agent Configuration for Core Technology & DDD Profile.
