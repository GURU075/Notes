# OpenFeign, WebClient, and RestClient in Spring Boot

## 1. Overview

Spring Boot applications often call external REST APIs or other microservices. The three commonly used clients are **Spring Cloud OpenFeign**, **WebClient**, and **RestClient**.

- **OpenFeign** is a declarative, interface-based HTTP client.
- **WebClient** is a reactive and non-blocking HTTP client.
- **RestClient** is a modern, fluent, synchronous HTTP client.

The correct choice depends mainly on whether your application is imperative or reactive, whether streaming is required, and whether your team prefers declarative interfaces or explicit request-building code.

> Current Spring direction: use `RestClient` for synchronous HTTP access and `WebClient` for reactive, asynchronous, or streaming use cases. Spring Cloud OpenFeign remains widely used for declarative microservice communication, but Spring describes it as feature-complete.

---

## 2. Why It Matters

### Where it is used

- Microservice-to-microservice communication
- Calling payment, order, inventory, user, and notification services
- Integrating with third-party APIs
- Uploading and downloading files
- Consuming Server-Sent Events or streaming APIs
- Sending JWT, OAuth2 tokens, API keys, and tracing headers
- Client-side load balancing and service discovery

### Why you should learn it

- It is frequently asked in Java backend and microservices interviews.
- The wrong client can add unnecessary complexity.
- Production HTTP calls require timeout, authentication, error handling, retries, observability, and resilience.
- You must understand blocking versus non-blocking execution.

### What problem it solves

Without a client abstraction, developers would have to manually:

- Open HTTP connections
- Construct URLs and headers
- Convert Java objects to JSON
- Convert JSON into Java objects
- Handle response status codes
- Manage connection pools and timeouts

---

# 3. Foundation Concepts

## 3.1 What is an HTTP client?

An HTTP client sends a request from one application to another HTTP server.

```text
Order Service
     |
     | GET /api/products/101
     v
Product Service
```

The Order Service is the client. The Product Service is the server.

## 3.2 Synchronous or blocking communication

In a synchronous call, the calling thread waits until the remote service returns a response or an error.

```text
Send request -> Wait -> Receive response -> Continue
```

```java
ProductResponse product = productClient.getProduct(101L);
System.out.println(product);
```

The second statement runs only after the HTTP call completes.

Use this model for:

- Traditional Spring MVC applications
- Simple request-response communication
- Normal CRUD integrations
- Applications where blocking is acceptable

## 3.3 Reactive or non-blocking communication

In a reactive call, the result is represented by a publisher such as `Mono<T>` or `Flux<T>`. The application can continue processing other work instead of keeping one thread blocked for every waiting request.

```java
Mono<ProductResponse> productMono = webClient.get()
        .uri("/products/{id}", 101L)
        .retrieve()
        .bodyToMono(ProductResponse.class);
```

Use this model for:

- Spring WebFlux applications
- Large numbers of concurrent I/O calls
- Streaming APIs
- Server-Sent Events
- Reactive composition of multiple services

## 3.4 Declarative versus programmatic clients

### Declarative client

You describe the HTTP endpoint using an interface and annotations.

```java
@FeignClient(name = "product-service")
public interface ProductClient {

    @GetMapping("/api/products/{id}")
    ProductResponse getProduct(@PathVariable Long id);
}
```

Spring creates the implementation.

### Programmatic or fluent client

You explicitly build the request.

```java
ProductResponse product = restClient.get()
        .uri("/api/products/{id}", id)
        .retrieve()
        .body(ProductResponse.class);
```

---

# 4. Shared DTOs Used in Examples

```java
public record ProductResponse(
        Long id,
        String name,
        BigDecimal price
) {
}
```

```java
public record CreateProductRequest(
        String name,
        BigDecimal price
) {
}
```

```java
public record ApiError(
        int status,
        String message,
        String path
) {
}
```

---

# 5. Spring Cloud OpenFeign

## 5.1 Overview

OpenFeign is a declarative REST client. You define a Java interface, annotate it with `@FeignClient`, and declare methods using Spring MVC annotations. Spring creates a proxy at runtime.

```text
Interface -> Spring proxy -> HTTP request -> Remote service
```

## 5.2 Advantages

- Minimal boilerplate
- Easy-to-read API contracts
- Familiar Spring MVC annotations
- Convenient for service-to-service communication
- Spring Cloud LoadBalancer integration
- Supports interceptors, encoders, decoders, logging, OAuth2, and resilience integrations
- Easy to mock in service-layer tests

## 5.3 Disadvantages

- Primarily blocking
- Implementation details are hidden behind a proxy
- Requires Spring Cloud dependency management
- Retry configuration can be dangerous for non-idempotent calls
- Low-level behavior may be less visible
- The project is feature-complete, so evaluate Spring HTTP interfaces for new designs

## 5.4 Dependency

Use a Spring Cloud release compatible with your Spring Boot version.

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

Do not randomly copy Spring Cloud versions from tutorials. Verify version compatibility.

## 5.5 Enable Feign clients

```java
@SpringBootApplication
@EnableFeignClients
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

For multi-module applications:

```java
@EnableFeignClients(basePackages = "com.example.order.client")
```

## 5.6 GET example

```java
@FeignClient(
        name = "product-service",
        url = "${clients.product-service.base-url}"
)
public interface ProductClient {

    @GetMapping("/api/products/{id}")
    ProductResponse getProductById(
            @PathVariable("id") Long id
    );
}
```

```yaml
clients:
  product-service:
    base-url: http://localhost:8081
```

Usage:

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final ProductClient productClient;

    public ProductResponse fetchProduct(Long id) {
        return productClient.getProductById(id);
    }
}
```

## 5.7 POST example

```java
@FeignClient(
        name = "product-service",
        url = "${clients.product-service.base-url}"
)
public interface ProductClient {

    @PostMapping("/api/products")
    ProductResponse createProduct(
            @RequestBody CreateProductRequest request
    );
}
```

```java
ProductResponse response = productClient.createProduct(
        new CreateProductRequest(
                "Mechanical Keyboard",
                new BigDecimal("89.99")
        )
);
```

## 5.8 Query parameters and headers

```java
@GetMapping("/api/products")
List<ProductResponse> searchProducts(
        @RequestParam("category") String category,
        @RequestParam("active") boolean active,
        @RequestHeader("X-Correlation-Id") String correlationId
);
```

## 5.9 Request interceptor

Use an interceptor for common headers.

```java
@Configuration
public class ProductFeignConfiguration {

    @Bean
    public RequestInterceptor correlationIdInterceptor() {
        return template -> {
            String correlationId = MDC.get("correlationId");

            if (correlationId != null && !correlationId.isBlank()) {
                template.header("X-Correlation-Id", correlationId);
            }
        };
    }
}
```

Attach client-specific configuration explicitly:

```java
@FeignClient(
        name = "product-service",
        url = "${clients.product-service.base-url}",
        configuration = ProductFeignConfiguration.class
)
public interface ProductClient {
}
```

## 5.10 Error decoder

```java
public class ProductErrorDecoder implements ErrorDecoder {

    private final ErrorDecoder defaultDecoder = new Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        return switch (response.status()) {
            case 404 -> new ProductNotFoundException(
                    "Product was not found"
            );
            case 409 -> new ProductConflictException(
                    "Product conflict occurred"
            );
            default -> defaultDecoder.decode(methodKey, response);
        };
    }
}
```

Register it:

```java
@Bean
public ErrorDecoder productErrorDecoder() {
    return new ProductErrorDecoder();
}
```

## 5.11 Timeouts

Configure both:

- Connection timeout
- Read timeout

Example configuration shape:

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          product-service:
            connectTimeout: 2000
            readTimeout: 3000
            loggerLevel: basic
```

Property names can vary between Spring Cloud generations. Verify your project's official documentation.

## 5.12 Service discovery

When service discovery and client-side load balancing are configured, use a logical name:

```java
@FeignClient(name = "product-service")
public interface ProductClient {
}
```

```text
product-service
   +-- instance 1
   +-- instance 2
   +-- instance 3
```

## 5.13 Circuit breaker and fallback

A circuit breaker can fail fast when a dependency is unhealthy.

```text
CLOSED -> normal calls
OPEN -> fail fast
HALF_OPEN -> limited test calls
```

Avoid fake fallback data.

Bad:

```java
return new ProductResponse(0L, "Product", BigDecimal.ZERO);
```

Better options:

- Throw a domain-specific availability exception
- Return explicitly degraded data
- Use safe cached data where business rules permit it
- Clearly mark dependency failure

## 5.14 When to use OpenFeign

Use it when:

- Your application is Spring MVC or otherwise imperative
- Calls are synchronous
- You prefer concise declarative interfaces
- You have many simple internal service endpoints
- Spring Cloud is already part of the architecture
- You use service discovery and load balancing

Avoid or reconsider it when:

- The application is fully reactive
- Streaming is required
- You need reactive composition
- You want fewer Spring Cloud dependencies
- You prefer Spring Framework HTTP interfaces
- The API requires unusual low-level HTTP handling

## 5.15 Interview answer: What is OpenFeign?

> Spring Cloud OpenFeign is a declarative, blocking HTTP client used mainly for synchronous service-to-service communication. We define an interface using `@FeignClient` and Spring MVC annotations, and Spring creates a proxy that converts method calls into HTTP requests. In production I configure connection and read timeouts, error decoding, authentication or tracing interceptors, logging, and resilience behavior.

## 5.16 Interview answer: Is OpenFeign non-blocking?

> No. The usual Spring Cloud OpenFeign model is blocking. The calling thread waits for the response. For end-to-end reactive communication or streaming, I use WebClient.

## 5.17 Interview answer: What happens internally?

> Spring scans `@FeignClient` interfaces, registers proxy beans, converts method annotations and arguments into HTTP requests, uses encoders and decoders for payload conversion, and delegates the call to the configured HTTP transport.

---

# 6. WebClient

## 6.1 Overview

`WebClient` is Spring's reactive HTTP client. It uses a fluent API and normally returns:

- `Mono<T>` for zero or one value
- `Flux<T>` for zero to many values

It supports non-blocking I/O, reactive composition, and streaming.

## 6.2 Advantages

- Non-blocking execution
- Reactive composition
- Excellent streaming support
- Suitable for high-concurrency I/O
- Fluent API
- Supports filters, codecs, status handling, and exchange customization
- Works well with Spring WebFlux

## 6.3 Disadvantages

- Higher learning curve
- Reactor stack traces can be difficult to understand
- Incorrect `.block()` usage defeats reactive design
- Blocking repositories or SDKs can damage reactive performance
- Often unnecessary for simple Spring MVC CRUD applications

## 6.4 Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

A Spring MVC application can use WebClient, but adding reactive complexity only to block every call is normally unnecessary when RestClient is available.

## 6.5 Configure WebClient

```java
@Configuration
public class WebClientConfiguration {

    @Bean
    public WebClient productWebClient(
            WebClient.Builder builder,
            @Value("${clients.product-service.base-url}") String baseUrl
    ) {
        return builder
                .baseUrl(baseUrl)
                .defaultHeader(
                        HttpHeaders.ACCEPT,
                        MediaType.APPLICATION_JSON_VALUE
                )
                .build();
    }
}
```

## 6.6 GET example

```java
@Service
@RequiredArgsConstructor
public class ProductGateway {

    private final WebClient productWebClient;

    public Mono<ProductResponse> getProduct(Long id) {
        return productWebClient.get()
                .uri("/api/products/{id}", id)
                .retrieve()
                .bodyToMono(ProductResponse.class);
    }
}
```

The pipeline runs when it is subscribed to, normally by the WebFlux runtime or another reactive operation.

## 6.7 POST example

```java
public Mono<ProductResponse> createProduct(
        CreateProductRequest request
) {
    return productWebClient.post()
            .uri("/api/products")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(request)
            .retrieve()
            .bodyToMono(ProductResponse.class);
}
```

## 6.8 Multiple-item response

```java
public Flux<ProductResponse> getAllProducts() {
    return productWebClient.get()
            .uri("/api/products")
            .retrieve()
            .bodyToFlux(ProductResponse.class);
}
```

To return one list:

```java
public Mono<List<ProductResponse>> getAllProductsAsList() {
    return productWebClient.get()
            .uri("/api/products")
            .retrieve()
            .bodyToFlux(ProductResponse.class)
            .collectList();
}
```

## 6.9 Error handling

```java
public Mono<ProductResponse> getProduct(Long id) {
    return productWebClient.get()
            .uri("/api/products/{id}", id)
            .retrieve()
            .onStatus(
                    status -> status.value() == 404,
                    response -> response.bodyToMono(ApiError.class)
                            .defaultIfEmpty(
                                    new ApiError(
                                            404,
                                            "Product not found",
                                            "/api/products/" + id
                                    )
                            )
                            .flatMap(error -> Mono.error(
                                    new ProductNotFoundException(
                                            error.message()
                                    )
                            ))
            )
            .onStatus(
                    HttpStatusCode::is5xxServerError,
                    response -> response.bodyToMono(String.class)
                            .defaultIfEmpty("Downstream failure")
                            .flatMap(body -> Mono.error(
                                    new ProductServiceException(body)
                            ))
            )
            .bodyToMono(ProductResponse.class);
}
```

## 6.10 `retrieve()` versus `exchangeToMono()`

Use `retrieve()` for normal body extraction:

```java
return webClient.get()
        .uri("/products/{id}", id)
        .retrieve()
        .bodyToMono(ProductResponse.class);
```

Use `exchangeToMono()` when handling depends heavily on status and headers:

```java
return webClient.get()
        .uri("/products/{id}", id)
        .exchangeToMono(response -> {
            if (response.statusCode().is2xxSuccessful()) {
                return response.bodyToMono(ProductResponse.class);
            }

            if (response.statusCode().value() == 404) {
                return Mono.empty();
            }

            return response.createException()
                    .flatMap(Mono::error);
        });
```

## 6.11 Combine independent calls concurrently

```java
Mono<ProductResponse> productMono =
        productClient.getProduct(productId);

Mono<CustomerResponse> customerMono =
        customerClient.getCustomer(customerId);

return Mono.zip(productMono, customerMono)
        .map(tuple -> new OrderSummary(
                tuple.getT1(),
                tuple.getT2()
        ));
```

This allows both remote calls to progress concurrently without manually managing threads.

## 6.12 Streaming example

```java
public Flux<PriceUpdate> streamPrices() {
    return webClient.get()
            .uri("/api/prices/stream")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .retrieve()
            .bodyToFlux(PriceUpdate.class);
}
```

This is a major reason to select WebClient.

## 6.13 Timeout configuration

Example with Reactor Netty:

```java
@Bean
public WebClient productWebClient(
        @Value("${clients.product-service.base-url}") String baseUrl
) {
    HttpClient httpClient = HttpClient.create()
            .option(
                    ChannelOption.CONNECT_TIMEOUT_MILLIS,
                    2000
            )
            .responseTimeout(Duration.ofSeconds(3))
            .doOnConnected(connection -> connection
                    .addHandlerLast(new ReadTimeoutHandler(3))
                    .addHandlerLast(new WriteTimeoutHandler(3))
            );

    return WebClient.builder()
            .baseUrl(baseUrl)
            .clientConnector(
                    new ReactorClientHttpConnector(httpClient)
            )
            .build();
}
```

Imports and exact APIs can differ by Spring and Reactor Netty version.

## 6.14 WebClient filter

```java
private ExchangeFilterFunction correlationIdFilter() {
    return (request, next) -> {
        String correlationId = MDC.get("correlationId");
        ClientRequest.Builder builder = ClientRequest.from(request);

        if (correlationId != null && !correlationId.isBlank()) {
            builder.header("X-Correlation-Id", correlationId);
        }

        return next.exchange(builder.build());
    };
}
```

```java
WebClient.builder()
        .baseUrl(baseUrl)
        .filter(correlationIdFilter())
        .build();
```

## 6.15 Understanding `.block()`

```java
ProductResponse product = webClient.get()
        .uri("/products/{id}", id)
        .retrieve()
        .bodyToMono(ProductResponse.class)
        .block();
```

`block()` waits synchronously and converts the reactive pipeline into blocking behavior.

It may be acceptable:

- At a carefully controlled imperative boundary
- In a command-line utility
- In tests
- In isolated non-reactive code

It is dangerous:

- Inside a reactive request thread
- Throughout a WebFlux application
- When it can block event-loop threads
- When every call is blocked and RestClient would be simpler

## 6.16 When to use WebClient

Use it when:

- Your application uses Spring WebFlux
- You require end-to-end non-blocking execution
- You consume streams or Server-Sent Events
- You need `Mono`/`Flux` composition
- You perform many concurrent I/O operations
- Backpressure matters

Avoid or reconsider it when:

- The application is simple Spring MVC CRUD
- Every call ends with `.block()`
- The team is not using reactive programming
- A synchronous API would be easier to maintain

## 6.17 Interview answer: What is WebClient?

> WebClient is Spring's reactive HTTP client. It provides a fluent API and returns `Mono` or `Flux`, supporting non-blocking request processing, reactive composition, and streaming. I use it for WebFlux applications, Server-Sent Events, high-concurrency I/O, or multiple asynchronous calls. I avoid `.block()` inside a reactive flow because it blocks the event-loop thread.

## 6.18 Interview answer: Can WebClient be used in Spring MVC?

> Yes, but the benefit depends on how the result is consumed. If every request is immediately blocked, RestClient is usually simpler. WebClient is valuable when the application preserves a reactive flow or requires streaming.

## 6.19 Interview answer: Mono versus Flux

> `Mono<T>` represents zero or one asynchronous value. `Flux<T>` represents zero to many values. A single-object API normally returns `Mono`, while a stream or multiple values can return `Flux`.

---

# 7. RestClient

## 7.1 Overview

`RestClient` is Spring Framework's modern synchronous HTTP client. It has a fluent API similar to WebClient, but it follows a blocking request-response model and returns normal Java values instead of Reactor types.

## 7.2 Advantages

- Modern fluent API
- Simple synchronous programming model
- No `Mono`, `Flux`, or `.block()` required
- Good for ordinary Spring MVC applications
- Supports status handling, interceptors, message conversion, and request factories
- Can reuse existing RestTemplate configuration during migration
- Native to Spring Framework

## 7.3 Disadvantages

- Blocking
- Not designed for reactive streaming
- More explicit code than a declarative interface
- Load balancing and advanced cloud features may need additional integration

## 7.4 Dependency

It is normally available through Spring MVC:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

RestClient was introduced in Spring Framework 6.1. Use a compatible Spring Boot version.

## 7.5 Configure RestClient

```java
@Configuration
public class RestClientConfiguration {

    @Bean
    public RestClient productRestClient(
            RestClient.Builder builder,
            @Value("${clients.product-service.base-url}") String baseUrl
    ) {
        return builder
                .baseUrl(baseUrl)
                .defaultHeader(
                        HttpHeaders.ACCEPT,
                        MediaType.APPLICATION_JSON_VALUE
                )
                .build();
    }
}
```

## 7.6 GET example

```java
@Service
@RequiredArgsConstructor
public class ProductGateway {

    private final RestClient productRestClient;

    public ProductResponse getProduct(Long id) {
        return productRestClient.get()
                .uri("/api/products/{id}", id)
                .retrieve()
                .body(ProductResponse.class);
    }
}
```

The method blocks until the response arrives or fails.

## 7.7 POST example

```java
public ProductResponse createProduct(
        CreateProductRequest request
) {
    return productRestClient.post()
            .uri("/api/products")
            .contentType(MediaType.APPLICATION_JSON)
            .body(request)
            .retrieve()
            .body(ProductResponse.class);
}
```

## 7.8 Generic list response

Use `ParameterizedTypeReference` because generic type information is erased at runtime.

```java
public List<ProductResponse> getProducts() {
    return productRestClient.get()
            .uri("/api/products")
            .retrieve()
            .body(
                    new ParameterizedTypeReference<
                            List<ProductResponse>
                    >() {
                    }
            );
}
```

## 7.9 Error handling

```java
public ProductResponse getProduct(Long id) {
    return productRestClient.get()
            .uri("/api/products/{id}", id)
            .retrieve()
            .onStatus(
                    status -> status.value() == 404,
                    (request, response) -> {
                        throw new ProductNotFoundException(
                                "Product not found: " + id
                        );
                    }
            )
            .onStatus(
                    HttpStatusCode::is5xxServerError,
                    (request, response) -> {
                        throw new ProductServiceException(
                                "Product Service is unavailable"
                        );
                    }
            )
            .body(ProductResponse.class);
}
```

## 7.10 Headers

```java
public ProductResponse getProduct(
        Long id,
        String correlationId
) {
    return productRestClient.get()
            .uri("/api/products/{id}", id)
            .header("X-Correlation-Id", correlationId)
            .retrieve()
            .body(ProductResponse.class);
}
```

For common headers, use an interceptor.

## 7.11 Interceptor

```java
@Bean
public RestClient productRestClient(
        RestClient.Builder builder,
        @Value("${clients.product-service.base-url}") String baseUrl
) {
    return builder
            .baseUrl(baseUrl)
            .requestInterceptor((request, body, execution) -> {
                String correlationId = MDC.get("correlationId");

                if (correlationId != null && !correlationId.isBlank()) {
                    request.getHeaders().add(
                            "X-Correlation-Id",
                            correlationId
                    );
                }

                return execution.execute(request, body);
            })
            .build();
}
```

## 7.12 `toEntity()`

Use this when status or response headers are needed.

```java
ResponseEntity<ProductResponse> response =
        productRestClient.get()
                .uri("/api/products/{id}", id)
                .retrieve()
                .toEntity(ProductResponse.class);

HttpStatusCode status = response.getStatusCode();
HttpHeaders headers = response.getHeaders();
ProductResponse body = response.getBody();
```

## 7.13 Timeout configuration

```java
@Bean
public RestClient productRestClient(
        RestClient.Builder builder
) {
    SimpleClientHttpRequestFactory requestFactory =
            new SimpleClientHttpRequestFactory();

    requestFactory.setConnectTimeout(Duration.ofSeconds(2));
    requestFactory.setReadTimeout(Duration.ofSeconds(3));

    return builder
            .baseUrl("http://localhost:8081")
            .requestFactory(requestFactory)
            .build();
}
```

For higher-traffic systems, use an HTTP transport with robust connection pooling.

## 7.14 Declarative HTTP interface with RestClient

Spring Framework supports declarative HTTP interfaces using `@HttpExchange`.

```java
@HttpExchange("/api/products")
public interface ProductHttpClient {

    @GetExchange("/{id}")
    ProductResponse getProduct(
            @PathVariable Long id
    );

    @PostExchange
    ProductResponse createProduct(
            @RequestBody CreateProductRequest request
    );
}
```

Create the proxy:

```java
@Bean
public ProductHttpClient productHttpClient(
        RestClient.Builder builder,
        @Value("${clients.product-service.base-url}") String baseUrl
) {
    RestClient restClient = builder
            .baseUrl(baseUrl)
            .build();

    RestClientAdapter adapter = RestClientAdapter.create(restClient);

    HttpServiceProxyFactory factory =
            HttpServiceProxyFactory
                    .builderFor(adapter)
                    .build();

    return factory.createClient(ProductHttpClient.class);
}
```

This provides a declarative style without Spring Cloud OpenFeign.

## 7.15 When to use RestClient

Use it when:

- The application uses Spring MVC
- Calls are synchronous
- You are building a new imperative application
- You prefer a fluent API
- You want to avoid unnecessary reactive types
- You are migrating from RestTemplate
- You need explicit request construction

Avoid or reconsider it when:

- The application is end-to-end reactive
- You require streaming or backpressure
- Your team has standardized on declarative OpenFeign clients
- Non-blocking I/O is a core requirement

## 7.16 Interview answer: What is RestClient?

> RestClient is Spring Framework's modern synchronous HTTP client. It provides a fluent API similar to WebClient, but it blocks and returns the response directly. I use it for new Spring MVC or imperative applications that need straightforward request-response integration.

## 7.17 Interview answer: RestClient versus RestTemplate

> Both are synchronous, but RestClient offers a modern fluent API and is preferred for new code. RestTemplate uses the older template-style API. RestClient can also reuse RestTemplate configuration, which helps during migration.

---

# 8. Detailed Comparison

| Feature | OpenFeign | WebClient | RestClient |
|---|---|---|---|
| Programming model | Declarative interface | Reactive fluent API | Synchronous fluent API |
| Blocking | Yes | No, when used reactively | Yes |
| Return type | DTO or `ResponseEntity` | `Mono<T>` or `Flux<T>` | DTO or `ResponseEntity` |
| Best fit | Declarative service clients | Reactive and streaming systems | Imperative Spring MVC systems |
| Main module | Spring Cloud | Spring WebFlux | Spring Framework |
| Boilerplate | Low | Medium | Medium |
| Streaming | Not the main use | Excellent | Not intended for reactive streaming |
| Learning curve | Low to medium | High | Low |
| Request visibility | Annotation-driven | Explicit | Explicit |
| Service discovery | Strong Spring Cloud integration | Additional configuration | Additional configuration |
| Reactive composition | No | Yes | No |
| Good for CRUD calls | Yes | Possible but often excessive | Yes |
| Proxy generated | Yes | No, unless using HTTP interfaces | No, unless using HTTP interfaces |

---

# 9. When to Use Which

## 9.1 Traditional Spring MVC application

Requirement:

```text
Order Service calls Product Service synchronously.
```

Preferred choice:

```text
RestClient
```

Choose OpenFeign when the organization already uses declarative clients and Spring Cloud integrations.

## 9.2 Reactive Spring WebFlux application

Requirement:

```text
Gateway calls multiple services concurrently and returns Mono<Response>.
```

Preferred choice:

```text
WebClient
```

Do not directly use blocking Feign or RestClient calls on the reactive event loop.

## 9.3 Streaming price updates

Preferred choice:

```text
WebClient
```

## 9.4 Many simple internal endpoints

Possible choice:

```text
OpenFeign
```

Alternative for new Spring Framework code:

```text
@HttpExchange + RestClient
```

## 9.5 Third-party API requiring explicit response handling

Use:

```text
RestClient for synchronous applications
WebClient for reactive applications
```

## 9.6 Existing RestTemplate project

Recommended migration:

```text
RestTemplate -> RestClient
```

Do not migrate to WebClient only for modernization when the application remains synchronous.

## 9.7 Easy rule

```text
Synchronous + explicit fluent API
    -> RestClient

Synchronous + declarative interface
    -> OpenFeign
    or @HttpExchange + RestClient

Reactive / non-blocking / streaming
    -> WebClient
```

---

# 10. Production Best Practices

## 10.1 Always configure timeouts

Configure:

- Connection timeout
- Read or response timeout
- Write timeout where required
- Overall operation timeout when appropriate

Without timeouts, one failing dependency can consume threads and connections until the whole service becomes unavailable.

## 10.2 Use connection pooling

Production clients should normally reuse connections.

Monitor:

- Active connections
- Idle connections
- Pending acquisition
- Maximum connections
- Connection lifetime

## 10.3 Retry carefully

Retry only when:

- The failure is transient
- The operation is safe or idempotent
- Retry count is limited
- Backoff and jitter are configured
- Total execution time remains bounded

Safer examples:

- GET
- HEAD
- Properly designed idempotent PUT

Dangerous examples:

- Payment POST
- Order creation POST
- Money transfer
- Any side-effecting operation without an idempotency key

## 10.4 Use circuit breakers

Circuit breakers:

- Fail fast during dependency failure
- Protect thread and connection pools
- Reduce cascading failure
- Give a downstream service time to recover

They do not replace timeouts, monitoring, or capacity planning.

## 10.5 Use bulkheads

Bulkheads isolate resources.

```text
Payment integration -> dedicated limited resources
Notification integration -> separate resources
```

A failed notification service should not consume resources required for payments.

## 10.6 Authentication

Common approaches:

- OAuth2 client credentials
- Bearer JWT
- API key
- Mutual TLS
- Basic authentication for limited legacy scenarios

Never:

- Hard-code secrets
- Commit credentials to Git
- Log Authorization headers
- Send secrets in query parameters

## 10.7 Propagate correlation IDs

```text
Client -> Order Service -> Payment Service
       X-Correlation-Id: abc-123
```

Use interceptors or filters instead of adding headers manually to every method.

## 10.8 Logging

Log:

- HTTP method
- Sanitized target URI
- Target service
- Response status
- Duration
- Correlation ID
- Retry count
- Exception category

Do not log:

- Tokens
- Passwords
- Card details
- Personal data without justification
- Full sensitive request or response bodies

## 10.9 Map technical exceptions to domain exceptions

Bad:

```java
throw feignException;
```

Better:

```java
throw new ProductNotFoundException(productId);
```

Examples:

- `ProductNotFoundException`
- `PaymentServiceUnavailableException`
- `InventoryConflictException`
- `ExternalApiAuthenticationException`

## 10.10 Keep DTOs separate from entities

Bad:

```java
ProductEntity product = productClient.getProduct(id);
```

Better:

```java
ProductResponse product = productClient.getProduct(id);
```

Reasons:

- Prevent persistence details from leaking
- Avoid lazy-loading problems
- Keep bounded contexts independent
- Allow API evolution

## 10.11 Test real HTTP behavior

Test:

- Success
- 400 validation errors
- 401/403 authentication failures
- 404 not found
- 409 conflict
- 500/503 failure
- Timeout
- Malformed JSON
- Empty body
- Required headers
- Retry behavior

Use an HTTP stub server rather than over-mocking fluent method chains.

## 10.12 Reuse configured clients

Bad:

```java
public ProductResponse getProduct(Long id) {
    RestClient client = RestClient.create("http://localhost:8081");
    return client.get()
            .uri("/products/{id}", id)
            .retrieve()
            .body(ProductResponse.class);
}
```

Better:

```java
@Bean
RestClient productRestClient(RestClient.Builder builder) {
    return builder
            .baseUrl(productBaseUrl)
            .build();
}
```

## 10.13 Do not mix blocking and reactive code carelessly

Bad:

```java
public Mono<OrderResponse> getOrder(Long id) {
    ProductResponse product = restClient.get()
            .uri("/products/{id}", id)
            .retrieve()
            .body(ProductResponse.class);

    return Mono.just(new OrderResponse(id, product));
}
```

The blocking call happens before `Mono.just()` and can block the event-loop thread.

For true reactive integration, use WebClient end to end.

---

# 11. Common Interview Questions and Answers

## 11.1 Difference between OpenFeign, WebClient, and RestClient

> OpenFeign is a declarative, interface-based blocking client commonly used for synchronous microservice calls. WebClient is reactive and non-blocking, so it is suitable for WebFlux, streaming, and high-concurrency I/O. RestClient is a modern fluent synchronous client for imperative Spring MVC applications. I select the client according to the application's execution model, not only according to syntax.

## 11.2 Which client for a new Spring MVC application?

> I normally choose RestClient because it is synchronous, modern, and simple. I choose OpenFeign when the organization already standardizes declarative clients and Spring Cloud integrations. I do not use WebClient only to call `.block()` on every request.

## 11.3 Which client for microservices?

> Microservices alone do not determine the client. For synchronous declarative calls, OpenFeign is convenient. For explicit synchronous calls, RestClient is suitable. For reactive or streaming flows, WebClient is the correct choice.

## 11.4 Why is WebClient non-blocking?

> WebClient can use a non-blocking HTTP engine and exposes results as Reactor publishers. Instead of holding one thread while waiting for I/O, response events continue the pipeline. The benefit is preserved only when the rest of the flow also avoids blocking.

## 11.5 Can RestClient make asynchronous calls?

> RestClient itself is synchronous. Running it on another thread only moves the blocking work; it does not make the underlying I/O reactive. For genuine reactive composition, I use WebClient.

## 11.6 Why not always use OpenFeign?

> Less code is not the only requirement. OpenFeign is blocking and proxy-driven. WebClient is better for reactive or streaming use cases, while RestClient gives explicit synchronous control without requiring Spring Cloud. The architecture should decide.

## 11.7 How do you handle downstream failures?

> I configure connection and response timeouts, map HTTP errors to domain exceptions, retry only transient and safe operations, use circuit breakers for unstable dependencies, propagate correlation IDs, and monitor latency and error rates. I avoid fake fallback data that looks like valid business data.

## 11.8 How do you secure outbound calls?

> I use OAuth2 client credentials, JWT propagation, API keys, or mutual TLS depending on the system. Credentials are added through interceptors or filters, stored in a secure secret manager, and excluded from logs.

## 11.9 What is wrong with `.block()` in WebFlux?

> `.block()` waits synchronously and can block an event-loop thread. Under load, this can cause thread starvation and reduce throughput. I return `Mono` or `Flux` and compose operations instead of blocking.

## 11.10 What metrics do you monitor?

> Request rate, success rate, error rate by status, latency percentiles, timeout count, retry count, circuit-breaker state, connection-pool saturation, and downstream availability. I avoid high-cardinality metric tags such as raw user IDs.

---

# 12. Scenario-Based Answers

## 12.1 Payment service

Recommended considerations:

- RestClient or OpenFeign for synchronous applications
- Strict timeouts
- Idempotency key
- No blind POST retry
- Audit trail
- Status reconciliation
- Circuit breaker

Interview answer:

> For payment creation, I do not blindly retry a failed POST because the payment may have succeeded even when the response was lost. I use an idempotency key, bounded timeouts, domain-specific error mapping, and a payment-status or reconciliation mechanism.

## 12.2 Dashboard requiring three independent services

In WebFlux:

```java
Mono.zip(
        customerClient.getCustomer(customerId),
        orderClient.getOrders(customerId),
        recommendationClient.getRecommendations(customerId)
)
```

Interview answer:

> Since the calls are independent, WebClient can start them concurrently and combine the results using `Mono.zip`, provided the entire flow remains non-blocking.

## 12.3 Internal product-service call

Interview answer:

> In a normal Spring MVC application, I use RestClient for explicit control or OpenFeign when the platform standard is declarative clients. In both cases I configure timeouts, error mapping, tracing, authentication, and resilience.

## 12.4 Large streaming file

Interview answer:

> I consider WebClient because it can process the response as a stream instead of loading the entire file into memory. I also verify buffer limits, storage behavior, cancellation, and backpressure.

---

# 13. Common Mistakes

## 13.1 No timeout

Impact:

- Thread exhaustion
- Connection exhaustion
- Queue growth
- Cascading failure

## 13.2 Retrying every error

Do not retry:

- 400 validation error
- 401 authentication error without corrected credentials
- 403 authorization error
- 404 in normal cases
- Non-idempotent operations without protection

## 13.3 Logging complete request and response bodies

This can expose secrets and personal information.

## 13.4 Using WebClient but blocking everywhere

If every call ends in `.block()` in an imperative application, use RestClient unless WebClient provides a special required capability.

## 13.5 Returning fake data on error

Bad:

```java
.onErrorReturn(
        new ProductResponse(
                0L,
                "Unknown",
                BigDecimal.ZERO
        )
)
```

This can silently turn infrastructure failure into incorrect business behavior.

## 13.6 Leaking client-library exceptions

Do not expose exceptions such as `FeignException$ServiceUnavailable` directly from controllers. Convert them to application-level errors.

---

# 14. Quick Revision Notes

## 14.1 One-minute revision

```text
OpenFeign
- Declarative interface
- Blocking
- Spring Cloud
- Uses @FeignClient
- Best for concise synchronous service clients
- Configure timeout, interceptor, decoder, and resilience

WebClient
- Reactive and non-blocking
- Returns Mono or Flux
- Best for WebFlux, streaming, and high concurrency
- Do not block event-loop threads
- Uses filters and onStatus

RestClient
- Synchronous and blocking
- Modern fluent API
- Best for new Spring MVC applications
- Returns normal Java objects
- Uses interceptors and onStatus
```

## 14.2 Selection cheat sheet

```text
Is the application reactive?
    |
    +-- Yes -> WebClient
    |
    +-- No
         |
         +-- Want explicit fluent calls?
         |       -> RestClient
         |
         +-- Want declarative interfaces?
                 -> OpenFeign
                 or @HttpExchange + RestClient
```

## 14.3 Interview one-liners

- OpenFeign is declarative and blocking.
- WebClient is reactive and non-blocking when used end to end.
- RestClient is modern and synchronous.
- Do not choose WebClient only because it is newer.
- Do not call `.block()` inside a reactive pipeline.
- Configure timeouts for every remote service.
- Retry only transient and safe operations.
- POST retries require idempotency protection.
- Circuit breakers reduce cascading failure.
- DTOs should be separate from JPA entities.
- Interceptors and filters handle common headers.
- Convert technical exceptions into domain exceptions.
- Use WebClient for streaming.
- Use RestClient for normal synchronous Spring MVC calls.
- Use OpenFeign for concise declarative service clients.
- OpenFeign is feature-complete, not unusable.
- `@HttpExchange` is Spring Framework's declarative HTTP-interface option.

## 14.4 Five-minute interview answer

> In Spring Boot, I choose the HTTP client according to the execution model. For a traditional Spring MVC application, I generally use RestClient because it is synchronous, fluent, and simple. If the team uses Spring Cloud and prefers declarative interfaces, OpenFeign is convenient because Spring generates a proxy from `@FeignClient` interfaces. For WebFlux, streaming, or high-concurrency non-blocking I/O, I use WebClient and preserve the flow using `Mono` and `Flux`. Regardless of the client, I configure timeouts, authentication, correlation IDs, domain-level error mapping, metrics, connection pooling, and resilience. I retry only transient and idempotent operations and avoid blocking reactive event-loop threads.

---

# 15. Practice Questions

## Basic

1. What is an HTTP client?
2. What is blocking I/O?
3. What is non-blocking I/O?
4. What is OpenFeign?
5. What is WebClient?
6. What is RestClient?
7. What is `Mono<T>`?
8. What is `Flux<T>`?
9. What is a declarative client?
10. What is an interceptor?

## Intermediate

1. Why does OpenFeign create a proxy?
2. Why should WebClient not be blocked in WebFlux?
3. How do you map a 404 response?
4. How do you propagate correlation IDs?
5. How do you configure timeouts?
6. What is client-side load balancing?
7. What is a circuit breaker?
8. When is retry safe?
9. Why should entities not be used as remote DTOs?
10. How do you test an HTTP client?

## Advanced

1. How can blocking calls exhaust a thread pool?
2. How does event-loop I/O differ from thread-per-request?
3. How do you handle a payment timeout with an unknown result?
4. How do connection pools affect throughput?
5. How do you prevent retry storms?
6. What is backpressure?
7. How do you isolate a slow dependency?
8. How do you avoid high-cardinality metrics?
9. How do you migrate RestTemplate to RestClient?
10. How do you create a client using `@HttpExchange`?

---

# 16. Final Recommendation

```text
Spring MVC / imperative application
    -> RestClient

Spring MVC with declarative Spring Cloud standard
    -> OpenFeign

Spring WebFlux / streaming / non-blocking application
    -> WebClient
```

No client is universally best. Select the client that matches the execution model, operational requirements, and team architecture.

---

# 17. Official References

- Spring Framework REST Clients:  
  https://docs.spring.io/spring-framework/reference/integration/rest-clients.html

- Spring Framework WebClient:  
  https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html

- Spring Cloud OpenFeign:  
  https://docs.spring.io/spring-cloud-openfeign/reference/index.html

- Spring Cloud OpenFeign Features:  
  https://docs.spring.io/spring-cloud-openfeign/reference/spring-cloud-openfeign.html
