---
title: "Java Spring Exception Handling"
date: 2025-08-12
---

# Exception handling in Spring Boot with Java

In Spring Boot controllers, it’s common to return either a successful DTO response or some kind of error response directly from the controller method.

For example, in this endpoint, we might do:

```java
@PostMapping
public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserDto data, UriComponentsBuilder uriBuilder) {
    if (userRepository.existsByEmail(data.getEmail())) {
        return ResponseEntity.badRequest().build(); // Return Bad request error
    }

    var userEntity = userMapper.toEntity(data);
    userEntity.setPassword(passwordEncoder.encode(data.getPassword()));
    userEntity.setRole(Role.USER);
    userRepository.save(userEntity);

    var userDto = userMapper.toDto(userEntity);
    var uri = uriBuilder
        .path("/users/{id}")
        .buildAndExpand(userDto.getId())
        .toUri();

    return ResponseEntity.created(uri).body(userDto); // Success, return the DTO
}
```

but as soon as you want to add a custom error message:

```java
@PostMapping
public ResponseEntity<?> createUser(@Valid @RequestBody CreateUserDto data, UriComponentsBuilder uriBuilder) {
    if (userRepository.existsByEmail(data.getEmail())) {
        return ResponseEntity.badRequest().body(Map.of("error", "Email is already in use")); // << Custom error message
    }

    var userEntity = userMapper.toEntity(data);
    userEntity.setPassword(passwordEncoder.encode(data.getPassword()));
    userEntity.setRole(Role.USER);
    userRepository.save(userEntity);

    var userDto = userMapper.toDto(userEntity);
    var uri = uriBuilder
            .path("/users/{id}")
            .buildAndExpand(userDto.getId())
            .toUri();

    return ResponseEntity.created(uri).body(userDto); // Success, return the DTO
}
```

you end up losing the strong return type and replacing it with `?` like so `ResponseEntity<?>`. Not only that, this approach can quickly get messy, especially if you start returning different error formats from different endpoints.

There are several options to solve this:

## Option 1: Use a common response wrapper

One way to keep strong typing is to define a wrapper type that can represent both success and error cases.

```java
public class ApiResponse<T> {
    private T data;
    private ErrorDto error;

    public static <T> ApiResponse<T> success(T data) {
        ApiResponse<T> response = new ApiResponse<>();
        response.data = data;
        return response;
    }

    public static <T> ApiResponse<T> error(String message) {
        ApiResponse<T> response = new ApiResponse<>();
        response.error = new ErrorDto(message);
        return response;
    }

    public T getData() { return data; }
    public ErrorDto getError() { return error; }
}
```

Then, in the controller

```java
@PostMapping
public ResponseEntity<ApiResponse<UserDto>> createUser(@Valid @RequestBody CreateUserDto data, UriComponentsBuilder uriBuilder) {
    if (userRepository.existsByEmail(data.getEmail())) {
        return ResponseEntity.badRequest().body(ApiResponse.error("Email already in use"));
    }

    // ...

    return ResponseEntity.created(uri).body(ApiResponse.success(userDto));
}
```

**Pros:**

- Strong typing: `ApiResponse<UserDto>` is consistent for both success and error cases. A wrapper type guarantees that
the client always gets a predictable JSON structure, e.g.

```json
// success
{
  "data": { "orderId": 123 },
  "error": null
}

// error
{
  "data": null,
  "error": { "message": "Cart not found" }
}
```

- Easy to extend with metadata (timestamps, status codes, etc.).

**Cons:**

- The JSON structure now wraps everything in `{ "data": ..., "error": ... }` which might differ from your current API contract.

## Option 2: Use a sealed interface (Java 17+)

If you’re on Java 17+, you can model your responses as a sealed interface:

```java
public sealed interface UserResult permits UserDto, ErrorDto {}
```

Then:

```java
@PostMapping
public ResponseEntity<UserResult> createUser(@Valid @RequestBody CreateUserDto data, UriComponentsBuilder uriBuilder) {
    if (userRepository.existsByEmail(data.getEmail())) {
        return ResponseEntity.badRequest().body(new ErrorDto("Email already in use"));
    }

    // ...

    return ResponseEntity.created(uri).body(userDto);
}
```

**Pros**:

- The return type `UserResult` is still strongly typed, only `UserDto` or `ErrorDto` are allowed.
- No wrapper object required; JSON output remains as-is.

**Cons:**

- Requires Java 17+ (sealed types).
- Consumers still need to discriminate between types.
- Requires more boilerplate, though it's not that much. You can organize your files like this:

```text
checkout/
    dto/
        CheckoutResponseDto.java
        CheckoutErrorDto.java
        CheckoutResult.java // sealed interface
cart/
    dto/
        CartResponseDto.java
        CartErrorDto.java
        CartResult.java
```

## Option 3: Let Exceptions handle errors

Instead of returning an error DTO directly, you can throw exceptions for bad requests and let a `@ControllerAdvice` handle them.

```java
@ResponseStatus(HttpStatus.BAD_REQUEST)
public class BadRequestException extends RuntimeException {
    public BadRequestException(String message) {
        super(message);
    }
}
```

```java
public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserDto data, UriComponentsBuilder uriBuilder) {
    if (userRepository.existsByEmail(data.getEmail())) {
        throw new BadRequestException("Email is already in use");
    }

    // ...

    return ResponseEntity.created(uri).body(userDto);
}
```

Then in a global handler:

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BadRequestException.class)
    public ResponseEntity<ErrorDto> handleBadRequest(BadRequestException ex) {
        return ResponseEntity.badRequest().body(new ErrorDto(ex.getMessage()));
    }
}
```

**Pros:**

- Your controller’s return type stays `ResponseEntity<UserDto>`, no `?` needed.
- Error handling is centralized.

**Cons:**

- Semantics consideration: Control flow is now exception-driven and some devs dislike this for expected errors, because exceptions traditionally signal “unexpected” problems. If you throw them for normal validation (like "cart not found"), some see it as misuse. It hides the normal flow of the program in "error" branches
- Discoverability: With return types, you can see possible outcomes in the method signature (`ResponseEntity<ApiResponse<...>>`).
  With exceptions, the signature may not show what can go wrong unless you document it.

**Note:** That said, in Spring MVC this pattern is extremely common. The framework is designed so that you can throw custom exceptions and handle them centrally via `@ControllerAdvice`.

## Option 4: Hybrid of Option 1 + Option 3

Same as Option 1:

```java
public class ApiResponse<T> {
    private T data;
    private ErrorDto error;

    public static <T> ApiResponse<T> success(T data) {
        ApiResponse<T> response = new ApiResponse<>();
        response.data = data;
        return response;
    }

    public static <T> ApiResponse<T> error(String message) {
        ApiResponse<T> response = new ApiResponse<>();
        response.error = new ErrorDto(message);
        return response;
    }

    public T getData() { return data; }
    public ErrorDto getError() { return error; }
}
```

Throw this for validation or business rule failures:

```java
@ResponseStatus(HttpStatus.BAD_REQUEST)
public class ApiException extends RuntimeException {
    public ApiException(String message) {
        super(message);
    }
}
```

Global exception handler:

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ApiException.class)
    public ResponseEntity<ApiResponse<?>> handleApiException(ApiException ex) {
        return ResponseEntity
            .badRequest()
            .body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<?>> handleGeneralException(Exception ex) {
        // Log actual error internally
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.error("An unexpected error occurred"));
    }
}
```

Now your controller stays lean and returns only success cases:

```java
public ResponseEntity<ApiResponse<UserDto>> createUser(@Valid @RequestBody CreateUserDto data, UriComponentsBuilder uriBuilder) {
    if (userRepository.existsByEmail(data.getEmail())) {
        throw new ApiException("Email is already in use");
    }

    // ...

    return ResponseEntity.created(uri).body(ApiResponse.success(userDto));
}
```

Some of the advantages of this approach:

- Consistent JSON shape `{ "data": ..., "error": ... }` for both success and error.
- Clean controller methods (no more `if (...) return error;`).
- Centralized error handling.

Taking this one step further, if you want to support more nuanced error types (e.g. Not Found, Unauthorized, Conflict, Internal Error, etc.), you can extend the pattern so each exception:

1. Has a dedicated type (or at least a dedicated HTTP status).
2. Still gets wrapped in your ApiResponse for consistent JSON.

Create a Base exception. Instead of ApiException being tied to 400, make it abstract and allow specifying an HTTP status:

```java
public abstract class ApiException extends RuntimeException {
    private final HttpStatus status;

    protected ApiException(HttpStatus status, String message) {
        super(message);
        this.status = status;
    }

    public HttpStatus getStatus() {
        return status;
    }
}
```

Create specific exceptions. These make your controller code self-documenting and let the handler pick the right status automatically.

```java
public class BadRequestException extends ApiException {
    public BadRequestException(String message) {
        super(HttpStatus.BAD_REQUEST, message);
    }
}

public class NotFoundException extends ApiException {
    public NotFoundException(String message) {
        super(HttpStatus.NOT_FOUND, message);
    }
}

public class UnauthorizedException extends ApiException {
    public UnauthorizedException(String message) {
        super(HttpStatus.UNAUTHORIZED, message);
    }
}
```

Now in `GlobalExceptionHandler` one handler method can handle all `ApiException` types and use their status:

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ApiException.class)
    public ResponseEntity<ApiResponse<?>> handleApiException(ApiException ex) {
        return ResponseEntity
            .status(ex.getStatus())
            .body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<?>> handleUnexpectedException(Exception ex) {
        // Log internally for debugging
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.error("An unexpected error occurred"));
    }
}
```

Controller example:

```java
@PostMapping
@Transactional
public ResponseEntity<ApiResponse<CheckoutResponseDto>> checkout(
    @Valid @RequestBody CheckoutRequestDto request) {

    var cart = cartRepository.getCartWithItems(request.getCartId())
        .orElseThrow(() -> new NotFoundException("Cart not found"));

    if (cart.getItems().isEmpty()) {
        throw new BadRequestException("Cart is empty");
    }

    // ...

    return ResponseEntity.ok(ApiResponse.success(new CheckoutResponseDto(order.getId())));
}
```

## The `ErrorDto`

Let's talk about the `ErrorDto` and see how we can improve it.

As you can see it's a single property class, something like this:

```java
public record ErrorDto(String error) {}
```

That works fine when you always return one error string. However APIs often need to return several validation errors at once (e.g. "name is required", "phone is required" etc.)

In that case, instead of a plain list of strings, you usually wrap it in an envelope object:

```java
public record ErrorResponseDto(
    String message,         // high-level error, e.g. "Validation failed"
    List<String> errors     // list of detailed messages
) {}
```

Example JSON response:

```json
{
  "message": "Validation failed",
  "errors": [
    "name is required",
    "phne is required"
  ]
}
```

Why not just return `List<String>`? You could, but it’s not idiomatic because:

- You lose flexibility (can’t easily add fields like `timestamp`, `status`, `path`, etc.).
- Clients usually expect a consistent error envelope (especially when consuming large APIs).

A richer version would look something like this:

```java
public record ErrorResponseDto(
    int status,                  // HTTP status (e.g. 400)
    String error,                // short reason (e.g. "Bad Request")
    String message,              // human-readable (e.g. "Validation failed")
    String path,                 // endpoint (e.g. "/api/products")
    List<String> errors,         // detailed errors
    Instant timestamp            // when it happened
) {}
```

Example JSON:

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "path": "/api/products",
  "errors": [
    "name is required",
    "email is required"
  ],
  "timestamp": "2025-09-05T14:22:30Z"
}
```

There is also an [RFC 9457 Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html) that standardizes API error responses. It's worth checking that out too.

I hope this was helpful, glhf `:)`