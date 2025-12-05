# Google Gemini CLI Sub-Agents
The Google Gemini CLI (released in 2025) supports a sub-agent system that allows you to delegate specialized tasks to other AI agents. Here's how to use them:

## Basic Concept
Sub-agents in Gemini CLI are specialized agents that can be invoked to handle specific tasks like code review, research, or file operations.

## Using Sub-Agents
1. Built-in Sub-Agents
You can invoke sub-agents using the @ syntax in your prompts:

```
# Example: Using a code review sub-agent
gemini "Review this code @code-review"

# Example: Using a research agent
gemini "@research explain quantum computing basics"

```

2. Creating Custom Sub-Agents
You can define custom sub-agents in your configuration file (typically ~/.gemini/config.json or GEMINI.md):
```
{
  "agents": {
    "my-agent": {
      "description": "A custom agent for specific tasks",
      "systemPrompt": "You are a specialized assistant for..."
    }
  }
}
```

3. Using GEMINI.md Files
Create a GEMINI.md file in your project root to define project-specific agents:

```
markdown
# Sub-Agents

## @reviewer
Reviews code for best practices and security issues.

## @documenter
Generates documentation for code files.

```

## Example Commands for Sub-Agents
### Using @springboot-expert
bash
# Code review
@springboot-expert Review this service class for DDD violations and suggest improvements

# Architecture guidance
@springboot-expert How should I structure a new bounded context for "Shipping" following our standards?

# Implementation help
@springboot-expert Implement a use case for processing refunds following clean architecture

# Best practices
@springboot-expert What's the best way to handle distributed transactions across order and payment services?

# Performance optimization
@springboot-expert How can I optimize this repository query for better performance?


### Using @code-reviewer
bash
# Full code review
@code-reviewer Review src/main/java/com/example/order/OrderService.java

# Architecture review
@code-reviewer Check if this module follows our clean architecture principles

# Security review
@code-reviewer Review this controller for security vulnerabilities

### Using @generator
bash
# Generate aggregate
@generator Create a new aggregate for "Shipment" with tracking, status updates, and delivery confirmation

# Generate use case
@generator Create a use case for "CancelOrder" with proper validation and compensation logic

# Generate tests
@generator Create comprehensive tests for the OrderService including unit, integration, and architecture tests

### Using @refactor-to-fp
bash
# Refactor to functional style
@refactor-to-fp Convert this imperative code to use streams and Optional

# Use pattern matching
@refactor-to-fp Refactor this switch statement to use sealed types and pattern matching

### Using @migrator
bash
# Migration help
@migrator Help me migrate this @Value configuration to @ConfigurationProperties record

# Modernize code
@migrator Update this code to use JDK 21 features like virtual threads and pattern matching


### Quick Reference
#### Package Structure
text
com.company.project/
├── domain/           # No framework dependencies
│   ├── model/        # Aggregates, Entities, Value Objects
│   ├── repository/   # Repository interfaces (ports)
│   ├── service/      # Domain services
│   └── event/        # Domain events
├── application/      # Use cases
│   ├── usecase/      # Command/Query handlers
│   ├── port/         # Input/Output ports
│   └── dto/          # DTOs
├── infrastructure/   # Framework adapters
│   ├── persistence/  # Repository implementations
│   ├── messaging/    # Kafka/RabbitMQ adapters
│   ├── external/     # External API clients
│   └── config/       # Spring configurations
└── interfaces/       # Entry points
    ├── rest/         # REST controllers
    └── messaging/    # Message listeners

### Key Principles Checklist
 Domain layer has zero framework dependencies
 Use records for Value Objects and DTOs
 Aggregates protect invariants
 Rich domain models (no anemic models)
 Use Optional correctly (no isPresent/get pattern)
 Pattern matching with sealed types
 Virtual threads enabled for I/O operations
 Configuration externalized via environment variables
 Stateless application design
 Structured logging with correlation IDs
 Circuit breakers for external services
 Idempotent message processing
 Transactional outbox for events
 Comprehensive health checks
 Security at multiple layers

### Technology Stack
Component	Technology
Framework	Spring Boot 3.2+
JDK	21+ with preview features
Database	PostgreSQL 16+
Cache	Redis 7+
Messaging	Apache Kafka
Observability	Micrometer + OpenTelemetry
Resilience	Resilience4j
Testing	JUnit 5 + Testcontainers + ArchUnit
Build	Gradle (Kotlin DSL)
Container	Docker with multi-stage builds
text

---

This completes the comprehensive `GEMINI.md` file. To use it:

1. **Save this entire content** as `GEMINI.md` in your project root
2. **Start Gemini CLI** in your project directory: `gemini`
3. **Invoke sub-agents** with `@agent-name` prefix:
   - `@springboot-expert` - For architecture and implementation guidance
   - `@code-reviewer` - For code reviews
   - `@generator` - For generating code
   - `@refactor-to-fp` - For functional programming refactoring
   - `@migrator` - For migrating legacy code

Would you like me to:
1. Add more specific sections for any topic?
2. Create separate focused GEMINI.md files for specific concerns?
3. Add more example code for any particular pattern?
Claude Opus 4.5
