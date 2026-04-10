# GitHub Copilot Integration Guide

GitHub Copilot provides inline suggestions and chat-based assistance for working with go-kratos.

## Installation

### Project Setup

1. Clone kratos-skills to your project:

```bash
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills
```

2. Create `.github/copilot-instructions.md`:

```bash
mkdir -p .github
cat > .github/copilot-instructions.md << 'EOF'
# Kratos Skills for GitHub Copilot

When working with go-kratos projects:

## Project Structure
```
.
├── api/                  # Protocol buffer definitions
├── cmd/                  # Application entry points
├── internal/
│   ├── biz/             # Business logic layer
│   ├── data/            # Data access layer
│   ├── server/          # HTTP/gRPC servers
│   └── service/         # Service implementations
├── configs/             # Configuration files
└── go.mod
```

## Clean Architecture
- Transport Layer (internal/server/): HTTP/gRPC concerns only
- Service Layer (internal/service/): Request handling, validation
- Biz Layer (internal/biz/): Business logic, use cases
- Data Layer (internal/data/): Database operations, repositories

## Key Patterns

### Service Implementation
```go
// internal/service/userservice.go
type UserService struct {
    pb.UnimplementedUserServer
    uc *biz.UserUsecase
    log *log.Helper
}

func NewUserService(uc *biz.UserUsecase, logger log.Logger) *UserService {
    return &UserService{
        uc: uc,
        log: log.NewHelper(logger),
    }
}

func (s *UserService) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserReply, error) {
    // Validate input
    if req.Name == "" {
        return nil, errors.BadRequest("NAME_EMPTY", "name is required")
    }
    
    // Call biz layer
    user, err := s.uc.Create(ctx, &biz.User{
        Name:  req.Name,
        Email: req.Email,
    })
    if err != nil {
        return nil, err
    }
    
    return &pb.CreateUserReply{
        Id:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}
```

### Biz Layer
```go
// internal/biz/user.go
type User struct {
    ID    int64
    Name  string
    Email string
}

type UserRepo interface {
    Create(ctx context.Context, u *User) error
    Get(ctx context.Context, id int64) (*User, error)
    Update(ctx context.Context, u *User) error
    Delete(ctx context.Context, id int64) error
}

type UserUsecase struct {
    repo UserRepo
    log  *log.Helper
}

func NewUserUsecase(repo UserRepo, logger log.Logger) *UserUsecase {
    return &UserUsecase{
        repo: repo,
        log:  log.NewHelper(logger),
    }
}

func (uc *UserUsecase) Create(ctx context.Context, u *User) (*User, error) {
    // Business logic here
    if err := uc.repo.Create(ctx, u); err != nil {
        return nil, err
    }
    return u, nil
}
```

### Data Layer
```go
// internal/data/user.go
type userRepo struct {
    data *Data
    log  *log.Helper
}

func NewUserRepo(data *Data, logger log.Logger) biz.UserRepo {
    return &userRepo{
        data: data,
        log:  log.NewHelper(logger),
    }
}

func (r *userRepo) Create(ctx context.Context, u *biz.User) error {
    return r.data.db.User.Create().
        SetName(u.Name).
        SetEmail(u.Email).
        Exec(ctx)
}
```

## Commands
- Create project: `kratos new project-name`
- Generate proto: `kratos proto client api/*.proto`
- Generate service: `kratos proto server api/*.proto`
- Run wire: `cd internal && wire`

## Error Handling
- Use kratos errors package
- Always wrap errors with context
- Return appropriate HTTP/gRPC status codes

## Middleware
- Use for auth, logging, metrics, tracing
- Keep middleware chain minimal
- Order matters: auth → logging → recovery

## Resources
- Full patterns: .ai-context/kratos-skills/references/
- Best practices: .ai-context/kratos-skills/best-practices/
- Troubleshooting: .ai-context/kratos-skills/troubleshooting/
EOF
```

### VS Code Settings

Add to your VS Code settings (`.vscode/settings.json`):

```json
{
  "github.copilot.chat.codeGeneration.instructions": [
    {
      "file": ".github/copilot-instructions.md"
    }
  ]
}
```

## Usage

### Inline Suggestions

As you type, Copilot provides kratos-appropriate suggestions:

```go
// You type:
func (s *UserService) GetUser(

// Copilot suggests:
func (s *UserService) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserReply, error) {
    user, err := s.uc.Get(ctx, req.Id)
    if err != nil {
        return nil, err
    }
    return &pb.GetUserReply{
        Id:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}
```

### Chat Interface

Use Copilot Chat (Cmd/Ctrl + Shift + I) for complex queries:

```
You: Create a complete user service with CRUD operations

Copilot: I'll create a complete user service following kratos patterns...

1. Creating proto definition
2. Generating code
3. Implementing service layer
4. Implementing biz layer
5. Implementing data layer
6. Wiring dependencies

[Generated code for all layers]
```

### Code Explanation

```
You: Explain this kratos service code

Copilot: This is the service layer in go-kratos clean architecture:

1. UserService struct embeds pb.UnimplementedUserServer for gRPC
2. It depends on UserUsecase from biz layer
3. CreateUser method:
   - Validates input
   - Calls biz layer for business logic
   - Returns protobuf response

The service layer handles transport concerns and delegates business logic to the biz layer.
```

## Common Workflows

### Creating a New Service

```
You: Create an order service with CreateOrder and GetOrder methods

Copilot: Creating order service...

1. First, let's create the proto definition:

```protobuf
syntax = "proto3";

package api.order;

option go_package = "order-service/api;api";

service Order {
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderReply);
  rpc GetOrder (GetOrderRequest) returns (GetOrderReply);
}

message CreateOrderRequest {
  int64 user_id = 1;
  repeated OrderItem items = 2;
}

message OrderItem {
  int64 product_id = 1;
  int32 quantity = 2;
}

message CreateOrderReply {
  int64 order_id = 1;
  string status = 2;
}

message GetOrderRequest {
  int64 order_id = 1;
}

message GetOrderReply {
  int64 order_id = 1;
  int64 user_id = 2;
  repeated OrderItem items = 3;
  string status = 4;
}
```

2. Generate code: kratos proto server api/order.proto
3. [Continues with service implementation...]
```

### Adding Middleware

```
You: Add authentication middleware to my HTTP server

Copilot: Creating auth middleware...

```go
// internal/server/http.go
func NewHTTPServer(c *conf.Server, greeter *service.GreeterService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
            validate.Validator(),
            authMiddleware(),  // Add auth middleware
        ),
    }
    // ...
}

func authMiddleware() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            // Extract token from context
            // Validate token
            // Add user info to context
            return handler(ctx, req)
        }
    }
}
```
```

### Database Operations

```
You: Implement a user repository with GORM

Copilot: Implementing user repository...

```go
// internal/data/user.go
type userRepo struct {
    data *Data
    log  *log.Helper
}

func NewUserRepo(data *Data, logger log.Logger) biz.UserRepo {
    return &userRepo{
        data: data,
        log:  log.NewHelper(logger),
    }
}

func (r *userRepo) Create(ctx context.Context, u *biz.User) error {
    user := &User{
        Name:  u.Name,
        Email: u.Email,
    }
    result := r.data.db.WithContext(ctx).Create(user)
    if result.Error != nil {
        return result.Error
    }
    u.ID = user.ID
    return nil
}

func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    var user User
    result := r.data.db.WithContext(ctx).First(&user, id)
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrRecordNotFound) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        return nil, result.Error
    }
    return &biz.User{
        ID:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}
```
```

## Best Practices

### 1. Clear Prompts

```
❌ "Fix this"
✅ "Add input validation to CreateUser following kratos patterns"
```

### 2. Provide Context

```
You: I'm building an e-commerce service. Create an order repository
that handles order creation and retrieval with proper error handling.
```

### 3. Reference Files

```
You: Follow the pattern in internal/service/userservice.go to create
a similar product service
```

## Troubleshooting

### Instructions Not Applied

```bash
# Verify file exists
cat .github/copilot-instructions.md

# Check VS Code settings
cat .vscode/settings.json

# Reload VS Code window
```

### Irrelevant Suggestions

1. Ensure `.ai-context/kratos-skills/` exists
2. Check that go.mod contains kratos
3. Verify proto files are present

### Chat Not Working

1. Check Copilot subscription
2. Verify Copilot extension is installed
3. Check GitHub authentication

## Tips and Tricks

### 1. Quick Pattern Reference

```
You: Show me the kratos middleware pattern

Copilot: Here's the middleware pattern from kratos-skills:

```go
func MyMiddleware() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            // Pre-processing
            // ...
            
            // Call handler
            reply, err := handler(ctx, req)
            
            // Post-processing
            // ...
            
            return reply, err
        }
    }
}
```
```

### 2. Generate Multiple Files

```
You: Generate all files needed for a complete product service

Copilot: I'll generate:
1. api/product.proto
2. internal/service/productservice.go
3. internal/biz/product.go
4. internal/data/product.go
5. internal/data/ent/schema/product.go
```

### 3. Explain Errors

```
You: What does this error mean? "wire: go-kratos/kratos/v2: no Go files"

Copilot: This error occurs when Wire can't find the kratos package.
Solutions:
1. Run `go mod tidy` to download dependencies
2. Ensure go.mod includes kratos
3. Check that you're running wire from internal/ directory
```

## Next Steps

1. Try the [examples](../examples/) to see kratos-skills in action
2. Explore the [pattern guides](../references/) for detailed patterns
3. Review [best practices](../best-practices/) for production code
4. Check [troubleshooting](../troubleshooting/) for common issues

## Resources

- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
- [go-kratos Documentation](https://go-kratos.dev/)
