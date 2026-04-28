# Web Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build unified API response infrastructure with centralized exception handling and comprehensive request/response logging for Spring Boot library management admin backend.

**Architecture:** Implement HandlerInterceptor for request/response logging, @ControllerAdvice for centralized exception handling, and a generic ApiResponse wrapper for all responses. Add dependencies minimally (Spring Boot built-ins only).

**Tech Stack:** Spring Boot 3.5.14, SLF4J, Maven

---

## File Structure

```
src/main/java/com/zahan/app/libmgmt/
├── common/
│   ├── response/
│   │   └── ApiResponse.java         # Generic response wrapper
│   ├── interceptor/
│   │   └── ApiResponseInterceptor.java  # Request/response logging interceptor
│   ├── exception/
│   │   ├── GlobalExceptionHandler.java  # Centralized exception handling
│   │   ├── ApiException.java            # Base custom exception
│   │   ├── ResourceNotFoundException.java
│   │   └── BadRequestException.java
│   └── util/
│       └── RequestIdGenerator.java  # Generate unique request IDs
├── api/
│   └── controller/
│       └── HelloController.java     # Sample REST endpoint
├── config/
│   └── WebConfig.java               # Register interceptor
└── LibmgmtApplication.java

src/main/resources/
└── application.yml                  # Logging configuration
```

---

## Task 1: Create ApiResponse Generic Wrapper

**Files:**
- Create: `src/main/java/com/zahan/app/libmgmt/common/response/ApiResponse.java`

- [ ] **Step 1: Write test for ApiResponse builder**

Create test file: `src/test/java/com/zahan/app/libmgmt/common/response/ApiResponseTest.java`

```java
package com.zahan.app.libmgmt.common.response;

import org.junit.jupiter.api.Test;
import java.time.LocalDateTime;

import static org.junit.jupiter.api.Assertions.*;

public class ApiResponseTest {
    
    @Test
    public void testApiResponseBuilder_Success() {
        LocalDateTime now = LocalDateTime.now();
        String testData = "test";
        
        ApiResponse<String> response = ApiResponse.builder()
            .code(200)
            .message("Success")
            .data(testData)
            .timestamp(now)
            .requestId("req-123")
            .path("/api/hello")
            .duration(50L)
            .build();
        
        assertEquals(200, response.getCode());
        assertEquals("Success", response.getMessage());
        assertEquals("test", response.getData());
        assertEquals("req-123", response.getRequestId());
        assertEquals("/api/hello", response.getPath());
        assertEquals(50L, response.getDuration());
    }
    
    @Test
    public void testApiResponseBuilder_NullData() {
        ApiResponse<Object> response = ApiResponse.builder()
            .code(404)
            .message("Not Found")
            .data(null)
            .timestamp(LocalDateTime.now())
            .requestId("req-456")
            .path("/api/missing")
            .duration(10L)
            .build();
        
        assertNull(response.getData());
        assertEquals(404, response.getCode());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/hanzai/IDEAWorkSpace/lib-management-admin
mvn test -Dtest=ApiResponseTest#testApiResponseBuilder_Success -v
```

Expected: FAIL - class does not exist

- [ ] **Step 3: Create ApiResponse class**

```java
package com.zahan.app.libmgmt.common.response;

import com.fasterxml.jackson.annotation.JsonFormat;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ApiResponse<T> {
    
    private Integer code;
    
    private String message;
    
    private T data;
    
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
    private LocalDateTime timestamp;
    
    private String requestId;
    
    private String path;
    
    private Long duration;
    
    public static <T> ApiResponse<T> success(T data, String requestId, String path, Long duration) {
        return ApiResponse.<T>builder()
            .code(200)
            .message("Success")
            .data(data)
            .timestamp(LocalDateTime.now())
            .requestId(requestId)
            .path(path)
            .duration(duration)
            .build();
    }
    
    public static <T> ApiResponse<T> error(Integer code, String message, String requestId, String path, Long duration) {
        return ApiResponse.<T>builder()
            .code(code)
            .message(message)
            .data(null)
            .timestamp(LocalDateTime.now())
            .requestId(requestId)
            .path(path)
            .duration(duration)
            .build();
    }
}
```

Note: Add Lombok dependency to `pom.xml` if not already present:

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -Dtest=ApiResponseTest -v
```

Expected: PASS - 2 tests passed

- [ ] **Step 5: Commit**

```bash
git add pom.xml src/main/java/com/zahan/app/libmgmt/common/response/ApiResponse.java src/test/java/com/zahan/app/libmgmt/common/response/ApiResponseTest.java
git commit -m "feat: add ApiResponse generic wrapper class

- Generic response wrapper for all API endpoints
- Includes code, message, data, timestamp, requestId, path, duration
- Builder pattern for easy construction
- Success and error factory methods

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Task 2: Create Custom Exceptions

**Files:**
- Create: `src/main/java/com/zahan/app/libmgmt/common/exception/ApiException.java`
- Create: `src/main/java/com/zahan/app/libmgmt/common/exception/ResourceNotFoundException.java`
- Create: `src/main/java/com/zahan/app/libmgmt/common/exception/BadRequestException.java`

- [ ] **Step 1: Write test for exceptions**

Create: `src/test/java/com/zahan/app/libmgmt/common/exception/ApiExceptionTest.java`

```java
package com.zahan.app.libmgmt.common.exception;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

public class ApiExceptionTest {
    
    @Test
    public void testApiException_WithMessageAndCode() {
        ApiException ex = new ApiException("Test error", 400);
        
        assertEquals("Test error", ex.getMessage());
        assertEquals(400, ex.getCode());
    }
    
    @Test
    public void testResourceNotFoundException_IsApiException() {
        ResourceNotFoundException ex = new ResourceNotFoundException("User not found");
        
        assertTrue(ex instanceof ApiException);
        assertEquals("User not found", ex.getMessage());
        assertEquals(404, ex.getCode());
    }
    
    @Test
    public void testBadRequestException_IsApiException() {
        BadRequestException ex = new BadRequestException("Invalid input");
        
        assertTrue(ex instanceof ApiException);
        assertEquals("Invalid input", ex.getMessage());
        assertEquals(400, ex.getCode());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -Dtest=ApiExceptionTest -v
```

Expected: FAIL - classes do not exist

- [ ] **Step 3: Create ApiException base class**

```java
package com.zahan.app.libmgmt.common.exception;

public class ApiException extends RuntimeException {
    
    private final Integer code;
    
    public ApiException(String message, Integer code) {
        super(message);
        this.code = code;
    }
    
    public ApiException(String message, Integer code, Throwable cause) {
        super(message, cause);
        this.code = code;
    }
    
    public Integer getCode() {
        return code;
    }
}
```

- [ ] **Step 4: Create ResourceNotFoundException**

```java
package com.zahan.app.libmgmt.common.exception;

public class ResourceNotFoundException extends ApiException {
    
    public ResourceNotFoundException(String message) {
        super(message, 404);
    }
    
    public ResourceNotFoundException(String message, Throwable cause) {
        super(message, 404, cause);
    }
}
```

- [ ] **Step 5: Create BadRequestException**

```java
package com.zahan.app.libmgmt.common.exception;

public class BadRequestException extends ApiException {
    
    public BadRequestException(String message) {
        super(message, 400);
    }
    
    public BadRequestException(String message, Throwable cause) {
        super(message, 400, cause);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
mvn test -Dtest=ApiExceptionTest -v
```

Expected: PASS - 3 tests passed

- [ ] **Step 7: Commit**

```bash
git add src/main/java/com/zahan/app/libmgmt/common/exception/ApiException.java \
         src/main/java/com/zahan/app/libmgmt/common/exception/ResourceNotFoundException.java \
         src/main/java/com/zahan/app/libmgmt/common/exception/BadRequestException.java \
         src/test/java/com/zahan/app/libmgmt/common/exception/ApiExceptionTest.java
git commit -m "feat: add custom exception hierarchy

- ApiException base with HTTP code support
- ResourceNotFoundException (404)
- BadRequestException (400)
- All extend ApiException for centralized handling

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Task 3: Create RequestId Generator Utility

**Files:**
- Create: `src/main/java/com/zahan/app/libmgmt/common/util/RequestIdGenerator.java`

- [ ] **Step 1: Write test for RequestIdGenerator**

Create: `src/test/java/com/zahan/app/libmgmt/common/util/RequestIdGeneratorTest.java`

```java
package com.zahan.app.libmgmt.common.util;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

public class RequestIdGeneratorTest {
    
    @Test
    public void testGenerateRequestId_Format() {
        String requestId = RequestIdGenerator.generateRequestId();
        
        assertNotNull(requestId);
        assertTrue(requestId.startsWith("req-"));
        assertTrue(requestId.length() > 4);
    }
    
    @Test
    public void testGenerateRequestId_Uniqueness() {
        String id1 = RequestIdGenerator.generateRequestId();
        String id2 = RequestIdGenerator.generateRequestId();
        
        assertNotEquals(id1, id2);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -Dtest=RequestIdGeneratorTest -v
```

Expected: FAIL - class does not exist

- [ ] **Step 3: Create RequestIdGenerator utility**

```java
package com.zahan.app.libmgmt.common.util;

import java.util.UUID;

public class RequestIdGenerator {
    
    private RequestIdGenerator() {
    }
    
    public static String generateRequestId() {
        return "req-" + UUID.randomUUID().toString().substring(0, 13);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -Dtest=RequestIdGeneratorTest -v
```

Expected: PASS - 2 tests passed

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/zahan/app/libmgmt/common/util/RequestIdGenerator.java \
         src/test/java/com/zahan/app/libmgmt/common/util/RequestIdGeneratorTest.java
git commit -m "feat: add RequestIdGenerator utility for request tracking

- Generate unique request IDs with 'req-' prefix
- Uses UUID for uniqueness
- Utility class with no state

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Task 4: Create GlobalExceptionHandler

**Files:**
- Create: `src/main/java/com/zahan/app/libmgmt/common/exception/GlobalExceptionHandler.java`

- [ ] **Step 1: Create GlobalExceptionHandler**

```java
package com.zahan.app.libmgmt.common.exception;

import com.zahan.app.libmgmt.common.response.ApiResponse;
import com.zahan.app.libmgmt.common.util.RequestIdGenerator;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.context.request.WebRequest;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

@Slf4j
@ControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    
    @ExceptionHandler(ApiException.class)
    public ResponseEntity<ApiResponse<Object>> handleApiException(
            ApiException ex,
            WebRequest request) {
        
        String requestId = RequestIdGenerator.generateRequestId();
        String path = request.getDescription(false).replace("uri=", "");
        long duration = 0;
        
        log.error("[API] requestId={} | method={} | path={} | status={} | message={}",
            requestId, "UNKNOWN", path, ex.getCode(), ex.getMessage());
        
        ApiResponse<Object> response = ApiResponse.error(
            ex.getCode(),
            ex.getMessage(),
            requestId,
            path,
            duration
        );
        
        return new ResponseEntity<>(response, HttpStatus.valueOf(ex.getCode()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Object>> handleGenericException(
            Exception ex,
            WebRequest request) {
        
        String requestId = RequestIdGenerator.generateRequestId();
        String path = request.getDescription(false).replace("uri=", "");
        long duration = 0;
        
        log.error("[API] requestId={} | method={} | path={} | status={} | message={}",
            requestId, "UNKNOWN", path, 500, ex.getMessage(), ex);
        
        ApiResponse<Object> response = ApiResponse.error(
            500,
            "Internal Server Error",
            requestId,
            path,
            duration
        );
        
        return new ResponseEntity<>(response, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

- [ ] **Step 2: Verify GlobalExceptionHandler compiles**

```bash
mvn clean compile
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add src/main/java/com/zahan/app/libmgmt/common/exception/GlobalExceptionHandler.java
git commit -m "feat: add GlobalExceptionHandler for centralized error handling

- Catches ApiException and specific exception types
- Returns unified ApiResponse error format
- Logs errors with requestId tracking
- Maps custom exceptions to appropriate HTTP status codes

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Task 5: Create ApiResponseInterceptor

**Files:**
- Create: `src/main/java/com/zahan/app/libmgmt/common/interceptor/ApiResponseInterceptor.java`

- [ ] **Step 1: Create ApiResponseInterceptor**

```java
package com.zahan.app.libmgmt.common.interceptor;

import com.zahan.app.libmgmt.common.util.RequestIdGenerator;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.servlet.HandlerInterceptor;

@Slf4j
public class ApiResponseInterceptor implements HandlerInterceptor {
    
    private static final String REQUEST_ID_ATTR = "requestId";
    private static final String START_TIME_ATTR = "startTime";
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String requestId = RequestIdGenerator.generateRequestId();
        request.setAttribute(REQUEST_ID_ATTR, requestId);
        request.setAttribute(START_TIME_ATTR, System.currentTimeMillis());
        
        String method = request.getMethod();
        String path = request.getRequestURI();
        
        log.info("[API] requestId={} | method={} | path={} | START",
            requestId, method, path);
        
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                               Object handler, Exception ex) throws Exception {
        String requestId = (String) request.getAttribute(REQUEST_ID_ATTR);
        Long startTime = (Long) request.getAttribute(START_TIME_ATTR);
        
        String method = request.getMethod();
        String path = request.getRequestURI();
        int status = response.getStatus();
        
        long duration = System.currentTimeMillis() - startTime;
        
        log.info("[API] requestId={} | method={} | path={} | status={} | duration={}ms",
            requestId, method, path, status, duration);
    }
}
```

- [ ] **Step 2: Verify ApiResponseInterceptor compiles**

```bash
mvn clean compile
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add src/main/java/com/zahan/app/libmgmt/common/interceptor/ApiResponseInterceptor.java
git commit -m "feat: add ApiResponseInterceptor for request/response logging

- Logs all HTTP requests with method, path, requestId
- Calculates response duration in milliseconds
- Captures request and response lifecycle
- Uses unique requestId for request tracing

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Task 6: Create WebConfig to Register Interceptor

**Files:**
- Create: `src/main/java/com/zahan/app/libmgmt/config/WebConfig.java`

- [ ] **Step 1: Create WebConfig**

```java
package com.zahan.app.libmgmt.config;

import com.zahan.app.libmgmt.common.interceptor.ApiResponseInterceptor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new ApiResponseInterceptor())
            .addPathPatterns("/api/**");
    }
}
```

- [ ] **Step 2: Verify WebConfig compiles**

```bash
mvn clean compile
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add src/main/java/com/zahan/app/libmgmt/config/WebConfig.java
git commit -m "feat: add WebConfig to register ApiResponseInterceptor

- Registers interceptor for all /api/** paths
- Enables request/response logging for all API endpoints

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Task 7: Create Sample HelloController

**Files:**
- Create: `src/main/java/com/zahan/app/libmgmt/api/controller/HelloController.java`

- [ ] **Step 1: Write integration test for HelloController**

Create: `src/test/java/com/zahan/app/libmgmt/api/controller/HelloControllerTest.java`

```java
package com.zahan.app.libmgmt.api.controller;

import com.zahan.app.libmgmt.common.response.ApiResponse;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
public class HelloControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    public void testHelloEndpoint_ReturnsSuccess() throws Exception {
        mockMvc.perform(get("/api/hello"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(200))
            .andExpect(jsonPath("$.message").value("Success"))
            .andExpect(jsonPath("$.data.greeting").exists())
            .andExpect(jsonPath("$.requestId").exists())
            .andExpect(jsonPath("$.path").value("/api/hello"))
            .andExpect(jsonPath("$.timestamp").exists())
            .andExpect(jsonPath("$.duration").exists());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -Dtest=HelloControllerTest -v
```

Expected: FAIL - endpoint does not exist

- [ ] **Step 3: Create HelloController**

```java
package com.zahan.app.libmgmt.api.controller;

import com.zahan.app.libmgmt.common.response.ApiResponse;
import com.zahan.app.libmgmt.common.util.RequestIdGenerator;
import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api")
public class HelloController {
    
    @GetMapping("/hello")
    public ApiResponse<Map<String, String>> hello(HttpServletRequest request) {
        long startTime = (Long) request.getAttribute("startTime");
        String requestId = (String) request.getAttribute("requestId");
        
        long duration = System.currentTimeMillis() - startTime;
        
        Map<String, String> data = new HashMap<>();
        data.put("greeting", "Hello, Welcome to Library Management Admin API!");
        
        return ApiResponse.success(data, requestId, request.getRequestURI(), duration);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -Dtest=HelloControllerTest -v
```

Expected: PASS - 1 test passed

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/zahan/app/libmgmt/api/controller/HelloController.java \
         src/test/java/com/zahan/app/libmgmt/api/controller/HelloControllerTest.java
git commit -m "feat: add HelloController sample REST endpoint

- GET /api/hello returns greeting message
- Demonstrates ApiResponse wrapper with all fields
- Uses interceptor-captured requestId and timing
- Sample endpoint for testing web integration

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Task 8: Configure Logging in application.yml

**Files:**
- Modify: `src/main/resources/application.yml`

- [ ] **Step 1: Create or update application.yml with logging configuration**

If file does not exist, create it:

```yaml
spring:
  application:
    name: libmgmt

logging:
  level:
    root: INFO
    com.zahan.app.libmgmt: DEBUG
    com.zahan.app.libmgmt.common.interceptor: INFO
    com.zahan.app.libmgmt.common.exception: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
```

If file exists, add/merge the logging section.

- [ ] **Step 2: Verify application.yml is valid YAML**

```bash
mvn clean compile
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/application.yml
git commit -m "config: add logging configuration for API interceptor and exceptions

- Set root log level to INFO
- Set lib-management-admin to DEBUG for detailed logs
- Configure console pattern with timestamp and thread info
- Enables comprehensive request/response and exception logging

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Task 9: Run Full Test Suite

**Files:**
- Test: All test classes created in previous tasks

- [ ] **Step 1: Run complete test suite**

```bash
mvn clean test -v
```

Expected: BUILD SUCCESS with all tests passing (should show ~9+ tests)

- [ ] **Step 2: Review test output for failures**

If any test fails, review the failure message and fix the corresponding implementation.

- [ ] **Step 3: Build the application**

```bash
mvn clean package -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Verify JAR is created**

```bash
ls -lh target/libmgmt-*.jar
```

Expected: JAR file exists in target/

- [ ] **Step 5: Commit final state**

```bash
git add -A
git commit -m "test: verify all tests pass and application builds successfully

- All unit and integration tests passing
- Full build successful
- Ready for deployment

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Self-Review Against Spec

**Spec Coverage Check:**

1. ✅ **Unified API Response Format** (Task 1) - ApiResponse with code, message, data, timestamp, requestId, path, duration
2. ✅ **Exception Handling** (Task 2, 4) - Custom exceptions, GlobalExceptionHandler for centralized handling
3. ✅ **Request/Response Logging** (Task 5) - ApiResponseInterceptor logs all requests verbosely
4. ✅ **Sample Hello Endpoint** (Task 7) - HelloController demonstrates API response format
5. ✅ **Logging Configuration** (Task 8) - application.yml with proper logging levels
6. ✅ **Interceptor Registration** (Task 6) - WebConfig registers interceptor for /api/** paths

**Placeholder Scan:** ✅ No TBDs, all code complete, all commands explicit

**Type Consistency:** ✅ ApiResponse<T> used consistently, requestId String type throughout, duration Long type throughout

**Scope:** ✅ Single focus - web integration layer, appropriate for one implementation plan

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-04-28-web-integration-implementation.md`. 

**Two execution options:**

**1. Subagent-Driven (recommended)** - Fresh subagent per task, systematic review, fast iteration with quality gates

**2. Inline Execution** - Execute all tasks in this session with checkpoints for review between major phases

Which approach would you prefer?
