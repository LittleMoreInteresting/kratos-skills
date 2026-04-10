# Plan Architecture Pattern

This pattern demonstrates how to plan a microservices architecture using go-kratos.

## Purpose

Plan and design a complete microservices architecture:
- Service decomposition
- API design
- Data modeling
- Communication patterns
- Deployment strategy

## When to Use

- Greenfield projects
- System redesign
- Adding major features
- Architecture reviews

## Implementation

### Step 1: Define Requirements

```yaml
# requirements.yaml
project:
  name: e-commerce-platform
  description: Online shopping platform
  
domains:
  - name: user-management
    description: User registration, authentication, profiles
    
  - name: product-catalog
    description: Product management, search, categories
    
  - name: order-management
    description: Order processing, history, tracking
    
  - name: inventory-management
    description: Stock management, reservations
    
  - name: payment-processing
    description: Payment handling, refunds
    
  - name: notification-service
    description: Email, SMS, push notifications

requirements:
  functional:
    - Users can register and login
    - Users can browse products
    - Users can place orders
    - Users can track orders
    - Admins can manage products
    - Admins can manage orders
    
  non_functional:
    - 99.9% uptime
    - < 200ms response time (p95)
    - 10,000 concurrent users
    - GDPR compliant
    - PCI DSS compliant (payment)
```

### Step 2: Service Decomposition

```yaml
# services.yaml
services:
  - name: user-service
    domain: user-management
    responsibilities:
      - User registration
      - User authentication
      - User profile management
      - Password reset
    
    apis:
      - name: RegisterUser
        method: POST
        path: /v1/users
        
      - name: LoginUser
        method: POST
        path: /v1/users/login
        
      - name: GetUser
        method: GET
        path: /v1/users/{id}
        
      - name: UpdateUser
        method: PUT
        path: /v1/users/{id}
        
      - name: DeleteUser
        method: DELETE
        path: /v1/users/{id}
    
    database:
      type: mysql
      entities:
        - User
        - UserProfile
        - PasswordResetToken
    
    dependencies: []
    
  - name: product-service
    domain: product-catalog
    responsibilities:
      - Product CRUD
      - Category management
      - Product search
      - Inventory tracking
    
    apis:
      - name: CreateProduct
        method: POST
        path: /v1/products
        
      - name: GetProduct
        method: GET
        path: /v1/products/{id}
        
      - name: ListProducts
        method: GET
        path: /v1/products
        
      - name: SearchProducts
        method: GET
        path: /v1/products/search
        
      - name: UpdateProduct
        method: PUT
        path: /v1/products/{id}
        
      - name: DeleteProduct
        method: DELETE
        path: /v1/products/{id}
    
    database:
      type: mysql
      entities:
        - Product
        - Category
        - ProductImage
    
    dependencies:
      - inventory-service
      
  - name: order-service
    domain: order-management
    responsibilities:
      - Order creation
      - Order tracking
      - Order history
      - Order cancellation
    
    apis:
      - name: CreateOrder
        method: POST
        path: /v1/orders
        
      - name: GetOrder
        method: GET
        path: /v1/orders/{id}
        
      - name: ListOrders
        method: GET
        path: /v1/orders
        
      - name: UpdateOrderStatus
        method: PUT
        path: /v1/orders/{id}/status
        
      - name: CancelOrder
        method: POST
        path: /v1/orders/{id}/cancel
    
    database:
      type: mysql
      entities:
        - Order
        - OrderItem
        - OrderStatusHistory
    
    dependencies:
      - user-service
      - product-service
      - inventory-service
      - payment-service
      
  - name: inventory-service
    domain: inventory-management
    responsibilities:
      - Stock tracking
      - Inventory reservation
      - Stock alerts
    
    apis:
      - name: GetStock
        method: GET
        path: /v1/inventory/{product_id}
        
      - name: ReserveStock
        method: POST
        path: /v1/inventory/reserve
        
      - name: ReleaseStock
        method: POST
        path: /v1/inventory/release
        
      - name: UpdateStock
        method: PUT
        path: /v1/inventory/{product_id}
    
    database:
      type: redis
      entities:
        - Stock
        - Reservation
        
    dependencies: []
    
  - name: payment-service
    domain: payment-processing
    responsibilities:
      - Payment processing
      - Refund handling
      - Payment method management
    
    apis:
      - name: ProcessPayment
        method: POST
        path: /v1/payments
        
      - name: GetPayment
        method: GET
        path: /v1/payments/{id}
        
      - name: RefundPayment
        method: POST
        path: /v1/payments/{id}/refund
    
    database:
      type: mysql
      entities:
        - Payment
        - PaymentMethod
        - Refund
    
    dependencies:
      - user-service
      
  - name: notification-service
    domain: notification
    responsibilities:
      - Email notifications
      - SMS notifications
      - Push notifications
    
    apis:
      - name: SendEmail
        method: POST
        path: /v1/notifications/email
        
      - name: SendSMS
        method: POST
        path: /v1/notifications/sms
        
      - name: SendPush
        method: POST
        path: /v1/notifications/push
    
    database:
      type: mongodb
      entities:
        - Notification
        - NotificationTemplate
    
    dependencies:
      - user-service
```

### Step 3: Define Communication Patterns

```yaml
# communication.yaml
patterns:
  synchronous:
    - name: user-to-order
      from: order-service
      to: user-service
      protocol: gRPC
      purpose: Validate user, get user info
      
    - name: order-to-product
      from: order-service
      to: product-service
      protocol: gRPC
      purpose: Get product details, validate prices
      
    - name: order-to-inventory
      from: order-service
      to: inventory-service
      protocol: gRPC
      purpose: Reserve stock
      
    - name: order-to-payment
      from: order-service
      to: payment-service
      protocol: gRPC
      purpose: Process payment
      
  asynchronous:
    - name: order-created
      producer: order-service
      consumers:
        - notification-service
        - inventory-service
      event: OrderCreated
      purpose: Notify user, update inventory
      
    - name: payment-completed
      producer: payment-service
      consumers:
        - order-service
        - notification-service
      event: PaymentCompleted
      purpose: Update order status, send receipt
      
    - name: inventory-low
      producer: inventory-service
      consumers:
        - notification-service
      event: InventoryLow
      purpose: Alert admin
```

### Step 4: Data Model

```yaml
# data-model.yaml
entities:
  User:
    fields:
      - id: int64
      - email: string
      - password_hash: string
      - name: string
      - phone: string
      - status: enum [active, inactive, suspended]
      - created_at: timestamp
      - updated_at: timestamp
      
  Product:
    fields:
      - id: int64
      - name: string
      - description: string
      - price: decimal
      - category_id: int64
      - status: enum [active, inactive]
      - created_at: timestamp
      - updated_at: timestamp
      
  Order:
    fields:
      - id: int64
      - user_id: int64
      - status: enum [pending, confirmed, paid, shipped, delivered, cancelled]
      - total_amount: decimal
      - shipping_address: json
      - created_at: timestamp
      - updated_at: timestamp
      
  OrderItem:
    fields:
      - id: int64
      - order_id: int64
      - product_id: int64
      - quantity: int
      - unit_price: decimal
      - total_price: decimal
      
  Inventory:
    fields:
      - product_id: int64
      - quantity: int
      - reserved: int
      - updated_at: timestamp
```

### Step 5: Generate Architecture Document

```go
// planner.go
package main

import (
    "fmt"
    "os"
    "text/template"
)

const architectureTemplate = `# {{.Project.Name}} Architecture

## Overview
{{.Project.Description}}

## Services

{{range .Services}}
### {{.Name}}
**Domain**: {{.Domain}}

**Responsibilities**:
{{range .Responsibilities}}
- {{.}}
{{end}}

**APIs**:
{{range .APIs}}
- {{.Method}} {{.Path}} - {{.Name}}
{{end}}

**Database**:
- Type: {{.Database.Type}}
- Entities: {{range .Database.Entities}}{{.}}, {{end}}

**Dependencies**:
{{range .Dependencies}}
- {{.}}
{{else}}
None
{{end}}

{{end}}

## Communication Patterns

### Synchronous
{{range .Patterns.Synchronous}}
- **{{.From}} → {{.To}}**
  - Protocol: {{.Protocol}}
  - Purpose: {{.Purpose}}
{{end}}

### Asynchronous
{{range .Patterns.Asynchronous}}
- **{{.Event}}**
  - Producer: {{.Producer}}
  - Consumers: {{range .Consumers}}{{.}}, {{end}}
  - Purpose: {{.Purpose}}
{{end}}

## Data Model

{{range .Entities}}
### {{.Name}}
{{range .Fields}}
- {{.Name}}: {{.Type}}
{{end}}

{{end}}

## Deployment

### Docker Compose
\`\`\`yaml
version: '3.8'
services:
{{range .Services}}
  {{.Name}}:
    build: ./{{.Name}}
    ports:
      - "{{.Port}}:8000"
    environment:
      - DATABASE_URL=mysql://...
{{end}}
\`\`\`

### Kubernetes
See k8s/ directory for Kubernetes manifests.
`

type ArchitecturePlanner struct {
    Project    Project
    Services   []Service
    Patterns   CommunicationPatterns
    Entities   []Entity
}

func (p *ArchitecturePlanner) Generate(outputPath string) error {
    tmpl, err := template.New("architecture").Parse(architectureTemplate)
    if err != nil {
        return err
    }
    
    file, err := os.Create(outputPath)
    if err != nil {
        return err
    }
    defer file.Close()
    
    return tmpl.Execute(file, p)
}
```

## Usage

### Command Line

```bash
# Generate architecture document
go run planner.go -spec services.yaml -output ARCHITECTURE.md
```

### In Claude Code

```
You: Plan architecture for an e-commerce platform

Claude: I'll plan a microservices architecture for your e-commerce platform...

## Analysis

Based on requirements, I'll design 6 services:

1. **User Service** - Authentication & profiles
2. **Product Service** - Product catalog
3. **Order Service** - Order management
4. **Inventory Service** - Stock management
5. **Payment Service** - Payment processing
6. **Notification Service** - Email & SMS

## Service Communication

- Synchronous: gRPC between services
- Asynchronous: Events for notifications

## Data Storage

- MySQL: User, Product, Order, Payment data
- Redis: Inventory, sessions
- MongoDB: Notifications

## API Gateway

All requests go through API Gateway for:
- Authentication
- Rate limiting
- Request routing

Would you like me to generate the complete architecture document?
```

## Output Example

```markdown
# E-Commerce Platform Architecture

## Overview
Online shopping platform with microservices architecture.

## Services

### User Service
**Domain**: User Management

**Responsibilities**:
- User registration
- User authentication
- User profile management
- Password reset

**APIs**:
- POST /v1/users - RegisterUser
- POST /v1/users/login - LoginUser
- GET /v1/users/{id} - GetUser

**Database**:
- Type: MySQL
- Entities: User, UserProfile

**Dependencies**: None

### Order Service
**Domain**: Order Management

**Responsibilities**:
- Order creation
- Order tracking
- Order history

**APIs**:
- POST /v1/orders - CreateOrder
- GET /v1/orders/{id} - GetOrder

**Database**:
- Type: MySQL
- Entities: Order, OrderItem

**Dependencies**:
- user-service
- product-service
- inventory-service
- payment-service

## Communication Patterns

### Synchronous
- **order-service → user-service**
  - Protocol: gRPC
  - Purpose: Validate user

### Asynchronous
- **OrderCreated**
  - Producer: order-service
  - Consumers: notification-service, inventory-service
  - Purpose: Notify user, update inventory
```
