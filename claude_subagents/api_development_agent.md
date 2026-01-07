# Claude Sub-Agent: API Development Agent

## Agent Role
Specialized agent for implementing REST Controllers, API endpoints, and OpenAPI documentation in Spring Boot applications.

## When to Invoke
Invoke this agent when:
- Creating REST endpoints
- Implementing controllers
- Designing API contracts
- Writing OpenAPI documentation
- Handling request/response DTOs
- Implementing API validation
- Setting up error handling

## Controller Implementation Principles

### Core Principles
1. **Controllers are thin** - They only validate input and delegate to use cases
2. **Use OpenAPI annotations** - Comprehensive API documentation
3. **Always disable open-in-view** - For production environments
4. **Delegate to use cases** - Controllers don't contain business logic

### Basic Controller Structure
```java
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
}
```

## OpenAPI Documentation

### Controller-Level Annotations
```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Tag(name = "Orders", description = "Order management operations")
public class OrderController {
    
    private final SubmitOrderUseCase submitOrderUseCase;
    private final GetOrderDetailsUseCase getOrderDetailsUseCase;
    private final CancelOrderUseCase cancelOrderUseCase;
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(
        summary = "Submit a new order",
        description = "Creates and submits a new order for the specified customer"
    )
    @ApiResponses({
        @ApiResponse(
            responseCode = "201",
            description = "Order submitted successfully",
            content = @Content(schema = @Schema(implementation = OrderResponse.class))
        ),
        @ApiResponse(
            responseCode = "400",
            description = "Invalid request data",
            content = @Content(schema = @Schema(implementation = ProblemDetail.class))
        ),
        @ApiResponse(
            responseCode = "404",
            description = "Customer not found",
            content = @Content(schema = @Schema(implementation = ProblemDetail.class))
        ),
        @ApiResponse(
            responseCode = "409",
            description = "Insufficient stock",
            content = @Content(schema = @Schema(implementation = ProblemDetail.class))
        ),
        @ApiResponse(
            responseCode = "422",
            description = "Business rule violation",
            content = @Content(schema = @Schema(implementation = ProblemDetail.class))
        )
    })
    public OrderResponse submitOrder(
            @Valid @RequestBody 
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "Order submission request",
                required = true
            ) 
            SubmitOrderCommand command) {
        return submitOrderUseCase.execute(command);
    }
    
    @GetMapping("/{orderId}")
    @Operation(summary = "Get order by ID")
    @ApiResponses({
        @ApiResponse(
            responseCode = "200",
            description = "Order found",
            content = @Content(schema = @Schema(implementation = OrderResponse.class))
        ),
        @ApiResponse(
            responseCode = "404",
            description = "Order not found"
        )
    })
    public OrderResponse getOrder(
            @PathVariable 
            @Parameter(description = "Order unique identifier", example = "550e8400-e29b-41d4-a716-446655440000")
            UUID orderId) {
        return getOrderDetailsUseCase.execute(orderId);
    }
    
    @DeleteMapping("/{orderId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @Operation(summary = "Cancel an order")
    @ApiResponses({
        @ApiResponse(responseCode = "204", description = "Order cancelled successfully"),
        @ApiResponse(responseCode = "404", description = "Order not found"),
        @ApiResponse(responseCode = "422", description = "Order cannot be cancelled")
    })
    public void cancelOrder(
            @PathVariable UUID orderId,
            @RequestParam 
            @Parameter(description = "Cancellation reason", example = "Customer changed their mind")
            String reason) {
        cancelOrderUseCase.execute(new CancelOrderCommand(orderId, reason));
    }
}
```

## DTO Schema Annotations

### Request DTOs
```java
@Schema(description = "Request to submit a new order")
public record SubmitOrderCommand(
    
    @Schema(
        description = "Unique identifier of the customer placing the order",
        example = "550e8400-e29b-41d4-a716-446655440000",
        requiredMode = Schema.RequiredMode.REQUIRED
    )
    @NotNull(message = "Customer ID is required")
    UUID customerId,
    
    @Schema(
        description = "List of items to include in the order",
        minLength = 1
    )
    @NotEmpty(message = "Order must have at least one item")
    List<@Valid OrderItemCommand> items
    
) {}

@Schema(description = "Individual item in an order")
public record OrderItemCommand(
    
    @Schema(
        description = "Product unique identifier",
        example = "123e4567-e89b-12d3-a456-426614174000"
    )
    @NotNull
    UUID productId,
    
    @Schema(
        description = "Quantity to order",
        minimum = "1",
        example = "2"
    )
    @Min(value = 1, message = "Quantity must be at least 1")
    int quantity
    
) {}
```

### Response DTOs
```java
@Schema(description = "Order details response")
public record OrderResponse(
    
    @Schema(description = "Order unique identifier")
    UUID id,
    
    @Schema(description = "Current order status", example = "SUBMITTED")
    String status,
    
    @Schema(description = "Total order amount", example = "99.99")
    BigDecimal totalAmount,
    
    @Schema(description = "Currency code", example = "USD")
    String currency,
    
    @Schema(description = "Order line items")
    List<OrderItemResponse> items,
    
    @Schema(description = "Timestamp when order was created")
    Instant createdAt
    
) {}
```

## HTTP Status Codes

### Standard Status Codes
- **200 OK** - Successful GET, PUT, PATCH requests
- **201 Created** - Successful POST requests that create resources
- **204 No Content** - Successful DELETE requests
- **400 Bad Request** - Invalid request data (validation errors)
- **404 Not Found** - Resource not found
- **409 Conflict** - Business conflict (e.g., insufficient stock)
- **422 Unprocessable Entity** - Business rule violation
- **500 Internal Server Error** - Unexpected server errors

### Example Usage
```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)  // 201
public OrderResponse createOrder(@Valid @RequestBody CreateOrderRequest request) {
    return orderService.createOrder(request);
}

@DeleteMapping("/{orderId}")
@ResponseStatus(HttpStatus.NO_CONTENT)  // 204
public void deleteOrder(@PathVariable UUID orderId) {
    orderService.deleteOrder(orderId);
}
```

## Request Validation

### Bean Validation
```java
public record CreateOrderRequest(
    @NotNull(message = "Customer ID is required")
    UUID customerId,
    
    @NotEmpty(message = "Order must have at least one item")
    @Valid
    List<OrderItemRequest> items
) {}

public record OrderItemRequest(
    @NotNull
    UUID productId,
    
    @Min(value = 1, message = "Quantity must be at least 1")
    int quantity,
    
    @NotNull
    @Positive(message = "Price must be positive")
    BigDecimal unitPrice
) {}
```

### Controller Validation
```java
@PostMapping
public ResponseEntity<OrderResponse> createOrder(
        @Valid @RequestBody CreateOrderRequest request) {
    // Validation happens automatically via @Valid
    OrderResponse response = orderService.createOrder(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

## Error Handling

### Global Exception Handler
```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> handleValidationErrors(
            MethodArgumentNotValidException ex) {
        ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problemDetail.setTitle("Validation Error");
        problemDetail.setDetail("Request validation failed");
        
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(error -> new FieldError(error.getField(), error.getDefaultMessage()))
            .toList();
        
        problemDetail.setProperty("errors", errors);
        return ResponseEntity.badRequest().body(problemDetail);
    }
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ProblemDetail> handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        problemDetail.setTitle("Resource Not Found");
        problemDetail.setDetail(ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(problemDetail);
    }
    
    @ExceptionHandler(DomainRuleException.class)
    public ResponseEntity<ProblemDetail> handleDomainRuleViolation(DomainRuleException ex) {
        ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        problemDetail.setTitle("Business Rule Violation");
        problemDetail.setDetail(ex.getMessage());
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(problemDetail);
    }
}
```

## API Mapper Pattern

### DTO to Domain Mapping
```java
@Component
public class OrderApiMapper {
    
    public OrderResponse toResponse(Order domain) {
        return new OrderResponse(
            domain.getId().value(),
            domain.getCustomerId(),
            domain.getStatus().name(),
            toLineResponses(domain.getLines()),
            domain.getTotalAmount().amount(),
            domain.getTotalAmount().currency().getCurrencyCode(),
            domain.getCreatedAt(),
            domain.getUpdatedAt(),
            domain.getTotalItems(),
            domain.canBeCancelled()
        );
    }
    
    public List<OrderLineResponse> toLineResponses(List<OrderLine> lines) {
        return lines.stream()
            .map(this::toLineResponse)
            .collect(Collectors.toList());
    }
    
    public List<OrderLineData> toLineData(List<OrderLineRequest> requests) {
        return requests.stream()
            .map(r -> new OrderLineData(
                r.productId(), 
                r.productName(), 
                r.quantity(), 
                r.unitPrice()
            ))
            .collect(Collectors.toList());
    }
}
```

## Implementation Checklist

- [ ] Controller is thin (delegates to use cases)
- [ ] OpenAPI annotations present on all endpoints
- [ ] All possible response codes documented
- [ ] DTOs annotated with @Schema
- [ ] Request validation via @Valid
- [ ] Proper HTTP status codes returned
- [ ] Global exception handler defined
- [ ] ProblemDetail used for error responses (RFC 7807)
- [ ] API mapper used for DTO ↔ Domain conversion
- [ ] No business logic in controllers
- [ ] open-in-view disabled for production

## Best Practices

1. **Keep Controllers Thin**: Controllers should only handle HTTP concerns
2. **Document Everything**: Use OpenAPI annotations comprehensively
3. **Validate Early**: Use Bean Validation on request DTOs
4. **Consistent Error Responses**: Use ProblemDetail (RFC 7807) for errors
5. **Proper Status Codes**: Use appropriate HTTP status codes
6. **Version APIs**: Use `/api/v1/` prefix for versioning
7. **Use Records for DTOs**: Immutable DTOs as Java records
8. **Separate Concerns**: Controllers don't know about domain models directly

