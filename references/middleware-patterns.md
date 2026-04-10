# Middleware Patterns

This guide covers middleware and interceptor implementation in go-kratos.

## Overview

Middleware in go-kratos provides cross-cutting concerns like logging, authentication, metrics, and tracing. Both HTTP and gRPC use the same middleware interface.

## Middleware Interface

```go
// Middleware function signature
type Middleware func(Handler) Handler

// Handler function signature
type Handler func(ctx context.Context, req interface{}) (interface{}, error)
```

## Built-in Middlewares

### Recovery

```go
import "github.com/go-kratos/kratos/v2/middleware/recovery"

// Default recovery
recovery.Recovery()

// Custom recovery handler
recovery.Recovery(
    recovery.WithHandler(func(ctx context.Context, req, err interface{}) error {
        // Custom panic handling
        log.Errorf("Panic recovered: %v", err)
        return errors.InternalServer("PANIC", "internal server error")
    }),
)
```

### Logging

```go
import "github.com/go-kratos/kratos/v2/middleware/logging"

// Server-side logging
logging.Server(logger)

// Client-side logging
logging.Client(logger)
```

### Validation

```go
import "github.com/go-kratos/kratos/v2/middleware/validate"

// Proto validation using validate.rules
validate.Validator()
```

### Metadata

```go
import "github.com/go-kratos/kratos/v2/middleware/metadata"

// Server-side metadata
metadata.Server()

// Client-side metadata
metadata.Client()
```

## Custom Middleware

### Authentication Middleware

### ✅ Correct Pattern

```go
// internal/middleware/auth.go
package middleware

import (
    "context"
    "strings"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/go-kratos/kratos/v2/transport"
)

// context key type
type userContextKey struct{}

// UserClaims represents JWT claims
type UserClaims struct {
    UserID   int64
    Username string
    Roles    []string
}

// Auth middleware validates JWT token
func Auth(secret string) middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            // Extract token from transport
            tr, ok := transport.FromServerContext(ctx)
            if !ok {
                return nil, errors.Unauthorized("TRANSPORT_ERROR", "transport not found")
            }
            
            tokenString := tr.RequestHeader().Get("Authorization")
            if tokenString == "" {
                return nil, errors.Unauthorized("MISSING_TOKEN", "authorization token required")
            }
            
            // Remove "Bearer " prefix
            tokenString = strings.TrimPrefix(tokenString, "Bearer ")
            
            // Validate token
            claims, err := validateToken(tokenString, secret)
            if err != nil {
                return nil, errors.Unauthorized("INVALID_TOKEN", err.Error())
            }
            
            // Add claims to context
            ctx = context.WithValue(ctx, userContextKey{}, claims)
            
            return handler(ctx, req)
        }
    }
}

// GetUserClaims extracts user claims from context
func GetUserClaims(ctx context.Context) (*UserClaims, bool) {
    claims, ok := ctx.Value(userContextKey{}).(*UserClaims)
    return claims, ok
}

// validateToken validates JWT token
func validateToken(tokenString, secret string) (*UserClaims, error) {
    // Implement JWT validation
    // Using github.com/golang-jwt/jwt/v5
    token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte(secret), nil
    })
    
    if err != nil || !token.Valid {
        return nil, errors.New("invalid token")
    }
    
    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok {
        return nil, errors.New("invalid claims")
    }
    
    return &UserClaims{
        UserID:   int64(claims["user_id"].(float64)),
        Username: claims["username"].(string),
    }, nil
}
```

### Authorization Middleware (RBAC)

```go
// internal/middleware/rbac.go
package middleware

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/middleware"
)

// RBAC middleware checks user roles
type RBACConfig struct {
    // Map of method to required roles
    MethodRoles map[string][]string
}

// RBAC creates RBAC middleware
func RBAC(config *RBACConfig) middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            // Get user claims
            claims, ok := GetUserClaims(ctx)
            if !ok {
                return nil, errors.Forbidden("NO_CLAIMS", "user claims not found")
            }
            
            // Get method name from context
            method, _ := grpc.Method(ctx)
            
            // Check if method requires roles
            requiredRoles, exists := config.MethodRoles[method]
            if !exists {
                // No roles required for this method
                return handler(ctx, req)
            }
            
            // Check if user has required role
            hasRole := false
            for _, role := range requiredRoles {
                for _, userRole := range claims.Roles {
                    if role == userRole {
                        hasRole = true
                        break
                    }
                }
            }
            
            if !hasRole {
                return nil, errors.Forbidden("INSUFFICIENT_PERMISSIONS", "insufficient permissions")
            }
            
            return handler(ctx, req)
        }
    }
}
```

### Metrics Middleware

```go
// internal/middleware/metrics.go
package middleware

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "kratos_request_duration_seconds",
            Help:    "Request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "status"},
    )
    
    requestTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "kratos_request_total",
            Help: "Total number of requests",
        },
        []string{"method", "status"},
    )
)

// Metrics middleware records request metrics
func Metrics() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            start := time.Now()
            
            reply, err := handler(ctx, req)
            
            duration := time.Since(start).Seconds()
            method, _ := grpc.Method(ctx)
            status := "success"
            if err != nil {
                status = "error"
            }
            
            requestDuration.WithLabelValues(method, status).Observe(duration)
            requestTotal.WithLabelValues(method, status).Inc()
            
            return reply, err
        }
    }
}
```

### Tracing Middleware

```go
// internal/middleware/tracing.go
package middleware

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/go-kratos/kratos/v2/middleware/tracing"
)

// Tracing middleware configuration
func Tracing() middleware.Middleware {
    return tracing.Server(
        tracing.WithPropagator(propagation.TraceContext{}),
    )
}
```

### Rate Limiting Middleware

```go
// internal/middleware/ratelimit.go
package middleware

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/middleware"
    "golang.org/x/time/rate"
)

// RateLimiter middleware limits request rate
type RateLimiter struct {
    limiter *rate.Limiter
}

// NewRateLimiter creates rate limiter middleware
func NewRateLimiter(rps int) middleware.Middleware {
    rl := &RateLimiter{
        limiter: rate.NewLimiter(rate.Limit(rps), rps),
    }
    
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            if !rl.limiter.Allow() {
                return nil, errors.TooManyRequests("RATE_LIMIT", "rate limit exceeded")
            }
            return handler(ctx, req)
        }
    }
}
```

## Middleware Chain

### ✅ Correct Pattern

```go
// internal/server/http.go
func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            // Order matters!
            recovery.Recovery(),           // 1. Recover from panics
            tracing.Tracing(),              // 2. Start trace span
            logging.Server(logger),         // 3. Log requests
            metadata.Server(),              // 4. Extract metadata
            Auth(c.Auth.Secret),            // 5. Authenticate
            RBAC(rbacConfig),               // 6. Authorize
            validate.Validator(),           // 7. Validate request
            Metrics(),                      // 8. Record metrics
            NewRateLimiter(100),            // 9. Rate limit
        ),
    }
    // ...
}
```

### Middleware Order Best Practices

| Order | Middleware | Purpose |
|-------|------------|---------|
| 1 | Recovery | Catch panics first |
| 2 | Tracing | Start span early |
| 3 | Logging | Log all requests |
| 4 | Metadata | Extract context |
| 5 | Auth | Authenticate user |
| 6 | RBAC | Check permissions |
| 7 | Validation | Validate input |
| 8 | Metrics | Record metrics |
| 9 | Rate Limit | Protect resources |

## Conditional Middleware

### Path-based Middleware

```go
// internal/server/http.go

func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
        ),
    }
    
    srv := http.NewServer(opts...)
    
    // Register public routes (no auth)
    v1.RegisterGreeterHTTPServer(srv, greeterService)
    
    // Register protected routes (with auth)
    authMiddleware := http.Middleware(
        Auth(c.Auth.Secret),
        validate.Validator(),
    )
    v1.RegisterUserHTTPServer(srv, userService, authMiddleware)
    
    return srv
}
```

## Testing Middleware

```go
// internal/middleware/auth_test.go
package middleware

import (
    "context"
    "testing"
    
    "github.com/stretchr/testify/assert"
)

func TestAuth(t *testing.T) {
    authMiddleware := Auth("test-secret")
    
    handler := func(ctx context.Context, req interface{}) (interface{}, error) {
        claims, ok := GetUserClaims(ctx)
        assert.True(t, ok)
        assert.Equal(t, int64(1), claims.UserID)
        return "success", nil
    }
    
    // Test with valid token
    ctx := transport.NewServerContext(context.Background(), &mockTransport{
        header: map[string][]string{
            "Authorization": {"Bearer valid-token"},
        },
    })
    
    reply, err := authMiddleware(handler)(ctx, nil)
    assert.NoError(t, err)
    assert.Equal(t, "success", reply)
    
    // Test without token
    ctx = transport.NewServerContext(context.Background(), &mockTransport{
        header: map[string][]string{},
    })
    
    _, err = authMiddleware(handler)(ctx, nil)
    assert.Error(t, err)
    assert.Contains(t, err.Error(), "MISSING_TOKEN")
}
```

## Summary

### ✅ Always Follow

- Use middleware for cross-cutting concerns
- Order middleware correctly (recovery first)
- Extract context values properly
- Use conditional middleware for different routes
- Test middleware independently
- Keep middleware focused and single-purpose

### ❌ Never Do

- Put business logic in middleware
- Skip recovery middleware
- Use wrong middleware order
- Create tightly coupled middleware
- Ignore context propagation
- Forget to test middleware
