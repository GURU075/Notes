# Spring Security Authentication – Complete Interview Notes

## 1. What is Authentication?

Authentication means:

> Verifying who the user is and checking whether the provided credentials are correct.

Example:

```text
Username: guru@gmail.com
Password: guru123
```

Spring Security checks:

- Does the user exist?
- Is the password correct?
- Is the account active?
- Is the account locked or expired?

---

## 2. High-Level Authentication Flow

```text
Client sends login request
        ↓
Spring Security Filter Chain
        ↓
AuthenticationManager
        ↓
AuthenticationProvider
        ↓
UserDetailsService
        ↓
Database
        ↓
PasswordEncoder verifies password
        ↓
Authenticated user stored in SecurityContextHolder
```

---

## 3. Step-by-Step Authentication Flow

### Step 1: Client sends login credentials

Example request:

```http
POST /login

username=guru
password=12345
```

The request first enters the Spring Security filter chain.

---

### Step 2: Security filter extracts credentials

For username-password login, Spring commonly uses:

```java
UsernamePasswordAuthenticationFilter
```

It reads the username and password and creates an unauthenticated object:

```java
UsernamePasswordAuthenticationToken authentication =
        new UsernamePasswordAuthenticationToken(
                username,
                password
        );
```

At this stage:

```java
authentication.isAuthenticated(); // false
```

This object represents an authentication request.

---

### Step 3: Filter calls `AuthenticationManager`

The filter sends the authentication request to:

```java
AuthenticationManager
```

Conceptually:

```java
Authentication result =
        authenticationManager.authenticate(authentication);
```

Its responsibility is:

> Accept credentials, verify them and return an authenticated object if they are valid.

Main method:

```java
public interface AuthenticationManager {

    Authentication authenticate(
            Authentication authentication
    ) throws AuthenticationException;
}
```

---

### Step 4: `ProviderManager` selects a provider

The common implementation of `AuthenticationManager` is:

```java
ProviderManager
```

It may contain multiple providers:

```text
ProviderManager
    ├── DaoAuthenticationProvider
    ├── LDAP AuthenticationProvider
    ├── JWT AuthenticationProvider
    └── Custom AuthenticationProvider
```

For normal database-based username-password authentication, Spring usually uses:

```java
DaoAuthenticationProvider
```

---

### Step 5: `DaoAuthenticationProvider` loads the user

`DaoAuthenticationProvider` performs two main operations:

```text
1. Load the user using UserDetailsService
2. Verify the password using PasswordEncoder
```

---

### Step 6: `UserDetailsService` loads user data

Main method:

```java
UserDetails loadUserByUsername(String username)
        throws UsernameNotFoundException;
```

Example implementation:

```java
@Service
public class CustomUserDetailsService
        implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(
            UserRepository userRepository
    ) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) {

        User user = userRepository
                .findByEmail(username)
                .orElseThrow(() ->
                        new UsernameNotFoundException(
                                "User not found"
                        )
                );

        return org.springframework.security.core.userdetails.User
                .withUsername(user.getEmail())
                .password(user.getPassword())
                .roles(user.getRole())
                .build();
    }
}
```

Important:

> `UserDetailsService` loads the user. It normally does not compare passwords.

---

### Step 7: `UserDetails` represents the user

`UserDetails` contains:

- Username
- Encoded password
- Roles and authorities
- Account locked status
- Account expired status
- Credentials expired status
- Enabled or disabled status

Quick difference:

```text
UserDetailsService → Loads the user
UserDetails        → Represents the user
```

---

### Step 8: `PasswordEncoder` verifies the password

Passwords must not be stored as plain text.

Example database value:

```text
$2a$10$U4VRg...
```

Spring compares:

```text
Raw password entered by user
                vs
Encoded password stored in database
```

Using:

```java
passwordEncoder.matches(
        rawPassword,
        encodedPassword
);
```

Password encoder configuration:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

During registration:

```java
user.setPassword(
        passwordEncoder.encode(request.getPassword())
);
```

During login, `DaoAuthenticationProvider` performs the comparison internally.

---

### Step 9: Authentication succeeds or fails

#### When credentials are correct

Spring returns an authenticated object containing:

- Principal
- Authorities
- `authenticated = true`

Example:

```java
Authentication authentication = ...;

authentication.getPrincipal();
authentication.getAuthorities();
authentication.isAuthenticated(); // true
```

The authenticated object is stored in:

```java
SecurityContextHolder
```

#### When credentials are incorrect

Spring may throw:

```text
BadCredentialsException
UsernameNotFoundException
DisabledException
LockedException
CredentialsExpiredException
```

---

## 4. What is `SecurityContextHolder`?

`SecurityContextHolder` stores the currently authenticated user.

Structure:

```text
SecurityContextHolder
        ↓
SecurityContext
        ↓
Authentication
        ↓
Principal + Authorities
```

Access the current user:

```java
Authentication authentication =
        SecurityContextHolder
                .getContext()
                .getAuthentication();

String username = authentication.getName();
```

In a controller:

```java
@GetMapping("/profile")
public String getProfile(Authentication authentication) {

    return "Logged-in user: "
            + authentication.getName();
}
```

---

## 5. Authentication vs Authorization

| Concept | Meaning | Example |
|---|---|---|
| Authentication | Who are you? | Verify username and password |
| Authorization | What are you allowed to do? | Only ADMIN can delete users |

Example configuration:

```java
http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/auth/**").permitAll()
        .requestMatchers("/admin/**").hasRole("ADMIN")
        .anyRequest().authenticated()
);
```

---

## 6. Complete Database Authentication Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final CustomUserDetailsService userDetailsService;

    public SecurityConfig(
            CustomUserDetailsService userDetailsService
    ) {
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {

        DaoAuthenticationProvider provider =
                new DaoAuthenticationProvider();

        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());

        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration configuration
    ) throws Exception {

        return configuration.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity http
    ) throws Exception {

        return http
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/auth/**").permitAll()
                        .requestMatchers("/admin/**")
                            .hasRole("ADMIN")
                        .anyRequest().authenticated()
                )
                .authenticationProvider(
                        authenticationProvider()
                )
                .httpBasic(Customizer.withDefaults())
                .build();
    }
}
```

---

## 7. Authentication in a JWT Application

### First login request

```text
Username + Password
        ↓
AuthenticationManager
        ↓
AuthenticationProvider
        ↓
UserDetailsService
        ↓
PasswordEncoder
        ↓
Authentication success
        ↓
Generate JWT
        ↓
Return JWT to client
```

Example login service:

```java
public String login(LoginRequest request) {

    Authentication authentication =
            authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(
                            request.getEmail(),
                            request.getPassword()
                    )
            );

    return jwtService.generateToken(authentication);
}
```

---

### Subsequent requests

Client sends:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Flow:

```text
Request with JWT
      ↓
JWT authentication filter
      ↓
Extract token
      ↓
Validate signature and expiry
      ↓
Extract username
      ↓
Load user if required
      ↓
Create authenticated Authentication object
      ↓
Store it in SecurityContextHolder
      ↓
Continue filter chain
```

A JWT filter commonly extends:

```java
public class JwtAuthenticationFilter
        extends OncePerRequestFilter {
}
```

Important distinction:

```text
UsernamePasswordAuthenticationFilter
→ Handles username-password login

OncePerRequestFilter
→ Commonly used to validate JWT on every request
```

---

## 8. Session-Based vs JWT Authentication

| Session-Based Authentication | JWT Authentication |
|---|---|
| Server stores user session | Server usually remains stateless |
| Browser sends session cookie | Client sends JWT in every request |
| SecurityContext may be stored in HTTP session | SecurityContext is rebuilt from token |
| Common in traditional web apps | Common in REST APIs and microservices |

For stateless JWT authentication:

```java
http.sessionManagement(session ->
        session.sessionCreationPolicy(
                SessionCreationPolicy.STATELESS
        )
);
```

---

# Interview Answers

## 1. How is authentication handled in Spring Security?

> When a login request comes in, it first passes through Spring Security's filter chain. A security filter extracts the credentials and creates an unauthenticated `Authentication` object. This object is passed to the `AuthenticationManager`, whose common implementation is `ProviderManager`.
>
> The manager delegates authentication to a suitable `AuthenticationProvider`. For normal username-password authentication, `DaoAuthenticationProvider` is generally used. It calls `UserDetailsService` to load the user from the database and uses `PasswordEncoder` to compare the entered password with the stored encoded password.
>
> If the credentials are valid, Spring creates an authenticated `Authentication` object and stores it in the `SecurityContextHolder`. After that, Spring uses the user's roles and authorities for authorization.

---

## 2. 30-second interview answer

> Spring Security handles authentication through its filter chain. A filter extracts the credentials and passes an `Authentication` object to the `AuthenticationManager`. The manager delegates to an `AuthenticationProvider`, such as `DaoAuthenticationProvider`. The provider uses `UserDetailsService` to load the user and `PasswordEncoder` to verify the password. On success, the authenticated object is stored in the `SecurityContextHolder`.

---

## 3. What is `AuthenticationManager`?

> `AuthenticationManager` coordinates the authentication process. It accepts an unauthenticated `Authentication` object and returns an authenticated object when the credentials are valid.

---

## 4. What is `AuthenticationProvider`?

> An `AuthenticationProvider` performs the actual authentication for a particular authentication type. For example, `DaoAuthenticationProvider` handles username-password authentication.

---

## 5. What is `UserDetailsService`?

> `UserDetailsService` loads the user from a source such as a database using the username. It returns a `UserDetails` object containing the username, encoded password, account status and authorities.

---

## 6. Does `UserDetailsService` validate the password?

> No. It normally only loads the user. `DaoAuthenticationProvider` uses `PasswordEncoder` to compare the entered password with the stored encoded password.

---

## 7. What is stored in `SecurityContextHolder`?

> It stores the current `SecurityContext`, which contains the authenticated `Authentication` object. From that object, we can access the logged-in user and their roles or authorities.

---

## 8. Why do we need `PasswordEncoder`?

> We should never store plain-text passwords. The password is encoded before storing it, and during login Spring Security compares the entered password with the stored encoded password using `PasswordEncoder`.

---

## 9. What happens after successful authentication?

> Spring creates an authenticated `Authentication` object, stores it in the `SecurityContextHolder` and continues the filter chain. After that, authorization rules decide whether the user can access the requested endpoint.

---

## 10. What happens when authentication fails?

> Spring throws an `AuthenticationException`, such as `BadCredentialsException` or `LockedException`. For a REST API, this normally results in a `401 Unauthorized` response.

---

# Quick Revision Table

| Component | Responsibility |
|---|---|
| Security Filter | Extracts credentials from the request |
| Authentication | Represents authentication request or authenticated user |
| AuthenticationManager | Coordinates authentication |
| ProviderManager | Selects the correct provider |
| AuthenticationProvider | Performs the actual authentication |
| DaoAuthenticationProvider | Handles database username-password authentication |
| UserDetailsService | Loads the user |
| UserDetails | Represents user details |
| PasswordEncoder | Encodes and verifies passwords |
| SecurityContextHolder | Stores the authenticated user |
| GrantedAuthority | Represents roles and permissions |

---

# One-Line Flow for Revision

```text
Filter → AuthenticationManager → ProviderManager
→ AuthenticationProvider → UserDetailsService
→ PasswordEncoder → SecurityContextHolder
```
