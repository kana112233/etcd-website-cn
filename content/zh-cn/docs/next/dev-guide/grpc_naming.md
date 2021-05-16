---
title: gRPC naming and discovery
weight: 3500
description: "go-grpc: for resolving gRPC endpoints with an etcd backend"
---

etcd提供了一个gRPC解析器来支持备用名称系统，该名称系统从etcd中发现gRPC服务。 基本机制是基于watch对以服务名称为前缀的键的更新。

## Using etcd discovery with go-grpc 在gro-grpc使用etcd

etcd提供了一个gRPC解析器，用于解析使用etcd做后端的grpc访问点。使用etcd客户端初始化解析器：
The etcd client provides a gRPC resolver for resolving gRPC endpoints with an etcd backend. The resolver is initialized with an etcd client:

```go
import (
	"go.etcd.io/etcd/v3/clientv3"
	resolver "go.etcd.io/etcd/client/v3/naming/resolver"

	"google.golang.org/grpc"
)

cli, cerr := clientv3.NewFromURL("http://localhost:2379")
etcdResolver, err := resolver.NewBuilder(cli);
conn, gerr := grpc.Dial("etcd:///foo/bar/my-service", grpc.WithResolvers(etcdResolver))
```

## Managing service endpoints 管理服务访问点

etcd解析器将解析目标（例如"foo/bar/my-service/"）前缀下的所有keys，用JSON编码的values是潜在的服务访问点。
创建新的keys来添加服务访问点，通过删除keys来删除服务访问点。（help wanted）
The etcd resolver treats all keys under the prefix of the resolution target following a "/" (e.g., "foo/bar/my-service/")，
with JSON-encoded (historically go-grpc `naming.Update`) values as potential service endpoints.

Endpoints are added to the service by creating new keys and removed from the service by deleting keys.

### 添加一个访问点

通过`etcdctl`创建新的访问点:

```sh
ETCDCTL_API=3 etcdctl put foo/bar/my-service/1.2.3.4 '{"Addr":"1.2.3.4","Metadata":"..."}'
```

etcd客户端的方法`endpoints.Manager`也可以使用与`Addr`相匹配的key来注册新的访问点：

```go

em := endpoints.NewManager(client, "foo/bar/my-service")
err := em.AddEndpoint(context.TODO(),"foo/bar/my-service/e1", endpoints.Endpoint{Addr:"1.2.3.4"});
```

### 删除一个访问点

使用`etcdctl`从服务上删除主机：

```sh
ETCDCTL_API=3 etcdctl del foo/bar/my-service/1.2.3.4
```

etcd客户端的方法`endpoints.Manager`也支持删除访问点:

```go
em := endpoints.NewManager(client, "foo/bar/my-service")
err := em.DeleteEndpoint(context.TODO(), "foo/bar/my-service/e1");
```

### 使用lease注册一个访问点

使用lease注册访问点确保如果这个主机不能维护心跳(例如，计算机发生故障),则会将其从服务中删除:

```sh
lease=`ETCDCTL_API=3 etcdctl lease grant 5 | cut -f2 -d' '`
ETCDCTL_API=3 etcdctl put --lease=$lease my-service/1.2.3.4 '{"Addr":"1.2.3.4","Metadata":"..."}'
ETCDCTL_API=3 etcdctl lease keep-alive $lease
```
In the golang:

```go
em := endpoints.NewManager(client, "foo/bar/my-service")
err := endpoints.AddEndpoint(context.TODO(), "foo/bar/my-service/e1", endpoints.Endpoint{Addr:"1.2.3.4"});
```

### 自动更新访问点

如果需要在一个事物中修改多个访问点, 可以直接使用`endpoints.Manager`:

```
em := endpoints.NewManager(c, "foo")

err := em.Update(context.TODO(), []*endpoints.UpdateWithOpts{
    endpoints.NewDeleteUpdateOpts("foo/bar/my-service/e1", endpoints.Endpoint{Addr: "1.2.3.4"}),
	endpoints.NewAddUpdateOpts("foo/bar/my-service/e1", endpoints.Endpoint{Addr: "1.2.3.14"})})
```

