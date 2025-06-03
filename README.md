# E-commerce Microservices Project: Line-by-Line and Conceptual Explanation

Welcome! This document explains every single file in your Java e-commerce microservices project. It covers line-by-line explanations of code, concepts like microservices, Docker, Kafka, controllers, and more. Even if you are a beginner, you will find everything you need to understand how and why each piece works.

---

## Table of Contents

1. [Introduction to Key Concepts](#introduction-to-key-concepts)
   - [Microservices](#microservices)
   - [Controllers](#controllers)
   - [Kafka](#kafka)
   - [Docker](#docker)
   - [Spring Boot](#spring-boot)
   - [JPA & Databases](#jpa--databases)
2. [Repository Structure](#repository-structure)
3. [File-by-File, Line-by-Line Explanation](#file-by-file-line-by-line-explanation)
   - [1. ms-orders](#1-ms-orders)
   - [2. ms-payment](#2-ms-payment)
   - [3. ms-stock](#3-ms-stock)
   - [4. commons](#4-commons)
   - [5. docker-compose.yml](#5-docker-composeyml)
   - [6. Root pom.xml](#6-root-pomxml)
4. [Glossary of Terms](#glossary-of-terms)

---

## Introduction to Key Concepts

### Microservices

**Microservices** is an architectural style that structures an application as a collection of small, independently deployable services. Each service is:
- Loosely coupled: Works independently.
- Highly maintainable and testable.
- Organized around business capabilities (e.g., orders, payments, stock).
- Owned by small teams.

In this project:
- `ms-orders` handles orders.
- `ms-payments` handles payment logic.
- `ms-stock` manages product inventory.

### Controllers

In web frameworks (like Spring Boot), a **controller** is a class that handles HTTP requests. It takes input (like a web form submission), processes it (sometimes by talking to services or a database), and returns a response (like a web page or some data).

### Kafka

**Kafka** is an open-source distributed event streaming platform. Think of it as a messaging system:
- Services can send events (messages) to Kafka "topics".
- Other services can listen for those events.
- It's great for building systems where different parts need to communicate in real-time but independently.

### Docker

**Docker** is a tool to package software and all its dependencies into a "container" that can run anywhere. It's like a box that holds your application and everything it needs to run, ensuring it works the same way on any machine.

### Spring Boot

**Spring Boot** simplifies the process of creating stand-alone, production-grade Spring-based applications. It auto-configures a lot of things so you can focus on your business logic.

### JPA & Databases

**JPA (Java Persistence API)** is a standard for mapping Java objects to database tables. In Spring, this is handled by Spring Data JPA, making database operations much simpler.

---

## Repository Structure

```
ecommerce/
├── commons/       # Shared code (domain objects, enums, etc.)
├── ms-orders/     # Orders microservice
├── ms-payment/    # Payments microservice
├── ms-stock/      # Stock/inventory microservice
├── docker-compose.yml
└── pom.xml        # Root Maven config
```

---

## File-by-File, Line-by-Line Explanation

---

### 1. ms-orders

#### i) DockerFile

```dockerfile
FROM adoptopenjdk/openjdk11:jdk-11.0.2.9-slim
WORKDIR /opt
EXPOSE 8080
COPY target/*.jar /opt/app.jar
ENTRYPOINT exec java $JAVA_OPTS -jar app.jar
```

**Explanation:**
- `FROM ...`: This line sets the base image for the container. It uses an OpenJDK 11 slim image, so your Java app runs in a lightweight environment.
- `WORKDIR /opt`: Sets `/opt` as the working directory inside the container (where commands will be run).
- `EXPOSE 8080`: Informs Docker that the app will use port 8080. (Note: the actual Spring Boot app uses 9091, as in `application.yml`, so you might need to expose 9091 instead.)
- `COPY target/*.jar /opt/app.jar`: Copies the built Java app (from `target/` directory) into the container as `app.jar`.
- `ENTRYPOINT ...`: Sets the command to run when the container starts: start the Java app with any extra options (`$JAVA_OPTS`).

---

#### ii) src/test/java/com/zatribune/spring/ecommerce/orders/OnlineApplicationTests.java

```java
package com.zatribune.spring.ecommerce.orders;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class OrderApplicationTests {

    @Test
    void contextLoads() {
    }

}
```

**Explanation:**
- `package ...`: Declares the Java package.
- `import ...`: Imports required classes:
  - `Test` for marking test methods.
  - `SpringBootTest` for loading the Spring app context for testing.
- `@SpringBootTest`: Tells Spring to start the full application context for the test.
- `class OrderApplicationTests { ... }`: Test class.
- `@Test void contextLoads() {}`: A test case that checks if the application context loads successfully. If this fails, your app configuration is broken.

---

#### iii) src/main/java/com/zatribune/spring/ecommerce/orders/config/KafkaConfig.java

**Summary:** Configures Kafka topics and streams for the order service.

```java
package com.zatribune.spring.ecommerce.orders.config;

import domain.Order;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.admin.NewTopic;
import org.apache.kafka.common.serialization.Serde;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.streams.state.KeyValueBytesStoreSupplier;
import org.apache.kafka.streams.state.Stores;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskExecutor;
import org.springframework.kafka.config.TopicBuilder;
import org.springframework.kafka.support.serializer.JsonSerde;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import com.zatribune.spring.ecommerce.orders.service.OrderService;

import java.time.Duration;

import static domain.Topics.*;

@Slf4j
@Configuration
public class KafkaConfig {

    private final OrderService orderService;

    @Autowired
    public KafkaConfig(OrderService orderService) {
        this.orderService = orderService;
    }

    @Bean
    public NewTopic orders() {
        return TopicBuilder.name(ORDERS)
                .partitions(3)
                .compact()
                .build();
    }

    @Bean
    public NewTopic paymentTopic() {
        return TopicBuilder.name(PAYMENTS)
                .partitions(3)
                .compact()
                .build();
    }

    @Bean
    public NewTopic stockTopic() {
        return TopicBuilder.name(STOCK)
                .partitions(3)
                .compact()
                .build();
    }

    //this is the one that will get results from other topics to check
    @Bean
    public KStream<Long, Order> stream(StreamsBuilder builder) {
        //provides serialization and deserialization in JSON format

        Serde<Long> keySerde =  Serdes.Long(); // Key serializer/deserializer for Long
        JsonSerde<Order> valueSerde = new JsonSerde<>(Order.class); // Value serde for Order objects

        // Create streams from the PAYMENTS and STOCK topics
        KStream<Long, Order> paymentStream = builder
                .stream(PAYMENTS, Consumed.with(keySerde, valueSerde));

        KStream<Long, Order> stockStream = builder
                .stream(STOCK,Consumed.with(keySerde, valueSerde));

        // Join payment and stock streams on order ID
        paymentStream.join(
                stockStream,
                orderService::confirm, // Combines the two Orders (payment & stock info)
                JoinWindows.of(Duration.ofSeconds(10)), // Match events within 10 seconds window
                StreamJoined.with(keySerde, valueSerde, valueSerde)
        )
        .peek((k,v)->log.info("Kafka stream match: key[{}],value[{}]",k,v)) // Log joined records
        .to(ORDERS); // Send result to ORDERS topic

        return paymentStream;
    }

    /**
     * To build a persistent key-value store
     * This KTable will be used to store all the Orders
     ***/
    @Bean
    public KTable<Long, Order> table(StreamsBuilder builder) {

        KeyValueBytesStoreSupplier store = Stores.persistentKeyValueStore(ORDERS);

        Serde<Long> keySerde =  Serdes.Long();
        JsonSerde<Order> valueSerde = new JsonSerde<>(Order.class);

        KStream<Long, Order> stream = builder
                .stream(ORDERS, Consumed.with(keySerde, valueSerde))
                .peek((k,v)->log.info("Kafka persistence table: key[{}],value[{}]",k,v));

        return stream.toTable(Materialized.<Long, Order>as(store)
                .withKeySerde(keySerde)
                .withValueSerde(valueSerde));
    }

    @Bean
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(5);
        executor.setThreadNamePrefix("kafkaSender-");
        executor.initialize();
        return executor;
    }
}
```

**Line-by-line:**
- `@Slf4j`: Lombok annotation to auto-generate a logger named `log`.
- `@Configuration`: This class provides Spring beans (configurations).
- `private final OrderService orderService;`: Dependency for business logic when combining stream results.
- `@Autowired`: Tells Spring to inject `OrderService`.
- `@Bean public NewTopic ...`: Creates Kafka topics for orders, payments, and stock, each with 3 partitions and log compaction enabled.
- `@Bean public KStream ...`: Sets up a Kafka Streams pipeline:
  - Deserializes keys (Long) and values (Order) in JSON.
  - Reads two topics: `PAYMENTS` and `STOCK`.
  - Joins them by order ID within 10 seconds.
  - Combines values using the `OrderService.confirm` method.
  - Logs and publishes results to the `ORDERS` topic.
- `@Bean public KTable ...`: Sets up a persistent table (backed by Kafka) to store all orders, keyed by order ID.
- `@Bean public TaskExecutor ...`: Configures an async task executor for background tasks (e.g., sending Kafka messages).

---

#### iv) src/main/java/com/zatribune/spring/ecommerce/orders/controller/OrderController.java

**Summary:** Handles HTTP requests for orders.

```java
package com.zatribune.spring.ecommerce.orders.controller;

import domain.Order;
import domain.OrderStatus;
import domain.Topics;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.streams.StoreQueryParameters;
import org.apache.kafka.streams.state.KeyValueIterator;
import org.apache.kafka.streams.state.QueryableStoreTypes;
import org.apache.kafka.streams.state.ReadOnlyKeyValueStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.config.StreamsBuilderFactoryBean;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.web.bind.annotation.*;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.atomic.AtomicLong;

@Slf4j
@RequestMapping("/orders")
@RestController
public class OrderController {

    private final AtomicLong id = new AtomicLong();
    private final KafkaTemplate<Long, Order> kafkaTemplate;
    private final StreamsBuilderFactoryBean kafkaStreamsFactory;

    @Autowired
    public OrderController(KafkaTemplate<Long, Order> kafkaTemplate,
                           StreamsBuilderFactoryBean kafkaStreamsFactory) {
        this.kafkaTemplate = kafkaTemplate;
        this.kafkaStreamsFactory = kafkaStreamsFactory;
    }

    @PostMapping
    public Order create(@RequestBody Order order) throws ExecutionException, InterruptedException {
        order.setId(id.incrementAndGet()); // Assigns a unique ID to the order
        order.setStatus(OrderStatus.NEW); // Sets status to NEW
        log.info("Sent: {}", order); // Logs order
        return kafkaTemplate.send(Topics.ORDERS, order.getId(), order).get().getProducerRecord().value();
        // Sends order to Kafka ORDERS topic, waits for result, returns sent order
    }

    @GetMapping
    public List<Order> all() {
        List<Order> orders = new ArrayList<>();
        ReadOnlyKeyValueStore<Long, Order> store = kafkaStreamsFactory
                .getKafkaStreams()
                .store(StoreQueryParameters.fromNameAndType(
                        Topics.ORDERS,
                        QueryableStoreTypes.keyValueStore()));
        KeyValueIterator<Long, Order> it = store.all();
        it.forEachRemaining(kv -> orders.add(kv.value));
        return orders;
    }
}
```

**Line-by-line:**
- `@Slf4j`: Enables logging with `log.info(...)`.
- `@RestController`: Marks the class as a REST API controller.
- `@RequestMapping("/orders")`: All endpoints start with `/orders`.
- `AtomicLong id = new AtomicLong();`: Thread-safe incrementing for unique order IDs.
- `KafkaTemplate<Long, Order>`: Used to send messages to Kafka.
- `StreamsBuilderFactoryBean`: Accesses Kafka Streams state stores.
- `@Autowired`: Injects dependencies.
- `@PostMapping`: Handles HTTP POST request to `/orders`.
  - Receives an `Order` from the HTTP request body.
  - Assigns ID and status.
  - Sends order to Kafka.
  - Waits for the send operation to finish (blocking).
  - Returns the sent order.
- `@GetMapping`: Handles HTTP GET request to `/orders`.
  - Fetches all current orders from the Kafka state store.
  - Returns a list of orders.

---

#### v) src/main/java/com/zatribune/spring/ecommerce/orders/service/OrderService.java

```java
package com.zatribune.spring.ecommerce.orders.service;

import domain.Order;

public interface OrderService {

    Order confirm(Order orderPayment, Order orderStock);
}
```

**Explanation:**
- Defines an interface (contract) for an order service.
- `Order confirm(Order orderPayment, Order orderStock);`
  - Takes two orders (one from payment, one from stock), returns a combined order.
  - Used to merge/join results from Kafka streams.

---

#### vi) src/main/java/com/zatribune/spring/ecommerce/orders/service/OrderServiceImpl.java

```java
package com.zatribune.spring.ecommerce.orders.service;

import domain.Order;
import domain.OrderSource;
import domain.OrderStatus;
import org.springframework.stereotype.Service;

import static domain.OrderStatus.*;
import static domain.OrderSource.*;

@Service
public class OrderServiceImpl implements OrderService{

    @Override
    public Order confirm(Order orderPayment, Order orderStock) {

        Order o = Order.builder()
                .id(orderPayment.getId())
                .customerId(orderPayment.getCustomerId())
                .productId(orderPayment.getProductId())
                .productCount(orderPayment.getProductCount())
                .price(orderPayment.getPrice())
                .build();

        if (orderPayment.getStatus().equals(ACCEPT) &&
                orderStock.getStatus().equals(ACCEPT)) {
            o.setStatus(CONFIRMED);
        } else if (orderPayment.getStatus().equals(REJECT) &&
                orderStock.getStatus().equals(REJECT)) {
            o.setStatus(REJECTED);
        } else if (orderPayment.getStatus().equals(REJECT) ||
                orderStock.getStatus().equals(REJECT)) {
            OrderSource source = orderPayment.getStatus().equals(REJECT)
                    ? PAYMENT : STOCK;
            o.setStatus(OrderStatus.ROLLBACK);
            o.setSource(source);
        }
        return o;
    }

}
```

**Line-by-line:**
- `@Service`: Marks this class as a Spring service (for dependency injection).
- `Order confirm(...)`: Combines payment and stock orders:
  - Builds new order object from payment order fields.
  - If both payment and stock accepted: order is confirmed.
  - If both rejected: order is rejected.
  - If either rejected: set status to rollback, and source to whichever failed.
  - Returns the combined order.

---

#### vii) src/main/java/com/zatribune/spring/ecommerce/orders/OrderApplication.java

```java
package com.zatribune.spring.ecommerce.orders;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.kafka.annotation.EnableKafkaStreams;
import org.springframework.scheduling.annotation.EnableAsync;

@EnableAsync
@EnableKafkaStreams
@SpringBootApplication
public class OrderApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }

}
```

**Line-by-line:**
- `@EnableAsync`: Enables async processing (background tasks).
- `@EnableKafkaStreams`: Enables Kafka Streams support.
- `@SpringBootApplication`: Main configuration for Spring Boot.
- `public static void main`: Entry point. Runs the Spring Boot app.

---

#### viii) src/main/resources/application.yml

**Key configuration for the orders service.**

```yaml
spring:
  application.name: ms-orders
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: create-drop
  datasource:
    url: jdbc:mysql://127.0.0.1:33066/commerce
    username: root
    password: 1234
  h2:
    console:
      enabled: true

spring.kafka:
  bootstrap-servers: 127.0.0.1:39092, 127.0.0.1:29092
  producer:
    key-serializer: org.apache.kafka.common.serialization.LongSerializer
    value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
  streams:
    properties:
      default.key.serde: org.apache.kafka.common.serialization.Serdes$LongSerde
      default.value.serde: org.springframework.kafka.support.serializer.JsonSerde
      spring.json.trusted.packages: "*"
    state-dir: /tmp/kafka-streams/

spring.output.ansi.enabled: ALWAYS

logging.pattern.console: "%clr(%d{HH:mm:ss.SSS}){blue} %clr(---){faint} %clr([%15.15t]){yellow} %clr(:){red} %clr(%m){faint}%n"

server:
  port: 9091
```

**Explanation:**
- Configures the service name, JPA settings, database connection, Kafka brokers, serialization, Kafka Streams state directory, logging, and sets the HTTP port to 9091.

---

#### ix) pom.xml

**This is the Maven configuration for the orders microservice.**

- Declares project coordinates and parent (for shared settings).
- Declares dependencies:
  - `commons` (shared domain code)
  - Spring Boot starters (data JPA, web, test)
  - Kafka and Spring Kafka
  - Jackson (JSON serialization)
  - MySQL driver
- Configures Spring Boot Maven plugin (builds the app).
- Excludes Lombok from the final jar.

---

### 2. ms-payment

#### i) DockerFile

(Identical to ms-orders, see above.)

---

#### ii) pom.xml

(Identical in structure to ms-orders, except artifactId is `ms-payment`.)

---

#### iii) src/test/java/com/zatribune/spring/ecommerce/payments/PaymentApplicationTests.java

(Identical to ms-orders test.)

---

#### iv) src/main/java/com/zatribune/spring/ecommerce/payments/db/entities/Customer.java

```java
package com.zatribune.spring.ecommerce.payments.db.entities;

import lombok.*;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Entity
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private int amountAvailable;
    private int amountReserved;

    @Override
    public String toString() {
        return "Customer{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", amountAvailable=" + amountAvailable +
                ", amountReserved=" + amountReserved +
                '}';
    }
}
```

**Line-by-line:**
- Lombok annotations: generates getters, setters, no-args and all-args constructors, builder methods for the class.
- `@Entity`: Marks class as a JPA entity (maps to a table).
- Fields: `id`, `name`, `amountAvailable` (money in account), `amountReserved` (money reserved for pending payments).
- `@Id`, `@GeneratedValue`: Marks `id` as primary key, auto-generated.
- `toString`: String representation for debugging/logging.

---

#### v) src/main/java/com/zatribune/spring/ecommerce/payments/db/repository/CustomerRepository.java

```java
package com.zatribune.spring.ecommerce.payments.db.repository;

import com.zatribune.spring.ecommerce.payments.db.entities.Customer;
import org.springframework.data.jpa.repository.JpaRepository;

public interface CustomerRepository extends JpaRepository<Customer, Long> {
}
```

- Extends `JpaRepository` for `Customer` entity, using `Long` as ID.
- Inherits CRUD methods (findAll, save, etc.) from Spring Data.

---

#### vi) src/main/java/com/zatribune/spring/ecommerce/payments/db/DevBootstrap.java

**Pre-populates the database with customers at app startup.**

```java
package com.zatribune.spring.ecommerce.payments.db;

import com.zatribune.spring.ecommerce.payments.db.entities.Customer;
import com.zatribune.spring.ecommerce.payments.db.repository.CustomerRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import java.util.List;

@Slf4j
@Component
public class DevBootstrap implements CommandLineRunner {

    private final CustomerRepository repository;

    @Autowired
    public DevBootstrap(CustomerRepository repository) {
        this.repository = repository;
    }

    @Override
    public void run(String... args) throws Exception {
        Customer p1=Customer.builder().name("John Doe")
                .amountAvailable(4000)
                .amountReserved(0)
                .build();

        Customer p2=Customer.builder().name("Muhammad Ali")
                .amountAvailable(8000)
                .amountReserved(0)
                .build();

        Customer p3=Customer.builder().name("Steve Jobs")
                .amountAvailable(1000)
                .amountReserved(0)
                .build();

        Customer p4=Customer.builder().name("Bill Gits")
                .amountAvailable(2000)
                .amountReserved(0)
                .build();

        repository.saveAll(List.of(p1,p2,p3,p4));
    }
}
```

- `@Component`: Spring will launch this at startup.
- Implements `CommandLineRunner`: `run` called on startup.
- Creates four customers and saves them to the MySQL database.

---

#### vii) src/main/java/com/zatribune/spring/ecommerce/payments/listener/OrderListener.java

**Listens to Kafka ORDERS topic for new orders.**

```java
package com.zatribune.spring.ecommerce.payments.listener;

import domain.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;
import com.zatribune.spring.ecommerce.payments.service.OrderService;

@Slf4j
@Component
public class OrderListener {

    private final OrderService orderService;

    @Autowired
    public OrderListener(OrderService orderService) {
        this.orderService = orderService;
    }

    @KafkaListener(id = KafkaIds.ORDERS, topics = Topics.ORDERS, groupId = KafkaGroupIds.PAYMENTS)
    public void onEvent(Order o) {
        log.info("Received: {}" , o);
        if (o.getStatus().equals(OrderStatus.NEW))
            orderService.reserve(o);
        else
            orderService.confirm(o);
    }
}
```

- `@Component`: Spring-managed bean.
- `@KafkaListener`: Listens for messages from `ORDERS` topic, group `payments`.
- When a new order is received:
  - If status is `NEW`, tries to reserve funds.
  - Otherwise, confirms (or rolls back) the order.

---

#### viii) src/main/java/com/zatribune/spring/ecommerce/payments/service/OrderService.java

(Interface for payment logic.)

```java
package com.zatribune.spring.ecommerce.payments.service;

import domain.Order;

public interface OrderService {

    void reserve(Order order);

    void confirm(Order order);
}
```

- `reserve(Order order)`: Attempts to reserve funds for the order.
- `confirm(Order order)`: Finalizes or rolls back the reservation.

---

#### ix) src/main/java/com/zatribune/spring/ecommerce/payments/service/OrderServiceImpl.java

**Implements the business logic for handling payments.**

```java
package com.zatribune.spring.ecommerce.payments.service;

import com.zatribune.spring.ecommerce.payments.db.entities.Customer;
import com.zatribune.spring.ecommerce.payments.db.repository.CustomerRepository;
import domain.Order;
import domain.OrderSource;
import domain.Topics;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import static domain.OrderStatus.*;

@Slf4j
@Service
public class OrderServiceImpl implements OrderService{

    private static final OrderSource SOURCE = OrderSource.PAYMENT;
    private final CustomerRepository repository;
    private final KafkaTemplate<Long, Order> template;

    public OrderServiceImpl(CustomerRepository repository, KafkaTemplate<Long, Order> template) {
        this.repository = repository;
        this.template = template;
    }

    @Override
    public void reserve(Order order) {
        Customer customer = repository.findById(order.getCustomerId()).orElseThrow();
        log.info("reserve order [{}] , for customer[{}]",order.getId(), customer);
        if (order.getPrice() < customer.getAmountAvailable()) {
            order.setStatus(ACCEPT);
            customer.setAmountReserved(customer.getAmountReserved() + order.getPrice());
            customer.setAmountAvailable(customer.getAmountAvailable() - order.getPrice());
        } else {
            order.setStatus(REJECTED);
        }
        order.setSource(SOURCE);
        repository.save(customer);
        template.send(Topics.PAYMENTS, order.getId(), order);
        log.info("Sent: {}", order);
    }

    @Override
    public void confirm(Order order) {
        Customer customer = repository.findById(order.getCustomerId()).orElseThrow();
        log.info("confirm order [{}] , for customer[{}]",order.getId(), customer);
        if (order.getStatus().equals(CONFIRMED)) {
            customer.setAmountReserved(customer.getAmountReserved() - order.getPrice());
            repository.save(customer);
        } else if (order.getStatus().equals(ROLLBACK) && !order.getSource().equals(SOURCE)) {
            customer.setAmountReserved(customer.getAmountReserved() - order.getPrice());
            customer.setAmountAvailable(customer.getAmountAvailable() + order.getPrice());
            repository.save(customer);
        }

    }
}
```

- When reserving:
  - Checks if customer has enough funds.
  - If yes, marks order as accepted and updates reserved/available amounts.
  - If not, marks as rejected.
  - Always sets source to `PAYMENT`.
  - Saves customer and sends order status to Kafka.

- When confirming:
  - If order is confirmed, subtracts reserved funds.
  - If rolling back (and this service didn't initiate rollback), returns reserved funds to available.

---

#### x) src/main/java/com/zatribune/spring/ecommerce/payments/PaymentApplication.java

(Identical to ms-orders, except `@EnableKafka` instead of `@EnableKafkaStreams`.)

---

#### xi) src/main/resources/application.yml

(Configures database, Kafka, logging, etc. for payments.)

---

### 3. ms-stock

#### i) DockerFile

(Identical to ms-orders.)

---

#### ii) pom.xml

(Identical in structure to ms-orders, except artifactId is `ms-stock`.)

---

#### iii) src/test/java/com/zatribune/spring/ecommerce/stock/StockApplicationTests.java

(Identical to ms-orders test.)

---

#### iv) src/main/java/com/zatribune/spring/ecommerce/stock/db/entities/Product.java

```java
package com.zatribune.spring.ecommerce.stock.db.entities;

import lombok.*;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Entity
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private int availableItems;
    private int reservedItems;

    @Override
    public String toString() {
        return "Product{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", availableItems=" + availableItems +
                ", reservedItems=" + reservedItems +
                '}';
    }
}
```

- Like `Customer`, but for a product: `id`, `name`, amounts available and reserved.

---

#### v) src/main/java/com/zatribune/spring/ecommerce/stock/db/repository/ProductRepository.java

- JPA repository for `Product` entity.

---

#### vi) src/main/java/com/zatribune/spring/ecommerce/stock/db/DevBootstrap.java

- Populates database with four products.

---

#### vii) src/main/java/com/zatribune/spring/ecommerce/stock/listener/OrderListener.java

- Listens to Kafka ORDERS topic.
- If order is new, reserves inventory.
- Else, confirms or rolls back.

---

#### viii) src/main/java/com/zatribune/spring/ecommerce/stock/service/OrderService.java

- Interface for reserving/confirming stock.

---

#### ix) src/main/java/com/zatribune/spring/ecommerce/stock/service/OrderServiceImpl.java

**Contains logic for reserving/confirming stock.**

```java
package com.zatribune.spring.ecommerce.stock.service;

import com.zatribune.spring.ecommerce.stock.db.entities.Product;
import com.zatribune.spring.ecommerce.stock.db.repository.ProductRepository;
import domain.OrderSource;
import domain.OrderStatus;
import domain.Topics;
import domain.Order;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class OrderServiceImpl implements OrderService{

    private static final OrderSource SOURCE = OrderSource.STOCK;
    private final ProductRepository repository;
    private final KafkaTemplate<Long, Order> template;

    public OrderServiceImpl(ProductRepository repository, KafkaTemplate<Long, Order> template) {
        this.repository = repository;
        this.template = template;
    }

    @Override
    public void reserve(Order order) {
        Product product = repository.findById(order.getProductId()).orElseThrow();
        log.info("Found: {}", product);
        if (order.getStatus().equals(OrderStatus.NEW)) {
            if (order.getProductCount() < product.getAvailableItems()) {
                product.setReservedItems(product.getReservedItems() + order.getProductCount());
                product.setAvailableItems(product.getAvailableItems() - order.getProductCount());
                order.setStatus(OrderStatus.ACCEPT);
                repository.save(product);
            } else {
                order.setStatus(OrderStatus.REJECT);
            }
            template.send(Topics.STOCK, order.getId(), order);
            log.info("Sent: {}", order);
        }
    }

    @Override
    public void confirm(Order order) {
        Product product = repository.findById(order.getProductId()).orElseThrow();
        log.info("Found: {}", product);
        if (order.getStatus().equals(OrderStatus.CONFIRMED)) {
            product.setReservedItems(product.getReservedItems() - order.getProductCount());
            repository.save(product);
        } else if (order.getStatus().equals(OrderStatus.ROLLBACK) && !order.getSource().equals(SOURCE)) {
            product.setReservedItems(product.getReservedItems() - order.getProductCount());
            product.setAvailableItems(product.getAvailableItems() + order.getProductCount());
            repository.save(product);
        }
    }

}
```

- Similar to payments, but for stock.
- When reserving, checks if enough items are available.
- When confirming, releases/reserves inventory as needed.

---

#### x) src/main/java/com/zatribune/spring/ecommerce/stock/StockApplication.java

(Identical to ms-payments, but for ms-stock.)

---

#### xi) src/main/resources/application.yml

(Configures database, Kafka, logging, etc. for stock.)

---

### 4. commons

#### i) pom.xml

- Simple Maven config for a shared code module.

---

#### ii) src/main/java/domain/KafkaGroupIds.java

```java
package domain;

public interface KafkaGroupIds {
    String PAYMENTS = "payments";
    String STOCK = "stock";
}
```

- Interface with constants for Kafka group IDs.

---

#### iii) src/main/java/domain/KafkaIds.java

```java
package domain;

public interface KafkaIds {
    String ORDERS = "orders";
}
```

- Interface with constant for Kafka topic ID.

---

#### iv) src/main/java/domain/Order.java

```java
package domain;

import lombok.*;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Order {
    private Long id;
    private Long customerId;
    private Long productId;
    private int productCount;
    private int price;
    private OrderStatus status;
    private OrderSource source;

    @Override
    public String toString() {
        return "Order{" +
                "id=" + id +
                ", customerId=" + customerId +
                ", productId=" + productId +
                ", productCount=" + productCount +
                ", price=" + price +
                ", status=" + status +
                ", source=" + source +
                '}';
    }
}
```

- POJO (Plain Old Java Object) representing an order, with Lombok annotations for boilerplate code.

---

#### v) src/main/java/domain/OrderSource.java

```java
package domain;

public enum OrderSource {
    PAYMENT,STOCK
}
```

- Enum for where an order originated (payment or stock service).

---

#### vi) src/main/java/domain/OrderStatus.java

```java
package domain;

public enum OrderStatus {
    NEW,ACCEPT,REJECT,REJECTED,ROLLBACK,CONFIRMED
}
```

- Enum for order statuses.

---

#### vii) src/main/java/domain/Topics.java

```java
package domain;

public interface Topics {
    String ORDERS = "orders";
    String PAYMENTS = "payments";
    String STOCK = "stock";
}
```

- Interface with constants for Kafka topic names.

---

### 5. docker-compose.yml

**Defines all the containers (services) for local development:**

- Two Zookeeper nodes (for Kafka coordination).
- Two Kafka brokers.
- MySQL database (with persistent storage).
- Each service has its own ports, environment variables.
- MySQL created with database `commerce`, user/password `john.doe`/`1234`.

---

### 6. Root pom.xml

**Defines the multi-module Maven project:**

- Parent for all microservices and commons.
- Declares modules.
- Sets up Spring Boot as the main parent for dependency management.
- Includes Lombok (for code generation).
- Configures build plugins for Spring Boot and Maven release.

---

## Glossary of Terms

- **Microservice**: An independently deployable service, focused on a single business function.
- **Controller**: A class that handles HTTP requests/responses in web APIs.
- **Kafka**: A system for passing messages (events) between distributed services.
- **Docker**: A tool for packaging apps and dependencies into containers.
- **JPA**: Java standard for mapping objects to database tables.
- **Bean**: A Spring-managed object.
- **Kafka Topic**: A named queue for events/messages.
- **Kafka Stream**: A real-time processing pipeline for Kafka topics.
- **Kafka Group**: A set of consumers sharing a workload.
- **Producer**: Sends messages to Kafka.
- **Consumer**: Reads messages from Kafka.
- **Entity**: A Java class mapped to a database table.
- **Repository**: A class for CRUD database operations.
- **Service**: Business logic layer.
- **Test**: Checks if your code works as expected.

---

## Final Notes

If you have more questions about any file or concept, or want a visual diagram of how these services interact, just ask!

