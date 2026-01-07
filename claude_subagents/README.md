# Claude Sub-Agents Directory

This directory contains specialized Claude Sub-Agents for specific development tasks in Spring Boot applications. These agents are designed to be invoked when working on particular aspects of application development.

## Available Sub-Agents

### 1. `architecture_design_agent.md`
**Role:** Specialized agent for designing and implementing Spring Boot application architecture using Domain-Driven Design (DDD) principles.

**Invoke When:**
- Designing new features or components
- Setting up project structure
- Implementing domain models
- Creating bounded contexts
- Designing component boundaries
- Separating domain logic from infrastructure

**Key Capabilities:**
- Modular Monolith architecture (Component-Based)
- Rich Domain Model architecture (Full DDD)
- Domain layer implementation (Value Objects, Aggregates, Repository Ports)
- Infrastructure layer implementation (Adapters, Mappers)
- Application layer implementation (Use Cases)
- Component boundary design
- Layer dependency rules

**Architecture Patterns:**
- Package-by-Feature with strict boundaries
- Vertical Slice Architecture
- Ports and Adapters pattern
- Repository pattern
- Domain Events

### 2. `api_development_agent.md`
**Role:** Specialized agent for implementing REST Controllers, API endpoints, and OpenAPI documentation.

**Invoke When:**
- Creating REST endpoints
- Implementing controllers
- Designing API contracts
- Writing OpenAPI documentation
- Handling request/response DTOs
- Implementing API validation
- Setting up error handling

**Key Capabilities:**
- Thin controller implementation
- OpenAPI 3 documentation
- Request/Response DTO design
- Bean Validation
- HTTP status code management
- Global exception handling
- API mapper pattern
- ProblemDetail (RFC 7807) error responses

**Best Practices:**
- Controllers delegate to use cases
- Comprehensive OpenAPI annotations
- Early validation with Bean Validation
- Consistent error responses
- Proper HTTP status codes
- API versioning

### 3. `testing_agent.md`
**Role:** Specialized agent for writing tests using TestNG framework.

**Invoke When:**
- Writing unit tests for domain models
- Testing use cases and application services
- Writing integration tests
- Testing REST controllers
- Setting up test fixtures
- Writing architecture tests with ArchUnit
- Configuring TestNG

**Key Capabilities:**
- Framework-free domain unit tests
- Use case testing with mocks
- Integration testing with Testcontainers
- REST controller testing with MockMvc
- Test Object Mothers (fixtures)
- Architecture testing with ArchUnit
- TestNG-specific features (groups, dependencies, data providers)

**Testing Levels:**
- Unit tests (Domain layer)
- Use case tests (Application layer)
- Integration tests (Infrastructure layer)
- Controller tests (API layer)
- Architecture tests (ArchUnit)

### 4. `code_quality_validator_agent.md`
**Role:** Specialized agent for validating code against all Spring Boot development guidelines.

**Invoke When:**
- Reviewing code before committing
- Performing code quality checks
- Validating code against guidelines
- Checking code style compliance
- Ensuring best practices are followed
- Before code reviews

**Key Capabilities:**
- Comprehensive code validation against all guidelines
- Style compliance checking
- Language feature validation
- Framework compliance verification
- Architecture compliance checking
- API compliance validation
- Testing compliance verification
- Detailed validation reports with recommendations

**Validation Scope:**
- General style guidelines
- Java 21 language standards
- Functional programming style
- Spring Boot ecosystem
- Architecture guidelines
- API development
- Testing guidelines

### 5. `architecture_validator_agent.md`
**Role:** Specialized agent for validating architectural rules and enforcing DDD patterns.

**Invoke When:**
- Validating layer dependencies
- Checking architectural boundaries
- Enforcing DDD patterns
- Verifying component isolation
- Validating repository pattern implementation
- Checking domain model purity
- Before architectural refactoring

**Key Capabilities:**
- Dependency rule validation (Clean Architecture)
- Layer access rule enforcement
- Component boundary validation
- Repository pattern validation
- Domain model purity checks
- ArchUnit test generation
- Architectural violation reports

**Architectural Rules:**
- Dependency rule (all dependencies point inward)
- Layer access rules
- Component boundary rules
- Repository pattern rules
- Domain model purity rules
- DTO boundary rules

## How to Use Sub-Agents

### Invocation Pattern
When working on a specific task, invoke the relevant sub-agent by referencing its guidelines:

1. **Architecture Design Tasks:**
   - "Using the Architecture Design Agent, design a new Order component..."
   - "Following the Architecture Design Agent guidelines, implement a rich domain model for..."

2. **API Development Tasks:**
   - "Using the API Development Agent, create a REST endpoint for..."
   - "Following the API Development Agent, document this endpoint with OpenAPI..."

3. **Testing Tasks:**
   - "Using the Testing Agent, write unit tests for..."
   - "Following the Testing Agent guidelines, create integration tests for..."

### Agent Collaboration
Sub-agents work together with Claude Skills:
- **Skills** provide reference knowledge (coding standards, patterns, checklists)
- **Sub-agents** provide task-specific guidance and implementation patterns

Example workflow:
1. Reference `general_style_skill.md` for coding standards
2. Invoke `architecture_design_agent.md` for architecture design
3. Reference `java21_language_skill.md` for Java 21 features
4. Invoke `api_development_agent.md` for REST endpoint implementation
5. Invoke `testing_agent.md` for writing tests

## Agent Responsibilities

### Architecture Design Agent
- ✅ Component/bounded context design
- ✅ Domain model implementation
- ✅ Layer structure and boundaries
- ✅ Repository pattern implementation
- ✅ Component communication patterns

### API Development Agent
- ✅ REST controller implementation
- ✅ OpenAPI documentation
- ✅ Request/response DTOs
- ✅ Validation and error handling
- ✅ HTTP status codes

### Testing Agent
- ✅ Unit test implementation
- ✅ Integration test setup
- ✅ Test fixtures and Object Mothers
- ✅ Architecture test rules
- ✅ TestNG configuration

## Integration with Skills

Sub-agents complement Claude Skills:
- **Skills** = Reference knowledge (what to do)
- **Sub-agents** = Task-specific guidance (how to do it)

For example:
- `java21_language_skill.md` tells you to use Records
- `architecture_design_agent.md` shows you how to implement Value Objects as Records in your domain model

## Related Resources

- **Claude Skills:** See `claude_skills/` for reference knowledge
- **MCPs:** See `mcps/` for tools and capabilities
- **Source Guidelines:** See `knowledge_spring/` for original guideline files

