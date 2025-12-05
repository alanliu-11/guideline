
## 2. Functional Coding Style
### 2.1 Immutability
> Components: Fields in Service/Component classes must always be final.
> Data Structures: DTOs and Value Objects must be immutable.
> Collections: Return unmodifiable collections from methods.
```
java
public List<OrderItem> getItems() {
    return List.copyOf(this.items); // Defensive copy
}
```

### 2.2 Stream API
> Preference: Prefer the Stream API for collection processing.
> Pragmatism: Use traditional for loops if:
> Performance is critical (hot paths).
> Readability suffers in complex stream chains.
> Early exit with complex state is needed.

```
java
// Good: Simple, readable stream
var activeCustomerEmails = customers.stream()
    .filter(Customer::isActive)
    .map(Customer::email)
    .toList();

// Acceptable: Complex logic, use loop for clarity
List<ProcessedOrder> results = new ArrayList<>();
for (Order order : orders) {
    if (order.requiresSpecialHandling()) {
        var processed = processWithRetry(order);
        if (processed.failed()) {
            handleFailure(processed);
            continue;
        }
        results.add(processed);
    }
}
```

### 2.3 Optional Usage
> Use Optional strictly for return types to signal potential absence.
> Constraint: Never use Optional as:
  > Method parameters.
  > Class fields.
  > Collection elements.

```  
java
// ✅ Correct: Return type
public Optional<Customer> findByEmail(Email email);

// ❌ Wrong: Parameter
public void process(Optional<String> name); // NEVER

// ❌ Wrong: Field
private Optional<Address> address; // NEVER
```

