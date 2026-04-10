# Database Patterns

This guide covers database operations in go-kratos, including GORM, Ent, and caching strategies.

## GORM Integration

### Setup

### ✅ Correct Pattern

```go
// internal/data/data.go
package data

import (
    "github.com/go-kratos/kratos/v2/log"
    "gorm.io/driver/mysql"
    "gorm.io/driver/postgres"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
    
    "user-service/internal/conf"
)

// NewData creates database connection
type Data struct {
    db *gorm.DB
}

func NewData(c *conf.Data, log log.Logger) (*Data, func(), error) {
    // Configure GORM logger
    gormLogger := logger.New(
        log, // Use kratos logger
        logger.Config{
            SlowThreshold:             c.Database.SlowThreshold.AsDuration(),
            LogLevel:                  logger.Info,
            IgnoreRecordNotFoundError: true,
            Colorful:                  false,
        },
    )
    
    // Open connection
    db, err := gorm.Open(mysql.Open(c.Database.Source), &gorm.Config{
        Logger: gormLogger,
    })
    if err != nil {
        return nil, nil, err
    }
    
    // Get underlying sql.DB for connection pool config
    sqlDB, err := db.DB()
    if err != nil {
        return nil, nil, err
    }
    
    // Configure connection pool
    sqlDB.SetMaxIdleConns(int(c.Database.MaxIdleConns))
    sqlDB.SetMaxOpenConns(int(c.Database.MaxOpenConns))
    sqlDB.SetConnMaxLifetime(c.Database.ConnMaxLifetime.AsDuration())
    
    // Auto-migrate (optional, use migrations in production)
    // db.AutoMigrate(&User{}, &Order{})
    
    cleanup := func() {
        sqlDB.Close()
        log.Info("closing database connection")
    }
    
    return &Data{db: db}, cleanup, nil
}
```

### Configuration

```yaml
# configs/config.yaml
data:
  database:
    driver: mysql
    source: user:password@tcp(localhost:3306)/mydb?parseTime=True&loc=Local
    max_idle_conns: 10
    max_open_conns: 100
    conn_max_lifetime: 1h
    slow_threshold: 200ms
```

## CRUD Operations

### ✅ Correct Pattern

```go
// internal/data/user.go
package data

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/log"
    "gorm.io/gorm"
    
    "user-service/internal/biz"
)

type userRepo struct {
    data *Data
    log  *log.Helper
}

// NewUserRepo creates user repository
func NewUserRepo(data *Data, logger log.Logger) biz.UserRepo {
    return &userRepo{
        data: data,
        log:  log.NewHelper(logger),
    }
}

// Create creates a new user
func (r *userRepo) Create(ctx context.Context, u *biz.User) error {
    user := &User{
        Name:     u.Name,
        Email:    u.Email,
        Age:      u.Age,
        Password: u.Password,
        Status:   int32(u.Status),
    }
    
    result := r.data.db.WithContext(ctx).Create(user)
    if result.Error != nil {
        // Check for duplicate key error
        if isDuplicateKeyError(result.Error) {
            return errors.Conflict("DUPLICATE_EMAIL", "email already exists")
        }
        return result.Error
    }
    
    // Populate generated fields
    u.ID = user.ID
    u.CreatedAt = user.CreatedAt
    u.UpdatedAt = user.UpdatedAt
    
    return nil
}

// Get gets user by ID
func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    var user User
    result := r.data.db.WithContext(ctx).First(&user, id)
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrRecordNotFound) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        return nil, result.Error
    }
    
    return toBizUser(&user), nil
}

// GetByEmail gets user by email
func (r *userRepo) GetByEmail(ctx context.Context, email string) (*biz.User, error) {
    var user User
    result := r.data.db.WithContext(ctx).Where("email = ?", email).First(&user)
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrRecordNotFound) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        return nil, result.Error
    }
    
    return toBizUser(&user), nil
}

// Update updates user
func (r *userRepo) Update(ctx context.Context, u *biz.User) error {
    result := r.data.db.WithContext(ctx).Model(&User{}).
        Where("id = ?", u.ID).
        Updates(map[string]interface{}{
            "name":       u.Name,
            "age":        u.Age,
            "status":     int32(u.Status),
            "updated_at": gorm.Expr("NOW()"),
        })
    
    if result.Error != nil {
        return result.Error
    }
    if result.RowsAffected == 0 {
        return errors.NotFound("USER_NOT_FOUND", "user not found")
    }
    
    return nil
}

// Delete deletes user (soft delete)
func (r *userRepo) Delete(ctx context.Context, id int64) error {
    result := r.data.db.WithContext(ctx).Delete(&User{}, id)
    if result.Error != nil {
        return result.Error
    }
    if result.RowsAffected == 0 {
        return errors.NotFound("USER_NOT_FOUND", "user not found")
    }
    return nil
}

// List lists users with pagination
func (r *userRepo) List(ctx context.Context, page, pageSize int32) ([]*biz.User, int32, error) {
    var users []*User
    var total int64
    
    offset := (page - 1) * pageSize
    
    // Get total count
    if err := r.data.db.WithContext(ctx).Model(&User{}).Count(&total).Error; err != nil {
        return nil, 0, err
    }
    
    // Get users with pagination
    result := r.data.db.WithContext(ctx).
        Offset(int(offset)).
        Limit(int(pageSize)).
        Order("created_at DESC").
        Find(&users)
    
    if result.Error != nil {
        return nil, 0, result.Error
    }
    
    // Convert to biz models
    bizUsers := make([]*biz.User, len(users))
    for i, u := range users {
        bizUsers[i] = toBizUser(u)
    }
    
    return bizUsers, int32(total), nil
}

// Search searches users by keyword
func (r *userRepo) Search(ctx context.Context, keyword string, page, pageSize int32) ([]*biz.User, int32, error) {
    var users []*User
    var total int64
    
    offset := (page - 1) * pageSize
    
    query := r.data.db.WithContext(ctx).Model(&User{}).
        Where("name LIKE ? OR email LIKE ?", "%"+keyword+"%", "%"+keyword+"%")
    
    if err := query.Count(&total).Error; err != nil {
        return nil, 0, err
    }
    
    result := query.Offset(int(offset)).Limit(int(pageSize)).Find(&users)
    if result.Error != nil {
        return nil, 0, result.Error
    }
    
    bizUsers := make([]*biz.User, len(users))
    for i, u := range users {
        bizUsers[i] = toBizUser(u)
    }
    
    return bizUsers, int32(total), nil
}

// Helper functions
func toBizUser(u *User) *biz.User {
    return &biz.User{
        ID:        u.ID,
        Name:      u.Name,
        Email:     u.Email,
        Age:       u.Age,
        Password:  u.Password,
        Status:    biz.UserStatus(u.Status),
        CreatedAt: u.CreatedAt,
        UpdatedAt: u.UpdatedAt,
    }
}

func isDuplicateKeyError(err error) bool {
    // Check for MySQL duplicate key error
    if mysqlErr, ok := err.(*mysql.MySQLError); ok {
        return mysqlErr.Number == 1062
    }
    return false
}
```

### ❌ Common Mistakes

```go
// DON'T: Skip context
func (r *userRepo) Get(id int64) (*biz.User, error) {
    // ❌ No context parameter
    var user User
    r.data.db.First(&user, id)  // ❌ No WithContext
    return toBizUser(&user), nil
}

// DON'T: Ignore errors
func (r *userRepo) Create(u *biz.User) error {
    r.data.db.Create(&User{...})  // ❌ Not checking error
    return nil
}

// DON'T: Return data models
func (r *userRepo) Get(ctx context.Context, id int64) (*User, error) {
    // ❌ Returns data model instead of biz model
    var user User
    r.data.db.WithContext(ctx).First(&user, id)
    return &user, nil
}
```

## Transactions

### ✅ Correct Pattern

```go
// internal/data/user.go

// Transaction function type
type TransactionFunc func(ctx context.Context, tx *gorm.DB) error

// ExecInTransaction executes function in transaction
func (r *userRepo) ExecInTransaction(ctx context.Context, fn TransactionFunc) error {
    return r.data.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        return fn(ctx, tx)
    })
}

// CreateUserWithOrders creates user and orders in transaction
func (r *userRepo) CreateUserWithOrders(ctx context.Context, u *biz.User, orders []*biz.Order) error {
    return r.data.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        // Create user
        user := &User{
            Name:     u.Name,
            Email:    u.Email,
            Password: u.Password,
        }
        if err := tx.Create(user).Error; err != nil {
            return err
        }
        u.ID = user.ID
        
        // Create orders
        for _, order := range orders {
            order.UserID = u.ID
            o := &Order{
                UserID: user.ID,
                Amount: order.Amount,
                Status: order.Status,
            }
            if err := tx.Create(o).Error; err != nil {
                return err
            }
            order.ID = o.ID
        }
        
        return nil
    })
}
```

### Transaction in Biz Layer

```go
// internal/biz/user.go

func (uc *UserUsecase) CreateUserWithOrders(ctx context.Context, u *User, orders []*Order) (*User, error) {
    // Validate user
    if err := u.Validate(); err != nil {
        return nil, err
    }
    
    // Validate orders
    for _, order := range orders {
        if order.Amount <= 0 {
            return nil, errors.BadRequest("INVALID_AMOUNT", "order amount must be positive")
        }
    }
    
    // Execute in transaction
    if err := uc.repo.CreateUserWithOrders(ctx, u, orders); err != nil {
        return nil, err
    }
    
    return u, nil
}
```

## Complex Queries

### ✅ Correct Pattern

```go
// internal/data/user.go

// GetUserWithOrders gets user with their orders
func (r *userRepo) GetUserWithOrders(ctx context.Context, id int64) (*biz.User, []*biz.Order, error) {
    var user User
    if err := r.data.db.WithContext(ctx).First(&user, id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        return nil, nil, err
    }
    
    var orders []Order
    if err := r.data.db.WithContext(ctx).Where("user_id = ?", id).Find(&orders).Error; err != nil {
        return nil, nil, err
    }
    
    bizOrders := make([]*biz.Order, len(orders))
    for i, o := range orders {
        bizOrders[i] = &biz.Order{
            ID:     o.ID,
            UserID: o.UserID,
            Amount: o.Amount,
            Status: o.Status,
        }
    }
    
    return toBizUser(&user), bizOrders, nil
}

// GetActiveUsers gets users with active status and recent orders
func (r *userRepo) GetActiveUsers(ctx context.Context, since time.Time) ([]*biz.User, error) {
    var users []User
    
    result := r.data.db.WithContext(ctx).
        Joins("JOIN orders ON orders.user_id = users.id").
        Where("users.status = ?", biz.UserStatusActive).
        Where("orders.created_at > ?", since).
        Group("users.id").
        Having("COUNT(orders.id) > 0").
        Find(&users)
    
    if result.Error != nil {
        return nil, result.Error
    }
    
    bizUsers := make([]*biz.User, len(users))
    for i, u := range users {
        bizUsers[i] = toBizUser(&u)
    }
    
    return bizUsers, nil
}

// BatchCreate creates multiple users in batch
func (r *userRepo) BatchCreate(ctx context.Context, users []*biz.User) error {
    dataUsers := make([]User, len(users))
    for i, u := range users {
        dataUsers[i] = User{
            Name:     u.Name,
            Email:    u.Email,
            Password: u.Password,
            Status:   int32(u.Status),
        }
    }
    
    return r.data.db.WithContext(ctx).CreateInBatches(dataUsers, 100).Error
}
```

## Caching with Redis

### Setup

```go
// internal/data/data.go

import (
    "github.com/redis/go-redis/v9"
)

type Data struct {
    db  *gorm.DB
    rdb *redis.Client
}

func NewData(c *conf.Data, log log.Logger) (*Data, func(), error) {
    // ... database setup ...
    
    // Redis setup
    rdb := redis.NewClient(&redis.Options{
        Addr:         c.Redis.Addr,
        Password:     c.Redis.Password,
        DB:           int(c.Redis.Db),
        PoolSize:     int(c.Redis.PoolSize),
        MinIdleConns: int(c.Redis.MinIdleConns),
    })
    
    cleanup := func() {
        sqlDB.Close()
        rdb.Close()
        log.Info("closing database and redis connections")
    }
    
    return &Data{db: db, rdb: rdb}, cleanup, nil
}
```

### Cache-Aside Pattern

```go
// internal/data/user.go

import (
    "encoding/json"
    "fmt"
    "time"
)

const (
    userCacheKey     = "user:%d"
    userCacheTTL     = 10 * time.Minute
)

// GetWithCache gets user with caching
func (r *userRepo) GetWithCache(ctx context.Context, id int64) (*biz.User, error) {
    cacheKey := fmt.Sprintf(userCacheKey, id)
    
    // Try cache first
    cached, err := r.data.rdb.Get(ctx, cacheKey).Result()
    if err == nil {
        var user biz.User
        if err := json.Unmarshal([]byte(cached), &user); err == nil {
            r.log.Debugf("Cache hit for user %d", id)
            return &user, nil
        }
    }
    
    // Cache miss, get from database
    user, err := r.Get(ctx, id)
    if err != nil {
        return nil, err
    }
    
    // Update cache
    if data, err := json.Marshal(user); err == nil {
        r.data.rdb.Set(ctx, cacheKey, data, userCacheTTL)
    }
    
    return user, nil
}

// UpdateWithCache updates user and invalidates cache
func (r *userRepo) UpdateWithCache(ctx context.Context, u *biz.User) error {
    if err := r.Update(ctx, u); err != nil {
        return err
    }
    
    // Invalidate cache
    cacheKey := fmt.Sprintf(userCacheKey, u.ID)
    r.data.rdb.Del(ctx, cacheKey)
    
    return nil
}

// DeleteWithCache deletes user and invalidates cache
func (r *userRepo) DeleteWithCache(ctx context.Context, id int64) error {
    if err := r.Delete(ctx, id); err != nil {
        return err
    }
    
    // Invalidate cache
    cacheKey := fmt.Sprintf(userCacheKey, id)
    r.data.rdb.Del(ctx, cacheKey)
    
    return nil
}
```

## Ent Integration (Alternative to GORM)

### Setup

```go
// internal/data/ent/client.go

package data

import (
    "user-service/internal/data/ent"
    _ "user-service/internal/data/ent/runtime"
)

func NewEntClient(c *conf.Data) (*ent.Client, error) {
    client, err := ent.Open(c.Database.Driver, c.Database.Source)
    if err != nil {
        return nil, err
    }
    
    // Run auto-migration
    // ctx := context.Background()
    // if err := client.Schema.Create(ctx); err != nil {
    //     return nil, err
    // }
    
    return client, nil
}
```

### Repository with Ent

```go
// internal/data/user.go

import (
    "user-service/internal/data/ent"
    "user-service/internal/data/ent/user"
)

type userRepo struct {
    data *Data
    client *ent.Client
    log  *log.Helper
}

func (r *userRepo) Create(ctx context.Context, u *biz.User) error {
    created, err := r.client.User.Create().
        SetName(u.Name).
        SetEmail(u.Email).
        SetAge(int(u.Age)).
        SetPassword(u.Password).
        Save(ctx)
    
    if err != nil {
        if ent.IsConstraintError(err) {
            return errors.Conflict("DUPLICATE_EMAIL", "email already exists")
        }
        return err
    }
    
    u.ID = created.ID
    u.CreatedAt = created.CreatedAt
    u.UpdatedAt = created.UpdatedAt
    
    return nil
}

func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    u, err := r.client.User.Get(ctx, id)
    if err != nil {
        if ent.IsNotFound(err) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        return nil, err
    }
    
    return &biz.User{
        ID:        u.ID,
        Name:      u.Name,
        Email:     u.Email,
        Age:       int32(u.Age),
        Status:    biz.UserStatus(u.Status),
        CreatedAt: u.CreatedAt,
        UpdatedAt: u.UpdatedAt,
    }, nil
}
```

## Best Practices

### 1. Always Use Context

```go
// ✅ Correct
func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    result := r.data.db.WithContext(ctx).First(&user, id)
    // ...
}

// ❌ Incorrect
func (r *userRepo) Get(id int64) (*biz.User, error) {
    result := r.data.db.First(&user, id)  // No context!
    // ...
}
```

### 2. Proper Error Handling

```go
// ✅ Correct
func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    result := r.data.db.WithContext(ctx).First(&user, id)
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrRecordNotFound) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        return nil, result.Error
    }
    return toBizUser(&user), nil
}

// ❌ Incorrect
func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    r.data.db.WithContext(ctx).First(&user, id)
    return toBizUser(&user), nil  // Ignores errors!
}
```

### 3. Model Conversion

```go
// ✅ Correct - Convert between data and biz models
func toBizUser(u *User) *biz.User {
    return &biz.User{
        ID:   u.ID,
        Name: u.Name,
        // ...
    }
}

// ❌ Incorrect - Return data model directly
func (r *userRepo) Get(ctx context.Context, id int64) (*User, error) {
    var user User
    r.data.db.WithContext(ctx).First(&user, id)
    return &user, nil  // Returns data model!
}
```

### 4. Pagination

```go
// ✅ Correct
func (r *userRepo) List(ctx context.Context, page, pageSize int32) ([]*biz.User, int32, error) {
    var total int64
    r.data.db.WithContext(ctx).Model(&User{}).Count(&total)
    
    offset := (page - 1) * pageSize
    r.data.db.WithContext(ctx).Offset(int(offset)).Limit(int(pageSize)).Find(&users)
    
    return bizUsers, int32(total), nil
}

// ❌ Incorrect - No pagination limits
func (r *userRepo) List(ctx context.Context) ([]*biz.User, error) {
    r.data.db.WithContext(ctx).Find(&users)  // Could return millions!
    return bizUsers, nil
}
```

## Summary

### ✅ Always Follow

- Use context for all database operations
- Convert between data and biz models
- Handle errors properly (especially NotFound)
- Use transactions for multi-table operations
- Implement pagination for list queries
- Use caching for frequently accessed data
- Configure connection pooling

### ❌ Never Do

- Skip context propagation
- Return data models from repository
- Ignore database errors
- Load all records without pagination
- Skip transaction for related operations
- Hard-code database configuration
