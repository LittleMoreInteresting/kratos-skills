# Best Practices

Production recommendations for building go-kratos applications.

## Project Structure

### Recommended Layout

```
my-service/
├── api/                          # Protocol buffer definitions
│   └── v1/
│       ├── user.proto
│       ├── user.pb.go
│       ├── user_grpc.pb.go
│       └── user_http.pb.go
├── cmd/
│   └── my-service/
│       └── main.go              # Application entry point
├── configs/
│   ├── config.yaml              # Base configuration
├── internal/
│   ├── biz/                     # Business logic
│   │   ├── user.go
│   │   └── biz.go
│   ├── conf/                    # Configuration structs
│   │   └── conf.go
│   ├── data/                    # Data access layer
│   │   ├── user.go
│   │   └── data.go
│   ├── server/                  # HTTP/gRPC servers
│   │   ├── http.go
│   │   └── grpc.go
│   ├── service/                 # Service implementations
│   │   ├── userservice.go
│   │   └── service.go
│   └── wire.go                  # Wire configuration
├── pkg/                         # Public packages (optional)
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
└── README.md
```

## Configuration Management

### Environment-Based Configuration

```yaml
# configs/config.yaml (base)
server:
  http:
    addr: 0.0.0.0:8000
    timeout: 1s
  grpc:
    addr: 0.0.0.0:9000
    timeout: 1s

data:
  database:
    driver: mysql
    source: ${DATABASE_URL}
    max_idle_conns: 10
    max_open_conns: 100

log:
  level: info
  format: json
```

```yaml
# configs/config_dev.yaml
data:
  database:
    source: root:password@tcp(localhost:3306)/dev_db?parseTime=true

log:
  level: debug
  format: console
```

```yaml
# configs/config_prod.yaml
data:
  database:
    source: ${DATABASE_URL}
    max_idle_conns: 20
    max_open_conns: 200

log:
  level: warn
  format: json
```

### Configuration Loading

```go
// cmd/my-service/main.go
func main() {
    flag.Parse()
    
    // Load base config
    c := config.New(
        config.WithSource(
            file.NewSource(flagconf),
        ),
    )
    defer c.Close()
    
    if err := c.Load(); err != nil {
        panic(err)
    }
    
    var bc conf.Bootstrap
    if err := c.Scan(&bc); err != nil {
        panic(err)
    }
    
    // Override with environment variables
    // Use github.com/caarlos0/env for env parsing
}
```

## Logging

### Structured Logging

```go
// internal/service/userservice.go
func (s *UserService) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserReply, error) {
    s.log.Infow("creating user",
        "name", req.Name,
        "email", req.Email,
    )
    
    user, err := s.uc.Create(ctx, &biz.User{
        Name:  req.Name,
        Email: req.Email,
    })
    if err != nil {
        s.log.Errorw("failed to create user",
            "error", err,
            "email", req.Email,
        )
        return nil, err
    }
    
    s.log.Infow("user created",
        "id", user.ID,
        "email", user.Email,
    )
    
    return &pb.CreateUserReply{
        Id:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}
```

### Log Levels

| Level | Use Case |
|-------|----------|
| DEBUG | Development, detailed debugging |
| INFO | Normal operations |
| WARN | Unexpected but handled issues |
| ERROR | Errors that need attention |
| FATAL | Critical errors, application exit |

### Production Logging

```yaml
# configs/config.yaml
log:
  level: warn          # Only warn and above
  format: json         # Structured JSON logs
  output: file         # Log to file
  filename: /var/log/my-service/app.log
  max_size: 100        # MB
  max_backups: 10
  max_age: 30          # days
```

## Error Handling

### Error Codes

```go
// pkg/errors/errors.go
package errors

import (
    "github.com/go-kratos/kratos/v2/errors"
)

// Domain-specific error codes
var (
    // User errors
    ErrUserNotFound     = errors.NotFound("USER_NOT_FOUND", "user not found")
    ErrUserExists       = errors.Conflict("USER_EXISTS", "user already exists")
    ErrInvalidEmail     = errors.BadRequest("INVALID_EMAIL", "invalid email format")
    ErrInvalidPassword  = errors.BadRequest("INVALID_PASSWORD", "invalid password")
    
    // Auth errors
    ErrUnauthorized     = errors.Unauthorized("UNAUTHORIZED", "unauthorized")
    ErrInvalidToken     = errors.Unauthorized("INVALID_TOKEN", "invalid token")
    ErrTokenExpired     = errors.Unauthorized("TOKEN_EXPIRED", "token expired")
    
    // System errors
    ErrInternal         = errors.InternalServer("INTERNAL_ERROR", "internal server error")
    ErrDatabase         = errors.InternalServer("DATABASE_ERROR", "database error")
    ErrCache            = errors.InternalServer("CACHE_ERROR", "cache error")
)
```

### Error Wrapping

```go
// internal/data/user.go
func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    var user User
    result := r.data.db.WithContext(ctx).First(&user, id)
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrRecordNotFound) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        // Wrap internal errors
        return nil, errors.InternalServer("DATABASE_ERROR", "failed to get user").
            WithCause(result.Error)
    }
    return toBizUser(&user), nil
}
```

## Database

### Connection Pooling

```go
// internal/data/data.go
func NewData(c *conf.Data, log log.Logger) (*Data, func(), error) {
    db, err := gorm.Open(mysql.Open(c.Database.Source), &gorm.Config{})
    if err != nil {
        return nil, nil, err
    }
    
    sqlDB, err := db.DB()
    if err != nil {
        return nil, nil, err
    }
    
    // Production settings
    sqlDB.SetMaxIdleConns(20)           // Minimum connections
    sqlDB.SetMaxOpenConns(200)          // Maximum connections
    sqlDB.SetConnMaxLifetime(time.Hour) // Connection lifetime
    sqlDB.SetConnMaxIdleTime(10 * time.Minute)
    
    // Health check
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := sqlDB.PingContext(ctx); err != nil {
        return nil, nil, err
    }
    
    // ...
}
```

### Database Migrations

```go
// migrations/migrate.go
package migrations

import (
    "github.com/go-gormigrate/gormigrate/v2"
    "gorm.io/gorm"
)

func Migrate(db *gorm.DB) error {
    m := gormigrate.New(db, gormigrate.DefaultOptions, []*gormigrate.Migration{
        {
            ID: "20240101000001",
            Migrate: func(tx *gorm.DB) error {
                return tx.AutoMigrate(&User{})
            },
            Rollback: func(tx *gorm.DB) error {
                return tx.Migrator().DropTable("users")
            },
        },
        {
            ID: "20240101000002",
            Migrate: func(tx *gorm.DB) error {
                return tx.AutoMigrate(&Order{})
            },
            Rollback: func(tx *gorm.DB) error {
                return tx.Migrator().DropTable("orders")
            },
        },
    })
    
    return m.Migrate()
}
```

## Security

### Input Validation

```protobuf
// api/user/v1/user.proto
message CreateUserRequest {
    string name = 1 [
        (validate.rules).string = {
            min_len: 1,
            max_len: 50,
            pattern: "^[a-zA-Z0-9_]+$"
        }
    ];
    string email = 2 [
        (validate.rules).string = {email: true}
    ];
    string password = 3 [
        (validate.rules).string = {
            min_len: 8,
            max_len: 100,
            pattern: "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).+$"
        }
    ];
}
```

### Password Hashing

```go
// internal/biz/user.go
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword(
        []byte(password),
        bcrypt.DefaultCost, // 10
    )
    return string(bytes), err
}

func verifyPassword(password, hash string) error {
    return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
}
```

### Rate Limiting

```go
// internal/middleware/ratelimit.go
import "golang.org/x/time/rate"

func RateLimiter(rps int) middleware.Middleware {
    limiter := rate.NewLimiter(rate.Limit(rps), rps*2)
    
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            if !limiter.Allow() {
                return nil, errors.TooManyRequests(
                    "RATE_LIMIT",
                    "too many requests",
                )
            }
            return handler(ctx, req)
        }
    }
}
```

## Monitoring

### Health Checks

```go
// internal/server/health.go
package server

import (
    "context"
    "net/http"
    
    "github.com/go-kratos/kratos/v2/health"
)

func NewHealthServer() *http.Server {
    hs := health.NewServer()
    
    // Add health checks
    hs.AddChecker("database", health.CheckerFunc(func(ctx context.Context) error {
        // Check database connection
        return db.PingContext(ctx)
    }))
    
    hs.AddChecker("redis", health.CheckerFunc(func(ctx context.Context) error {
        // Check redis connection
        return rdb.Ping(ctx).Err()
    }))
    
    return hs
}
```

### Metrics

```go
// internal/middleware/metrics.go
import "github.com/prometheus/client_golang/prometheus"

var (
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path", "status"},
    )
    
    requestCount = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_request_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )
)

func init() {
    prometheus.MustRegister(requestDuration, requestCount)
}
```

## Performance

### Caching Strategy

```go
// internal/data/user.go
func (r *userRepo) GetWithCache(ctx context.Context, id int64) (*biz.User, error) {
    cacheKey := fmt.Sprintf("user:%d", id)
    
    // Try cache first
    cached, err := r.cache.Get(ctx, cacheKey).Result()
    if err == nil {
        var user biz.User
        if err := json.Unmarshal([]byte(cached), &user); err == nil {
            return &user, nil
        }
    }
    
    // Cache miss - get from DB
    user, err := r.Get(ctx, id)
    if err != nil {
        return nil, err
    }
    
    // Update cache asynchronously
    go func() {
        data, _ := json.Marshal(user)
        r.cache.Set(ctx, cacheKey, data, 10*time.Minute)
    }()
    
    return user, nil
}
```

### Request Timeouts

```yaml
# configs/config.yaml
server:
  http:
    addr: 0.0.0.0:8000
    timeout: 30s          # Request timeout
    read_timeout: 10s     # Read timeout
    write_timeout: 30s    # Write timeout
    idle_timeout: 120s    # Idle timeout
```

## Deployment

### Dockerfile

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o /bin/my-service cmd/my-service/main.go

# Runtime stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates
WORKDIR /app

COPY --from=builder /bin/my-service .
COPY configs/ ./configs/

EXPOSE 8000 9000

USER nonroot:nonroot

CMD ["./my-service", "-conf", "./configs"]
```

### Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
      - name: my-service
        image: my-registry/my-service:v1.0.0
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 9000
          name: grpc
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
```

## Summary

### Production Checklist

- [ ] Configuration externalized (env vars, config files)
- [ ] Structured logging (JSON format)
- [ ] Proper error handling with codes
- [ ] Input validation (proto validation)
- [ ] Database connection pooling
- [ ] Database migrations
- [ ] Password hashing
- [ ] Rate limiting
- [ ] Health checks
- [ ] Metrics collection
- [ ] Caching strategy
- [ ] Request timeouts
- [ ] Security headers
- [ ] Non-root Docker user
- [ ] Resource limits
- [ ] Liveness/readiness probes
