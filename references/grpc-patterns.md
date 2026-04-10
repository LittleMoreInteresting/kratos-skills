# gRPC Transport Patterns

This guide covers gRPC service development patterns for go-kratos.

## Core Architecture

gRPC in go-kratos follows the same four-layer architecture as HTTP:

```
gRPC Request → Transport → Service → Biz → Data
                   ↓
           Dependency Injection (Wire)
```

## gRPC Server Setup

### ✅ Correct Pattern

```go
// internal/server/grpc.go
package server

import (
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/go-kratos/kratos/v2/middleware/logging"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/validate"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    v1 "user-service/api/helloworld/v1"
    "user-service/internal/conf"
    "user-service/internal/service"
)

// NewGRPCServer creates gRPC server
func NewGRPCServer(c *conf.Server, greeter *service.GreeterService, logger log.Logger) *grpc.Server {
    var opts = []grpc.ServerOption{
        grpc.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
            validate.Validator(),
        ),
    }
    
    if c.Grpc.Network != "" {
        opts = append(opts, grpc.Network(c.Grpc.Network))
    }
    if c.Grpc.Addr != "" {
        opts = append(opts, grpc.Address(c.Grpc.Addr))
    }
    if c.Grpc.Timeout != nil {
        opts = append(opts, grpc.Timeout(c.Grpc.Timeout.AsDuration()))
    }
    
    srv := grpc.NewServer(opts...)
    v1.RegisterGreeterServer(srv, greeter)
    
    return srv
}
```

**Key Points:**
- Use same middleware pattern as HTTP
- Configure server with options
- Register gRPC handlers from generated code
- Share service implementation with HTTP

### ❌ Common Mistakes

```go
// DON'T: Separate service implementations for HTTP and gRPC
// ✅ HTTP and gRPC should share the same service implementation

// DON'T: Different middleware chains
// ✅ Use consistent middleware for both transports

// DON'T: Hard-code gRPC-specific configuration
srv := grpc.NewServer(
    grpc.Address(":9000"),  // ❌ Should come from config
)
```

## Protocol Buffers Definition

### ✅ Correct Pattern

```protobuf
// api/user/v1/user.proto
syntax = "proto3";

package api.user.v1;

option go_package = "user-service/api/user/v1;v1";

import "google/protobuf/timestamp.proto";
import "validate/validate.proto";
import "google/rpc/error_details.proto";

// User service definition
service User {
    // Create a new user
    rpc CreateUser (CreateUserRequest) returns (CreateUserReply) {
        option (google.api.http) = {
            post: "/v1/users"
            body: "*"
        };
    }
    
    // Get user by ID
    rpc GetUser (GetUserRequest) returns (GetUserReply) {
        option (google.api.http) = {
            get: "/v1/users/{id}"
        };
    }
    
    // Update user
    rpc UpdateUser (UpdateUserRequest) returns (UpdateUserReply) {
        option (google.api.http) = {
            put: "/v1/users/{id}"
            body: "*"
        };
    }
    
    // Delete user
    rpc DeleteUser (DeleteUserRequest) returns (DeleteUserReply) {
        option (google.api.http) = {
            delete: "/v1/users/{id}"
        };
    }
    
    // List users with pagination
    rpc ListUsers (ListUsersRequest) returns (ListUsersReply) {
        option (google.api.http) = {
            get: "/v1/users"
        };
    }
    
    // Stream users (server streaming)
    rpc StreamUsers (StreamUsersRequest) returns (stream User);
    
    // Bidirectional streaming
    rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}

// Request/Response messages
message CreateUserRequest {
    string name = 1 [(validate.rules).string = {min_len: 1, max_len: 50}];
    string email = 2 [(validate.rules).string = {email: true}];
    int32 age = 3 [(validate.rules).int32 = {gte: 18, lte: 120}];
}

message CreateUserReply {
    int64 id = 1;
    string name = 2;
    string email = 3;
    google.protobuf.Timestamp created_at = 4;
}

message GetUserRequest {
    int64 id = 1 [(validate.rules).int64 = {gt: 0}];
}

message GetUserReply {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    google.protobuf.Timestamp created_at = 5;
    google.protobuf.Timestamp updated_at = 6;
}

message UpdateUserRequest {
    int64 id = 1 [(validate.rules).int64 = {gt: 0}];
    string name = 2 [(validate.rules).string = {min_len: 1, max_len: 50}];
    int32 age = 3 [(validate.rules).int32 = {gte: 18, lte: 120}];
}

message UpdateUserReply {
    int64 id = 1;
    string name = 2;
    int32 age = 3;
    google.protobuf.Timestamp updated_at = 4;
}

message DeleteUserRequest {
    int64 id = 1 [(validate.rules).int64 = {gt: 0}];
}

message DeleteUserReply {
    bool success = 1;
}

message ListUsersRequest {
    int32 page = 1 [(validate.rules).int32 = {gte: 1}];
    int32 page_size = 2 [(validate.rules).int32 = {gte: 1, lte: 100}];
    string sort_by = 3;
    bool descending = 4;
}

message ListUsersReply {
    repeated User users = 1;
    int32 total = 2;
    int32 page = 3;
    int32 page_size = 4;
}

message StreamUsersRequest {
    int32 batch_size = 1 [(validate.rules).int32 = {gte: 1, lte: 1000}];
}

message ChatMessage {
    string user_id = 1;
    string content = 2;
    google.protobuf.Timestamp timestamp = 3;
}

// User entity
message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    google.protobuf.Timestamp created_at = 5;
    google.protobuf.Timestamp updated_at = 6;
}
```

**Key Points:**
- Use `validate.rules` for input validation
- Include HTTP annotations for gRPC-Gateway
- Use `google.protobuf.Timestamp` for timestamps
- Define clear request/response messages
- Use streaming for large datasets

### ❌ Common Mistakes

```protobuf
// DON'T: Skip validation rules
message CreateUserRequest {
    string name = 1;  // ❌ No validation
    string email = 2;  // ❌ No validation
}

// DON'T: Use string for timestamps
message User {
    string created_at = 1;  // ❌ Use google.protobuf.Timestamp
}

// DON'T: Mix concerns in messages
message UserRequest {
    // ❌ Don't mix create and update fields
    int64 id = 1;  // Only for update
    string name = 2;  // For both
    string password = 3;  // Only for create
}
```

## Service Implementation

### ✅ Correct Pattern

```go
// internal/service/userservice.go
package service

import (
    "context"
    "io"
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

// Unary RPC
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
        CreatedAt: timestamppb.New(user.CreatedAt),
    }, nil
}

// Server Streaming RPC
func (s *UserService) StreamUsers(req *v1.StreamUsersRequest, stream v1.User_StreamUsersServer) error {
    ctx := stream.Context()
    
    users, err := s.uc.ListAll(ctx)
    if err != nil {
        return err
    }
    
    batchSize := int(req.BatchSize)
    for i, user := range users {
        if err := stream.Send(&v1.User{
            Id:        user.ID,
            Name:      user.Name,
            Email:     user.Email,
            Age:       user.Age,
            CreatedAt: timestamppb.New(user.CreatedAt),
        }); err != nil {
            return err
        }
        
        // Send in batches
        if (i+1)%batchSize == 0 {
            s.log.Infof("Sent batch of %d users", batchSize)
        }
    }
    
    return nil
}

// Bidirectional Streaming RPC
func (s *UserService) Chat(stream v1.User_ChatServer) error {
    ctx := stream.Context()
    
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        
        s.log.Infof("Received message from %s: %s", msg.UserId, msg.Content)
        
        // Process message and send response
        response := &v1.ChatMessage{
            UserId:    "system",
            Content:   "Echo: " + msg.Content,
            Timestamp: timestamppb.Now(),
        }
        
        if err := stream.Send(response); err != nil {
            return err
        }
    }
}
```

### ❌ Common Mistakes

```go
// DON'T: Ignore context in streaming
func (s *UserService) StreamUsers(req *v1.StreamUsersRequest, stream v1.User_StreamUsersServer) error {
    // ❌ Not using stream.Context()
    users, err := s.uc.ListAll(context.Background())
    // ...
}

// DON'T: Not handling io.EOF in bidirectional streaming
func (s *UserService) Chat(stream v1.User_ChatServer) error {
    for {
        msg, err := stream.Recv()
        // ❌ Not checking for io.EOF
        if err != nil {
            return err  // Will error when client closes
        }
        // ...
    }
}
```

## gRPC Interceptors

### ✅ Correct Pattern

```go
// internal/server/grpc.go
func NewGRPCServer(c *conf.Server, user *service.UserService, logger log.Logger) *grpc.Server {
    var opts = []grpc.ServerOption{
        grpc.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
            validate.Validator(),
            authInterceptor(),
            metricsInterceptor(),
        ),
    }
    // ...
}

// Auth interceptor
func authInterceptor() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            // Extract metadata
            md, ok := metadata.FromIncomingContext(ctx)
            if !ok {
                return nil, errors.Unauthorized("MISSING_METADATA", "metadata required")
            }
            
            // Get authorization token
            tokens := md.Get("authorization")
            if len(tokens) == 0 {
                return nil, errors.Unauthorized("MISSING_TOKEN", "authorization token required")
            }
            
            // Validate token
            claims, err := validateToken(tokens[0])
            if err != nil {
                return nil, errors.Unauthorized("INVALID_TOKEN", "invalid token")
            }
            
            // Add claims to context
            ctx = context.WithValue(ctx, userClaimsKey{}, claims)
            
            return handler(ctx, req)
        }
    }
}

// Metrics interceptor
func metricsInterceptor() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            start := time.Now()
            
            reply, err := handler(ctx, req)
            
            duration := time.Since(start)
            method, _ := grpc.Method(ctx)
            
            // Record metrics
            metrics.RecordRPCDuration(method, duration)
            if err != nil {
                metrics.RecordRPCError(method)
            }
            
            return reply, err
        }
    }
}
```

## gRPC Client

### ✅ Correct Pattern

```go
// internal/data/greeterclient.go
package data

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/registry"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    v1 "user-service/api/helloworld/v1"
)

type GreeterClient struct {
    conn v1.GreeterClient
}

func NewGreeterClient(r registry.Discovery) (*GreeterClient, error) {
    conn, err := grpc.DialInsecure(
        context.Background(),
        grpc.WithEndpoint("discovery:///greeter-service"),
        grpc.WithDiscovery(r),
        grpc.WithMiddleware(
            recovery.Recovery(),
        ),
        grpc.WithTimeout(5*time.Second),
    )
    if err != nil {
        return nil, err
    }
    
    return &GreeterClient{
        conn: v1.NewGreeterClient(conn),
    }, nil
}

func (c *GreeterClient) SayHello(ctx context.Context, name string) (string, error) {
    reply, err := c.conn.SayHello(ctx, &v1.HelloRequest{Name: name})
    if err != nil {
        return "", err
    }
    return reply.Message, nil
}
```

### Service Discovery

```go
// With etcd discovery
import "github.com/go-kratos/kratos/contrib/registry/etcd/v2"

func NewGreeterClient(r *etcd.Registry) (*GreeterClient, error) {
    conn, err := grpc.DialInsecure(
        context.Background(),
        grpc.WithEndpoint("discovery:///greeter-service"),
        grpc.WithDiscovery(r),
    )
    // ...
}

// With consul discovery
import "github.com/go-kratos/kratos/contrib/registry/consul/v2"

func NewGreeterClient(r *consul.Registry) (*GreeterClient, error) {
    conn, err := grpc.DialInsecure(
        context.Background(),
        grpc.WithEndpoint("discovery:///greeter-service"),
        grpc.WithDiscovery(r),
    )
    // ...
}
```

## Error Handling in gRPC

### ✅ Correct Pattern

```go
import (
    "github.com/go-kratos/kratos/v2/errors"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func (s *UserService) GetUser(ctx context.Context, req *v1.GetUserRequest) (*v1.GetUserReply, error) {
    user, err := s.uc.Get(ctx, req.Id)
    if err != nil {
        // Convert to gRPC status
        if errors.IsNotFound(err) {
            return nil, status.Errorf(codes.NotFound, "user not found: %v", req.Id)
        }
        if errors.IsUnauthorized(err) {
            return nil, status.Errorf(codes.Unauthenticated, "unauthorized")
        }
        return nil, status.Errorf(codes.Internal, "internal error: %v", err)
    }
    
    return &v1.GetUserReply{
        Id:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}
```

### Error Code Mapping

| Kratos Error | gRPC Code |
|--------------|-----------|
| BadRequest | InvalidArgument |
| Unauthorized | Unauthenticated |
| Forbidden | PermissionDenied |
| NotFound | NotFound |
| Conflict | AlreadyExists |
| InternalServer | Internal |
| ServiceUnavailable | Unavailable |
| GatewayTimeout | DeadlineExceeded |

## Complete gRPC Workflow

### Step 1: Define Proto

```protobuf
// api/order/v1/order.proto
syntax = "proto3";

package api.order.v1;

option go_package = "order-service/api/order/v1;v1";

import "google/protobuf/timestamp.proto";
import "validate/validate.proto";

service Order {
    rpc CreateOrder (CreateOrderRequest) returns (CreateOrderReply);
    rpc GetOrder (GetOrderRequest) returns (GetOrderReply);
    rpc UpdateOrderStatus (UpdateOrderStatusRequest) returns (UpdateOrderStatusReply);
}

message CreateOrderRequest {
    int64 user_id = 1 [(validate.rules).int64 = {gt: 0}];
    repeated OrderItem items = 2 [(validate.rules).repeated = {min_items: 1}];
}

message OrderItem {
    int64 product_id = 1 [(validate.rules).int64 = {gt: 0}];
    int32 quantity = 2 [(validate.rules).int32 = {gt: 0}];
}

message CreateOrderReply {
    int64 order_id = 1;
    string status = 2;
    google.protobuf.Timestamp created_at = 3;
}

message GetOrderRequest {
    int64 order_id = 1 [(validate.rules).int64 = {gt: 0}];
}

message GetOrderReply {
    int64 order_id = 1;
    int64 user_id = 2;
    repeated OrderItem items = 3;
    string status = 4;
    google.protobuf.Timestamp created_at = 5;
}

message UpdateOrderStatusRequest {
    int64 order_id = 1 [(validate.rules).int64 = {gt: 0}];
    string status = 2 [(validate.rules).string = {in: ["pending", "confirmed", "shipped", "delivered", "cancelled"]}];
}

message UpdateOrderStatusReply {
    bool success = 1;
    google.protobuf.Timestamp updated_at = 2;
}
```

### Step 2: Generate Code

```bash
# Generate gRPC client and server code
kratos proto client api/order/v1/order.proto
kratos proto server api/order/v1/order.proto
```

### Step 3: Implement Service

```go
// internal/service/orderservice.go
package service

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/log"
    "google.golang.org/protobuf/types/known/timestamppb"
    v1 "order-service/api/order/v1"
    "order-service/internal/biz"
)

type OrderService struct {
    v1.UnimplementedOrderServer
    uc  *biz.OrderUsecase
    log *log.Helper
}

func NewOrderService(uc *biz.OrderUsecase, logger log.Logger) *OrderService {
    return &OrderService{
        uc:  uc,
        log: log.NewHelper(logger),
    }
}

func (s *OrderService) CreateOrder(ctx context.Context, req *v1.CreateOrderRequest) (*v1.CreateOrderReply, error) {
    items := make([]*biz.OrderItem, len(req.Items))
    for i, item := range req.Items {
        items[i] = &biz.OrderItem{
            ProductID: item.ProductId,
            Quantity:  item.Quantity,
        }
    }
    
    order, err := s.uc.Create(ctx, &biz.Order{
        UserID: req.UserId,
        Items:  items,
    })
    if err != nil {
        return nil, err
    }
    
    return &v1.CreateOrderReply{
        OrderId:   order.ID,
        Status:    order.Status,
        CreatedAt: timestamppb.New(order.CreatedAt),
    }, nil
}

// ... implement other methods
```

### Step 4: Configure gRPC Server

```go
// internal/server/grpc.go
func NewGRPCServer(c *conf.Server, order *service.OrderService, logger log.Logger) *grpc.Server {
    var opts = []grpc.ServerOption{
        grpc.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
            validate.Validator(),
        ),
    }
    
    if c.Grpc.Addr != "" {
        opts = append(opts, grpc.Address(c.Grpc.Addr))
    }
    
    srv := grpc.NewServer(opts...)
    v1.RegisterOrderServer(srv, order)
    
    return srv
}
```

## Testing gRPC Services

```go
// internal/service/orderservice_test.go
package service

import (
    "context"
    "testing"
    
    "github.com/go-kratos/kratos/v2/log"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    v1 "order-service/api/order/v1"
    "order-service/internal/biz"
)

type MockOrderUsecase struct {
    mock.Mock
}

func (m *MockOrderUsecase) Create(ctx context.Context, o *biz.Order) (*biz.Order, error) {
    args := m.Called(ctx, o)
    return args.Get(0).(*biz.Order), args.Error(1)
}

func TestOrderService_CreateOrder(t *testing.T) {
    mockUC := new(MockOrderUsecase)
    svc := NewOrderService(mockUC, log.DefaultLogger)
    
    mockUC.On("Create", mock.Anything, &biz.Order{
        UserID: 1,
        Items: []*biz.OrderItem{
            {ProductID: 1, Quantity: 2},
        },
    }).Return(&biz.Order{
        ID:     1,
        UserID: 1,
        Status: "pending",
    }, nil)
    
    reply, err := svc.CreateOrder(context.Background(), &v1.CreateOrderRequest{
        UserId: 1,
        Items: []*v1.OrderItem{
            {ProductId: 1, Quantity: 2},
        },
    })
    
    assert.NoError(t, err)
    assert.Equal(t, int64(1), reply.OrderId)
    assert.Equal(t, "pending", reply.Status)
}
```

## Summary

### ✅ Always Follow

- Define clear proto messages with validation
- Use streaming for large datasets
- Implement proper error handling with gRPC codes
- Use interceptors for cross-cutting concerns
- Share service implementation between HTTP and gRPC
- Use service discovery for client connections

### ❌ Never Do

- Skip proto validation rules
- Use different service implementations for HTTP/gRPC
- Ignore context in streaming RPCs
- Forget to handle io.EOF in bidirectional streaming
- Hard-code service endpoints
- Skip interceptor chain
