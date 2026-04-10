# Generate Service Pattern

This pattern demonstrates how to generate a complete go-kratos service.

## Purpose

Generate a production-ready service with all layers:
- Protocol buffer definitions
- Service layer (HTTP/gRPC handlers)
- Biz layer (business logic)
- Data layer (repository)
- Wire configuration

## When to Use

- Creating new microservices
- Adding new entities to existing project
- Rapid prototyping

## Implementation

### Step 1: Define Service Specification

```yaml
# service-spec.yaml
name: order-service
entities:
  - name: Order
    fields:
      - name: id
        type: int64
        primary: true
      - name: user_id
        type: int64
        required: true
      - name: items
        type: repeated OrderItem
      - name: total_amount
        type: double
      - name: status
        type: string
        enum: [pending, confirmed, shipped, delivered, cancelled]
      - name: created_at
        type: timestamp
      - name: updated_at
        type: timestamp

operations:
  - name: CreateOrder
    input: CreateOrderRequest
    output: CreateOrderReply
  - name: GetOrder
    input: GetOrderRequest
    output: GetOrderReply
  - name: ListOrders
    input: ListOrdersRequest
    output: ListOrdersReply
  - name: UpdateOrderStatus
    input: UpdateOrderStatusRequest
    output: UpdateOrderStatusReply
```

### Step 2: Generate Proto File

```go
// generator/proto.go
package generator

import (
    "fmt"
    "os"
    "strings"
    "text/template"
)

const protoTemplate = `syntax = "proto3";

package api.{{.Name}}.v1;

option go_package = "{{.ProjectName}}/api/{{.Name}}/v1;v1";

import "google/api/annotations.proto";
import "google/protobuf/timestamp.proto";
import "validate/validate.proto";

service {{.ServiceName}} {
{{range .Operations}}
    rpc {{.Name}} ({{.Input}}) returns ({{.Output}}) {
        option (google.api.http) = {
            {{.Method}}: "/v1/{{.Path}}"
            {{if .Body}}body: "{{.Body}}"{{end}}
        };
    }
{{end}}
}

{{range .Messages}}
message {{.Name}} {
{{range .Fields}}
    {{.Type}} {{.Name}} = {{.Number}}{{if .Validation}} [(validate.rules).{{.Validation}}]{{end}};
{{end}}
}
{{end}}
`

type ProtoGenerator struct {
    ProjectName string
    Name        string
    ServiceName string
    Operations  []Operation
    Messages    []Message
}

type Operation struct {
    Name   string
    Input  string
    Output string
    Method string
    Path   string
    Body   string
}

type Message struct {
    Name   string
    Fields []Field
}

type Field struct {
    Number     int
    Type       string
    Name       string
    Validation string
}

func (g *ProtoGenerator) Generate(outputPath string) error {
    tmpl, err := template.New("proto").Parse(protoTemplate)
    if err != nil {
        return err
    }
    
    file, err := os.Create(outputPath)
    if err != nil {
        return err
    }
    defer file.Close()
    
    return tmpl.Execute(file, g)
}
```

### Step 3: Generate Service Layer

```go
// generator/service.go
package generator

const serviceTemplate = `package service

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/log"
    pb "{{.ProjectName}}/api/{{.Name}}/v1"
    "{{.ProjectName}}/internal/biz"
)

type {{.ServiceName}} struct {
    pb.Unimplemented{{.ServiceName}}Server
    uc  *biz.{{.EntityName}}Usecase
    log *log.Helper
}

func New{{.ServiceName}}(uc *biz.{{.EntityName}}Usecase, logger log.Logger) *{{.ServiceName}} {
    return &{{.ServiceName}}{
        uc:  uc,
        log: log.NewHelper(logger),
    }
}

{{range .Operations}}
func (s *{{.ServiceName}}) {{.Name}}(ctx context.Context, req *pb.{{.Input}}) (*pb.{{.Output}}, error) {
    s.log.Infof("{{.Name}} called")
    
    {{if eq .Name "Create"}}
    // Convert request to biz model
    entity := &biz.{{.EntityName}}{
        // Map fields from req
    }
    
    result, err := s.uc.Create(ctx, entity)
    if err != nil {
        return nil, err
    }
    
    return &pb.{{.Output}}{
        // Map fields from result
    }, nil
    {{else if eq .Name "Get"}}
    result, err := s.uc.Get(ctx, req.Id)
    if err != nil {
        return nil, err
    }
    
    return &pb.{{.Output}}{
        // Map fields from result
    }, nil
    {{else}}
    // TODO: Implement {{.Name}}
    return &pb.{{.Output}}{}, nil
    {{end}}
}
{{end}}
`

func (g *ServiceGenerator) Generate(outputPath string) error {
    // Similar to proto generator
    return nil
}
```

### Step 4: Generate Biz Layer

```go
// generator/biz.go
package generator

const bizTemplate = `package biz

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/log"
)

// {{.EntityName}} is the domain model
type {{.EntityName}} struct {
{{range .Fields}}
    {{.Name}} {{.GoType}}
{{end}}
}

// Validate performs domain validation
func (e *{{.EntityName}}) Validate() error {
    // TODO: Add validation logic
    return nil
}

// {{.EntityName}}Repo is the repository interface
type {{.EntityName}}Repo interface {
    Create(ctx context.Context, e *{{.EntityName}}) error
    Get(ctx context.Context, id int64) (*{{.EntityName}}, error)
    Update(ctx context.Context, e *{{.EntityName}}) error
    Delete(ctx context.Context, id int64) error
    List(ctx context.Context, page, pageSize int32) ([]*{{.EntityName}}, int32, error)
}

// {{.EntityName}}Usecase handles business logic
type {{.EntityName}}Usecase struct {
    repo {{.EntityName}}Repo
    log  *log.Helper
}

func New{{.EntityName}}Usecase(repo {{.EntityName}}Repo, logger log.Logger) *{{.EntityName}}Usecase {
    return &{{.EntityName}}Usecase{
        repo: repo,
        log:  log.NewHelper(logger),
    }
}

func (uc *{{.EntityName}}Usecase) Create(ctx context.Context, e *{{.EntityName}}) (*{{.EntityName}}, error) {
    if err := e.Validate(); err != nil {
        return nil, err
    }
    
    e.CreatedAt = time.Now()
    e.UpdatedAt = time.Now()
    
    if err := uc.repo.Create(ctx, e); err != nil {
        return nil, err
    }
    
    return e, nil
}

func (uc *{{.EntityName}}Usecase) Get(ctx context.Context, id int64) (*{{.EntityName}}, error) {
    e, err := uc.repo.Get(ctx, id)
    if err != nil {
        if errors.IsNotFound(err) {
            return nil, errors.NotFound("{{.EntityName | upper}}_NOT_FOUND", "{{.EntityName | lower}} not found")
        }
        return nil, err
    }
    return e, nil
}

// TODO: Implement other methods
`
```

### Step 5: Generate Data Layer

```go
// generator/data.go
package generator

const dataTemplate = `package data

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/log"
    "gorm.io/gorm"
    
    "{{.ProjectName}}/internal/biz"
)

type {{.EntityName | lower}}Repo struct {
    data *Data
    log  *log.Helper
}

func New{{.EntityName}}Repo(data *Data, logger log.Logger) biz.{{.EntityName}}Repo {
    return &{{.EntityName | lower}}Repo{
        data: data,
        log:  log.NewHelper(logger),
    }
}

func (r *{{.EntityName | lower}}Repo) Create(ctx context.Context, e *biz.{{.EntityName}}) error {
    po := &{{.EntityName}}PO{
{{range .Fields}}
        {{.Name}}: e.{{.Name}},
{{end}}
    }
    
    result := r.data.db.WithContext(ctx).Create(po)
    if result.Error != nil {
        return result.Error
    }
    
    e.ID = po.ID
    return nil
}

func (r *{{.EntityName | lower}}Repo) Get(ctx context.Context, id int64) (*biz.{{.EntityName}}, error) {
    var po {{.EntityName}}PO
    result := r.data.db.WithContext(ctx).First(&po, id)
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrRecordNotFound) {
            return nil, errors.NotFound("{{.EntityName | upper}}_NOT_FOUND", "{{.EntityName | lower}} not found")
        }
        return nil, result.Error
    }
    
    return po.ToBiz(), nil
}

// {{.EntityName}}PO is the persistence object
type {{.EntityName}}PO struct {
{{range .Fields}}
    {{.Name}} {{.DBType}} {{.DBTag}}
{{end}}
}

func (po *{{.EntityName}}PO) TableName() string {
    return "{{.EntityName | lower}}s"
}

func (po *{{.EntityName}}PO) ToBiz() *biz.{{.EntityName}} {
    return &biz.{{.EntityName}}{
{{range .Fields}}
        {{.Name}}: po.{{.Name}},
{{end}}
    }
}
`
```

## Usage

### Command Line

```bash
# Generate service from spec
go run generator.go -spec service-spec.yaml -output ./

# Or use kratos CLI with custom template
kratos proto server api/order/v1/order.proto
```

### In Claude Code

```
You: Generate an order service with CRUD operations

Claude: I'll generate a complete order service...

1. Creating proto definition
2. Generating service layer
3. Generating biz layer
4. Generating data layer
5. Updating wire configuration

✅ Order service generated successfully!
```

## Output Structure

```
api/order/v1/
├── order.proto
├── order.pb.go
├── order_grpc.pb.go
└── order_http.pb.go

internal/
├── biz/
│   └── order.go
├── data/
│   └── order.go
└── service/
    └── orderservice.go
```
