# Claude Skills Directory

This directory contains Claude Skills for Spring Boot application development. These skills provide reference knowledge that can be consulted during development, code review, and architectural decisions.

## Available Skills

### 1. `general_style_skill.md`
**Purpose:** Core coding standards, principles, and workflow guidelines
**Use When:** Writing code, reviewing code, or making style decisions
**Key Topics:**
- General principles (clarity, naming, simplicity)
- Code organization and formatting
- Comments and documentation
- Error handling
- Workflow and collaboration

### 2. `java21_language_skill.md`
**Purpose:** Java 21 (LTS) language feature guidelines
**Use When:** Writing Java code to ensure modern, idiomatic Java 21 usage
**Key Topics:**
- Records for immutable data
- Sealed classes/interfaces
- Pattern matching
- Virtual threads
- Text blocks
- Sequenced collections

### 3. `functional_programming_skill.md`
**Purpose:** Functional programming style guidelines
**Use When:** Writing code that processes data, handles collections, or manages state
**Key Topics:**
- Immutability
- Stream API usage
- Optional usage patterns
- When to use functional vs imperative style

### 4. `spring_boot_ecosystem_skill.md`
**Purpose:** Spring Boot 3.x ecosystem guidelines
**Use When:** Working with Spring Boot, dependency injection, or Jakarta EE features
**Key Topics:**
- Jakarta EE migration (jakarta.* vs javax.*)
- Dependency injection (constructor injection only)
- Lombok restrictions
- Dependency management (BOM)
- Observability

### 5. `quick_reference_skill.md`
**Purpose:** Quick reference for common patterns, templates, and checklists
**Use When:** Rapid lookups during code review or development
**Key Topics:**
- Do's and don'ts summary
- Layer responsibilities
- Record templates (Value Objects, IDs, Events, DTOs)
- Common patterns (Factory Method, Specification, Policy)
- Code review checklists

### 6. `refactoring_skill.md`
**Purpose:** Guidelines for when and how to refactor code
**Use When:** Improving code structure without changing behavior
**Key Topics:**
- When to refactor (green bar rule)
- Golden rules of refactoring
- Common refactoring patterns
- Safe refactoring process
- Code smells to watch for

### 7. `design_patterns_skill.md`
**Purpose:** Guidance on when and how to use design patterns
**Use When:** Solving design problems or choosing architectural approaches
**Key Topics:**
- When to use patterns
- Creational patterns (Factory, Builder)
- Structural patterns (Adapter, DI, Repository)
- Behavioral patterns (Strategy, Template Method, Observer)
- Anti-patterns to avoid

### 8. `architectural_decisions_skill.md`
**Purpose:** Framework for documenting architectural and design decisions
**Use When:** Making significant technical decisions
**Key Topics:**
- What is an ADR
- When to create an ADR
- ADR template
- Integration with development

## How to Use These Skills

1. **Reference During Development:** Consult relevant skills when writing code
2. **Code Review:** Use checklists and guidelines from skills during reviews
3. **Architectural Decisions:** Use ADR skill when making significant decisions
4. **Refactoring:** Follow refactoring skill guidelines when improving code
5. **Pattern Selection:** Consult design patterns skill before applying patterns

## Skill Organization

Skills are organized by topic and purpose:
- **Core Skills:** General style, Java 21, Functional programming
- **Framework Skills:** Spring Boot ecosystem
- **Reference Skills:** Quick reference, checklists
- **Process Skills:** Refactoring, design patterns, ADRs

## Related Resources

- **Sub-agents:** See `claude_sub_agents/` for specialized task agents
- **MCPs:** See `mcps/` for tools and capabilities
- **Source Guidelines:** See `knowledge_spring/` for original guideline files

