# Common Issues and Solutions

Troubleshooting guide for go-kratos development.

## Proto Generation Issues

### Issue: `protoc-gen-go: program not found`

**Error:**
```
--go_out: protoc-gen-go: program not found or is not executable
```

**Solution:**
```bash
# Install protoc-gen-go
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# Add to PATH
export PATH=$PATH:$(go env GOPATH)/bin

# Verify
which protoc-gen-go
```

### Issue: `protoc-gen-go-http: program not found`

**Error:**
```
--go-http_out: protoc-gen-go-http: program not found
```

**Solution:**
```bash
# Install kratos http plugin
go install github.com/go-kratos/kratos/cmd/protoc-gen-go-http/v2@latest

# Verify
which protoc-gen-go-http
```

### Issue: `validate/validate.proto: File not found`

**Error:**
```
validate/validate.proto: File not found.
import "validate/validate.proto";
```

**Solution:**
```bash
# Download validate proto
mkdir -p third_party/validate
curl -o third_party/validate/validate.proto \
  https://raw.githubusercontent.com/envoyproxy/protoc-gen-validate/main/validate/validate.proto

# Generate with proto_path
kratos proto client api/user.proto --proto_path=./third_party
```

### Issue: `google/api/annotations.proto: File not found`

**Error:**
```
google/api/annotations.proto: File not found.
import "google/api/annotations.proto";
```

**Solution:**
```bash
# Download google/api protos
mkdir -p third_party/google/api
curl -o third_party/google/api/annotations.proto \
  https://raw.githubusercontent.com/googleapis/googleapis/master/google/api/annotations.proto
curl -o third_party/google/api/http.proto \
  https://raw.githubusercontent.com/googleapis/googleapis/master/google/api/http.proto

# Generate with proto_path
kratos proto client api/user.proto --proto_path=./third_party
```

## Wire Issues

### Issue: `wire: no injector found`

**Error:**
```
wire: no injector found for github.com/user/project/internal.wireApp
```

**Solution:**
```go
// internal/wire.go
//go:build wireinject
// +build wireinject

package internal

import "github.com/google/wire"

// Must include //go:build wireinject comment!
func wireApp(...) (*kratos.App, func(), error) {
    panic(wire.Build(...))
}
```

### Issue: `wire: provider not found`

**Error:**
```
wire: github.com/user/project/internal/biz.NewUserUsecase: no provider found for github.com/user/project/internal/biz.UserRepo
```

**Solution:**
```go
// internal/data/data.go
var ProviderSet = wire.NewSet(
    NewData,
    NewUserRepo,  // Must be included!
    // ...
)

// Ensure interface binding
var _ biz.UserRepo = (*userRepo)(nil)
```

### Issue: `wire: cycle detected`

**Error:**
```
wire: cycle detected: *service.UserService -> *biz.UserUsecase -> *data.userRepo -> *Data
```

**Solution:**
```go
// Check for circular dependencies
// Ensure Data doesn't depend on repositories

// internal/data/data.go
type Data struct {
    db *gorm.DB
}

// Repositories should depend on Data, not vice versa
```

## Database Issues

### Issue: `failed to initialize database`

**Error:**
```
failed to initialize database, got error dial tcp: connect: connection refused
```

**Solution:**
```yaml
# configs/config.yaml
data:
  database:
    driver: mysql
    source: user:password@tcp(localhost:3306)/dbname?parseTime=true&loc=Local
```

```bash
# Check database connection
mysql -u user -p -h localhost -e "SELECT 1"

# Verify connection string
go run -e 'package main; import "fmt"; func main() { fmt.Println("Connection test") }'
```

### Issue: `record not found`

**Error:**
```
record not found
```

**Solution:**
```go
// Check error type
result := db.First(&user, id)
if result.Error != nil {
    if errors.Is(result.Error, gorm.ErrRecordNotFound) {
        // Handle not found
        return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
    }
    return nil, result.Error
}
```

### Issue: `Error 1062: Duplicate entry`

**Error:**
```
Error 1062: Duplicate entry 'email@example.com' for key 'users.email'
```

**Solution:**
```go
// Check for duplicate before create
existing, err := r.GetByEmail(ctx, email)
if err != nil && !errors.IsNotFound(err) {
    return nil, err
}
if existing != nil {
    return nil, errors.Conflict("EMAIL_EXISTS", "email already exists")
}

// Or handle in create
if err := r.Create(ctx, user); err != nil {
    if isDuplicateKeyError(err) {
        return nil, errors.Conflict("EMAIL_EXISTS", "email already exists")
    }
    return nil, err
}
```

## Build Issues

### Issue: `undefined: conf.Bootstrap`

**Error:**
```
undefined: conf.Bootstrap
cannot use &bc (type *conf.Bootstrap) as type ...
```

**Solution:**
```bash
# Regenerate proto for conf
cd internal/conf
protoc --go_out=. --go_opt=paths=source_relative conf.proto

# Or ensure conf.proto is correct
```

### Issue: `undefined: wire.Gen`

**Error:**
```
undefined: wire.Gen
```

**Solution:**
```bash
# Install wire
go install github.com/google/wire/cmd/wire@latest

# Regenerate
cd internal && wire
```

### Issue: `import cycle not allowed`

**Error:**
```
import cycle not allowed
package github.com/user/project/internal/biz
    imports github.com/user/project/internal/data
    imports github.com/user/project/internal/biz
```

**Solution:**
```go
// internal/biz/user.go
// Define interface in biz layer
type UserRepo interface {
    Create(ctx context.Context, u *User) error
    Get(ctx context.Context, id int64) (*User, error)
}

// internal/data/user.go
// Implement interface in data layer
// NO import of biz package in data layer!
type userRepo struct {
    data *Data
}
```

## Runtime Issues

### Issue: `panic: runtime error: invalid memory address`

**Error:**
```
panic: runtime error: invalid memory address or nil pointer dereference
```

**Solution:**
```go
// Check nil pointers
if svc.uc == nil {
    panic("usecase is nil")
}

// Ensure proper initialization in wire
func wireApp(...) (*kratos.App, func(), error) {
    panic(wire.Build(
        // All providers must be included
        biz.ProviderSet,
        data.ProviderSet,
        service.ProviderSet,
        server.ProviderSet,
    ))
}
```

### Issue: `context deadline exceeded`

**Error:**
```
context deadline exceeded
```

**Solution:**
```yaml
# configs/config.yaml
server:
  http:
    timeout: 30s  # Increase timeout
  grpc:
    timeout: 30s
```

```go
// Or set timeout per request
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

reply, err := client.SayHello(ctx, req)
```

### Issue: `connection refused`

**Error:**
```
dial tcp localhost:9000: connect: connection refused
```

**Solution:**
```bash
# Check if server is running
netstat -tlnp | grep 9000

# Check server configuration
cat configs/config.yaml | grep -A 2 grpc

# Start server
go run cmd/my-service/main.go
```

## Configuration Issues

### Issue: `config file not found`

**Error:**
```
config file not found: ./configs/config.yaml
```

**Solution:**
```bash
# Check config file exists
ls -la configs/

# Run with correct path
go run cmd/my-service/main.go -conf ./configs

# Or use absolute path
go run cmd/my-service/main.go -conf /app/configs
```

### Issue: `cannot unmarshal string into Go struct`

**Error:**
```
cannot unmarshal string into Go struct field Bootstrap.server of type conf.Server
```

**Solution:**
```yaml
# configs/config.yaml
# Use correct types
server:
  http:
    addr: "0.0.0.0:8000"  # String
    timeout: 30s          # Duration
```

```go
// internal/conf/conf.go
type Server struct {
    Http *HTTP
}

type HTTP struct {
    Addr    string               `json:"addr"`
    Timeout *durationpb.Duration `json:"timeout"`
}
```

## Middleware Issues

### Issue: `middleware not working`

**Problem:** Middleware not being invoked

**Solution:**
```go
// internal/server/http.go
func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            recovery.Recovery(),    // Must be first!
            logging.Server(logger),
            authMiddleware(),       // Custom middleware
            validate.Validator(),
        ),
    }
    
    srv := http.NewServer(opts...)
    v1.RegisterUserHTTPServer(srv, user)
    
    return srv
}
```

### Issue: `auth middleware panics`

**Error:**
```
panic: runtime error: index out of range [0] with length 0
```

**Solution:**
```go
func authMiddleware() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            tr, ok := transport.FromServerContext(ctx)
            if !ok {
                return nil, errors.Unauthorized("TRANSPORT_ERROR", "no transport")
            }
            
            token := tr.RequestHeader().Get("Authorization")
            if token == "" {
                return nil, errors.Unauthorized("MISSING_TOKEN", "token required")
            }
            
            // ...
        }
    }
}
```

## Testing Issues

### Issue: `test timeout`

**Error:**
```
timeout: test ran for too long
```

**Solution:**
```bash
# Increase timeout
go test -timeout 30s ./...

# Or in test
func TestLongRunning(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    // ...
}
```

### Issue: `no tests to run`

**Error:**
```
testing: warning: no tests to run
```

**Solution:**
```go
// Test function must start with Test
func TestUserService(t *testing.T) {
    // ...
}

// Not test or Test_ (underscore)
```

## Debugging Tips

### Enable Debug Logging

```yaml
# configs/config.yaml
log:
  level: debug
  format: console
```

### Add Debug Middleware

```go
func debugMiddleware() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            log.Debugf("Request: %+v", req)
            
            reply, err := handler(ctx, req)
            
            log.Debugf("Response: %+v, Error: %v", reply, err)
            return reply, err
        }
    }
}
```

### Use Delve Debugger

```bash
# Install delve
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug
dlv debug cmd/my-service/main.go

# Set breakpoint
(dlv) break internal/service/userservice.go:45
(dlv) continue
```

## Getting Help

### Resources

- [go-kratos Documentation](https://go-kratos.dev/)
- [go-kratos GitHub Issues](https://github.com/go-kratos/kratos/issues)
- [Go Documentation](https://golang.org/doc/)

### Debug Checklist

- [ ] Check error messages carefully
- [ ] Verify all dependencies installed
- [ ] Check configuration files
- [ ] Review recent code changes
- [ ] Enable debug logging
- [ ] Check server logs
- [ ] Verify database connection
- [ ] Test with minimal example
