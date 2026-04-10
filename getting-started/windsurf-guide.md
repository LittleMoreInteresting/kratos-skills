# Windsurf IDE Integration Guide

Windsurf provides Cascade AI workflows for iterative development with go-kratos.

## Installation

### Project Setup

1. Clone kratos-skills to your project:

```bash
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills
```

2. Create `.windsurfrules` file in your project root:

```bash
cat > .windsurfrules << 'EOF'
# Kratos Skills for Windsurf

When working with go-kratos projects:

## Architecture (Clean Architecture)
```
Transport (internal/server/)
    ↓
Service (internal/service/)
    ↓
Biz (internal/biz/)
    ↓
Data (internal/data/)
```

## Layer Responsibilities

### Transport Layer
- HTTP/gRPC server setup
- Middleware registration
- Route configuration
- NO business logic

### Service Layer
- Request/response handling
- Input validation
- Protocol buffer conversion
- Calls Biz layer

### Biz Layer
- Business logic
- Use cases
- Domain models
- Repository interfaces

### Data Layer
- Database operations
- Repository implementations
- External service calls
- Cache operations

## Code Generation Commands
- `kratos new project-name` - Create new project
- `kratos proto client api/*.proto` - Generate proto client
- `kratos proto server api/*.proto` - Generate service code
- `cd internal && wire` - Generate wire dependencies

## Key Patterns

### Service Definition
```go
type UserService struct {
    pb.UnimplementedUserServer
    uc *biz.UserUsecase
}

func NewUserService(uc *biz.UserUsecase) *UserService {
    return &UserService{uc: uc}
}
```

### Biz Usecase
```go
type UserUsecase struct {
    repo UserRepo
}

type UserRepo interface {
    Create(ctx context.Context, u *User) error
    Get(ctx context.Context, id int64) (*User, error)
}
```

### Repository Implementation
```go
type userRepo struct {
    data *Data
}

func NewUserRepo(data *Data) biz.UserRepo {
    return &userRepo{data: data}
}
```

## Error Handling
- Use kratos errors package
- Wrap errors with context
- Return domain errors from Biz layer

## Middleware
- Auth, logging, metrics, tracing
- Order: recovery → logging → auth → validator

## Resources
- Patterns: .ai-context/kratos-skills/references/
- Best practices: .ai-context/kratos-skills/best-practices/
- Troubleshooting: .ai-context/kratos-skills/troubleshooting/
EOF
```

## Usage

### Cascade AI

Windsurf's Cascade AI enables iterative workflows:

```
You: Create a user management service

Cascade: I'll create a user management service step by step...

Step 1: Creating project structure
```bash
kratos new user-service
cd user-service
```

Step 2: Defining proto schema
```protobuf
service User {
  rpc CreateUser (CreateUserRequest) returns (CreateUserReply);
  rpc GetUser (GetUserRequest) returns (GetUserReply);
  rpc UpdateUser (UpdateUserRequest) returns (UpdateUserReply);
  rpc DeleteUser (DeleteUserRequest) returns (DeleteUserReply);
}
```

Step 3: Generating code
```bash
kratos proto server api/user.proto
```

Continue to next step? (Yes/No/Modify)
```

### Inline Assistance

As you code, Windsurf provides context-aware suggestions:

```go
// You type:
func (uc *UserUsecase) Create(

// Windsurf suggests:
func (uc *UserUsecase) Create(ctx context.Context, u *User) (*User, error) {
    // Validate user
    if u.Name == "" {
        return nil, errors.BadRequest("NAME_EMPTY", "name is required")
    }
    
    // Call repository
    if err := uc.repo.Create(ctx, u); err != nil {
        return nil, err
    }
    
    return u, nil
}
```

### Code Review

```
You: Review my service implementation

Cascade: Reviewing against kratos patterns...

✅ Proper layer separation
✅ Context propagation
⚠️ Missing input validation in UpdateUser
⚠️ No error wrapping in DeleteUser

Suggested fixes:
1. Add validation for UpdateUser
2. Wrap errors with proper context
3. Add logging

Apply fixes? (Yes/No/Show details)
```

## Common Workflows

### Project Creation

```
You: Create a new microservice for order management

Cascade: Creating order management microservice...

Step 1: Initialize project
```bash
kratos new order-service
cd order-service
```

Step 2: Define domain models
```go
// internal/biz/order.go
type Order struct {
    ID        int64
    UserID    int64
    Items     []*OrderItem
    Status    string
    CreatedAt time.Time
}

type OrderItem struct {
    ProductID int64
    Quantity  int32
    Price     float64
}
```

Step 3: Create proto definitions
```protobuf
service Order {
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderReply);
  rpc GetOrder (GetOrderRequest) returns (GetOrderReply);
  rpc ListOrders (ListOrdersRequest) returns (ListOrdersReply);
}
```

Continue? (Yes/No/Modify)
```

### Feature Addition

```
You: Add pagination to ListUsers

Cascade: Adding pagination to ListUsers...

Step 1: Update proto definition
```protobuf
message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
}

message ListUsersReply {
  repeated User users = 1;
  int32 total = 2;
}
```

Step 2: Update repository interface
```go
type UserRepo interface {
    List(ctx context.Context, page, pageSize int32) ([]*User, int32, error)
}
```

Step 3: Implement pagination
```go
func (r *userRepo) List(ctx context.Context, page, pageSize int32) ([]*User, int32, error) {
    var users []*User
    var total int64
    
    offset := (page - 1) * pageSize
    
    result := r.data.db.WithContext(ctx).Model(&User{}).Count(&total)
    if result.Error != nil {
        return nil, 0, result.Error
    }
    
    result = r.data.db.WithContext(ctx).Offset(int(offset)).Limit(int(pageSize)).Find(&users)
    if result.Error != nil {
        return nil, 0, result.Error
    }
    
    return users, int32(total), nil
}
```

Apply changes? (Yes/No/Review)
```

### Refactoring

```
You: Refactor my service to use dependency injection properly

Cascade: Refactoring for proper DI...

Step 1: Analyzing current structure...
Step 2: Identifying dependencies...
Step 3: Creating provider functions...
Step 4: Updating wire.go...
Step 5: Running wire generate...

✅ Refactoring complete!

Changes made:
- Extracted dependencies into provider functions
- Added wire tags for injection
- Generated wire_gen.go
- Updated service constructors
```

## Best Practices

### 1. Iterative Development

```
You: Create a product service

Cascade: I'll create it iteratively. Let's start with the proto definition.

[Creates proto]

Review and continue? (Yes/Modify/Start over)

You: Yes, continue

Cascade: Now generating code and implementing layers...

[Implements service, biz, data layers]

Review and continue? (Yes/Modify/Start over)
```

### 2. Clear Instructions

```
❌ "Make it better"
✅ "Add caching to the GetUser method using Redis"
```

### 3. Context Awareness

```
You: Following the pattern in userservice.go, create a similar method
for products

Cascade: Analyzing userservice.go pattern...
        Creating similar method for products...
```

## Troubleshooting

### Rules Not Applied

```bash
# Check .windsurfrules exists
cat .windsurfrules

# Verify in project root
ls -la .windsurfrules

# Restart Windsurf
```

### Cascade Not Responding

1. Check internet connection
2. Verify Windsurf subscription
3. Restart Windsurf

### Code Generation Fails

```bash
# Verify kratos CLI
which kratos
kratos --version

# Check proto files
ls api/*.proto

# Verify dependencies
go mod tidy
```

## Tips and Tricks

### 1. Step-by-Step Workflows

```
You: Create a complete authentication system

Cascade: I'll create it step by step:

1. Proto definitions
2. Service layer
3. Biz layer (JWT logic)
4. Data layer (token storage)
5. Middleware
6. Wire configuration

Starting with step 1...
```

### 2. Pattern Application

```
You: Apply repository pattern to all my entities

Cascade: Applying repository pattern...

- User: ✅ Created repository
- Product: ✅ Created repository
- Order: ✅ Created repository

All repositories follow the pattern from kratos-skills.
```

### 3. Error Analysis

```
You: Why am I getting "dependency not provided" error from Wire?

Cascade: Analyzing wire.go...

Issue: UserRepo interface not bound to implementation.

Fix: Add provider function:
```go
func NewUserRepo(data *Data) biz.UserRepo {
    return NewUserRepo(data)
}
```
```

## Next Steps

1. Try the [examples](../examples/) to see kratos-skills in action
2. Explore the [pattern guides](../references/) for detailed patterns
3. Review [best practices](../best-practices/) for production code
4. Check [troubleshooting](../troubleshooting/) for common issues

## Resources

- [Windsurf Documentation](https://docs.codeium.com/windsurf)
- [go-kratos Documentation](https://go-kratos.dev/)
