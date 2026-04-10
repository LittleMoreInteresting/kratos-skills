# Kratos CLI Reference

Complete reference for kratos CLI commands and code generation.

## Installation

### Install Kratos CLI

```bash
# Install latest version
go install github.com/go-kratos/kratos/cmd/kratos/v2@latest

# Verify installation
kratos --version
```

### Install Protocol Buffer Tools

```bash
# Install protoc (macOS)
brew install protobuf

# Install protoc (Ubuntu)
apt-get install -y protobuf-compiler

# Install Go protoc plugins
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/protobuf/cmd/protoc-gen-go-grpc@latest
go install github.com/go-kratos/kratos/cmd/protoc-gen-go-http/v2@latest
go install github.com/go-kratos/kratos/cmd/protoc-gen-go-errors/v2@latest

# Install validation plugin
go install github.com/envoyproxy/protoc-gen-validate@latest

# Add to PATH
export PATH=$PATH:$(go env GOPATH)/bin
```

## Project Commands

### Create New Project

```bash
# Create new project
kratos new my-service

# Create with specific layout
kratos new my-service -r https://github.com/go-kratos/kratos-layout.git

# Create in current directory
kratos new . --nomod
```

### Project Structure

```
my-service/
├── api/                    # Protocol buffer definitions
│   └── helloworld/
│       └── v1/
│           ├── helloworld.proto
│           ├── helloworld.pb.go
│           ├── helloworld_grpc.pb.go
│           └── helloworld_http.pb.go
├── cmd/                    # Application entry points
│   └── my-service/
│       └── main.go
├── configs/                # Configuration files
│   └── config.yaml
├── internal/               # Internal packages
│   ├── biz/               # Business logic
│   ├── conf/              # Configuration structs
│   ├── data/              # Data access layer
│   ├── server/            # HTTP/gRPC servers
│   └── service/           # Service implementations
├── go.mod
├── go.sum
└── README.md
```

## Protocol Buffer Commands

### Generate Client Code

```bash
# Generate proto client (message types)
kratos proto client api/helloworld/v1/helloworld.proto

# Generate with custom output
kratos proto client api/helloworld/v1/helloworld.proto --proto_path=./api
```

### Generate Server Code

```bash
# Generate proto server (service implementations)
kratos proto server api/helloworld/v1/helloworld.proto

# Generate with custom output directory
kratos proto server api/helloworld/v1/helloworld.proto --target-dir=./internal
```

### Generate All

```bash
# Generate both client and server
kratos proto client api/helloworld/v1/helloworld.proto
kratos proto server api/helloworld/v1/helloworld.proto
```

### Proto File Template

```protobuf
// api/user/v1/user.proto
syntax = "proto3";

package api.user.v1;

option go_package = "my-service/api/user/v1;v1";

import "google/api/annotations.proto";
import "validate/validate.proto";

service User {
    rpc CreateUser (CreateUserRequest) returns (CreateUserReply) {
        option (google.api.http) = {
            post: "/v1/users"
            body: "*"
        };
    }
    
    rpc GetUser (GetUserRequest) returns (GetUserReply) {
        option (google.api.http) = {
            get: "/v1/users/{id}"
        };
    }
    
    rpc UpdateUser (UpdateUserRequest) returns (UpdateUserReply) {
        option (google.api.http) = {
            put: "/v1/users/{id}"
            body: "*"
        };
    }
    
    rpc DeleteUser (DeleteUserRequest) returns (DeleteUserReply) {
        option (google.api.http) = {
            delete: "/v1/users/{id}"
        };
    }
    
    rpc ListUsers (ListUsersRequest) returns (ListUsersReply) {
        option (google.api.http) = {
            get: "/v1/users"
        };
    }
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
    int64 id = 1 [(validate.rules).int64 = {gt: 0}];
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
    int64 id = 1 [(validate.rules).int64 = {gt: 0}];
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
    int64 id = 1 [(validate.rules).int64 = {gt: 0}];
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

## Wire Commands

### Install Wire

```bash
go install github.com/google/wire/cmd/wire@latest
```

### Generate Wire Code

```bash
# Generate wire_gen.go
cd internal && wire

# Or with verbose output
cd internal && wire gen ./...
```

### Wire Configuration

```go
// internal/wire.go
//go:build wireinject
// +build wireinject

package internal

import (
    "github.com/go-kratos/kratos/v2"
    "github.com/go-kratos/kratos/v2/log"
    "github.com/google/wire"
    
    "my-service/internal/biz"
    "my-service/internal/conf"
    "my-service/internal/data"
    "my-service/internal/server"
    "my-service/internal/service"
)

func wireApp(*conf.Server, *conf.Data, log.Logger) (*kratos.App, func(), error) {
    panic(wire.Build(
        server.ProviderSet,
        data.ProviderSet,
        biz.ProviderSet,
        service.ProviderSet,
        newApp,
    ))
}
```

## Run Commands

### Run Application

```bash
# Run main.go
go run cmd/my-service/main.go

# Run with config
go run cmd/my-service/main.go -conf ./configs

# Run in debug mode
go run cmd/my-service/main.go -conf ./configs -v
```

## Build Commands

### Build Application

```bash
# Build binary
go build -o bin/my-service cmd/my-service/main.go

# Build for Linux
go build -o bin/my-service-linux-amd64 -ldflags "-s -w" cmd/my-service/main.go

# Build with version info
go build -o bin/my-service \
  -ldflags "-X main.Version=v1.0.0 -X main.BuildTime=$(date +%Y%m%d-%H:%M:%S)" \
  cmd/my-service/main.go
```

## Test Commands

### Run Tests

```bash
# Run all tests
go test ./...

# Run with coverage
go test -cover ./...

# Run with coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# Run specific package
go test ./internal/biz/...

# Run with verbose output
go test -v ./internal/biz/...

# Run benchmarks
go test -bench=. ./...
```

## Update Commands

### Update Dependencies

```bash
# Update kratos
go get -u github.com/go-kratos/kratos/v2

# Update all dependencies
go get -u ./...

# Tidy modules
go mod tidy

# Vendor dependencies
go mod vendor
```

## Common Workflows

### Create New Service

```bash
# 1. Create project
kratos new user-service
cd user-service

# 2. Define proto
cat > api/user/v1/user.proto << 'EOF'
syntax = "proto3";

package api.user.v1;
option go_package = "user-service/api/user/v1;v1";

service User {
    rpc GetUser (GetUserRequest) returns (GetUserReply);
}

message GetUserRequest {
    int64 id = 1;
}

message GetUserReply {
    int64 id = 1;
    string name = 2;
}
EOF

# 3. Generate code
kratos proto client api/user/v1/user.proto
kratos proto server api/user/v1/user.proto

# 4. Implement biz layer
# internal/biz/user.go

# 5. Implement data layer
# internal/data/user.go

# 6. Update wire
cd internal && wire

# 7. Run
cd .. && go run cmd/user-service/main.go
```

### Add New API Method

```bash
# 1. Update proto file
# Add new RPC to api/user/v1/user.proto

# 2. Regenerate code
kratos proto server api/user/v1/user.proto

# 3. Implement in service layer
# internal/service/userservice.go

# 4. Implement in biz layer (if needed)
# internal/biz/user.go

# 5. Regenerate wire (if new dependencies)
cd internal && wire
```

### Add New Entity

```bash
# 1. Create proto
cat > api/order/v1/order.proto << 'EOF'
syntax = "proto3";
package api.order.v1;
option go_package = "user-service/api/order/v1;v1";

service Order {
    rpc CreateOrder (CreateOrderRequest) returns (CreateOrderReply);
}

message CreateOrderRequest {
    int64 user_id = 1;
}

message CreateOrderReply {
    int64 order_id = 1;
}
EOF

# 2. Generate code
kratos proto client api/order/v1/order.proto
kratos proto server api/order/v1/order.proto

# 3. Create biz layer
# internal/biz/order.go

# 4. Create data layer
# internal/data/order.go

# 5. Update providers
# Add to internal/biz/biz.go
# Add to internal/data/data.go
# Add to internal/service/service.go

# 6. Regenerate wire
cd internal && wire
```

## Configuration

### Config File

```yaml
# configs/config.yaml
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
    source: user:password@tcp(127.0.0.1:3306)/test?parseTime=True&loc=Local
    max_idle_conns: 10
    max_open_conns: 100
    conn_max_lifetime: 1h
  redis:
    addr: 127.0.0.1:6379
    read_timeout: 0.2s
    write_timeout: 0.2s

auth:
  secret: your-secret-key
  expires: 7200s
```

### Config Structs

```go
// internal/conf/conf.go
package conf

import (
    "github.com/go-kratos/kratos/v2/config"
    "google.golang.org/protobuf/types/known/durationpb"
)

type Bootstrap struct {
    Server *Server
    Data   *Data
    Auth   *Auth
}

type Server struct {
    Http *HTTP
    Grpc *GRPC
}

type HTTP struct {
    Addr    string
    Timeout *durationpb.Duration
}

type GRPC struct {
    Addr    string
    Timeout *durationpb.Duration
}

type Data struct {
    Database *Database
    Redis    *Redis
}

type Database struct {
    Driver          string
    Source          string
    MaxIdleConns    int32
    MaxOpenConns    int32
    ConnMaxLifetime *durationpb.Duration
}

type Redis struct {
    Addr         string
    ReadTimeout  *durationpb.Duration
    WriteTimeout *durationpb.Duration
}

type Auth struct {
    Secret  string
    Expires *durationpb.Duration
}
```

## Troubleshooting

### Proto Generation Issues

```bash
# Check protoc version
protoc --version

# Check Go plugins
which protoc-gen-go
which protoc-gen-go-grpc
which protoc-gen-go-http

# Reinstall plugins if needed
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install github.com/go-kratos/kratos/cmd/protoc-gen-go-http/v2@latest
```

### Wire Issues

```bash
# Check wire installation
which wire

# Regenerate wire
cd internal && wire

# Clean and regenerate
rm internal/wire_gen.go
cd internal && wire
```

### Build Issues

```bash
# Clean cache
go clean -cache

# Tidy modules
go mod tidy

# Download dependencies
go mod download

# Verify dependencies
go mod verify
```

## Summary

### Common Commands

| Command | Description |
|---------|-------------|
| `kratos new <name>` | Create new project |
| `kratos proto client <proto>` | Generate proto client |
| `kratos proto server <proto>` | Generate proto server |
| `cd internal && wire` | Generate wire dependencies |
| `go run cmd/.../main.go` | Run application |
| `go build -o bin/...` | Build binary |
| `go test ./...` | Run tests |
| `go mod tidy` | Tidy modules |

### File Generation Order

1. Define `.proto` files
2. Run `kratos proto client`
3. Run `kratos proto server`
4. Implement biz layer
5. Implement data layer
6. Update provider sets
7. Run `wire` to generate dependencies
