# Web Integration Design: Unified API Response, Exception Handling & Logging

**Date:** 2026-04-28  
**Status:** Design Review  
**Project:** lib-management-admin (Spring Boot 3.5.14)

## Problem Statement

The library management admin backend needs:
1. A standardized API response format for all endpoints
2. Centralized exception handling with meaningful error messages
3. Comprehensive request/response logging for debugging and monitoring
4. A sample REST API endpoint to demonstrate the integration

**Goal:** Build reusable infrastructure that all future endpoints can use consistently.

---

## Architecture Overview

```
HTTP Request
    ↓
[ApiResponseInterceptor] → Log request (path, method, headers, body if present)
    ↓
[Spring Controller/Service]
    ↓
[Response/Exception]
    ↓
[GlobalExceptionHandler] OR [ApiResponse Wrapper]
    ↓
[ApiResponseInterceptor] → Log response (status code, body, duration)
    ↓
HTTP Response (Unified JSON)
```

---

## Core Components

### 1. **ApiResponse<T> - Unified Response Format**

**Location:** `com.zahan.app.libmgmt.common.response.ApiResponse`

**Structure:**
```java
{
  "code": 200,                    // HTTP status code
  "message": "Success",           // Human-readable message
  "data": {...},                  // Actual response payload (generic T)
  "timestamp": "2026-04-28T20:40:00Z",
  "requestId": "req-12345-abc",   // Unique request identifier
  "path": "/api/hello",           // Request endpoint path
  "duration": 45                  // Response time in ms
}
```

**Responsibilities:**
- Wrap all successful responses
- Provide builder pattern for easy construction
- Support generic data type for flexibility

### 2. **ApiResponseInterceptor - Request/Response Logging**

**Location:** `com.zahan.app.libmgmt.common.interceptor.ApiResponseInterceptor`

**Implements:** `HandlerInterceptor`

**Responsibilities:**
- **Pre-handle:** Capture request metadata (method, path, headers, body if readable)
- **Post-handle:** Calculate response duration
- **After completion:** Log full request/response cycle with all fields
- Generate unique `requestId` for tracking
- Store timing information for performance analysis

**Logging Output:**
```
[API] requestId=req-12345-abc | method=GET | path=/api/hello | status=200 | duration=45ms
```

### 3. **GlobalExceptionHandler - Centralized Error Handling**

**Location:** `com.zahan.app.libmgmt.common.exception.GlobalExceptionHandler`

**Implements:** `@ControllerAdvice`

**Handles:**
- `ApiException` (custom business logic exceptions)
- `MethodArgumentNotValidException` (validation errors)
- `HttpMessageNotReadableException` (malformed JSON)
- `ResourceNotFoundException` (custom 404)
- `Exception` (catch-all for unexpected errors)

**Response Format (Error Case):**
```json
{
  "code": 400,
  "message": "Invalid request parameter",
  "data": null,
  "timestamp": "2026-04-28T20:40:00Z",
  "requestId": "req-12345-abc",
  "path": "/api/hello",
  "duration": 15
}
```

### 4. **Sample HelloController**

**Location:** `com.zahan.app.libmgmt.api.controller.HelloController`

**Endpoint:**
- `GET /api/hello` → Returns greeting message
- Demonstrates successful response wrapping
- Can be extended with query parameters

**Response Example:**
```json
{
  "code": 200,
  "message": "Success",
  "data": {
    "greeting": "Hello, Welcome to Library Management Admin API!"
  },
  "timestamp": "2026-04-28T20:40:00Z",
  "requestId": "req-12345-abc",
  "path": "/api/hello",
  "duration": 5
}
```

### 5. **Logging Configuration**

**Location:** `application.yml`

**Setup:**
- Use SLF4J (default Spring Boot logger)
- Log level: `INFO` for normal operations, `DEBUG` available for development
- Log `ApiResponseInterceptor` output to standard application logs
- All logs include: timestamp, level, logger name, message

---

## File Structure

```
src/main/java/com/zahan/app/libmgmt/
├── common/
│   ├── response/
│   │   └── ApiResponse.java         # Unified response wrapper
│   ├── interceptor/
│   │   └── ApiResponseInterceptor.java  # Request/response logging
│   └── exception/
│       ├── GlobalExceptionHandler.java  # Centralized error handling
│       ├── ApiException.java            # Custom exception
│       └── ResourceNotFoundException.java
├── api/
│   └── controller/
│       └── HelloController.java     # Sample endpoint
├── config/
│   └── WebConfig.java               # Register interceptor
└── LibmgmtApplication.java

src/main/resources/
└── application.yml                   # Logging configuration
```

---

## Exception Hierarchy

- `ApiException` (extends `RuntimeException`)
  - `ResourceNotFoundException` (extends `ApiException`)
  - Other domain-specific exceptions can extend `ApiException`

**Usage:**
```java
throw new ResourceNotFoundException("User not found");
throw new ApiException("Invalid operation", 400);
```

---

## Logging Behavior (Verbose - All Requests)

**What gets logged:**
1. ✅ Every HTTP request (method, path, params, headers if configured)
2. ✅ Request body (for POST/PUT, if readable)
3. ✅ Every HTTP response (status code, body structure)
4. ✅ Response duration
5. ✅ Unique requestId for correlation
6. ✅ All exceptions with stack traces

**Log Format:**
```
[timestamp] [LEVEL] [logger] [requestId] - Message
```

---

## Configuration & Registration

**WebConfig.java:**
- Register `ApiResponseInterceptor` with Spring
- Configure which paths to intercept (all `/api/**` by default)

**GlobalExceptionHandler:**
- Automatically detected by Spring via `@ControllerAdvice`
- No explicit registration needed

---

## Integration Points for Future Development

1. **New Controllers:** Extend any controller, return data → automatic wrapping in `ApiResponse`
2. **New Exceptions:** Create custom exceptions extending `ApiException` → automatic handling
3. **New Logging Rules:** Modify `ApiResponseInterceptor` or logging config → applies globally
4. **Authentication:** Add to interceptor's pre-handle phase

---

## Testing Strategy

- **Unit Tests:** Test `ApiResponse` builder, exception handlers in isolation
- **Integration Tests:** Test full request/response cycle with mock endpoints
- **Manual Testing:** Use curl/Postman to verify response format and logging output

---

## Success Criteria

✅ All API responses follow unified format with: code, message, data, timestamp, requestId, path, duration  
✅ All exceptions caught and returned as unified error response  
✅ All requests/responses logged with verbosity (every HTTP call logged)  
✅ Sample `/api/hello` endpoint returns proper response format  
✅ Unique requestId generated per request for tracing  

---

## Notes & Constraints

- Uses Spring Boot's built-in `HandlerInterceptor` (no additional dependencies required)
- Logging uses SLF4J (included with Spring Boot)
- Response body logging may require custom wrapper for full request body capture
- Performance impact of verbose logging is minimal for typical library management operations
