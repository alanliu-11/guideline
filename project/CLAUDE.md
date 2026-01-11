
---

## CLAUDE.md Configuration File

**File: `CLAUDE.md` (Project Root)**

# Project: {{Project Name}}

## Overview
Spring Boot application using Domain-Driven Design architecture.

## Tech Stack
- Java 17+
- Spring Boot 3.5+
- MS SQL SERVER /PostgreSQL 
- Maven

---
## Directory Structure

```text
your-project/
├── .claude/
│   └── agents/
│       ├── modular-monolith-agent.md    # Agent 1
│       └── rich-domain-model-agent.md   # Agent 2
├── CLAUDE.md                            # Project configuration
└── src/
    └── main/java/...
```

## Sub-Agents

### 1. Modular Monolith Agent
- **File**: `.claude/agents/modular-monolith-agent.md`
- **Invoke with**: 
  - `@modular` or `@component`
  - Keywords: "create component", "modular monolith", "package by feature"
- **Use for**: Medium complexity features, clear boundaries, faster development

### 2. Rich Domain Model Agent  
- **File**: `.claude/agents/rich-domain-model-agent.md`
- **Invoke with**: 
  - `@ddd` or `@domain`
  - Keywords: "rich domain", "bounded context", "hexagonal", "full ddd"
- **Use for**: Complex business domains, strict separation, long-term maintainability

### 3. Architecture Validator Agent
- **File**: `.claude/agents/architecture-validator-agent.md`
- **Invoke with**: 
  - `@validate` or `@check`
  - Keywords: "validate architecture", "check boundaries", "architecture review"
- **Use for**: Validating rules, checking violations, generating ArchUnit tests

### 4. Testing Agent
- **File**: `.claude/agents/testing-agent.md`
- **Invoke with**: 
  - `@test` or `@testing`
  - Keywords: "write test", "unit test", "integration test", "controller test", "testng"
- **Use for**: Writing tests using TestNG, creating fixtures, architecture tests

### 5. Code Quality Validator Agent
- **File**: `.claude/agents/code-quality-validator-agent.md`
- **Invoke with**: 
  - `@quality` or `@review`
  - Keywords: "validate code", "code review", "check quality", "style check"
- **Use for**: Code quality validation, style checking, best practices enforcement

## Agent Workflow (Complete)

```text
┌─────────────────┐
│ 1. Generate     │
│ @modular or @ddd│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. Validate     │
│ @validate       │
│ Architecture    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 3. Quality      │
│ @quality        │
│ Code Review     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 4. Test         │
│ @test           │
│ Write Tests     │
└─────────────────┘
```
---

## Architecture Decision Guide

| Criteria | Use Modular Monolith | Use Rich Domain Model |
|----------|---------------------|----------------------|
| Complexity | Low to Medium | High |
| Team Experience | Any | DDD experienced |
| Time Constraint | Tight | Flexible |
| Domain Logic | Simple CRUD+ | Complex rules |
| Future Plans | May extract services | Core competitive advantage |

---

## Project Conventions

### Package Structure (Modular Monolith)
```text
com.{{company}}.{{project}}/
├── {{component}}/
│ ├── api/ # PUBLIC
│ └── internal/ # PRIVATE
```

### Package Structure (Rich Domain Model)
```text
com.{{company}}.{{project}}.{{boundedContext}}/
├── api/
├── application/
├── domain/ # PURE - No frameworks
└── infrastructure/
```

### Naming Conventions
| Type | Convention | Example |
|------|-----------|---------|
| Component | Singular, lowercase | `order`, `payment` |
| Bounded Context | Singular, lowercase | `ordering`, `billing` |
| Aggregate | PascalCase | `Order`, `Payment` |
| Value Object | PascalCase Record | `OrderId`, `Money` |
| Domain Event | Past tense + Event | `OrderCreatedEvent` |
| Repository Port | Aggregate + Repository | `OrderRepository` |
| Repository Adapter | Aggregate + RepositoryAdapter | `OrderRepositoryAdapter` |

---

## Commands Reference

### Modular Monolith Commands
```bash
# Create new component
claude "@modular create Order component with CRUD operations"

# Add cross-component communication
claude "@modular add Payment integration to Order component"

# Validate component boundaries
claude "@modular validate Order component isolation"
```

### Rich Domain Model Commands
```bash
# Create bounded context
claude "@ddd create Order bounded context with full DDD layers"

# Create aggregate with behaviors
claude "@ddd create Order aggregate with confirm(), cancel(), complete() behaviors"

# Generate infrastructure for domain
claude "@ddd generate persistence layer for Order aggregate"

# Validate domain purity
claude "@ddd validate Order domain layer has no framework dependencies"
```

### General Commands
```bash
# Compare approaches
claude "Compare modular monolith vs rich domain model for Inventory feature"

# Architecture review
claude "Review current architecture and suggest improvements"

# Generate tests
claude "Generate unit tests for Order aggregate"
```

### Architecture Validate Commands Summary

| Agent | Command | Purpose |
|-------|---------|---------|
| Modular Monolith | `@modular create X component` | Create component |
| Rich Domain Model | `@ddd create X bounded context` | Create DDD context |
| Validator | `@validate check X` | Validate architecture |
| Validator | `@validate generate tests` | Generate ArchUnit tests |

## Testing Commands Summary

| Command | Purpose |
|---------|---------|
| `@test domain for Order` | Generate domain model unit tests |
| `@test application for OrderService` | Generate application service tests |
| `@test integration for OrderRepository` | Generate repository integration tests |
| `@test controller for OrderController` | Generate REST controller tests |
| `@test fixture for Order` | Generate Object Mother |
| `@test architecture` | Generate ArchUnit tests |
| `@test all for Order` | Generate complete test suite |
## Quality Commands Summary

| Command | Purpose |
|---------|---------|
| `@quality validate file X` | Validate single file |
| `@quality validate package X` | Validate package |
| `@quality validate domain layer` | Validate layer |
| `@quality check java21` | Check Java 21 compliance |
| `@quality check architecture` | Check architecture |
| `@quality full review` | Full project validation |
| `@quality pre-commit` | Pre-commit validation |

## Quality Checklist
### Before PR - Modular Monolith
 - All internal/ classes are package-private
 - No cross-component internal/ imports
 - DTOs used at all boundaries
 - Service interface in api/ package
### Before PR - Rich Domain Model
 - Domain layer has ZERO framework imports
 - All business logic in aggregates (not services)
 - Repository is interface in domain, impl in infrastructure
 - Domain events for state changes
 - Value Objects are immutable Records

## Prohibited Patterns
### ❌ Never Do This
```java
// Cross-component entity reference
@ManyToOne
private PaymentEntity payment;  // WRONG!

// Direct repository access from other component
private final PaymentRepository paymentRepository;  // WRONG!

// Framework annotation in domain layer
@Entity  // WRONG in domain layer!
public class Order { }

// Anemic domain model
public void setStatus(Status s) { this.status = s; }  // WRONG!
```
### ✅ Always Do This
```java
// Reference by ID
private PaymentId paymentId;  // CORRECT!

// Use service interface
private final PaymentService paymentService;  // CORRECT!

// Pure domain class
public class Order { }  // CORRECT!

// Rich behavior
public void confirm() {
    validateCanConfirm();
    this.status = CONFIRMED;
    registerEvent(new OrderConfirmedEvent(this));
}  // CORRECT!
```
## File Locations
```text
{{project}}/
├── .claude/
│   └── agents/
│       ├── modular-monolith-agent.md
│       ├── rich-domain-model-agent.md
│       ├── architecture-validator-agent.md
│       ├── testing-agent.md
│       └── code-quality-validator-agent.md  
├── CLAUDE.md
├── src/
│   ├── main/java/...
│   └── test/
│       ├── java/
│       │   └── com/{{company}}/{{project}}/
│       │       ├── {{context}}/
│       │       │   ├── domain/model/
│       │       │   │   ├── {{Aggregate}}Test.java
│       │       │   │   └── MoneyTest.java
│       │       │   ├── application/service/
│       │       │   │   └── {{Aggregate}}ApplicationServiceTest.java
│       │       │   ├── infrastructure/persistence/
│       │       │   │   └── {{Aggregate}}RepositoryIntegrationTest.java
│       │       │   └── api/controller/
│       │       │       └── {{Aggregate}}ControllerTest.java
│       │       ├── architecture/
│       │       │   └── ArchitectureTest.java
│       │       └── test/
│       │           ├── mother/
│       │           │   ├── {{Aggregate}}Mother.java
│       │           │   └── {{ChildEntity}}Mother.java
│       │           └── listener/
│       │               └── TestExecutionListener.java
│       └── resources/
│           └── testng.xml
└── pom.xml / build.gradle  
```

---

## Agent Usage Examples

### Modular Monolith Agent (@modular)

#### Example 1: Create New Component
```bash
claude "@modular create Order component with:
- CreateOrderRequest with customerId and list of items
- OrderDto with id, status, totalAmount, createdAt
- CRUD operations plus getOrdersByCustomer
- Order model with create(), addItem(), confirm() behaviors"
```

#### Example 2: Add Cross-Component Communication
```bash
claude "@modular add Payment integration to Order component:
1. Inject PaymentService interface
2. Create PaymentRequest DTO for communication
3. Call payment initiation after order creation"
```
#### Example 3: Generate Component with Custom Methods
```bash
claude "@modular create Inventory component with:
- Public API: reserve(), release(), checkAvailability()
- Internal model with stock tracking
- Integration point for Order component"
```

### Rich Domain Model Agent (@ddd)
#### Example 1: Create Bounded Context
```bash
claude "@ddd create Order bounded context with:
- OrderId typed identifier
- Order aggregate with addItem(), confirm(), cancel(), complete() behaviors
- OrderLine child entity
- Money value object
- Domain events for all state changes
- Full persistence layer with JPA"
```
#### Example 2: Create Aggregate with Behaviors
```bash
claude "@ddd create Payment aggregate with:
- PaymentId, Amount, PaymentMethod value objects
- States: PENDING, AUTHORIZED, CAPTURED, REFUNDED, FAILED
- Behaviors: authorize(), capture(), refund()
- Domain events: PaymentAuthorizedEvent, PaymentCapturedEvent"
```
#### Example 3: Generate Infrastructure Layer
```bash
claude "@ddd generate persistence layer for Order aggregate:
1. OrderEntity with JPA annotations
2. OrderJpaRepository interface
3. OrderRepositoryAdapter implementing domain port
4. OrderPersistenceMapper for Entity <-> Domain mapping"
```

### Architecture Validator Agent (@validate)
#### Example 1: Validate Domain Purity
```bash
claude "@validate check domain layer purity for Order bounded context:
1. Scan for Spring annotations (@Service, @Component, @Repository)
2. Scan for JPA annotations (@Entity, @Column, @Id)
3. Check for forbidden imports (org.springframework.*, jakarta.persistence.*)
4. Verify all domain classes are pure Java"
```
#### Example 2: Verify Dependency Rule
```bash
claude "@validate verify dependency rule for Order package:
1. Check all arrows point toward domain layer
2. Verify application layer only imports from domain
3. Verify infrastructure layer only imports from domain
4. Check API layer does not import infrastructure"
```
#### Example 3: Check Component Boundaries
```bash
claude "@validate check component boundaries between Order and Payment:
1. Verify no Order imports from Payment.internal
2. Verify communication via interfaces only
3. Check DTOs used at boundaries
4. Verify no cross-component entity references"
```
#### Example 4: Full Architecture Validation
```bash
claude "@validate perform complete architecture validation:
1. Layer dependency analysis
2. Component boundary verification
3. Repository pattern compliance
4. Domain model purity check
5. Generate ArchUnit tests for enforcement"
```

### Testing Agent (@test)
#### Example 1: Generate Domain Unit Tests
```bash
claude "@test create domain tests for Order aggregate:
1. Test Order.create() factory method
2. Test addItem() behavior with valid and invalid inputs
3. Test confirm() with precondition validation
4. Test cancel() with reason and state checks
5. Test domain event registration
6. Use TestNG with groups: unit, fast, domain"
```
#### Example 2: Generate Application Service Tests
```bash
claude "@test create application service tests for OrderApplicationService:
1. Mock OrderRepository, PaymentService, DomainEventPublisher
2. Test create() use case with success and failure paths
3. Test confirm() with found/not found scenarios
4. Test domain exception propagation
5. Verify event publishing after state changes
6. Use @DataProvider for status transition tests"
```
#### Example 3: Generate Integration Tests
```bash
claude "@test create integration tests for Order repository:
1. Use Testcontainers with PostgreSQL
2. Test save and retrieve round-trip
3. Test findByStatus query
4. Verify aggregate integrity after persistence
5. Test cascade operations for child entities
6. Use TestNG with groups: integration, slow, persistence"
```
#### Example 4: Generate Controller Tests
```bash
claude "@test create controller tests for OrderController:
1. Use MockMvc with @WebMvcTest
2. Test POST /api/v1/orders returns 201
3. Test GET /api/v1/orders/{id} returns 200 or 404
4. Test validation errors return 400
5. Test domain exceptions return proper ProblemDetail
6. Verify OpenAPI response structure"
```
#### Example 5: Generate Test Fixtures
```bash
claude "@test create Object Mothers for Order bounded context:
1. OrderMother with factory methods for each status
2. OrderLineMother with default, expensive, cheap items
3. CustomerMother with active and inactive states
4. Parameterized methods for custom configurations"
```
#### Example 6: Generate Architecture Tests
```bash
claude "@test create ArchUnit tests for project:
1. Domain layer has no framework dependencies
2. Application layer does not import infrastructure
3. Controllers only depend on application services
4. Repository interfaces in domain, adapters in infrastructure
5. DTOs are records
6. No field injection allowed"
```
#### Example 7: Generate Complete Test Suite
```bash
claude "@test create complete test suite for Order bounded context:
1. Domain unit tests (Order, OrderLine, Money, OrderId)
2. Application service tests with mocks
3. Repository integration tests with Testcontainers
4. Controller tests with MockMvc
5. Object Mothers for all domain objects
6. ArchUnit architecture tests
7. TestNG configuration (testng.xml)"
```
#### Example 8: Generate Tests for Specific Behavior
```bash
claude "@test create tests for Order.confirm() behavior:
1. Should confirm draft order with items
2. Should throw exception for empty order
3. Should throw exception for non-draft order
4. Should register OrderConfirmedEvent
5. Should update status to CONFIRMED
6. Parameterized test for invalid initial states"
```

### Code Quality Validator Agent (@quality)
#### Example 1: Validate Single File
```bash
claude "@quality validate file src/main/java/com/example/order/domain/model/Order.java:
1. Check for framework dependencies
2. Verify Java 21 features used
3. Check naming conventions
4. Verify immutability patterns
5. Check business logic presence (not anemic)"
```
#### Example 2: Validate Package
```bash
claude "@quality validate package com.example.order.domain:
1. All classes follow domain layer rules
2. Value objects are records
3. Aggregates have business methods
4. No public setters
5. Domain events properly defined"
```
#### Example 3: Validate Layer Compliance
```bash
claude "@quality validate application layer for Order:
1. Constructor injection only (no @Autowired fields)
2. All fields are final
3. DTOs are records
4. Transactions properly configured
5. No business logic (orchestration only)"
```
#### Example 4: Check Java 21 Compliance
```bash
claude "@quality check java21 compliance for project:
1. DTOs using records (not classes)
2. Value objects using records
3. Pattern matching used where appropriate
4. Sealed classes for finite hierarchies
5. Text blocks for multi-line strings
6. Virtual threads configuration"
```
#### Example 5: Check Spring Boot Patterns
```bash
claude "@quality check spring boot patterns:
1. Jakarta namespace (not javax)
2. Constructor injection everywhere
3. No @Data on entities
4. Proper Lombok usage
5. BOM dependency management"
```
#### Example 6: Validate API Layer
```bash
claude "@quality validate api layer for Order:
1. Controllers are thin (no business logic)
2. OpenAPI annotations present
3. @Valid on request parameters
4. Proper HTTP status codes
5. ProblemDetail for errors
6. Request/Response DTOs as records"
```
#### Example 7: Full Project Quality Review
```bash
claude "@quality full review for Order bounded context:
1. Style and naming conventions
2. Java 21 feature compliance
3. Architecture layer compliance
4. Spring Boot patterns
5. API development standards
6. Generate quality score and report"
```
#### Example 8: Pre-Commit Validation
```bash
claude "@quality pre-commit check:
1. Validate all modified files
2. Check for critical violations
3. Verify no framework code in domain
4. Check DTO immutability
5. Verify constructor injection
6. Block commit if critical issues found"
```
#### Example 9: Compare Before/After Refactoring
```bash
claude "@quality compare quality before and after:
1. Analyze current code quality score
2. Identify top 5 issues to fix
3. Suggest refactoring for each issue
4. Estimate new quality score after fixes"
```
#### Example 10: Generate Fix Suggestions
```bash
claude "@quality suggest fixes for Order component:
1. List all violations by severity
2. Provide before/after code for each fix
3. Prioritize fixes by impact
4. Estimate effort for each fix"
```

Expected Output: Validation report with pass/fail for each rule

## Agent Selection Guide

| Task | Primary Agent | Supporting Agent(s) |
|------|---------------|---------------------|
| Create new component | `@modular` | `@validate`, `@quality` |
| Create bounded context | `@ddd` | `@validate`, `@quality` |
| Validate architecture | `@validate` | - |
| Code review | `@quality` | `@validate` |
| Write unit tests | `@test` | - |
| Write integration tests | `@test` | - |
| Pre-commit check | `@quality` | `@validate` |
| Refactoring | `@quality` | `@validate`, `@test` |
| Bug fix | `@test` | `@quality` |
| New API endpoint | `@modular` or `@ddd` | `@quality`, `@test` |


## Summary

| Agent | File | Trigger | Use Case |
|-------|------|---------|----------|
| Modular Monolith | `.claude/agents/modular-monolith-agent.md` | `@modular`, `@component` | Feature-based components |
| Rich Domain Model | `.claude/agents/rich-domain-model-agent.md` | `@ddd`, `@domain` | Complex DDD bounded contexts |
| Validator | `.claude/agents/architecture-validator-agent.md` | `@validate`, `@check` | Validate architectural rules |
| Testing | `.claude/agents/testing-agent.md` | `@test`, `@testing` | Write tests using TestNG |
| Code Quality | `.claude/agents/code-quality-validator-agent.md` | `@quality` | Code quality validation |
