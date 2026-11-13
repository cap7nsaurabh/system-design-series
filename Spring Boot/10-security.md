# 10 — Spring Security in Spring Boot

> A comprehensive, production-grade guide to Spring Security 6.x / Spring Boot 3.x architecture, authentication paradigms, stateless JWT, OAuth2 / OIDC resource servers, multi-chain security filters, method-level authorization, password hashing, and defense against OWASP vulnerabilities.

---

## Table of Contents

1. [Spring Security 6.x Architecture & Filter Lifecycle](#1-spring-security-6x-architecture--filter-lifecycle)
   - [1.1 The Servlet Filter Pipeline (`DelegatingFilterProxy` & `FilterChainProxy`)](#11-the-servlet-filter-pipeline-delegatingfilterproxy--filterchainproxy)
   - [1.2 Modern `SecurityFilterChain` Lambda DSL (Removal of `WebSecurityConfigurerAdapter`)](#12-modern-securityfilterchain-lambda-dsl-removal-of-websecurityconfigureradapter)
   - [1.3 `SecurityContext` & `SecurityContextHolder` Lifecycle](#13-securitycontext--securitycontextholder-lifecycle)
2. [Multi-Approach: Authentication Paradigms & Protocols](#2-multi-approach-authentication-paradigms--protocols)
   - [Approach 1: Stateless JSON Web Token (JWT) with JJWT / Nimbus](#approach-1-stateless-json-web-token-jwt-with-jjwt--nimbus)
   - [Approach 2: OAuth2 / OIDC Resource Server with Remote JWKS Endpoint](#approach-2-oauth2--oidc-resource-server-with-remote-jwks-endpoint)
   - [Approach 3: Stateful Session-Based Form Login with HTTP-Only Cookies](#approach-3-stateful-session-based-form-login-with-http-only-cookies)
   - [Approach 4: HTTP Basic Authentication & Custom API Key Headers](#approach-4-http-basic-authentication--custom-api-key-headers)
   - [Authentication Paradigms Comparison Matrix](#authentication-paradigms-comparison-matrix)
3. [Multi-Approach: Password Hashing & Credential Storage Strategies](#3-multi-approach-password-hashing--credential-storage-strategies)
   - [Approach 1: Adaptive BCrypt (`BCryptPasswordEncoder`)](#approach-1-adaptive-bcrypt-bcryptpasswordencoder)
   - [Approach 2: Memory-Hard Argon2 (`Argon2PasswordEncoder`)](#approach-2-memory-hard-argon2-argon2passwordencoder)
   - [Approach 3: Zero-Downtime Migration via `DelegatingPasswordEncoder`](#approach-3-zero-downtime-migration-via-delegatingpasswordencoder)
   - [Password Encoders Comparison Matrix](#password-encoders-comparison-matrix)
4. [Multi-Approach: Authorization & Access Control Models](#4-multi-approach-authorization--access-control-models)
   - [Approach 1: HTTP-Level URL Rules via `requestMatchers`](#approach-1-http-level-url-rules-via-requestmatchers)
   - [Approach 2: Method-Level Annotations (`@PreAuthorize`, `@PostAuthorize`, `@Secured`)](#approach-2-method-level-annotations-preauthorize-postauthorize-secured)
   - [Approach 3: Domain Object-Level Security with Custom `PermissionEvaluator`](#approach-3-domain-object-level-security-with-custom-permissionevaluator)
   - [Approach 4: Hierarchical Role & Authority Modeling (`RoleHierarchy`)](#approach-4-hierarchical-role--authority-modeling-rolehierarchy)
   - [Authorization Models Comparison Matrix](#authorization-models-comparison-matrix)
5. [Multi-Approach: Multi-Tenant & Multi-Chain Security Architecture](#5-multi-approach-multi-tenant--multi-chain-security-architecture)
   - [Approach 1: Ordered Multiple `SecurityFilterChain` Beans (`@Order`)](#approach-1-ordered-multiple-securityfilterchain-beans-order)
   - [Approach 2: Dynamic Multi-Tenant JWT Issuer Resolution (`JwtIssuerAuthenticationManagerResolver`)](#approach-2-dynamic-multi-tenant-jwt-issuer-resolution-jwtissuerauthenticationmanagerresolver)
   - [Multi-Chain Architecture Comparison Matrix](#multi-chain-architecture-comparison-matrix)
6. [Multi-Approach: CSRF Protection Across Client Topologies](#6-multi-approach-csrf-protection-across-client-topologies)
   - [Approach 1: Disabled CSRF (`csrf.disable()`) for Pure Stateless REST](#approach-1-disabled-csrf-csrfdisable-for-pure-stateless-rest)
   - [Approach 2: Cookie-Based Dual-Submit CSRF Repository (Angular / React SPAs)](#approach-2-cookie-based-dual-submit-csrf-repository-angular--react-spas)
   - [Approach 3: Session-Based CSRF for Server-Side Rendering (Thymeleaf)](#approach-3-session-based-csrf-for-server-side-rendering-thymeleaf)
   - [CSRF Strategies Comparison Matrix](#csrf-strategies-comparison-matrix)
7. [Mitigating OWASP Top 10 Vulnerabilities in Spring Boot](#7-mitigating-owasp-top-10-vulnerabilities-in-spring-boot)
   - [7.1 Broken Object Level Authorization (IDOR)](#71-broken-object-level-authorization-idor)
   - [7.2 Mass Assignment & DTO Isolation](#72-mass-assignment--dto-isolation)
   - [7.3 Cross-Origin Resource Sharing (CORS) Misconfiguration](#73-cross-origin-resource-sharing-cors-misconfiguration)
   - [7.4 Security Headers (HSTS, CSP, X-Frame-Options)](#74-security-headers-hsts-csp-x-frame-options)
8. [Production Security Testing (`@WithMockUser`, `@WithUserDetails`, MockMvc)](#8-production-security-testing-withmockuser-withuserdetails-mockmvc)
9. [Interview Traps, Anti-Patterns, & Gotchas](#9-interview-traps-anti-patterns--gotchas)
10. [Summary & Architectural Decision Trees](#10-summary--architectural-decision-trees)
11. [Practice Questions & Interview Scenarios](#11-practice-questions--interview-scenarios)

---

## 1. Spring Security 6.x Architecture & Filter Lifecycle

### 1.1 The Servlet Filter Pipeline (`DelegatingFilterProxy` & `FilterChainProxy`)

Spring Security is implemented entirely as a standard Java EE / Jakarta EE Servlet Filter chain. The integration between the servlet container lifecycle (Tomcat/Jetty) and Spring's `ApplicationContext` is mediated by two foundational classes:

```
Incoming HTTP Request
        │
        ▼
┌────────────────────────────────────────────────────────────────────────┐
│                      SERVLET CONTAINER (Tomcat)                        │
│                                                                        │
│   ┌──────────────────────────────────────────────────────────────┐     │
│   │                    DelegatingFilterProxy                     │     │
│   │         (Standard web.xml / Spring Boot Servlet Filter)      │     │
│   └──────────────────────────────┬───────────────────────────────┘     │
│                                  │ delegates to target bean            │
│   ┌──────────────────────────────▼───────────────────────────────┐     │
│   │                 FilterChainProxy (Spring Bean)               │     │
│   │              ("springSecurityFilterChain")                   │     │
│   │                                                              │     │
│   │  ┌────────────────────────────────────────────────────────┐  │     │
│   │  │ SecurityFilterChain #1: matches /api/** (Order 1)      │  │     │
│   │  │ [Cors] -> [JwtFilter] -> [AuthFilter] -> [HeaderWriter]│  │     │
│   │  └────────────────────────────────────────────────────────┘  │     │
│   │  ┌────────────────────────────────────────────────────────┐  │     │
│   │  │ SecurityFilterChain #2: matches /** (Order 2)          │  │     │
│   │  │ [Csrf] -> [UsernamePasswordAuth] -> [BasicAuth] -> ... │  │     │
│   │  └────────────────────────────────────────────────────────┘  │     │
│   └──────────────────────────────┬───────────────────────────────┘     │
│                                  │ passes down to servlet              │
│                                  ▼                                     │
│                          DispatcherServlet                             │
│                                  │                                     │
│                                  ▼                                     │
│                           @RestController                              │
└────────────────────────────────────────────────────────────────────────┘
```

1. **`DelegatingFilterProxy`**: A standard Servlet Filter registered with Tomcat. Because Tomcat starts before the Spring `ApplicationContext` is fully initialized, `DelegatingFilterProxy` acts as a lazy bridge, looking up the bean named `springSecurityFilterChain` only when the first request arrives.
2. **`FilterChainProxy`**: The Spring-managed master filter. It contains one or more `SecurityFilterChain` beans. For each incoming request, it evaluates matching URL patterns and passes the request through the **first matching chain**.

### 1.2 Modern `SecurityFilterChain` Lambda DSL (Removal of `WebSecurityConfigurerAdapter`)

> [!CAUTION]
> In Spring Boot 3.x and Spring Security 6.x, `WebSecurityConfigurerAdapter` is completely removed. Extending this class results in a compilation failure. All configuration must be defined via standalone `@Bean` methods returning `SecurityFilterChain`.

The fluent `.and()` method chaining has been superseded by the **Lambda DSL**, which enforces explicit scoping and eliminates syntax ambiguity:

```java
package com.example.security.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class ModernSecurityConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Modern Lambda DSL (No .and() chaining)
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/actuator/health").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );

        return http.build();
    }
}
```

### 1.3 `SecurityContext` & `SecurityContextHolder` Lifecycle

Upon successful authentication, user identity is encapsulated within an `Authentication` object and bound to the current thread via `SecurityContextHolder`:

```java
// Accessing authenticated caller details anywhere in application code
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
if (authentication != null && authentication.isAuthenticated()) {
    String username = authentication.getName();
    Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
    Object principal = authentication.getPrincipal(); // Custom UserDetails or Jwt
}
```

> [!WARNING]
> By default, `SecurityContextHolder` uses a `ThreadLocal` strategy. When handing tasks off to worker threads (`@Async`, `CompletableFuture`, thread pools), the security context is **not propagated**. You must configure a `TaskDecorator` or use `DelegatingSecurityContextExecutor` to prevent `NullPointerException`.

---

## 2. Multi-Approach: Authentication Paradigms & Protocols

```
                            AUTHENTICATION TOPOLOGIES
 Single-Service Monolith                                Distributed Cloud / IAM
 Cookie / Session-Based                                 Cryptographic Token-Based
       │                                                           │
       ▼                                                           ▼
[Approach 3: Form Login] ──> [Approach 4: Basic/API Key] ──> [Approach 1: JWT] ──> [Approach 2: OAuth2/OIDC]
(Stateful Session)           (Service Automation)            (Self-Contained)      (Centralized Keycloak/Okta)
```

---

### Approach 1: Stateless JSON Web Token (JWT) with JJWT / Nimbus

#### Scenario
Modern Single Page Applications (React, Angular, Vue) and mobile apps consuming stateless REST APIs where server-side session affinity cannot be maintained across horizontally autoscaled pods.

#### Use-Case
B2C e-commerce, banking mobile apps, and public REST APIs requiring self-contained identity tokens.

#### Benefits Over Others / Trade-offs
- **Benefits**: 100% stateless; horizontal pod autoscaling requires zero session replication; payload contains user claims and roles ($O(1)$ access).
- **Trade-offs**: Tokens cannot be easily revoked prior to their `exp` expiration without maintaining a distributed Redis token blacklist; payload is Base64 encoded (not encrypted; never store passwords or PII in JWT claims).

#### Well-Documented Code Example

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

```java
package com.example.security.jwt;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.time.Duration;
import java.time.Instant;
import java.util.Date;
import java.util.List;
import java.util.Map;
import java.util.function.Function;

@Service
public class ProductionJwtService {

    @Value("${application.security.jwt.secret-key}")
    private String secretKey;

    @Value("${application.security.jwt.expiration-minutes:60}")
    private long jwtExpirationMinutes;

    private SecretKey getSignInKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public String generateToken(UserDetails userDetails, Map<String, Object> extraClaims) {
        List<String> roles = userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .toList();

        Instant now = Instant.now();
        Instant expiry = now.plus(Duration.ofMinutes(jwtExpirationMinutes));

        return Jwts.builder()
            .claims(extraClaims)
            .subject(userDetails.getUsername())
            .claim("roles", roles)
            .issuedAt(Date.from(now))
            .expiration(Date.from(expiry))
            .signWith(getSignInKey(), Jwts.SIG.HS256)
            .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return (username.equals(userDetails.getUsername())) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = Jwts.parser()
            .verifyWith(getSignInKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
        return claimsResolver.apply(claims);
    }
}
```

```java
package com.example.security.jwt;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpHeaders;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final ProductionJwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(ProductionJwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    FilterChain filterChain) throws ServletException, IOException {

        final String authHeader = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        final String jwt = authHeader.substring(7);
        final String userEmail = jwtService.extractUsername(jwt);

        // Authenticate only if context has not already been set
        if (userEmail != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(userEmail);

            if (jwtService.isTokenValid(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                    userDetails,
                    null,
                    userDetails.getAuthorities()
                );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

---

### Approach 2: OAuth2 / OIDC Resource Server with Remote JWKS Endpoint

#### Scenario
Enterprise distributed architectures integrating with a centralized Identity Provider (Keycloak, Auth0, Okta, Azure AD) where microservices validate cryptographically signed JWTs using public RSA/EC key sets (JWKS).

#### Use-Case
Multi-service internal ecosystem where user login occurs on an external IAM server, and backend Spring Boot microservices only verify signatures.

#### Benefits Over Others / Trade-offs
- **Benefits**: Asymmetric signing (RS256); microservices only need the public key (`jwk-set-uri`), completely removing shared secret leaks; automatic key rotation supported.
- **Trade-offs**: Microservices require network access to the Identity Provider during startup/caching to retrieve public keys.

#### Well-Documented Code Example

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.company.com/realms/enterprise
          jwk-set-uri: https://auth.company.com/realms/enterprise/protocol/openid-connect/certs
```

```java
package com.example.security.oauth2;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;
import org.springframework.security.authentication.AbstractAuthenticationToken;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.security.web.SecurityFilterChain;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Configuration
public class OAuth2ResourceServerConfig {

    @Bean
    public SecurityFilterChain resourceServerFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(customJwtAuthenticationConverter()))
            );

        return http.build();
    }

    /**
     * Converts Keycloak/Okta realm_access roles into Spring Security GrantedAuthorities.
     */
    private Converter<Jwt, ? extends AbstractAuthenticationToken> customJwtAuthenticationConverter() {
        return jwt -> {
            Collection<GrantedAuthority> authorities = extractRealmRoles(jwt);
            return new JwtAuthenticationToken(jwt, authorities, jwt.getClaimAsString("preferred_username"));
        };
    }

    @SuppressWarnings("unchecked")
    private Collection<GrantedAuthority> extractRealmRoles(Jwt jwt) {
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess == null || !realmAccess.containsKey("roles")) {
            return List.of();
        }

        List<String> roles = (List<String>) realmAccess.get("roles");
        return roles.stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
            .collect(Collectors.toList());
    }
}
```

---

### Approach 3: Stateful Session-Based Form Login with HTTP-Only Cookies

#### Scenario
Traditional monolithic web applications rendering server-side HTML via Thymeleaf, JSP, or Freemarker.

#### Use-Case
Internal company admin portals, CMS backends.

#### Benefits Over Others / Trade-offs
- **Benefits**: Easy session invalidation on logout; browser manages cookie transmission automatically; immune to token theft via JavaScript (HttpOnly).
- **Trade-offs**: State stored in server memory (requires sticky sessions or Redis Spring Session for multi-node clusters); strictly requires CSRF protection.

#### Well-Documented Code Example

```java
package com.example.security.formlogin;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

@Configuration
public class FormLoginWebSecurityConfig {

    @Bean
    @Order(2)
    public SecurityFilterChain webUiFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/admin/**", "/login", "/logout")
            .csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/css/**", "/js/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/admin/dashboard", true)
                .failureUrl("/login?error=true")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout=true")
                .deleteCookies("JSESSIONID")
                .invalidateHttpSession(true)
            );

        return http.build();
    }
}
```

---

### Approach 4: HTTP Basic Authentication & Custom API Key Headers

#### Scenario
System-to-system automated integrations, background daemons, or legacy machine webhooks.

#### Use-Case
Automated health probe monitors, CI/CD pipeline deployment webhooks.

#### Benefits Over Others / Trade-offs
- **Benefits**: Extremely simple; credentials or API key passed directly in headers without pre-flight authentication exchanges.
- **Trade-offs**: Credentials transmitted with every request; Base64 is easily decoded if TLS/HTTPS is terminated incorrectly.

#### Well-Documented Code Example

```java
package com.example.security.apikey;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

@Component
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {

    private static final String API_KEY_HEADER = "X-API-KEY";
    private final String expectedSecretKey = "secret-partner-api-key-998811";

    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    FilterChain filterChain) throws ServletException, IOException {

        String apiKey = request.getHeader(API_KEY_HEADER);
        if (apiKey != null) {
            if (expectedSecretKey.equals(apiKey)) {
                UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(
                    "Service-Partner", 
                    null, 
                    List.of(new SimpleGrantedAuthority("ROLE_SERVICE"))
                );
                SecurityContextHolder.getContext().setAuthentication(auth);
            } else {
                throw new BadCredentialsException("Invalid API Key header");
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

---

### Authentication Paradigms Comparison Matrix

| Paradigm | Session State | Client Compatibility | Token Revocation | CSRF Protection Required? | Best For |
|---|---|---|---|---|---|
| **Approach 1: Stateless JWT** | **Stateless** | SPAs, Mobile Apps | Via Redis Blacklist | No | Microservices, Public REST APIs |
| **Approach 2: OAuth2 / OIDC** | **Stateless** | Enterprise, SPAs | Via IdP Revocation | No | Centralized IAM (Keycloak / Okta) |
| **Approach 3: Form Login** | **Stateful** (Server Session) | Traditional Browsers | Instant Session Kill | **Yes (Mandatory)** | Server-rendered Web Apps (Thymeleaf) |
| **Approach 4: API Key / Basic**| **Stateless** | Daemons, Webhooks | Instant DB Lookup | No | Machine-to-Machine Background Jobs |

---

## 3. Multi-Approach: Password Hashing & Credential Storage Strategies

Plaintext passwords or weak hashing (MD5, SHA-256) represent critical security liabilities. Modern algorithms incorporate adjustable work factors and per-user salts.

---

### Approach 1: Adaptive BCrypt (`BCryptPasswordEncoder`)

#### Scenario
The default industry standard for standard web applications.

#### Benefits Over Others / Trade-offs
- **Benefits**: Built-in 128-bit salt; 60-character fixed hash output; logarithmic cost factor ($2^{\text{strength}}$ iterations) can be increased over time.
- **Trade-offs**: Not memory-hard (vulnerable to custom ASIC / GPU hardware cracking clusters compared to Argon2).

#### Well-Documented Code Example

```java
package com.example.security.crypto;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class BCryptSecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        // Strength 12 = 2^12 (4,096 rounds). Calibrated to execute in ~150-250ms on modern server CPUs
        return new BCryptPasswordEncoder(12);
    }
}
```

---

### Approach 2: Memory-Hard Argon2 (`Argon2PasswordEncoder`)

#### Scenario
High-security financial, government, or healthcare systems requiring state-of-the-art resistance to hardware brute-forcing.

#### Benefits Over Others / Trade-offs
- **Benefits**: Winner of the Password Hashing Competition; memory-hard (requires substantial RAM per hash, thwarting GPU/ASIC parallel attacks).
- **Trade-offs**: High CPU and RAM consumption per authentication request; requires tuning parameters to prevent DoS.

#### Well-Documented Code Example

```java
package com.example.security.crypto;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.argon2.Argon2PasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class Argon2SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        // saltLength: 16 bytes, hashLength: 32 bytes, parallelism: 1 thread, memory: 64MB, iterations: 3
        return new Argon2PasswordEncoder(16, 32, 1, 65536, 3);
    }
}
```

---

### Approach 3: Zero-Downtime Migration via `DelegatingPasswordEncoder`

#### Scenario
Legacy Spring applications migrating from obsolete hashing (e.g. SHA-256, MD5, or low-cost BCrypt) to modern BCrypt/Argon2 without forcing all users to reset their passwords.

#### Benefits Over Others / Trade-offs
- **Benefits**: Inspects password prefixes (e.g. `{bcrypt}`, `{argon2}`, `{sha256}`); verifies legacy hashes on login and automatically updates the database to the new default algorithm.

#### Well-Documented Code Example

```java
package com.example.security.crypto;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class DelegatingEncoderConfig {

    @Bean
    public PasswordEncoder delegatingPasswordEncoder() {
        // Supports {bcrypt}, {argon2}, {pbkdf2}, {scrypt} with modern bcrypt as default
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```

```java
// Automatic Hash Upgrade Service
@Service
public class AuthenticationUpgradeService {

    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;

    public AuthenticationUpgradeService(PasswordEncoder passwordEncoder, UserRepository userRepository) {
        this.passwordEncoder = passwordEncoder;
        this.userRepository = userRepository;
    }

    public boolean authenticateAndUpgrade(String rawPassword, User user) {
        if (!passwordEncoder.matches(rawPassword, user.getPasswordHash())) {
            return false;
        }

        // Check if hash algorithm is outdated
        if (passwordEncoder.upgradeEncoding(user.getPasswordHash())) {
            user.setPasswordHash(passwordEncoder.encode(rawPassword));
            userRepository.save(user); // Transparent upgrade!
        }
        return true;
    }
}
```

---

### Password Encoders Comparison Matrix

| Algorithm | Memory Hard? | Resistance to GPU Cracking | CPU Cost Factor | Output Length | Production Status |
|---|---|---|---|---|---|
| **Approach 1: BCrypt** | No | Moderate | $2^{\text{cost}}$ | 60 chars | **Standard Production Default** |
| **Approach 2: Argon2** | **Yes (64MB RAM)** | **Highest (Immune to GPUs)** | Configurable | Variable | **Highest Security Standard** |
| **Approach 3: Delegating**| Depends on prefix | Dual-Validation | Dynamic | Variable | **Zero-Downtime Migrations** |

---

## 4. Multi-Approach: Authorization & Access Control Models

Authorization determines whether an authenticated identity possesses permission to perform a specific action.

---

### Approach 1: HTTP-Level URL Rules via `requestMatchers`

#### Scenario
Coarse-grained perimeter access control grouped by URI path prefixes.

#### Use-Case
Securing entire `/api/v1/admin/**` paths to administrators.

#### Well-Documented Code Example

```java
http.authorizeHttpRequests(auth -> auth
    // Precise rules first
    .requestMatchers(HttpMethod.GET, "/api/v1/articles/**").permitAll()
    .requestMatchers(HttpMethod.POST, "/api/v1/articles/**").hasAuthority("ARTICLE_WRITE")
    .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
    // Catch-all must be last
    .anyRequest().authenticated()
);
```

---

### Approach 2: Method-Level Annotations (`@PreAuthorize`, `@PostAuthorize`, `@Secured`)

#### Scenario
Fine-grained authorization evaluated directly at the service or controller method boundary using Spring Expression Language (SpEL).

#### Use-Case
Verifying that an updating user matches the ID of the record being modified.

#### Well-Documented Code Example

```java
package com.example.security.method;

import org.springframework.security.access.prepost.PostAuthorize;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.stereotype.Service;

@Service
@EnableMethodSecurity // Default prePostEnabled = true in Spring Boot 3
public class SecureOrderManagementService {

    // SpEL evaluating caller principal matching parameter ID
    @PreAuthorize("#customerId == authentication.principal.id or hasRole('ADMIN')")
    public Order getCustomerOrder(Long customerId, Long orderId) {
        return findOrder(orderId);
    }

    // PostAuthorize checks returned object property before releasing to caller
    @PostAuthorize("returnObject.creatorUsername == authentication.name or hasRole('SUPERVISOR')")
    public Invoice generateConfidentialInvoice(Long invoiceId) {
        return findInvoice(invoiceId);
    }

    private Order findOrder(Long id) { return new Order(); }
    private Invoice findInvoice(Long id) { return new Invoice(); }
}
```

---

### Approach 3: Domain Object-Level Security with Custom `PermissionEvaluator`

#### Scenario
Complex enterprise authorization matrices where permissions depend on runtime relationships between users and domain entities.

#### Use-Case
Document management where a user can `READ` a folder, but only `OWNER` can `DELETE`.

#### Well-Documented Code Example

```java
package com.example.security.evaluator;

import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;

import java.io.Serializable;

@Component
public class DomainEntityPermissionEvaluator implements PermissionEvaluator {

    @Override
    public boolean hasPermission(Authentication auth, Object targetDomainObject, Object permission) {
        if (targetDomainObject == null || !(permission instanceof String)) {
            return false;
        }

        String targetType = targetDomainObject.getClass().getSimpleName().toUpperCase();
        return checkPermission(auth, targetDomainObject, targetType, permission.toString());
    }

    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId, String targetType, Object permission) {
        // Resolves ID from database and evaluates
        return true;
    }

    private boolean checkPermission(Authentication auth, Object target, String type, String permission) {
        if ("DOCUMENT".equals(type) && "DELETE".equals(permission)) {
            return auth.getAuthorities().stream().anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
        }
        return false;
    }
}
```

```java
// Method Security Usage
@PreAuthorize("hasPermission(#documentId, 'DOCUMENT', 'DELETE')")
public void deleteDocument(Long documentId) {
    repository.deleteById(documentId);
}
```

---

### Approach 4: Hierarchical Role & Authority Modeling (`RoleHierarchy`)

#### Scenario
Complex role topologies where `ROLE_ADMIN` should automatically inherit all permissions granted to `ROLE_STAFF` and `ROLE_USER`.

#### Well-Documented Code Example

```java
package com.example.security.hierarchy;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.access.hierarchicalroles.RoleHierarchy;
import org.springframework.security.access.hierarchicalroles.RoleHierarchyImpl;

@Configuration
public class RoleHierarchyConfig {

    @Bean
    public RoleHierarchy roleHierarchy() {
        RoleHierarchyImpl hierarchy = new RoleHierarchyImpl();
        hierarchy.setHierarchy("""
            ROLE_SUPERADMIN > ROLE_ADMIN
            ROLE_ADMIN > ROLE_MANAGER
            ROLE_MANAGER > ROLE_USER
        """);
        return hierarchy;
    }
}
```

---

### Authorization Models Comparison Matrix

| Authorization Model | Granularity | Performance Overhead | Evaluates Return Object? | Best For |
|---|---|---|---|---|
| **Approach 1: `requestMatchers`** | URL Path | **Fastest** | No | Perimeter access, Public vs Private |
| **Approach 2: `@PreAuthorize`** | Method Parameter | Low (AOP) | No | Method inputs & User IDs |
| **Approach 3: `@PostAuthorize`** | Returned Object | Low (AOP) | **Yes** | Ownership validation on DB queries |
| **Approach 4: `PermissionEvaluator`**| Domain Entity | Moderate | **Yes** | Multi-tenant complex ACL permissions |

---

## 5. Multi-Approach: Multi-Tenant & Multi-Chain Security Architecture

In enterprise microservices, different API segments require completely different authentication schemes (e.g. `/api/**` uses stateless JWT, while `/actuator/**` uses HTTP Basic, and `/admin/**` uses Form Login).

---

### Approach 1: Ordered Multiple `SecurityFilterChain` Beans (`@Order`)

#### Scenario
A single Spring Boot application hosting both customer REST APIs (JWT) and internal back-office administration interfaces (Form Login).

#### Well-Documented Code Example

```java
package com.example.security.multichain;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
public class MultiFilterChainSecurityConfig {

    // CHAIN 1: REST API (Highest Precedence - Order 1)
    @Bean
    @Order(1)
    public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http, JwtAuthenticationFilter jwtFilter) throws Exception {
        http
            .securityMatcher("/api/**") // Only matches /api/** paths!
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    // CHAIN 2: Admin Web Interface (Order 2 - Catch-all for remaining paths)
    @Bean
    @Order(2)
    public SecurityFilterChain webSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/error").permitAll()
                .anyRequest().hasRole("ADMIN")
            )
            .formLogin(form -> form.loginPage("/login").permitAll());

        return http.build();
    }
}
```

---

### Approach 2: Dynamic Multi-Tenant JWT Issuer Resolution (`JwtIssuerAuthenticationManagerResolver`)

#### Scenario
B2B SaaS platforms where each corporate tenant authenticates against their own dedicated Identity Provider (e.g., Tenant A uses `https://auth.tenanta.com`, Tenant B uses `https://auth.tenantb.com`).

#### Well-Documented Code Example

```java
package com.example.security.multitenant;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManagerResolver;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.server.resource.authentication.JwtIssuerAuthenticationManagerResolver;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class MultiTenantSecurityConfig {

    @Bean
    public SecurityFilterChain multiTenantFilterChain(HttpSecurity http) throws Exception {
        AuthenticationManagerResolver<HttpServletRequest> resolver = 
            JwtIssuerAuthenticationManagerResolver.fromTrustedIssuers(
                "https://auth.tenanta.com",
                "https://auth.tenantb.com",
                "https://auth.tenantc.com"
            );

        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.authenticationManagerResolver(resolver));

        return http.build();
    }
}
```

---

### Multi-Chain Architecture Comparison Matrix

| Multi-Chain Strategy | Matching Mechanism | Execution Behavior | Best For |
|---|---|---|---|
| **Approach 1: `@Order` Filter Chains** | `securityMatcher("/path/**")` | Evaluated in Order; First match wins | Mixed APIs & Admin Portals |
| **Approach 2: Dynamic Issuer Resolver** | JWT Header `iss` Issuer Claim | Dynamically routes to correct JWKS | Multi-Tenant SaaS Microservices |

---

## 6. Multi-Approach: CSRF Protection Across Client Topologies

Cross-Site Request Forgery (CSRF) tricks a user's browser into submitting state-changing requests using cached authentication cookies.

---

### Approach 1: Disabled CSRF (`csrf.disable()`) for Pure Stateless REST

#### Scenario
Pure REST microservices where authentication is mediated strictly through `Authorization: Bearer <JWT>` headers and browsers never store authentication in session cookies.

#### Why It Is Safe:
Browsers do not automatically attach custom `Authorization` headers to cross-origin requests. Without cookies, CSRF attacks are fundamentally impossible.

---

### Approach 2: Cookie-Based Dual-Submit CSRF Repository (Angular / React SPAs)

#### Scenario
Single Page Applications authenticating via session cookies where the frontend must read the CSRF token and return it inside an `X-XSRF-TOKEN` header.

#### Well-Documented Code Example

```java
package com.example.security.csrf;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

@Configuration
public class SpaCsrfSecurityConfig {

    @Bean
    public SecurityFilterChain spaSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            // CookieCsrfTokenRepository withHttpOnlyFalse allows Angular/React to read XSRF-TOKEN cookie
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringRequestMatchers("/api/v1/webhooks/**")
            )
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());

        return http.build();
    }
}
```

---

### Approach 3: Session-Based CSRF for Server-Side Rendering (Thymeleaf)

#### Scenario
Traditional MVC apps where Spring Security automatically injects hidden `_csrf` form tokens into HTML forms.

```html
<!-- Thymeleaf automatically injects hidden CSRF token -->
<form th:action="@{/process}" method="post">
    <input type="text" name="data" />
    <button type="submit">Submit</button>
</form>
```

---

### CSRF Strategies Comparison Matrix

| Strategy | Token Delivery | JavaScript Readable? | Best For |
|---|---|---|---|
| **Approach 1: Disabled** | None | N/A | Pure Stateless REST (Bearer JWT) |
| **Approach 2: Cookie Repo** | `XSRF-TOKEN` Cookie | **Yes (HttpOnly = false)** | Angular / React / Vue SPAs |
| **Approach 3: Session Repo** | Hidden `<input>` field | No | Thymeleaf / Spring MVC Forms |

---

## 7. Mitigating OWASP Top 10 Vulnerabilities in Spring Boot

### 7.1 Broken Object Level Authorization (IDOR)
Never trust user-supplied entity IDs without verifying ownership:

```java
@PreAuthorize("@securityService.isOrderOwner(#orderId, authentication)")
@GetMapping("/api/orders/{orderId}")
public OrderResponse fetchOrder(@PathVariable Long orderId) { ... }
```

### 7.2 Mass Assignment & DTO Isolation
Never bind incoming request JSON directly to JPA `@Entity` classes:

```java
// VULNERABLE: Attacker injects {"role": "ADMIN"} into payload
public void updateUser(@RequestBody User user) { repo.save(user); }

// SECURE: Use strict record DTOs
public record UpdateUserRequest(@NotBlank String firstName, @NotBlank String lastName) {}
```

### 7.3 Cross-Origin Resource Sharing (CORS) Misconfiguration
Always configure CORS through Spring Security to ensure pre-flight `OPTIONS` requests are handled correctly:

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowedOrigins(List.of("https://app.company.com"));
    configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    configuration.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    configuration.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```

### 7.4 Security Headers (HSTS, CSP, X-Frame-Options)

```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.deny()) // Clickjacking defense
    .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true).maxAgeInSeconds(31536000))
    .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
);
```

---

## 8. Production Security Testing (`@WithMockUser`, `@WithUserDetails`, MockMvc)

```java
package com.example.security.test;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.delete;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
class SecurityIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void unauthenticatedCallShouldReturn401() throws Exception {
        mockMvc.perform(get("/api/v1/orders"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(username = "regularUser", roles = "USER")
    void standardUserShouldBeForbiddenFromAdminEndpoints() throws Exception {
        mockMvc.perform(delete("/api/v1/admin/users/1"))
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(username = "adminUser", roles = "ADMIN")
    void adminUserShouldSucceed() throws Exception {
        mockMvc.perform(delete("/api/v1/admin/users/1"))
            .andExpect(status().isNoContent());
    }
}
```

---

## 9. Interview Traps, Anti-Patterns, & Gotchas

### Trap 1: The `ROLE_` Prefix Mismatch
`hasRole("ADMIN")` automatically checks for the authority `ROLE_ADMIN`. If authorities are stored as `ADMIN`, `hasRole("ADMIN")` fails. Use `hasAuthority("ADMIN")` for exact matches.

### Trap 2: Self-Invocation Bypass on `@PreAuthorize`
Calling an `@PreAuthorize` method from within the same class bypasses the CGLIB proxy. Security checks never execute!

### Trap 3: SecurityContext Loss in Asynchronous Threads
Because `SecurityContextHolder` defaults to `ThreadLocal`, `@Async` methods lose the authenticated user context. Always configure a `DelegatingSecurityContextExecutor` or `TaskDecorator`.

### Trap 4: Committing JWT Secrets to Git
Committing raw JWT secrets in `application.yml` allows anyone with repository access to forge administrator tokens. Always use environment variables or secret vaults.

### Trap 5: Naive In-Memory UserDetailsService in Production
Leaving `InMemoryUserDetailsManager` in production code risks credential loss on restarts and violates auditing standards.

---

## 10. Summary & Architectural Decision Trees

```
                           SPRING SECURITY ARCHITECTURE TREE
                                           │
         ┌─────────────────────────────────┴─────────────────────────────────┐
         ▼                                                                   ▼
[Single Page / Mobile Apps]                                         [Monolithic MVC Apps]
  ├── Authentication: Stateless JWT / OAuth2 Resource Server          ├── Authentication: Form Login + Cookies
  ├── Sessions: SessionCreationPolicy.STATELESS                       ├── Sessions: SessionCreationPolicy.IF_REQUIRED
  ├── CSRF: csrf.disable() (Bearer Headers)                           ├── CSRF: CookieCsrfTokenRepository
  └── Filters: OncePerRequestFilter (JwtFilter)                       └── Authorization: requestMatchers + URL
```

---

## 11. Practice Questions & Interview Scenarios

1. **The Order Gotcha**: An engineer defines `requestMatchers("/api/**").authenticated()` before `requestMatchers("/api/public/**").permitAll()`. Why does the public endpoint return HTTP 401?
   - *Answer*: Spring Security evaluates rules sequentially. The first matching rule wins. More specific paths must always be declared before generic wildcards.

2. **Async Security Leak**: A scheduled `@Async` reporting method throws `NullPointerException` when fetching `SecurityContextHolder.getContext().getAuthentication()`. Why, and how is it fixed?
   - *Answer*: `SecurityContextHolder` uses `ThreadLocal`. Async worker threads do not inherit this context. Fix by wrapping the task executor with `DelegatingSecurityContextExecutor` or configuring a `TaskDecorator`.

3. **Symmetric vs Asymmetric JWT**: When should an architect mandate RS256 over HS256?
   - *Answer*: In multi-service microservice environments. With RS256, only the authentication service holds the private key to sign tokens, while hundreds of downstream microservices only need the public key to verify signatures.
