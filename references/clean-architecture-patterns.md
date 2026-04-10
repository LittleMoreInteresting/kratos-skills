# Clean Architecture Patterns

This guide covers clean architecture implementation in go-kratos (Transport → Service → Biz → Data).

## Core Principles

### Four-Layer Architecture

```
┌─────────────────────────────────────────┐
│           Transport Layer               │
│    (HTTP/gRPC server, middleware)       │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│           Service Layer                 │
│   (Request handling, validation)        │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│             Biz Layer                   │
│  (Business logic, use cases, domain)    │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│            Data Layer                   │
│   (Database, cache, external APIs)      │
└─────────────────────────────────────────┘
```

### Dependency Rule

Dependencies always point inward:
- Transport depends on Service
- Service depends on Biz
- Biz depends on Data (via interfaces)
- Data has no outward dependencies

## Biz Layer

### Domain Models

### ✅ Correct Pattern

```go
// internal/biz/user.go
package biz

import (
    "context"
    "time"
    
    "github.com/go-kratos/kratos/v2/log"
)

// User is a domain model
type User struct {
    ID        int64
    Name      string
    Email     string
    Age       int32
    Password  string // hashed
    Status    UserStatus
    CreatedAt time.Time
    UpdatedAt time.Time
}

// UserStatus represents user status
type UserStatus int32

const (
    UserStatusInactive UserStatus = 0
    UserStatusActive   UserStatus = 1
    UserStatusBanned   UserStatus = 2
)

// Validate performs domain validation
func (u *User) Validate() error {
    if u.Name == "" {
        return errors.BadRequest("INVALID_NAME", "name is required")
    }
    if len(u.Name) > 50 {
        return errors.BadRequest("NAME_TOO_LONG", "name must be less than 50 characters")
    }
    if u.Age < 18 || u.Age > 120 {
        return errors.BadRequest("INVALID_AGE", "age must be between 18 and 120")
    }
    return nil
}

// CanLogin checks if user can login
func (u *User) CanLogin() bool {
    return u.Status == UserStatusActive
}
```

### Repository Interfaces

```go
// internal/biz/user.go

// UserRepo defines user repository interface
type UserRepo interface {
    // Create creates a new user
    Create(ctx context.Context, u *User) error
    
    // Get gets user by ID
    Get(ctx context.Context, id int64) (*User, error)
    
    // GetByEmail gets user by email
    GetByEmail(ctx context.Context, email string) (*User, error)
    
    // Update updates user
    Update(ctx context.Context, u *User) error
    
    // Delete deletes user
    Delete(ctx context.Context, id int64) error
    
    // List lists users with pagination
    List(ctx context.Context, page, pageSize int32) ([]*User, int32, error)
    
    // UpdateStatus updates user status
    UpdateStatus(ctx context.Context, id int64, status UserStatus) error
}
```

### Usecase Implementation

```go
// internal/biz/user.go

// UserUsecase handles user business logic
type UserUsecase struct {
    repo UserRepo
    log  *log.Helper
}

// NewUserUsecase creates a new UserUsecase
func NewUserUsecase(repo UserRepo, logger log.Logger) *UserUsecase {
    return &UserUsecase{
        repo: repo,
        log:  log.NewHelper(logger),
    }
}

// Create creates a new user
func (uc *UserUsecase) Create(ctx context.Context, u *User) (*User, error) {
    // 1. Domain validation
    if err := u.Validate(); err != nil {
        return nil, err
    }
    
    // 2. Check if email exists
    existing, err := uc.repo.GetByEmail(ctx, u.Email)
    if err != nil && !errors.IsNotFound(err) {
        return nil, err
    }
    if existing != nil {
        return nil, errors.Conflict("EMAIL_EXISTS", "email already exists")
    }
    
    // 3. Hash password
    hashedPassword, err := hashPassword(u.Password)
    if err != nil {
        return nil, errors.InternalServer("HASH_ERROR", "failed to hash password")
    }
    u.Password = hashedPassword
    
    // 4. Set defaults
    u.Status = UserStatusActive
    u.CreatedAt = time.Now()
    u.UpdatedAt = time.Now()
    
    // 5. Save to repository
    if err := uc.repo.Create(ctx, u); err != nil {
        return nil, err
    }
    
    uc.log.Infof("User created: id=%d, email=%s", u.ID, u.Email)
    
    return u, nil
}

// Get gets user by ID
func (uc *UserUsecase) Get(ctx context.Context, id int64) (*User, error) {
    u, err := uc.repo.Get(ctx, id)
    if err != nil {
        if errors.IsNotFound(err) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        return nil, err
    }
    return u, nil
}

// Update updates user
func (uc *UserUsecase) Update(ctx context.Context, id int64, name string, age int32) (*User, error) {
    // 1. Get existing user
    u, err := uc.repo.Get(ctx, id)
    if err != nil {
        return nil, err
    }
    
    // 2. Update fields
    u.Name = name
    u.Age = age
    u.UpdatedAt = time.Now()
    
    // 3. Validate
    if err := u.Validate(); err != nil {
        return nil, err
    }
    
    // 4. Save
    if err := uc.repo.Update(ctx, u); err != nil {
        return nil, err
    }
    
    return u, nil
}

// Delete deletes user
func (uc *UserUsecase) Delete(ctx context.Context, id int64) error {
    // Check if user exists
    _, err := uc.repo.Get(ctx, id)
    if err != nil {
        return err
    }
    
    return uc.repo.Delete(ctx, id)
}

// List lists users with pagination
func (uc *UserUsecase) List(ctx context.Context, page, pageSize int32) ([]*User, int32, error) {
    return uc.repo.List(ctx, page, pageSize)
}

// helper function
func hashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}
```

### ❌ Common Mistakes

```go
// DON'T: Put transport concerns in biz layer
func (uc *UserUsecase) Create(ctx context.Context, u *User) (*User, error) {
    // ❌ HTTP-specific logic doesn't belong here
    if ctx.Value("http_header") != nil {
        // ...
    }
    // ...
}

// DON'T: Skip validation
func (uc *UserUsecase) Create(ctx context.Context, u *User) (*User, error) {
    // ❌ No validation
    return uc.repo.Create(ctx, u)
}

// DON'T: Return transport-specific errors
func (uc *UserUsecase) Get(ctx context.Context, id int64) (*User, error) {
    u, err := uc.repo.Get(ctx, id)
    if err != nil {
        // ❌ Don't return HTTP status codes
        return nil, fmt.Errorf("HTTP 404: user not found")
    }
    return u, nil
}
```

## Data Layer

### Repository Implementation

### ✅ Correct Pattern

```go
// internal/data/user.go
package data

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/log"
    "gorm.io/gorm"
    
    "user-service/internal/biz"
)

type userRepo struct {
    data *Data
    log  *log.Helper
}

// NewUserRepo creates a new user repository
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
        return result.Error
    }
    
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
    result := r.data.db.WithContext(ctx).Model(&User{}).Where("id = ?", u.ID).Updates(map[string]interface{}{
        "name":       u.Name,
        "age":        u.Age,
        "status":     int32(u.Status),
        "updated_at": u.UpdatedAt,
    })
    if result.Error != nil {
        return result.Error
    }
    if result.RowsAffected == 0 {
        return errors.NotFound("USER_NOT_FOUND", "user not found")
    }
    return nil
}

// Delete deletes user
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
    result := r.data.db.WithContext(ctx).Model(&User{}).Count(&total)
    if result.Error != nil {
        return nil, 0, result.Error
    }
    
    // Get users
    result = r.data.db.WithContext(ctx).Offset(int(offset)).Limit(int(pageSize)).Find(&users)
    if result.Error != nil {
        return nil, 0, result.Error
    }
    
    bizUsers := make([]*biz.User, len(users))
    for i, u := range users {
        bizUsers[i] = toBizUser(u)
    }
    
    return bizUsers, int32(total), nil
}

// UpdateStatus updates user status
func (r *userRepo) UpdateStatus(ctx context.Context, id int64, status biz.UserStatus) error {
    result := r.data.db.WithContext(ctx).Model(&User{}).Where("id = ?", id).Update("status", int32(status))
    if result.Error != nil {
        return result.Error
    }
    if result.RowsAffected == 0 {
        return errors.NotFound("USER_NOT_FOUND", "user not found")
    }
    return nil
}

// Helper function to convert data model to biz model
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
```

### Data Model

```go
// internal/data/user.go

// User is the database model
type User struct {
    ID        int64     `gorm:"primaryKey"`
    Name      string    `gorm:"size:50;not null"`
    Email     string    `gorm:"size:100;uniqueIndex;not null"`
    Age       int32     `gorm:"not null"`
    Password  string    `gorm:"size:255;not null"`
    Status    int32     `gorm:"default:1"`
    CreatedAt time.Time `gorm:"autoCreateTime"`
    UpdatedAt time.Time `gorm:"autoUpdateTime"`
}

// TableName returns table name
func (User) TableName() string {
    return "users"
}
```

### Data Provider

```go
// internal/data/data.go
package data

import (
    "github.com/go-kratos/kratos/v2/log"
    "github.com/google/wire"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    
    "user-service/internal/conf"
)

// ProviderSet is data providers
var ProviderSet = wire.NewSet(
    NewData,
    NewUserRepo,
    // Add more repositories here
)

// Data contains database and cache connections
type Data struct {
    db *gorm.DB
}

// NewData creates a new Data
func NewData(c *conf.Data, logger log.Logger) (*Data, func(), error) {
    log := log.NewHelper(logger)
    
    db, err := gorm.Open(mysql.Open(c.Database.Source), &gorm.Config{})
    if err != nil {
        return nil, nil, err
    }
    
    sqlDB, err := db.DB()
    if err != nil {
        return nil, nil, err
    }
    
    // Configure connection pool
    sqlDB.SetMaxIdleConns(int(c.Database.MaxIdleConns))
    sqlDB.SetMaxOpenConns(int(c.Database.MaxOpenConns))
    sqlDB.SetConnMaxLifetime(c.Database.ConnMaxLifetime.AsDuration())
    
    cleanup := func() {
        sqlDB.Close()
        log.Info("closing database connection")
    }
    
    return &Data{db: db}, cleanup, nil
}
```

### ❌ Common Mistakes

```go
// DON'T: Return data models from repository
func (r *userRepo) Get(ctx context.Context, id int64) (*User, error) {
    // ❌ Returns data model instead of biz model
    var user User
    r.data.db.First(&user, id)
    return &user, nil
}

// DON'T: Skip error handling
func (r *userRepo) Create(ctx context.Context, u *biz.User) error {
    // ❌ No error handling
    r.data.db.Create(&User{...})
    return nil
}

// DON'T: Use context.Background()
func (r *userRepo) Get(ctx context.Context, id int64) (*biz.User, error) {
    // ❌ Not using passed context
    result := r.data.db.WithContext(context.Background()).First(&user, id)
    // ...
}
```

## Dependency Injection with Wire

### Provider Functions

```go
// internal/biz/biz.go
package biz

import "github.com/google/wire"

// ProviderSet is biz providers
var ProviderSet = wire.NewSet(
    NewUserUsecase,
    // Add more usecases here
)
```

```go
// internal/service/service.go
package service

import "github.com/google/wire"

// ProviderSet is service providers
var ProviderSet = wire.NewSet(
    NewUserService,
    // Add more services here
)
```

### Wire Configuration

```go
// internal/server/server.go
package server

import (
    "github.com/google/wire"
)

// ProviderSet is server providers
var ProviderSet = wire.NewSet(
    NewGRPCServer,
    NewHTTPServer,
)
```

```go
// internal/wire.go
//go:build wireinject
// +build wireinject

package internal

import (
    "github.com/go-kratos/kratos/v2"
    "github.com/go-kratos/kratos/v2/log"
    "github.com/google/wire"
    
    "user-service/internal/biz"
    "user-service/internal/conf"
    "user-service/internal/data"
    "user-service/internal/server"
    "user-service/internal/service"
)

// wireApp initializes the application
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

```go
// internal/app.go
package internal

import (
    "github.com/go-kratos/kratos/v2"
    "github.com/go-kratos/kratos/v2/log"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    "github.com/go-kratos/kratos/v2/transport/http"
)

// newApp creates a new kratos application
func newApp(logger log.Logger, gs *grpc.Server, hs *http.Server) *kratos.App {
    return kratos.New(
        kratos.Name("user-service"),
        kratos.Version("v1.0.0"),
        kratos.Logger(logger),
        kratos.Server(
            gs,
            hs,
        ),
    )
}
```

## Testing

### Biz Layer Tests

```go
// internal/biz/user_test.go
package biz

import (
    "context"
    "testing"
    
    "github.com/go-kratos/kratos/v2/log"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

type MockUserRepo struct {
    mock.Mock
}

func (m *MockUserRepo) Create(ctx context.Context, u *User) error {
    args := m.Called(ctx, u)
    return args.Error(0)
}

func (m *MockUserRepo) Get(ctx context.Context, id int64) (*User, error) {
    args := m.Called(ctx, id)
    return args.Get(0).(*User), args.Error(1)
}

// ... implement other mock methods

func TestUserUsecase_Create(t *testing.T) {
    mockRepo := new(MockUserRepo)
    uc := NewUserUsecase(mockRepo, log.DefaultLogger)
    
    tests := []struct {
        name    string
        user    *User
        mock    func()
        wantErr bool
    }{
        {
            name: "success",
            user: &User{
                Name:     "John",
                Email:    "john@example.com",
                Age:      25,
                Password: "password123",
            },
            mock: func() {
                mockRepo.On("GetByEmail", mock.Anything, "john@example.com").
                    Return(nil, errors.NotFound("NOT_FOUND", ""))
                mockRepo.On("Create", mock.Anything, mock.Anything).
                    Return(nil)
            },
            wantErr: false,
        },
        {
            name: "email_exists",
            user: &User{
                Name:     "John",
                Email:    "john@example.com",
                Age:      25,
                Password: "password123",
            },
            mock: func() {
                mockRepo.On("GetByEmail", mock.Anything, "john@example.com").
                    Return(&User{ID: 1}, nil)
            },
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tt.mock()
            _, err := uc.Create(context.Background(), tt.user)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
            }
            mockRepo.AssertExpectations(t)
        })
    }
}
```

## Summary

### ✅ Always Follow

- Define domain models in Biz layer
- Use interfaces for repository abstraction
- Implement business logic in usecases
- Convert between data and biz models
- Use dependency injection with Wire
- Pass context through all layers
- Write unit tests for Biz layer

### ❌ Never Do

- Mix transport concerns with business logic
- Skip repository interfaces
- Return data models from Biz layer
- Use context.Background()
- Skip validation in Biz layer
- Tight coupling between layers
- Skip error handling
