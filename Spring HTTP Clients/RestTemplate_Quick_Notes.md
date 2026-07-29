# RestTemplate

## 1. Overview

RestTemplate is Spring's synchronous (blocking) HTTP client used to
communicate with REST APIs.

It provides built-in methods to perform GET, POST, PUT, DELETE, and
other HTTP requests without manually creating HTTP connections.

Although RestTemplate is still widely used in existing enterprise
applications, Spring recommends using **RestClient** for new synchronous
applications and **WebClient** for reactive applications.

------------------------------------------------------------------------

## 2. Why It Matters

### Where it is used

-   Legacy Spring Boot applications
-   Existing enterprise projects
-   Calling external REST APIs
-   Microservice-to-microservice communication

### Why you should learn it

-   Many companies still use it in production.
-   Interviewers often ask about it.
-   It helps you understand migration to RestClient.

### What problem it solves

It simplifies:

-   Sending HTTP requests
-   Parsing JSON responses
-   Handling HTTP methods
-   Managing request/response objects

------------------------------------------------------------------------

## 3. Key Concepts

### Blocking HTTP Client

RestTemplate is synchronous.

The current thread waits until the server sends a response.

``` java
Product product =
        restTemplate.getForObject(
                "/products/1",
                Product.class
        );
```

### Template Method Pattern

RestTemplate follows the **Template Method Design Pattern**.

Spring provides predefined methods like:

-   `getForObject()`
-   `getForEntity()`
-   `postForObject()`
-   `exchange()`

instead of manually handling HTTP connections.

### Common Methods

-   `getForObject()`
-   `getForEntity()`
-   `postForObject()`
-   `put()`
-   `delete()`
-   `exchange()`

### Configuration

``` java
@Bean
RestTemplate restTemplate() {
    return new RestTemplate();
}
```

### Error Handling

-   `ResponseErrorHandler`

### Interceptors

-   Authentication
-   Logging
-   Correlation IDs
-   Common headers

------------------------------------------------------------------------

## 4. Syntax

### GET

``` java
Product product = restTemplate.getForObject(url, Product.class);
```

### POST

``` java
Product product = restTemplate.postForObject(url, request, Product.class);
```

### PUT

``` java
restTemplate.put(url, request);
```

### DELETE

``` java
restTemplate.delete(url);
```

### exchange()

``` java
ResponseEntity<Product> response =
        restTemplate.exchange(
                url,
                HttpMethod.GET,
                entity,
                Product.class
        );
```

------------------------------------------------------------------------

## 5. Advantages

-   Easy to learn
-   Simple API
-   Blocking programming model
-   Stable
-   Used in many legacy applications

------------------------------------------------------------------------

## 6. Limitations

-   Blocking
-   No reactive support
-   No streaming support
-   Deprecated for new development

------------------------------------------------------------------------

## 7. RestTemplate vs RestClient

  RestTemplate       RestClient
  ------------------ -------------
  Older API          Modern API
  Blocking           Blocking
  Template style     Fluent API
  Maintenance mode   Recommended

------------------------------------------------------------------------

## 8. RestTemplate vs WebClient

  RestTemplate     WebClient
  ---------------- -------------------
  Blocking         Non-blocking
  Spring MVC       Spring WebFlux
  Returns Object   Returns Mono/Flux
  Easier           Reactive

------------------------------------------------------------------------

## 9. When Should You Use It?

Use RestTemplate when maintaining legacy applications.

For new projects use:

-   RestClient (Blocking)
-   WebClient (Reactive)

------------------------------------------------------------------------

## 10. Interview Questions

### What is RestTemplate?

> RestTemplate is Spring's synchronous HTTP client used for calling REST
> APIs.

### Is RestTemplate deprecated?

> It is in maintenance mode. Spring recommends RestClient or WebClient
> for new development.

### Difference between RestTemplate and WebClient?

-   RestTemplate: Blocking
-   WebClient: Non-blocking

------------------------------------------------------------------------

## 11. Quick Revision

``` text
✓ Blocking
✓ Synchronous
✓ Template Method Pattern
✓ Legacy HTTP Client

Methods:
getForObject()
getForEntity()
postForObject()
put()
delete()
exchange()
```

------------------------------------------------------------------------

## 12. Summary

-   Blocking HTTP client
-   Used in legacy Spring Boot applications
-   Learn the concepts for interviews
-   Prefer RestClient for new projects
