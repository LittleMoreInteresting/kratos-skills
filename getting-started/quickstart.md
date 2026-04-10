# go-kratos Quickstart for Agents

## 1) Bootstrap

```bash
# create project from template
kratos new helloworld
cd helloworld

# initialize dependencies/tools
make init

# generate protobuf & transport code
make api

# generate config bindings
make config

# generate wire dependency injection
make generate

# compile
make build
```

If the repository already exists, run only the missing steps.

## 2) kratos CLI command reference (with examples)

### Project and template

- `kratos new <project-name>`: scaffold a new kratos project.
- `kratos new <project-name> -r <repo-url>`: scaffold from custom template repo.

Example:

```bash
kratos new user-center
kratos new order-service -r https://github.com/go-kratos/kratos-layout
```

### Proto generation

- `kratos proto add <path/to/file.proto>`: create a proto file quickly.
- `kratos proto client <path/to/file.proto>`: generate client code for services.
- `kratos proto server <path/to/file.proto> -t internal/service`: generate service stub.

Example:

```bash
kratos proto add api/user/v1/user.proto
kratos proto client api/user/v1/user.proto
kratos proto server api/user/v1/user.proto -t internal/service
```

### Run and debug

- `kratos run`: run the current service in development mode.

Example:

```bash
kratos run
```

## 3) Typical directory conventions

- `api/` protobuf contracts and generated transport code
- `cmd/` entrypoints
- `internal/conf/` config structures
- `internal/service/` transport handlers
- `internal/biz/` use-cases and repo interfaces
- `internal/data/` repo implementations and data clients
- `internal/server/` HTTP and gRPC server setup

## 4) Make/go command cheatsheet

```bash
# regenerate API code (pb.go, grpc/http transport)
make api

# regenerate wire injection
make generate

# regenerate config structures
make config

# run tests
go test ./...

# static checks (if configured)
go vet ./...
```

## 5) End-to-end feature example

Add a new `CreateUser` endpoint in an existing service:

```bash
# 1) update contract
vim api/user/v1/user.proto

# 2) regenerate transport/client stubs
make api

# 3) optionally regenerate server/client stubs with kratos cli
kratos proto server api/user/v1/user.proto -t internal/service
kratos proto client api/user/v1/user.proto

# 4) implement biz + data + service
#    internal/biz/user.go
#    internal/data/user_repo.go
#    internal/service/user.go

# 5) refresh wire when constructor/provider changed
make generate

# 6) verify
go test ./...
make build
```

## 6) Delivery checklist

- API contract updated in `api/*.proto`
- generated code refreshed (`make api`, optional `kratos proto ...`)
- wire set updated when constructors change (`make generate`)
- tests pass
- configuration keys documented and mapped
