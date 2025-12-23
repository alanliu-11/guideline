## 6. REST Controller (Interface Layer)

### 6.1 Controller Implementation
> Controllers are thin - they only validate input and delegate to use cases.  
> Use OpenAPI annotations for API documentation.
> Always disable open-in-view for production.  

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
### 6.1 API Documentation (OpenAPI)

#### 6.1.1 OpenAPI Configuration
> Use OpenAPI 3 for comprehensive API documentation.  
> Annotate controllers with OpenAPI annotations for comprehensive documentation.  
> Document all possible response codes and error scenarios.

```
java
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

#### 6.1.2 DTO Schema Annotations
> Annotate DTOs with @Schema for better API documentation.  
> Provide examples and descriptions for all fields.

```
java
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
