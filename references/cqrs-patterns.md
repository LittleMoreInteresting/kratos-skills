# CQRS Patterns (go-kratos)

This guide covers Command Query Responsibility Segregation (CQRS) patterns in go-kratos.

## Overview

CQRS separates read and write operations into different models:

- **Command Model**: Handles writes, complex business logic, data validation
- **Query Model**: Handles reads, optimized for query performance
- **Event Bus**: Connects command and query sides through events

```
┌─────────────────┐         ┌─────────────────┐
│   Command Side  │         │   Query Side    │
│                 │         │                 │
│  HTTP/gRPC API  │         │  HTTP/gRPC API  │
│       ↓         │         │       ↓         │
│  Command Handler│         │  Query Handler  │
│       ↓         │         │       ↓         │
│   Aggregate     │ Events   │  Read Model     │
│   (Write DB)    │─────────▶│  (Read DB)      │
└─────────────────┘         └─────────────────┘
```

---

## Project Structure

```
cqrs-app/
├── api/                      # Protocol buffer definitions
│   └── order/
│       ├── command.proto     # Write operations
│       └── query.proto       # Read operations
├── cmd/
│   ├── command/              # Command side service
│   │   └── main.go
│   └── query/                # Query side service
│       └── main.go
├── internal/
│   ├── command/              # Command side logic
│   │   ├── biz/              # Aggregates, commands
│   │   ├── data/             # Write repositories
│   │   ├── service/          # Command handlers
│   │   └── server/
│   ├── query/                # Query side logic
│   │   ├── biz/              # Query handlers
│   │   ├── data/             # Read repositories
│   │   ├── service/          # Query handlers
│   │   └── server/
│   └── event/                # Shared event definitions
├── pkg/
│   └── events/               # Domain events
└── configs/
```

---

## Command Side

### Command API Definition

```protobuf
// api/order/command.proto
syntax = "proto3";

package api.order.command;

option go_package = "order-service/api/order/command;v1";

import "google/protobuf/timestamp.proto";
import "validate/validate.proto";

service OrderCommandService {
    rpc CreateOrder (CreateOrderCommand) returns (OrderCreated);
    rpc UpdateOrderStatus (UpdateOrderStatusCommand) returns (OrderStatusUpdated);
    rpc CancelOrder (CancelOrderCommand) returns (OrderCancelled);
}

// Commands
message CreateOrderCommand {
    int64 customer_id = 1 [(validate.rules).int64 = {gt: 0}];
    repeated OrderItem items = 2 [(validate.rules).repeated = {min_items: 1}];
    string shipping_address = 3 [(validate.rules).string.min_len = 10];
}

message UpdateOrderStatusCommand {
    int64 order_id = 1 [(validate.rules).int64 = {gt: 0}];
    string status = 2 [(validate.rules).string = {
        in: ["confirmed", "shipped", "delivered"]
    }];
}

message CancelOrderCommand {
    int64 order_id = 1 [(validate.rules).int64 = {gt: 0}];
    string reason = 2 [(validate.rules).string.min_len = 1];
}

// Events (responses)
message OrderCreated {
    int64 order_id = 1;
    int64 customer_id = 2;
    double total_amount = 3;
    google.protobuf.Timestamp created_at = 4;
}

message OrderStatusUpdated {
    int64 order_id = 1;
    string old_status = 2;
    string new_status = 3;
    google.protobuf.Timestamp updated_at = 4;
}

message OrderCancelled {
    int64 order_id = 1;
    string reason = 2;
    google.protobuf.Timestamp cancelled_at = 3;
}

message OrderItem {
    int64 product_id = 1 [(validate.rules).int64 = {gt: 0}];
    int32 quantity = 2 [(validate.rules).int32 = {gt: 0}];
    double price = 3 [(validate.rules).double = {gt: 0}];
}
```

### Aggregate Root

```go
// internal/command/biz/order_aggregate.go
package biz

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/errors"
)

// OrderStatus represents order status
type OrderStatus string

const (
    OrderStatusPending    OrderStatus = "pending"
    OrderStatusConfirmed  OrderStatus = "confirmed"
    OrderStatusShipped    OrderStatus = "shipped"
    OrderStatusDelivered  OrderStatus = "delivered"
    OrderStatusCancelled  OrderStatus = "cancelled"
)

// OrderAggregate is the write model
type OrderAggregate struct {
    ID              int64
    CustomerID      int64
    Items           []*OrderItem
    Status          OrderStatus
    ShippingAddress string
    TotalAmount     float64
    CreatedAt       time.Time
    UpdatedAt       time.Time
    Version         int // Optimistic locking
}

type OrderItem struct {
    ProductID int64
    Quantity  int32
    Price     float64
}

// Domain methods

func (o *OrderAggregate) Create(customerID int64, items []*OrderItem, address string) error {
    if len(items) == 0 {
        return errors.BadRequest("EMPTY_ORDER", "order must have at least one item")
    }
    
    o.CustomerID = customerID
    o.Items = items
    o.ShippingAddress = address
    o.Status = OrderStatusPending
    o.CreatedAt = time.Now()
    o.UpdatedAt = time.Now()
    
    // Calculate total
    for _, item := range items {
        o.TotalAmount += item.Price * float64(item.Quantity)
    }
    
    return nil
}

func (o *OrderAggregate) Confirm() error {
    if o.Status != OrderStatusPending {
        return errors.BadRequest("INVALID_STATUS", "only pending orders can be confirmed")
    }
    
    o.Status = OrderStatusConfirmed
    o.UpdatedAt = time.Now()
    o.Version++
    
    return nil
}

func (o *OrderAggregate) Cancel(reason string) error {
    if o.Status == OrderStatusDelivered {
        return errors.BadRequest("INVALID_STATUS", "delivered orders cannot be cancelled")
    }
    
    if o.Status == OrderStatusCancelled {
        return errors.BadRequest("ALREADY_CANCELLED", "order is already cancelled")
    }
    
    o.Status = OrderStatusCancelled
    o.UpdatedAt = time.Now()
    o.Version++
    
    return nil
}

// Events

type OrderCreatedEvent struct {
    OrderID     int64
    CustomerID  int64
    Items       []*OrderItem
    TotalAmount float64
    CreatedAt   time.Time
}

type OrderConfirmedEvent struct {
    OrderID   int64
    Version   int
    Timestamp time.Time
}

type OrderCancelledEvent struct {
    OrderID   int64
    Reason    string
    Timestamp time.Time
}
```

### Command Handler

```go
// internal/command/biz/order_usecase.go
package biz

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/log"
)

// OrderCommandRepo handles write operations
type OrderCommandRepo interface {
    Save(ctx context.Context, order *OrderAggregate) error
    Update(ctx context.Context, order *OrderAggregate) error
    Find(ctx context.Context, id int64) (*OrderAggregate, error)
}

// EventPublisher publishes domain events
type EventPublisher interface {
    Publish(ctx context.Context, event interface{}) error
}

type OrderCommandUsecase struct {
    repo      OrderCommandRepo
    publisher EventPublisher
    log       *log.Helper
}

func NewOrderCommandUsecase(repo OrderCommandRepo, publisher EventPublisher, logger log.Logger) *OrderCommandUsecase {
    return &OrderCommandUsecase{
        repo:      repo,
        publisher: publisher,
        log:       log.NewHelper(logger),
    }
}

func (uc *OrderCommandUsecase) CreateOrder(ctx context.Context, cmd *CreateOrderCommand) (*OrderCreatedEvent, error) {
    // Create aggregate
    order := &OrderAggregate{}
    if err := order.Create(cmd.CustomerID, cmd.Items, cmd.ShippingAddress); err != nil {
        return nil, err
    }
    
    // Save to write database
    if err := uc.repo.Save(ctx, order); err != nil {
        return nil, err
    }
    
    // Create and publish event
    event := &OrderCreatedEvent{
        OrderID:     order.ID,
        CustomerID:  order.CustomerID,
        Items:       order.Items,
        TotalAmount: order.TotalAmount,
        CreatedAt:   order.CreatedAt,
    }
    
    if err := uc.publisher.Publish(ctx, event); err != nil {
        uc.log.Errorw("failed to publish event", "error", err)
        // Log error but don't fail - event can be replayed
    }
    
    return event, nil
}

func (uc *OrderCommandUsecase) ConfirmOrder(ctx context.Context, orderID int64) error {
    // Load aggregate
    order, err := uc.repo.Find(ctx, orderID)
    if err != nil {
        return err
    }
    
    // Execute domain logic
    if err := order.Confirm(); err != nil {
        return err
    }
    
    // Save changes
    if err := uc.repo.Update(ctx, order); err != nil {
        return err
    }
    
    // Publish event
    event := &OrderConfirmedEvent{
        OrderID:   order.ID,
        Version:   order.Version,
        Timestamp: time.Now(),
    }
    
    return uc.publisher.Publish(ctx, event)
}

func (uc *OrderCommandUsecase) CancelOrder(ctx context.Context, orderID int64, reason string) error {
    order, err := uc.repo.Find(ctx, orderID)
    if err != nil {
        return err
    }
    
    if err := order.Cancel(reason); err != nil {
        return err
    }
    
    if err := uc.repo.Update(ctx, order); err != nil {
        return err
    }
    
    event := &OrderCancelledEvent{
        OrderID:   order.ID,
        Reason:    reason,
        Timestamp: time.Now(),
    }
    
    return uc.publisher.Publish(ctx, event)
}
```

---

## Query Side

### Query API Definition

```protobuf
// api/order/query.proto
syntax = "proto3";

package api.order.query;

option go_package = "order-service/api/order/query;v1";

import "google/protobuf/timestamp.proto";

service OrderQueryService {
    rpc GetOrder (GetOrderQuery) returns (OrderDetail);
    rpc ListOrders (ListOrdersQuery) returns (OrderList);
    rpc GetCustomerOrders (GetCustomerOrdersQuery) returns (OrderList);
    rpc GetOrderSummary (GetOrderSummaryQuery) returns (OrderSummary);
}

// Queries
message GetOrderQuery {
    int64 order_id = 1;
}

message ListOrdersQuery {
    int32 page = 1;
    int32 page_size = 2;
    string status = 3;
    string sort_by = 4;
}

message GetCustomerOrdersQuery {
    int64 customer_id = 1;
    int32 page = 2;
    int32 page_size = 3;
}

message GetOrderSummaryQuery {
    int64 customer_id = 1;
}

// Read Models (DTOs)
message OrderDetail {
    int64 order_id = 1;
    int64 customer_id = 2;
    string customer_name = 3;
    repeated OrderItem items = 4;
    double total_amount = 5;
    string status = 6;
    string shipping_address = 7;
    google.protobuf.Timestamp created_at = 8;
    google.protobuf.Timestamp updated_at = 9;
}

message OrderList {
    repeated OrderSummary orders = 1;
    int32 total = 2;
    int32 page = 3;
    int32 page_size = 4;
}

message OrderSummary {
    int64 order_id = 1;
    int64 customer_id = 2;
    double total_amount = 3;
    string status = 4;
    google.protobuf.Timestamp created_at = 5;
    int32 item_count = 6;
}

message OrderItem {
    int64 product_id = 1;
    string product_name = 2;
    int32 quantity = 3;
    double price = 4;
}
```

### Read Model Repository

```go
// internal/query/biz/order_query.go
package biz

import (
    "context"
    
    v1 "order-service/api/order/query/v1"
)

// OrderQueryRepo handles optimized read queries
type OrderQueryRepo interface {
    GetOrder(ctx context.Context, orderID int64) (*v1.OrderDetail, error)
    ListOrders(ctx context.Context, query *v1.ListOrdersQuery) (*v1.OrderList, error)
    GetCustomerOrders(ctx context.Context, customerID int64, page, pageSize int32) (*v1.OrderList, error)
    GetOrderSummary(ctx context.Context, customerID int64) (*v1.OrderSummary, error)
}

type OrderQueryUsecase struct {
    repo OrderQueryRepo
}

func NewOrderQueryUsecase(repo OrderQueryRepo) *OrderQueryUsecase {
    return &OrderQueryUsecase{repo: repo}
}

func (uc *OrderQueryUsecase) GetOrder(ctx context.Context, orderID int64) (*v1.OrderDetail, error) {
    return uc.repo.GetOrder(ctx, orderID)
}

func (uc *OrderQueryUsecase) ListOrders(ctx context.Context, query *v1.ListOrdersQuery) (*v1.OrderList, error) {
    return uc.repo.ListOrders(ctx, query)
}
```

### Query Data Access

```go
// internal/query/data/order_query_repo.go
package data

import (
    "context"
    
    v1 "order-service/api/order/query/v1"
)

// OrderReadModel represents the denormalized read model
type OrderReadModel struct {
    OrderID         int64
    CustomerID      int64
    CustomerName    string
    TotalAmount     float64
    Status          string
    ShippingAddress string
    CreatedAt       time.Time
    UpdatedAt       time.Time
    Items           []OrderItemReadModel
}

type OrderItemReadModel struct {
    ProductID   int64
    ProductName string
    Quantity    int32
    Price       float64
}

type orderQueryRepo struct {
    db  *gorm.DB // Read-optimized database
    rdb *redis.Client // Cache
}

func (r *orderQueryRepo) GetOrder(ctx context.Context, orderID int64) (*v1.OrderDetail, error) {
    // Try cache first
    cacheKey := fmt.Sprintf("order:%d", orderID)
    
    cached, err := r.rdb.Get(ctx, cacheKey).Result()
    if err == nil {
        var detail v1.OrderDetail
        if err := json.Unmarshal([]byte(cached), &detail); err == nil {
            return &detail, nil
        }
    }
    
    // Query denormalized read model
    var order OrderReadModel
    err = r.db.WithContext(ctx).
        Table("order_read_models").
        Where("order_id = ?", orderID).
        First(&order).Error
    if err != nil {
        return nil, err
    }
    
    // Query items
    var items []OrderItemReadModel
    err = r.db.WithContext(ctx).
        Table("order_item_read_models").
        Where("order_id = ?", orderID).
        Find(&items).Error
    if err != nil {
        return nil, err
    }
    
    // Build response
    detail := &v1.OrderDetail{
        OrderId:         order.OrderID,
        CustomerId:      order.CustomerID,
        CustomerName:    order.CustomerName,
        TotalAmount:     order.TotalAmount,
        Status:          order.Status,
        ShippingAddress: order.ShippingAddress,
        CreatedAt:       timestamppb.New(order.CreatedAt),
        UpdatedAt:       timestamppb.New(order.UpdatedAt),
    }
    
    for _, item := range items {
        detail.Items = append(detail.Items, &v1.OrderItem{
            ProductId:   item.ProductID,
            ProductName: item.ProductName,
            Quantity:    item.Quantity,
            Price:       item.Price,
        })
    }
    
    // Cache result
    if data, err := json.Marshal(detail); err == nil {
        r.rdb.Set(ctx, cacheKey, data, 5*time.Minute)
    }
    
    return detail, nil
}

func (r *orderQueryRepo) ListOrders(ctx context.Context, query *v1.ListOrdersQuery) (*v1.OrderList, error) {
    db := r.db.WithContext(ctx).Table("order_read_models")
    
    // Apply filters
    if query.Status != "" {
        db = db.Where("status = ?", query.Status)
    }
    
    // Get total count
    var total int64
    if err := db.Count(&total).Error; err != nil {
        return nil, err
    }
    
    // Apply sorting and pagination
    offset := (query.Page - 1) * query.PageSize
    
    var orders []OrderReadModel
    err := db.Order("created_at DESC").
        Offset(int(offset)).
        Limit(int(query.PageSize)).
        Find(&orders).Error
    if err != nil {
        return nil, err
    }
    
    // Build response
    result := &v1.OrderList{
        Total:    int32(total),
        Page:     query.Page,
        PageSize: query.PageSize,
    }
    
    for _, o := range orders {
        result.Orders = append(result.Orders, &v1.OrderSummary{
            OrderId:    o.OrderID,
            CustomerId: o.CustomerID,
            TotalAmount: o.TotalAmount,
            Status:     o.Status,
            CreatedAt:  timestamppb.New(o.CreatedAt),
            ItemCount:  int32(len(o.Items)),
        })
    }
    
    return result, nil
}
```

---

## Event Projection

### Event Handler

```go
// internal/query/biz/event_handler.go
package biz

import (
    "context"
    "encoding/json"
    
    "github.com/go-kratos/kratos/v2/log"
    "user-service/internal/event"
)

type OrderEventHandler struct {
    repo OrderQueryRepo
    rdb  *redis.Client
    log  *log.Helper
}

func NewOrderEventHandler(repo OrderQueryRepo, rdb *redis.Client, logger log.Logger) *OrderEventHandler {
    return &OrderEventHandler{
        repo: repo,
        rdb:  rdb,
        log:  log.NewHelper(logger),
    }
}

func (h *OrderEventHandler) HandleOrderCreated(ctx context.Context, msg event.Event) error {
    var event commandbiz.OrderCreatedEvent
    if err := json.Unmarshal(msg.Value(), &event); err != nil {
        return err
    }
    
    // Insert into read model
    readModel := &OrderReadModel{
        OrderID:     event.OrderID,
        CustomerID:  event.CustomerID,
        TotalAmount: event.TotalAmount,
        Status:      "pending",
        CreatedAt:   event.CreatedAt,
    }
    
    // Save to read database
    if err := h.repo.InsertReadModel(ctx, readModel); err != nil {
        return err
    }
    
    // Invalidate related caches
    h.rdb.Del(ctx, fmt.Sprintf("customer_orders:%d", event.CustomerID))
    
    h.log.Infow("order read model created", "order_id", event.OrderID)
    return nil
}

func (h *OrderEventHandler) HandleOrderConfirmed(ctx context.Context, msg event.Event) error {
    var event commandbiz.OrderConfirmedEvent
    if err := json.Unmarshal(msg.Value(), &event); err != nil {
        return err
    }
    
    // Update read model
    if err := h.repo.UpdateStatus(ctx, event.OrderID, "confirmed"); err != nil {
        return err
    }
    
    // Invalidate cache
    h.rdb.Del(ctx, fmt.Sprintf("order:%d", event.OrderID))
    
    return nil
}
```

---

## Database Separation

### Write Database (Command Side)

```sql
-- Optimized for transactional consistency
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    shipping_address TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    version INT DEFAULT 0,
    INDEX idx_customer (customer_id),
    INDEX idx_status (status)
) ENGINE=InnoDB;

CREATE TABLE order_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id)
) ENGINE=InnoDB;
```

### Read Database (Query Side)

```sql
-- Denormalized for fast reads
CREATE TABLE order_read_models (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    customer_name VARCHAR(100),
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    shipping_address TEXT,
    item_count INT DEFAULT 0,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    -- Pre-computed fields for faster queries
    total_items_quantity INT DEFAULT 0,
    INDEX idx_customer_created (customer_id, created_at DESC),
    INDEX idx_status_created (status, created_at DESC)
) ENGINE=InnoDB;

-- Materialized view for aggregations
CREATE TABLE customer_order_summaries (
    customer_id BIGINT PRIMARY KEY,
    total_orders INT DEFAULT 0,
    total_spent DECIMAL(12,2) DEFAULT 0,
    last_order_at TIMESTAMP,
    INDEX idx_total_spent (total_spent DESC)
) ENGINE=InnoDB;
```

---

## Best Practices

### ✅ Always Follow

- Separate command and query models clearly
- Use events to synchronize read models
- Design read models for specific query requirements
- Use eventual consistency between command and query sides
- Implement idempotent event handlers
- Version your events for backward compatibility
- Use optimistic locking on command side

### ❌ Never Do

```go
// DON'T: Use the same model for reads and writes
func (s *Service) GetOrder(ctx context.Context, id int64) (*OrderAggregate, error) {
    // ❌ Exposing aggregate for reads
}

// DON'T: Query from write database in read operations
func (uc *QueryUsecase) ListOrders(ctx context.Context) ([]*OrderAggregate, error) {
    // ❌ Should use read model
}

// DON'T: Skip event publishing on command side
func (uc *CommandUsecase) CreateOrder(ctx context.Context, cmd *CreateCommand) error {
    // ... save order ...
    // ❌ Missing event publishing
}

// DON'T: Block command execution on read model update
func (uc *CommandUsecase) CreateOrder(ctx context.Context, cmd *CreateCommand) error {
    // ... save order ...
    if err := uc.updateReadModel(ctx); err != nil {
        return err  // ❌ Should be async
    }
}
```

---

## Testing

```go
// internal/command/biz/order_test.go
func TestOrderCommandUsecase_CreateOrder(t *testing.T) {
    mockRepo := new(MockOrderCommandRepo)
    mockPublisher := new(MockEventPublisher)
    
    uc := NewOrderCommandUsecase(mockRepo, mockPublisher, log.DefaultLogger)
    
    mockRepo.On("Save", mock.Anything, mock.AnythingOfType("*biz.OrderAggregate")).
        Return(nil).
        Run(func(args mock.Arguments) {
            order := args.Get(1).(*biz.OrderAggregate)
            order.ID = 1 // Simulate DB assigning ID
        })
    
    mockPublisher.On("Publish", mock.Anything, mock.AnythingOfType("*biz.OrderCreatedEvent")).
        Return(nil)
    
    event, err := uc.CreateOrder(context.Background(), &CreateOrderCommand{
        CustomerID: 1,
        Items: []*OrderItem{
            {ProductID: 1, Quantity: 2, Price: 10.0},
        },
    })
    
    assert.NoError(t, err)
    assert.Equal(t, int64(1), event.OrderID)
    assert.Equal(t, 20.0, event.TotalAmount)
}
```
