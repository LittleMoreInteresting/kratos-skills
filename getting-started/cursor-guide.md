# Cursor IDE Integration Guide

Cursor provides an IDE-native experience for using kratos-skills with chat-based interactions and inline suggestions.

## Installation

### Project Setup

1. Clone kratos-skills to your project:

```bash
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills
```

2. Create `.cursorrules` file in your project root:

```bash
cat > .cursorrules << 'EOF'
# Kratos Skills Integration

When working with go-kratos projects:

## Architecture
- Follow clean architecture: Transport → Service → Biz → Data
- Use Wire for dependency injection
- Define interfaces in Biz layer, implement in Data layer
- Pass context.Context through all layers

## Code Generation
- Use kratos CLI for project scaffolding: `kratos new project-name`
- Generate proto code: `kratos proto client api/*.proto`
- Generate service code: `kratos proto server api/*.proto`
- Run wire after dependency changes: `cd internal && wire`

## Patterns
- Reference .ai-context/kratos-skills/references/ for detailed patterns
- Use middleware for cross-cutting concerns
- Implement proper error handling with kratos errors package
- Use structured logging with kratos log

## Best Practices
- Keep transport layer thin (HTTP/gRPC concerns only)
- Implement business logic in Biz layer
- Use repository pattern for data access
- Add unit tests for Biz layer
EOF
```

### Global Setup (Optional)

To use across all projects, add to Cursor's global rules:

1. Open Cursor Settings
2. Navigate to "Cursor Settings" > "Rules"
3. Add the kratos patterns above

## Usage

### Chat Interface

Use Cursor's chat (Cmd/Ctrl + L) to interact with kratos-skills:

```
You: Create a user service following kratos patterns

Cursor: I'll create a user service following go-kratos clean architecture...

1. Creating proto definition in api/user.proto
2. Generating code with kratos proto
3. Implementing service layer
4. Implementing biz layer
5. Implementing data layer
6. Wiring dependencies

✅ User service created!
```

### Inline Suggestions

As you type, Cursor provides kratos-appropriate suggestions:

```go
// You type:
func (s *UserService) CreateUser(

// Cursor suggests:
func (s *UserService) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserReply, error) {
    // Validate input
    if req.Name == "" {
        return nil, errors.BadRequest("NAME_EMPTY", "name is required")
    }
    
    // Call biz layer
    return s.uc.Create(ctx, req)
}
```

### Code Review

Ask Cursor to review your code:

```
You: Review this service implementation for kratos best practices

Cursor: Reviewing against kratos patterns...

✅ Proper context propagation
✅ Clean architecture separation
⚠️ Missing input validation
⚠️ Error not wrapped properly

Suggestions:
1. Add validation in service layer
2. Use errors.Wrap for error handling
3. Add logging middleware
```

## Common Workflows

### Creating a New Project

```
You: Create a new kratos project called "order-service"

Cursor: Creating new kratos project...

```bash
kratos new order-service
cd order-service
```

Project structure created:
- api/ - Protocol buffer definitions
- cmd/ - Application entry points
- internal/ - Internal packages
  - biz/ - Business logic
  - data/ - Data access layer
  - server/ - HTTP/gRPC servers
  - service/ - Service implementations
- configs/ - Configuration files
```

### Adding an API Endpoint

```
You: Add a GetUser endpoint to my user service

Cursor: Adding GetUser endpoint...

1. Updating api/user.proto:

```protobuf
service User {
  rpc CreateUser (CreateUserRequest) returns (CreateUserReply);
  rpc GetUser (GetUserRequest) returns (GetUserReply);  // Added
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserReply {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

2. Generating code: kratos proto server api/user.proto
3. Implementing GetUser in service layer
4. Implementing GetUser in biz layer
5. Implementing GetUser in data layer

✅ GetUser endpoint added!
```

### Adding Middleware

```
You: Add logging middleware to my HTTP server

Cursor: Adding logging middleware...

1. Creating middleware/logging.go
2. Implementing logging middleware
3. Registering in HTTP server
4. Testing middleware

✅ Logging middleware added!
```

## Best Practices

### 1. Reference Patterns

```
You: How should I implement the repository pattern?

Cursor: Based on .ai-context/kratos-skills/references/database-patterns.md:

```go
// In internal/biz/user.go
type UserRepo interface {
    Create(ctx context.Context, u *User) error
    Get(ctx context.Context, id int64) (*User, error)
    Update(ctx context.Context, u *User) error
    Delete(ctx context.Context, id int64) error
}

// In internal/data/user.go
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
```
```

### 2. Use Context

```
You: Should I pass context to my database calls?

Cursor: Yes, always pass context for:
- Request cancellation
- Distributed tracing
- Timeout handling

```go
// ✅ Correct
func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    return r.data.db.User.Get(ctx, id)
}

// ❌ Incorrect
func (r *userRepo) Get(id int64) (*biz.User, error) {
    return r.data.db.User.Get(context.Background(), id)
}
```
```

### 3. Error Handling

```
You: How should I handle errors in kratos?

Cursor: Use kratos errors package:

```go
import "github.com/go-kratos/kratos/v2/errors"

// Create error
err := errors.BadRequest("USER_NOT_FOUND", "user not found")

// Wrap error
err := errors.InternalServer("DATABASE_ERROR", "database error").WithCause(err)

// Check error code
if errors.IsBadRequest(err) {
    // Handle bad request
}
```
```

## Troubleshooting

### Rules Not Applied

```bash
# Check .cursorrules exists
cat .cursorrules

# Verify file is in project root
ls -la .cursorrules

# Restart Cursor
```

### Suggestions Not Relevant

1. Ensure `.ai-context/kratos-skills/` exists
2. Check that proto files are properly defined
3. Verify go.mod contains kratos dependency

### Code Generation Issues

```bash
# Verify kratos CLI
which kratos
kratos --version

# Check proto compiler
which protoc
protoc --version

# Regenerate code
kratos proto server api/*.proto
```

## Tips and Tricks

### 1. Quick Pattern Lookup

```
You: Show me the HTTP handler pattern

Cursor: From .ai-context/kratos-skills/references/http-patterns.md:

```go
func (s *UserService) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserReply, error) {
    // 1. Validate input
    // 2. Call biz layer
    // 3. Return response
}
```
```

### 2. Compare Patterns

```
You: What's the difference between service and biz layers?

Cursor: Service layer (internal/service/):
- HTTP/gRPC transport concerns
- Request/response handling
- Input validation
- Calls biz layer

Biz layer (internal/biz/):
- Business logic
- Use cases
- Domain models
- Repository interfaces
```

### 3. Generate Tests

```
You: Generate unit tests for my user biz layer

Cursor: Generating tests...

```go
func TestUserUsecase_Create(t *testing.T) {
    // Mock repository
    // Test cases
    // Assertions
}
```
```

## Next Steps

1. Try the [examples](../examples/) to see kratos-skills in action
2. Explore the [pattern guides](../references/) for detailed patterns
3. Review [best practices](../best-practices/) for production code
4. Check [troubleshooting](../troubleshooting/) for common issues

## Resources

- [Cursor Documentation](https://cursor.sh/docs)
- [go-kratos Documentation](https://go-kratos.dev/)
