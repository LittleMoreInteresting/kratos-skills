# HTTP API Patterns (go-kratos)

## Handler responsibilities

In go-kratos, HTTP handlers usually live in `internal/service` and should:

- parse/validate request transport objects
- call biz use-cases
- map results/errors to response DTOs

Avoid putting business branching logic here.

## Suggested flow

1. `service` receives request DTO.
2. `service` invokes `biz` method with context.
3. `biz` orchestrates domain rules and calls repo interfaces.
4. `data` persists/fetches.
5. `service` maps domain output to response DTO.

## Error mapping

- Use domain errors in `biz` (e.g., not found, conflict, invalid state).
- Convert to transport status in `service` (HTTP status or gRPC codes).
- Keep message stable and non-sensitive.

## Middleware baseline

- recovery
- logging
- tracing
- metadata / request-id propagation
- authn/authz (as needed)
- rate limiting (public APIs)

---

## Complete HTTP API Workflow

### 1. HTTP Server Setup

```go
// internal/server/http.go
package server

import (
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/go-kratos/kratos/v2/middleware/logging"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/tracing"
    "github.com/go-kratos/kratos/v2/middleware/validate"
    "github.com/go-kratos/kratos/v2/transport/http"
    
    v1 "user-service/api/user/v1"
    "user-service/internal/conf"
    "user-service/internal/service"
)

// NewHTTPServer creates an HTTP server
func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            recovery.Recovery(),      // 1. Recover from panics
            tracing.Server(),         // 2. Start trace span
            logging.Server(logger),   // 3. Log requests
            validate.Validator(),     // 4. Validate request
        ),
    }
    
    if c.Http.Network != "" {
        opts = append(opts, http.Network(c.Http.Network))
    }
    if c.Http.Addr != "" {
        opts = append(opts, http.Address(c.Http.Addr))
    }
    if c.Http.Timeout != nil {
        opts = append(opts, http.Timeout(c.Http.Timeout.AsDuration()))
    }
    
    srv := http.NewServer(opts...)
    
    // Register HTTP handlers from generated code
    v1.RegisterUserHTTPServer(srv, user)
    
    return srv
}
```

### 2. Service Layer Implementation

```go
// internal/service/userservice.go
package service

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/log"
    v1 "user-service/api/user/v1"
    "user-service/internal/biz"
)

type UserService struct {
    v1.UnimplementedUserServer
    uc  *biz.UserUsecase
    log *log.Helper
}

func NewUserService(uc *biz.UserUsecase, logger log.Logger) *UserService {
    return &UserService{
        uc:  uc,
        log: log.NewHelper(logger),
    }
}

// CreateUser creates a new user
func (s *UserService) CreateUser(ctx context.Context, req *v1.CreateUserRequest) (*v1.CreateUserReply, error) {
    // Call biz layer
    user, err := s.uc.Create(ctx, &biz.User{
        Name:  req.Name,
        Email: req.Email,
        Age:   req.Age,
    })
    if err != nil {
        return nil, err
    }
    
    // Map to response DTO
    return &v1.CreateUserReply{
        Id:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}

// GetUser gets user by ID
func (s *UserService) GetUser(ctx context.Context, req *v1.GetUserRequest) (*v1.GetUserReply, error) {
    user, err := s.uc.Get(ctx, req.Id)
    if err != nil {
        return nil, err
    }
    
    return &v1.GetUserReply{
        Id:    user.ID,
        Name:  user.Name,
        Email: user.Email,
        Age:   user.Age,
    }, nil
}

// UpdateUser updates user information
func (s *UserService) UpdateUser(ctx context.Context, req *v1.UpdateUserRequest) (*v1.UpdateUserReply, error) {
    user, err := s.uc.Update(ctx, req.Id, req.Name, req.Age)
    if err != nil {
        return nil, err
    }
    
    return &v1.UpdateUserReply{
        Id:   user.ID,
        Name: user.Name,
        Age:  user.Age,
    }, nil
}

// DeleteUser deletes a user
func (s *UserService) DeleteUser(ctx context.Context, req *v1.DeleteUserRequest) (*v1.DeleteUserReply, error) {
    err := s.uc.Delete(ctx, req.Id)
    if err != nil {
        return nil, err
    }
    
    return &v1.DeleteUserReply{
        Success: true,
    }, nil
}

// ListUsers lists users with pagination
func (s *UserService) ListUsers(ctx context.Context, req *v1.ListUsersRequest) (*v1.ListUsersReply, error) {
    users, total, err := s.uc.List(ctx, req.Page, req.PageSize)
    if err != nil {
        return nil, err
    }
    
    reply := &v1.ListUsersReply{
        Total:    total,
        Page:     req.Page,
        PageSize: req.PageSize,
    }
    
    for _, u := range users {
        reply.Users = append(reply.Users, &v1.User{
            Id:    u.ID,
            Name:  u.Name,
            Email: u.Email,
            Age:   u.Age,
        })
    }
    
    return reply, nil
}
```

### 3. Custom HTTP Routes

For non-gRPC-gateway routes, use direct route registration:

```go
// internal/server/http.go
func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    srv := http.NewServer(opts...)
    
    // Register generated handlers
    v1.RegisterUserHTTPServer(srv, user)
    
    // Register custom routes
    router := srv.Route("/")
    
    // Health check
    router.GET("/health", func(ctx http.Context) error {
        return ctx.JSON(200, map[string]string{
            "status": "healthy",
        })
    })
    
    // Metrics endpoint
    router.GET("/metrics", func(ctx http.Context) error {
        // Return prometheus metrics
        return ctx.String(200, "metrics data")
    })
    
    return srv
}
```

---

## HTTP Error Handling

### Custom Error Encoder

```go
// internal/server/http.go
package server

import (
    stdhttp "net/http"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/transport/http"
)

// HTTPError represents HTTP error response
type HTTPError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Details string `json:"details,omitempty"`
}

func (e *HTTPError) Error() string {
    return e.Message
}

// errorEncoder encodes error to HTTP response
func errorEncoder(w stdhttp.ResponseWriter, r *stdhttp.Request, err error) {
    se := errors.FromError(err)
    
    codec, _ := http.CodecForRequest(r, "Accept")
    
    body := &HTTPError{
        Code:    int(se.Code),
        Message: se.Message,
        Details: se.Metadata["details"],
    }
    
    data, _ := codec.Marshal(body)
    
    w.Header().Set("Content-Type", "application/"+codec.Name())
    w.WriteHeader(int(se.Code) / 100) // Convert gRPC code to HTTP status
    w.Write(data)
}

// NewHTTPServer with custom error encoder
func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.ErrorEncoder(errorEncoder),
        http.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
        ),
    }
    
    srv := http.NewServer(opts...)
    v1.RegisterUserHTTPServer(srv, user)
    
    return srv
}
```

### Error Code Mapping

| HTTP Status | gRPC Code | Kratos Error |
|-------------|-----------|--------------|
| 400 | 3 (InvalidArgument) | BadRequest |
| 401 | 16 (Unauthenticated) | Unauthorized |
| 403 | 7 (PermissionDenied) | Forbidden |
| 404 | 5 (NotFound) | NotFound |
| 409 | 6 (AlreadyExists) | Conflict |
| 429 | 8 (ResourceExhausted) | TooManyRequests |
| 500 | 13 (Internal) | InternalServer |
| 503 | 14 (Unavailable) | ServiceUnavailable |

---

## File Handling

### File Upload

```go
// internal/server/http.go
func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    srv := http.NewServer(opts...)
    
    router := srv.Route("/")
    
    // File upload endpoint
    router.POST("/upload", func(ctx http.Context) error {
        req := ctx.Request()
        
        // Get form file
        file, handler, err := req.FormFile("file")
        if err != nil {
            return errors.BadRequest("UPLOAD_ERROR", err.Error())
        }
        defer file.Close()
        
        // Get additional form data
        name := req.FormValue("name")
        
        // Save file
        dst, err := os.Create(fmt.Sprintf("./uploads/%s", handler.Filename))
        if err != nil {
            return errors.InternalServer("SAVE_ERROR", err.Error())
        }
        defer dst.Close()
        
        if _, err := io.Copy(dst, file); err != nil {
            return errors.InternalServer("COPY_ERROR", err.Error())
        }
        
        return ctx.String(200, fmt.Sprintf("File %s uploaded successfully", name))
    })
    
    return srv
}
```

### File Download

```go
// internal/server/http.go
router.GET("/download/{filename}", func(ctx http.Context) error {
    filename := ctx.Vars().Get("filename")
    filepath := fmt.Sprintf("./files/%s", filename)
    
    // Check file exists
    if _, err := os.Stat(filepath); os.IsNotExist(err) {
        return errors.NotFound("FILE_NOT_FOUND", "file not found")
    }
    
    // Set headers for download
    ctx.Response().Header().Set("Content-Type", "application/octet-stream")
    ctx.Response().Header().Set(
        "Content-Disposition",
        fmt.Sprintf("attachment; filename=%s", filename),
    )
    ctx.Response().Header().Set("Access-Control-Expose-Headers", "Content-Disposition")
    
    // Serve file
    return ctx.File(filepath)
})
```

---

## CORS Configuration

```go
// internal/server/http.go
import "github.com/gorilla/handlers"

func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Filter(
            handlers.CORS(
                handlers.AllowedOrigins([]string{"*"}),
                handlers.AllowedMethods([]string{"GET", "POST", "PUT", "DELETE", "OPTIONS"}),
                handlers.AllowedHeaders([]string{"Content-Type", "Authorization"}),
                handlers.AllowCredentials(),
            ),
        ),
        http.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
        ),
    }
    
    srv := http.NewServer(opts...)
    v1.RegisterUserHTTPServer(srv, user)
    
    return srv
}
```

---

## Third-Party Router Integration

### Echo Integration

```go
// internal/server/http.go
import "github.com/labstack/echo/v4"

func NewHTTPServer(c *conf.Server, logger log.Logger) *http.Server {
    // Create Echo router
    e := echo.New()
    
    // Add Echo routes
    e.GET("/home", func(ctx echo.Context) error {
        return ctx.JSON(200, map[string]string{
            "message": "Hello from Echo",
        })
    })
    
    // Create kratos HTTP server with Echo
    srv := http.NewServer(http.Address(":8000"))
    srv.HandlePrefix("/", e)
    
    return srv
}
```

### Gorilla Mux Integration

```go
// internal/server/http.go
import "github.com/gorilla/mux"

func NewHTTPServer(c *conf.Server, logger log.Logger) *http.Server {
    // Create Mux router
    r := mux.NewRouter()
    
    // Add routes
    r.HandleFunc("/users/{id}", func(w stdhttp.ResponseWriter, r *stdhttp.Request) {
        vars := mux.Vars(r)
        id := vars["id"]
        // Handle request
    }).Methods("GET")
    
    // Create kratos HTTP server
    srv := http.NewServer(http.Address(":8000"))
    srv.HandlePrefix("/", r)
    
    return srv
}
```

---

## Testing HTTP Services

```go
// internal/service/userservice_test.go
package service

import (
    "context"
    "testing"
    
    "github.com/go-kratos/kratos/v2/log"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    v1 "user-service/api/user/v1"
    "user-service/internal/biz"
)

type MockUserUsecase struct {
    mock.Mock
}

func (m *MockUserUsecase) Create(ctx context.Context, u *biz.User) (*biz.User, error) {
    args := m.Called(ctx, u)
    return args.Get(0).(*biz.User), args.Error(1)
}

func (m *MockUserUsecase) Get(ctx context.Context, id int64) (*biz.User, error) {
    args := m.Called(ctx, id)
    return args.Get(0).(*biz.User), args.Error(1)
}

func TestUserService_CreateUser(t *testing.T) {
    mockUC := new(MockUserUsecase)
    svc := NewUserService(mockUC, log.DefaultLogger)
    
    mockUC.On("Create", mock.Anything, &biz.User{
        Name:  "John",
        Email: "john@example.com",
        Age:   25,
    }).Return(&biz.User{
        ID:    1,
        Name:  "John",
        Email: "john@example.com",
    }, nil)
    
    reply, err := svc.CreateUser(context.Background(), &v1.CreateUserRequest{
        Name:  "John",
        Email: "john@example.com",
        Age:   25,
    })
    
    assert.NoError(t, err)
    assert.Equal(t, int64(1), reply.Id)
    assert.Equal(t, "John", reply.Name)
}
```

---

## Summary

### ✅ Always Follow

- Keep transport logic in `service` layer
- Use proto-generated request/response DTOs
- Map domain errors to appropriate HTTP status codes
- Use middleware for cross-cutting concerns
- Pass `context.Context` through all layers
- Validate input using proto validation rules

### ❌ Never Do

- Put business logic in HTTP handlers
- Return internal errors directly to clients
- Skip input validation
- Use `context.Background()` in handlers
- Mix HTTP-specific logic with business logic
- Hard-code configuration values
