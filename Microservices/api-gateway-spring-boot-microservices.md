# API Gateway in Spring Boot Microservices — Quick Revision Notes

## 1. Why do we need an API Gateway?

In a microservices architecture, we may have many services:

```text
Frontend
   |
   |--> Event Service
   |--> Venue Service
   |--> Show Service
   |--> Seat Service
   |--> Booking Service
   |--> Payment Service
```

Without an API Gateway, the frontend needs to know the URL and port of every microservice.

Example:

```text
Event Service   -> localhost:8081
Venue Service   -> localhost:8082
Show Service    -> localhost:8083
Booking Service -> localhost:8085
```

This creates unnecessary coupling between the client and internal microservices.

---

## 2. What is an API Gateway?

An **API Gateway is the single entry point for client requests in a microservices architecture**.

Instead of calling each microservice directly, the client calls the Gateway.

```text
                       +--> Event Service
                       |
                       +--> Venue Service
                       |
Client --> API Gateway +--> Show Service
                       |
                       +--> Booking Service
                       |
                       +--> Payment Service
```

The Gateway decides which microservice should receive the request.

---

## 3. Simple Real-Life Example

Think of an API Gateway like the **reception desk in an office**.

You enter the office and say:

```text
I want to meet someone from Accounts.
```

You do not need to know the exact desk or floor.

The receptionist directs you to the correct department.

Similarly:

```text
Client
   |
   | GET /events/101
   v
API Gateway
   |
   | Route to Event Service
   v
Event Service
```

---

## 4. What is Routing?

**Routing means deciding which microservice should receive a request.**

Example routing rules:

```text
/events/**    -> Event Service
/venues/**    -> Venue Service
/shows/**     -> Show Service
/seats/**     -> Seat Service
/bookings/**  -> Booking Service
/payments/**  -> Payment Service
```

If the client sends:

```http
GET /events/101
```

the Gateway routes it to Event Service.

If the client sends:

```http
POST /bookings
```

the Gateway routes it to Booking Service.

---

## 5. Does API Gateway contain business logic?

Generally, **no**.

The Gateway should not contain logic such as:

```text
Check whether seat A1 exists
Lock seat A1
Calculate ticket price
Create booking
Process payment
```

That business logic belongs inside the appropriate microservices.

The Gateway mainly handles cross-cutting concerns around incoming and outgoing requests.

---

## 6. Common Responsibilities of an API Gateway

An API Gateway can handle:

- Request routing
- Authentication
- Basic authorization checks
- Rate limiting
- Logging
- Distributed tracing
- CORS
- Header modification
- Request/response filtering
- Integration with service discovery
- Load balancing

---

## 7. Authentication Example

Suppose the client sends:

```http
POST /bookings
Authorization: Bearer <JWT_TOKEN>
```

The request first reaches the Gateway.

```text
Client
   |
   | JWT Token
   v
API Gateway
   |
   |-- Invalid token --> 401 Unauthorized
   |
   |-- Valid token
           |
           v
     Booking Service
```

The Gateway can perform an initial authentication check before forwarding the request.

Important:

```text
Gateway security does NOT always mean microservices need no security.
Sensitive services should still protect important operations appropriately.
```

---

## 8. API Gateway vs Service-to-Service Communication

This distinction is very important.

### Client to Microservice

```text
Frontend
   |
   v
API Gateway
   |
   v
Show Service
```

### Microservice to Microservice

Suppose Show Service wants to verify whether an event exists.

```text
Show Service
   |
   v
Event Service
```

For this internal communication, we commonly use:

```text
Feign Client
RestClient
WebClient
```

So remember:

```text
API Gateway
= Client -> Microservice

Feign / RestClient / WebClient
= Microservice -> Microservice
```

---

## 9. API Gateway vs Orchestrator

Do not confuse these concepts.

### API Gateway

Main responsibility:

```text
Where should this incoming request go?
```

Example:

```text
Client
   |
   v
Gateway
   |
   v
Booking Service
```

### Orchestrator

Main responsibility:

```text
Which business steps should happen and in what order?
```

Example:

```text
Booking Orchestrator
       |
       |--> Lock Seats
       |
       |--> Create Payment
       |
       |--> Confirm Booking
       |
       |--> Send Notification
```

Easy revision:

```text
Gateway      = Request traffic management
Orchestrator = Business workflow coordination
```

---

## 10. Spring Cloud Gateway

In Spring Boot microservices, a common API Gateway solution is:

```text
Spring Cloud Gateway
```

Example architecture:

```text
                     Client
                       |
                       v
                +-------------+
                | API Gateway |
                |    :8080    |
                +------+------+
                       |
          +------------+------------+
          |            |            |
          v            v            v
    Event Service  Venue Service  Show Service
       :8081          :8082          :8083
```

---

# 11. Three Important Spring Cloud Gateway Concepts

The three most important concepts are:

```text
1. Route
2. Predicate
3. Filter
```

---

## 11.1 Route

A **Route defines where a request should go**.

Example:

```text
/events/** -> Event Service
```

Meaning:

```text
Requests starting with /events
should be forwarded to Event Service.
```

---

## 11.2 Predicate

A **Predicate decides whether a request matches a route**.

Example:

```text
Path=/events/**
```

Request:

```text
/events/101
```

Result:

```text
MATCH
```

Request:

```text
/bookings/101
```

Result:

```text
NO MATCH
```

So:

```text
Predicate = Should this route be selected?
```

---

## 11.3 Filter

A **Filter processes or modifies the request or response**.

Example flow:

```text
Request
   |
   v
API Gateway
   |
   v
Authentication Filter
   |
   v
Logging Filter
   |
   v
Add Header
   |
   v
Event Service
```

Filters can be used for:

```text
JWT validation
Logging
Headers
CORS
Rate limiting
Request modification
Response modification
```

---

# 12. Route vs Predicate vs Filter

Quick revision:

```text
Route
= Where should the request go?

Predicate
= Does this request match this route?

Filter
= What should happen to the request or response?
```

Example:

```text
GET /events/101
```

Gateway processing:

```text
Request
   |
   v
Predicate
Path=/events/**
   |
   | Match
   v
Authentication Filter
   |
   v
Logging Filter
   |
   v
Route
   |
   v
Event Service
```

---

# 13. Full Request Flow Example

Client sends:

```http
GET /events/101
Authorization: Bearer XYZ
```

Flow:

```text
                 GET /events/101
                        |
                        v
                +---------------+
                |  API Gateway  |
                +-------+-------+
                        |
                    Predicate
                        |
                Path=/events/**
                        |
                      Match
                        |
                        v
                   JWT Filter
                        |
                   Token valid
                        |
                        v
                     Routing
                        |
                        v
                +---------------+
                | Event Service |
                +-------+-------+
                        |
                        v
                    Database
```

Response:

```json
{
  "id": "101",
  "title": "Avengers"
}
```

Response flow:

```text
Event Service
     |
     v
API Gateway
     |
     v
Frontend
```

---

# 14. Multiple Instances and Load Balancing

Suppose Show Service has multiple running instances:

```text
Show Service Instance 1
Show Service Instance 2
Show Service Instance 3
```

Conceptually:

```text
                    +--> Show Service #1
                    |
Client --> Gateway -+--> Show Service #2
                    |
                    +--> Show Service #3
```

With service discovery and load balancing, requests can be distributed among healthy instances.

Common related concepts:

```text
Service Discovery
Eureka
Load Balancing
```

---

# 15. ShowSeat Example Architecture

For a ShowSeat-style project:

```text
                         Client
                           |
                           v
                    +-------------+
                    | API Gateway |
                    +------+------+
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
 Event Service        Venue Service       Show Service
                                               |
                                               v
                                           Seat Service
                                               |
                                               v
                                         Booking Service
                                               |
                                               v
                                         Payment Service
```

Example Gateway routes:

```text
/events/**    -> Event Service
/venues/**    -> Venue Service
/shows/**     -> Show Service
/seats/**     -> Seat Service
/bookings/**  -> Booking Service
/payments/**  -> Payment Service
```

---

# 16. Interview Answer

A simple human-like interview answer:

> API Gateway acts as a single entry point for clients in a microservices architecture. Instead of exposing every microservice directly to the client, requests first come to the Gateway. The Gateway routes them to the appropriate service based on rules such as the request path. It can also handle common concerns like authentication, logging, CORS, rate limiting, header modification, and integration with service discovery and load balancing. In Spring-based applications, Spring Cloud Gateway is commonly used for this purpose.

Example:

> If `/events/**` comes to the Gateway, it can route the request to Event Service, and if `/bookings/**` comes, it can route it to Booking Service. The frontend only needs to know the Gateway URL.

---

# 17. Important Interview Differences

## API Gateway vs Feign Client

```text
API Gateway
Client -> Microservice

Feign Client
Microservice -> Microservice
```

## API Gateway vs Orchestrator

```text
API Gateway
Routes incoming requests.

Orchestrator
Coordinates multiple business operations.
```

## Gateway vs Business Service

```text
Gateway
Routing, security, filters, rate limiting, CORS.

Business Service
Actual business logic.
```

---

# 18. Learning Order

Recommended order:

```text
API Gateway Basics
        |
        v
Route
        |
        v
Predicate
        |
        v
Filter
        |
        v
Spring Cloud Gateway Setup
        |
        v
Connect Event Service
        |
        v
Connect Show Service
        |
        v
Service Discovery
        |
        v
Load Balancing
        |
        v
JWT Authentication
        |
        v
CORS
        |
        v
Rate Limiting
        |
        v
Circuit Breaker / Resilience
        |
        v
Interview Scenarios
```

---

# 19. One-Minute Revision

```text
API Gateway
= Single entry point for clients.

Routing
= Send request to correct microservice.

Route
= Defines destination.

Predicate
= Checks whether request matches a route.

Filter
= Processes/modifies request or response.

Gateway
= Client -> Microservice.

Feign / RestClient / WebClient
= Microservice -> Microservice.

Gateway
!= Orchestrator.

Spring solution
= Spring Cloud Gateway.
```
