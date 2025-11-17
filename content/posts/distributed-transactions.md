---
title: "Understanding Distributed Transactions: 2PC and Saga Patterns"
date: 2025-01-17
tags: ["distributed-systems", "java", "spring-boot", "postgresql", "transactions", "microservices"]
authors: ["Hao"]
---

# Understanding Distributed Transactions: 2PC and Saga Patterns

In modern microservices architectures, managing transactions across multiple services and databases is one of the most challenging problems. Unlike monolithic applications where a single database transaction can maintain ACID properties, distributed systems require sophisticated patterns to ensure data consistency. In this post, we'll explore two fundamental approaches: Two-Phase Commit (2PC) and the Saga pattern, with practical examples using Java Spring Boot and PostgreSQL.

## The Challenge of Distributed Transactions

Imagine an e-commerce system split into multiple microservices:
- **Order Service**: Manages customer orders
- **Payment Service**: Processes payments
- **Inventory Service**: Tracks product stock

When a customer places an order, we need to:
1. Create an order record
2. Process the payment
3. Reduce inventory

All these operations must succeed or fail together. If payment succeeds but inventory update fails, we have inconsistent data. This is where distributed transaction patterns come in.

## Two-Phase Commit (2PC)

### Overview

Two-Phase Commit is a classic distributed transaction protocol that ensures all participants in a transaction either commit or rollback together. It works in two phases:

**Phase 1 - Prepare Phase:**
- The coordinator asks all participants to prepare for commit
- Each participant performs the operation but doesn't commit
- Participants respond with "ready" or "abort"

**Phase 2 - Commit Phase:**
- If all participants are ready, coordinator sends commit
- Otherwise, coordinator sends rollback to all

### Spring Boot Implementation with PostgreSQL

Let's implement 2PC using Spring Boot's JTA (Java Transaction API) support with Atomikos transaction manager.

#### 1. Dependencies (pom.xml)

```xml
<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- JTA with Atomikos -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jta-atomikos</artifactId>
    </dependency>

    <!-- PostgreSQL Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
</dependencies>
```

#### 2. Configuration

```java
@Configuration
@EnableTransactionManagement
public class DatabaseConfig {

    @Bean(name = "orderDataSource")
    @Primary
    public DataSource orderDataSource() {
        AtomikosDataSourceBean dataSource = new AtomikosDataSourceBean();
        dataSource.setUniqueResourceName("orderDB");
        dataSource.setXaDataSourceClassName("org.postgresql.xa.PGXADataSource");

        Properties properties = new Properties();
        properties.setProperty("url", "jdbc:postgresql://localhost:5432/orderdb");
        properties.setProperty("user", "postgres");
        properties.setProperty("password", "password");

        dataSource.setXaProperties(properties);
        dataSource.setMaxPoolSize(10);
        return dataSource;
    }

    @Bean(name = "inventoryDataSource")
    public DataSource inventoryDataSource() {
        AtomikosDataSourceBean dataSource = new AtomikosDataSourceBean();
        dataSource.setUniqueResourceName("inventoryDB");
        dataSource.setXaDataSourceClassName("org.postgresql.xa.PGXADataSource");

        Properties properties = new Properties();
        properties.setProperty("url", "jdbc:postgresql://localhost:5432/inventorydb");
        properties.setProperty("user", "postgres");
        properties.setProperty("password", "password");

        dataSource.setXaProperties(properties);
        dataSource.setMaxPoolSize(10);
        return dataSource;
    }

    @Bean
    public JtaTransactionManager transactionManager() {
        return new JtaTransactionManager();
    }
}
```

#### 3. Entity Classes

```java
// Order Entity
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String customerId;
    private String productId;
    private Integer quantity;
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    // Getters and setters
}

// Inventory Entity
@Entity
@Table(name = "inventory")
public class Inventory {
    @Id
    private String productId;

    private Integer availableStock;
    private Integer reservedStock;

    // Getters and setters
}
```

#### 4. Service Implementation

```java
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private InventoryRepository inventoryRepository;

    @Transactional
    public Order createOrder(OrderRequest request) {
        // Phase 1: Both operations are prepared
        // Create order
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setProductId(request.getProductId());
        order.setQuantity(request.getQuantity());
        order.setAmount(request.getAmount());
        order.setStatus(OrderStatus.PENDING);

        order = orderRepository.save(order);

        // Reserve inventory
        Inventory inventory = inventoryRepository
            .findById(request.getProductId())
            .orElseThrow(() -> new RuntimeException("Product not found"));

        if (inventory.getAvailableStock() < request.getQuantity()) {
            throw new InsufficientStockException("Not enough stock available");
        }

        inventory.setAvailableStock(
            inventory.getAvailableStock() - request.getQuantity()
        );
        inventory.setReservedStock(
            inventory.getReservedStock() + request.getQuantity()
        );

        inventoryRepository.save(inventory);

        // Phase 2: If we reach here, both operations commit together
        // If any exception occurs, both operations rollback
        order.setStatus(OrderStatus.CONFIRMED);
        return orderRepository.save(order);
    }
}
```

### Pros and Cons of 2PC

**Advantages:**
- Strong consistency guarantee
- ACID properties maintained across distributed resources
- Straightforward to understand and implement

**Disadvantages:**
- Blocking protocol - participants must wait for coordinator
- Single point of failure (coordinator)
- Not suitable for high-latency networks
- Performance overhead
- Doesn't scale well with many participants

## Saga Pattern

### Overview

The Saga pattern takes a different approach. Instead of locking resources and coordinating a single distributed transaction, it breaks the operation into a series of local transactions, each with a compensating transaction that can undo the changes.

There are two main implementations:
1. **Choreography**: Each service publishes events and listens to others
2. **Orchestration**: A central coordinator manages the saga

### Saga with Orchestration (Spring Boot)

Let's implement an orchestration-based saga for our order processing example.

#### 1. Saga State Machine

```java
public enum SagaStep {
    CREATE_ORDER,
    RESERVE_INVENTORY,
    PROCESS_PAYMENT,
    CONFIRM_ORDER,
    COMPLETED
}

public enum SagaStatus {
    STARTED,
    IN_PROGRESS,
    COMPLETED,
    FAILED,
    COMPENSATING,
    COMPENSATED
}
```

#### 2. Saga Orchestrator

```java
@Service
public class OrderSagaOrchestrator {

    @Autowired
    private OrderService orderService;

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private PaymentService paymentService;

    @Autowired
    private SagaStateRepository sagaStateRepository;

    public OrderResponse processOrder(OrderRequest request) {
        SagaState sagaState = initializeSaga(request);

        try {
            // Step 1: Create Order
            Order order = executeStep(sagaState, SagaStep.CREATE_ORDER,
                () -> orderService.createOrder(request));

            // Step 2: Reserve Inventory
            executeStep(sagaState, SagaStep.RESERVE_INVENTORY,
                () -> inventoryService.reserveStock(
                    order.getProductId(),
                    order.getQuantity()
                ));

            // Step 3: Process Payment
            executeStep(sagaState, SagaStep.PROCESS_PAYMENT,
                () -> paymentService.processPayment(
                    order.getCustomerId(),
                    order.getAmount()
                ));

            // Step 4: Confirm Order
            executeStep(sagaState, SagaStep.CONFIRM_ORDER,
                () -> orderService.confirmOrder(order.getId()));

            // Mark saga as completed
            sagaState.setStatus(SagaStatus.COMPLETED);
            sagaStateRepository.save(sagaState);

            return OrderResponse.success(order);

        } catch (Exception e) {
            // Compensate for completed steps
            compensate(sagaState);
            return OrderResponse.failure(e.getMessage());
        }
    }

    private <T> T executeStep(SagaState sagaState,
                              SagaStep step,
                              Supplier<T> action) {
        sagaState.setCurrentStep(step);
        sagaState.setStatus(SagaStatus.IN_PROGRESS);
        sagaStateRepository.save(sagaState);

        try {
            T result = action.get();
            sagaState.addCompletedStep(step);
            sagaStateRepository.save(sagaState);
            return result;
        } catch (Exception e) {
            sagaState.setStatus(SagaStatus.FAILED);
            sagaStateRepository.save(sagaState);
            throw e;
        }
    }

    private void compensate(SagaState sagaState) {
        sagaState.setStatus(SagaStatus.COMPENSATING);
        sagaStateRepository.save(sagaState);

        List<SagaStep> completedSteps = sagaState.getCompletedSteps();

        // Execute compensations in reverse order
        for (int i = completedSteps.size() - 1; i >= 0; i--) {
            SagaStep step = completedSteps.get(i);

            try {
                switch (step) {
                    case CREATE_ORDER:
                        orderService.cancelOrder(sagaState.getOrderId());
                        break;
                    case RESERVE_INVENTORY:
                        inventoryService.releaseStock(
                            sagaState.getProductId(),
                            sagaState.getQuantity()
                        );
                        break;
                    case PROCESS_PAYMENT:
                        paymentService.refundPayment(
                            sagaState.getCustomerId(),
                            sagaState.getAmount()
                        );
                        break;
                }
            } catch (Exception e) {
                // Log compensation failure
                // In production, you might want to retry or alert
                log.error("Compensation failed for step: " + step, e);
            }
        }

        sagaState.setStatus(SagaStatus.COMPENSATED);
        sagaStateRepository.save(sagaState);
    }

    private SagaState initializeSaga(OrderRequest request) {
        SagaState sagaState = new SagaState();
        sagaState.setStatus(SagaStatus.STARTED);
        sagaState.setOrderRequest(request);
        return sagaStateRepository.save(sagaState);
    }
}
```

#### 3. Compensating Transactions

```java
@Service
public class InventoryService {

    @Autowired
    private InventoryRepository inventoryRepository;

    @Transactional
    public void reserveStock(String productId, Integer quantity) {
        Inventory inventory = inventoryRepository
            .findById(productId)
            .orElseThrow(() -> new ProductNotFoundException());

        if (inventory.getAvailableStock() < quantity) {
            throw new InsufficientStockException();
        }

        inventory.setAvailableStock(
            inventory.getAvailableStock() - quantity
        );
        inventory.setReservedStock(
            inventory.getReservedStock() + quantity
        );

        inventoryRepository.save(inventory);
    }

    @Transactional
    public void releaseStock(String productId, Integer quantity) {
        Inventory inventory = inventoryRepository
            .findById(productId)
            .orElseThrow(() -> new ProductNotFoundException());

        inventory.setAvailableStock(
            inventory.getAvailableStock() + quantity
        );
        inventory.setReservedStock(
            inventory.getReservedStock() - quantity
        );

        inventoryRepository.save(inventory);
    }

    @Transactional
    public void confirmReservation(String productId, Integer quantity) {
        Inventory inventory = inventoryRepository
            .findById(productId)
            .orElseThrow(() -> new ProductNotFoundException());

        inventory.setReservedStock(
            inventory.getReservedStock() - quantity
        );

        inventoryRepository.save(inventory);
    }
}
```

### Saga with Choreography (Event-Driven)

For choreography, services communicate through events:

```java
@Service
public class OrderEventHandler {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Autowired
    private OrderRepository orderRepository;

    @TransactionalEventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Publish inventory reservation request
        eventPublisher.publishEvent(
            new ReserveInventoryEvent(
                event.getOrderId(),
                event.getProductId(),
                event.getQuantity()
            )
        );
    }

    @TransactionalEventListener
    public void handleInventoryReserved(InventoryReservedEvent event) {
        // Publish payment processing request
        eventPublisher.publishEvent(
            new ProcessPaymentEvent(
                event.getOrderId(),
                event.getCustomerId(),
                event.getAmount()
            )
        );
    }

    @TransactionalEventListener
    public void handlePaymentFailed(PaymentFailedEvent event) {
        // Compensate: Release inventory
        eventPublisher.publishEvent(
            new ReleaseInventoryEvent(
                event.getOrderId(),
                event.getProductId(),
                event.getQuantity()
            )
        );

        // Cancel order
        Order order = orderRepository.findById(event.getOrderId())
            .orElseThrow();
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
    }
}
```

### Pros and Cons of Saga

**Advantages:**
- No blocking - better performance
- Better scalability
- Works well with microservices
- More resilient to failures
- Suitable for long-running transactions

**Disadvantages:**
- Eventual consistency (not immediate)
- Complex to implement and test
- Compensating transactions can be tricky
- Need to handle partial failures
- Debugging can be challenging

## PostgreSQL Configuration for Distributed Transactions

For 2PC with PostgreSQL, ensure XA transactions are enabled:

```sql
-- Enable prepared transactions in postgresql.conf
max_prepared_transactions = 100

-- Create databases
CREATE DATABASE orderdb;
CREATE DATABASE inventorydb;

-- Grant necessary permissions
GRANT ALL PRIVILEGES ON DATABASE orderdb TO postgres;
GRANT ALL PRIVILEGES ON DATABASE inventorydb TO postgres;
```

## When to Use Which Pattern?

### Use 2PC When:
- Strong consistency is absolutely required
- Transaction spans a small number of services (2-3)
- Operations are short-lived
- Network is reliable and low-latency
- You can afford the performance overhead

### Use Saga When:
- Working with microservices at scale
- Can tolerate eventual consistency
- Transactions involve many services
- Long-running business processes
- Need better performance and availability
- Services are distributed across networks

## Best Practices

1. **Idempotency**: Make all operations idempotent to handle retries safely
2. **Logging**: Comprehensive logging for debugging distributed flows
3. **Monitoring**: Track saga states and compensations
4. **Timeouts**: Set appropriate timeouts for each step
5. **Circuit Breakers**: Protect services from cascading failures
6. **Testing**: Test both happy paths and compensation scenarios

## Conclusion

Distributed transactions are challenging but essential in microservices architectures. 2PC provides strong consistency at the cost of performance and scalability, while Saga patterns offer better scalability with eventual consistency. Choose based on your specific requirements, and remember that sometimes the best solution is to reconsider your service boundaries to minimize the need for distributed transactions altogether.

In the next posts, we'll dive deeper into event sourcing, CQRS patterns, and how they complement these transaction patterns in building robust distributed systems.

## References

- [Spring Boot JTA Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-jta.html)
- [PostgreSQL XA Transactions](https://www.postgresql.org/docs/current/sql-prepare-transaction.html)
- [Microservices Patterns by Chris Richardson](https://microservices.io/patterns/data/saga.html)
