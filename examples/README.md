# Examples

Example projects demonstrating go-kratos patterns.

## Available Examples

### 1. Hello World

Basic kratos service with HTTP and gRPC endpoints.

```bash
cd hello-world
kratos new .
```

### 2. User Service

Complete CRUD service with database integration.

```bash
cd user-service
# See README.md for setup instructions
```

### 3. Order Service

Service with multiple dependencies and event handling.

```bash
cd order-service
# See README.md for setup instructions
```

## Running Examples

### Prerequisites

```bash
# Install kratos CLI
go install github.com/go-kratos/kratos/cmd/kratos/v2@latest

# Install protocol buffer tools
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/protobuf/cmd/protoc-gen-go-grpc@latest
go install github.com/go-kratos/kratos/cmd/protoc-gen-go-http/v2@latest

# Add to PATH
export PATH=$PATH:$(go env GOPATH)/bin
```

### Run Example

```bash
# Navigate to example
cd hello-world

# Install dependencies
go mod tidy

# Generate code
kratos proto client api/helloworld/v1/helloworld.proto
cd internal && wire && cd ..

# Run
go run cmd/hello-world/main.go

# Test
curl http://localhost:8000/helloworld/foo
```

## Example Structure

Each example follows the standard kratos layout:

```
example/
├── api/                    # Protocol buffer definitions
├── cmd/                    # Application entry points
├── configs/                # Configuration files
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

## Contributing

To add a new example:

1. Create directory under `examples/`
2. Follow kratos best practices
3. Include README.md with setup instructions
4. Add to this list
