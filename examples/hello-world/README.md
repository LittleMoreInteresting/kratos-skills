# Hello World Example

Basic go-kratos service with HTTP and gRPC endpoints.

## Quick Start

### 1. Create Project

```bash
# Create new kratos project
kratos new hello-world
cd hello-world
```

### 2. Define Proto

```protobuf
// api/helloworld/v1/helloworld.proto
syntax = "proto3";

package api.helloworld.v1;

option go_package = "hello-world/api/helloworld/v1;v1";

import "google/api/annotations.proto";

service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply) {
        option (google.api.http) = {
            get: "/helloworld/{name}"
        };
    }
}

message HelloRequest {
    string name = 1;
}

message HelloReply {
    string message = 1;
}
```

### 3. Generate Code

```bash
kratos proto client api/helloworld/v1/helloworld.proto
kratos proto server api/helloworld/v1/helloworld.proto
cd internal && wire && cd ..
```

### 4. Run

```bash
go run cmd/hello-world/main.go
```

### 5. Test

```bash
# HTTP
curl http://localhost:8000/helloworld/world

# gRPC
ggrpcurl -plaintext localhost:9000 list
ggrpcurl -plaintext -d '{"name": "world"}' localhost:9000 helloworld.Greeter/SayHello
```

## Project Structure

```
hello-world/
├── api/
│   └── helloworld/
│       └── v1/
│           ├── helloworld.proto
│           ├── helloworld.pb.go
│           ├── helloworld_grpc.pb.go
│           └── helloworld_http.pb.go
├── cmd/
│   └── hello-world/
│       └── main.go
├── configs/
│   └── config.yaml
├── internal/
│   ├── biz/
│   │   └── greeter.go
│   ├── conf/
│   │   └── conf.go
│   ├── data/
│   │   └── greeter.go
│   ├── server/
│   │   ├── http.go
│   │   └── grpc.go
│   ├── service/
│   │   └── greeterservice.go
│   └── wire_gen.go
├── go.mod
└── go.sum
```

## Key Files

### Service Implementation

```go
// internal/service/greeterservice.go
package service

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/log"
    v1 "hello-world/api/helloworld/v1"
)

type GreeterService struct {
    v1.UnimplementedGreeterServer
    log *log.Helper
}

func NewGreeterService(logger log.Logger) *GreeterService {
    return &GreeterService{
        log: log.NewHelper(logger),
    }
}

func (s *GreeterService) SayHello(ctx context.Context, req *v1.HelloRequest) (*v1.HelloReply, error) {
    s.log.Infof("SayHello called with name: %s", req.Name)
    
    return &v1.HelloReply{
        Message: "Hello " + req.Name,
    }, nil
}
```

### HTTP Server

```go
// internal/server/http.go
package server

import (
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/transport/http"
    v1 "hello-world/api/helloworld/v1"
    "hello-world/internal/conf"
    "hello-world/internal/service"
)

func NewHTTPServer(c *conf.Server, greeter *service.GreeterService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            recovery.Recovery(),
        ),
    }
    
    if c.Http.Addr != "" {
        opts = append(opts, http.Address(c.Http.Addr))
    }
    
    srv := http.NewServer(opts...)
    v1.RegisterGreeterHTTPServer(srv, greeter)
    
    return srv
}
```

### Main

```go
// cmd/hello-world/main.go
package main

import (
    "flag"
    "os"
    
    "github.com/go-kratos/kratos/v2"
    "github.com/go-kratos/kratos/v2/config"
    "github.com/go-kratos/kratos/v2/config/file"
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    "github.com/go-kratos/kratos/v2/transport/http"
    
    _ "go.uber.org/automaxprocs"
    
    "hello-world/internal/conf"
)

var (
    flagconf string
)

func init() {
    flag.StringVar(&flagconf, "conf", "../../configs", "config path, eg: -conf config.yaml")
}

func newApp(logger log.Logger, gs *grpc.Server, hs *http.Server) *kratos.App {
    return kratos.New(
        kratos.Name("hello-world"),
        kratos.Version("v1.0.0"),
        kratos.Logger(logger),
        kratos.Server(
            gs,
            hs,
        ),
    )
}

func main() {
    flag.Parse()
    
    logger := log.With(log.NewStdLogger(os.Stdout),
        "service.name", "hello-world",
        "service.version", "v1.0.0",
    )
    
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
    
    app, cleanup, err := wireApp(bc.Server, &bc.Data, logger)
    if err != nil {
        panic(err)
    }
    defer cleanup()
    
    if err := app.Run(); err != nil {
        panic(err)
    }
}
```

## Testing

```bash
# Run tests
go test ./...

# Run with coverage
go test -cover ./...
```

## Next Steps

- Add database integration
- Add middleware (auth, logging, metrics)
- Add more endpoints
- Deploy to Kubernetes
