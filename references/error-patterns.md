# Error Handling Patterns (go-kratos)

This guide covers error handling patterns in go-kratos, including error definitions, error codes, and error propagation across layers.

## Core Concepts

- Use `github.com/go-kratos/kratos/v2/errors` package for all errors
- Define domain-specific errors with unique reason codes
- Propagate errors through context without losing information
- Convert errors to appropriate transport formats at the boundary

## Error Structure

```go
// errors.Error structure
type Error struct {
    Code     int               // HTTP/gRPC status code
    Reason   string            // Machine-readable error reason
    Message  string            // Human-readable error message
    Metadata map[string]string // Additional error context
}
```

---

## Basic Error Handling

### Creating Errors

```go
import "github.com/go-kratos/kratos/v2/errors"

// Simple error
err := errors.New(404, "USER_NOT_FOUND", "user not found")

// Error with formatting
err := errors.New(404, "USER_NOT_FOUND", "user %d not found", userID)

// Error with metadata
err := errors.New(400, "VALIDATION_ERROR", "validation failed").
    WithMetadata(map[string]string{
        "field": "email",
        "value": "invalid-email",
    })
```

### Predefined Error Functions

```go
// Common HTTP status errors
errors.BadRequest(reason, message)           // 400
errors.Unauthorized(reason, message)         // 401
errors.Forbidden(reason, message)            // 403
errors.NotFound(reason, message)             // 404
errors.Conflict(reason, message)             // 409
errors.TooManyRequests(reason, message)      // 429
errors.InternalServer(reason, message)       // 500
errors.ServiceUnavailable(reason, message)   // 503
errors.GatewayTimeout(reason, message)       // 504
```

### Error Checking

```go
// Check error code
if errors.Code(err) == 404 {
    // Handle not found
}

// Check error reason
e := errors.FromError(err)
if e.Reason == "USER_NOT_FOUND" {
    // Handle user not found
}

// Check if error is of specific type
if errors.IsNotFound(err) {
    // Handle not found
}

// Check error code range
code := errors.Code(err)
if code >= 400 && code < 500 {
    // Handle client error
}
```

---

## Domain Error Definitions

### Method 1: Package-level Error Variables

```go
// internal/biz/errors.go
package biz

import "github.com/go-kratos/kratos/v2/errors"

// User domain errors
var (
    ErrUserNotFound    = errors.NotFound("USER_NOT_FOUND", "user not found")
    ErrUserExists      = errors.Conflict("USER_EXISTS", "user already exists")
    ErrInvalidEmail    = errors.BadRequest("INVALID_EMAIL", "invalid email format")
    ErrInvalidPassword = errors.BadRequest("INVALID_PASSWORD", "password too weak")
    ErrUserBanned      = errors.Forbidden("USER_BANNED", "user is banned")
)

// Order domain errors
var (
    ErrOrderNotFound    = errors.NotFound("ORDER_NOT_FOUND", "order not found")
    ErrInvalidOrderStatus = errors.BadRequest("INVALID_ORDER_STATUS", "invalid order status")
    ErrInsufficientStock  = errors.Conflict("INSUFFICIENT_STOCK", "insufficient stock")
)
```

### Method 2: Proto-defined Errors (Recommended)

Define errors in proto file for code generation:

```protobuf
// api/errors/error_reason.proto
syntax = "proto3";

package api.errors;

import "errors/errors.proto";

option go_package = "user-service/api/errors;v1";

enum ErrorReason {
    // Default error code for all errors in this enum
    option (errors.default_code) = 500;

    // User errors
    USER_NOT_FOUND = 0 [(errors.code) = 404];
    USER_EXISTS = 1 [(errors.code) = 409];
    INVALID_EMAIL = 2 [(errors.code) = 400];
    INVALID_PASSWORD = 3 [(errors.code) = 400];
    USER_BANNED = 4 [(errors.code) = 403];

    // Order errors
    ORDER_NOT_FOUND = 10 [(errors.code) = 404];
    INVALID_ORDER_STATUS = 11 [(errors.code) = 400];
    INSUFFICIENT_STOCK = 12 [(errors.code) = 409];
}
```

Generate code:
```bash
kratos proto client api/errors/error_reason.proto
```

Generated usage:
```go
import "user-service/api/errors"

// Create error
err := errors.ErrorUserNotFound("user %d not found", userID)

// Check error
if errors.IsUserNotFound(err) {
    // Handle user not found
}

// Get error code
code := errors.ErrorUserNotFound().Code  // 404
```

---

## Error Handling in Layers

### Biz Layer

```go
// internal/biz/user.go
package biz

import (
    "context"
    "github.com/go-kratos/kratos/v2/errors"
)

func (uc *UserUsecase) GetUser(ctx context.Context, id int64) (*User, error) {
    user, err := uc.repo.Get(ctx, id)
    if err != nil {
        if errors.IsNotFound(err) {
            // Return domain error
            return nil, ErrUserNotFound
        }
        // Wrap internal error
        return nil, errors.InternalServer("DATABASE_ERROR", "failed to get user").
            WithCause(err)
    }
    
    if user.Status == UserStatusBanned {
        return nil, ErrUserBanned
    }
    
    return user, nil
}

func (uc *UserUsecase) CreateUser(ctx context.Context, u *User) error {
    // Check if email exists
    existing, err := uc.repo.GetByEmail(ctx, u.Email)
    if err != nil && !errors.IsNotFound(err) {
        return err
    }
    if existing != nil {
        return ErrUserExists
    }
    
    // Validate email
    if !isValidEmail(u.Email) {
        return ErrInvalidEmail
    }
    
    return uc.repo.Create(ctx, u)
}
```

### Data Layer

```go
// internal/data/user.go
package data

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/errors"
    "gorm.io/gorm"
)

func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    var user User
    result := r.data.db.WithContext(ctx).First(&user, id)
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrRecordNotFound) {
            // Return not found error
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        // Return internal error with cause
        return nil, errors.InternalServer("DATABASE_ERROR", "database error").
            WithCause(result.Error)
    }
    return toBizUser(&user), nil
}
```

### Service Layer

```go
// internal/service/userservice.go
package service

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/errors"
    v1 "user-service/api/user/v1"
)

func (s *UserService) GetUser(ctx context.Context, req *v1.GetUserRequest) (*v1.GetUserReply, error) {
    user, err := s.uc.GetUser(ctx, req.Id)
    if err != nil {
        // Error is already properly formatted, just return it
        return nil, err
    }
    
    return &v1.GetUserReply{
        Id:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}
```

---

## Error Wrapping and Unwrapping

### Wrapping Errors with Context

```go
import "github.com/go-kratos/kratos/v2/errors"

// Original error
dbErr := db.Query(...)

// Wrap with context
err := errors.InternalServer("DATABASE_ERROR", "failed to fetch user").
    WithCause(dbErr)

// Add metadata
err = err.WithMetadata(map[string]string{
    "user_id": "12345",
    "table":   "users",
})
```

### Unwrapping Errors

```go
// Get root cause
if cause := errors.Cause(err); cause != nil {
    log.Printf("Root cause: %v", cause)
}

// Check if error contains specific error
if errors.Is(err, gorm.ErrRecordNotFound) {
    // Handle record not found
}

// Convert to kratos error
kratosErr := errors.FromError(err)
fmt.Printf("Code: %d, Reason: %s, Message: %s\n",
    kratosErr.Code,
    kratosErr.Reason,
    kratosErr.Message,
)
```

---

## Error Code Mapping

### HTTP to gRPC Code Mapping

| HTTP Status | gRPC Code | Kratos Function |
|-------------|-----------|-----------------|
| 200 | 0 (OK) | - |
| 400 | 3 (InvalidArgument) | `errors.BadRequest()` |
| 401 | 16 (Unauthenticated) | `errors.Unauthorized()` |
| 403 | 7 (PermissionDenied) | `errors.Forbidden()` |
| 404 | 5 (NotFound) | `errors.NotFound()` |
| 409 | 6 (AlreadyExists) | `errors.Conflict()` |
| 429 | 8 (ResourceExhausted) | `errors.TooManyRequests()` |
| 500 | 13 (Internal) | `errors.InternalServer()` |
| 502 | 14 (Unavailable) | `errors.ServiceUnavailable()` |
| 503 | 14 (Unavailable) | `errors.ServiceUnavailable()` |
| 504 | 4 (DeadlineExceeded) | `errors.GatewayTimeout()` |

---

## Custom Error Encoder/Decoder

### HTTP Error Encoder

```go
// internal/server/http.go
package server

import (
    stdhttp "net/http"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/transport/http"
)

// CustomError represents custom error response
type CustomError struct {
    Code      int    `json:"code"`
    Reason    string `json:"reason"`
    Message   string `json:"message"`
    RequestID string `json:"request_id,omitempty"`
    Path      string `json:"path,omitempty"`
}

func errorEncoder(w stdhttp.ResponseWriter, r *stdhttp.Request, err error) {
    se := errors.FromError(err)
    
    codec, _ := http.CodecForRequest(r, "Accept")
    
    // Get request ID from context if available
    requestID := r.Header.Get("X-Request-ID")
    
    body := CustomError{
        Code:      int(se.Code),
        Reason:    se.Reason,
        Message:   se.Message,
        RequestID: requestID,
        Path:      r.URL.Path,
    }
    
    data, _ := codec.Marshal(body)
    
    w.Header().Set("Content-Type", "application/"+codec.Name())
    w.WriteHeader(http.StatusOK) // Always return 200 for consistent response
    w.Write(data)
}

// NewHTTPServer with custom error encoder
func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.ErrorEncoder(errorEncoder),
    }
    
    srv := http.NewServer(opts...)
    return srv
}
```

### gRPC Status Conversion

```go
import (
    "github.com/go-kratos/kratos/v2/errors"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func toGRPCError(err error) error {
    se := errors.FromError(err)
    
    var code codes.Code
    switch se.Code {
    case 400:
        code = codes.InvalidArgument
    case 401:
        code = codes.Unauthenticated
    case 403:
        code = codes.PermissionDenied
    case 404:
        code = codes.NotFound
    case 409:
        code = codes.AlreadyExists
    case 429:
        code = codes.ResourceExhausted
    case 500:
        code = codes.Internal
    case 503:
        code = codes.Unavailable
    case 504:
        code = codes.DeadlineExceeded
    default:
        code = codes.Unknown
    }
    
    st := status.New(code, se.Message)
    return st.Err()
}
```

---

## Error Logging and Monitoring

### Structured Error Logging

```go
// internal/middleware/logging.go
package middleware

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/middleware"
)

func ErrorLogging(logger log.Logger) middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            reply, err := handler(ctx, req)
            
            if err != nil {
                se := errors.FromError(err)
                
                // Log with structured fields
                log.Context(ctx).Errorw(
                    "request_error",
                    "code", se.Code,
                    "reason", se.Reason,
                    "message", se.Message,
                    "metadata", se.Metadata,
                )
                
                // Send to error tracking service (e.g., Sentry)
                // sentry.CaptureException(err)
            }
            
            return reply, err
        }
    }
}
```

### Error Metrics

```go
// internal/middleware/metrics.go
package middleware

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/prometheus/client_golang/prometheus"
)

var (
    errorCounter = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "kratos_errors_total",
            Help: "Total number of errors by reason",
        },
        []string{"reason", "code"},
    )
)

func ErrorMetrics() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            reply, err := handler(ctx, req)
            
            if err != nil {
                se := errors.FromError(err)
                errorCounter.WithLabelValues(
                    se.Reason,
                    fmt.Sprintf("%d", se.Code),
                ).Inc()
            }
            
            return reply, err
        }
    }
}
```

---

## Best Practices

### ✅ Always Follow

- Define domain-specific errors with unique reason codes
- Use proto-generated errors for consistency
- Return errors at the correct abstraction level
- Wrap internal errors with context using `WithCause`
- Use predefined error functions (`errors.NotFound()`, etc.)
- Check errors using `errors.Is()` and `errors.IsNotFound()` helpers
- Log errors with structured fields for observability

### ❌ Never Do

```go
// DON'T: Use generic errors
return fmt.Errorf("user not found")  // ❌
return errors.New(404, "USER_NOT_FOUND", "user not found")  // ✅

// DON'T: Expose internal details
return errors.InternalServer("DB_CONNECTION_FAILED", err.Error())  // ❌
return errors.InternalServer("INTERNAL_ERROR", "internal server error")  // ✅

// DON'T: Create errors without codes
return errors.New(0, "UNKNOWN", "unknown error")  // ❌
return errors.InternalServer("UNKNOWN", "unknown error")  // ✅

// DON'T: Swallow errors
if err != nil {
    return nil  // ❌
}
if err != nil {
    return nil, err  // ✅
}

// DON'T: Check error messages
if err.Error() == "user not found" {  // ❌
    // ...
}
if errors.IsNotFound(err) {  // ✅
    // ...
}
```

---

## Testing Error Handling

```go
// internal/biz/user_test.go
package biz

import (
    "context"
    "testing"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/stretchr/testify/assert"
)

func TestUserUsecase_GetUser_NotFound(t *testing.T) {
    mockRepo := new(MockUserRepo)
    uc := NewUserUsecase(mockRepo, log.DefaultLogger)
    
    mockRepo.On("Get", mock.Anything, int64(999)).
        Return(nil, errors.NotFound("USER_NOT_FOUND", ""))
    
    user, err := uc.GetUser(context.Background(), 999)
    
    assert.Error(t, err)
    assert.Nil(t, user)
    assert.True(t, errors.IsNotFound(err))
    assert.Equal(t, "USER_NOT_FOUND", errors.FromError(err).Reason)
}

func TestUserUsecase_CreateUser_AlreadyExists(t *testing.T) {
    mockRepo := new(MockUserRepo)
    uc := NewUserUsecase(mockRepo, log.DefaultLogger)
    
    mockRepo.On("GetByEmail", mock.Anything, "john@example.com").
        Return(&User{ID: 1}, nil)
    
    err := uc.CreateUser(context.Background(), &User{
        Email: "john@example.com",
    })
    
    assert.Error(t, err)
    assert.True(t, errors.IsConflict(err))
}
```
