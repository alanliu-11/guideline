## 10. Complete Example Summary

### 10.1 Full Request Flow Diagram
> Request flow follows hexagonal architecture layers.  
> Controllers delegate to use cases, which orchestrate domain operations.

```
text
┌──────────────────────────────────────────────────────────────────────────────┐
│                              REQUEST FLOW                                    │
└──────────────────────────────────────────────────────────────────────────────┘

  HTTP Request
       │
       ▼
┌─────────────────┐
│ OrderController │  (interfaces/rest/)
│  - Validates    │
│  - Delegates    │
└────────┬────────┘
         │ SubmitOrderCommand
         ▼
┌─────────────────────┐
│ SubmitOrderUseCase  │  (application/usecase/)
│  - Orchestrates     │
│  - Transaction      │
└────────┬────────────┘
         │
    ┌────┴────┬───────────────┐
    ▼         ▼               ▼
┌────────┐ ┌────────┐ ┌──────────────┐
│Customer│ │ Order  │ │InventoryPort│  (domain/ & application/port/)
│  Repo  │ │  Repo  │ └──────────────┘
└───┬────┘ └───┬────┘         │
    │          │              │
    ▼          ▼              ▼
┌────────────────────────────────────┐
│          INFRASTRUCTURE            │
│  - JpaCustomerRepository           │  (infrastructure/)
│  - JpaOrderRepository              │
│  - InventoryClientAdapter          │
└────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│    Database     │
│    External     │
│    Services     │
└─────────────────┘
```

### 10.2 Complete File Listing Example
> Standard package structure following DDD and hexagonal architecture.  
> Clear separation between domain, application, infrastructure, and interfaces.

```
text
src/main/java/com/company/orderservice/
│
├── OrderServiceApplication.java
│
├── domain/
│   ├── model/
│   │   ├── Order.java
│   │   ├── OrderItem.java
│   │   ├── OrderId.java
│   │   ├── OrderStatus.java
│   │   ├── CustomerId.java
│   │   ├── ProductId.java
│   │   ├── Money.java
│   │   ├── Currency.java
│   │   └── Quantity.java
│   ├── repository/
│   │   ├── OrderRepository.java
│   │   └── CustomerRepository.java
│   ├── service/
│   │   └── PricingService.java
│   ├── event/
│   │   ├── OrderSubmittedEvent.java
│   │   ├── OrderCancelledEvent.java
│   │   └── OrderShippedEvent.java
│   └── exception/
│       ├── DomainException.java
│       ├── DomainRuleException.java
│       ├── EntityNotFoundException.java
│       ├── OrderNotFoundException.java
│       ├── CustomerNotFoundException.java
│       ├── InsufficientStockException.java
│       └── CurrencyMismatchException.java
│
├── application/
│   ├── usecase/
│   │   ├── SubmitOrderUseCase.java
│   │   ├── CancelOrderUseCase.java
│   │   ├── GetOrderDetailsUseCase.java
│   │   └── ListCustomerOrdersUseCase.java
│   ├── port/
│   │   ├── input/
│   │   │   ├── SubmitOrderPort.java
│   │   │   └── CancelOrderPort.java
│   │   └── output/
│   │       ├── PaymentGatewayPort.java
│   │       ├── InventoryPort.java
│   │       └── NotificationPort.java
│   ├── dto/
│   │   ├── command/
│   │   │   ├── SubmitOrderCommand.java
│   │   │   ├── CancelOrderCommand.java
│   │   │   └── OrderItemCommand.java
│   │   └── response/
│   │       ├── OrderResponse.java
│   │       ├── OrderSummaryResponse.java
│   │       └── OrderItemResponse.java
│   ├── mapper/
│   │   └── OrderMapper.java
│   └── eventhandler/
│       ├── OrderSubmittedEventHandler.java
│       └── OrderCancelledEventHandler.java
│
├── infrastructure/
│   ├── persistence/
│   │   ├── JpaOrderRepository.java
│   │   ├── OrderRepositoryAdapter.java
│   │   ├── JpaCustomerRepository.java
│   │   ├── CustomerRepositoryAdapter.java
│   │   └── converter/
│   │       ├── OrderIdConverter.java
│   │       ├── CustomerIdConverter.java
│   │       └── MoneyConverter.java
│   ├── messaging/
│   │   ├── kafka/
│   │   │   ├── KafkaOrderEventPublisher.java
│   │   │   └── KafkaConfig.java
│   │   └── rabbitmq/
│   │       └── RabbitMQOrderEventPublisher.java
│   ├── external/
│   │   ├── payment/
│   │   │   ├── StripePaymentGatewayAdapter.java
│   │   │   ├── StripeClient.java
│   │   │   └── StripeConfig.java
│   │   ├── inventory/
│   │   │   ├── InventoryServiceAdapter.java
│   │   │   └── InventoryClient.java
│   │   └── notification/
│   │       ├── EmailNotificationAdapter.java
│   │       └── SendGridClient.java
│   └── config/
│       ├── JpaConfig.java
│       ├── JacksonConfig.java
│       ├── SecurityConfig.java
│       ├── ObservabilityConfig.java
│       └── NativeImageConfig.java
│
└── interfaces/
    ├── rest/
    │   ├── OrderController.java
    │   ├── CustomerController.java
    │   ├── GlobalExceptionHandler.java
    │   └── ApiDocumentationConfig.java
    └── messaging/
        ├── OrderEventListener.java
        └── InventoryEventListener.java
```
