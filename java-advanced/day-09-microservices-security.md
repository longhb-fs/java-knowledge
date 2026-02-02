# Day 9: Security in Microservices

## Mục tiêu
- OAuth 2.0 & OpenID Connect
- JWT token handling
- API Gateway security
- Service-to-service authentication

---

## 1. Security Challenges in Microservices

### 1.1. Attack Surface

```
┌─────────────────────────────────────────────────────────────────┐
│            MICROSERVICES SECURITY CHALLENGES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Monolith:                                                     │
│   ┌─────────────────────────────────┐                           │
│   │  Single Security Boundary       │                           │
│   │  ┌─────┬─────┬─────┬─────┐     │                           │
│   │  │ A   │  B  │  C  │  D  │     │  One entry point          │
│   │  └─────┴─────┴─────┴─────┘     │  Trusted internal calls   │
│   └─────────────────────────────────┘                           │
│                                                                  │
│   Microservices:                                                │
│   ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐                     │
│   │  A  │◄──►│  B  │◄──►│  C  │◄──►│  D  │                     │
│   └──┬──┘    └──┬──┘    └──┬──┘    └──┬──┘                     │
│      │          │          │          │                         │
│   Network    Network    Network    Network                      │
│   Exposed    Exposed    Exposed    Exposed                      │
│                                                                  │
│   Challenges:                                                    │
│   ✗ Multiple entry points                                       │
│   ✗ Network traffic can be intercepted                          │
│   ✗ Service impersonation possible                              │
│   ✗ Token propagation needed                                    │
│   ✗ Credential management complex                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Security Principles

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY PRINCIPLES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Defense in Depth                                              │
│   ├── Edge Security (API Gateway)                               │
│   ├── Transport Security (TLS)                                  │
│   ├── Authentication (OAuth2/JWT)                               │
│   ├── Authorization (RBAC/ABAC)                                 │
│   └── Data Security (Encryption)                                │
│                                                                  │
│   Zero Trust Architecture                                       │
│   ├── Never trust, always verify                                │
│   ├── Verify every service-to-service call                      │
│   ├── Encrypt all network traffic                               │
│   └── Implement least privilege                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. OAuth 2.0 & OpenID Connect

### 2.1. OAuth 2.0 Flows

```
┌─────────────────────────────────────────────────────────────────┐
│                 AUTHORIZATION CODE FLOW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌────────┐                           ┌────────────────┐       │
│   │  User  │                           │ Authorization  │       │
│   │Browser │                           │    Server      │       │
│   └───┬────┘                           └───────┬────────┘       │
│       │                                        │                │
│   1.  │ ─── Login Request ───────────────────► │                │
│       │                                        │                │
│   2.  │ ◄─── Login Page ─────────────────────  │                │
│       │                                        │                │
│   3.  │ ─── Username/Password ───────────────► │                │
│       │                                        │                │
│   4.  │ ◄─── Authorization Code (redirect) ──  │                │
│       │                                        │                │
│       │      ┌───────────┐                     │                │
│       │      │  Backend  │                     │                │
│       │      │  Server   │                     │                │
│       │      └─────┬─────┘                     │                │
│       │            │                           │                │
│   5.  │ ─ Code ─►  │ ─── Exchange Code ──────► │                │
│       │            │                           │                │
│   6.  │            │ ◄─── Access Token ──────  │                │
│       │            │      Refresh Token        │                │
│       │            │      ID Token             │                │
│       │ ◄─ Token ─ │                           │                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2. Token Types

```
┌────────────────────────────────────────────────────────────────┐
│                    TOKEN TYPES                                  │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Access Token (OAuth 2.0)                                     │
│   ├── Used to access protected resources                       │
│   ├── Short-lived (minutes to hours)                           │
│   ├── Sent in Authorization header                             │
│   └── Can be JWT or opaque                                     │
│                                                                 │
│   Refresh Token (OAuth 2.0)                                    │
│   ├── Used to obtain new access tokens                         │
│   ├── Long-lived (days to months)                              │
│   ├── Stored securely, never sent to APIs                      │
│   └── Can be rotated for security                              │
│                                                                 │
│   ID Token (OpenID Connect)                                    │
│   ├── Contains user identity claims                            │
│   ├── Always a JWT                                             │
│   ├── Used for authentication                                  │
│   └── Never sent to APIs                                       │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. JWT Implementation

### 3.1. JWT Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    JWT STRUCTURE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Header.Payload.Signature                                      │
│                                                                  │
│   Header (Algorithm & Type):                                    │
│   {                                                              │
│     "alg": "RS256",                                             │
│     "typ": "JWT"                                                │
│   }                                                              │
│                                                                  │
│   Payload (Claims):                                             │
│   {                                                              │
│     "sub": "user123",                                           │
│     "name": "John Doe",                                         │
│     "email": "john@example.com",                                │
│     "roles": ["ROLE_USER", "ROLE_ADMIN"],                       │
│     "iss": "auth-server",                                       │
│     "aud": "api-gateway",                                       │
│     "iat": 1704067200,                                          │
│     "exp": 1704070800                                           │
│   }                                                              │
│                                                                  │
│   Signature:                                                    │
│   RSASHA256(                                                    │
│     base64UrlEncode(header) + "." +                             │
│     base64UrlEncode(payload),                                   │
│     privateKey                                                  │
│   )                                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2. JWT Service

```java
@Service
public class JwtService {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration:3600000}")
    private long expiration;

    private Key getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secret);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList()));

        return Jwts.builder()
            .setClaims(claims)
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSigningKey(), SignatureAlgorithm.HS256)
            .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    @SuppressWarnings("unchecked")
    public List<String> extractRoles(String token) {
        Claims claims = extractAllClaims(token);
        return claims.get("roles", List.class);
    }
}
```

### 3.3. JWT Authentication Filter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        final String jwt = authHeader.substring(7);

        try {
            final String username = jwtService.extractUsername(jwt);

            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(jwt, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,
                            userDetails.getAuthorities()
                        );

                    authToken.setDetails(
                        new WebAuthenticationDetailsSource().buildDetails(request)
                    );

                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (ExpiredJwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\": \"Token expired\"}");
            return;
        } catch (JwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\": \"Invalid token\"}");
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```

---

## 4. Authorization Server Setup

### 4.1. Spring Authorization Server

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-oauth2-authorization-server</artifactId>
</dependency>
```

```java
@Configuration
public class AuthorizationServerConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http)
            throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);

        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            .oidc(Customizer.withDefaults());

        http.exceptionHandling(exceptions ->
            exceptions.authenticationEntryPoint(
                new LoginUrlAuthenticationEntryPoint("/login")));

        return http.build();
    }

    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient webClient = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("web-client")
            .clientSecret("{noop}web-secret")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("http://localhost:4200/callback")
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope("read")
            .scope("write")
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofHours(1))
                .refreshTokenTimeToLive(Duration.ofDays(7))
                .build())
            .build();

        RegisteredClient serviceClient = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("order-service")
            .clientSecret("{noop}order-secret")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            .scope("internal")
            .build();

        return new InMemoryRegisteredClientRepository(webClient, serviceClient);
    }

    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        RSAKey rsaKey = generateRsa();
        JWKSet jwkSet = new JWKSet(rsaKey);
        return (jwkSelector, securityContext) -> jwkSelector.select(jwkSet);
    }

    private static RSAKey generateRsa() {
        KeyPair keyPair = generateRsaKey();
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
        return new RSAKey.Builder(publicKey)
            .privateKey(privateKey)
            .keyID(UUID.randomUUID().toString())
            .build();
    }

    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
            .issuer("http://localhost:9000")
            .build();
    }
}
```

### 4.2. Custom Token Claims

```java
@Component
public class CustomTokenCustomizer implements OAuth2TokenCustomizer<JwtEncodingContext> {

    private final UserRepository userRepository;

    @Override
    public void customize(JwtEncodingContext context) {
        if (context.getTokenType().equals(OAuth2TokenType.ACCESS_TOKEN)) {
            Authentication principal = context.getPrincipal();

            if (principal instanceof UsernamePasswordAuthenticationToken) {
                User user = userRepository.findByUsername(principal.getName())
                    .orElseThrow();

                context.getClaims()
                    .claim("user_id", user.getId())
                    .claim("email", user.getEmail())
                    .claim("roles", user.getRoles().stream()
                        .map(Role::getName)
                        .collect(Collectors.toList()));
            }
        }
    }
}
```

---

## 5. Resource Server Configuration

### 5.1. Basic Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:9000
          # Or specify JWK Set URI directly
          # jwk-set-uri: http://localhost:9000/oauth2/jwks
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
            );

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter jwtAuthenticationConverter =
            new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(
            grantedAuthoritiesConverter);

        return jwtAuthenticationConverter;
    }
}
```

### 5.2. Method Security

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping
    @PreAuthorize("hasRole('ADMIN')")
    public List<User> getAllUsers() {
        return userService.findAll();
    }

    @GetMapping("/{id}")
    @PreAuthorize("hasRole('USER') and #id == authentication.principal.claims['user_id']")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public User createUser(@RequestBody CreateUserRequest request) {
        return userService.create(request);
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.claims['user_id']")
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}

// Custom permission evaluator
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {

    @Override
    public boolean hasPermission(Authentication authentication,
            Object targetDomainObject, Object permission) {

        if (targetDomainObject instanceof Order order) {
            Jwt jwt = (Jwt) authentication.getPrincipal();
            String userId = jwt.getClaim("user_id");

            if ("read".equals(permission)) {
                return order.getUserId().equals(userId) ||
                    hasRole(authentication, "ADMIN");
            }
        }
        return false;
    }

    @Override
    public boolean hasPermission(Authentication authentication,
            Serializable targetId, String targetType, Object permission) {
        // Load object and check permission
        return false;
    }
}
```

---

## 6. API Gateway Security

### 6.1. Gateway Authentication

```java
@Configuration
public class GatewaySecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/auth/**").permitAll()
                .pathMatchers("/actuator/health").permitAll()
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
            );

        return http.build();
    }

    private Converter<Jwt, Mono<AbstractAuthenticationToken>> jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter jwtAuthenticationConverter =
            new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(
            grantedAuthoritiesConverter);

        return new ReactiveJwtAuthenticationConverterAdapter(jwtAuthenticationConverter);
    }
}
```

### 6.2. Token Relay Filter

```java
@Component
public class TokenRelayGatewayFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return exchange.getPrincipal()
            .filter(principal -> principal instanceof JwtAuthenticationToken)
            .cast(JwtAuthenticationToken.class)
            .map(JwtAuthenticationToken::getToken)
            .map(jwt -> {
                // Forward token to downstream services
                ServerHttpRequest request = exchange.getRequest().mutate()
                    .header("Authorization", "Bearer " + jwt.getTokenValue())
                    .header("X-User-Id", jwt.getClaimAsString("user_id"))
                    .header("X-User-Roles", String.join(",",
                        jwt.getClaimAsStringList("roles")))
                    .build();
                return exchange.mutate().request(request).build();
            })
            .defaultIfEmpty(exchange)
            .flatMap(chain::filter);
    }

    @Override
    public int getOrder() {
        return -100;
    }
}
```

---

## 7. Service-to-Service Authentication

### 7.1. Client Credentials Flow

```java
@Configuration
public class OAuth2ClientConfig {

    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrationRepository,
            OAuth2AuthorizedClientService authorizedClientService) {

        OAuth2AuthorizedClientProvider authorizedClientProvider =
            OAuth2AuthorizedClientProviderBuilder.builder()
                .clientCredentials()
                .build();

        AuthorizedClientServiceOAuth2AuthorizedClientManager authorizedClientManager =
            new AuthorizedClientServiceOAuth2AuthorizedClientManager(
                clientRegistrationRepository, authorizedClientService);

        authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);

        return authorizedClientManager;
    }

    @Bean
    public WebClient webClient(OAuth2AuthorizedClientManager authorizedClientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2Client =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);

        oauth2Client.setDefaultClientRegistrationId("internal-service");

        return WebClient.builder()
            .apply(oauth2Client.oauth2Configuration())
            .build();
    }
}
```

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          internal-service:
            client-id: order-service
            client-secret: order-secret
            authorization-grant-type: client_credentials
            scope: internal
        provider:
          internal-service:
            token-uri: http://auth-server:9000/oauth2/token
```

### 7.2. mTLS (Mutual TLS)

```yaml
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    client-auth: need  # Require client certificate
    trust-store: classpath:truststore.p12
    trust-store-password: changeit
```

```java
@Configuration
public class MtlsWebClientConfig {

    @Bean
    public WebClient mtlsWebClient() throws Exception {
        // Load client certificate
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        keyStore.load(new FileInputStream("client.p12"), "password".toCharArray());

        KeyManagerFactory keyManagerFactory =
            KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        keyManagerFactory.init(keyStore, "password".toCharArray());

        // Load trust store
        KeyStore trustStore = KeyStore.getInstance("PKCS12");
        trustStore.load(new FileInputStream("truststore.p12"), "password".toCharArray());

        TrustManagerFactory trustManagerFactory =
            TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init(trustStore);

        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(
            keyManagerFactory.getKeyManagers(),
            trustManagerFactory.getTrustManagers(),
            null
        );

        HttpClient httpClient = HttpClient.create()
            .secure(sslContextSpec -> sslContextSpec.sslContext(
                SslContextBuilder.forClient()
                    .keyManager(keyManagerFactory)
                    .trustManager(trustManagerFactory)
                    .build()
            ));

        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

---

## 8. Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                MICROSERVICES SECURITY ARCHITECTURE               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   External User                                                  │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    API GATEWAY                           │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│   │  │ Rate Limit  │  │ JWT Verify  │  │ RBAC Check  │     │   │
│   │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│   └────────────────────────┬────────────────────────────────┘   │
│                            │                                     │
│   ┌────────────────────────┼────────────────────────────────┐   │
│   │                        │   INTERNAL NETWORK              │   │
│   │            ┌───────────┼───────────┐                    │   │
│   │            ▼           ▼           ▼                    │   │
│   │       ┌─────────┐ ┌─────────┐ ┌─────────┐              │   │
│   │       │ User    │ │ Order   │ │ Payment │              │   │
│   │       │ Service │ │ Service │ │ Service │              │   │
│   │       └────┬────┘ └────┬────┘ └────┬────┘              │   │
│   │            │           │           │                    │   │
│   │     mTLS / Client Credentials for internal calls       │   │
│   │                                                         │   │
│   │       ┌────────────────────────────────────┐           │   │
│   │       │        AUTHORIZATION SERVER         │           │   │
│   │       │  ┌─────────┐  ┌─────────────────┐  │           │   │
│   │       │  │  OAuth  │  │ User Directory  │  │           │   │
│   │       │  │  2.0    │  │   (LDAP/DB)     │  │           │   │
│   │       │  └─────────┘  └─────────────────┘  │           │   │
│   │       └────────────────────────────────────┘           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Security Best Practices

```java
// 1. Never log sensitive data
log.info("User {} logged in", username);  // Good
log.info("Password: {}", password);        // BAD!

// 2. Use HTTPS everywhere
// 3. Validate all input
public void createUser(@Valid @RequestBody CreateUserRequest request) {
    // Validation annotations on DTO
}

// 4. Use prepared statements (JPA handles this)
// 5. Implement proper CORS
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowedOrigins(List.of("https://app.example.com"));
    configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    configuration.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    configuration.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}

// 6. Secure sensitive configuration
// Use Vault or encrypted properties

// 7. Implement audit logging
@EventListener
public void onAuthenticationSuccess(AuthenticationSuccessEvent event) {
    auditService.log(AuditEvent.LOGIN_SUCCESS, event.getAuthentication().getName());
}

// 8. Regular token rotation
// 9. Implement account lockout
// 10. Use strong password policies
```

---

## 10. Bài tập thực hành

### Bài 1: JWT Authentication
Implement JWT authentication service với Spring Security.

### Bài 2: OAuth 2.0 Authorization Server
Setup Authorization Server với multiple client types.

### Bài 3: API Gateway Security
Secure API Gateway với JWT validation và rate limiting.

### Bài 4: Service-to-Service Auth
Implement Client Credentials flow giữa services.

---

## 11. Key Takeaways

| Concept | Description |
|---------|-------------|
| OAuth 2.0 | Authorization framework |
| OpenID Connect | Authentication layer on OAuth |
| JWT | Self-contained token format |
| Access Token | Short-lived API access |
| Refresh Token | Long-lived token renewal |
| mTLS | Mutual certificate auth |

```
Security Checklist:
☑ HTTPS everywhere
☑ JWT validation at gateway
☑ Token expiration enforced
☑ Roles/permissions checked
☑ Audit logging enabled
☑ Rate limiting configured
☑ CORS properly configured
☑ Secrets in Vault
```

---

## Navigation

- [← Day 8: Distributed Transactions](./day-08-distributed-transactions.md)
- [Day 10: Microservices Testing →](./day-10-microservices-testing.md)
