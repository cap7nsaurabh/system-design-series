# Messaging & Event-Driven Architecture in Spring Boot

In modern distributed systems and microservices architectures, synchronous HTTP calls between services often lead to tight coupling, cascading failures, and degraded performance. Event-Driven Architecture (EDA) and Message Queues provide a robust alternative by decoupling producers and consumers, enabling asynchronous communication, and improving overall system resilience and scalability.

This module provides a highly detailed, interview-focused deep dive into Messaging & Event-Driven Architecture in Spring Boot. We will cover core concepts, Spring for Apache Kafka, Spring AMQP (RabbitMQ), distributed design patterns, testing strategies, and critical interview traps.

---

## 1. Event-Driven Architecture Basics

### 1.1 Synchronous vs Asynchronous Microservices

When designing microservices, service-to-service communication fundamentally falls into two categories:

| Feature | Synchronous (REST/gRPC) | Asynchronous (Messaging/EDA) |
| :--- | :--- | :--- |
| **Coupling** | High. Caller must know the callee's address and API. | Low. Producer and consumer are independent. |
| **Availability**| Both services must be up simultaneously (Temporal Coupling). | Consumer can be down; messages are stored in the broker. |
| **Latency** | Caller waits for the response, adding to total request time. | Caller fires the event and returns immediately. |
| **Failure Handling**| Cascading failures if downstream services are slow/down. | Resilient. Messages are queued and processed when services recover. |
| **Scalability** | Harder to scale individually under sudden traffic spikes. | Easy to scale consumers to drain queues faster (Load Leveling). |

> [!IMPORTANT]
> A common interview question is: "When would you choose synchronous over asynchronous communication?"
> **Answer**: Choose synchronous when an immediate response is strictly required for the user experience (e.g., login authentication, fetching real-time account balance). Choose asynchronous for background processing, notifications, or eventual consistency operations (e.g., sending a welcome email, processing an order).

### 1.2 Decoupling with Message Brokers

A **Message Broker** (e.g., Kafka, RabbitMQ) acts as an intermediary. Services no longer talk directly to one another. Instead, a service (Producer) sends a message to the broker, and other services (Consumers) read from the broker. This provides:
- **Temporal Decoupling**: Producers and consumers don't need to be alive at the same time.
- **Location Decoupling**: Producers don't need to know the IPs or hostnames of consumers.

### 1.3 Pub/Sub vs Point-to-Point

Messaging models generally fall into two paradigms:

1. **Point-to-Point (Queue-based)**:
   - A message is sent to a specific queue.
   - Even if multiple consumers listen to the queue, exactly **one** consumer processes each message.
   - Great for distributing workloads (e.g., job queues).

2. **Publish/Subscribe (Topic-based)**:
   - A message is published to a topic.
   - **Multiple** independent consumers (or consumer groups) can subscribe to the topic.
   - Every subscriber gets a copy of the message.
   - Great for event broadcasting (e.g., "OrderCreatedEvent").

---

## 2. Spring for Apache Kafka

Apache Kafka is a distributed event streaming platform capable of handling trillions of events a day. Unlike traditional message queues, Kafka stores streams of records continuously in an append-only log.

### 2.1 Kafka Concepts
- **Topic**: A category or feed name to which records are published.
- **Partition**: Topics are divided into partitions for scalability. A partition is an ordered, immutable sequence of records.
- **Offset**: A unique sequential ID assigned to each record within a partition.
- **Consumer Group**: Consumers sharing the same group ID. Each partition is consumed by exactly one consumer within a consumer group.

### 2.2 Setup and application.yml Configuration

**Dependencies (pom.xml):**
```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

**application.yml:**
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all # Wait for all replicas to acknowledge (highest durability)
      retries: 3
    consumer:
      group-id: "order-service-group"
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: "com.example.events"
      enable-auto-commit: false # Crucial for manual acknowledgment
    listener:
      ack-mode: manual_immediate
      concurrency: 3 # Number of threads/consumers in the listener container
```

> [!TIP]
> Always use `ErrorHandlingDeserializer` in Spring Kafka. If a standard deserializer fails (e.g., poison pill JSON), the consumer will infinitely loop trying to read the same offset. `ErrorHandlingDeserializer` catches the exception, allows the message to be skipped or sent to a DLT, and moves the offset forward.

### 2.3 KafkaTemplate for Producing Messages

Spring provides `KafkaTemplate` to send messages.

```java
package com.example.producer;

import com.example.events.OrderCreatedEvent;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Service;
import java.util.concurrent.CompletableFuture;

@Service
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    public OrderEventPublisher(KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publishOrderCreated(OrderCreatedEvent event) {
        String topic = "order-events";
        String routingKey = event.getOrderId(); // Key ensures same order goes to same partition

        // Spring Kafka 3.x returns CompletableFuture
        CompletableFuture<SendResult<String, OrderCreatedEvent>> future = 
            kafkaTemplate.send(topic, routingKey, event);

        future.whenComplete((result, ex) -> {
            if (ex == null) {
                System.out.println("Message sent successfully to partition " 
                    + result.getRecordMetadata().partition() 
                    + " with offset " + result.getRecordMetadata().offset());
            } else {
                System.err.println("Failed to send message: " + ex.getMessage());
                // Handle failure (e.g., save to fallback DB table)
            }
        });
    }
}
```

### 2.4 @KafkaListener for Consuming Messages

```java
package com.example.consumer;

import com.example.events.OrderCreatedEvent;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component
public class OrderEventConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventConsumer.class);

    @KafkaListener(
        topics = "order-events", 
        groupId = "inventory-service-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderCreated(@Payload OrderCreatedEvent event, Acknowledgment acknowledgment) {
        try {
            log.info("Received order created event for order: {}", event.getOrderId());
            
            // Business logic to update inventory
            processInventoryUpdate(event);
            
            // Explicitly acknowledge the message
            acknowledgment.acknowledge();
        } catch (Exception e) {
            log.error("Error processing message for order: {}", event.getOrderId(), e);
            // Do not acknowledge here; let it trigger error handling/retries
            throw e;
        }
    }

    private void processInventoryUpdate(OrderCreatedEvent event) {
        // Implementation
    }
}
```

### 2.5 Acknowledgment Modes (MANUAL vs AUTO)

Spring Kafka provides several acknowledgment modes via `ContainerProperties.AckMode`:

1. **RECORD**: Commits the offset after the listener returns for *each* record.
2. **BATCH**: (Default) Commits the offset after the listener returns for the *entire batch* of records polled.
3. **MANUAL**: The listener must call `Acknowledgment.acknowledge()`. The commit is actually performed when the batch finishes.
4. **MANUAL_IMMEDIATE**: Commits immediately when `acknowledgment.acknowledge()` is called (synchronous block until committed).

> [!WARNING]
> Relying on auto-commit (`enable-auto-commit=true`) can lead to message loss. If a consumer pulls messages, auto-commits the offset in the background, but then crashes before processing them, those messages are lost forever. **Always use MANUAL or MANUAL_IMMEDIATE with `enable-auto-commit=false` in production.**

### 2.6 Error Handling and RetryTopic (Non-blocking Retries)

In Kafka, if you block a consumer thread to sleep and retry, no other messages on that partition can be processed. This is a **blocking retry**. 
Spring Kafka provides **Non-blocking Retries** using `@RetryableTopic`. It creates separate retry topics (e.g., `order-events-retry-0`, `order-events-retry-1`) with backoff delays, leaving the main topic unblocked.

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.annotation.RetryableTopic;
import org.springframework.kafka.retrytopic.TopicSuffixingStrategy;
import org.springframework.retry.annotation.Backoff;

@Component
public class RobustConsumer {

    @RetryableTopic(
        attempts = "4", // Main + 3 retries
        backoff = @Backoff(delay = 1000, multiplier = 2.0),
        topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE,
        exclude = {IllegalArgumentException.class} // Don't retry validation errors
    )
    @KafkaListener(topics = "order-events", groupId = "shipping-group")
    public void listen(OrderCreatedEvent event) {
        // Process... throws exception on transient failure
    }
    
    // Fallback method (DLT handler)
    @org.springframework.kafka.annotation.DltHandler
    public void handleDlt(OrderCreatedEvent event, @org.springframework.kafka.support.KafkaHeaders.ReceivedTopic String topic) {
        System.out.println("Message sent to DLT from topic " + topic + ". OrderID: " + event.getOrderId());
        // Save to manual intervention DB table
    }
}
```

---

## 3. Spring AMQP (RabbitMQ)

RabbitMQ is a traditional message broker implementing the AMQP protocol. It is highly flexible and excels at complex routing.

### 3.1 RabbitMQ Core Concepts
- **Exchange**: Receives messages from producers and pushes them to queues based on routing rules.
- **Queue**: A buffer that stores messages.
- **Binding**: A link between a queue and an exchange.
- **Routing Key**: A key that the exchange uses to decide which queue(s) to route the message to.

### 3.2 Exchange Types

| Exchange Type | Routing Logic | Use Case |
| :--- | :--- | :--- |
| **Direct** | Exact match of routing key. | Point-to-point communication (e.g., `pdf.create`). |
| **Topic** | Wildcard match (`*` matches one word, `#` matches zero or more). | Subscribing to specific event categories (e.g., `order.*.created`). |
| **Fanout** | Broadcasts to ALL bound queues, ignoring routing key. | Pub/Sub broadcasting (e.g., cache invalidation). |
| **Headers** | Routes based on AMQP message headers instead of routing keys. | Complex routing logic based on metadata. |

### 3.3 Setup and application.yml

**Dependencies:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

**application.yml:**
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000ms
```

### 3.4 RabbitMQ Configuration (Exchanges, Queues, Bindings)

```java
package com.example.config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {

    public static final String EXCHANGE_NAME = "order.exchange";
    public static final String QUEUE_NAME = "inventory.queue";
    public static final String ROUTING_KEY = "order.created";

    // 1. Declare Exchange
    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange(EXCHANGE_NAME);
    }

    // 2. Declare Queue
    @Bean
    public Queue inventoryQueue() {
        // queueName, durable, exclusive, autoDelete
        return new Queue(QUEUE_NAME, true); 
    }

    // 3. Bind Queue to Exchange
    @Bean
    public Binding binding(Queue inventoryQueue, DirectExchange orderExchange) {
        return BindingBuilder.bind(inventoryQueue).to(orderExchange).with(ROUTING_KEY);
    }
}
```

### 3.5 RabbitTemplate and @RabbitListener

**Producer:**
```java
@Service
public class RabbitProducer {
    private final RabbitTemplate rabbitTemplate;

    public RabbitProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void sendOrderMessage(OrderCreatedEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitConfig.EXCHANGE_NAME, 
            RabbitConfig.ROUTING_KEY, 
            event
        );
    }
}
```

**Consumer:**
```java
import com.rabbitmq.client.Channel;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;
import java.io.IOException;

@Component
public class RabbitConsumer {

    @RabbitListener(queues = RabbitConfig.QUEUE_NAME)
    public void receiveMessage(OrderCreatedEvent event, 
                               Channel channel, 
                               @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {
        try {
            System.out.println("Processing event: " + event.getOrderId());
            // Process logic...
            
            // Manual Ack
            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            // Reject and re-queue, or send to DLX
            // basicReject(deliveryTag, requeue)
            channel.basicReject(deliveryTag, false); 
        }
    }
}
```

### 3.6 Dead Letter Exchanges (DLX) in RabbitMQ

To handle failed messages in RabbitMQ, we configure the primary queue to forward rejected messages to a Dead Letter Exchange.

```java
@Bean
public DirectExchange deadLetterExchange() {
    return new DirectExchange("dlx.exchange");
}

@Bean
public Queue deadLetterQueue() {
    return new Queue("dlx.queue");
}

@Bean
public Binding dlxBinding() {
    return BindingBuilder.bind(deadLetterQueue()).to(deadLetterExchange()).with("dlx.routing.key");
}

@Bean
public Queue inventoryQueue() {
    return QueueBuilder.durable("inventory.queue")
            .withArgument("x-dead-letter-exchange", "dlx.exchange")
            .withArgument("x-dead-letter-routing-key", "dlx.routing.key")
            .build();
}
```
Now, if a consumer calls `channel.basicReject(deliveryTag, false)`, the message is automatically routed to `dlx.queue`.

---

## 4. Distributed Patterns in Messaging

### 4.1 The Dual-Write Problem
A classic microservices problem: You need to save an entity to the database and publish an event to Kafka.
```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order); // Write 1
    kafkaTemplate.send("orders", order); // Write 2
}
```
**The Problem**: 
- If the DB commit succeeds but Kafka is down, the system is inconsistent.
- If Kafka send succeeds, but the DB commit rolls back (e.g., constraint violation), the system is inconsistent (consumers act on phantom data).
You cannot use standard `@Transactional` to span a database and a Kafka cluster (2PC/XA is too slow and not supported cleanly by Kafka).

### 4.2 The Transactional Outbox Pattern (Crucial for Interviews)
To solve the dual-write problem, we use the **Outbox Pattern**.
Instead of sending to Kafka directly, we write the event to an "outbox" table in the *same database transaction* as the business entity. A separate background process reads the outbox table and publishes to Kafka.

**Step 1: Entity and Outbox Model**
```java
@Entity
public class Order {
    @Id private Long id;
    private String status;
}

@Entity
public class OutboxEvent {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    private String payload; // JSON
    private boolean processed;
}
```

**Step 2: Save both in one transaction**
```java
@Service
public class OrderService {
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = new Order(request.getId(), "CREATED");
        orderRepository.save(order); // Business data
        
        OutboxEvent event = new OutboxEvent(
            "ORDER", order.getId().toString(), "OrderCreated", 
            objectMapper.writeValueAsString(order)
        );
        outboxRepository.save(event); // Event data
    }
}
```

**Step 3: Background Publisher (Polling or CDC)**
```java
@Component
public class OutboxRelay {
    
    @Scheduled(fixedDelay = 1000)
    public void relayEvents() {
        List<OutboxEvent> pending = outboxRepository.findByProcessedFalse();
        for (OutboxEvent event : pending) {
            kafkaTemplate.send("orders", event.getAggregateId(), event.getPayload());
            event.setProcessed(true);
            outboxRepository.save(event);
        }
    }
}
```
> [!NOTE]
> Polling works for small scale. For large scale, use **Change Data Capture (CDC)** tools like **Debezium**, which read the database transaction log (e.g., MySQL binlog, Postgres WAL) and stream events directly to Kafka with zero polling overhead.

### 4.3 Saga Pattern (Choreography vs Orchestration)
Since distributed transactions (XA) lock resources and scale poorly, the Saga pattern is used for distributed workflows (e.g., Order -> Payment -> Inventory).

- **Choreography**: Each service listens to events and publishes new events. No central coordinator. (Good for simple workflows, hard to trace).
- **Orchestration**: A central service (Orchestrator) commands other services and handles state. (Better for complex workflows, avoids cyclic dependencies).

### 4.4 Idempotency in Consumers
Because networks are unreliable, message brokers guarantee **at-least-once delivery**. This means your consumer *will* receive duplicate messages. Your consumer MUST be **idempotent** (processing the same message 1 time or 100 times yields the exact same final state).

**Implementation via Database Idempotency Key:**
```java
@Service
public class PaymentConsumer {
    
    @Transactional
    @KafkaListener(topics = "payments")
    public void processPayment(PaymentEvent event) {
        // 1. Check if event was already processed
        if (processedEventRepository.existsById(event.getEventId())) {
            log.info("Duplicate message ignored: {}", event.getEventId());
            return; 
        }
        
        // 2. Do business logic
        accountService.deductBalance(event.getUserId(), event.getAmount());
        
        // 3. Save event ID to idempotency table
        processedEventRepository.save(new ProcessedEvent(event.getEventId()));
    }
}
```

---

## 5. ActiveMQ / JMS (Brief)

While Kafka and RabbitMQ dominate microservices, legacy enterprise applications often use JMS (Java Message Service) via Apache ActiveMQ or Artemis.

- **JmsTemplate**: Spring's synchronous and helper class for JMS.
- **@JmsListener**: Marks a method to asynchronously receive messages.

```java
@Component
public class JmsConsumer {
    @JmsListener(destination = "invoice.queue")
    public void receive(String message) {
        System.out.println("Received: " + message);
    }
}
```
*Note: JMS strictly distinguishes between Queues (Point-to-Point) and Topics (Pub/Sub) at the API level.*

---

## 6. Testing Messaging

Messaging makes testing asynchronous. You must test if messages are actually sent/received without requiring an external broker to be manually started.

### 6.1 Testcontainers
Testcontainers spins up actual Docker containers of Kafka/RabbitMQ during integration tests.

```java
@SpringBootTest
@Testcontainers
public class KafkaIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.3.0"));

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Test
    void testMessageFlow() {
        kafkaTemplate.send("test-topic", "test-data");
        // Use Awaitility to wait for async processing
        Awaitility.await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            // assert database state changed by consumer
        });
    }
}
```

### 6.2 Embedded Kafka
Spring provides `@EmbeddedKafka`, an in-memory broker. It's faster than Testcontainers but less identical to production.
```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, brokerProperties = { "listeners=PLAINTEXT://localhost:9092", "port=9092" })
public class EmbeddedKafkaTest { ... }
```

---

## 7. ⚠️ Interview Traps & Gotchas

### Trap 1: Blocking the Consumer Thread
**Gotcha**: Calling `Thread.sleep()` or making heavy synchronous HTTP calls inside a `@KafkaListener` blocks the consumer thread. If you don't poll the broker within `max.poll.interval.ms`, Kafka assumes the consumer is dead, triggers a partition rebalance, and the message is re-delivered, creating an infinite loop of failures.
**Solution**: Use non-blocking retries (RetryTopic), asynchronous processing, or increase `max.poll.interval.ms` carefully.

### Trap 2: Committing Offsets Before Processing Finishes
**Gotcha**: Using `enable-auto-commit=true`. The consumer fetches a batch, Kafka auto-commits. The application crashes mid-batch. Result: Data Loss.
**Solution**: Always use `AckMode.MANUAL_IMMEDIATE` and only call `.acknowledge()` at the absolute end of the `try` block.

### Trap 3: Poison Pill Messages
**Gotcha**: A producer sends invalid JSON. The consumer throws a DeserializationException before reaching your `@KafkaListener`. The offset is not committed. The consumer fetches the exact same bad message again infinitely.
**Solution**: Use `ErrorHandlingDeserializer` in Spring Kafka, which catches the error and wraps it, allowing the consumer to skip or route to a DLT.

### Trap 4: @Transactional and KafkaTemplate
**Gotcha**: A candidate writes:
```java
@Transactional
public void doWork() {
    kafkaTemplate.send("topic", msg);
    if(true) throw new RuntimeException("DB fails");
}
```
**Trap Question**: Does the Kafka message get rolled back?
**Answer**: By default, NO! Kafka sends the message immediately over the network. If the DB rolls back, the Kafka message is already gone.
To fix this, you must enable **Kafka Transactions** and configure the `KafkaTransactionManager` to synchronize with the JPA Transaction manager, OR use the Transactional Outbox pattern (which is vastly preferred).

### Trap 5: Duplicate Message Processing
**Gotcha**: Assuming a message is processed only once. Network timeouts during acknowledgment can cause brokers to resend messages.
**Solution**: Always design consumers to be idempotent. Use unique event IDs and check against a database/cache before processing.

---

## 8. 🔍 Deep Dive: Internal Mechanics

### How `@KafkaListener` Works Under the Hood
When Spring boots up, it scans for `@KafkaListener` annotations. For each annotation, the `KafkaListenerAnnotationBeanPostProcessor` creates a `MessageListenerContainer` (usually `ConcurrentMessageListenerContainer`).
1. The container spins up `N` threads (defined by the `concurrency` property).
2. Each thread creates a `KafkaConsumer` and calls `.poll()`.
3. When messages arrive, they are dispatched to your annotated method via reflection.
4. If `AckMode` is manual, Spring provides the `Acknowledgment` object, which holds a reference to the `Consumer` to allow calling `commitSync()` or `commitAsync()`.

### RabbitMQ Prefetch vs Kafka Polling
- **RabbitMQ Prefetch**: Dictates how many unacknowledged messages the broker will push to a consumer simultaneously. If prefetch is 10, the broker pushes 10 messages and waits for acks before pushing more. 
- **Kafka Polling**: The consumer actively pulls messages. `max.poll.records` dictates the maximum size of the batch returned in a single `.poll()` call.

---

## 9. 📝 Summary & Key Takeaways

1. **Decoupling**: Message brokers decouple microservices temporally and geographically.
2. **Kafka vs RabbitMQ**: Kafka is an append-only log suited for high-throughput event streaming and replayability. RabbitMQ is an AMQP broker suited for complex routing and traditional point-to-point queues.
3. **Dual-Write Problem**: Solved cleanly via the Transactional Outbox Pattern.
4. **Idempotency**: Message brokers guarantee at-least-once delivery; your application must handle duplicates gracefully.
5. **Spring Abstractions**: Spring unifies messaging with `XXXTemplate` for producers and `@XXXListener` for consumers, drastically reducing boilerplate.

---

## 10. ❓ Practice Questions

1. Explain the difference between `Topic` in Kafka and `Exchange` in RabbitMQ.
2. How would you implement the Transactional Outbox pattern if you couldn't use a cron job to poll the database?
3. Walk through the exact mechanism of a Poison Pill infinite loop in Kafka and how Spring's `ErrorHandlingDeserializer` resolves it.
4. What happens if a Kafka consumer takes longer than `max.poll.interval.ms` to process a message? How does this impact the consumer group?
5. Design an idempotent consumer for an `OrderPaidEvent`. What database tables and constraints would you need?
