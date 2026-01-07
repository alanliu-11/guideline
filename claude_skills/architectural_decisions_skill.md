# Claude Skill: Architectural Decision Records (ADRs)

## Purpose
This skill provides a framework for documenting architectural and design decisions. Use this when making significant technical decisions that affect the codebase structure or approach.

## What is an ADR?

An Architectural Decision Record (ADR) is a document that captures an important architectural decision made along with its context and consequences.

## When to Create an ADR

Create an ADR when you make decisions about:
- Architecture patterns (e.g., choosing DDD over anemic domain model)
- Technology choices (e.g., choosing TestNG over JUnit)
- Framework decisions (e.g., choosing Spring Boot over Quarkus)
- Design patterns (e.g., choosing Repository pattern over Active Record)
- Significant refactoring approaches
- Trade-offs between different solutions

## ADR Template

```markdown
# ADR-XXX: [Short Title]

## Status
[Proposed | Accepted | Deprecated | Superseded]

## Context
[Describe the issue motivating this decision or change]

## Decision
[Describe the change that you're proposing or have decided to implement]

## Consequences
[Describe the resulting context, after applying the decision]

## Alternatives Considered
[Describe alternative options that were considered]

## Notes
[Any additional notes or references]
```

## Example ADR

```markdown
# ADR-001: Use Java Records for Value Objects

## Status
Accepted

## Context
We need to represent immutable value objects in our domain model. 
Traditional approaches include:
- Mutable classes with getters/setters
- Lombok @Value annotation
- Immutable classes with final fields

## Decision
Use Java 21 Records for all Value Objects in the domain model.

## Consequences
**Positive:**
- Built-in immutability
- Concise syntax
- Automatic equals/hashCode/toString
- Pattern matching support
- No external dependencies

**Negative:**
- Requires Java 21+
- Less flexibility than classes (but we don't need it)

## Alternatives Considered
1. Lombok @Value - Adds dependency, less explicit
2. Immutable classes - More verbose, manual equals/hashCode
3. Mutable classes - Violates immutability principle

## Notes
This aligns with our Java 21 baseline and functional programming style.
```

## Key Principles

1. **Document decisions, not discussions** - Focus on what was decided, not all the debate
2. **Keep it concise** - ADRs should be brief and to the point
3. **Update status** - Mark ADRs as deprecated or superseded when decisions change
4. **Link related ADRs** - Reference other ADRs when decisions are related
5. **Review periodically** - Revisit ADRs to ensure they're still relevant

## Integration with Development

- Create ADR when making significant architectural decisions
- Reference ADR in code comments when the decision affects implementation
- Update ADR status when decisions change
- Review ADRs during architecture reviews

