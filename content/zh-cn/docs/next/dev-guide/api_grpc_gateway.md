---
title: Why gRPC gateway 为什么是gRPC gateway
weight: 3375
description: Why you should consider using the gRPC gateway 为什么你应该考虑使用gRPC gateway
---

etcd v3 使用[gRPC][grpc]作为它的通讯协议。
etcd项目包括了一个基于gRPC的[Go client][go-client]和一个通过gRPC和etcd集群进行通信的命令行工具：[etcdctl][etcdctl]。

对于不支持gRPC的语言，etcd提供了JSON API的项目：[gRPC gateway][grpc-gateway]。
For languages with no gRPC support, etcd provides a JSON [gRPC gateway][grpc-gateway].
这个gateway提供一个RESTful代理，该代理将从HTTP/JSON请求转换为gRPC消息。

## 使用grpc gateway

The gateway accepts a [JSON mapping][json-mapping] for etcd's [protocol buffer][api-ref] message definitions.
注意`key`and`value`字段被定义为字节数组，因此必须使用JSON进行base64编码。下面的例子使用`curl`，当然任意的其他的HTTP/JSON客户端也是可以的。

### 笔记

从etcd v3.3，gRPC gateway访问点已经改变了：
gRPC gateway endpoint has changed since etcd v3.3:

- etcd v3.2 or before uses only `[CLIENT-URL]/v3alpha/*`.
- etcd v3.3 uses `[CLIENT-URL]/v3beta/*` while keeping `[CLIENT-URL]/v3alpha/*`.
- etcd v3.4 uses `[CLIENT-URL]/v3/*` while keeping `[CLIENT-URL]/v3beta/*`.
  - **`[CLIENT-URL]/v3alpha/*` is deprecated**.
- etcd v3.5 or later uses only `[CLIENT-URL]/v3/*`.
  - **`[CLIENT-URL]/v3beta/*` is deprecated**.

gRPC-gateway不支持使用TLS通用名称的身份验证
gRPC-gateway does not support authentication using TLS Common Name.

### Put and get keys

Use the `/v3/kv/range` and `/v3/kv/put` services to read and write keys:
使用`/v3/kv/range`和`/v3/kv/put`服务来读和写keys：

```bash
<<COMMENT
https://www.base64encode.org/
foo is 'Zm9v' in Base64
bar is 'YmFy'
COMMENT

curl -L http://localhost:2379/v3/kv/put \
  -X POST -d '{"key": "Zm9v", "value": "YmFy"}'
# {"header":{"cluster_id":"12585971608760269493","member_id":"13847567121247652255","revision":"2","raft_term":"3"}}

curl -L http://localhost:2379/v3/kv/range \
  -X POST -d '{"key": "Zm9v"}'
# {"header":{"cluster_id":"12585971608760269493","member_id":"13847567121247652255","revision":"2","raft_term":"3"},"kvs":[{"key":"Zm9v","create_revision":"2","mod_revision":"2","version":"1","value":"YmFy"}],"count":"1"}

# get all keys prefixed with "foo"
curl -L http://localhost:2379/v3/kv/range \
  -X POST -d '{"key": "Zm9v", "range_end": "Zm9w"}'
# {"header":{"cluster_id":"12585971608760269493","member_id":"13847567121247652255","revision":"2","raft_term":"3"},"kvs":[{"key":"Zm9v","create_revision":"2","mod_revision":"2","version":"1","value":"YmFy"}],"count":"1"}
```

### Watch keys

Use the `/v3/watch` service to watch keys:

```bash
curl -N http://localhost:2379/v3/watch \
  -X POST -d '{"create_request": {"key":"Zm9v"} }' &
# {"result":{"header":{"cluster_id":"12585971608760269493","member_id":"13847567121247652255","revision":"1","raft_term":"2"},"created":true}}

curl -L http://localhost:2379/v3/kv/put \
  -X POST -d '{"key": "Zm9v", "value": "YmFy"}' >/dev/null 2>&1
# {"result":{"header":{"cluster_id":"12585971608760269493","member_id":"13847567121247652255","revision":"2","raft_term":"2"},"events":[{"kv":{"key":"Zm9v","create_revision":"2","mod_revision":"2","version":"1","value":"YmFy"}}]}}
```

### Transactions

使用`/v3/kv/txn`发出一个事物：

```bash
# target CREATE
curl -L http://localhost:2379/v3/kv/txn \
  -X POST \
  -d '{"compare":[{"target":"CREATE","key":"Zm9v","createRevision":"2"}],"success":[{"requestPut":{"key":"Zm9v","value":"YmFy"}}]}'
# {"header":{"cluster_id":"12585971608760269493","member_id":"13847567121247652255","revision":"3","raft_term":"2"},"succeeded":true,"responses":[{"response_put":{"header":{"revision":"3"}}}]}
```

```bash
# target VERSION
curl -L http://localhost:2379/v3/kv/txn \
  -X POST \
  -d '{"compare":[{"version":"4","result":"EQUAL","target":"VERSION","key":"Zm9v"}],"success":[{"requestRange":{"key":"Zm9v"}}]}'
# {"header":{"cluster_id":"14841639068965178418","member_id":"10276657743932975437","revision":"6","raft_term":"3"},"succeeded":true,"responses":[{"response_range":{"header":{"revision":"6"},"kvs":[{"key":"Zm9v","create_revision":"2","mod_revision":"6","version":"4","value":"YmF6"}],"count":"1"}}]}
```

### Authentication 认证

给`/v3/auth`服务设置认证：

```bash
# create root user
curl -L http://localhost:2379/v3/auth/user/add \
  -X POST -d '{"name": "root", "password": "pass"}'
# {"header":{"cluster_id":"14841639068965178418","member_id":"10276657743932975437","revision":"1","raft_term":"2"}}

# create root role
curl -L http://localhost:2379/v3/auth/role/add \
  -X POST -d '{"name": "root"}'
# {"header":{"cluster_id":"14841639068965178418","member_id":"10276657743932975437","revision":"1","raft_term":"2"}}

# grant root role
curl -L http://localhost:2379/v3/auth/user/grant \
  -X POST -d '{"user": "root", "role": "root"}'
# {"header":{"cluster_id":"14841639068965178418","member_id":"10276657743932975437","revision":"1","raft_term":"2"}}

# enable auth
curl -L http://localhost:2379/v3/auth/enable -X POST -d '{}'
# {"header":{"cluster_id":"14841639068965178418","member_id":"10276657743932975437","revision":"1","raft_term":"2"}}
```

Authenticate with etcd for an authentication token using `/v3/auth/authenticate`:

```bash
# get the auth token for the root user
curl -L http://localhost:2379/v3/auth/authenticate \
  -X POST -d '{"name": "root", "password": "pass"}'
# {"header":{"cluster_id":"14841639068965178418","member_id":"10276657743932975437","revision":"1","raft_term":"2"},"token":"sssvIpwfnLAcWAQH.9"}
```

Set the `Authorization` header to the authentication token to fetch a key using authentication credentials:

```bash
curl -L http://localhost:2379/v3/kv/put \
  -H 'Authorization: sssvIpwfnLAcWAQH.9' \
  -X POST -d '{"key": "Zm9v", "value": "YmFy"}'
# {"header":{"cluster_id":"14841639068965178418","member_id":"10276657743932975437","revision":"2","raft_term":"2"}}
```

## Swagger

生成[Swagger][swagger]API可以在[rpc.swagger.json][swagger-doc]中找到。

[api-ref]: ./api_reference_v3
[etcdctl]: https://github.com/etcd-io/etcd/tree/master/etcdctl
[go-client]: https://github.com/etcd-io/etcd/tree/master/client/v3
[grpc]: https://www.grpc.io/
[grpc-gateway]: https://github.com/grpc-ecosystem/grpc-gateway
[json-mapping]: https://developers.google.com/protocol-buffers/docs/proto3#json
[swagger]: http://swagger.io/
[swagger-doc]: ../apispec/swagger/rpc.swagger.json
