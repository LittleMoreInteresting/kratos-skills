# Distributed Tracing Patterns (go-kratos)

This guide covers OpenTelemetry integration for distributed tracing and metrics in go-kratos.

## Overview

- Use OpenTelemetry for standardized tracing and metrics
- Support multiple exporters (Jaeger, OTLP, Prometheus)
- Automatic context propagation across service boundaries
- Built-in middleware for HTTP and gRPC

## Setup

### Dependencies

```bash
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/sdk
go get go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc
go get go.opentelemetry.io/otel/exporters/prometheus
go get github.com/go-kratos/kratos/v2/middleware/tracing
go get github.com/go-kratos/kratos/v2/middleware/metrics
```

---

## Basic Tracing Configuration

### 1. Tracer Provider Setup

```go
// internal/tracing/tracing.go
package tracing

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/log"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.4.0"
    "go.opentelemetry.io/otel/trace"
)

// TracerProvider manages the OpenTelemetry tracer provider
type TracerProvider struct {
    tp *sdktrace.TracerProvider
}

// NewTracerProvider creates a new tracer provider
func NewTracerProvider(ctx context.Context, serviceName, endpoint string) (*TracerProvider, error) {
    // Create OTLP exporter
    exp, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint(endpoint),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    // Create tracer provider
    tp := sdktrace.NewTracerProvider(
        // Set sampling rate to 100% for development
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(1.0))),
        // Batch span processor for production
        sdktrace.WithBatcher(exp),
        // Resource attributes
        sdktrace.WithResource(resource.NewSchemaless(
            semconv.ServiceNameKey.String(serviceName),
            attribute.String("env", "dev"),
            attribute.String("version", "v1.0.0"),
        )),
    )
    
    // Set as global tracer provider
    otel.SetTracerProvider(tp)
    
    // Set propagator
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))
    
    return &TracerProvider{tp: tp}, nil
}

// Tracer returns a tracer instance
func (p *TracerProvider) Tracer(name string) trace.Tracer {
    return p.tp.Tracer(name)
}

// Shutdown gracefully shuts down the tracer provider
func (p *TracerProvider) Shutdown(ctx context.Context) error {
    return p.tp.Shutdown(ctx)
}
```

### 2. HTTP Server with Tracing

```go
// internal/server/http.go
package server

import (
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/middleware/logging"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/tracing"
    "github.com/go-kratos/kratos/v2/transport/http"
    
    v1 "user-service/api/user/v1"
    "user-service/internal/conf"
    "user-service/internal/service"
    
    "go.opentelemetry.io/otel/trace"
)

func NewHTTPServer(
    c *conf.Server,
    user *service.UserService,
    logger log.Logger,
    tp trace.TracerProvider,
) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            recovery.Recovery(),
            tracing.Server(
                tracing.WithTracerProvider(tp),
            ),
            logging.Server(logger),
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

### 3. gRPC Server with Tracing

```go
// internal/server/grpc.go
package server

import (
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/middleware/logging"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/tracing"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    
    v1 "user-service/api/user/v1"
    "user-service/internal/conf"
    "user-service/internal/service"
    
    "go.opentelemetry.io/otel/trace"
)

func NewGRPCServer(
    c *conf.Server,
    user *service.UserService,
    logger log.Logger,
    tp trace.TracerProvider,
) *grpc.Server {
    var opts = []grpc.ServerOption{
        grpc.Middleware(
            recovery.Recovery(),
            tracing.Server(
                tracing.WithTracerProvider(tp),
            ),
            logging.Server(logger),
        ),
    }
    
    if c.Grpc.Addr != "" {
        opts = append(opts, grpc.Address(c.Grpc.Addr))
    }
    
    srv := grpc.NewServer(opts...)
    v1.RegisterUserServer(srv, user)
    
    return srv
}
```

---

## Client Tracing

### HTTP Client

```go
// internal/data/client.go
package data

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/tracing"
    "github.com/go-kratos/kratos/v2/transport/http"
)

func NewExampleClient(tp trace.TracerProvider) (*http.Client, error) {
    conn, err := http.NewClient(
        context.Background(),
        http.WithEndpoint("http://example-service:8000"),
        http.WithMiddleware(
            recovery.Recovery(),
            tracing.Client(
                tracing.WithTracerProvider(tp),
            ),
        ),
        http.WithTimeout(5*time.Second),
    )
    if err != nil {
        return nil, err
    }
    
    return conn, nil
}
```

### gRPC Client

```go
// internal/data/client.go
package data

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/tracing"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    grpcx "google.golang.org/grpc"
)

func NewGreeterClient(tp trace.TracerProvider) (*grpc.ClientConn, error) {
    conn, err := grpc.DialInsecure(
        context.Background(),
        grpc.WithEndpoint("discovery:///greeter-service"),
        grpc.WithMiddleware(
            recovery.Recovery(),
            tracing.Client(
                tracing.WithTracerProvider(tp),
            ),
        ),
        grpc.WithTimeout(5*time.Second),
        // Enable tracing stats handler for remote IP recording
        grpc.WithOptions(grpcx.WithStatsHandler(&tracing.ClientHandler{})),
    )
    if err != nil {
        return nil, err
    }
    
    return conn, nil
}
```

---

## Custom Span Creation

### Manual Span Creation

```go
// internal/biz/user.go
package biz

import (
    "context"
    
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/trace"
)

func (uc *UserUsecase) CreateUser(ctx context.Context, u *User) error {
    // Get tracer from context or use global
    tracer := trace.SpanFromContext(ctx).TracerProvider().Tracer("user-usecase")
    
    // Create span
    ctx, span := tracer.Start(ctx, "CreateUser")
    defer span.End()
    
    // Add attributes
    span.SetAttributes(
        attribute.String("user.email", u.Email),
        attribute.Int64("user.age", int64(u.Age)),
    )
    
    // Check if email exists
    existing, err := uc.repo.GetByEmail(ctx, u.Email)
    if err != nil {
        // Record error
        span.RecordError(err)
        span.SetStatus(codes.Error, "failed to check email")
        return err
    }
    
    if existing != nil {
        // Add event
        span.AddEvent("email_already_exists")
        return ErrUserExists
    }
    
    // Create user
    if err := uc.repo.Create(ctx, u); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "failed to create user")
        return err
    }
    
    // Set OK status
    span.SetStatus(codes.Ok, "user created successfully")
    span.SetAttributes(attribute.Int64("user.id", u.ID))
    
    return nil
}
```

### Repository Layer Spans

```go
// internal/data/user.go
package data

import (
    "context"
    
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/trace"
)

func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    ctx, span := r.data.tracer.Start(ctx, "userRepo.Get")
    defer span.End()
    
    span.SetAttributes(attribute.Int64("user.id", id))
    
    var user User
    result := r.data.db.WithContext(ctx).First(&user, id)
    if result.Error != nil {
        span.RecordError(result.Error)
        return nil, result.Error
    }
    
    return toBizUser(&user), nil
}
```

---

## Metrics with OpenTelemetry

### Meter Provider Setup

```go
// internal/metrics/metrics.go
package metrics

import (
    "github.com/go-kratos/kratos/v2/middleware/metrics"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/exporters/prometheus"
    "go.opentelemetry.io/otel/metric"
    sdkmetric "go.opentelemetry.io/otel/sdk/metric"
    "go.opentelemetry.io/otel/sdk/resource"
    semconv "go.opentelemetry.io/otel/semconv/v1.4.0"
)

func NewMeterProvider(serviceName string) (metric.MeterProvider, error) {
    // Create Prometheus exporter
    exporter, err := prometheus.New()
    if err != nil {
        return nil, err
    }
    
    // Create meter provider
    provider := sdkmetric.NewMeterProvider(
        sdkmetric.WithResource(
            resource.NewWithAttributes(
                semconv.SchemaURL,
                semconv.ServiceNameKey.String(serviceName),
                attribute.String("environment", "production"),
            ),
        ),
        sdkmetric.WithReader(exporter),
        sdkmetric.WithView(
            // Custom histogram buckets for request duration
            metrics.DefaultSecondsHistogramView(
                metrics.DefaultServerSecondsHistogramName,
            ),
        ),
    )
    
    // Set as global meter provider
    otel.SetMeterProvider(provider)
    
    return provider, nil
}
```

### Server Metrics

```go
// internal/server/http.go
package server

import (
    "github.com/go-kratos/kratos/v2/middleware/metrics"
    "github.com/go-kratos/kratos/v2/transport/http"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "go.opentelemetry.io/otel/metric"
)

func NewHTTPServer(
    c *conf.Server,
    user *service.UserService,
    logger log.Logger,
    meter metric.Meter,
    tp trace.TracerProvider,
) (*http.Server, error) {
    // Create metrics
    counter, err := metrics.DefaultRequestsCounter(
        meter,
        metrics.DefaultServerRequestsCounterName,
    )
    if err != nil {
        return nil, err
    }
    
    seconds, err := metrics.DefaultSecondsHistogram(
        meter,
        metrics.DefaultServerSecondsHistogramName,
    )
    if err != nil {
        return nil, err
    }
    
    var opts = []http.ServerOption{
        http.Middleware(
            recovery.Recovery(),
            tracing.Server(tracing.WithTracerProvider(tp)),
            logging.Server(logger),
            metrics.Server(
                metrics.WithRequests(counter),
                metrics.WithSeconds(seconds),
            ),
        ),
    }
    
    srv := http.NewServer(opts...)
    
    // Register Prometheus metrics endpoint
    srv.HandlePrefix("/metrics", promhttp.Handler())
    
    v1.RegisterUserHTTPServer(srv, user)
    
    return srv, nil
}
```

### Custom Metrics

```go
// internal/biz/user.go
package biz

import (
    "context"
    "go.opentelemetry.io/otel/metric"
)

type UserUsecase struct {
    repo   UserRepo
    meter  metric.Meter
    
    // Custom metrics
    userCreatedCounter metric.Int64Counter
    userActiveGauge    metric.Int64UpDownCounter
}

func NewUserUsecase(repo UserRepo, meter metric.Meter, logger log.Logger) (*UserUsecase, error) {
    // Create custom counters
    createdCounter, err := meter.Int64Counter(
        "user.created",
        metric.WithDescription("Number of users created"),
    )
    if err != nil {
        return nil, err
    }
    
    activeGauge, err := meter.Int64UpDownCounter(
        "user.active",
        metric.WithDescription("Number of active users"),
    )
    if err != nil {
        return nil, err
    }
    
    return &UserUsecase{
        repo:               repo,
        meter:              meter,
        userCreatedCounter: createdCounter,
        userActiveGauge:    activeGauge,
    }, nil
}

func (uc *UserUsecase) CreateUser(ctx context.Context, u *User) error {
    // ... validation logic ...
    
    if err := uc.repo.Create(ctx, u); err != nil {
        return err
    }
    
    // Increment counter
    uc.userCreatedCounter.Add(ctx, 1)
    uc.userActiveGauge.Add(ctx, 1)
    
    return nil
}
```

---

## Logging with Trace Context

### Logger Setup

```go
// cmd/server/main.go
package main

import (
    "os"
    
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/middleware/tracing"
)

func main() {
    // Create logger with trace context
    logger := log.NewStdLogger(os.Stdout)
    logger = log.With(logger,
        "trace_id", tracing.TraceID(),
        "span_id", tracing.SpanID(),
    )
    
    // Use logger in your application
    log := log.NewHelper(logger)
    log.Info("server starting...")
}
```

### Output Example

```json
{
  "level": "info",
  "msg": "user created",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "user_id": "12345"
}
```

---

## Configuration

### Config Proto

```protobuf
// internal/conf/conf.proto
syntax = "proto3";

package internal.conf;

message Bootstrap {
    Server server = 1;
    Data data = 2;
    Trace trace = 3;
    Metric metric = 4;
}

message Trace {
    bool enabled = 1;
    string endpoint = 2;  // Jaeger/OTLP endpoint
    float sampling_rate = 3;  // 0.0 - 1.0
}

message Metric {
    bool enabled = 1;
    string endpoint = 2;
}
```

### Config YAML

```yaml
# configs/config.yaml
trace:
  enabled: true
  endpoint: jaeger:4317
  sampling_rate: 1.0  # 100% sampling for dev

metric:
  enabled: true
  endpoint: :9090
```

---

## Testing with Tracing

```go
// internal/biz/user_test.go
package biz

import (
    "context"
    "testing"
    
    "github.com/stretchr/testify/assert"
    "go.opentelemetry.io/otel/trace"
)

type mockTracer struct {
    trace.Tracer
}

func TestUserUsecase_CreateUser(t *testing.T) {
    // Create noop tracer for testing
    tracer := trace.NewNoopTracerProvider().Tracer("test")
    
    mockRepo := new(MockUserRepo)
    uc := NewUserUsecase(mockRepo, tracer, log.DefaultLogger)
    
    ctx := context.Background()
    user := &User{
        Name:  "John",
        Email: "john@example.com",
    }
    
    mockRepo.On("GetByEmail", mock.Anything, "john@example.com").
        Return(nil, errors.NotFound("NOT_FOUND", ""))
    mockRepo.On("Create", mock.Anything, user).
        Return(nil)
    
    err := uc.CreateUser(ctx, user)
    assert.NoError(t, err)
}
```

---

## Best Practices

### ✅ Always Follow

- Use OpenTelemetry for standardized tracing
- Set appropriate sampling rates (100% dev, 1-10% prod)
- Always propagate context through all layers
- Add meaningful attributes to spans
- Use span events for important milestones
- Record errors with stack traces
- Enable tracing for both HTTP and gRPC

### ❌ Never Do

```go
// DON'T: Break context propagation
func (s *Service) Handle(ctx context.Context, req *Request) {
    // ❌ Loses trace context
    go process(context.Background(), req)
}

// DON'T: Create spans without ending them
ctx, span := tracer.Start(ctx, "operation")
// ❌ Missing defer span.End()

// DON'T: Ignore span errors
if err != nil {
    // ❌ Should record error
    return err
}
if err != nil {
    span.RecordError(err)  // ✅
    return err
}

// DON'T: Use high cardinality attributes
span.SetAttributes(attribute.String("user.email", email))  // ❌ Too high cardinality
span.SetAttributes(attribute.Int64("user.id", userID))     // ✅ Better
```

---

## Troubleshooting

### No Traces in Jaeger

1. Check endpoint configuration
2. Verify sampling rate > 0
3. Ensure spans are ended
4. Check for network connectivity

### Missing Parent Spans

1. Verify context propagation
2. Check for context cancellation
3. Ensure middleware is applied correctly

### High Memory Usage

1. Reduce batch size
2. Lower sampling rate
3. Check for span leaks (spans not ended)
