# Stateless vs Stateful APIs — Quick Revision Notes

## 1. Core Idea

### Stateless

A **stateless system/API does not remember client-specific information between requests**.

Every request should contain all the information required to process it.

### Stateful

A **stateful system/API remembers some client-specific information between requests**.

Future requests can depend on state that the server stored earlier.

---

## 2. Easy Memory Trick

### Stateless

> Every request says: **"Here is everything you need."**

### Stateful

> Every request says: **"Here is my ID. You already know the rest."**

---

# 3. Stateless Request / API

A stateless API treats every request independently.

Example:

```http
GET /api/orders HTTP/1.1
Host: example.com
Authorization: Bearer <JWT_TOKEN>
```

Another request:

```http
GET /api/profile HTTP/1.1
Host: example.com
Authorization: Bearer <JWT_TOKEN>
```

The server validates the token again for each request.

### Flow

```text
Request 1 + JWT
      |
      v
Validate JWT
      |
      v
Process request
      |
      v
Response


Request 2 + JWT
      |
      v
Validate JWT again
      |
      v
Process request
      |
      v
Response
```

The server does not depend on a previously stored login session.

---

# 4. Stateful Request / API

A stateful API keeps information about the client on the server.

Example:

```text
User logs in
    |
    v
Server creates session
    |
    v
Session ID = ABC123
    |
    v
Server stores:
ABC123 -> User information
```

The next request sends the session ID:

```http
GET /api/profile HTTP/1.1
Host: example.com
Cookie: JSESSIONID=ABC123
```

The server uses `ABC123` to find previously stored information.

```text
ABC123
   |
   v
Session Store
   |
   v
username = guru
userId   = 101
role     = USER
```

---

# 5. Important Terminology

Technically, we usually say:

- **Stateless API/application**
- **Stateful API/application**

rather than saying that one individual HTTP request itself is stateless or stateful.

The meaning is about whether the server relies on stored client state between requests.

---

# 6. Designing a Stateful API with HttpSession

Example Spring Boot controller:

```java
@RestController
public class LoginController {

    @PostMapping("/login")
    public String login(
            @RequestParam String username,
            HttpSession session) {

        session.setAttribute("username", username);

        return "Login successful";
    }
}
```

Important line:

```java
session.setAttribute("username", username);
```

The server stores information in the HTTP session.

---

# 7. Why Don't We Return the Session ID Manually?

When `HttpSession` is created, the servlet container usually sends the session ID automatically as a cookie.

The controller returns:

```java
return "Login successful";
```

But the HTTP response can look like:

```http
HTTP/1.1 200 OK
Set-Cookie: JSESSIONID=9F8A72B1C4D5E6F7; Path=/; HttpOnly

Login successful
```

There are two separate parts:

```text
Response Header:
Set-Cookie: JSESSIONID=9F8A72B1C4D5E6F7

Response Body:
Login successful
```

The browser stores the cookie automatically.

---

# 8. Subsequent Request Format

After login, the browser/client sends the session ID in the Cookie header.

```http
GET /profile HTTP/1.1
Host: localhost:8080
Cookie: JSESSIONID=9F8A72B1C4D5E6F7
```

Notice:

The complete `HttpSession` object is **not** sent by the client.

Only the session ID is sent.

---

# 9. Retrieving Data from HttpSession

```java
@GetMapping("/profile")
public String profile(HttpSession session) {

    String username =
            (String) session.getAttribute("username");

    return "Welcome " + username;
}
```

Flow:

```text
Client Request
Cookie: JSESSIONID=ABC123
          |
          v
Server reads Session ID
          |
          v
Find session ABC123
          |
          v
Read username
          |
          v
Controller
```

---

# 10. What Can HttpSession Store?

Example:

```java
session.setAttribute("username", "guru");
session.setAttribute("userId", 101L);
session.setAttribute("role", "ADMIN");
```

Retrieve:

```java
String username =
        (String) session.getAttribute("username");

Long userId =
        (Long) session.getAttribute("userId");

String role =
        (String) session.getAttribute("role");
```

Conceptually:

```text
Session ID:
ABC123

Session Data:
{
    username = "guru",
    userId   = 101,
    role     = "ADMIN"
}
```

---

# 11. Spring Security — Stateless Configuration

For JWT-based REST APIs:

```java
http
    .sessionManagement(session ->
        session.sessionCreationPolicy(
            SessionCreationPolicy.STATELESS
        )
    );
```

Meaning:

Spring Security should not rely on an HTTP session to remember authentication.

Typical flow:

```text
Request
   |
   v
JWT Token
   |
   v
JWT Filter
   |
   v
Validate Token
   |
   v
Create SecurityContext
   |
   v
Controller
```

For the next request, the JWT is validated again.

---

# 12. Spring Security — Stateful Configuration

For session-based authentication:

```java
http
    .sessionManagement(session ->
        session.sessionCreationPolicy(
            SessionCreationPolicy.IF_REQUIRED
        )
    );
```

Typical flow:

```text
Login
  |
  v
Authenticate User
  |
  v
Create Session
  |
  v
JSESSIONID sent to client
  |
  v
Future request sends JSESSIONID
```

---

# 13. Stateless vs Stateful Authentication

## Stateless Authentication

Common example:

```text
JWT / OAuth Access Token
```

Request:

```http
Authorization: Bearer <JWT>
```

The server does not need a stored login session.

---

## Stateful Authentication

Common example:

```text
HttpSession / JSESSIONID
```

Request:

```http
Cookie: JSESSIONID=ABC123
```

The server uses the session ID to retrieve stored authentication/session information.

---

# 14. Comparison Table

| Stateless API | Stateful API |
|---|---|
| Server does not keep client session state | Server keeps client/session state |
| Each request is independent | Future requests can depend on earlier state |
| JWT is commonly used | Session/JSESSIONID is commonly used |
| Easier horizontal scaling | Needs session management while scaling |
| Any server instance can process request | Server must access correct/shared session |
| Common in REST APIs | Common in traditional web apps |
| Common in microservices | Often used with server-side sessions |

---

# 15. Why Stateless Is Easier to Scale

Imagine three instances:

```text
              Load Balancer
             /      |      \
        Server 1 Server 2 Server 3
```

With a stateless API:

```text
Request + JWT
```

can go to any instance.

Every server has enough information to process the request.

---

# 16. Problem with Stateful APIs in Multiple Instances

Suppose:

```text
Login
  |
  v
Server 1
  |
  v
ABC123 -> Gururaj
```

The next request may go to:

```text
Server 2
```

Server 2 might not have session `ABC123`.

So this can fail unless session sharing is handled.

---

# 17. Shared Session Store

A common design is:

```text
             Load Balancer
             /          \
        Server 1      Server 2
             \          /
                Redis
                  |
                  v
         Shared Sessions
```

Example:

```text
ABC123 -> Gururaj
XYZ456 -> Rahul
```

Both server instances can retrieve the same session.

---

# 18. When to Use Stateless APIs

Prefer stateless APIs for:

- REST APIs
- Microservices
- Mobile backends
- Public APIs
- JWT authentication
- OAuth-based systems
- Horizontally scalable applications
- Cloud-native applications

Typical microservice example:

```text
API Gateway
    |
    +---- User Service
    |
    +---- Order Service
    |
    +---- Payment Service
```

Request:

```http
Authorization: Bearer <JWT>
```

Any service instance can validate the request.

---

# 19. When to Use Stateful APIs

Use stateful behavior when the server genuinely needs to maintain information across multiple requests.

Examples:

- Session-based authentication
- Traditional server-rendered web applications
- Multi-step forms/workflows
- Some shopping-cart implementations
- Admin/internal applications
- Legacy applications
- Conversational/session workflows

Example multi-step form:

```text
Step 1
Personal Information
      |
      v
Stored in Session

Step 2
Address Information
      |
      v
Stored in Session

Step 3
Payment
      |
      v
Use previous session data
```

---

# 20. Very Important: Stateless Does NOT Mean No Database

A stateless microservice can still store:

- Users
- Orders
- Payments
- Bookings
- Products
- Seats
- Transactions

in a database.

For example:

```text
Stateless Booking Service
       |
       v
PostgreSQL
       |
       v
Bookings Table
```

The database contains **business data**.

Stateless means the server is not depending on temporary client/session state from previous HTTP requests.

---

# 21. Real-World Example

## Stateless Booking API

```http
POST /bookings
Authorization: Bearer <JWT>
Content-Type: application/json

{
    "showId": 101,
    "seatIds": [1, 2]
}
```

The request contains what is necessary.

The server can process it without remembering a previous HTTP request.

---

## Stateful Booking Flow

First request:

```http
POST /booking/start
```

Server creates:

```text
Session ABC123

selectedShow = 101
```

Next request:

```http
POST /booking/select-seats
Cookie: JSESSIONID=ABC123
```

Server updates:

```text
Session ABC123

selectedShow  = 101
selectedSeats = [A1, A2]
```

Next request:

```http
POST /booking/payment
Cookie: JSESSIONID=ABC123
```

This request depends on previously stored session information.

That makes the workflow stateful.

---

# 22. Common Interview Question

## Q: What is stateless?

**Answer:**

> Stateless means the server does not maintain client-specific session information between requests. Every request contains all the information required to process it. REST APIs using JWT commonly follow a stateless approach.

---

# 23. Common Interview Question

## Q: What is stateful?

**Answer:**

> Stateful means the server maintains information about the client between multiple requests. Future requests can use previously stored information. Session-based authentication using HttpSession and JSESSIONID is a common example.

---

# 24. Interview Question

## Q: What is the difference between stateless and stateful APIs?

**Answer:**

> In a stateless API, every request is independent and contains everything required for processing. The server does not maintain client session state between requests. In a stateful API, the server stores client-specific information, such as an HTTP session, and future requests can depend on that information.

---

# 25. Interview Question

## Q: How do you design a stateful API?

**Answer:**

> To design a stateful API, I can maintain client-specific state on the server using HTTP sessions. After login, the server creates a session and sends a session identifier such as JSESSIONID to the client. The client sends that ID in subsequent requests, and the server retrieves the stored session information. In distributed applications, sessions can be stored centrally in something like Redis so that multiple instances can access them.

---

# 26. Interview Question

## Q: Does the server return the JSESSIONID in the response body?

**Answer:**

> Normally no. The servlet container sends it automatically in the HTTP response header using `Set-Cookie`. The browser stores that cookie and sends it in future requests using the `Cookie` header.

Example response:

```http
Set-Cookie: JSESSIONID=ABC123
```

Next request:

```http
Cookie: JSESSIONID=ABC123
```

---

# 27. Interview Question

## Q: When would you use stateless instead of stateful?

**Answer:**

> I generally prefer stateless APIs for REST APIs and microservices because requests are independent and any application instance can process them, which makes horizontal scaling easier. I would use a stateful approach when the application genuinely needs to maintain user-specific context across requests, such as session-based authentication or certain multi-step workflows.

---

# 28. Interview Question

## Q: Why are microservices usually stateless?

**Answer:**

> Stateless services are easier to scale because a request does not depend on data stored in one specific service instance. A load balancer can route the request to any available instance. This also simplifies deployment, failover, and horizontal scaling.

---

# 29. Interview Question

## Q: Can a stateless application use a database?

**Answer:**

> Yes. Stateless does not mean the application cannot store data. It can store business data such as users, bookings, and payments in a database. Stateless specifically means the application does not rely on client-specific HTTP session state stored between requests.

---

# 30. Interview Question

## Q: What happens with HttpSession when multiple servers are used?

**Answer:**

> If the session is stored only in one application instance, another instance may not know about it. To solve this, applications can use a shared session store such as Redis, session replication, or sometimes sticky sessions. A shared store is often preferred when server-side sessions are required.

---

# 31. Quick Revision Diagram

## Stateless

```text
Client
   |
   | Request + JWT
   v
Server
   |
   | Validate JWT
   v
Response

Next Request
   |
   | Request + JWT
   v
Any Server Instance
```

---

## Stateful

```text
Client
   |
   | Login
   v
Server
   |
   | Create Session
   v
JSESSIONID = ABC123
   |
   v
Client stores cookie

Next Request
   |
   | Cookie: JSESSIONID=ABC123
   v
Server
   |
   v
Load stored session
```

---

# 32. One-Line Revision

```text
Stateless = Client sends everything needed on every request.

Stateful = Client sends an identifier and server retrieves previously stored state.
```

---

# 33. Final Interview Shortcut

Remember these keywords:

### Stateless

```text
Independent requests
JWT
No server-side client session
Easy scaling
Microservices
REST APIs
```

### Stateful

```text
Session
JSESSIONID
Server remembers client
Shared session store
Redis
Multi-step workflow
```

---

# 34. Best 30-Second Interview Answer

> Stateless means the server does not maintain client-specific session state between requests, so every request contains the information required to process it. JWT-based REST APIs are a common example. Stateful means the server stores some client-specific state, such as an HTTP session, and subsequent requests refer to that state using an identifier such as JSESSIONID. For REST microservices I generally prefer stateless design because it is easier to scale horizontally, while stateful design is useful when server-side session or workflow state is actually required.
