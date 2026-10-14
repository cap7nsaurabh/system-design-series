# 10 — Spring Security in Spring Boot

> [!IMPORTANT]
> This module targets **Spring Boot 3.x / Spring Security 6.x**. The old `WebSecurityConfigurerAdapter` pattern is **removed** in Spring Security 6. All configuration is done via `SecurityFilterChain` beans. Code examples reflect this modern approach.

---

## Table of Contents

1. [Spring Security Architecture](#1-spring-security-architecture)
2. [Spring Boot Auto-Configuration](#2-spring-boot-auto-configuration)
3. [Authentication Deep Dive](#3-authentication-deep-dive)
4. [JWT Authentication — Production Pattern](#4-jwt-authentication--production-pattern)
5. [SecurityFilterChain Configuration](#5-securityfilterchain-configuration)
6. [Method Security](#6-method-security)
7. [OAuth2 Overview](#7-oauth2-overview)
8. [Role vs Authority vs Permission](#8-role-vs-authority-vs-permission)
9. [Common Security Vulnerabilities](#9-common-security-vulnerabilities)
10. [Security Testing](#10-security-testing)
11. [⚠️ Interview Traps & Gotchas](#️-interview-traps--gotchas)
12. [🔍 Deep Dive — Internal Mechanics](#-deep-dive--internal-mechanics)
13. [📝 Summary & Key Takeaways](#-summary--key-takeaways)
14. [❓ Practice Questions](#-practice-questions)

---

## 1. Spring Security Architecture

Spring Security is a powerful, highly customizable authentication and access-control framework. Before writing a single line of configuration, you must understand how requests flow through it.

### 1.1 The Security Filter Chain — The Cornerstone

Spring Security is fundamentally a **chain of servlet filters**. Every HTTP request passes through this chain before reaching your controllers. The chain enforces authentication and authorization at the HTTP layer.

```
Incoming HTTP Request
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│                  Servlet Container (Tomcat)               │
│                                                          │
│   ┌────────────────────────────────────────────────┐    │
│   │           DelegatingFilterProxy                │    │
│   │  (registered as "springSecurityFilterChain")   │    │
│   └────────────────┬───────────────────────────────┘    │
│                    │ delegates to                        │
│   ┌────────────────▼───────────────────────────────┐    │
│   │               FilterChainProxy                 │    │
│   │                                                │    │
│   │  ┌──────────────────────────────────────────┐ │    │
│   │  │         SecurityFilterChain #1            │ │    │
│   │  │  (matches /api/**)                        │ │    │
│   │  │  Filter1 → Filter2 → Filter3 → ...       │ │    │
│   │  └──────────────────────────────────────────┘ │    │
│   │  ┌──────────────────────────────────────────┐ │    │
│   │  │         SecurityFilterChain #2            │ │    │
│   │  │  (matches /admin/**)                      │ │    │
│   │  │  Filter1 → Filter2 → Filter3 → ...       │ │    │
│   │  └──────────────────────────────────────────┘ │    │
│   └────────────────────────────────────────────────┘    │
│                    │                                     │
│                    ▼                                     │
│              DispatcherServlet                           │
│                    │                                     │
│                    ▼                                     │
│              @RestController                             │
└──────────────────────────────────────────────────────────┘
```

### 1.2 DelegatingFilterProxy

`DelegatingFilterProxy` is a standard **servlet filter** registered in the servlet container. It acts as a bridge between the servlet container's lifecycle and Spring's `ApplicationContext`. It delegates all work to a Spring-managed bean named `springSecurityFilterChain`.

**Why does this indirection exist?**
The servlet container initializes filters before Spring's `ApplicationContext` is fully loaded. `DelegatingFilterProxy` delays the actual delegation until the Spring context is ready.

```java
// Conceptual representation of DelegatingFilterProxy
public class DelegatingFilterProxy implements Filter {
    private Filter delegate; // = FilterChainProxy bean from Spring context

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        // Lazily looks up the bean named "springSecurityFilterChain"
        if (this.delegate == null) {
            this.delegate = findFilterBean("springSecurityFilterChain");
        }
        this.delegate.doFilter(req, res, chain);
    }
}
```

### 1.3 FilterChainProxy

`FilterChainProxy` is the real Spring Security entry point. It holds a list of `SecurityFilterChain` instances and, for each incoming request, finds the **first matching chain** and runs the request through it.

```java
// Simplified FilterChainProxy logic
public class FilterChainProxy implements Filter {
    private List<SecurityFilterChain> filterChains;

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        HttpServletRequest request = (HttpServletRequest) req;

        // Find the first matching SecurityFilterChain
        for (SecurityFilterChain filterChain : filterChains) {
            if (filterChain.matches(request)) {
                List<Filter> filters = filterChain.getFilters();
                // Run through all filters in the chain
                new VirtualFilterChain(chain, filters).doFilter(request, res);
                return;
            }
        }
        // No match — pass through
        chain.doFilter(req, res);
    }
}
```

> [!NOTE]
> Only the **first** matching `SecurityFilterChain` runs. This is critical when you have multiple chains (e.g., one for `/api/**` and another for everything else). Order matters — use `@Order` annotation on your beans.

### 1.4 Default Filter Ordering in Spring Security 6

Spring Security installs the following filters in this order (partial list — full list has 33+ filters):

| Order | Filter | Purpose |
|-------|--------|---------|
| 100 | `DisableEncodeUrlFilter` | Prevents session ID in URLs |
| 200 | `WebAsyncManagerIntegrationFilter` | Async SecurityContext propagation |
| 300 | `SecurityContextHolderFilter` | Loads/saves SecurityContext |
| 400 | `HeaderWriterFilter` | Adds security headers (X-Frame-Options, etc.) |
| 500 | `CorsFilter` | CORS handling |
| 600 | `CsrfFilter` | CSRF token validation |
| 700 | `LogoutFilter` | Logout processing |
| 800 | `UsernamePasswordAuthenticationFilter` | Form login |
| 900 | `BasicAuthenticationFilter` | HTTP Basic authentication |
| 1000 | `BearerTokenAuthenticationFilter` | JWT/OAuth2 bearer tokens |
| 1100 | `RequestCacheAwareFilter` | Replays saved requests |
| 1200 | `SecurityContextHolderAwareRequestFilter` | Servlet API integration |
| 1300 | `AnonymousAuthenticationFilter` | Sets anonymous user if no auth |
| 1400 | `ExceptionTranslationFilter` | Converts security exceptions to responses |
| 1500 | `AuthorizationFilter` | Access control decisions |

### 1.5 Authentication vs Authorization

These two concepts are fundamentally different but often confused:

| Concept | Question | Spring Class | When |
|---------|----------|-------------|------|
| **Authentication** | *Who are you?* | `AuthenticationManager` | Before accessing any resource |
| **Authorization** | *What can you do?* | `AuthorizationManager` | After successful authentication |

- **Authentication**: Verifying identity (username + password, JWT token, OAuth2 token)
- **Authorization**: Verifying permissions (can this user access `/admin`?)

### 1.6 SecurityContext and SecurityContextHolder

Once a user is authenticated, Spring Security stores the result in a `SecurityContext`:

```java
// SecurityContext holds the Authentication object
public interface SecurityContext {
    Authentication getAuthentication();
    void setAuthentication(Authentication authentication);
}

// Authentication holds principal, credentials, and authorities
public interface Authentication extends Principal {
    Collection<? extends GrantedAuthority> getAuthorities();
    Object getCredentials();  // password (cleared after auth)
    Object getPrincipal();    // UserDetails or username
    boolean isAuthenticated();
}
```

`SecurityContextHolder` provides a static access point:

```java
// Reading the current user anywhere in your code
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();

// Getting UserDetails if you stored it as principal
UserDetails userDetails = (UserDetails) auth.getPrincipal();
```

### 1.7 ThreadLocal and SecurityContext Propagation

By default, `SecurityContextHolder` uses `ThreadLocal` strategy — meaning the `SecurityContext` is bound to the **current thread**.

```java
// SecurityContextHolder strategies
SecurityContextHolder.MODE_THREADLOCAL      // default — per-thread
SecurityContextHolder.MODE_INHERITABLETHREADLOCAL // child threads inherit context
SecurityContextHolder.MODE_GLOBAL          // single global context (testing only)
```

> [!WARNING]
> In `@Async` methods, the security context is **NOT** automatically propagated because the async method runs on a different thread. This is a classic interview trap (covered later).

**Fix for `@Async` + Security:**

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(500);
        // THIS is the key — wraps tasks with SecurityContext delegation
        executor.setTaskDecorator(new DelegatingSecurityContextRunnable.Factory());
        executor.initialize();
        return executor;
    }
}

// Or manually propagate in individual async calls:
@Async
public void processAsync() {
    // SecurityContext is NOT available here by default!
    Authentication auth = SecurityContextHolder.getContext().getAuthentication(); // NULL
}
```

**Better fix — use `DelegatingSecurityContextExecutor`:**

```java
@Bean
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.initialize();
    // Wraps the executor so SecurityContext is propagated
    return new DelegatingSecurityContextExecutor(executor.getObject());
}
```

---

## 2. Spring Boot Auto-Configuration

### 2.1 What Spring Boot Auto-Configures

When you add `spring-boot-starter-security` to your project, Spring Boot automatically configures:

1. **A default `UserDetailsService`** with a single in-memory user (`user` / generated password)
2. **HTTP Basic authentication** for all endpoints
3. **Form login** at `/login`
4. **CSRF protection** enabled
5. **Security headers** (X-Frame-Options, X-XSS-Protection, etc.)
6. **Session fixation protection**
7. **Logout** at `/logout`

```
// You'll see this in logs on startup:
Using generated security password: 3d4a87b2-5f6e-4c1a-b2e3-8f9d0a1b2c3d
This generated password is for development use only.
```

### 2.2 The Auto-Configuration Classes

```java
// SpringBootWebSecurityConfiguration — provides the default SecurityFilterChain
@ConditionalOnDefaultWebSecurity
@ConditionalOnWebApplication(type = Type.SERVLET)
class SpringBootWebSecurityConfiguration {

    @Bean
    @Order(SecurityProperties.BASIC_AUTH_ORDER)
    SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests((requests) -> requests.anyRequest().authenticated());
        http.formLogin(withDefaults());
        http.httpBasic(withDefaults());
        return http.build();
    }
}
```

The `@ConditionalOnDefaultWebSecurity` annotation means: **"only apply this if the user hasn't defined their own `SecurityFilterChain` bean."** Once you define your own, this entire auto-config backs off.

### 2.3 Customizing via application.properties

```properties
# Override default user credentials (dev only!)
spring.security.user.name=admin
spring.security.user.password=secret
spring.security.user.roles=ADMIN

# Filter order for Spring Security
spring.security.filter.order=10
```

### 2.4 Overriding Defaults — Define Your Own SecurityFilterChain

As soon as you provide a `SecurityFilterChain` bean, Spring Boot's defaults are completely overridden:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/actuator/health").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults())
            .csrf(csrf -> csrf.disable()); // for REST APIs
        return http.build();
    }
}
```

---

## 3. Authentication Deep Dive

### 3.1 Authentication Flow — The Big Picture

```
Client sends credentials (username + password)
               │
               ▼
    UsernamePasswordAuthenticationFilter
               │
               │ creates UsernamePasswordAuthenticationToken
               ▼
         AuthenticationManager
         (ProviderManager)
               │
               │ iterates over providers
               ▼
       AuthenticationProvider #1
       (DaoAuthenticationProvider)
               │
               │ calls loadUserByUsername()
               ▼
         UserDetailsService
               │
               │ returns UserDetails
               ▼
         PasswordEncoder.matches()
               │
               │ verified
               ▼
    Authentication object (with authorities)
               │
               ▼
    SecurityContextHolder.getContext()
         .setAuthentication(auth)
```

### 3.2 AuthenticationManager

`AuthenticationManager` has a single method:

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

The primary implementation is `ProviderManager`, which delegates to a list of `AuthenticationProvider` instances:

```java
// ProviderManager simplified
public class ProviderManager implements AuthenticationManager {
    private List<AuthenticationProvider> providers;

    @Override
    public Authentication authenticate(Authentication authentication) {
        for (AuthenticationProvider provider : providers) {
            if (provider.supports(authentication.getClass())) {
                try {
                    return provider.authenticate(authentication);
                } catch (AuthenticationException e) {
                    lastException = e;
                }
            }
        }
        throw lastException;
    }
}
```

### 3.3 AuthenticationProvider

```java
public interface AuthenticationProvider {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
    boolean supports(Class<?> authentication); // which token type this handles
}
```

**Built-in providers:**
| Provider | Token Type | Use Case |
|----------|-----------|---------|
| `DaoAuthenticationProvider` | `UsernamePasswordAuthenticationToken` | Username/password via UserDetailsService |
| `JwtAuthenticationProvider` | `BearerTokenAuthenticationToken` | JWT tokens |
| `RememberMeAuthenticationProvider` | `RememberMeAuthenticationToken` | Remember-me cookies |
| `AnonymousAuthenticationProvider` | `AnonymousAuthenticationToken` | Unauthenticated users |

### 3.4 UserDetailsService and UserDetails

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}

public interface UserDetails {
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();  // hashed password
    String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

**Custom UserDetailsService implementation:**

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getEmail())
            .password(user.getPasswordHash())  // already hashed in DB
            .roles(user.getRoles().toArray(new String[0]))
            .accountExpired(!user.isActive())
            .accountLocked(user.isLocked())
            .credentialsExpired(user.isPasswordExpired())
            .build();
    }
}
```

**Custom UserDetails implementation (recommended for richer data):**

```java
@Data
@AllArgsConstructor
public class AppUserDetails implements UserDetails {

    private final Long id;
    private final String email;
    private final String passwordHash;
    private final Set<GrantedAuthority> authorities;
    private final boolean enabled;
    private final boolean accountNonLocked;

    @Override
    public String getUsername() { return email; }

    @Override
    public String getPassword() { return passwordHash; }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() { return authorities; }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return accountNonLocked; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return enabled; }
}
```

### 3.5 PasswordEncoder — BCrypt and Why Raw Passwords Are Dangerous

> [!CAUTION]
> **Never store raw (plaintext) passwords.** If your database is breached, all user passwords are immediately exposed. With proper hashing, an attacker must crack each hash individually.

```java
// Register the PasswordEncoder bean
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // strength = 12 (2^12 rounds)
    // Default strength is 10 — increase for better security at cost of CPU
}
```

**BCrypt characteristics:**
- Adaptive — cost factor can increase as hardware improves
- Built-in salt — generates a unique salt per password
- Slow by design — makes brute-force attacks computationally expensive
- Output is always 60 characters long

```java
// Using BCrypt
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);
String hashed = encoder.encode("myPassword123");
// $2a$12$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy

boolean matches = encoder.matches("myPassword123", hashed); // true
boolean matches2 = encoder.matches("wrongPassword", hashed); // false
```

**DelegatingPasswordEncoder — recommended for migration:**

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // Default encoder is bcrypt, but can decode {bcrypt}, {pbkdf2}, {scrypt}, {sha256}
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
// Stored passwords look like: {bcrypt}$2a$10$...
// Allows gradual migration from old hashing to bcrypt
```

### 3.6 Form Login vs Basic Auth vs JWT

| Feature | Form Login | HTTP Basic | JWT |
|---------|-----------|-----------|-----|
| Transport | Cookie (session ID) | Base64 header | Bearer token header |
| State | Stateful (server session) | Stateless | Stateless |
| Best for | Browser-based apps | Simple API clients | REST APIs, SPAs |
| CSRF needed | ✅ Yes | ❌ No | ❌ No |
| Logout | Easy (invalidate session) | Impossible (client must stop sending) | Token expiry/blacklist |
| Security | Good | Credentials every request | Good (short-lived tokens) |

### 3.7 Custom Authentication Provider

```java
@Component
@RequiredArgsConstructor
public class ApiKeyAuthenticationProvider implements AuthenticationProvider {

    private final ApiKeyRepository apiKeyRepository;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String apiKey = (String) authentication.getCredentials();

        ApiKeyEntity entity = apiKeyRepository.findByKey(apiKey)
            .orElseThrow(() -> new BadCredentialsException("Invalid API key"));

        if (!entity.isActive()) {
            throw new DisabledException("API key is disabled");
        }

        List<GrantedAuthority> authorities = entity.getScopes()
            .stream()
            .map(scope -> new SimpleGrantedAuthority("SCOPE_" + scope))
            .collect(Collectors.toList());

        return new ApiKeyAuthenticationToken(entity.getClientId(), apiKey, authorities);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return ApiKeyAuthenticationToken.class.isAssignableFrom(authentication);
    }
}

// Register it in SecurityConfig
@Bean
public AuthenticationManager authenticationManager(
        AuthenticationConfiguration config,
        ApiKeyAuthenticationProvider apiKeyProvider) throws Exception {

    ProviderManager manager = (ProviderManager) config.getAuthenticationManager();
    manager.getProviders().add(apiKeyProvider);
    return manager;
}
```

---

## 4. JWT Authentication — Production Pattern

### 4.1 What is JWT?

JSON Web Token (JWT) is an open standard (RFC 7519) for securely transmitting information between parties as a JSON object. A JWT consists of three Base64URL-encoded parts separated by dots:

```
header.payload.signature
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyQGV4YW1wbGUuY29tIiwicm9sZXMiOlsiUk9MRV9VU0VSIl0sImlhdCI6MTcyMDAwMDAwMCwiZXhwIjoxNzIwMDAzNjAwfQ.abc123signature
```

**Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload (Claims):**
```json
{
  "sub": "user@example.com",   // subject (user identifier)
  "iss": "my-app",             // issuer
  "aud": "my-api",             // audience
  "iat": 1720000000,           // issued at
  "exp": 1720003600,           // expiration (1 hour later)
  "roles": ["ROLE_USER"],      // custom claim
  "userId": 42                 // custom claim
}
```

**Signature:**
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### 4.2 JWT Claims

| Claim | Name | Description |
|-------|------|-------------|
| `sub` | Subject | User identifier (email or user ID) |
| `iss` | Issuer | Who issued the token |
| `aud` | Audience | Intended recipient |
| `exp` | Expiration | Unix timestamp when token expires |
| `nbf` | Not Before | Token not valid before this time |
| `iat` | Issued At | When token was issued |
| `jti` | JWT ID | Unique identifier (for blacklisting) |

### 4.3 Signing Algorithms: HS256 vs RS256

| Algorithm | Type | Key | Use Case |
|-----------|------|-----|---------|
| **HS256** | Symmetric | Single shared secret | Single-service, simple setup |
| **RS256** | Asymmetric | Private key signs, Public key verifies | Multi-service, public verification |
| **ES256** | Asymmetric (EC) | Elliptic curve | High-security, smaller keys |

> [!IMPORTANT]
> **Use RS256 in microservices**. With HS256, every service that needs to verify tokens must have the secret — creating multiple attack surfaces. With RS256, the auth service keeps the private key, and other services only need the public key to verify.

### 4.4 Project Setup — Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

### 4.5 JWT Service

```java
@Service
public class JwtService {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration:3600000}") // 1 hour default
    private long jwtExpiration;

    @Value("${jwt.refresh-expiration:604800000}") // 7 days default
    private long refreshExpiration;

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public String generateToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails);
    }

    public String generateToken(Map<String, Object> extraClaims, UserDetails userDetails) {
        return buildToken(extraClaims, userDetails, jwtExpiration);
    }

    public String generateRefreshToken(UserDetails userDetails) {
        return buildToken(new HashMap<>(), userDetails, refreshExpiration);
    }

    private String buildToken(
            Map<String, Object> extraClaims,
            UserDetails userDetails,
            long expiration) {

        List<String> roles = userDetails.getAuthorities()
            .stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList());

        return Jwts.builder()
            .claims(extraClaims)
            .subject(userDetails.getUsername())
            .claim("roles", roles)
            .issuedAt(new Date(System.currentTimeMillis()))
            .expiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSigningKey(), Jwts.SIG.HS256)
            .compact();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return (username.equals(userDetails.getUsername())) && !isTokenExpired(token);
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
}
```

### 4.6 JwtAuthenticationFilter — Extending OncePerRequestFilter

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");

        // 1. If no Bearer token, skip this filter
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        // 2. Extract the JWT
        final String jwt = authHeader.substring(7);
        final String username;

        try {
            username = jwtService.extractUsername(jwt);
        } catch (JwtException e) {
            log.warn("Invalid JWT token: {}", e.getMessage());
            filterChain.doFilter(request, response);
            return;
        }

        // 3. If username extracted and no existing authentication
        if (username != null &&
            SecurityContextHolder.getContext().getAuthentication() == null) {

            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            // 4. Validate the token
            if (jwtService.isTokenValid(jwt, userDetails)) {

                // 5. Create authentication token
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,           // credentials = null (no password needed)
                        userDetails.getAuthorities()
                    );

                // 6. Add request details (IP, session, etc.)
                authToken.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request)
                );

                // 7. Set authentication in SecurityContext
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        // 8. Always continue the filter chain
        filterChain.doFilter(request, response);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // Skip JWT filter for public endpoints
        String path = request.getServletPath();
        return path.startsWith("/api/auth/") ||
               path.startsWith("/actuator/health");
    }
}
```

### 4.7 Auth Controller — Login Endpoint

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    private final RefreshTokenService refreshTokenService;

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody @Valid LoginRequest request) {
        // Throws AuthenticationException if credentials are wrong
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword())
        );

        UserDetails userDetails = userDetailsService.loadUserByUsername(request.getEmail());
        String accessToken = jwtService.generateToken(userDetails);
        String refreshToken = refreshTokenService.createRefreshToken(request.getEmail());

        return ResponseEntity.ok(new AuthResponse(accessToken, refreshToken));
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(@RequestBody RefreshRequest request) {
        String email = refreshTokenService.validateAndExtractEmail(request.getRefreshToken());
        UserDetails userDetails = userDetailsService.loadUserByUsername(email);
        String newAccessToken = jwtService.generateToken(userDetails);

        return ResponseEntity.ok(new AuthResponse(newAccessToken, request.getRefreshToken()));
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(@RequestBody RefreshRequest request) {
        refreshTokenService.invalidateRefreshToken(request.getRefreshToken());
        return ResponseEntity.noContent().build();
    }
}

// DTOs
public record LoginRequest(@NotBlank String email, @NotBlank String password) {}
public record AuthResponse(String accessToken, String refreshToken) {}
public record RefreshRequest(String refreshToken) {}
```

### 4.8 SecurityConfig with JWT

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())  // No CSRF for stateless REST
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authenticationProvider(authenticationProvider())
            // Add JWT filter BEFORE the standard UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
                .accessDeniedHandler((req, res, e) -> res.sendError(403))
            );

        return http.build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://myapp.com", "http://localhost:3000"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

### 4.9 Refresh Token Pattern

```java
@Entity
@Table(name = "refresh_tokens")
@Data
public class RefreshToken {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String token;

    @Column(nullable = false)
    private String userEmail;

    @Column(nullable = false)
    private LocalDateTime expiryDate;

    private boolean revoked = false;
}

@Service
@RequiredArgsConstructor
@Transactional
public class RefreshTokenService {

    private final RefreshTokenRepository tokenRepository;

    @Value("${jwt.refresh-expiration:604800000}")
    private long refreshExpiration;

    public String createRefreshToken(String email) {
        // Revoke existing tokens for this user
        tokenRepository.revokeAllByUserEmail(email);

        RefreshToken token = new RefreshToken();
        token.setToken(UUID.randomUUID().toString());
        token.setUserEmail(email);
        token.setExpiryDate(LocalDateTime.now().plusSeconds(refreshExpiration / 1000));

        return tokenRepository.save(token).getToken();
    }

    public String validateAndExtractEmail(String tokenValue) {
        RefreshToken token = tokenRepository.findByToken(tokenValue)
            .orElseThrow(() -> new InvalidTokenException("Refresh token not found"));

        if (token.isRevoked()) {
            throw new InvalidTokenException("Refresh token has been revoked");
        }
        if (token.getExpiryDate().isBefore(LocalDateTime.now())) {
            tokenRepository.delete(token);
            throw new InvalidTokenException("Refresh token has expired");
        }

        return token.getUserEmail();
    }

    public void invalidateRefreshToken(String tokenValue) {
        tokenRepository.findByToken(tokenValue)
            .ifPresent(t -> { t.setRevoked(true); tokenRepository.save(t); });
    }
}
```

### 4.10 JWT Storage: localStorage vs HttpOnly Cookie

| Aspect | localStorage | HttpOnly Cookie |
|--------|-------------|----------------|
| **XSS vulnerability** | ⚠️ Accessible by JS — vulnerable to XSS | ✅ Not accessible by JS |
| **CSRF vulnerability** | ✅ Not sent automatically — safe from CSRF | ⚠️ Auto-sent by browser — needs CSRF protection |
| **Ease of use** | Simple JavaScript access | Requires server-side cookie management |
| **Mobile apps** | ✅ Straightforward | ⚠️ More complex |
| **Recommendation** | Avoid for sensitive tokens | **Preferred for web apps** |

> [!TIP]
> **Best practice**: Store access tokens in memory (JavaScript variable), store refresh tokens in HttpOnly cookies. This way XSS can steal the short-lived access token but cannot steal the long-lived refresh token.

### 4.11 JWT Vulnerabilities

**1. The `alg:none` Attack**
An attacker modifies the JWT header to `{"alg":"none"}` and removes the signature. A vulnerable library then accepts the token without verification.

```java
// VULNERABLE — never accept "none" algorithm
// SAFE — JJWT rejects "none" by default in modern versions
// Always use verifyWith() to require signature verification:
Jwts.parser()
    .verifyWith(signingKey)  // Requires signature — rejects alg:none
    .build()
    .parseSignedClaims(token);
```

**2. Weak Secret (HS256)**
```properties
# BAD — too short, dictionary-attackable
jwt.secret=mysecret

# GOOD — minimum 256 bits (32 bytes), cryptographically random
jwt.secret=7Hk2Pq9mNxR4vYwZ3cBjLsFdGeAuItOp6QnVkWmXyE1
```

**3. JWT Not Validated on Server (Trusting Client-Side)**
Always validate the JWT signature on every request. Never trust claims without verification.

**4. Sensitive Data in Payload**
JWT payload is Base64-encoded, not encrypted. Anyone can decode it.
```java
// BAD — password in JWT
.claim("password", user.getPassword())

// OK — only include non-sensitive identity claims
.subject(user.getEmail())
.claim("roles", user.getRoles())
```

---

## 5. SecurityFilterChain Configuration

### 5.1 HttpSecurity in Spring Boot 3.x

`WebSecurityConfigurerAdapter` was deprecated in Spring Security 5.7 and **removed in 6.x**. Use `SecurityFilterChain` beans instead.

```java
// ❌ OLD WAY (Spring Security 5.x and earlier) — DO NOT USE
@Configuration
public class OldSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().anyRequest().authenticated();
    }
}

// ✅ NEW WAY (Spring Security 6.x / Spring Boot 3.x)
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

### 5.2 Authorization Rules — requestMatchers

```java
http.authorizeHttpRequests(auth -> auth
    // 1. Permit specific paths without authentication
    .requestMatchers("/", "/home", "/about").permitAll()
    .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
    .requestMatchers("/api/auth/**").permitAll()

    // 2. Require specific roles
    .requestMatchers("/api/admin/**").hasRole("ADMIN")  // adds ROLE_ prefix
    .requestMatchers("/api/manager/**").hasAnyRole("ADMIN", "MANAGER")

    // 3. Require specific authorities
    .requestMatchers("/api/reports/**").hasAuthority("REPORT_READ")  // no prefix added

    // 4. Require authentication (any role)
    .requestMatchers("/api/**").authenticated()

    // 5. Deny all remaining (safety net)
    .anyRequest().denyAll()
);
```

> [!WARNING]
> Rules are evaluated **in order** — put more specific rules first. If you put `.anyRequest().authenticated()` before specific permit rules, those specific rules will never be reached.

### 5.3 CSRF — What It Is and When to Disable

**Cross-Site Request Forgery (CSRF)**: An attack where a malicious site tricks a user's browser into making authenticated requests to your site.

**How CSRF works:**
1. User logs into `bank.com` (session cookie stored)
2. User visits `evil.com`
3. `evil.com` has a hidden form that submits to `bank.com/transfer`
4. Browser automatically includes the session cookie → transfer executes!

**Spring Security's CSRF protection**: Generates a CSRF token that must be included in state-changing requests (POST, PUT, DELETE). Since `evil.com` can't read the CSRF token (same-origin policy), the attack fails.

```java
// CSRF enabled (default) — for traditional web apps
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
);

// CSRF disabled — acceptable for pure REST APIs with:
// 1. STATELESS session management
// 2. JWT or Basic auth (no cookies)
// 3. CORS properly configured
http.csrf(csrf -> csrf.disable());
```

> [!NOTE]
> **CSRF attacks only work if authentication relies on cookies (session-based).** If your REST API uses JWT in the `Authorization` header, CSRF is not a concern — the browser doesn't automatically include custom headers in cross-origin requests.

### 5.4 Session Management

```java
http.sessionManagement(session -> session
    // STATELESS — no session created, no session used (for REST + JWT)
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
    // Other options:
    // SessionCreationPolicy.ALWAYS      — always create session
    // SessionCreationPolicy.IF_REQUIRED — create if needed (default)
    // SessionCreationPolicy.NEVER       — never create, use if exists
);
```

**Session Fixation Protection** (for stateful apps):

```java
http.sessionManagement(session -> session
    .sessionFixation(fixation -> fixation.changeSessionId()) // default
    // .migrateSession()  — old behavior, creates new session and copies attributes
    // .newSession()      — new session, doesn't copy attributes (most secure)
    // .none()            — no protection (dangerous)
    .maximumSessions(1)   // prevent concurrent sessions
    .expiredSessionStrategy(event -> {
        event.getResponse().sendError(HttpStatus.UNAUTHORIZED.value(), "Session expired");
    })
);
```

### 5.5 CORS in Spring Security

> [!IMPORTANT]
> When using Spring Security, CORS must be configured through Spring Security's CORS support, not just with `@CrossOrigin` or `WebMvcConfigurer`. Otherwise, CORS preflight requests (`OPTIONS`) will be rejected by Spring Security before reaching your CORS configuration.

```java
// Method 1: CorsConfigurationSource bean (preferred)
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOriginPatterns(List.of("https://*.mycompany.com", "http://localhost:*"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setExposedHeaders(List.of("Authorization", "X-Total-Count"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}

// In SecurityFilterChain:
http.cors(cors -> cors.configurationSource(corsConfigurationSource()))
```

### 5.6 Security Headers

Spring Security automatically adds security headers:

```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.deny())  // X-Frame-Options: DENY
    .contentTypeOptions(withDefaults())   // X-Content-Type-Options: nosniff
    .xssProtection(xss -> xss.enable())  // X-XSS-Protection: 1; mode=block
    .httpStrictTransportSecurity(hsts -> hsts
        .maxAgeInSeconds(31536000)
        .includeSubDomains(true)
        .preload(true)
    )
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; script-src 'self'")
    )
);
```

---

## 6. Method Security

### 6.1 Enabling Method Security

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,  // @PreAuthorize, @PostAuthorize (default: true)
    securedEnabled = true,  // @Secured
    jsr250Enabled = true    // @RolesAllowed
)
public class MethodSecurityConfig {
    // No additional code needed
}
```

> [!NOTE]
> In Spring Security 6, `@EnableMethodSecurity` replaces the old `@EnableGlobalMethodSecurity`. The `prePostEnabled` flag defaults to `true` in Spring Security 6.

### 6.2 @PreAuthorize — The Most Powerful

```java
@Service
public class DocumentService {

    // Simple role check
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAllDocuments() { /* ... */ }

    // Multiple roles
    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    public List<Document> getAllDocuments() { /* ... */ }

    // Authority check
    @PreAuthorize("hasAuthority('DOCUMENT_WRITE')")
    public Document createDocument(Document doc) { /* ... */ }

    // Using method parameter in SpEL
    @PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
    public User getUserById(Long userId) { /* ... */ }

    // Checking the argument object's property
    @PreAuthorize("#document.ownerEmail == authentication.name or hasRole('ADMIN')")
    public void updateDocument(Document document) { /* ... */ }

    // Combining conditions
    @PreAuthorize("isAuthenticated() and @documentService.isOwner(#docId, authentication)")
    public void updateDocumentById(Long docId) { /* ... */ }

    // Custom SpEL with Spring beans (note the '@' prefix for beans)
    @PreAuthorize("@permissionEvaluator.hasPermission(authentication, #projectId, 'PROJECT', 'WRITE')")
    public void updateProject(Long projectId) { /* ... */ }
}
```

### 6.3 @PostAuthorize

`@PostAuthorize` runs **after** the method executes and can check the return value:

```java
@Service
public class ReportService {

    // Check the return value — returnObject is the result
    @PostAuthorize("returnObject.ownerEmail == authentication.name or hasRole('ADMIN')")
    public Report getReport(Long reportId) {
        return reportRepository.findById(reportId).orElseThrow();
        // If the report doesn't belong to the user, throws AccessDeniedException
        // AFTER the DB query executes
    }
}
```

> [!WARNING]
> `@PostAuthorize` still executes the method even if access is denied — it just throws an exception afterwards. Use `@PreAuthorize` when you want to prevent execution entirely (more efficient).

### 6.4 @Secured

```java
@Service
public class AdminService {

    // Note: requires securedEnabled = true in @EnableMethodSecurity
    @Secured("ROLE_ADMIN")  // Must include ROLE_ prefix
    public void performAdminAction() { /* ... */ }

    @Secured({"ROLE_ADMIN", "ROLE_MANAGER"})
    public List<User> getAllUsers() { /* ... */ }
}
```

`@Secured` is less flexible than `@PreAuthorize` — it doesn't support SpEL expressions.

### 6.5 @RolesAllowed (JSR-250)

```java
@Service
public class HrService {

    // Note: requires jsr250Enabled = true in @EnableMethodSecurity
    @RolesAllowed("ADMIN")  // Note: no ROLE_ prefix needed here
    public void accessHrData() { /* ... */ }

    @RolesAllowed({"ADMIN", "HR_MANAGER"})
    public List<Employee> getAllEmployees() { /* ... */ }
}
```

### 6.6 When Method Security Doesn't Work — Self-Invocation

> [!CAUTION]
> Method security uses **Spring AOP proxies**. When a method calls another method **within the same class**, the call bypasses the proxy — security annotations are **ignored**.

```java
@Service
public class OrderService {

    // This security check WORKS when called from outside the class
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long orderId) {
        // ...
    }

    public void processAndDelete(Long orderId) {
        // ⚠️ This call BYPASSES @PreAuthorize — no security check!
        this.deleteOrder(orderId);
        // Because 'this' refers to the actual object, not the proxy
    }
}
```

**Solutions:**

```java
// Solution 1: Inject the bean into itself (not clean but works)
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // Spring injects the proxy

    public void processAndDelete(Long orderId) {
        self.deleteOrder(orderId);  // Goes through the proxy — security checked!
    }
}

// Solution 2: Move the secured method to a separate service
@Service
public class OrderDeletionService {
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long orderId) { /* ... */ }
}

@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderDeletionService deletionService;

    public void processAndDelete(Long orderId) {
        deletionService.deleteOrder(orderId);  // Works — different bean
    }
}
```

### 6.7 Custom Permission Evaluator

For complex, domain-object-level permissions:

```java
@Component
public class AppPermissionEvaluator implements PermissionEvaluator {

    private final ProjectRepository projectRepository;

    @Override
    public boolean hasPermission(Authentication auth, Object targetDomainObject, Object permission) {
        if (targetDomainObject instanceof Project project) {
            return checkProjectPermission(auth, project, (String) permission);
        }
        return false;
    }

    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId,
                                  String targetType, Object permission) {
        if ("PROJECT".equals(targetType)) {
            Project project = projectRepository.findById((Long) targetId).orElse(null);
            if (project == null) return false;
            return checkProjectPermission(auth, project, (String) permission);
        }
        return false;
    }

    private boolean checkProjectPermission(Authentication auth, Project project, String permission) {
        String username = auth.getName();
        return switch (permission) {
            case "READ"  -> project.getMembers().contains(username) || hasRole(auth, "ADMIN");
            case "WRITE" -> project.getOwner().equals(username) || hasRole(auth, "ADMIN");
            case "DELETE"-> hasRole(auth, "ADMIN");
            default -> false;
        };
    }

    private boolean hasRole(Authentication auth, String role) {
        return auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_" + role));
    }
}

// Register it
@Bean
public MethodSecurityExpressionHandler methodSecurityExpressionHandler(
        AppPermissionEvaluator evaluator) {
    DefaultMethodSecurityExpressionHandler handler = new DefaultMethodSecurityExpressionHandler();
    handler.setPermissionEvaluator(evaluator);
    return handler;
}

// Usage:
@PreAuthorize("hasPermission(#projectId, 'PROJECT', 'WRITE')")
public void updateProject(Long projectId, ProjectUpdateDto dto) { /* ... */ }
```

---

## 7. OAuth2 Overview

### 7.1 OAuth2 Roles

| Role | Description | Example |
|------|-------------|---------|
| **Resource Owner** | The user | End user |
| **Client** | App requesting access | Your web app |
| **Authorization Server** | Issues tokens | Google, Okta, Keycloak |
| **Resource Server** | Hosts protected resources | Your REST API |

### 7.2 OAuth2 Flows

**Authorization Code Flow (most secure, for web apps):**

```
User → Client App → Authorization Server (login page)
Authorization Server → Client App (code)
Client App → Authorization Server (code + secret → tokens)
Client App → Resource Server (access token)
```

**Client Credentials Flow (machine-to-machine):**

```
Service A → Authorization Server (client_id + client_secret → access token)
Service A → Service B (access token)
```

**Implicit Flow** — **Deprecated** in OAuth 2.1. Tokens were returned directly in the URL fragment, which is insecure.

### 7.3 Spring Boot OAuth2 Resource Server

Your REST API acts as a **Resource Server** — it receives and validates JWT tokens issued by an Authorization Server:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```properties
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Option 1: JWKS endpoint (RS256) — recommended for production
          jwk-set-uri: https://auth-server.example.com/.well-known/jwks.json
          # Option 2: Static symmetric secret (HS256)
          # secret: your-secret-key
          issuer-uri: https://auth-server.example.com
```

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
            );
        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthorityPrefix("ROLE_");
        authoritiesConverter.setAuthoritiesClaimName("roles"); // custom claim name

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return converter;
    }
}
```

### 7.4 Spring Boot OAuth2 Client

When your app needs to **call** protected APIs on behalf of a user:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
        provider:
          google:
            issuer-uri: https://accounts.google.com
```

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/", "/error").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2Login(oauth2 -> oauth2
            .loginPage("/login")
            .defaultSuccessUrl("/dashboard")
            .userInfoEndpoint(userInfo -> userInfo
                .userService(customOAuth2UserService())
            )
        )
        .logout(logout -> logout.logoutSuccessUrl("/"));
    return http.build();
}
```

---

## 8. Role vs Authority vs Permission

### 8.1 The Key Distinction

In Spring Security, **everything** is an `Authority` (a `GrantedAuthority`). Roles are just authorities with a `ROLE_` prefix convention.

```java
// Both stored as GrantedAuthority
new SimpleGrantedAuthority("ROLE_ADMIN");    // Role
new SimpleGrantedAuthority("USER_READ");      // Authority/Permission
new SimpleGrantedAuthority("SCOPE_openid");  // OAuth2 Scope
```

### 8.2 hasRole() vs hasAuthority()

This is one of the most common interview gotchas:

```java
// hasRole() AUTOMATICALLY adds "ROLE_" prefix
.hasRole("ADMIN")      // checks for "ROLE_ADMIN" authority
.hasRole("USER")       // checks for "ROLE_USER" authority

// hasAuthority() uses the EXACT string — no prefix added
.hasAuthority("ROLE_ADMIN")   // same as hasRole("ADMIN")
.hasAuthority("USER_READ")    // no prefix added
.hasAuthority("ADMIN")        // NOT the same as hasRole("ADMIN")!
```

```java
// Granting authorities
UserDetails user = User.builder()
    .username("john")
    .password(encoded)
    .roles("ADMIN", "USER")         // stored as ROLE_ADMIN, ROLE_USER
    .authorities("REPORT_READ",     // stored as-is
                 "ROLE_MANAGER")    // stored as-is (you added ROLE_ manually)
    .build();

// In @PreAuthorize
@PreAuthorize("hasRole('ADMIN')")        // ✅ checks ROLE_ADMIN
@PreAuthorize("hasAuthority('ADMIN')")   // ❌ checks ADMIN (no prefix!) — FAILS
@PreAuthorize("hasAuthority('ROLE_ADMIN')")  // ✅ checks ROLE_ADMIN
@PreAuthorize("hasAuthority('REPORT_READ')") // ✅ checks REPORT_READ
```

### 8.3 Designing a Fine-Grained Permission System

```java
// Enum-based permissions
public enum Permission {
    USER_READ("user:read"),
    USER_WRITE("user:write"),
    USER_DELETE("user:delete"),
    REPORT_READ("report:read"),
    REPORT_WRITE("report:write");

    private final String permission;
    Permission(String permission) { this.permission = permission; }
    public String getPermission() { return permission; }
}

// Role-to-Permission mapping
public enum Role {
    USER(Set.of(Permission.USER_READ)),
    MANAGER(Set.of(Permission.USER_READ, Permission.REPORT_READ, Permission.REPORT_WRITE)),
    ADMIN(Set.of(Permission.values()));  // all permissions

    private final Set<Permission> permissions;

    public List<GrantedAuthority> getGrantedAuthorities() {
        List<GrantedAuthority> authorities = permissions.stream()
            .map(p -> new SimpleGrantedAuthority(p.getPermission()))
            .collect(Collectors.toList());
        // Also add the role itself
        authorities.add(new SimpleGrantedAuthority("ROLE_" + this.name()));
        return authorities;
    }
}

// UserDetailsService using this model
@Override
public UserDetails loadUserByUsername(String username) {
    User user = userRepository.findByEmail(username).orElseThrow(...);
    return org.springframework.security.core.userdetails.User.builder()
        .username(user.getEmail())
        .password(user.getPasswordHash())
        .authorities(user.getRole().getGrantedAuthorities())
        .build();
}

// Usage in controllers
@PreAuthorize("hasAuthority('user:write')")
public void createUser(UserDto dto) { /* ... */ }

@PreAuthorize("hasAuthority('report:read') or hasRole('ADMIN')")
public List<Report> getReports() { /* ... */ }
```

---

## 9. Common Security Vulnerabilities

### 9.1 SQL Injection — How Spring Data Protects

SQL Injection occurs when user input is embedded directly in SQL queries.

```java
// ❌ VULNERABLE — never do this
@Repository
public class VulnerableUserRepo {
    @PersistenceContext
    private EntityManager em;

    public User findByUsername(String username) {
        // Attacker sends: ' OR '1'='1
        String query = "SELECT u FROM User u WHERE u.username = '" + username + "'";
        return (User) em.createQuery(query).getSingleResult();
    }
}

// ✅ SAFE — Spring Data JPA uses parameterized queries by default
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username); // Uses parameterized query internally

    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmail(@Param("email") String email); // Named parameter
}

// ✅ SAFE — Native query with parameter
@Query(value = "SELECT * FROM users WHERE email = :email", nativeQuery = true)
Optional<User> findByEmailNative(@Param("email") String email);
```

### 9.2 XSS — Cross-Site Scripting Prevention

XSS allows attackers to inject malicious scripts into pages viewed by other users.

```java
// Spring Security adds X-XSS-Protection header by default

// For server-side rendered content, use Thymeleaf (auto-escapes by default)
// th:text="${userInput}" — safely escaped
// th:utext="${userInput}" — UNSAFE, renders raw HTML

// For REST APIs, validate and sanitize inputs
@PostMapping("/comments")
public ResponseEntity<Comment> addComment(
        @RequestBody @Valid CommentRequest request) {
    // Use a sanitization library like OWASP Java HTML Sanitizer
    String sanitized = Sanitizers.FORMATTING.and(Sanitizers.LINKS)
        .sanitize(request.getContent());
    // ...
}
```

```xml
<dependency>
    <groupId>com.googlecode.owasp-java-html-sanitizer</groupId>
    <artifactId>owasp-java-html-sanitizer</artifactId>
    <version>20220608.1</version>
</dependency>
```

### 9.3 CSRF — Already Covered in Section 5.3

Quick summary: enabled by default, safe to disable only for stateless REST APIs using JWT/Bearer tokens.

### 9.4 Insecure Direct Object Reference (IDOR)

IDOR occurs when an API exposes internal object IDs that users can manipulate to access other users' data.

```java
// ❌ VULNERABLE — any authenticated user can access any order
@GetMapping("/api/orders/{orderId}")
public ResponseEntity<Order> getOrder(@PathVariable Long orderId) {
    return ResponseEntity.ok(orderRepository.findById(orderId).orElseThrow());
}

// ✅ SAFE — verify the order belongs to the authenticated user
@GetMapping("/api/orders/{orderId}")
public ResponseEntity<Order> getOrder(
        @PathVariable Long orderId,
        @AuthenticationPrincipal UserDetails userDetails) {

    Order order = orderRepository.findById(orderId).orElseThrow();

    if (!order.getUserEmail().equals(userDetails.getUsername()) &&
        !userDetails.getAuthorities().contains(new SimpleGrantedAuthority("ROLE_ADMIN"))) {
        throw new AccessDeniedException("You don't have access to this order");
    }

    return ResponseEntity.ok(order);
}

// Or use @PostAuthorize
@PostAuthorize("returnObject.body.userEmail == authentication.name or hasRole('ADMIN')")
@GetMapping("/api/orders/{orderId}")
public ResponseEntity<Order> getOrder(@PathVariable Long orderId) {
    return ResponseEntity.ok(orderRepository.findById(orderId).orElseThrow());
}

// Best: Use @PreAuthorize with custom permission evaluator
@PreAuthorize("@orderSecurity.canAccess(authentication, #orderId)")
@GetMapping("/api/orders/{orderId}")
public ResponseEntity<Order> getOrder(@PathVariable Long orderId) {
    return ResponseEntity.ok(orderRepository.findById(orderId).orElseThrow());
}
```

### 9.5 Mass Assignment

Mass assignment occurs when API accepts and applies all fields from user input, including privileged fields.

```java
// ❌ VULNERABLE — user can set isAdmin: true in JSON
@PutMapping("/api/users/{id}")
public User updateUser(@PathVariable Long id, @RequestBody User user) {
    user.setId(id);
    return userRepository.save(user);  // Saves whatever the client sent!
}

// ✅ SAFE — use dedicated DTOs that only expose allowed fields
public record UpdateUserRequest(
    @NotBlank String firstName,
    @NotBlank String lastName,
    @Email String email
    // Notice: no 'role', no 'isAdmin', no 'id' — never accept these from client
) {}

@PutMapping("/api/users/{id}")
public ResponseEntity<UserDto> updateUser(
        @PathVariable Long id,
        @RequestBody @Valid UpdateUserRequest request,
        @AuthenticationPrincipal UserDetails currentUser) {

    User user = userRepository.findById(id).orElseThrow();
    user.setFirstName(request.firstName());
    user.setLastName(request.lastName());
    user.setEmail(request.email());
    // Role/admin status NOT touched — controlled server-side
    return ResponseEntity.ok(userMapper.toDto(userRepository.save(user)));
}
```

### 9.6 Security Misconfiguration Patterns

```java
// ❌ Anti-pattern 1: Permitting all in development and forgetting to tighten
http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll()); // Never in production!

// ❌ Anti-pattern 2: Weak JWT secret in properties file
# application.properties
jwt.secret=secret  # Too short, committed to VCS

// ✅ Use environment variables or secrets manager
jwt.secret=${JWT_SECRET}  # From environment variable

// ❌ Anti-pattern 3: Logging sensitive information
log.info("User login attempt: {} / {}", username, password);  // NEVER log passwords!

// ❌ Anti-pattern 4: Exposing all actuator endpoints
management.endpoints.web.exposure.include=*  # Dangerous!

// ✅ Expose only necessary endpoints
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.show-details=never  # Don't show internals to public
```

---

## 10. Security Testing

### 10.1 Test Dependencies

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 10.2 @WithMockUser

The simplest way to simulate an authenticated user:

```java
@SpringBootTest
@AutoConfigureMockMvc
class DocumentControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @WithMockUser(username = "user@test.com", roles = {"USER"})
    void shouldReturnDocumentsForAuthenticatedUser() throws Exception {
        mockMvc.perform(get("/api/documents"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$").isArray());
    }

    @Test
    @WithMockUser(username = "admin@test.com", roles = {"ADMIN"})
    void shouldAllowAdminToDeleteDocument() throws Exception {
        mockMvc.perform(delete("/api/documents/1"))
            .andExpect(status().isNoContent());
    }

    @Test
    void shouldReturn401ForUnauthenticatedUser() throws Exception {
        mockMvc.perform(get("/api/documents"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = {"USER"})  // NOT admin
    void shouldReturn403ForUnauthorizedUser() throws Exception {
        mockMvc.perform(delete("/api/documents/1"))
            .andExpect(status().isForbidden());
    }

    // Testing with specific authorities (not roles)
    @Test
    @WithMockUser(authorities = {"document:read", "ROLE_USER"})
    void shouldAllowUserWithReadAuthority() throws Exception {
        mockMvc.perform(get("/api/documents/1"))
            .andExpect(status().isOk());
    }
}
```

### 10.3 @WithUserDetails

Uses the actual `UserDetailsService` to load a real user from your test data:

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setup() {
        // Create a real user in the test database
        User user = new User("testuser@example.com", encodedPassword, Role.USER);
        userRepository.save(user);
    }

    @Test
    @WithUserDetails(value = "testuser@example.com",
                     userDetailsServiceBeanName = "customUserDetailsService")
    void shouldLoadRealUserDetails() throws Exception {
        mockMvc.perform(get("/api/orders"))
            .andExpect(status().isOk());
        // Real UserDetails used — custom fields available in SecurityContext
    }
}
```

### 10.4 MockMvc Security Testing — Full Examples

```java
@SpringBootTest
@AutoConfigureMockMvc
class AuthControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void shouldLoginSuccessfully() throws Exception {
        LoginRequest request = new LoginRequest("user@test.com", "password123");

        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.accessToken").isNotEmpty())
            .andExpect(jsonPath("$.refreshToken").isNotEmpty());
    }

    @Test
    void shouldRejectInvalidCredentials() throws Exception {
        LoginRequest request = new LoginRequest("user@test.com", "wrongPassword");

        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void shouldAccessProtectedEndpointWithJwt() throws Exception {
        // First, get a token
        String token = getToken("user@test.com", "password123");

        mockMvc.perform(get("/api/profile")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }

    private String getToken(String email, String password) throws Exception {
        LoginRequest request = new LoginRequest(email, password);
        String response = mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andReturn().getResponse().getContentAsString();
        return objectMapper.readTree(response).get("accessToken").asText();
    }
}
```

### 10.5 Custom Security Mock Annotation

```java
// Create a custom annotation for DRY tests
@Retention(RetentionPolicy.RUNTIME)
@WithMockUser(username = "admin@test.com", roles = {"ADMIN", "USER"})
public @interface WithMockAdmin {}

@Retention(RetentionPolicy.RUNTIME)
@WithMockUser(username = "user@test.com", roles = {"USER"},
              authorities = {"user:read", "report:read"})
public @interface WithMockRegularUser {}

// Usage — much cleaner
@Test
@WithMockAdmin
void adminCanAccessDashboard() throws Exception {
    mockMvc.perform(get("/api/admin/dashboard"))
        .andExpect(status().isOk());
}

@Test
@WithMockRegularUser
void regularUserCannotAccessAdmin() throws Exception {
    mockMvc.perform(get("/api/admin/dashboard"))
        .andExpect(status().isForbidden());
}
```

### 10.6 Testing Method Security

```java
@SpringBootTest
class DocumentServiceSecurityTest {

    @Autowired
    private DocumentService documentService;

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminShouldDeleteDocument() {
        assertDoesNotThrow(() -> documentService.deleteDocument(1L));
    }

    @Test
    @WithMockUser(roles = "USER")
    void regularUserShouldNotDeleteDocument() {
        assertThrows(AccessDeniedException.class,
            () -> documentService.deleteDocument(1L));
    }

    @Test
    void unauthenticatedUserShouldNotDeleteDocument() {
        assertThrows(AuthenticationCredentialsNotFoundException.class,
            () -> documentService.deleteDocument(1L));
    }
}
```

---

## ⚠️ Interview Traps & Gotchas

### Trap 1: The ROLE_ Prefix Trap

```java
// This will FAIL if the user has authority "ROLE_ADMIN"
@PreAuthorize("hasAuthority('ADMIN')")  // ❌ Checks for "ADMIN" not "ROLE_ADMIN"

// This will WORK
@PreAuthorize("hasRole('ADMIN')")        // ✅ Checks for "ROLE_ADMIN"
@PreAuthorize("hasAuthority('ROLE_ADMIN')") // ✅ Checks for "ROLE_ADMIN"

// When granting via .roles()
User.withDefaultPasswordEncoder()
    .username("admin")
    .password("pass")
    .roles("ADMIN")  // Internally stores as "ROLE_ADMIN"
    .build();
```

**Interview answer**: `hasRole("ADMIN")` automatically prepends `ROLE_` and checks for `ROLE_ADMIN`. `hasAuthority("ADMIN")` checks for the exact string `ADMIN`. If authorities are stored as `ROLE_ADMIN`, then `hasAuthority("ADMIN")` will always fail.

---

### Trap 2: CSRF Disabled for REST — Is It Always OK?

**Answer**: Only safe when ALL of these are true:
1. Session management is `STATELESS` (no cookies)
2. Authentication uses `Authorization` header (JWT/Bearer), not session cookies
3. CORS is properly configured to restrict origins

If your REST API uses session-based authentication (form login + cookies), CSRF must be enabled even for REST APIs.

---

### Trap 3: JWT Secret in application.properties

```properties
# ❌ NEVER DO THIS — committed to version control!
jwt.secret=my-super-secret-key-that-everyone-can-see

# ✅ Use environment variable
jwt.secret=${JWT_SECRET}

# ✅ Or use Spring Cloud Config / AWS Secrets Manager / Vault
```

**Why it matters**: If the JWT secret is compromised, an attacker can forge valid JWT tokens for any user, including admins.

---

### Trap 4: SecurityContext in @Async

```java
@Async
public CompletableFuture<Report> generateReport() {
    // ❌ This is NULL by default — SecurityContext is not propagated
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    String username = auth.getName(); // NullPointerException!
    // ...
}
```

**Fix**: Use `DelegatingSecurityContextExecutor` or `MODE_INHERITABLETHREADLOCAL`:

```java
// Option 1: Change the strategy globally
@PostConstruct
public void init() {
    SecurityContextHolder.setStrategyName(
        SecurityContextHolder.MODE_INHERITABLETHREADLOCAL
    );
}

// Option 2: DelegatingSecurityContextExecutor (preferred, explicit)
@Bean
public Executor asyncExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.initialize();
    return new DelegatingSecurityContextExecutor(executor.getObject());
}
```

---

### Trap 5: Filter Order — Spring Security Filter Position

Spring Security's `FilterChainProxy` is registered at a specific order in the filter chain. By default it's at order `SecurityProperties.DEFAULT_FILTER_ORDER = -100`. This means it runs very early.

**Common trap**: Adding a custom `Filter` bean and expecting it to run before Spring Security:

```java
// ❌ This filter might run AFTER Spring Security's filter
@Component
@Order(1)
public class MyLoggingFilter implements Filter { /* ... */ }

// ✅ Explicitly register before Spring Security
@Bean
public FilterRegistrationBean<MyLoggingFilter> loggingFilter() {
    FilterRegistrationBean<MyLoggingFilter> bean = new FilterRegistrationBean<>();
    bean.setFilter(new MyLoggingFilter());
    bean.setOrder(SecurityProperties.DEFAULT_FILTER_ORDER - 1); // Before security
    return bean;
}
```

---

### Trap 6: Why Session Should Be STATELESS for REST APIs

Common misconception: "STATELESS just means we don't use sessions, but it doesn't hurt to leave session management at IF_REQUIRED."

**Wrong!** If you leave session creation enabled, Spring Security will create sessions and set `JSESSIONID` cookies. Then:
- Your API looks stateless but isn't
- Horizontal scaling requires sticky sessions or session replication
- Clients can accidentally become stateful

Always explicitly set `STATELESS` for REST APIs:

```java
http.sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

---

### Trap 7: @PreAuthorize on Private Methods

```java
@Service
public class PaymentService {

    // ❌ THIS ANNOTATION DOES NOTHING
    @PreAuthorize("hasRole('ADMIN')")
    private void processRefund(Long paymentId) {
        // Not secured! AOP proxy only wraps public methods
    }

    // ✅ Must be public for AOP to intercept
    @PreAuthorize("hasRole('ADMIN')")
    public void processRefundPublic(Long paymentId) {
        // This IS secured
    }
}
```

**Also**: Even if you make it public, if it's called from within the same class via `this.processRefundPublic()`, the proxy is bypassed — security is not applied.

---

### Trap 8: ExceptionTranslationFilter — Order Matters

`ExceptionTranslationFilter` catches `AccessDeniedException` and `AuthenticationException` and translates them to HTTP responses. It ONLY catches exceptions thrown downstream. If you throw an exception in a filter that runs AFTER `ExceptionTranslationFilter`, it won't be caught.

---

### Trap 9: Multiple SecurityFilterChain — Only First Match Wins

```java
@Bean
@Order(1)
public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .csrf(csrf -> csrf.disable())
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
    return http.build();
}

@Bean
@Order(2)
public SecurityFilterChain webChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/**")  // Matches everything else
        .formLogin(withDefaults())
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
    return http.build();
}
// Requests to /api/** → apiChain (Order 1)
// All other requests → webChain (Order 2)
```

---

### Trap 10: Password Encoding During User Creation

```java
// ❌ Storing raw password in database!
User user = new User(email, rawPassword);
userRepository.save(user);

// ✅ Always encode before saving
@Service
@RequiredArgsConstructor
public class UserService {
    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;

    public User createUser(String email, String rawPassword) {
        String encodedPassword = passwordEncoder.encode(rawPassword);
        User user = new User(email, encodedPassword);
        return userRepository.save(user);
    }
}
```

---

## 🔍 Deep Dive — Internal Mechanics

### How AuthenticationManager.authenticate() Works End-to-End

When a login request hits `UsernamePasswordAuthenticationFilter`:

1. **Filter extracts credentials** → creates `UsernamePasswordAuthenticationToken(username, password)` — **not authenticated** (second constructor param is null)

2. **Calls `authenticationManager.authenticate(token)`** → goes to `ProviderManager`

3. **`ProviderManager` iterates providers**:
   - Finds `DaoAuthenticationProvider` (supports `UsernamePasswordAuthenticationToken`)
   - Calls `DaoAuthenticationProvider.authenticate(token)`

4. **`DaoAuthenticationProvider` logic**:
   ```java
   // Simplified
   UserDetails user = userDetailsService.loadUserByUsername(username);
   if (!passwordEncoder.matches(rawPassword, user.getPassword())) {
       throw new BadCredentialsException("Wrong password");
   }
   // Create authenticated token
   return new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
   ```

5. **Returns authenticated `Authentication`** → stored in `SecurityContextHolder`

6. **`SecurityContextPersistenceFilter`** (or `SecurityContextHolderFilter` in Spring Security 6) saves the `SecurityContext` to the session (if stateful) or does nothing (if stateless)

### How JWT Filter Works Alongside DaoAuthenticationProvider

When using JWT:
- `JwtAuthenticationFilter` runs **before** `UsernamePasswordAuthenticationFilter`
- If a valid JWT is found, it sets authentication in `SecurityContextHolder`
- `UsernamePasswordAuthenticationFilter` sees that authentication is already set and skips
- `AuthorizationFilter` at the end checks the authorities from the JWT-populated authentication

### SecurityContextHolder Internal Strategy Pattern

```java
// SecurityContextHolder internally delegates to a strategy
public class SecurityContextHolder {
    private static SecurityContextHolderStrategy strategy;

    static {
        // Default: ThreadLocalSecurityContextHolderStrategy
        strategy = new ThreadLocalSecurityContextHolderStrategy();
    }

    public static SecurityContext getContext() {
        return strategy.getContext();
    }
}

// ThreadLocalSecurityContextHolderStrategy uses ThreadLocal
class ThreadLocalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {
    private static final ThreadLocal<Supplier<SecurityContext>> contextHolder =
        new ThreadLocal<>();

    public SecurityContext getContext() {
        return getDeferredContext().get();
    }
}
```

### Authorization Decision Making — Spring Security 6

In Spring Security 6, authorization is handled by `AuthorizationFilter` (replaced the old `FilterSecurityInterceptor`):

```java
// Simplified AuthorizationFilter logic
public class AuthorizationFilter extends GenericFilterBean {

    private final AuthorizationManager<HttpServletRequest> authorizationManager;

    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        HttpServletRequest request = (HttpServletRequest) req;

        AuthorizationDecision decision = authorizationManager.check(
            SecurityContextHolder::getContext,
            request
        );

        if (decision != null && !decision.isGranted()) {
            throw new AccessDeniedException("Access Denied");
        }

        chain.doFilter(req, res);
    }
}
```

The `AuthorizationManager` evaluates `requestMatchers` rules in order to decide whether to grant access.

---

## 📝 Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| **Filter Chain** | Every request passes through `DelegatingFilterProxy → FilterChainProxy → SecurityFilterChain` |
| **WebSecurityConfigurerAdapter** | **Removed in Spring Security 6** — use `SecurityFilterChain` beans |
| **Authentication vs Authorization** | Who are you? vs What can you do? |
| **SecurityContext** | Stored in `ThreadLocal` — not auto-propagated to async threads |
| **BCrypt** | Always use for password encoding — never store raw passwords |
| **JWT** | Stateless — store roles in claims, use RS256 for microservices |
| **CSRF** | Disable only for stateless REST APIs using JWT/Bearer tokens |
| **hasRole vs hasAuthority** | `hasRole("ADMIN")` checks `ROLE_ADMIN`; `hasAuthority("ADMIN")` checks `ADMIN` |
| **@PreAuthorize** | Only works on **public** methods in Spring-managed beans called via proxy |
| **Method security self-invocation** | Bypass AOP proxy — security annotations ignored for same-class calls |
| **JWT secret** | Never in `application.properties` — use environment variables or secrets manager |
| **STATELESS sessions** | Always set for REST APIs to prevent accidental stateful behavior |
| **OAuth2 Resource Server** | Use `spring-boot-starter-oauth2-resource-server` + JWKS URI for production |
| **IDOR protection** | Always verify ownership server-side — never trust the client-sent ID |
| **Mass assignment** | Always use dedicated DTOs — never bind request body directly to entities |

---

## ❓ Practice Questions

### Conceptual

1. Explain the difference between `DelegatingFilterProxy` and `FilterChainProxy`. Why does the indirection exist?

2. A request comes in with a valid JWT. Trace the exact path through Spring Security's filter chain, naming specific filters.

3. Why is `@PreAuthorize` on a private method ineffective? What is the underlying mechanism?

4. You have a microservices architecture. Why should you use RS256 instead of HS256 for JWT signing? What are the operational advantages?

5. Explain the difference between `SessionCreationPolicy.STATELESS` and `SessionCreationPolicy.NEVER`. When would you use each?

6. How does Spring Security's CSRF protection work? Why is it safe to disable for stateless REST APIs?

7. What happens to the `SecurityContext` when using `@Async`? How do you fix this?

### Scenario-Based

8. Your team reports that `@PreAuthorize("hasRole('ADMIN')")` is not working even when the user has `ROLE_ADMIN`. What could be the cause?

9. You're migrating from `WebSecurityConfigurerAdapter` to the new Spring Security 6 style. What key changes do you make?

10. A user complains they can access another user's invoice by changing the ID in the URL. What vulnerability is this, and how do you fix it?

11. Your JWT authentication works fine for regular requests but fails in `@Async` service methods where `SecurityContextHolder.getContext().getAuthentication()` returns null. Fix this.

12. You need two separate security configurations: one for your REST API (`/api/**`, stateless JWT) and one for your admin UI (`/admin/**`, form login). How do you implement this?

### Code Review

13. Review this code and identify all security issues:
```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userRepo.findById(id).orElseThrow();
    }

    @PutMapping("/users/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody User user) {
        user.setId(id);
        return userRepo.save(user);
    }
}
```

14. What is wrong with this SecurityConfig?
```java
http.authorizeHttpRequests(auth -> auth
    .anyRequest().authenticated()
    .requestMatchers("/api/public/**").permitAll()
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
);
```

15. Is this JWT configuration production-ready? What's missing?
```properties
jwt.secret=secret123
jwt.expiration=86400000
```

---

> [!TIP]
> **Interview Pro Tip**: When asked about Spring Security, always mention that `WebSecurityConfigurerAdapter` is removed in Spring Boot 3.x / Spring Security 6.x. This immediately signals to interviewers that you're working with modern Spring. Then explain the `SecurityFilterChain` bean approach.

> [!NOTE]
> **Further Reading**:
> - [Spring Security Reference Documentation](https://docs.spring.io/spring-security/reference/index.html)
> - [OWASP Top Ten](https://owasp.org/www-project-top-ten/)
> - [JWT.io](https://jwt.io/) — JWT debugger and documentation
> - [Spring Security Architecture Guide](https://spring.io/guides/topicals/spring-security-architecture)
