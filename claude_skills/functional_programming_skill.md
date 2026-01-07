# Claude Skill: Functional Programming Style

## Purpose
This skill provides guidelines for writing functional-style code in Java. Use this when writing code that processes data, handles collections, or manages state.

## Immutability

**Components:**
- Fields in Service/Component classes must always be `final`
- Data Structures: DTOs and Value Objects must be immutable
- Collections: Return unmodifiable collections from methods

**Example:**
```java
public List<OrderItem> getItems() {
    return List.copyOf(this.items); // Defensive copy
}
```

## Stream API

**Preference:** Prefer the Stream API for collection processing.

**Pragmatism:** Use traditional for loops if:
- Performance is critical (hot paths)
- Readability suffers in complex stream chains
- Early exit with complex state is needed

## Optional Usage

**Use Optional strictly for return types to signal potential absence.**

**Constraint: Never use Optional as:**
- Method parameters
- Class fields
- Collection elements

**Correct Usage:**
```java
// ✅ Correct: Return type
public Optional<Customer> findByEmail(Email email);
```

**Wrong Usage:**
```java
// ❌ Wrong: Parameter
public void process(Optional<String> name); // NEVER

// ❌ Wrong: Field
private Optional<Address> address; // NEVER
```

## Key Principles

1. **Immutability First:** Make data structures immutable by default
2. **Defensive Copies:** Return copies of internal collections
3. **Streams for Transformation:** Use Stream API for data processing
4. **Optional for Absence:** Use Optional only as return types
5. **Final Fields:** All service/component fields should be final

## When to Use Functional Style

**Use functional style when:**
- Transforming collections
- Filtering data
- Mapping between types
- Reducing/aggregating values
- Processing data pipelines

**Avoid functional style when:**
- Performance is critical and imperative code is faster
- Complex state management is needed
- Early exit conditions are complex
- Readability suffers

