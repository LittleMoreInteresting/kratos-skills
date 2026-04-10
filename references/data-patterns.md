# Data & Repository Patterns (go-kratos)

## Layer contract

- `biz` defines repository interfaces near use-cases.
- `data` implements interfaces with SQL/Redis/other backends.
- `biz` never imports concrete database drivers.

## Constructor conventions

- Provide `NewData(...)` for shared clients (DB, cache, broker).
- Provide `New<Domain>Repo(data *Data, logger log.Logger)` per repository.
- Add providers to wire set and regenerate injectors.

## Transactions

### Core Principles

- Keep transaction boundaries in `data` layer.
- Expose higher-level atomic operations to `biz` when needed.
- Avoid leaking transaction objects (e.g., `*gorm.DB`, `*sql.Tx`) into `biz`.
- Use `context.Context` to implicitly pass transaction state across repository calls.

### Standard Transaction Pattern (GORM / SQL / Ent)

#### 1. Define Transaction Interface in `biz`

```go
// biz/biz.go
package biz

import (
    "context"
    "github.com/google/wire"
)

// ProviderSet is biz providers.
var ProviderSet = wire.NewSet(NewUserUsecase)

// Transaction defines the interface for executing operations within a transaction.
type Transaction interface {
    InTx(context.Context, func(ctx context.Context) error) error
}
```

#### 2. Implement Transaction Manager in `data`

**GORM Example:**

```go
// data/data.go
package data

import (
    "context"
    "github.com/go-kratos/kratos/v2/log"
    "github.com/google/wire"
    "gorm.io/gorm"
)

var ProviderSet = wire.NewSet(NewData, NewDB, NewTransaction, NewUserRepo, NewCardRepo)

type Data struct {
    db  *gorm.DB
    log *log.Helper
}

// contextTxKey is used to store transaction in context.
type contextTxKey struct{}

// InTx executes the given function within a database transaction.
func (d *Data) InTx(ctx context.Context, fn func(ctx context.Context) error) error {
    return d.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        ctx = context.WithValue(ctx, contextTxKey{}, tx)
        return fn(ctx)
    })
}

// DB returns the database instance for the given context.
// Returns transaction if present in context, otherwise returns default DB.
func (d *Data) DB(ctx context.Context) *gorm.DB {
    tx, ok := ctx.Value(contextTxKey{}).(*gorm.DB)
    if ok {
        return tx
    }
    return d.db
}

// NewTransaction creates a Transaction implementation.
func NewTransaction(d *Data) biz.Transaction {
    return d
}

func NewData(db *gorm.DB, logger log.Logger) (*Data, func(), error) {
    l := log.NewHelper(log.With(logger, "module", "data"))
    return &Data{db: db, log: l}, func() {}, nil
}
```

**SQL (database/sql) Example:**

```go
// data/data.go
package data

import (
    "context"
    "database/sql"
    "github.com/go-kratos/kratos/v2/log"
    "github.com/google/wire"
)

var ProviderSet = wire.NewSet(NewData, NewDB, NewTransaction, NewUserRepo)

type Data struct {
    db  *sql.DB
    log *log.Helper
}

type contextTxKey struct{}

func (d *Data) InTx(ctx context.Context, fn func(ctx context.Context) error) error {
    tx, err := d.db.Begin()
    if err != nil {
        return err
    }
    defer func() { _ = tx.Rollback() }()

    err = fn(context.WithValue(ctx, contextTxKey{}, tx))
    if err != nil {
        return err
    }
    return tx.Commit()
}

func (d *Data) DB(ctx context.Context) *sql.Tx {
    tx, ok := ctx.Value(contextTxKey{}).(*sql.Tx)
    if ok {
        return tx
    }
    return nil // or return queries.New(d.db) for sqlc
}

func NewTransaction(d *Data) biz.Transaction {
    return d
}
```

#### 3. Repository Implementation

Repositories use `DB(ctx)` to automatically participate in transactions when called within `InTx`:

```go
// data/user.go
package data

import (
    "context"
    "github.com/go-kratos/kratos/v2/log"
)

type userRepo struct {
    data *Data
    log  *log.Helper
}

func NewUserRepo(data *Data, logger log.Logger) biz.UserRepo {
    return &userRepo{
        data: data,
        log:  log.NewHelper(logger),
    }
}

func (r *userRepo) CreateUser(ctx context.Context, u *biz.User) (int64, error) {
    // Automatically uses transaction if ctx contains tx
    result := r.data.DB(ctx).Create(&User{Name: u.Name, Email: u.Email})
    return result.LastInsertId(), result.Error
}
```

#### 4. Usecase Layer - Orchestrating Transactions

```go
// biz/transaction.go
package biz

import (
    "context"
    "github.com/go-kratos/kratos/v2/log"
)

type UserRepo interface {
    CreateUser(ctx context.Context, u *User) (int64, error)
}

type CardRepo interface {
    CreateCard(ctx context.Context, userID int64) (int64, error)
}

type UserUsecase struct {
    userRepo UserRepo
    cardRepo CardRepo
    tm       Transaction
}

func NewUserUsecase(user UserRepo, card CardRepo, tm Transaction, logger log.Logger) *UserUsecase {
    return &UserUsecase{
        userRepo: user,
        cardRepo: card,
        tm:       tm,
    }
}

// CreateUser creates a user and their associated card atomically.
func (uc *UserUsecase) CreateUser(ctx context.Context, u *User) (int, error) {
    var id int64
    err := uc.tm.InTx(ctx, func(ctx context.Context) error {
        var err error
        id, err = uc.userRepo.CreateUser(ctx, u)
        if err != nil {
            return err
        }
        _, err = uc.cardRepo.CreateCard(ctx, id)
        return err
    })
    if err != nil {
        return 0, err
    }
    return int(id), nil
}
```

### MongoDB Transaction Pattern

MongoDB uses a different pattern due to its session-based transactions:

```go
// biz/biz.go
package biz

type Transaction interface {
    StartSession(ctx context.Context) (sessionCtx context.Context, endSession func(ctx context.Context), err error)
    ExecTx(sessionCtx context.Context, cmd func(ctx context.Context) error) error
}

// biz/usecase.go
func (uc *UserUsecase) CreateUser(ctx context.Context, u *User) error {
    sessionCtx, endSession, err := uc.tm.StartSession(ctx)
    if err != nil {
        return err
    }
    defer endSession(sessionCtx)

    return uc.tm.ExecTx(sessionCtx, func(ctx context.Context) error {
        if err := uc.userRepo.CreateUser(ctx, u); err != nil {
            return err
        }
        return uc.cardRepo.CreateCard(ctx, u.ID)
    })
}
```

### Key Design Decisions

| Aspect | Recommendation |
|--------|----------------|
| **Transaction Control** | Keep in `data` layer; expose `InTx` interface to `biz` |
| **Context Propagation** | Use `context.WithValue` to pass transaction state |
| **Repo Participation** | Repos automatically detect and use tx via `DB(ctx)` method |
| **Error Handling** | Return errors from `InTx` callback to trigger rollback |
| **Nested Calls** | Design repos to work both with and without transactions |
| **MongoDB** | Use explicit `StartSession` + `ExecTx` pattern |

### Anti-Patterns to Avoid

1. **Don't pass transaction objects to biz layer:**
   ```go
   // BAD: biz should not depend on *gorm.DB
   func (uc *Usecase) Create(ctx context.Context, tx *gorm.DB) error
   ```

2. **Don't manually manage transactions in usecase:**
   ```go
   // BAD: transaction logic leaks into biz
   tx := db.Begin()
   // ... operations ...
   tx.Commit()
   ```

3. **Don't ignore context in repository calls:**
   ```go
   // BAD: bypasses transaction propagation
   db.Create(&user)  // Should be: r.data.DB(ctx).Create(&user)
   ```

## Ent ORM Pattern

[Ent](https://entgo.io/) is a powerful entity framework for Go that provides type-safe ORM capabilities.

### Setup

```bash
go install entgo.io/ent/cmd/ent@latest
go get entgo.io/ent
go get entgo.io/ent/dialect/sql
go get ariga.io/entcache
```

### Schema Definition

```go
// internal/data/ent/schema/user.go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/field"
    "entgo.io/ent/schema/index"
    "entgo.io/ent/schema/edge"
)

// User holds the schema definition for the User entity.
type User struct {
    ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int64("id"),
        field.String("name").
            MaxLen(50).
            NotEmpty(),
        field.String("email").
            Unique(),
        field.Int("age").
            Range(18, 120),
        field.Enum("status").
            Values("active", "inactive", "banned").
            Default("active"),
        field.Time("created_at").
            Default(time.Now).
            Immutable(),
        field.Time("updated_at").
            Default(time.Now).
            UpdateDefault(time.Now),
    }
}

// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("cards", Card.Type),
        edge.To("orders", Order.Type),
    }
}

// Indexes of the User.
func (User) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("email").Unique(),
        index.Fields("status", "created_at"),
    }
}
```

### Ent Client Setup

```go
// internal/data/data.go
package data

import (
    "context"
    "database/sql"
    
    "entgo.io/ent/dialect"
    "github.com/go-kratos/kratos/v2/log"
    "github.com/google/wire"
    _ "github.com/go-sql-driver/mysql"
    
    "user-service/internal/data/ent"
)

var ProviderSet = wire.NewSet(NewData, NewEntClient, NewUserRepo)

type Data struct {
    db  *ent.Client
    log *log.Helper
}

func NewEntClient(c *conf.Data) (*ent.Client, error) {
    // Open database connection
    drv, err := sql.Open("mysql", c.Database.Source)
    if err != nil {
        return nil, err
    }
    
    // Create Ent client
    client := ent.NewClient(
        ent.Driver(dialect.NewDriver("mysql", drv)),
    )
    
    // Run auto migration
    if err := client.Schema.Create(context.Background()); err != nil {
        return nil, err
    }
    
    return client, nil
}

func NewData(client *ent.Client, logger log.Logger) (*Data, func(), error) {
    log := log.NewHelper(log.With(logger, "module", "data"))
    
    d := &Data{
        db:  client,
        log: log,
    }
    
    cleanup := func() {
        if err := d.db.Close(); err != nil {
            log.Errorw("failed to close ent client", "error", err)
        }
    }
    
    return d, cleanup, nil
}
```

### Ent Repository Implementation

```go
// internal/data/user.go
package data

import (
    "context"
    
    "entgo.io/ent/dialect/sql"
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/log"
    
    "user-service/internal/biz"
    "user-service/internal/data/ent"
    "user-service/internal/data/ent/user"
)

type userRepo struct {
    data *Data
    log  *log.Helper
}

func NewUserRepo(data *Data, logger log.Logger) biz.UserRepo {
    return &userRepo{
        data: data,
        log:  log.NewHelper(logger),
    }
}

func (r *userRepo) toBiz(u *ent.User) *biz.User {
    return &biz.User{
        ID:        u.ID,
        Name:      u.Name,
        Email:     u.Email,
        Age:       u.Age,
        Status:    biz.UserStatus(u.Status),
        CreatedAt: u.CreatedAt,
        UpdatedAt: u.UpdatedAt,
    }
}

func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    u, err := r.data.db.User.Get(ctx, id)
    if err != nil {
        if ent.IsNotFound(err) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        return nil, err
    }
    return r.toBiz(u), nil
}

func (r *userRepo) GetByEmail(ctx context.Context, email string) (*biz.User, error) {
    u, err := r.data.db.User.
        Query().
        Where(user.Email(email)).
        Only(ctx)
    if err != nil {
        if ent.IsNotFound(err) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        return nil, err
    }
    return r.toBiz(u), nil
}

func (r *userRepo) List(ctx context.Context, page, pageSize int) ([]*biz.User, error) {
    offset := (page - 1) * pageSize
    
    users, err := r.data.db.User.
        Query().
        Order(ent.Desc(user.FieldCreatedAt)).
        Offset(offset).
        Limit(pageSize).
        All(ctx)
    if err != nil {
        return nil, err
    }
    
    result := make([]*biz.User, len(users))
    for i, u := range users {
        result[i] = r.toBiz(u)
    }
    
    return result, nil
}

func (r *userRepo) Create(ctx context.Context, u *biz.User) error {
    _, err := r.data.db.User.
        Create().
        SetName(u.Name).
        SetEmail(u.Email).
        SetAge(u.Age).
        Save(ctx)
    return err
}

func (r *userRepo) Update(ctx context.Context, u *biz.User) error {
    return r.data.db.User.
        UpdateOneID(u.ID).
        SetName(u.Name).
        SetAge(u.Age).
        Exec(ctx)
}

func (r *userRepo) Delete(ctx context.Context, id int64) error {
    return r.data.db.User.
        DeleteOneID(id).
        Exec(ctx)
}
```

### Ent with Eager Loading

```go
func (r *userRepo) GetWithCards(ctx context.Context, id int64) (*biz.User, error) {
    u, err := r.data.db.User.
        Query().
        Where(user.ID(id)).
        WithCards().  // Eager load cards
        Only(ctx)
    if err != nil {
        return nil, err
    }
    
    user := r.toBiz(u)
    
    // Map cards
    for _, c := range u.Edges.Cards {
        user.Cards = append(user.Cards, &biz.Card{
            ID:     c.ID,
            UserID: c.UserID,
            Number: c.Number,
        })
    }
    
    return user, nil
}
```

### Ent Transaction Pattern

```go
// data/data.go
func (d *Data) InTx(ctx context.Context, fn func(ctx context.Context) error) error {
    return d.db.Tx(ctx, func(tx *ent.Tx) error {
        return fn(ent.NewContext(ctx, tx.Client()))
    })
}

// Repository usage
func (r *userRepo) CreateWithCard(ctx context.Context, u *biz.User, card *biz.Card) error {
    return r.data.db.Tx(ctx, func(tx *ent.Tx) error {
        // Create user
        user, err := tx.User.
            Create().
            SetName(u.Name).
            SetEmail(u.Email).
            Save(ctx)
        if err != nil {
            return err
        }
        
        // Create card
        _, err = tx.Card.
            Create().
            SetUserID(user.ID).
            SetNumber(card.Number).
            Save(ctx)
        if err != nil {
            return err
        }
        
        return nil
    })
}
```

---

## Redis Cache Pattern

### Dependencies

```bash
go get github.com/redis/go-redis/v9
```

### Redis Client Setup

```go
// internal/data/data.go
package data

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/log"
    "github.com/google/wire"
    "github.com/redis/go-redis/v9"
)

var ProviderSet = wire.NewSet(NewData, NewRedis, NewDB, NewUserRepo)

type Data struct {
    db  *gorm.DB
    rdb *redis.Client
    log *log.Helper
}

func NewRedis(c *conf.Data) *redis.Client {
    rdb := redis.NewClient(&redis.Options{
        Addr:         c.Redis.Addr,
        Password:     c.Redis.Password,
        DB:           c.Redis.Db,
        PoolSize:     10,
        MinIdleConns: 5,
        MaxRetries:   3,
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
    })
    
    return rdb
}

func NewData(db *gorm.DB, rdb *redis.Client, logger log.Logger) (*Data, func(), error) {
    log := log.NewHelper(log.With(logger, "module", "data"))
    
    d := &Data{
        db:  db,
        rdb: rdb,
        log: log,
    }
    
    cleanup := func() {
        if err := rdb.Close(); err != nil {
            log.Errorw("failed to close redis", "error", err)
        }
    }
    
    return d, cleanup, nil
}
```

### Cache-Aside Pattern

```go
// internal/data/user.go
package data

import (
    "context"
    "encoding/json"
    "fmt"
    "time"
    
    "github.com/redis/go-redis/v9"
)

const (
    userCacheKey     = "user:%d"
    userCacheTTL     = 10 * time.Minute
    userCacheNilTTL  = 1 * time.Minute  // Cache nil to prevent cache penetration
)

type userRepo struct {
    data *Data
    log  *log.Helper
}

func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    cacheKey := fmt.Sprintf(userCacheKey, id)
    
    // Try cache first
    cached, err := r.data.rdb.Get(ctx, cacheKey).Result()
    if err == nil {
        var user biz.User
        if err := json.Unmarshal([]byte(cached), &user); err == nil {
            r.log.Debugw("cache hit", "key", cacheKey)
            return &user, nil
        }
    } else if err != redis.Nil {
        r.log.Warnw("redis get error", "error", err)
    }
    
    // Cache miss - get from DB
    user, err := r.getFromDB(ctx, id)
    if err != nil {
        return nil, err
    }
    
    // Update cache asynchronously
    go r.setCache(context.Background(), cacheKey, user)
    
    return user, nil
}

func (r *userRepo) getFromDB(ctx context.Context, id int64) (*biz.User, error) {
    var user User
    result := r.data.db.WithContext(ctx).First(&user, id)
    if result.Error != nil {
        return nil, result.Error
    }
    return toBizUser(&user), nil
}

func (r *userRepo) setCache(ctx context.Context, key string, user *biz.User) {
    data, err := json.Marshal(user)
    if err != nil {
        r.log.Warnw("marshal error", "error", err)
        return
    }
    
    if err := r.data.rdb.Set(ctx, key, data, userCacheTTL).Err(); err != nil {
        r.log.Warnw("cache set error", "error", err)
    }
}

func (r *userRepo) DeleteCache(ctx context.Context, id int64) error {
    cacheKey := fmt.Sprintf(userCacheKey, id)
    return r.data.rdb.Del(ctx, cacheKey).Err()
}
```

### Write-Through Cache

```go
func (r *userRepo) Update(ctx context.Context, u *biz.User) error {
    // Update database first
    result := r.data.db.WithContext(ctx).Model(&User{}).
        Where("id = ?", u.ID).
        Updates(map[string]interface{}{
            "name":       u.Name,
            "age":        u.Age,
            "updated_at": time.Now(),
        })
    
    if result.Error != nil {
        return result.Error
    }
    
    // Invalidate cache
    if err := r.DeleteCache(ctx, u.ID); err != nil {
        r.log.Warnw("cache delete error", "error", err)
        // Don't return error - cache will be refreshed on next read
    }
    
    return nil
}

func (r *userRepo) Create(ctx context.Context, u *biz.User) error {
    user := &User{
        Name:  u.Name,
        Email: u.Email,
        Age:   u.Age,
    }
    
    result := r.data.db.WithContext(ctx).Create(user)
    if result.Error != nil {
        return result.Error
    }
    
    u.ID = user.ID
    
    // Pre-warm cache
    cacheKey := fmt.Sprintf(userCacheKey, user.ID)
    r.setCache(context.Background(), cacheKey, u)
    
    return nil
}
```

### Cache Batch Operations

```go
func (r *userRepo) GetMulti(ctx context.Context, ids []int64) ([]*biz.User, error) {
    if len(ids) == 0 {
        return nil, nil
    }
    
    // Build cache keys
    keys := make([]string, len(ids))
    for i, id := range ids {
        keys[i] = fmt.Sprintf(userCacheKey, id)
    }
    
    // MGet from cache
    cached, err := r.data.rdb.MGet(ctx, keys...).Result()
    if err != nil {
        r.log.Warnw("redis mget error", "error", err)
    }
    
    result := make([]*biz.User, 0, len(ids))
    missingIDs := make([]int64, 0)
    
    // Process cached results
    for i, val := range cached {
        if val == nil {
            missingIDs = append(missingIDs, ids[i])
            continue
        }
        
        var user biz.User
        if err := json.Unmarshal([]byte(val.(string)), &user); err == nil {
            result = append(result, &user)
        } else {
            missingIDs = append(missingIDs, ids[i])
        }
    }
    
    // Fetch missing from DB
    if len(missingIDs) > 0 {
        users, err := r.getMultiFromDB(ctx, missingIDs)
        if err != nil {
            return nil, err
        }
        
        result = append(result, users...)
        
        // Update cache
        go r.setMultiCache(context.Background(), users)
    }
    
    return result, nil
}

func (r *userRepo) getMultiFromDB(ctx context.Context, ids []int64) ([]*biz.User, error) {
    var users []User
    result := r.data.db.WithContext(ctx).Find(&users, ids)
    if result.Error != nil {
        return nil, result.Error
    }
    
    result := make([]*biz.User, len(users))
    for i, u := range users {
        result[i] = toBizUser(&u)
    }
    
    return result, nil
}
```

### Cache Decorator Pattern

```go
// internal/data/cache/user_cache.go
package cache

import (
    "context"
    "encoding/json"
    "fmt"
    "time"
    
    "github.com/redis/go-redis/v9"
)

// Cache is a generic cache interface
type Cache[T any] struct {
    client *redis.Client
    ttl    time.Duration
}

func NewCache[T any](client *redis.Client, ttl time.Duration) *Cache[T] {
    return &Cache[T]{
        client: client,
        ttl:    ttl,
    }
}

func (c *Cache[T]) Get(ctx context.Context, key string) (*T, error) {
    data, err := c.client.Get(ctx, key).Result()
    if err != nil {
        return nil, err
    }
    
    var value T
    if err := json.Unmarshal([]byte(data), &value); err != nil {
        return nil, err
    }
    
    return &value, nil
}

func (c *Cache[T]) Set(ctx context.Context, key string, value *T) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    
    return c.client.Set(ctx, key, data, c.ttl).Err()
}

func (c *Cache[T]) Delete(ctx context.Context, key string) error {
    return c.client.Del(ctx, key).Err()
}
```

## Cache Strategy Summary

- Prefer cache-aside for read-heavy entities.
- Define TTL by domain tolerance, not arbitrary constants.
- On writes, update or invalidate cache deterministically.
- Use cache decorator for reusable cache logic.
- Handle cache failures gracefully (don't fail on cache error).
- Consider cache stampede protection for hot keys.
