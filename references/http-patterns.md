# HTTP Transport Patterns

This guide covers HTTP API development patterns for go-kratos.

## Core Architecture

### Four-Layer Pattern

go-kratos follows clean architecture with four distinct layers:

```
HTTP Request → Transport → Service → Biz → Data
                    ↓
            Dependency Injection (Wire)
```

1. **Transport Layer** (`internal/server/`) - HTTP/gRPC server setup
2. **Service Layer** (`internal/service/`) - Request handling, validation
3. **Biz Layer** (`internal/biz/`) - Business logic
4. **Data Layer** (`internal/data/`) - Data access

## HTTP Server Setup

### ✅ Correct Pattern

```go
// internal/server/http.go
package server

import (
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/go-kratos/kratos/v2/middleware/logging"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/validate"
    "github.com/go-kratos/kratos/v2/transport/http"
    v1 "user-service/api/helloworld/v1"
    "user-service/internal/conf"
    "user-service/internal/service"
)

// NewHTTPServer creates HTTP server
func NewHTTPServer(c *conf.Server, greeter *service.GreeterService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
            validate.Validator(),
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
    v1.RegisterGreeterHTTPServer(srv, greeter)
    
    return srv
}
```

**Key Points:**
- Use middleware chain for cross-cutting concerns
- Configure server with options pattern
- Register HTTP handlers from generated code
- Inject dependencies via constructor

### ❌ Common Mistakes

```go
// DON'T: Hard-code configuration
srv := http.NewServer(
    http.Address(":8000"),  // ❌ Hard-coded port
)

// DON'T: Skip middleware
srv := http.NewServer()  // ❌ No recovery, logging, validation

// DON'T: Create server without dependencies
func NewHTTPServer() *http.Server {  // ❌ Missing service dependencies
    srv := http.NewServer()
    // Can't register handlers without services
    return srv
}
```

## Service Implementation

### ✅ Correct Pattern

```go
// internal/service/greeterservice.go
package service

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/log"
    v1 "user-service/api/helloworld/v1"
    "user-service/internal/biz"
)

// GreeterService is a greeter service.
type GreeterService struct {
    v1.UnimplementedGreeterServer
    
    uc  *biz.GreeterUsecase
    log *log.Helper
}

// NewGreeterService creates a new GreeterService.
func NewGreeterService(uc *biz.GreeterUsecase, logger log.Logger) *GreeterService {
    return &GreeterService{
        uc:  uc,
        log: log.NewHelper(logger),
    }
}

// SayHello implements helloworld.GreeterServer.
func (s *GreeterService) SayHello(ctx context.Context, req *v1.HelloRequest) (*v1.HelloReply, error) {
    // 1. Validate input
    if req.Name == "" {
        return nil, v1.ErrorBadRequest("NAME_EMPTY", "name is required")
    }
    
    // 2. Call biz layer
    g, err := s.uc.CreateGreeter(ctx, &biz.Greeter{
        Hello: req.Name,
    })
    if err != nil {
        return nil, err
    }
    
    // 3. Return response
    return &v1.HelloReply{
        Message: "Hello " + g.Hello,
    }, nil
}
```

**Key Points:**
- Embed generated `UnimplementedXxxServer`
- Inject biz usecase via constructor
- Validate input in service layer
- Call biz layer for business logic
- Return protobuf responses

### ❌ Common Mistakes

```go
// DON'T: Business logic in service
func (s *GreeterService) SayHello(ctx context.Context, req *v1.HelloRequest) (*v1.HelloReply, error) {
    // ❌ Direct database access
    user, err := s.db.Query("SELECT * FROM users WHERE name = ?", req.Name)
    
    // ❌ Business logic here
    if user.Status == "inactive" {
        return nil, errors.New("user inactive")
    }
    
    return &v1.HelloReply{Message: "Hello " + user.Name}, nil
}

// DON'T: Skip validation
func (s *GreeterService) SayHello(ctx context.Context, req *v1.HelloRequest) (*v1.HelloReply, error) {
    // ❌ No input validation
    g, err := s.uc.CreateGreeter(ctx, &biz.Greeter{Hello: req.Name})
    return &v1.HelloReply{Message: "Hello " + g.Hello}, err
}

// DON'T: Return non-protobuf types
func (s *GreeterService) SayHello(ctx context.Context, req *v1.HelloRequest) (string, error) {
    // ❌ Returns string instead of *v1.HelloReply
    return "Hello " + req.Name, nil
}
```

## Request/Response Handling

### ✅ Correct Pattern

```go
// Proto definition
message CreateUserRequest {
    string name = 1 [(validate.rules).string = {min_len: 1, max_len: 50}];
    string email = 2 [(validate.rules).string = {email: true}];
    int32 age = 3 [(validate.rules).int32 = {gte: 18, lte: 120}];
}

message CreateUserReply {
    int64 id = 1;
    string name = 2;
    string email = 3;
    string created_at = 4;
}

// Service implementation
func (s *UserService) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserReply, error) {
    // Validation is handled by validate.Validator() middleware
    // based on proto validation rules
    
    // Convert proto to biz model
    user := &biz.User{
        Name:  req.Name,
        Email: req.Email,
        Age:   req.Age,
    }
    
    // Call biz layer
    created, err := s.uc.Create(ctx, user)
    if err != nil {
        return nil, err
    }
    
    // Convert biz model to proto
    return &pb.CreateUserReply{
        Id:        created.ID,
        Name:      created.Name,
        Email:     created.Email,
        CreatedAt: created.CreatedAt.Format(time.RFC3339),
    }, nil
}
```

### ❌ Common Mistakes

```go
// DON'T: Manual validation when proto validation exists
func (s *UserService) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserReply, error) {
    // ❌ Redundant validation (proto validation already handles this)
    if req.Name == "" {
        return nil, errors.New("name is required")
    }
    if len(req.Name) > 50 {
        return nil, errors.New("name too long")
    }
    // ...
}

// DON'T: Leak internal models
func (s *UserService) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*biz.User, error) {
    // ❌ Returns internal biz model instead of proto
    return s.uc.Create(ctx, user)
}
```

## Error Handling

### ✅ Correct Pattern

```go
import "github.com/go-kratos/kratos/v2/errors"

func (s *UserService) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserReply, error) {
    user, err := s.uc.Get(ctx, req.Id)
    if err != nil {
        // Check if it's a not found error
        if errors.IsNotFound(err) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        // Wrap internal errors
        return nil, errors.InternalServer("DATABASE_ERROR", "failed to get user").WithCause(err)
    }
    
    return &pb.GetUserReply{
        Id:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}
```

### Error Codes Reference

```go
// 4xx Client Errors
errors.BadRequest("INVALID_PARAM", "invalid parameter")
errors.Unauthorized("UNAUTHORIZED", "unauthorized")
errors.Forbidden("FORBIDDEN", "forbidden")
errors.NotFound("RESOURCE_NOT_FOUND", "resource not found")
errors.Conflict("RESOURCE_CONFLICT", "resource conflict")

// 5xx Server Errors
errors.InternalServer("INTERNAL_ERROR", "internal server error")
errors.ServiceUnavailable("SERVICE_UNAVAILABLE", "service unavailable")
errors.GatewayTimeout("GATEWAY_TIMEOUT", "gateway timeout")
```

## Middleware

### ✅ Correct Pattern

```go
// internal/server/http.go
func NewHTTPServer(c *conf.Server, greeter *service.GreeterService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            // Order matters: recovery first, then logging, then auth, then validation
            recovery.Recovery(),           // 1. Recover from panics
            logging.Server(logger),        // 2. Log requests
            authMiddleware(),              // 3. Authenticate
            validate.Validator(),          // 4. Validate requests
        ),
    }
    // ...
}

// Custom auth middleware
func authMiddleware() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            // Extract token from HTTP header
            if header, ok := transport.FromServerContext(ctx); ok {
                token := header.RequestHeader().Get("Authorization")
                if token == "" {
                    return nil, errors.Unauthorized("MISSING_TOKEN", "authorization token required")
                }
                
                // Validate token
                claims, err := validateToken(token)
                if err != nil {
                    return nil, errors.Unauthorized("INVALID_TOKEN", "invalid token")
                }
                
                // Add user info to context
                ctx = context.WithValue(ctx, userIDKey{}, claims.UserID)
            }
            
            return handler(ctx, req)
        }
    }
}
```

### ❌ Common Mistakes

```go
// DON'T: Wrong middleware order
http.Middleware(
    validate.Validator(),     // ❌ Validation before auth
    authMiddleware(),         // Should be: auth before validation
    recovery.Recovery(),      // ❌ Recovery should be first
)

// DON'T: Expensive operations in middleware
func slowMiddleware() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            // ❌ Don't do heavy work here
            time.Sleep(100 * time.Millisecond)
            
            return handler(ctx, req)
        }
    }
}
```

## Complete HTTP API Workflow

### Step 1: Define Proto

```protobuf
// api/user/v1/user.proto
syntax = "proto3";

package api.user.v1;

option go_package = "user-service/api/user/v1;v1";

import "validate/validate.proto";

service User {
    rpc CreateUser (CreateUserRequest) returns (CreateUserReply);
    rpc GetUser (GetUserRequest) returns (GetUserReply);
    rpc UpdateUser (UpdateUserRequest) returns (UpdateUserReply);
    rpc DeleteUser (DeleteUserRequest) returns (DeleteUserReply);
    rpc ListUsers (ListUsersRequest) returns (ListUsersReply);
}

message CreateUserRequest {
    string name = 1 [(validate.rules).string = {min_len: 1, max_len: 50}];
    string email = 2 [(validate.rules).string = {email: true}];
    int32 age = 3 [(validate.rules).int32 = {gte: 18, lte: 120}];
}

message CreateUserReply {
    int64 id = 1;
    string name = 2;
    string email = 3;
    string created_at = 4;
}

message GetUserRequest {
    int64 id = 1;
}

message GetUserReply {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    string created_at = 5;
    string updated_at = 6;
}

message UpdateUserRequest {
    int64 id = 1;
    string name = 2 [(validate.rules).string = {min_len: 1, max_len: 50}];
    int32 age = 3 [(validate.rules).int32 = {gte: 18, lte: 120}];
}

message UpdateUserReply {
    int64 id = 1;
    string name = 2;
    int32 age = 3;
    string updated_at = 4;
}

message DeleteUserRequest {
    int64 id = 1;
}

message DeleteUserReply {
    bool success = 1;
}

message ListUsersRequest {
    int32 page = 1 [(validate.rules).int32 = {gte: 1}];
    int32 page_size = 2 [(validate.rules).int32 = {gte: 1, lte: 100}];
}

message ListUsersReply {
    repeated User users = 1;
    int32 total = 2;
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
}
```

### Step 2: Generate Code

```bash
# Generate proto client and server code
kratos proto client api/user/v1/user.proto
kratos proto server api/user/v1/user.proto
```

### Step 3: Implement Service Layer

```go
// internal/service/userservice.go
package service

import (
    "context"
    "time"
    
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

func (s *UserService) CreateUser(ctx context.Context, req *v1.CreateUserRequest) (*v1.CreateUserReply, error) {
    user, err := s.uc.Create(ctx, &biz.User{
        Name:  req.Name,
        Email: req.Email,
        Age:   req.Age,
    })
    if err != nil {
        return nil, err
    }
    
    return &v1.CreateUserReply{
        Id:        user.ID,
        Name:      user.Name,
        Email:     user.Email,
        CreatedAt: user.CreatedAt.Format(time.RFC3339),
    }, nil
}

func (s *UserService) GetUser(ctx context.Context, req *v1.GetUserRequest) (*v1.GetUserReply, error) {
    user, err := s.uc.Get(ctx, req.Id)
    if err != nil {
        return nil, err
    }
    
    return &v1.GetUserReply{
        Id:        user.ID,
        Name:      user.Name,
        Email:     user.Email,
        Age:       user.Age,
        CreatedAt: user.CreatedAt.Format(time.RFC3339),
        UpdatedAt: user.UpdatedAt.Format(time.RFC3339),
    }, nil
}

// ... implement other methods
```

### Step 4: Implement Biz Layer

See [clean-architecture-patterns.md](clean-architecture-patterns.md) for biz layer implementation.

### Step 5: Implement Data Layer

See [database-patterns.md](database-patterns.md) for data layer implementation.

### Step 6: Configure HTTP Server

```go
// internal/server/http.go
func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
            validate.Validator(),
        ),
    }
    
    if c.Http.Addr != "" {
        opts = append(opts, http.Address(c.Http.Addr))
    }
    
    srv := http.NewServer(opts...)
    v1.RegisterUserHTTPServer(srv, user)
    
    return srv
}
```

### Step 7: Wire Dependencies

See [kratos-cli-commands.md](kratos-cli-commands.md) for Wire configuration.

## Testing HTTP Handlers

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
        Age:   25,
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

## Summary

### ✅ Always Follow

- Use middleware chain in correct order
- Validate input in service layer
- Call biz layer for business logic
- Return protobuf responses
- Use proper error codes
- Inject dependencies via constructor

### ❌ Never Do

- Put business logic in service layer
- Skip input validation
- Return non-protobuf types
- Hard-code configuration
- Skip middleware
- Access database directly from service
