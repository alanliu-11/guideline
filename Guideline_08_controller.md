## 8. REST Controller (Interface Layer)

### 8.1 Controller Implementation
> Controllers are thin - they only validate input and delegate to use cases.  
> Use OpenAPI annotations for API documentation.

```
java
// Located in: interfaces/rest/

@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Tag(name = "Orders", description = "Order management endpoints")
public class OrderController {
    
    private final SubmitOrderUseCase submitOrderUseCase;
    private final GetOrderDetailsUseCase getOrderDetailsUseCase;
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(summary = "Submit a new order")
    public OrderResponse submitOrder(@Valid @RequestBody SubmitOrderCommand command) {
        return submitOrderUseCase.execute(command);
    }
    
    @GetMapping("/{orderId}")
    @Operation(summary = "Get order details by ID")
    public OrderResponse getOrder(@PathVariable UUID orderId) {
        return getOrderDetailsUseCase.execute(orderId);
    }
    
    @GetMapping
    @Operation(summary = "Get orders by customer")
    public List<OrderSummaryResponse> getOrdersByCustomer(
            @RequestParam UUID customerId) {
        return getOrderDetailsUseCase.executeByCustomer(customerId);
    }
}
```

### 8.2 Global Exception Handler
> Use ProblemDetail (RFC 7807) for consistent error responses.  
> Map domain exceptions to appropriate HTTP status codes.

```
java
// Located in: interfaces/rest/

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    
    // Domain Exceptions -> 4xx Client Errors
    @ExceptionHandler(EntityNotFoundException.class)
    public ProblemDetail handleNotFound(EntityNotFoundException ex) {
        log.warn("Entity not found: {}", ex.getMessage());
        
        var problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, 
            ex.getMessage()
        );
        problem.setTitle("Resource Not Found");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
    
    @ExceptionHandler(DomainRuleException.class)
    public ProblemDetail handleDomainRule(DomainRuleException ex) {
        log.warn("Domain rule violation: {}", ex.getMessage());
        
        var problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.UNPROCESSABLE_ENTITY, 
            ex.getMessage()
        );
        problem.setTitle("Business Rule Violation");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
    
    @ExceptionHandler(InsufficientStockException.class)
    public ProblemDetail handleInsufficientStock(InsufficientStockException ex) {
        log.warn("Insufficient stock: {}", ex.getMessage());
        
        var problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT, 
            ex.getMessage()
        );
        problem.setTitle("Insufficient Stock");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
    
    // Validation Exceptions
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {
        
        var errors = ex.getBindingResult().getFieldErrors().stream()
            .map(error -> new ValidationError(
                error.getField(), 
                error.getDefaultMessage(),
                error.getRejectedValue()))
            .toList();
        
        var problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, 
            "Validation failed"
        );
        problem.setTitle("Validation Error");
        problem.setProperty("errors", errors);
        problem.setProperty("timestamp", Instant.now());
        
        return ResponseEntity.badRequest().body(problem);
    }
    
    // Catch-all for unexpected errors
    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception ex) {
        log.error("Unexpected error occurred", ex);
        
        var problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, 
            "An unexpected error occurred. Please try again later."
        );
        problem.setTitle("Internal Server Error");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
    
    public record ValidationError(String field, String message, Object rejectedValue) {}
}
```
