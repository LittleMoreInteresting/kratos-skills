# Service Registry Patterns (go-kratos)

This guide covers service registration and discovery patterns in go-kratos.

## Overview

- Support multiple registries: etcd, consul, nacos, eureka, zookeeper
- Automatic service registration on startup
- Automatic service deregistration on shutdown
- Client-side service discovery with load balancing

## Supported Registries

| Registry | Package | Use Case |
|----------|---------|----------|
| etcd | `github.com/go-kratos/kratos/contrib/registry/etcd/v2` | Kubernetes native |
| Consul | `github.com/go-kratos/kratos/contrib/registry/consul/v2` | HashiCorp stack |
| Nacos | `github.com/go-kratos/kratos/contrib/registry/nacos/v2` | Alibaba Cloud |
| Eureka | `github.com/go-kratos/kratos/contrib/registry/eureka/v2` | Spring Cloud |
| Zookeeper | `github.com/go-kratos/kratos/contrib/registry/zookeeper/v2` | Apache ecosystem |

---

## etcd Registry

### Dependencies

```bash
go get github.com/go-kratos/kratos/contrib/registry/etcd/v2
go get go.etcd.io/etcd/client/v3
```

### Server Configuration

```go
// internal/server/registry.go
package server

import (
    "github.com/go-kratos/kratos/contrib/registry/etcd/v2"
    "github.com/go-kratos/kratos/v2/registry"
    etcdclient "go.etcd.io/etcd/client/v3"
)

func NewRegistrar() (registry.Registrar, func(), error) {
    // Create etcd client
    client, err := etcdclient.New(etcdclient.Config{
        Endpoints: []string{"127.0.0.1:2379"},
        // For production, use TLS
        // TLS: &tls.Config{...}
    })
    if err != nil {
        return nil, nil, err
    }
    
    // Create etcd registry
    r := etcd.New(client)
    
    cleanup := func() {
        client.Close()
    }
    
    return r, cleanup, nil
}
```

```go
// cmd/server/main.go
package main

import (
    "github.com/go-kratos/kratos/v2"
)

func main() {
    // ... create servers ...
    
    registrar, cleanup, err := server.NewRegistrar()
    if err != nil {
        log.Fatal(err)
    }
    defer cleanup()
    
    app := kratos.New(
        kratos.Name("user-service"),
        kratos.Version("v1.0.0"),
        kratos.Server(httpSrv, grpcSrv),
        kratos.Registrar(registrar),  // Register service
    )
    
    if err := app.Run(); err != nil {
        log.Fatal(err)
    }
}
```

### Client Configuration

```go
// internal/data/client.go
package data

import (
    "context"
    
    "github.com/go-kratos/kratos/contrib/registry/etcd/v2"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    etcdclient "go.etcd.io/etcd/client/v3"
)

func NewGreeterClient() (*grpc.ClientConn, error) {
    // Create etcd client
    client, err := etcdclient.New(etcdclient.Config{
        Endpoints: []string{"127.0.0.1:2379"},
    })
    if err != nil {
        return nil, err
    }
    
    // Create discovery
    r := etcd.New(client)
    
    // Create gRPC client with discovery
    conn, err := grpc.DialInsecure(
        context.Background(),
        grpc.WithEndpoint("discovery:///greeter-service"),
        grpc.WithDiscovery(r),
    )
    if err != nil {
        return nil, err
    }
    
    return conn, nil
}
```

---

## Consul Registry

### Dependencies

```bash
go get github.com/go-kratos/kratos/contrib/registry/consul/v2
go get github.com/hashicorp/consul/api
```

### Server Configuration

```go
// internal/server/registry.go
package server

import (
    "github.com/go-kratos/kratos/contrib/registry/consul/v2"
    "github.com/go-kratos/kratos/v2/registry"
    "github.com/hashicorp/consul/api"
)

func NewRegistrar() (registry.Registrar, func(), error) {
    // Create Consul client
    consulClient, err := api.NewClient(api.DefaultConfig())
    if err != nil {
        return nil, nil, err
    }
    
    // Create Consul registry
    r := consul.New(consulClient)
    
    return r, func() {}, nil
}
```

### Server with Custom Configuration

```go
// internal/server/registry.go
package server

import (
    "github.com/go-kratos/kratos/contrib/registry/consul/v2"
    "github.com/hashicorp/consul/api"
)

func NewRegistrarWithConfig() (registry.Registrar, error) {
    config := api.DefaultConfig()
    config.Address = "consul-server:8500"
    config.Datacenter = "dc1"
    config.Token = "your-consul-token"
    
    client, err := api.NewClient(config)
    if err != nil {
        return nil, err
    }
    
    // Create registry with custom options
    r := consul.New(client,
        consul.WithHeartbeat(true),
        consul.WithHealthCheck(true),
    )
    
    return r, nil
}
```

### Client Configuration

```go
// internal/data/client.go
package data

import (
    "context"
    
    "github.com/go-kratos/kratos/contrib/registry/consul/v2"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    "github.com/hashicorp/consul/api"
)

func NewOrderClient() (*grpc.ClientConn, error) {
    consulClient, err := api.NewClient(api.DefaultConfig())
    if err != nil {
        return nil, err
    }
    
    r := consul.New(consulClient)
    
    conn, err := grpc.DialInsecure(
        context.Background(),
        grpc.WithEndpoint("discovery:///order-service"),
        grpc.WithDiscovery(r),
        grpc.WithTimeout(5*time.Second),
    )
    if err != nil {
        return nil, err
    }
    
    return conn, nil
}
```

---

## Nacos Registry

### Dependencies

```bash
go get github.com/go-kratos/kratos/contrib/registry/nacos/v2
go get github.com/nacos-group/nacos-sdk-go/v2
```

### Server Configuration

```go
// internal/server/registry.go
package server

import (
    "github.com/go-kratos/kratos/contrib/registry/nacos/v2"
    "github.com/go-kratos/kratos/v2/registry"
    "github.com/nacos-group/nacos-sdk-go/v2/common/constant"
    "github.com/nacos-group/nacos-sdk-go/v2/vo"
)

func NewRegistrar() (registry.Registrar, func(), error) {
    // Create Nacos client config
    clientConfig := constant.ClientConfig{
        NamespaceId:         "public",
        TimeoutMs:           5000,
        NotLoadCacheAtStart: true,
        LogDir:              "/tmp/nacos/log",
        CacheDir:            "/tmp/nacos/cache",
    }
    
    // Create Nacos server config
    serverConfigs := []constant.ServerConfig{
        {
            IpAddr: "127.0.0.1",
            Port:   8848,
        },
    }
    
    // Create Nacos registry
    r, err := nacos.New(clientConfig, serverConfigs)
    if err != nil {
        return nil, nil, err
    }
    
    return r, func() {}, nil
}
```

### Client Configuration

```go
// internal/data/client.go
package data

import (
    "context"
    
    "github.com/go-kratos/kratos/contrib/registry/nacos/v2"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    "github.com/nacos-group/nacos-sdk-go/v2/common/constant"
)

func NewPaymentClient() (*grpc.ClientConn, error) {
    clientConfig := constant.ClientConfig{
        NamespaceId: "public",
    }
    
    serverConfigs := []constant.ServerConfig{
        {IpAddr: "127.0.0.1", Port: 8848},
    }
    
    r, err := nacos.New(clientConfig, serverConfigs)
    if err != nil {
        return nil, err
    }
    
    conn, err := grpc.DialInsecure(
        context.Background(),
        grpc.WithEndpoint("discovery:///payment-service"),
        grpc.WithDiscovery(r),
    )
    if err != nil {
        return nil, err
    }
    
    return conn, nil
}
```

---

## Complete Integration Example

### Wire Configuration

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

func wireApp(*conf.Server, *conf.Data, *conf.Registry, log.Logger) (*kratos.App, func(), error) {
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
// internal/server/server.go
package server

import (
    "github.com/go-kratos/kratos/contrib/registry/etcd/v2"
    "github.com/go-kratos/kratos/v2/registry"
    "github.com/google/wire"
    etcdclient "go.etcd.io/etcd/client/v3"
    
    "user-service/internal/conf"
)

var ProviderSet = wire.NewSet(
    NewRegistrar,
    NewGRPCServer,
    NewHTTPServer,
)

// NewRegistrar creates a registry based on configuration
func NewRegistrar(c *conf.Registry) (registry.Registrar, func(), error) {
    switch c.Type {
    case "etcd":
        return newEtcdRegistrar(c.Etcd)
    case "consul":
        return newConsulRegistrar(c.Consul)
    case "nacos":
        return newNacosRegistrar(c.Nacos)
    default:
        return nil, nil, fmt.Errorf("unsupported registry type: %s", c.Type)
    }
}

func newEtcdRegistrar(cfg *conf.Etcd) (registry.Registrar, func(), error) {
    client, err := etcdclient.New(etcdclient.Config{
        Endpoints: cfg.Endpoints,
    })
    if err != nil {
        return nil, nil, err
    }
    
    r := etcd.New(client)
    return r, func() { client.Close() }, nil
}
```

### Configuration

```protobuf
// internal/conf/conf.proto
syntax = "proto3";

package internal.conf;

message Bootstrap {
    Server server = 1;
    Data data = 2;
    Registry registry = 3;
}

message Registry {
    string type = 1;  // etcd, consul, nacos
    Etcd etcd = 2;
    Consul consul = 3;
    Nacos nacos = 4;
}

message Etcd {
    repeated string endpoints = 1;
}

message Consul {
    string address = 1;
    string datacenter = 2;
    string token = 3;
}

message Nacos {
    repeated string endpoints = 1;
    string namespace = 2;
}
```

```yaml
# configs/config.yaml
registry:
  type: etcd
  etcd:
    endpoints:
      - 127.0.0.1:2379
  # consul:
  #   address: 127.0.0.1:8500
  #   datacenter: dc1
  # nacos:
  #   endpoints:
  #     - 127.0.0.1:8848
```

---

## Service Instance Configuration

### Custom Metadata

```go
// cmd/server/main.go
package main

import (
    "github.com/go-kratos/kratos/v2/registry"
)

func main() {
    // ... create servers ...
    
    registrar, cleanup, err := server.NewRegistrar()
    if err != nil {
        log.Fatal(err)
    }
    defer cleanup()
    
    app := kratos.New(
        kratos.Name("user-service"),
        kratos.Version("v1.0.0"),
        kratos.Metadata(map[string]string{
            "region": "us-west-1",
            "zone":   "a",
            "weight": "100",
        }),
        kratos.Server(httpSrv, grpcSrv),
        kratos.Registrar(registrar),
    )
    
    if err := app.Run(); err != nil {
        log.Fatal(err)
    }
}
```

### Service Instance Options

```go
// internal/server/registry.go
package server

import (
    "github.com/go-kratos/kratos/contrib/registry/etcd/v2"
    "github.com/go-kratos/kratos/v2/registry"
    etcdclient "go.etcd.io/etcd/client/v3"
)

func NewRegistrarWithOptions() (registry.Registrar, func(), error) {
    client, err := etcdclient.New(etcdclient.Config{
        Endpoints: []string{"127.0.0.1:2379"},
    })
    if err != nil {
        return nil, nil, err
    }
    
    // Create registry with custom options
    r := etcd.New(client,
        etcd.WithMaxRetry(5),
        etcd.WithHeartbeat(10),  // Heartbeat interval in seconds
    )
    
    return r, func() { client.Close() }, nil
}
```

---

## Load Balancing

### Client Configuration

```go
// internal/data/client.go
package data

import (
    "context"
    
    "github.com/go-kratos/kratos/contrib/registry/etcd/v2"
    "github.com/go-kratos/kratos/v2/selector"
    "github.com/go-kratos/kratos/v2/selector/random"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    etcdclient "go.etcd.io/etcd/client/v3"
)

func NewClientWithLoadBalancer() (*grpc.ClientConn, error) {
    client, err := etcdclient.New(etcdclient.Config{
        Endpoints: []string{"127.0.0.1:2379"},
    })
    if err != nil {
        return nil, err
    }
    
    r := etcd.New(client)
    
    // Set global selector (random, wrr, p2c)
    selector.SetGlobalSelector(random.New())
    
    conn, err := grpc.DialInsecure(
        context.Background(),
        grpc.WithEndpoint("discovery:///order-service"),
        grpc.WithDiscovery(r),
        grpc.WithSelector(random.New()),  // Or use specific selector
    )
    if err != nil {
        return nil, err
    }
    
    return conn, nil
}
```

### Load Balancer Types

```go
import (
    "github.com/go-kratos/kratos/v2/selector/p2c"      // Power of Two Choices
    "github.com/go-kratos/kratos/v2/selector/random"   // Random
    "github.com/go-kratos/kratos/v2/selector/wrr"      // Weighted Round Robin
)

// P2C (recommended for production)
selector.SetGlobalSelector(p2c.New())

// Random
selector.SetGlobalSelector(random.New())

// Weighted Round Robin
selector.SetGlobalSelector(wrr.New())
```

---

## Health Checks

### gRPC Health Check

```go
// internal/server/grpc.go
package server

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/transport/grpc"
    "google.golang.org/grpc/health"
    "google.golang.org/grpc/health/grpc_health_v1"
)

func NewGRPCServer(c *conf.Server, user *service.UserService, logger log.Logger) *grpc.Server {
    var opts = []grpc.ServerOption{
        grpc.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
        ),
    }
    
    srv := grpc.NewServer(opts...)
    v1.RegisterUserServer(srv, user)
    
    // Register health check
    hs := health.NewServer()
    hs.SetServingStatus("user-service", grpc_health_v1.HealthCheckResponse_SERVING)
    grpc_health_v1.RegisterHealthServer(srv, hs)
    
    return srv
}
```

---

## Testing

### Mock Registry

```go
// internal/test/registry.go
package test

import (
    "context"
    
    "github.com/go-kratos/kratos/v2/registry"
)

type MockRegistry struct {
    services map[string][]*registry.ServiceInstance
}

func NewMockRegistry() *MockRegistry {
    return &MockRegistry{
        services: make(map[string][]*registry.ServiceInstance),
    }
}

func (m *MockRegistry) Register(ctx context.Context, service *registry.ServiceInstance) error {
    m.services[service.Name] = append(m.services[service.Name], service)
    return nil
}

func (m *MockRegistry) Deregister(ctx context.Context, service *registry.ServiceInstance) error {
    // Remove from registry
    return nil
}

func (m *MockRegistry) GetService(ctx context.Context, name string) ([]*registry.ServiceInstance, error) {
    return m.services[name], nil
}

func (m *MockRegistry) Watch(ctx context.Context, name string) (registry.Watcher, error) {
    return &mockWatcher{}, nil
}

type mockWatcher struct{}

func (w *mockWatcher) Next() ([]*registry.ServiceInstance, error) {
    return nil, nil
}

func (w *mockWatcher) Stop() error {
    return nil
}
```

---

## Best Practices

### ✅ Always Follow

- Use service discovery for inter-service communication
- Set appropriate heartbeat intervals
- Register health checks
- Use consistent service naming conventions
- Set service metadata for routing
- Handle registry failures gracefully

### ❌ Never Do

```go
// DON'T: Hard-code service addresses
conn, err := grpc.Dial("127.0.0.1:9000")  // ❌
conn, err := grpc.DialInsecure(
    ctx,
    grpc.WithEndpoint("discovery:///service"),  // ✅
    grpc.WithDiscovery(r),
)

// DON'T: Skip error handling for registry
r, _ := etcd.New(client)  // ❌
r, err := etcd.New(client)  // ✅
if err != nil {
    return err
}

// DON'T: Forget to close registry client
defer cleanup()  // ✅
```

---

## Troubleshooting

### Service Not Found

1. Check if service is registered
2. Verify service name matches
3. Check network connectivity to registry
4. Verify registry configuration

### Connection Issues

1. Check registry health
2. Verify service instances are healthy
3. Check for network partitions
4. Verify load balancer configuration
