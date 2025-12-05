
## 4. Project Structure & Layout
> Strictly follow this Hexagonal/Ports-and-Adapters package structure.   
> Ensure that inner layers (Domain) never import outer layers (Infrastructure/Interfaces).  

> Full Structure  
```
text
com.company.project/
├── domain/                        # No framework dependencies (Pure Java)
│   ├── model/                     # Aggregates, Entities, Value Objects
│   │   ├── Order.java             # Aggregate Root
│   │   ├── OrderItem.java         # Entity within Order aggregate
│   │   ├── OrderId.java           # Typed ID (Value Object)
│   │   ├── OrderStatus.java       # Enum or Sealed Interface
│   │   └── Money.java             # Value Object
│   ├── repository/                # Repository interfaces (Output Ports)
│   │   └── OrderRepository.java   # Interface only
│   ├── service/                   # Domain services (stateless logic)
│   │   └── PricingService.java    # Logic spanning multiple aggregates
│   └── event/                     # Domain events (Records)
│       ├── OrderSubmittedEvent.java
│       └── OrderCancelledEvent.java
├── application/                   # Application orchestration (Use Cases)
│   ├── usecase/                   # Command/Query handlers
│   │   ├── SubmitOrderUseCase.java
│   │   └── GetOrderDetailsUseCase.java
│   ├── port/                      # Input/Output port definitions
│   │   ├── input/
│   │   │   └── SubmitOrderPort.java
│   │   └── output/
│   │       └── PaymentGatewayPort.java
│   ├── dto/                       # Data Transfer Objects (Records)
│   │   ├── CreateOrderRequest.java
│   │   └── OrderResponse.java
│   └── mapper/                    # Entity <-> DTO mappers
│       └── OrderMapper.java
├── infrastructure/                # Framework adapters & Implementations
│   ├── persistence/               # Repository implementations
│   │   ├── JpaOrderRepository.java
│   │   ├── OrderJpaEntity.java    # JPA-specific entity (if separating)
│   │   └── OrderIdConverter.java  # JPA AttributeConverter
│   ├── messaging/                 # Kafka/RabbitMQ adapters
│   │   ├── OrderEventPublisher.java
│   │   └── KafkaOrderEventAdapter.java
│   ├── external/                  # External API clients
│   │   ├── PaymentGatewayAdapter.java
│   │   └── InventoryClientAdapter.java
│   └── config/                    # Spring configurations
│       ├── JpaConfig.java
│       ├── SecurityConfig.java
│       └── ObservabilityConfig.java
└── interfaces/                    # Primary Adapters / Entry points
    ├── rest/                      # REST controllers
    │   ├── OrderController.java
    │   └── GlobalExceptionHandler.java
    └── messaging/                 # Message listeners
        └── OrderEventListener.java
```        

> Simple Component Structure 

```
text
com.company.project/
├── domain_one/                    
│   ├── model/                     # Aggregates, Entities, Value Objects
│   ├── repository/                # Repository interfaces (Output Ports)
│   ├── service/                   # Domain services (stateless logic)
│   ├── api/                       # API endpoints
│   ├── port/                      # Input/Output port definitions
│   ├── dto/                       # Data Transfer Objects (Records)
│   └── mapper/                    # Entity <-> DTO mappers
│   ├── persistence/               # Repository implementations
│   ├── messaging/                 # Kafka/RabbitMQ adapters
│   ├── external/                  # External API clients
│   └── config/                    # Spring configurations
├── domain_Two/   
```        

> Layer Dependency Rules  

```
text
┌─────────────────────────────────────────────────────────────┐
│                      INTERFACES                             │
│              (REST Controllers, Message Listeners)          │
└─────────────────────────┬───────────────────────────────────┘
                          │ depends on
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      APPLICATION                            │
│                (Use Cases, DTOs, Mappers)                   │
└─────────────────────────┬───────────────────────────────────┘
                          │ depends on
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                        DOMAIN                               │
│        (Entities, Value Objects, Repository Interfaces)     │
└─────────────────────────────────────────────────────────────┘
                          ▲
                          │ implements
┌─────────────────────────┴───────────────────────────────────┐
│                     INFRASTRUCTURE                          │
│          (JPA Repositories, External Clients, Configs)      │
└─────────────────────────────────────────────────────────────┘
```