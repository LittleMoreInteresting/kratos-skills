# Validation Patterns (go-kratos)

This guide covers input validation patterns in go-kratos using `protoc-gen-validate` (PGV).

## Overview

- Use `protoc-gen-validate` for declarative validation rules
- Enable `validate.Validator()` middleware for automatic validation
- Handle validation errors appropriately in service layer

## Setup

### 1. Install protoc-gen-validate

```bash
# Install the protoc plugin
go install github.com/envoyproxy/protoc-gen-validate@latest

# Add to your go.mod
go get github.com/envoyproxy/protoc-gen-validate/validate
```

### 2. Update Makefile

```makefile
# Add validate proto path
API_PROTO_FILES=$(shell find api -name *.proto)
PROTOC_VALIDATE=$(GOPATH)/pkg/mod/github.com/envoyproxy/protoc-gen-validate@v0.6.13

.PHONY: api
api:
	protoc --proto_path=. \
	       --proto_path=$(PROTOC_VALIDATE) \
	       --go_out=paths=source_relative:. \
	       --go-grpc_out=paths=source_relative:. \
	       --go-http_out=paths=source_relative:. \
	       --validate_out=paths=source_relative,lang=go:. \
	       $(API_PROTO_FILES)
```

---

## Validation Rules

### Numeric Types

```protobuf
syntax = "proto3";

import "validate/validate.proto";

message NumericRequest {
    // Integer validation
    int64 id = 1 [(validate.rules).int64 = {gt: 0}];           // > 0
    int32 age = 2 [(validate.rules).int32 = {gt: 0, lt: 120}]; // 1-119
    
    // Unsigned integer
    uint32 code = 3 [(validate.rules).uint32 = {in: [1, 2, 3]}]; // Must be 1, 2, or 3
    uint64 size = 4 [(validate.rules).uint64 = {lte: 10000}];    // <= 10000
    
    // Float/double
    float score = 5 [(validate.rules).float = {gte: 0, lte: 100}];
    double price = 6 [(validate.rules).double = {not_in: [0, -1]}]; // Cannot be 0 or -1
    
    // Constant value
    bool active = 7 [(validate.rules).bool.const = true]; // Must be true
}
```

### String Validation

```protobuf
message StringRequest {
    // Length validation
    string phone = 1 [(validate.rules).string.len = 11];           // Exact length
    string name = 2 [(validate.rules).string = {min_len: 1, max_len: 50}];
    string description = 3 [(validate.rules).string.min_bytes = 10];
    
    // Pattern validation (regex)
    string email = 4 [(validate.rules).string.email = true];
    string ipv4 = 5 [(validate.rules).string.ipv4 = true];
    string ipv6 = 6 [(validate.rules).string.ipv6 = true];
    string uri = 7 [(validate.rules).string.uri = true];
    string uuid = 8 [(validate.rules).string.uuid = true];
    
    // Custom pattern
    string username = 9 [(validate.rules).string.pattern = "^[a-zA-Z0-9_]+$"];
    string hex = 10 [(validate.rules).string.pattern = "(?i)^[0-9a-f]+$"];
    
    // Prefix/suffix/contains
    string path = 11 [(validate.rules).string.prefix = "/api/v1"];
    string file = 12 [(validate.rules).string.suffix = ".txt"];
    string content = 13 [(validate.rules).string.contains = "required"];
    
    // Constant value
    string version = 14 [(validate.rules).string.const = "v1"];
    
    // Well-known formats
    string timestamp = 15 [(validate.rules).string.timestamp = true];
    string duration = 16 [(validate.rules).string.duration = true];
}
```

### Enum Validation

```protobuf
enum Status {
    STATUS_UNSPECIFIED = 0;
    STATUS_ACTIVE = 1;
    STATUS_INACTIVE = 2;
    STATUS_PENDING = 3;
}

message EnumRequest {
    // Must be defined (not UNSPECIFIED)
    Status status = 1 [(validate.rules).enum.defined_only = true];
    
    // Must be specific value
    Status required_status = 2 [(validate.rules).enum.const = 1]; // Must be ACTIVE
    
    // Must be in allowed values
    Status allowed_status = 3 [(validate.rules).enum = {in: [1, 2]}]; // ACTIVE or INACTIVE
}
```

### Message Validation

```protobuf
message Address {
    string street = 1 [(validate.rules).string.min_len = 1];
    string city = 2 [(validate.rules).string.min_len = 1];
    string zip = 3 [(validate.rules).string.pattern = "^\\d{5}$"];
}

message UserRequest {
    // Required message field
    Address address = 1 [(validate.rules).message.required = true];
    
    // Skip validation for this field
    Address optional_address = 2 [(validate.rules).message.skip = true];
}
```

### Repeated Fields (Lists)

```protobuf
message ListRequest {
    // List size
    repeated string tags = 1 [(validate.rules).repeated = {min_items: 1, max_items: 10}];
    
    // Unique items
    repeated int64 ids = 2 [(validate.rules).repeated.unique = true];
    
    // Validate each item
    repeated string emails = 3 [(validate.rules).repeated = {
        min_items: 1,
        unique: true,
        items: {string: {email: true}}
    }];
    
    // Validate nested messages
    repeated Address addresses = 4 [(validate.rules).repeated = {
        min_items: 1,
        items: {message: {required: true}}
    }];
}
```

### Map Validation

```protobuf
message MapRequest {
    // Map size
    map<string, string> metadata = 1 [(validate.rules).map = {min_pairs: 1, max_pairs: 10}];
    
    // Validate keys and values
    map<string, int32> scores = 2 [(validate.rules).map = {
        keys: {string: {min_len: 1}},
        values: {int32: {gte: 0, lte: 100}}
    }];
}
```

### Oneof Validation

```protobuf
message OneofRequest {
    oneof contact {
        option (validate.required) = true; // At least one must be set
        
        string email = 1 [(validate.rules).string.email = true];
        string phone = 2 [(validate.rules).string.pattern = "^\\d{11}$"];
    }
}
```

### Any Validation

```protobuf
import "google/protobuf/any.proto";

message AnyRequest {
    google.protobuf.Any details = 1 [(validate.rules).any.required = true];
    
    google.protobuf.Any type_details = 2 [(validate.rules).any = {
        required: true,
        in: ["type.googleapis.com/google.protobuf.Duration"]
    }];
}
```

---

## Complete API Example

```protobuf
// api/user/v1/user.proto
syntax = "proto3";

package api.user.v1;

option go_package = "user-service/api/user/v1;v1";

import "validate/validate.proto";
import "google/api/annotations.proto";
import "google/protobuf/timestamp.proto";

service UserService {
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
    string name = 1 [(validate.rules).string = {
        min_len: 1,
        max_len: 50,
        pattern: "^[a-zA-Z0-9_\\s]+$"
    }];
    
    string email = 2 [(validate.rules).string.email = true];
    
    int32 age = 3 [(validate.rules).int32 = {
        gte: 18,
        lte: 120
    }];
    
    string password = 4 [(validate.rules).string = {
        min_len: 8,
        max_len: 32,
        pattern: "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).+$"
    }];
    
    repeated string tags = 5 [(validate.rules).repeated = {
        max_items: 10,
        items: {string: {min_len: 1, max_len: 20}}
    }];
}

message CreateUserReply {
    int64 id = 1;
    google.protobuf.Timestamp created_at = 2;
}

message GetUserRequest {
    int64 id = 1 [(validate.rules).int64 = {gt: 0}];
}

message GetUserReply {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    repeated string tags = 5;
    google.protobuf.Timestamp created_at = 6;
    google.protobuf.Timestamp updated_at = 7;
}

message UpdateUserRequest {
    int64 id = 1 [(validate.rules).int64 = {gt: 0}];
    
    string name = 2 [(validate.rules).string = {
        min_len: 1,
        max_len: 50,
        ignore_empty: true  // Allow empty to skip update
    }];
    
    int32 age = 3 [(validate.rules).int32 = {
        gte: 18,
        lte: 120,
        ignore_empty: true
    }];
}

message UpdateUserReply {
    int64 id = 1;
    google.protobuf.Timestamp updated_at = 2;
}

message DeleteUserRequest {
    int64 id = 1 [(validate.rules).int64 = {gt: 0}];
}

message DeleteUserReply {
    bool success = 1;
}

message ListUsersRequest {
    int32 page = 1 [(validate.rules).int32 = {gte: 1}];
    int32 page_size = 2 [(validate.rules).int32 = {
        gte: 1,
        lte: 100
    }];
    string sort_by = 3 [(validate.rules).string = {
        in: ["id", "name", "created_at"],
        ignore_empty: true
    }];
}

message ListUsersReply {
    repeated User users = 1;
    int32 total = 2;
    int32 page = 3;
    int32 page_size = 4;
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    repeated string tags = 5;
}
```

---

## Server Configuration

### Enable Validation Middleware

```go
// internal/server/http.go
package server

import (
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/validate"
    "github.com/go-kratos/kratos/v2/transport/http"
    
    v1 "user-service/api/user/v1"
    "user-service/internal/conf"
    "user-service/internal/service"
)

func NewHTTPServer(c *conf.Server, user *service.UserService, logger log.Logger) *http.Server {
    var opts = []http.ServerOption{
        http.Middleware(
            recovery.Recovery(),
            validate.Validator(),  // Enable validation
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

```go
// internal/server/grpc.go
package server

import (
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/validate"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    
    v1 "user-service/api/user/v1"
    "user-service/internal/conf"
    "user-service/internal/service"
)

func NewGRPCServer(c *conf.Server, user *service.UserService, logger log.Logger) *grpc.Server {
    var opts = []grpc.ServerOption{
        grpc.Middleware(
            recovery.Recovery(),
            validate.Validator(),  // Enable validation
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

## Custom Validation

### Programmatic Validation

```go
// internal/biz/user.go
package biz

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/errors"
)

func (uc *UserUsecase) CreateUser(ctx context.Context, u *User) error {
    // Custom business validation
    if err := uc.validateUser(u); err != nil {
        return err
    }
    
    return uc.repo.Create(ctx, u)
}

func (uc *UserUsecase) validateUser(u *User) error {
    // Check reserved usernames
    reserved := []string{"admin", "root", "system"}
    for _, r := range reserved {
        if u.Name == r {
            return errors.BadRequest("RESERVED_USERNAME", "username is reserved")
        }
    }
    
    // Check email domain
    allowedDomains := []string{"@company.com", "@partner.com"}
    hasAllowedDomain := false
    for _, domain := range allowedDomains {
        if strings.HasSuffix(u.Email, domain) {
            hasAllowedDomain = true
            break
        }
    }
    if !hasAllowedDomain {
        return errors.BadRequest("INVALID_EMAIL_DOMAIN", "email domain not allowed")
    }
    
    return nil
}
```

### Custom Proto Validator

```go
// internal/validator/custom.go
package validator

import (
    "context"
    "fmt"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/go-playground/validator/v10"
)

var validate = validator.New()

func init() {
    // Register custom validation
    validate.RegisterValidation("phone", validatePhone)
}

func validatePhone(fl validator.FieldLevel) bool {
    phone := fl.Field().String()
    // Validate phone number format
    return len(phone) == 11 && phone[0] == '1'
}

// CustomValidator middleware
func CustomValidator() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            if err := validate.Struct(req); err != nil {
                return nil, errors.BadRequest("VALIDATION_ERROR", err.Error())
            }
            return handler(ctx, req)
        }
    }
}
```

---

## Validation Error Handling

### Default Error Response

When validation fails, kratos returns:

```json
{
  "code": 400,
  "reason": "VALIDATION_ERROR",
  "message": "invalid field SomeField: ...",
  "metadata": {}
}
```

### Custom Validation Error Handler

```go
// internal/server/http.go
package server

import (
    stdhttp "net/http"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/transport/http"
)

func validationErrorEncoder(w stdhttp.ResponseWriter, r *stdhttp.Request, err error) {
    se := errors.FromError(err)
    
    // Check if it's a validation error
    if se.Reason == "VALIDATION_ERROR" {
        codec, _ := http.CodecForRequest(r, "Accept")
        
        body := map[string]interface{}{
            "code":    400,
            "message": "validation failed",
            "errors": []map[string]string{
                {
                    "field":   se.Metadata["field"],
                    "message": se.Message,
                },
            },
        }
        
        data, _ := codec.Marshal(body)
        w.Header().Set("Content-Type", "application/"+codec.Name())
        w.WriteHeader(400)
        w.Write(data)
        return
    }
    
    // Default error encoding
    http.DefaultErrorEncoder(w, r, err)
}
```

---

## Validation Rules Reference

### Numeric Rules

| Rule | Type | Description |
|------|------|-------------|
| `const` | number | Exact value |
| `lt`/`lte` | number | Less than / less than or equal |
| `gt`/`gte` | number | Greater than / greater than or equal |
| `in` | number[] | Must be one of the values |
| `not_in` | number[] | Must not be any of the values |

### String Rules

| Rule | Type | Description |
|------|------|-------------|
| `const` | string | Exact value |
| `len`/`min_len`/`max_len` | int | Length constraints |
| `len_bytes`/`min_bytes`/`max_bytes` | int | Byte length constraints |
| `pattern` | string | Regex pattern |
| `prefix`/`suffix` | string | Prefix/suffix requirements |
| `contains` | string | Must contain substring |
| `not_contains` | string | Must not contain substring |
| `email` | bool | Valid email format |
| `hostname` | bool | Valid hostname |
| `ip`/`ipv4`/`ipv6` | bool | Valid IP address |
| `uri`/`uri_ref` | bool | Valid URI |
| `uuid` | bool | Valid UUID |
| `timestamp` | bool | Valid timestamp |
| `duration` | bool | Valid duration |

### Message Rules

| Rule | Type | Description |
|------|------|-------------|
| `required` | bool | Message must be set |
| `skip` | bool | Skip validation for this field |

### Repeated Rules

| Rule | Type | Description |
|------|------|-------------|
| `min_items`/`max_items` | int | Size constraints |
| `unique` | bool | All items must be unique |
| `items` | rules | Rules for each item |

### Map Rules

| Rule | Type | Description |
|------|------|-------------|
| `min_pairs`/`max_pairs` | int | Size constraints |
| `keys` | rules | Rules for keys |
| `values` | rules | Rules for values |

---

## Best Practices

### ✅ Always Follow

- Define validation rules in proto files
- Use semantic validation (email, uuid, etc.) when possible
- Set reasonable limits (string length, list size)
- Enable validation middleware
- Handle validation errors gracefully
- Validate at both proto and business layers

### ❌ Never Do

```protobuf
// DON'T: Skip validation
message Request {
    string email = 1;  // ❌ No validation
}

// DON'T: Use overly restrictive rules
message Request {
    string name = 1 [(validate.rules).string.const = "John"];  // ❌ Too restrictive
}

// DON'T: Forget to enable middleware
// ❌ Without validate.Validator(), proto rules won't be checked

// DON'T: Duplicate validation in code
// ❌ Proto already validates, don't duplicate in Go code
```

---

## Testing Validation

```go
// internal/service/userservice_test.go
package service

import (
    "context"
    "testing"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/stretchr/testify/assert"
    v1 "user-service/api/user/v1"
)

func TestUserService_CreateUser_Validation(t *testing.T) {
    svc := NewUserService(mockUC, log.DefaultLogger)
    
    tests := []struct {
        name    string
        req     *v1.CreateUserRequest
        wantErr bool
        errCode int
    }{
        {
            name: "valid request",
            req: &v1.CreateUserRequest{
                Name:     "John Doe",
                Email:    "john@example.com",
                Age:      25,
                Password: "SecurePass123",
            },
            wantErr: false,
        },
        {
            name: "invalid email",
            req: &v1.CreateUserRequest{
                Name:     "John",
                Email:    "invalid-email",
                Age:      25,
                Password: "SecurePass123",
            },
            wantErr: true,
            errCode: 400,
        },
        {
            name: "age too young",
            req: &v1.CreateUserRequest{
                Name:     "John",
                Email:    "john@example.com",
                Age:      16,
                Password: "SecurePass123",
            },
            wantErr: true,
            errCode: 400,
        },
        {
            name: "password too weak",
            req: &v1.CreateUserRequest{
                Name:     "John",
                Email:    "john@example.com",
                Age:      25,
                Password: "weak",
            },
            wantErr: true,
            errCode: 400,
        },
        {
            name: "name too long",
            req: &v1.CreateUserRequest{
                Name:     string(make([]byte, 51)),
                Email:    "john@example.com",
                Age:      25,
                Password: "SecurePass123",
            },
            wantErr: true,
            errCode: 400,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := svc.CreateUser(context.Background(), tt.req)
            if tt.wantErr {
                assert.Error(t, err)
                if tt.errCode > 0 {
                    assert.Equal(t, tt.errCode, errors.Code(err))
                }
            } else {
                assert.NoError(t, err)
            }
        })
    }
}
```
