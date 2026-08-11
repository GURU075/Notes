# Spring Security Authorization – Complete Interview Notes

## 1. What is Authorization?

Authorization means:

> Deciding what an authenticated user is allowed to access or perform.

Authentication answers:

```text
Who are you?
```

Authorization answers:

```text
What are you allowed to do?
```

Example:

```text
USER  → Can view their profile
ADMIN → Can view, create, update and delete users
```

A user must normally be authenticated before Spring Security can authorize access.

---

## 2. Authentication vs Authorization

| Authentication | Authorization |
|---|---|
| Verifies the user's identity | Verifies the user's permissions |
| Checks username, password, JWT, OTP, etc. | Checks roles and authorities |
| Happens first | Happens after authentication |
| Failure normally returns `401 Unauthorized` | Failure normally returns `403 Forbidden` |
| Example: “Is this really Guru?” | Example: “Can Guru delete this user?” |

### Simple interview answer

> Authentication checks who the user is, while authorization checks what that user is permitted to do. In Spring Security, authentication creates an `Authentication` object, and authorization uses the roles or authorities inside that object to make access-control decisions.

---

## 3. High-Level Authorization Flow

```text
Client sends request
        ↓
Spring Security Filter Chain
        ↓
SecurityContextHolder provides Authentication
        ↓
Spring checks the configured authorization rule
        ↓
AuthorizationManager compares required permission
with user's GrantedAuthority values
        ↓
Access granted or denied
        ↓
Controller/service method executes only if allowed
```

Example:

```text
Request: DELETE /api/users/10

Required authority: USER_DELETE

Logged-in user authorities:
ROLE_ADMIN, USER_READ, USER_DELETE

Result: Access granted
```

---

## 4. Where Does Spring Get the User's Permissions?

After successful authentication, Spring Security stores the authenticated user in:

```java
SecurityContextHolder
```

The `Authentication` object contains:

- Principal
- Credentials
- Authentication status
- Collection of `GrantedAuthority` values

Example:

```java
Authentication authentication =
        SecurityContextHolder
                .getContext()
                .getAuthentication();

Collection<? extends GrantedAuthority> authorities =
        authentication.getAuthorities();
```

Possible authorities:

```text
ROLE_USER
ROLE_ADMIN
USER_READ
USER_CREATE
USER_DELETE
BOOKING_CANCEL
```

Spring Security reads these authorities while deciding whether access should be granted.

---

## 5. What is `GrantedAuthority`?

`GrantedAuthority` represents a permission assigned to the current user.

Its main method is:

```java
String getAuthority();
```

The commonly used implementation is:

```java
SimpleGrantedAuthority
```

Example:

```java
List<GrantedAuthority> authorities = List.of(
        new SimpleGrantedAuthority("ROLE_ADMIN"),
        new SimpleGrantedAuthority("USER_READ"),
        new SimpleGrantedAuthority("USER_DELETE")
);
```

These values are placed inside the authenticated `Authentication` object.

### Human-like interview answer

> A `GrantedAuthority` is simply a permission that belongs to the logged-in user. It may represent a role like `ROLE_ADMIN` or a fine-grained permission like `USER_DELETE`. Spring compares these values with the permission required by the endpoint or method.

---

## 6. Roles vs Authorities

Both roles and authorities are ultimately stored as `GrantedAuthority` values.

### Authority

An authority is an exact permission string:

```java
.hasAuthority("USER_DELETE")
```

Spring looks for exactly:

```text
USER_DELETE
```

### Role

A role is usually a broader group of permissions:

```java
.hasRole("ADMIN")
```

By default, Spring adds the prefix:

```text
ROLE_
```

Therefore:

```java
.hasRole("ADMIN")
```

checks for:

```text
ROLE_ADMIN
```

### Important difference

| Code | Required stored value |
|---|---|
| `hasRole("ADMIN")` | `ROLE_ADMIN` |
| `hasAuthority("ROLE_ADMIN")` | `ROLE_ADMIN` |
| `hasAuthority("ADMIN")` | `ADMIN` |

### Common mistake

Incorrect:

```java
.hasRole("ROLE_ADMIN")
```

This can make Spring search for:

```text
ROLE_ROLE_ADMIN
```

Correct:

```java
.hasRole("ADMIN")
```

or:

```java
.hasAuthority("ROLE_ADMIN")
```

### Interview answer

> A role is generally a high-level responsibility such as ADMIN or USER, while an authority is a specific permission such as USER_READ or USER_DELETE. Internally, both are represented using `GrantedAuthority`. The main difference is that `hasRole("ADMIN")` automatically checks for `ROLE_ADMIN`, whereas `hasAuthority()` checks the exact string.

---

# 7. Request-Level Authorization

Request-level authorization protects URL endpoints.

It is configured inside:

```java
SecurityFilterChain
```

Example:

```java
@Bean
public SecurityFilterChain securityFilterChain(
        HttpSecurity http
) throws Exception {

    return http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers(
                            "/api/auth/**",
                            "/public/**"
                    ).permitAll()

                    .requestMatchers("/api/admin/**")
                    .hasRole("ADMIN")

                    .requestMatchers(
                            HttpMethod.GET,
                            "/api/users/**"
                    ).hasAuthority("USER_READ")

                    .requestMatchers(
                            HttpMethod.POST,
                            "/api/users/**"
                    ).hasAuthority("USER_CREATE")

                    .requestMatchers(
                            HttpMethod.DELETE,
                            "/api/users/**"
                    ).hasAuthority("USER_DELETE")

                    .anyRequest().authenticated()
            )
            .build();
}
```

---

## 8. Meaning of Common Authorization Rules

### `permitAll()`

Anyone can access the endpoint.

```java
.requestMatchers("/api/auth/**").permitAll()
```

### `authenticated()`

Any logged-in user can access it.

```java
.requestMatchers("/api/profile/**").authenticated()
```

### `hasRole()`

The user must have one particular role.

```java
.requestMatchers("/api/admin/**")
.hasRole("ADMIN")
```

Required authority:

```text
ROLE_ADMIN
```

### `hasAnyRole()`

The user must have at least one listed role.

```java
.requestMatchers("/api/reports/**")
.hasAnyRole("ADMIN", "MANAGER")
```

### `hasAuthority()`

The user must have an exact permission.

```java
.requestMatchers(
        HttpMethod.DELETE,
        "/api/bookings/**"
).hasAuthority("BOOKING_DELETE")
```

### `hasAnyAuthority()`

The user must have at least one listed permission.

```java
.requestMatchers("/api/reports/**")
.hasAnyAuthority(
        "REPORT_READ",
        "REPORT_EXPORT"
)
```

### `denyAll()`

Nobody can access the endpoint.

```java
.requestMatchers("/internal/**").denyAll()
```

---

## 9. Rule Ordering is Important

Spring evaluates request authorization rules in order.

Correct:

```java
.authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/admin/**")
            .hasRole("ADMIN")
        .requestMatchers("/api/**")
            .authenticated()
        .anyRequest()
            .denyAll()
)
```

Put specific request matchers first and general rules later.

---

# 10. How Request Authorization Works Internally

```text
1. Request enters the SecurityFilterChain
2. Spring obtains Authentication from SecurityContextHolder
3. RequestMatcher finds the matching authorization rule
4. AuthorizationManager reads the user's GrantedAuthority values
5. Required authority is compared with user authorities
6. Positive decision → request continues
7. Negative decision → AccessDeniedException
```

Modern Spring Security uses:

```java
AuthorizationManager
```

for access-control decisions.

---

## 11. What is `AuthorizationManager`?

`AuthorizationManager` makes the final authorization decision.

It checks:

```text
Current Authentication
+
Protected object, such as an HTTP request or method invocation
```

### Interview answer

> `AuthorizationManager` is the component that makes the final access-control decision. It checks the current `Authentication` and the protected resource, such as an HTTP request or service method, and decides whether access should be granted or denied.

---

# 12. Method-Level Authorization

Method-level authorization protects service or controller methods.

Example:

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long userId) {
    userRepository.deleteById(userId);
}
```

---

## 13. Enabling Method Security

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}
```

This enables:

```text
@PreAuthorize
@PostAuthorize
@PreFilter
@PostFilter
```

---

## 14. `@PreAuthorize`

Checks permission before method execution.

```java
@PreAuthorize("hasAuthority('USER_DELETE')")
public void deleteUser(Long id) {
    userRepository.deleteById(id);
}
```

Multiple conditions:

```java
@PreAuthorize(
    "hasRole('ADMIN') or hasAuthority('USER_DELETE')"
)
public void deleteUser(Long id) {
    userRepository.deleteById(id);
}
```

---

## 15. Accessing Method Parameters

```java
@PreAuthorize(
    "#userId == authentication.principal.id or hasRole('ADMIN')"
)
public UserResponse getUser(Long userId) {
    return userService.getUser(userId);
}
```

Meaning:

```text
Allow when:
- The logged-in user requests their own data
OR
- The logged-in user is ADMIN
```

---

## 16. `@PostAuthorize`

Checks authorization after the method runs but before the result is returned.

```java
@PostAuthorize(
    "returnObject.ownerEmail == authentication.name"
)
public Document getDocument(Long id) {
    return documentRepository.findById(id)
            .orElseThrow();
}
```

Prefer `@PreAuthorize` when the decision can be made before executing the method.

---

## 17. `@PreFilter`

Filters an input collection before method execution.

```java
@PreFilter(
    "filterObject.owner == authentication.name"
)
public void updateDocuments(
        List<Document> documents
) {
}
```

---

## 18. `@PostFilter`

Filters a returned collection after method execution.

```java
@PostFilter(
    "filterObject.owner == authentication.name"
)
public List<Document> getDocuments() {
    return documentRepository.findAll();
}
```

For large data, filtering in the database is normally more efficient.

---

# 19. Request-Level vs Method-Level Authorization

| Request-Level | Method-Level |
|---|---|
| Protects endpoints | Protects Java methods |
| Configured in `SecurityFilterChain` | Uses annotations |
| Stops request before controller | Protects business operation |
| Good for broad URL rules | Good for ownership and business rules |

Recommended:

```text
Request level → First security boundary
Method level  → Business-level defense in depth
```

---

# 20. Custom Authorization Logic

```java
@Component("bookingSecurity")
public class BookingSecurity {

    private final BookingRepository bookingRepository;

    public BookingSecurity(
            BookingRepository bookingRepository
    ) {
        this.bookingRepository = bookingRepository;
    }

    public boolean canCancel(
            Authentication authentication,
            Long bookingId
    ) {
        return bookingRepository
                .findById(bookingId)
                .filter(booking ->
                        booking.getUserEmail()
                                .equals(authentication.getName())
                )
                .filter(booking ->
                        booking.getStatus()
                                == BookingStatus.CONFIRMED
                )
                .isPresent();
    }
}
```

Use it:

```java
@PreAuthorize(
    "@bookingSecurity.canCancel(authentication, #bookingId)"
)
public void cancelBooking(Long bookingId) {
    bookingService.cancel(bookingId);
}
```

### Interview answer

> When authorization depends on business data, I keep the rule in a separate security component and call it from `@PreAuthorize`. This makes the authorization logic testable and reusable.

---

# 21. Role-Based Access Control — RBAC

RBAC means:

```text
Access is granted based on the user's role.
```

Example:

| Role | Permissions |
|---|---|
| USER | View events and book seats |
| ORGANIZER | Create events and shows |
| ADMIN | Manage users and all bookings |

```java
.requestMatchers(
        HttpMethod.POST,
        "/api/events/**"
).hasAnyRole("ORGANIZER", "ADMIN")
```

---

# 22. Permission-Based Authorization

Fine-grained permissions:

```text
EVENT_READ
EVENT_CREATE
EVENT_UPDATE
EVENT_DELETE
BOOKING_CANCEL
USER_DELETE
```

Example:

```java
.requestMatchers(
        HttpMethod.POST,
        "/api/events/**"
).hasAuthority("EVENT_CREATE")
```

A role can include multiple permissions.

---

# 23. Mapping Roles and Permissions from Database

```java
private Collection<? extends GrantedAuthority>
getAuthorities(User user) {

    Set<SimpleGrantedAuthority> authorities =
            new HashSet<>();

    for (Role role : user.getRoles()) {

        authorities.add(
                new SimpleGrantedAuthority(
                        "ROLE_" + role.getName()
                )
        );

        for (Permission permission :
                role.getPermissions()) {

            authorities.add(
                    new SimpleGrantedAuthority(
                            permission.getName()
                    )
            );
        }
    }

    return authorities;
}
```

Possible output:

```text
ROLE_ORGANIZER
EVENT_CREATE
EVENT_UPDATE
SHOW_CREATE
```

---

# 24. Authorization in JWT Applications

A JWT may contain:

```json
{
  "sub": "guru@gmail.com",
  "roles": ["ROLE_USER"],
  "permissions": [
    "BOOKING_CREATE",
    "BOOKING_CANCEL"
  ]
}
```

JWT filter flow:

```text
1. Read token
2. Validate token
3. Extract roles and permissions
4. Create Authentication
5. Store it in SecurityContextHolder
6. Authorization rules check authorities
```

Important:

> JWT validation performs authentication. Checking endpoint permissions performs authorization.

---

# 25. What Happens When Authorization Fails?

When the user is authenticated but lacks permission, Spring throws:

```java
AccessDeniedException
```

Typical response:

```http
403 Forbidden
```

---

## 26. Difference Between 401 and 403

### `401 Unauthorized`

Usually means authentication is missing or invalid.

Examples:

- Missing JWT
- Invalid JWT
- Expired JWT
- Wrong credentials

Handled by:

```java
AuthenticationEntryPoint
```

### `403 Forbidden`

Means the user is authenticated but lacks permission.

Handled by:

```java
AccessDeniedHandler
```

### Interview answer

> A 401 response means authentication is missing or invalid. A 403 response means authentication succeeded, but the user does not have the required role or permission.

---

# 27. Custom 401 and 403 Handling

```java
http.exceptionHandling(exception -> exception
        .authenticationEntryPoint(
                customAuthenticationEntryPoint
        )
        .accessDeniedHandler(
                customAccessDeniedHandler
        )
);
```

---

# 28. Human-Like Main Interview Answer

> After authentication succeeds, Spring Security stores the logged-in user's details in the `SecurityContextHolder`. The `Authentication` object contains the user's roles and permissions as `GrantedAuthority` values.
>
> When the user calls an endpoint, Spring checks the authorization rules configured in `SecurityFilterChain`, such as `hasRole`, `hasAuthority`, or `authenticated`. Internally, an `AuthorizationManager` compares the permission required by the endpoint with the authorities available in the current `Authentication`.
>
> We can also apply authorization at method level using annotations such as `@PreAuthorize`. If the user has the required permission, execution continues. Otherwise, Spring throws `AccessDeniedException` and normally returns `403 Forbidden`.

---

## 29. 30-Second Interview Answer

> Spring Security performs authorization after authentication. It reads the current `Authentication` from the `SecurityContextHolder`, checks its `GrantedAuthority` values, and compares them with the rule configured for the endpoint or method. We configure rules using `authorizeHttpRequests`, `hasRole`, `hasAuthority`, or annotations like `@PreAuthorize`. If permission is missing, Spring returns `403 Forbidden`.

---

# 30. Common Interview Questions

## What is authorization?

> Authorization checks whether an authenticated user has permission to access an endpoint, method, or resource.

## Where are roles and permissions stored?

> They are stored as `GrantedAuthority` values inside the authenticated `Authentication` object.

## What is the difference between `hasRole()` and `hasAuthority()`?

> `hasAuthority()` checks an exact string. `hasRole("ADMIN")` normally checks for `ROLE_ADMIN`.

## Why use method security?

> Method security protects the business operation itself, even if the method is invoked from another controller or component.

## What happens when permission is missing?

> Spring throws `AccessDeniedException` and normally returns `403 Forbidden`.

## Can authorization rules come from the database?

> Yes. Roles and permissions can be loaded from database tables, converted to `GrantedAuthority` objects, and then checked by Spring Security.

## How does JWT authorization work?

> The JWT filter validates the token and creates an authenticated object containing roles or permissions. Spring then checks those authorities against endpoint or method rules.

---

# 31. Best Practices

1. Deny access by default.
2. Put specific URL matchers before general matchers.
3. Use fine-grained permissions for complex systems.
4. Use both request-level and method-level authorization.
5. Verify resource ownership on the server.
6. Keep complex business authorization in a separate component.
7. Return 401 for authentication failure and 403 for authorization failure.
8. Follow the principle of least privilege.
9. Never rely only on frontend authorization checks.
10. Test both allowed and denied scenarios.

---

# 32. Quick Revision Table

| Component / Method | Purpose |
|---|---|
| `SecurityContextHolder` | Provides current authenticated user |
| `Authentication` | Contains principal and authorities |
| `GrantedAuthority` | Represents role or permission |
| `AuthorizationManager` | Makes access-control decision |
| `authorizeHttpRequests()` | Configures URL authorization |
| `requestMatchers()` | Selects protected requests |
| `permitAll()` | Allows everyone |
| `authenticated()` | Requires login |
| `hasRole("ADMIN")` | Requires `ROLE_ADMIN` |
| `hasAuthority("USER_DELETE")` | Requires exact permission |
| `@EnableMethodSecurity` | Enables method authorization |
| `@PreAuthorize` | Checks before method execution |
| `@PostAuthorize` | Checks returned result |
| `@PreFilter` | Filters input collection |
| `@PostFilter` | Filters returned collection |
| `AuthenticationEntryPoint` | Handles 401 |
| `AccessDeniedHandler` | Handles 403 |

---

# 33. One-Line Flow for Revision

```text
Authentication → SecurityContextHolder
→ GrantedAuthority → AuthorizationManager
→ Access Granted or 403 Forbidden
```

---

## Official References

- Spring Security Reference — Authorization Architecture
- Spring Security Reference — Authorize HTTP Requests
- Spring Security Reference — Method Security
- Spring Security Reference — Authentication Architecture
