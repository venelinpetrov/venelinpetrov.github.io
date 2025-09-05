---
title: "JWT authentication with Spring Boot and Web clients"
date: 2025-09-05
---

# JWT authentication with Spring Boot and Web clients

This article is a high level overview of the JWT authentication process in the context of Spring Boot and any web client.

## The backend

We need 4 main pieces

1. `SecurityConfig` (configures which routes are authenticated, or combines rules from feature configurations)
2. `JwtAuthenticationFilter` (http interceptor)
3. `JwtService` (token creation and validation)
4. `AuthController` (/login, returns a token if credentials match)

### Flow when visiting an authenticated routes
- We check Authorization headers with each request (`JwtAuthenticationFilter`)
- If there is no header or no token, just pass along the request and response objects to the next filter and return
- Otherwise, extract the token (when user logged successfully (AuthController) and received a token)

```java
@Component
@AllArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtService jwtService;

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        var authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        var token = authHeader.replace("Bearer ", "");

        var jwt = jwtService.parseToken(token);

        if (jwt == null || jwt.isExpired()) {
            filterChain.doFilter(request, response);
            return;
        }
        var role = jwt.getRole();
        var userId = jwt.getUserId();
        var authentication = new UsernamePasswordAuthenticationToken(
            userId,
            null,
            List.of(new SimpleGrantedAuthority("ROLE_" + role))
        );

        authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
        SecurityContextHolder.getContext().setAuthentication(authentication);
        filterChain.doFilter(request, response);
    }
}

```
- We check the token for validity via the `JwtService::validateToken` method

```java
@AllArgsConstructor
@Service
public class JwtService {
    private final JwtConfig jwtConfig;

    public Jwt generateAccessToken(User user) {
        return generateToken(user, jwtConfig.getAccessTokenExpiration());
    }

    public Jwt generateRefreshToken(User user) {
        return generateToken(user, jwtConfig.getRefreshTokenExpiration());
    }

    private Jwt generateToken(User user, long tokenExpirationSeconds) {
        Date now = new Date();
        Date expiryDate = new Date(System.currentTimeMillis() + tokenExpirationSeconds * 1000);

        var claims = Jwts.claims()
            .subject(user.getId().toString())
            .add("email", user.getEmail())
            .add("name", user.getName())
            .add("role", user.getRole().toString())
            .issuedAt(now)
            .expiration(expiryDate)
            .build();

        String token = Jwts.builder()
            .claims(claims)
            .signWith(jwtConfig.getSigningKey())
            .compact();

        return new Jwt(claims, token);
    }

    public Jwt parseToken(String token) {
        try {
            var claims = Jwts.parser()
                .verifyWith(jwtConfig.getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();

            return new Jwt(claims, token);
        } catch (JwtException e) {
            System.out.println("Invalid JWT: " + e.getMessage());
            return null;
        }
    }
}
```

```java
public class Jwt {
    private final Claims claims;
    private final String token;

    public Jwt(Claims claims, String token) {
        this.claims = claims;
        this.token = token;
    }

    public Long getUserId() {
        return Long.valueOf(claims.getSubject());
    }

    public String getRole() {
        return claims.get("role", String.class);
    }

    public boolean isExpired() {
        return claims.getExpiration().before(new java.util.Date());
    }

    @Override
    public String toString() {
        return token;
    }
}

```


- If the token is invalid pass along the request and response objects and return
- Finally, if the token was valid we set the user identity in the `SecurityContextHolder`
- Pass along the request and response objects as usual

> **Note**: In `SecurityConfig`, we set `JwtAuthenticationFilter` to be the first in the filter chain. This will become important later


```java
@AllArgsConstructor
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final List<SecurityRules> featureSecurityRules;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationProvider authenticationProvider(
            UserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder
    ) {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(c -> c.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(c -> {
                featureSecurityRules.forEach(r -> r.configure(c));
                c.anyRequest().authenticated();
            })
            .cors(c -> {})
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(c -> {
                c.authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED));
                c.accessDeniedHandler(((
                        request,
                        response,
                        accessDeniedException) -> response.setStatus(HttpStatus.FORBIDDEN.value()))
                );
            });

        return http.build();
    }
}
```

- Next in the filter chain is `AnonymousAuthenticationFilter` (set by Spring Boot). This filter creates an `AnonymousAuthenticationToken` and stores it in `SecurityContextHolder`. This means every request has some Authentication object, but an anonymous one if the user is not logged in.
- Then comes the `FilterSecurityInterceptor`. It compares the `Authentication` object from the `SecurityContextHolder` and the rules in `authorizeHttpRequests`:
    * If route is `.authenticated()` and user is anonymous - rejects with 401
    * If route is `.permitAll()` - allow
    * If route is `.authenticated()` and user from JWT - allow


### Flow during login

- We hit the unauthenticated route `POST /login`

```java
@AllArgsConstructor
@RequestMapping("/auth")
@RestController
public class AuthController {
    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;
    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final JwtConfig jwtConfig;

    @PostMapping("/login")
    public ResponseEntity<JwtResponse> login(@Valid @RequestBody LoginDto body, HttpServletResponse response) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(body.getEmail(), body.getPassword())
        );
        var user = userRepository.findByEmail(body.getEmail()).orElseThrow();
        var accessToken = jwtService.generateAccessToken(user);
        var refreshToken = jwtService.generateRefreshToken(user);

        var cookie = new Cookie("refreshToken", refreshToken.toString());
        cookie.setHttpOnly(true);
        cookie.setPath("/auth/refresh");
        cookie.setSecure(true);
        cookie.setAttribute("SameSite", "None");
        cookie.setMaxAge(jwtConfig.getRefreshTokenExpiration());
        response.addCookie(cookie);

        return ResponseEntity.ok(new JwtResponse(accessToken.toString()));
    }

    @PostMapping("/refresh")
    public ResponseEntity<JwtResponse> refresh(@CookieValue(value = "refreshToken") String refreshToken) {
        var jwt = jwtService.parseToken(refreshToken);

        if (jwt == null || jwt.isExpired()) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        var user = userRepository.findById(jwt.getUserId()).orElseThrow();
        var accessToken = jwtService.generateAccessToken(user);

        return ResponseEntity.ok(new JwtResponse(accessToken.toString()));
    }

    @ExceptionHandler(BadCredentialsException.class)
    public ResponseEntity<Void> handleBadCredentialsException(BadCredentialsException e) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
    }
}
```

- The controller delegates to `authenticationManager.authenticate(email, password)`
- `AuthenticationManager` calls `DaoAuthenticationProvider`
- `DaoAuthenticationProvider` calls `userDetailsService.loadUserByUsername(email)`
- `UserService` runs `findByEmail(...)` and returns a `UserDetails` object.

> Note: `UserService` implements `UserDetailsService` and overrides `loadUserByUsername`. This way we can use email, username, phone, id, etc, whatever unique identifier to find the user.

```java
@AllArgsConstructor
@Service
public class UserService implements UserDetailsService {
    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        var user = userRepository.findByEmail(email).orElseThrow(() -> new UsernameNotFoundException("User not found"));

        return new User(
            user.getEmail(),
            user.getPassword(),
            Collections.emptyList()
        );
    }
}

```

- Password check happens via `BCryptPasswordEncoder.matches(...)`

## Working with web clients

Let's have a general overview of the process, from a web client's perspective.

When a web app hits the `/auth/login` endpoint, the backend generates 2 tokens - an access token and a refresh token. The access token is used to authenticate the user when hitting protected endpoints by attaching it to each request as a request header. The backend decodes the token and decides whether the user is authenticated or not.

The authentication token is short-lived, meaning it will expire soon after it has been issued. We do this for security reasons. If a token leaks somehow, this will minimize the damage, because it will expire in few minutes.

This is where the refresh token comes into play. At some point, the access token eventually expires and the next authenticated request will be denied with http error 401. Because we don't want users to log in every 5 minutes we send them a refresh token too, via a cookie. Note that this token lives much longer, a day, a week, whatever the requirements are.

> **Security note:**
This cookie MUST be `http only`, which makes it unreachable from JavaScript code. This is extremely important, because if the refresh token leaks, an attacker can forge as many access token as they want.

The `http only` cookies are handled by the browser. The client doesn't need to do anything special to send them to the backend, they are attached to the request automatically.

> **Security note:** You must also set the `secure` attribute. This will prevent cookies from being observed by unauthorized parties due to the transmission of the cookie in clear text. Browsers will only send cookies with the `secure` attribute when the request is going to an HTTPS page.

Now, as soon as an authenticated endpoint responds with 401, our client sends `/auth/refresh` request and the cookie with the refresh token goes with it. In turns, the backend checks the validity of the refresh token and if it is valid it creates a new access token and responds to the client with it. The client receives the new access token and sets it **in-memory** for future requests, then retries the previous request with the new access token. This is how we implement a long "user session" with short-lived access token.

> **Security note:** It's tempting to store the access token in a local storage, and some tutorials even suggest it. What they are trying to avoid is logging out the user when the page refreshes. You should never store any security critical information in local storage, because it can be reached via JavaScript and thus makes it vulnerable to XSS attacks. That is why we store the access token in-memory

When the user refreshes the page the in-memory access token is lost, but that's not a problem in our case. The first authenticated request that gets rejected with 401 will trigger the refresh logic, the client will receive the new access token, attach it to this and any future requests until the access token expires again and everything repeats.

## Refreshing refresh tokens

You might be wondering what happens when the refresh token eventually expires too. A common solution is to generate a new refresh token each time the client requests one. This is called a "sliding window". If it hasn't been requested for a long enough time it just expires, at that time the access token expired too and the user is effectively logged out.

## Logging out

What about logging out, indeed? Unlike the traditional session-based authentication, there is no easy way to "revoke the session", because there is no session or state. The whole point of JWT authentication is to be stateless, which solves many problems but also creates this one.

The simplest solution for both, the client and the server is to just wait for the access token to expire (usually 5-10 minutes). The client app just deletes the token from memory and the user is logged out. The access token is still active though, but the security risk is considered very small.

## Security considerations

There is still a problem though, what happens if a user is hacked somehow and you want to minmize damadge by "revoking" their access? You can't. The only possible solution with this setup is to rotate the signing key, but this will affect all your users - they will all be logged out, because their current tokens will have invalid signatures.

This is an escape hatch solution and it is not generally recommended unless the security breach was indeed very serious (e.g. leaking your secrets).

Another approach would be to store refresh tokens server-side (or Redis set) as an "allow list".

```
refresh_tokens
- id (jti / uuid)
- user_id
- token_hash
- device_id (optional)
- issued_at
- expires_at
- replaced_by (nullable)    // for rotation chains
- revoked_at (nullable)
- reason (nullable)
```

> **Note** Store refresh tokens hashed!!! Leaking a database with refresh tokens in plain text will be catastriphic.

Do one-time rotation + reuse detection. On every refresh:

- Verify the refresh token by hashing and lookup.
- If found and not expired/revoked, issue a new access token and a new refresh token.
- Mark the old refresh token as replaced_by = new_jti and effectively unusable.
- If an already replaced token is presented (reuse): assume compromise, revoke the entire token family for that user/device.

Optional, but this also allows for revoking device-scoped sessions.

What about already-issued access tokens? You can’t reliably yank purely stateless access tokens already out in the wild. Mitigate by keeping access-token TTL short (e.g. 10 minutes).

I hope this was helpful, glhf `:)`