# Event-Driven Architecture Patterns (go-kratos)

This guide covers event-driven architecture patterns in go-kratos using message brokers like Kafka, RabbitMQ, and NATS.

## Overview

- Decouple services using asynchronous messaging
- Support multiple message brokers (Kafka, RabbitMQ, NATS, Redis Pub/Sub)
- Implement event sourcing and CQRS patterns
- Handle message delivery guarantees (at-least-once, exactly-once)

## Core Interfaces

```go
// internal/event/event.go
package event

import "context"

// Event represents a domain event
type Event interface {
    Key() string      // Routing key / partition key
    Value() []byte    // Serialized event data
    Type() string     // Event type
    Timestamp() int64 // Event timestamp
}

// Handler processes events
type Handler func(context.Context, Event) error

// Sender publishes events
type Sender interface {
    Send(ctx context.Context, msg Event) error
    Close() error
}

// Receiver consumes events
type Receiver interface {
    Receive(ctx context.Context, handler Handler) error
    Close() error
}
```

---

## Kafka Integration

### Dependencies

```bash
go get github.com/segmentio/kafka-go
go get github.com/go-kratos/kratos/contrib/transport/kafka/v2
```

### Event Definition

```go
// internal/event/event.go
package event

import (
    "encoding/json"
    "time"
)

// Message implements Event interface
type Message struct {
    key       string
    value     []byte
    eventType string
    timestamp int64
}

func (m *Message) Key() string      { return m.key }
func (m *Message) Value() []byte    { return m.value }
func (m *Message) Type() string     { return m.eventType }
func (m *Message) Timestamp() int64 { return m.timestamp }

// NewMessage creates a new event message
func NewMessage(key string, value []byte) Event {
    return &Message{
        key:       key,
        value:     value,
        timestamp: time.Now().Unix(),
    }
}

// Domain Event Types

// UserCreatedEvent
type UserCreatedEvent struct {
    UserID    int64     `json:"user_id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

func (e *UserCreatedEvent) ToEvent() (Event, error) {
    data, err := json.Marshal(e)
    if err != nil {
        return nil, err
    }
    return &Message{
        key:       fmt.Sprintf("user:%d", e.UserID),
        value:     data,
        eventType: "user.created",
        timestamp: time.Now().Unix(),
    }, nil
}
```

### Kafka Sender

```go
// internal/event/kafka/sender.go
package kafka

import (
    "context"
    
    "github.com/segmentio/kafka-go"
    "user-service/internal/event"
)

type sender struct {
    writer *kafka.Writer
    topic  string
}

// NewSender creates a Kafka sender
func NewSender(brokers []string, topic string) (event.Sender, error) {
    w := &kafka.Writer{
        Topic:    topic,
        Addr:     kafka.TCP(brokers...),
        Balancer: &kafka.LeastBytes{},
        // For production, configure:
        // RequiredAcks: kafka.RequireAll,
        // Async:        false,
    }
    
    return &sender{
        writer: w,
        topic:  topic,
    }, nil
}

func (s *sender) Send(ctx context.Context, msg event.Event) error {
    return s.writer.WriteMessages(ctx, kafka.Message{
        Key:   []byte(msg.Key()),
        Value: msg.Value(),
        Headers: []kafka.Header{
            {Key: "event_type", Value: []byte(msg.Type())},
            {Key: "timestamp", Value: []byte(fmt.Sprintf("%d", msg.Timestamp()))},
        },
    })
}

func (s *sender) Close() error {
    return s.writer.Close()
}
```

### Kafka Receiver

```go
// internal/event/kafka/receiver.go
package kafka

import (
    "context"
    "fmt"
    
    "github.com/go-kratos/kratos/v2/log"
    "github.com/segmentio/kafka-go"
    "user-service/internal/event"
)

type receiver struct {
    reader *kafka.Reader
    topic  string
    log    *log.Helper
}

// NewReceiver creates a Kafka receiver
func NewReceiver(brokers []string, topic, groupID string, logger log.Logger) (event.Receiver, error) {
    r := kafka.NewReader(kafka.ReaderConfig{
        Brokers:  brokers,
        GroupID:  groupID,
        Topic:    topic,
        MinBytes: 10e3, // 10KB
        MaxBytes: 10e6, // 10MB
        // For production:
        // CommitInterval: 1 * time.Second,
        // StartOffset:    kafka.FirstOffset,
    })
    
    return &receiver{
        reader: r,
        topic:  topic,
        log:    log.NewHelper(logger),
    }, nil
}

func (r *receiver) Receive(ctx context.Context, handler event.Handler) error {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            default:
            }
            
            m, err := r.reader.FetchMessage(ctx)
            if err != nil {
                r.log.Errorw("failed to fetch message", "error", err)
                continue
            }
            
            msg := &event.Message{
                key:   string(m.Key),
                value: m.Value,
            }
            
            // Extract headers
            for _, h := range m.Headers {
                switch h.Key {
                case "event_type":
                    msg.eventType = string(h.Value)
                case "timestamp":
                    fmt.Sscanf(string(h.Value), "%d", &msg.timestamp)
                }
            }
            
            // Process message
            if err := handler(ctx, msg); err != nil {
                r.log.Errorw("message handler error", "error", err)
                // Decide: retry, DLQ, or continue
                continue
            }
            
            // Commit offset
            if err := r.reader.CommitMessages(ctx, m); err != nil {
                r.log.Errorw("failed to commit message", "error", err)
            }
        }
    }()
    
    return nil
}

func (r *receiver) Close() error {
    return r.reader.Close()
}
```

### Event Publisher Service

```go
// internal/biz/event.go
package biz

import (
    "context"
    "encoding/json"
    
    "user-service/internal/event"
)

type EventPublisher struct {
    sender event.Sender
}

func NewEventPublisher(sender event.Sender) *EventPublisher {
    return &EventPublisher{sender: sender}
}

func (p *EventPublisher) PublishUserCreated(ctx context.Context, user *User) error {
    evt := &UserCreatedEvent{
        UserID:    user.ID,
        Name:      user.Name,
        Email:     user.Email,
        CreatedAt: user.CreatedAt,
    }
    
    msg, err := evt.ToEvent()
    if err != nil {
        return err
    }
    
    return p.sender.Send(ctx, msg)
}

func (p *EventPublisher) PublishUserUpdated(ctx context.Context, user *User) error {
    evt := map[string]interface{}{
        "user_id":    user.ID,
        "name":       user.Name,
        "email":      user.Email,
        "updated_at": user.UpdatedAt,
    }
    
    data, err := json.Marshal(evt)
    if err != nil {
        return err
    }
    
    return p.sender.Send(ctx, event.NewMessage(
        fmt.Sprintf("user:%d", user.ID),
        data,
    ))
}
```

---

## Event Consumer

### Consumer Group

```go
// internal/event/consumer/user_consumer.go
package consumer

import (
    "context"
    "encoding/json"
    
    "github.com/go-kratos/kratos/v2/log"
    "user-service/internal/biz"
    "user-service/internal/event"
)

type UserConsumer struct {
    receiver event.Receiver
    userUC   *biz.UserUsecase
    log      *log.Helper
}

func NewUserConsumer(receiver event.Receiver, userUC *biz.UserUsecase, logger log.Logger) *UserConsumer {
    return &UserConsumer{
        receiver: receiver,
        userUC:   userUC,
        log:      log.NewHelper(logger),
    }
}

func (c *UserConsumer) Start(ctx context.Context) error {
    return c.receiver.Receive(ctx, c.handleEvent)
}

func (c *UserConsumer) handleEvent(ctx context.Context, msg event.Event) error {
    c.log.Infow("received event", 
        "type", msg.Type(),
        "key", msg.Key(),
    )
    
    switch msg.Type() {
    case "user.created":
        return c.handleUserCreated(ctx, msg)
    case "user.updated":
        return c.handleUserUpdated(ctx, msg)
    default:
        c.log.Warnw("unknown event type", "type", msg.Type())
        return nil
    }
}

func (c *UserConsumer) handleUserCreated(ctx context.Context, msg event.Event) error {
    var evt biz.UserCreatedEvent
    if err := json.Unmarshal(msg.Value(), &evt); err != nil {
        return err
    }
    
    // Process event
    // e.g., send welcome email, update search index
    c.log.Infow("user created event processed", "user_id", evt.UserID)
    
    return nil
}

func (c *UserConsumer) handleUserUpdated(ctx context.Context, msg event.Event) error {
    // Process user update event
    return nil
}
```

---

## Domain Events in Biz Layer

### Event Definition

```go
// internal/biz/event.go
package biz

import (
    "context"
    "time"
)

// DomainEvent represents a domain event
type DomainEvent struct {
    ID        string
    Type      string
    Payload   interface{}
    Timestamp time.Time
}

// EventPublisher publishes domain events
type EventPublisher interface {
    Publish(ctx context.Context, event *DomainEvent) error
}

// EventHandler handles domain events
type EventHandler interface {
    Handle(ctx context.Context, event *DomainEvent) error
}
```

### Publishing Events

```go
// internal/biz/user.go
package biz

import (
    "context"
    "time"
)

type UserUsecase struct {
    repo      UserRepo
    publisher EventPublisher
    log       *log.Helper
}

func (uc *UserUsecase) CreateUser(ctx context.Context, u *User) error {
    // Validate and create user
    if err := u.Validate(); err != nil {
        return err
    }
    
    if err := uc.repo.Create(ctx, u); err != nil {
        return err
    }
    
    // Publish event
    event := &DomainEvent{
        ID:   generateUUID(),
        Type: "user.created",
        Payload: map[string]interface{}{
            "user_id": u.ID,
            "email":   u.Email,
        },
        Timestamp: time.Now(),
    }
    
    if err := uc.publisher.Publish(ctx, event); err != nil {
        uc.log.Errorw("failed to publish event", "error", err)
        // Log but don't fail - event can be replayed from DB if needed
    }
    
    return nil
}
```

---

## Outbox Pattern

The outbox pattern ensures reliable event publishing:

```go
// internal/data/outbox.go
package data

import (
    "context"
    "database/sql"
    "encoding/json"
    "time"
    
    "gorm.io/gorm"
)

// OutboxMessage represents an unpublished event
type OutboxMessage struct {
    ID        int64 `gorm:"primaryKey"`
    Topic     string
    Key       string
    Payload   []byte
    Headers   []byte
    CreatedAt time.Time
    SentAt    *time.Time
}

func (OutboxMessage) TableName() string {
    return "outbox_messages"
}

type Outbox struct {
    db  *gorm.DB
    log *log.Helper
}

func NewOutbox(db *gorm.DB, logger log.Logger) *Outbox {
    return &Outbox{
        db:  db,
        log: log.NewHelper(logger),
    }
}

// Save stores an event in the outbox (within transaction)
func (o *Outbox) Save(ctx context.Context, tx *gorm.DB, topic, key string, payload interface{}) error {
    data, err := json.Marshal(payload)
    if err != nil {
        return err
    }
    
    msg := &OutboxMessage{
        Topic:     topic,
        Key:       key,
        Payload:   data,
        CreatedAt: time.Now(),
    }
    
    return tx.WithContext(ctx).Create(msg).Error
}

// GetPending retrieves unsent messages
func (o *Outbox) GetPending(ctx context.Context, limit int) ([]*OutboxMessage, error) {
    var messages []*OutboxMessage
    err := o.db.WithContext(ctx).
        Where("sent_at IS NULL").
        Order("created_at ASC").
        Limit(limit).
        Find(&messages).Error
    return messages, err
}

// MarkSent marks a message as sent
func (o *Outbox) MarkSent(ctx context.Context, id int64) error {
    now := time.Now()
    return o.db.WithContext(ctx).
        Model(&OutboxMessage{}).
        Where("id = ?", id).
        Update("sent_at", &now).Error
}
```

### Outbox Relay Worker

```go
// internal/event/outbox/relay.go
package outbox

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/log"
    "user-service/internal/data"
    "user-service/internal/event"
)

type Relay struct {
    outbox *data.Outbox
    sender event.Sender
    log    *log.Helper
    ticker *time.Ticker
}

func NewRelay(outbox *data.Outbox, sender event.Sender, logger log.Logger) *Relay {
    return &Relay{
        outbox: outbox,
        sender: sender,
        log:    log.NewHelper(logger),
        ticker: time.NewTicker(5 * time.Second),
    }
}

func (r *Relay) Start(ctx context.Context) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            case <-r.ticker.C:
                if err := r.process(ctx); err != nil {
                    r.log.Errorw("outbox relay error", "error", err)
                }
            }
        }
    }()
}

func (r *Relay) process(ctx context.Context) error {
    messages, err := r.outbox.GetPending(ctx, 100)
    if err != nil {
        return err
    }
    
    for _, msg := range messages {
        // Send to message broker
        evt := event.NewMessage(msg.Key, msg.Payload)
        
        if err := r.sender.Send(ctx, evt); err != nil {
            r.log.Errorw("failed to send message", "id", msg.ID, "error", err)
            continue
        }
        
        // Mark as sent
        if err := r.outbox.MarkSent(ctx, msg.ID); err != nil {
            r.log.Errorw("failed to mark sent", "id", msg.ID, "error", err)
        }
    }
    
    return nil
}

func (r *Relay) Stop() {
    r.ticker.Stop()
}
```

---

## Wire Configuration

```go
// internal/wire.go
//go:build wireinject
// +build wireinject

package internal

import (
    "github.com/google/wire"
    
    "user-service/internal/event"
    "user-service/internal/event/kafka"
    "user-service/internal/event/consumer"
)

var eventSet = wire.NewSet(
    kafka.NewSender,
    kafka.NewReceiver,
    consumer.NewUserConsumer,
)

func InitializeEventSender(brokers []string, topic string) (event.Sender, error) {
    return kafka.NewSender(brokers, topic)
}
```

---

## Best Practices

### ✅ Always Follow

- Use the outbox pattern for critical events
- Design events to be immutable and self-contained
- Include event metadata (timestamp, correlation ID, version)
- Handle duplicate events (idempotency)
- Use separate consumer groups for different services
- Monitor consumer lag and processing errors

### ❌ Never Do

```go
// DON'T: Publish events outside transaction
func (uc *Usecase) Create(ctx context.Context, entity *Entity) error {
    uc.repo.Create(ctx, entity)
    uc.publisher.Publish(ctx, event)  // ❌ Event sent even if DB fails
}

// DON'T: Process events without idempotency check
func (c *Consumer) Handle(ctx context.Context, msg Event) error {
    c.service.Process(msg)  // ❌ May process duplicate
}

// DON'T: Block on event publishing
func (uc *Usecase) Create(ctx context.Context, entity *Entity) error {
    // ❌ Slows down response
    if err := uc.publisher.Publish(ctx, event); err != nil {
        return err
    }
}
```

---

## Testing

```go
// internal/event/memory/memory.go
package memory

import (
    "context"
    "sync"
    
    "user-service/internal/event"
)

type MemoryEventBus struct {
    mu       sync.RWMutex
    handlers []event.Handler
    messages []event.Event
}

func New() *MemoryEventBus {
    return &MemoryEventBus{
        handlers: make([]event.Handler, 0),
        messages: make([]event.Event, 0),
    }
}

func (b *MemoryEventBus) Send(ctx context.Context, msg event.Event) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    b.messages = append(b.messages, msg)
    
    for _, h := range b.handlers {
        go h(ctx, msg)
    }
    
    return nil
}

func (b *MemoryEventBus) Receive(ctx context.Context, handler event.Handler) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    b.handlers = append(b.handlers, handler)
    return nil
}

func (b *MemoryEventBus) Close() error { return nil }

// Test usage
func TestUserUsecase_Create(t *testing.T) {
    bus := memory.New()
    publisher := biz.NewEventPublisher(bus)
    uc := biz.NewUserUsecase(mockRepo, publisher, log.DefaultLogger)
    
    // Subscribe to events
    var receivedEvent event.Event
    bus.Receive(context.Background(), func(ctx context.Context, msg event.Event) error {
        receivedEvent = msg
        return nil
    })
    
    // Create user
    err := uc.CreateUser(context.Background(), &biz.User{
        Name:  "John",
        Email: "john@example.com",
    })
    
    assert.NoError(t, err)
    assert.NotNil(t, receivedEvent)
    assert.Equal(t, "user.created", receivedEvent.Type())
}
```
