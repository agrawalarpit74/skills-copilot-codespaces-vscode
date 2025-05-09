Okay, this is a great microservice use case! Let's design a robust and extendable Spring Boot project for this.

**Core Design Principles:**

1.  **Modularity:** Each data source (Kafka topic, API) should have its own dedicated component (listener or client).
2.  **Abstraction:** Use interfaces for data fetching and publishing to allow for easy swapping or adding of implementations.
3.  **Configuration-Driven:** Most settings (topic names, API URLs, credentials) should be externalized.
4.  **Standardization:** Transform incoming data (if necessary) into a common format before sending to Pub/Sub.
5.  **Error Handling & Resilience:** Implement proper error handling, retries (where appropriate), and logging.
6.  **Asynchronous Operations:** Leverage Spring's asynchronous capabilities for non-blocking operations.

**Project Structure:**

```
data-ingestion-service/
├── pom.xml
├── .gitignore
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── example/
│   │   │           └── dataingestionservice/
│   │   │               ├── DataIngestionServiceApplication.java
│   │   │               ├── config/
│   │   │               │   ├── KafkaConsumerConfig.java
│   │   │               │   ├── GcpPubSubConfig.java
│   │   │               │   ├── WebClientConfig.java (or RestTemplateConfig)
│   │   │               │   └── AppConfig.java (for general beans like ObjectMapper)
│   │   │               ├── model/
│   │   │               │   ├── GenericEvent.java      // Common format for Pub/Sub
│   │   │               │   └── SourceEvent.java       // Wrapper for raw events with source info
│   │   │               ├── connector/
│   │   │               │   ├── kafka/
│   │   │               │   │   ├── GithubKafkaListener.java
│   │   │               │   │   ├── SonarQubeKafkaListener.java
│   │   │               │   │   ├── RallyKafkaListener.java
│   │   │               │   │   └── JiraKafkaListener.java
│   │   │               │   └── api/
│   │   │               │       ├── ApiClient.java         // Interface for API fetching
│   │   │               │       ├── SomePlatformApiClient.java // Example implementation
│   │   │               │       └── ApiPollingScheduler.java // For scheduled API calls
│   │   │               ├── service/
│   │   │               │   ├── EventProcessorService.java // Processes and forwards events
│   │   │               │   └── PubSubPublishService.java  // Handles publishing to GCP Pub/Sub
│   │   │               ├── exception/
│   │   │               │   ├── DataIngestionException.java
│   │   │               │   └── GlobalExceptionHandler.java
│   │   │               └── util/
│   │   │                   └── JsonUtil.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── logback-spring.xml (optional, for custom logging)
│   └── test/
│       └── java/
│           └── com/
│               └── example/
│                   └── dataingestionservice/
│                       └── // unit and integration tests
```

---

**1. `pom.xml` (Key Dependencies)**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.1.5</version> <!-- Or latest stable Spring Boot 3.x -->
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>data-ingestion-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>data-ingestion-service</name>
    <description>Microservice to fetch data from various sources and send to GCP Pub/Sub</description>

    <properties>
        <java.version>17</java.version>
        <spring-cloud-gcp.version>4.8.4</spring-cloud-gcp.version> <!-- Check for latest -->
        <spring-cloud.version>2022.0.4</spring-cloud.version> <!-- Check for latest compatible -->
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.google.cloud</groupId>
                <artifactId>spring-cloud-gcp-dependencies</artifactId>
                <version>${spring-cloud-gcp.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId> <!-- For Actuator, health checks, etc. -->
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId> <!-- For WebClient (async HTTP) -->
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        <dependency>
            <groupId>com.google.cloud</groupId>
            <artifactId>spring-cloud-gcp-starter-pubsub</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Test Dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

**2. `application.yml` (Configuration)**

```yaml
spring:
  application:
    name: data-ingestion-service
  kafka:
    consumer:
      bootstrap-servers: localhost:9092 # Your Kafka brokers
      group-id: data-ingestion-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer # Assuming JSON strings
      properties:
        spring.json.trusted.packages: "*" # Or specific packages if known
    listener:
      ack-mode: MANUAL_IMMEDIATE # For better control and error handling

  cloud:
    gcp:
      project-id: your-gcp-project-id # Your GCP Project ID
      # Credentials can be auto-detected if running on GCP, or set via GOOGLE_APPLICATION_CREDENTIALS env var
      # pubsub:
        # emulator-host: localhost:8085 # Optional: for local testing with Pub/Sub emulator

# Application specific configurations
app:
  kafka:
    topics:
      github: github-webhook-events
      sonarqube: sonarqube-webhook-events
      rally: rally-webhook-events
      jira: jira-webhook-events
  gcp:
    pubsub:
      topic-name: projects/your-gcp-project-id/topics/ingested-events # Full topic path
  api:
    some-platform:
      base-url: https://api.someplatform.com/v1
      api-key: ${SOME_PLATFORM_API_KEY} # Use environment variables for secrets

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics

logging:
  level:
    com.example.dataingestionservice: DEBUG
    org.springframework.kafka: INFO
    com.google.cloud.pubsub: INFO
```
**Note on Secrets:** For API keys, use Spring Cloud Config Server, HashiCorp Vault, Kubernetes Secrets, or environment variables. Avoid hardcoding in `application.yml`.

---

**3. Java Source Code Files:**

**`DataIngestionServiceApplication.java`**
```java
package com.example.dataingestionservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling // To enable @Scheduled tasks for API polling
public class DataIngestionServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(DataIngestionServiceApplication.class, args);
    }
}
```

**`config/KafkaConsumerConfig.java`**
(Spring Boot auto-configures much of Kafka, but you might need this for custom error handlers, retry templates, or specific `ConsumerFactory` settings. For simple cases, `application.yml` might be enough.)
```java
package com.example.dataingestionservice.config;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ContainerProperties;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.util.backoff.FixedBackOff;

import java.util.HashMap;
import java.util.Map;

@EnableKafka
@Configuration
public class KafkaConsumerConfig {

    private static final Logger logger = LoggerFactory.getLogger(KafkaConsumerConfig.class);

    @Value("${spring.kafka.consumer.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${spring.kafka.consumer.group-id}")
    private String groupId;

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        // Add any other consumer properties here
        // props.put(JsonDeserializer.TRUSTED_PACKAGES, "*"); // If using JsonDeserializer
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        // Example: Simple Error Handler (logs and skips)
        // For production, consider a DeadLetterPublishingRecoverer
        DefaultErrorHandler errorHandler = new DefaultErrorHandler((consumerRecord, exception) -> {
            logger.error("Error processing Kafka message: {} from topic {}. Skipping.",
                         consumerRecord.value(), consumerRecord.topic(), exception);
        }, new FixedBackOff(1000L, 2L)); // Retry twice with 1s interval
        factory.setCommonErrorHandler(errorHandler);

        return factory;
    }
}
```

**`config/GcpPubSubConfig.java`**
(Spring Cloud GCP auto-configures `PubSubTemplate`. This is mostly for illustration or if you need custom `Publisher` beans.)
```java
package com.example.dataingestionservice.config;

// import com.google.cloud.pubsub.v1.Publisher;
// import com.google.protobuf.ByteString;
// import com.google.pubsub.v1.PubsubMessage;
// import com.google.pubsub.v1.TopicName;
// import org.springframework.beans.factory.annotation.Value;
// import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
// import java.io.IOException;

@Configuration
public class GcpPubSubConfig {

    // Spring Cloud GCP auto-configures PubSubTemplate, which is generally preferred.
    // If you need a direct Publisher bean:
    /*
    @Value("${app.gcp.pubsub.topic-name}")
    private String topicNameStr; // e.g., projects/your-project/topics/your-topic

    @Bean
    public Publisher publisher() throws IOException {
        TopicName topicName = TopicName.parse(topicNameStr);
        return Publisher.newBuilder(topicName).build();
    }
    */
}
```

**`config/WebClientConfig.java`**
```java
package com.example.dataingestionservice.config;

import io.netty.channel.ChannelOption;
import io.netty.handler.timeout.ReadTimeoutHandler;
import io.netty.handler.timeout.WriteTimeoutHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.netty.http.client.HttpClient;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

@Configuration
public class WebClientConfig {

    private static final int CONNECT_TIMEOUT_MILLIS = 5000;
    private static final int RESPONSE_TIMEOUT_SECONDS = 10;
    private static final int READ_TIMEOUT_SECONDS = 10;
    private static final int WRITE_TIMEOUT_SECONDS = 10;

    @Bean
    public WebClient defaultWebClient() {
        HttpClient httpClient = HttpClient.create()
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, CONNECT_TIMEOUT_MILLIS)
                .responseTimeout(Duration.ofSeconds(RESPONSE_TIMEOUT_SECONDS))
                .doOnConnected(conn ->
                        conn.addHandlerLast(new ReadTimeoutHandler(READ_TIMEOUT_SECONDS, TimeUnit.SECONDS))
                            .addHandlerLast(new WriteTimeoutHandler(WRITE_TIMEOUT_SECONDS, TimeUnit.SECONDS)));

        return WebClient.builder()
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .build();
    }
}
```

**`config/AppConfig.java`**
```java
package com.example.dataingestionservice.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class AppConfig {

    @Bean
    @Primary // Make this the default ObjectMapper
    public ObjectMapper objectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(new JavaTimeModule()); // For Java 8 Date/Time
        // Configure other properties as needed
        // objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return objectMapper;
    }
}
```

**`model/GenericEvent.java`**
```java
package com.example.dataingestionservice.model;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.time.Instant;
import java.util.Map;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class GenericEvent {
    private String eventId;       // Unique ID for this event, can be generated
    private String sourceType;    // e.g., "GITHUB", "SONARQUBE", "API_SOMEPLATFORM"
    private String eventType;     // e.g., "PUSH", "QUALITY_GATE_CHANGED", "USER_STORY_UPDATED"
    private Instant timestamp;    // When the event occurred or was ingested
    private Map<String, String> metadata; // Optional: common metadata like correlation ID
    private Object payload;       // The actual event data (can be Map<String, Object> or raw JSON string)
}
```

**`model/SourceEvent.java`** (Optional, if you want to wrap raw messages with source info early)
```java
package com.example.dataingestionservice.model;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class SourceEvent<T> {
    private String sourceSystem; // "GITHUB", "SONARQUBE", etc.
    private T rawPayload;
    private String originalTopic; // For Kafka events
}
```

**`connector/kafka/GithubKafkaListener.java`**
```java
package com.example.dataingestionservice.connector.kafka;

import com.example.dataingestionservice.model.SourceEvent;
import com.example.dataingestionservice.service.EventProcessorService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

@Component
public class GithubKafkaListener {

    private static final Logger logger = LoggerFactory.getLogger(GithubKafkaListener.class);
    private final EventProcessorService eventProcessorService;

    @Autowired
    public GithubKafkaListener(EventProcessorService eventProcessorService) {
        this.eventProcessorService = eventProcessorService;
    }

    @KafkaListener(topics = "${app.kafka.topics.github}",
                   groupId = "${spring.kafka.consumer.group-id}",
                   containerFactory = "kafkaListenerContainerFactory") // Refer to the bean in KafkaConsumerConfig
    public void listenGithubEvents(@Payload String message, Acknowledgment acknowledgment) {
        logger.debug("Received Github event: {}", message);
        try {
            SourceEvent<String> sourceEvent = new SourceEvent<>("GITHUB", message, "app.kafka.topics.github");
            eventProcessorService.processEvent(sourceEvent);
            acknowledgment.acknowledge(); // Acknowledge message on successful processing
            logger.info("Successfully processed and acknowledged Github event.");
        } catch (Exception e) {
            logger.error("Error processing Github event: {}. Manual acknowledgment will prevent redelivery based on error handler.", message, e);
            // The DefaultErrorHandler in KafkaConsumerConfig will handle this.
            // If you need specific non-retryable logic, you might throw a specific exception
            // and configure the error handler to not retry for that exception.
            // Acknowledgment here is tricky with MANUAL_IMMEDIATE and a global error handler.
            // The global error handler might attempt retries before this catch block is even hit
            // if the deserialization or initial listener invocation fails.
            // If processing in eventProcessorService fails, and you want to ensure it's not retried,
            // the DefaultErrorHandler should be configured accordingly, or you might consider a
            // different AckMode or more complex error handling strategy.
            // For now, we assume the DefaultErrorHandler handles retry/skip logic.
        }
    }
}
// Similar listeners for SonarQube, Rally, JIRA
// SonarQubeKafkaListener.java, RallyKafkaListener.java, JiraKafkaListener.java
```

**`connector/api/ApiClient.java`** (Interface)
```java
package com.example.dataingestionservice.connector.api;

import reactor.core.publisher.Flux; // Or Mono<List<SomeData>> etc.
import com.example.dataingestionservice.model.SourceEvent;

public interface ApiClient<T> {
    String getSourceSystemName();
    Flux<SourceEvent<T>> fetchData(); // Returns a Flux of events from the API
}
```

**`connector/api/SomePlatformApiClient.java`** (Example Implementation)
```java
package com.example.dataingestionservice.connector.api;

import com.example.dataingestionservice.model.SourceEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Flux;
import com.fasterxml.jackson.databind.ObjectMapper; // Use your configured ObjectMapper

import java.util.Map; // Or a specific DTO for the platform's data

@Component("somePlatformApiClient") // Give it a name
public class SomePlatformApiClient implements ApiClient<Map<String, Object>> { // Assuming API returns JSON objects

    private static final Logger logger = LoggerFactory.getLogger(SomePlatformApiClient.class);
    private final WebClient webClient;
    private final String baseUrl;
    private final String apiKey;
    private final ObjectMapper objectMapper; // Autowire your configured ObjectMapper

    private static final String SOURCE_SYSTEM_NAME = "SOME_PLATFORM";

    @Autowired
    public SomePlatformApiClient(WebClient defaultWebClient, // Use the pre-configured bean
                                 @Value("${app.api.some-platform.base-url}") String baseUrl,
                                 @Value("${app.api.some-platform.api-key}") String apiKey,
                                 ObjectMapper objectMapper) {
        this.webClient = defaultWebClient.mutate().baseUrl(baseUrl).build();
        this.baseUrl = baseUrl;
        this.apiKey = apiKey;
        this.objectMapper = objectMapper;
    }

    @Override
    public String getSourceSystemName() {
        return SOURCE_SYSTEM_NAME;
    }

    @Override
    @SuppressWarnings("unchecked") // If payload is Map
    public Flux<SourceEvent<Map<String, Object>>> fetchData() {
        logger.info("Fetching data from {}", SOURCE_SYSTEM_NAME);
        return webClient.get()
                .uri("/data-endpoint?param=value") // Adjust URI as needed
                .header("Authorization", "Bearer " + apiKey) // Or other auth mechanism
                .retrieve()
                .bodyToFlux(Map.class) // Expecting a stream of JSON objects, or use bodyToMono(List.class)
                .map(payload -> new SourceEvent<>(SOURCE_SYSTEM_NAME, (Map<String, Object>)payload, null))
                .doOnError(error -> logger.error("Error fetching data from {}: {}", SOURCE_SYSTEM_NAME, error.getMessage()))
                .onErrorResume(e -> Flux.empty()); // Continue with empty if error, or handle differently
    }
}
```

**`connector/api/ApiPollingScheduler.java`**
```java
package com.example.dataingestionservice.connector.api;

import com.example.dataingestionservice.service.EventProcessorService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class ApiPollingScheduler {

    private static final Logger logger = LoggerFactory.getLogger(ApiPollingScheduler.class);
    private final List<ApiClient<?>> apiClients; // Spring injects all beans implementing ApiClient
    private final EventProcessorService eventProcessorService;

    @Autowired
    public ApiPollingScheduler(List<ApiClient<?>> apiClients, EventProcessorService eventProcessorService) {
        this.apiClients = apiClients;
        this.eventProcessorService = eventProcessorService;
        logger.info("Initialized ApiPollingScheduler with {} API clients.", apiClients.size());
        apiClients.forEach(client -> logger.info("Registered API client: {}", client.getSourceSystemName()));
    }

    // Example: Poll every 5 minutes. Use cron expression for more complex schedules.
    @Scheduled(fixedRateString = "${app.api.polling.fixedRate:300000}") // Default 5 mins, configurable
    public void pollApis() {
        logger.info("Starting API polling cycle...");
        apiClients.forEach(client -> {
            logger.debug("Polling API for source: {}", client.getSourceSystemName());
            client.fetchData()
                .subscribe(
                    sourceEvent -> {
                        try {
                            eventProcessorService.processEvent(sourceEvent);
                        } catch (Exception e) {
                            logger.error("Error processing event from API client {}: {}",
                                    client.getSourceSystemName(), sourceEvent, e);
                        }
                    },
                    error -> logger.error("Error during API fetch for {}: {}", client.getSourceSystemName(), error.getMessage()),
                    () -> logger.debug("Finished polling API for source: {}", client.getSourceSystemName())
                );
        });
    }
}
```

**`service/EventProcessorService.java`**
```java
package com.example.dataingestionservice.service;

import com.example.dataingestionservice.model.GenericEvent;
import com.example.dataingestionservice.model.SourceEvent;
import com.example.dataingestionservice.util.JsonUtil;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

@Service
public class EventProcessorService {

    private static final Logger logger = LoggerFactory.getLogger(EventProcessorService.class);
    private final PubSubPublishService pubSubPublishService;
    private final ObjectMapper objectMapper; // Use the configured one

    @Autowired
    public EventProcessorService(PubSubPublishService pubSubPublishService, ObjectMapper objectMapper) {
        this.pubSubPublishService = pubSubPublishService;
        this.objectMapper = objectMapper;
    }

    public void processEvent(SourceEvent<?> sourceEvent) {
        logger.debug("Processing event from source: {}", sourceEvent.getSourceSystem());
        try {
            GenericEvent genericEvent = transformToGenericEvent(sourceEvent);
            pubSubPublishService.publishEvent(genericEvent);
            logger.info("Successfully processed and queued event ID: {} from source: {}",
                        genericEvent.getEventId(), genericEvent.getSourceType());
        } catch (Exception e) {
            logger.error("Failed to process event from source {}: {}", sourceEvent.getSourceSystem(), sourceEvent.getRawPayload(), e);
            // Depending on requirements, you might re-throw a custom exception
            // or handle specific errors (e.g., send to a dead-letter topic)
            throw new RuntimeException("Event processing failed for source: " + sourceEvent.getSourceSystem(), e);
        }
    }

    private GenericEvent transformToGenericEvent(SourceEvent<?> sourceEvent) {
        // This is a crucial part: How do you standardize events?
        // For simplicity, we'll try to parse the payload as JSON into a Map.
        // You might need more sophisticated logic based on event types.

        Object payload = sourceEvent.getRawPayload();
        Map<String, Object> payloadMap = null;
        String eventType = "UNKNOWN"; // Default event type

        if (payload instanceof String) {
            try {
                payloadMap = objectMapper.readValue((String) payload, new TypeReference<Map<String, Object>>() {});
                // Attempt to extract a common event type field if it exists
                // This is highly dependent on your webhook structures
                if (sourceEvent.getSourceSystem().equals("GITHUB") && payloadMap.containsKey("action")) {
                    eventType = (String) payloadMap.get("action");
                } else if (sourceEvent.getSourceSystem().equals("JIRA") && payloadMap.containsKey("webhookEvent")) {
                    eventType = (String) payloadMap.get("webhookEvent");
                } // Add more specific event type extractions
            } catch (Exception e) {
                logger.warn("Could not parse payload string as JSON for source {}. Storing as raw string.",
                            sourceEvent.getSourceSystem(), e);
                // If it's not JSON or parsing fails, keep the raw string payload
            }
        } else if (payload instanceof Map) {
             // If payload is already a Map (e.g. from API client)
            payloadMap = (Map<String, Object>) payload;
             // Similar logic to extract eventType if applicable
        }


        Map<String, String> metadata = new HashMap<>();
        if (sourceEvent.getOriginalTopic() != null) {
            metadata.put("originalKafkaTopic", sourceEvent.getOriginalTopic());
        }

        return new GenericEvent(
                UUID.randomUUID().toString(),
                sourceEvent.getSourceSystem(),
                eventType.toUpperCase(), // Standardize event type to uppercase
                Instant.now(),
                metadata,
                payloadMap != null ? payloadMap : payload // Use parsed map or original payload
        );
    }
}
```

**`service/PubSubPublishService.java`**
```java
package com.example.dataingestionservice.service;

import com.example.dataingestionservice.model.GenericEvent;
import com.example.dataingestionservice.util.JsonUtil;
import com.google.cloud.spring.pubsub.core.PubSubTemplate;
import com.google.cloud.spring.pubsub.support.BasicAcknowledgeablePubsubMessage;
import com.google.cloud.spring.pubsub.support.GcpPubSubHeaders;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Service;
import org.springframework.util.concurrent.ListenableFutureCallback;

import java.util.concurrent.CompletableFuture;

@Service
public class PubSubPublishService {

    private static final Logger logger = LoggerFactory.getLogger(PubSubPublishService.class);
    private final PubSubTemplate pubSubTemplate;
    private final String topicName; // Full topic path: projects/project-id/topics/topic-id

    @Autowired
    public PubSubPublishService(PubSubTemplate pubSubTemplate,
                                @Value("${app.gcp.pubsub.topic-name}") String topicName) {
        this.pubSubTemplate = pubSubTemplate;
        this.topicName = topicName;
    }

    public void publishEvent(GenericEvent event) {
        try {
            String jsonPayload = JsonUtil.toJson(event); // Using a utility for serialization
            logger.debug("Publishing to Pub/Sub topic {}: {}", topicName, jsonPayload);

            CompletableFuture<String> future = pubSubTemplate.publish(this.topicName, jsonPayload);

            future.whenComplete((messageId, throwable) -> {
                if (throwable != null) {
                    logger.error("Failed to publish message ID {} to Pub/Sub topic {}: {}",
                                 event.getEventId(), topicName, throwable.getMessage());
                    // Consider retry logic or dead-lettering here for critical failures
                } else {
                    logger.info("Successfully published message ID {} (Pub/Sub ID: {}) to topic {}",
                                event.getEventId(), messageId, topicName);
                }
            });

        } catch (Exception e) {
            logger.error("Error serializing or publishing event ID {} to Pub/Sub: {}", event.getEventId(), e.getMessage(), e);
            // Handle exception (e.g., metrics, alerts)
        }
    }

    // Alternative using Spring Messaging Message, which can be more flexible
    public void publishEventWithMessage(GenericEvent event) {
        try {
            String jsonPayload = JsonUtil.toJson(event);
            Message<String> message = MessageBuilder.withPayload(jsonPayload)
                // .setHeader(GcpPubSubHeaders.ORDERING_KEY, event.getSomeOrderingKey()) // If needed
                .build();

            pubSubTemplate.send(this.topicName, message)
                .whenComplete((messageId, throwable) -> {
                    if (throwable != null) {
                        logger.error("Failed to publish (Message API) event ID {} to Pub/Sub topic {}: {}",
                                     event.getEventId(), topicName, throwable.getMessage());
                    } else {
                        logger.info("Successfully published (Message API) event ID {} (Pub/Sub ID: {}) to topic {}",
                                    event.getEventId(), messageId, topicName);
                    }
                });

        } catch (Exception e) {
            logger.error("Error serializing or publishing (Message API) event ID {} to Pub/Sub: {}", event.getEventId(), e.getMessage(), e);
        }
    }
}
```

**`util/JsonUtil.java`**
```java
package com.example.dataingestionservice.util;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

public class JsonUtil {

    // It's better to inject the Spring-configured ObjectMapper
    // This is a fallback or for static contexts if needed.
    private static final ObjectMapper objectMapper = new ObjectMapper()
            .registerModule(new JavaTimeModule()); // For Java 8 Date/Time serialization

    public static String toJson(Object obj) throws JsonProcessingException {
        return objectMapper.writeValueAsString(obj);
    }

    public static <T> T fromJson(String json, Class<T> clazz) throws JsonProcessingException {
        return objectMapper.readValue(json, clazz);
    }
}
```
**Note:** It's generally better to `@Autowired` the `ObjectMapper` bean created in `AppConfig` rather than having a static one in `JsonUtil`, to ensure consistency with Spring's configuration. Modify `PubSubPublishService` and `EventProcessorService` to use the autowired `ObjectMapper`.

**`exception/DataIngestionException.java` & `GlobalExceptionHandler.java`**
(Standard custom exception and `@ControllerAdvice` for REST APIs if you add any. Not strictly necessary if you only have Kafka listeners and scheduled tasks, but good practice.)

---

**Key Considerations & Next Steps:**

1.  **Error Handling & Dead Letter Queues (DLQs):**
    *   **Kafka:** Configure `DefaultErrorHandler` with a `DeadLetterPublishingRecoverer` to send poison-pill messages to a DLQ topic if retries fail.
    *   **Pub/Sub:** GCP Pub/Sub supports dead-letter topics at the subscription level. Configure this in GCP.
    *   **API Calls:** Implement retries (e.g., with `WebClient`'s retry mechanisms or Spring Retry) for transient network issues.
2.  **Idempotency:** If downstream systems require it, ensure your processing logic can handle duplicate messages gracefully (e.g., if Kafka redelivers or Pub/Sub has "at-least-once" delivery). `GenericEvent.eventId` can help.
3.  **Schema Management:** For Kafka and potentially Pub/Sub, consider using a schema registry (like Confluent Schema Registry for Kafka, or Avro/Protobuf schemas with Pub/Sub) for robust data contracts.
4.  **Security:**
    *   Secure API keys (Vault, Spring Cloud Config, environment variables).
    *   Use TLS for Kafka and GCP communications.
    *   Proper IAM roles for the service account running this microservice on GCP.
5.  **Testing:**
    *   **Unit Tests:** For services, mappers, API clients (using `MockWebServer`).
    *   **Integration Tests:** With embedded Kafka (`spring-kafka-test`) and Pub/Sub emulator.
6.  **Observability:**
    *   **Logging:** Already included with SLF4J.
    *   **Metrics:** Spring Boot Actuator + Micrometer (e.g., Prometheus). Track messages processed, errors, latency.
    *   **Tracing:** Spring Cloud Sleuth / OpenTelemetry for distributed tracing if this service calls others or is part of a larger flow.
7.  **Configuration Management:** For different environments (dev, staging, prod), use Spring profiles and externalized configuration.
8.  **Payload Transformation Complexity:** The `transformToGenericEvent` method is a critical point. If transformations are complex or vary wildly, you might need a strategy pattern or dedicated transformer classes per source.
9.  **Backpressure:** If your service ingests data faster than it can publish to Pub/Sub, `PubSubTemplate` handles some buffering, but monitor queue depths and consider rate limiting or scaling if necessary. WebFlux for API calls helps manage backpressure from reactive streams.

This comprehensive structure should give you a solid foundation for building your data ingestion microservice. Remember to adapt and refine based on the specific details of your webhook payloads and API responses.
