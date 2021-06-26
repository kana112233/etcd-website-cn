---
title: gRPC proxy gRPC代理
weight: 4350
description: A stateless etcd reverse proxy operating at the gRPC layer
在gRPC层的无状态的etcd反向代理操作
---
gRPC代理是在gRPC层(L7)无状态下etcd反向代理操作。
The gRPC proxy is a stateless etcd reverse proxy operating at the gRPC layer (L7). 
代理设计为了减少整个核心etcd集群上的总处理。
The proxy is designed to reduce the total processing load on the core etcd cluster. 
为了横向稳定性，它合并watch和lease API请求。
For horizontal scalability, it coalesces watch and lease API requests. 
为了保护集群免受客户端客户端的侵害，它会缓存key的range请求。
To protect the cluster against abusive clients, it caches key range requests.
gRPC代理支持多个etcd服务器端点。当代理开始，随机的选取一个etcd服务器端点使用。
The gRPC proxy supports multiple etcd server endpoints. When the proxy starts, it randomly picks one etcd server endpoint to use. 
此端点为所有请求提供服务，直到代理检测到端点故障。
This endpoint serves all requests until the proxy detects an endpoint failure. 
如果gRPC代理检测到一个endpoint失败，就会切换不同的endpoint，如果可用，向其客户屏蔽故障。
If the gRPC proxy detects an endpoint failure, it switches to a different endpoint, if available, to hide failures from its clients. 
其他的重试策略，例如加权循环，在未来也许会支持。
Other retry policies, such as weighted round-robin, may be supported in the future.

## Scalable watch API 可扩展的watch API

gRPC代理将同一key或range上多个客户端 watches合并为一个连接到etcd服务器的(`s-watcher`)。
The gRPC proxy coalesces multiple client watchers (`c-watchers`) on the same key or range into a single watcher (`s-watcher`) connected to an etcd server. 
代理从`s-watcher`到`c-watchers`广播所有的事件。
The proxy broadcasts all events from the `s-watcher` to its `c-watchers`.

假设N个客户端watch相同的key， 一个gRPC代理可以将etcd服务器上的监视负载从N减少到1.
Assuming N clients watch the same key, one gRPC proxy can reduce the watch load on the etcd server from N to 1. 
用户可以部署多个gRPC代理以进一步分配服务器负载。
Users can deploy multiple gRPC proxies to further distribute server load.

在下面的例子，这些客户端watch key A。gRPC代理合并了三个watchers，创建一个附加到etcd服务器的watcher。
In the following example, three clients watch on key A. The gRPC proxy coalesces the three watchers, creating a single  watcher attached to the etcd server.

```
            +-------------+
            | etcd server |
            +------+------+
                   ^ watch key A (s-watcher)
                   |
           +-------+-----+
           | gRPC proxy  | <-------+
           |             |         |
           ++-----+------+         |watch key A (c-watcher)
watch key A ^     ^ watch key A    |
(c-watcher) |     | (c-watcher)    |
    +-------+-+  ++--------+  +----+----+
    |  client |  |  client |  |  client |
    |         |  |         |  |         |
    +---------+  +---------+  +---------+
```

### Limitations 限制

有效地将多个客户端watchers合并为一个watcher，gRPC代理在可能的情况下将新的`c-watchers`合并到现有的`s-watcher`中。
To effectively coalesce multiple client watchers into a single watcher, the gRPC proxy coalesces new `c-watchers` into an existing `s-watcher` when possible. 
由于网络延迟或缓冲未传递事件，这个合并的`s-watcher`可能与etcd服务器不同步。
This coalesced `s-watcher` may be out of sync with the etcd server due to network delays or buffered undelivered events. 

当未指定watch revisoin时，gRPC代理不保证`c-watcher`将从最近存储的revision开始监视。
When the watch revision is unspecified, the gRPC proxy will not guarantee the `c-watcher` will start watching from the most recent store revision. 
例如，如果客户端从修订版1000的etcd服务器上watch。如果客户端从gRPC代理观看，则可能从revision990开始watch。
For example, if a client watches from an etcd server with revision 1000, that watcher will begin at revision 1000. If a client watches from the gRPC proxy, may begin watching from revision 990.

类似的限制适用于取消。当watcher取消时，etcd的服务器revision也许比取消响应的revision更大。
Similar limitations apply to cancellation. When the watcher is cancelled, the etcd server’s revision may be greater than the cancellation response revision.
对于大多数用例，这两个限制应该不会导致问题。
These two limitations should not cause problems for most use cases. 
在未来，可能还有其他选项可以强制watcher绕过gRPC代理以获得更准确的revision响应。
In the future, there may be additional options to force the watcher to bypass the gRPC proxy for more accurate revision responses.

## Scalable lease API 可扩展的lease API

为了保持其租约有效，客户端必须至少建立一个gRPC流到etcd服务器以发送周期性心跳。
To keep its leases alive, a client must establish at least one gRPC stream to an etcd server for sending periodic heartbeats. 
如果etcd工作负载涉及分布在许多客户端的大量租赁活动，则这些流可能会导致CPU使用率过高。
If an etcd workload involves heavy lease activity spread over many clients, these streams may contribute to excessive CPU utilization. 

为了减少核心集群上的流总数，代理支持租用流合并。
To reduce the total number of streams on the core cluster, the proxy supports lease stream coalescing.

假设N个客户端正在更新leases，单个gRPC代理将etcd服务器上的流负载从N减少到1.
Assuming N clients are updating leases, a single gRPC proxy reduces the stream load on the etcd server from N to 1.
部署可能有额外的gRPC代理，以进一步跨多个代理分发流。
Deployments may have additional gRPC proxies to further distribute streams across multiple proxies.

接下来的例子，3个客户端更新三个独立的lease(`L1`, `L2`, and `L3`).
In the following example, three clients update three independent leases (`L1`, `L2`, and `L3`). 
gRPC代理将合并3个lease流(`c-streams`)合并成一个连接到etcd服务器的租用保持存活的流 (`s-stream`)。
The gRPC proxy coalesces the three client lease streams (`c-streams`) into a single lease keep alive stream (`s-stream`) attached to an etcd server. 
代理将客户端租用心跳从c-stream转发到s-stream,然后将响应返回到相应的c-stream。
The proxy forwards client-side lease heartbeats from the c-streams to the s-stream, then returns the responses to the corresponding c-streams.

```
          +-------------+
          | etcd server |
          +------+------+
                 ^
                 | heartbeat L1, L2, L3
                 | (s-stream)
                 v
         +-------+-----+
         | gRPC proxy  +<-----------+
         +---+------+--+            | heartbeat L3
             ^      ^               | (c-stream)
heartbeat L1 |      | heartbeat L2  |
(c-stream)   v      v (c-stream)    v
      +------+-+  +-+------+  +-----+--+
      | client |  | client |  | client |
      +--------+  +--------+  +--------+
```

## Abusive clients protection 滥用客户端保护

gRPC代理缓存请求的响应当它不会破坏一致性要求。
The gRPC proxy caches responses for requests when it does not break consistency requirements. 
这个可以保护etcd服务器免受客户端for循环的滥用。
This can protect the etcd server from abusive clients in tight for loops.

## Start etcd gRPC proxy 开始etcd gRPC代理

考虑一个具有以下静态端点的etcd集群：
Consider an etcd cluster with the following static endpoints:

|Name|Address|Hostname|
|------|---------|------------------|
|infra0|10.0.1.10|infra0.example.com|
|infra1|10.0.1.11|infra1.example.com|
|infra2|10.0.1.12|infra2.example.com|

通过以下命令启动etcd gRPC代理以使用这些静态访问点:
Start the etcd gRPC proxy to use these static endpoints with the command:

```bash
$ etcd grpc-proxy start --endpoints=infra0.example.com,infra1.example.com,infra2.example.com --listen-addr=127.0.0.1:2379
```
etcd gRPC代理启动并监听2379端口。它将客户端请求转发到上面提供的三个访问点之一。
The etcd gRPC proxy starts and listens on port 2379. It forwards client requests to one of the three endpoints provided above.

通过代理发送请求
Sending requests through the proxy:

```bash
$ ETCDCTL_API=3 etcdctl --endpoints=127.0.0.1:2379 put foo bar
OK
$ ETCDCTL_API=3 etcdctl --endpoints=127.0.0.1:2379 get foo
foo
bar
```

## Client endpoint synchronization and name resolution 客户端访问点同步和名称解析

代理支持注册访问点通过写入用户定义的访问点。
The proxy supports registering its endpoints for discovery by writing to a user-defined endpoint. 
这个服务两个目的。第一个，允许客户端将其端点与一组代理访问点同步以实现高可用性。第二，它是etcd的端点提供者[gRPC naming](../dev-guide/grpc_naming.md).
This serves two purposes. First, it allows clients to synchronize their endpoints against a set of proxy endpoints for high availability. Second, it is an endpoint provider for etcd [gRPC naming](../dev-guide/grpc_naming.md).

注册通过提供用户定义的前缀代理
Register proxy(s) by providing a user-defined prefix:

```bash
$ etcd grpc-proxy start --endpoints=localhost:2379 \
  --listen-addr=127.0.0.1:23790 \
  --advertise-client-url=127.0.0.1:23790 \
  --resolver-prefix="___grpc_proxy_endpoint" \
  --resolver-ttl=60

$ etcd grpc-proxy start --endpoints=localhost:2379 \
  --listen-addr=127.0.0.1:23791 \
  --advertise-client-url=127.0.0.1:23791 \
  --resolver-prefix="___grpc_proxy_endpoint" \
  --resolver-ttl=60
```

The proxy will list all its members for member list:

```bash
ETCDCTL_API=3 etcdctl --endpoints=http://localhost:23790 member list --write-out table

+----+---------+--------------------------------+------------+-----------------+
| ID | STATUS  |              NAME              | PEER ADDRS |  CLIENT ADDRS   |
+----+---------+--------------------------------+------------+-----------------+
|  0 | started | Gyu-Hos-MBP.sfo.coreos.systems |            | 127.0.0.1:23791 |
|  0 | started | Gyu-Hos-MBP.sfo.coreos.systems |            | 127.0.0.1:23790 |
+----+---------+--------------------------------+------------+-----------------+
```

这让客户端可以通过Sync自动发现代理访问点：
This lets clients automatically discover proxy endpoints through Sync:

```go
cli, err := clientv3.New(clientv3.Config{
    Endpoints: []string{"http://localhost:23790"},
})
if err != nil {
    log.Fatal(err)
}
defer cli.Close()

// fetch registered grpc-proxy endpoints
if err := cli.Sync(context.Background()); err != nil {
    log.Fatal(err)
}
```
注意如果没有解析器前缀的情况下配置了代理，
Note that if a proxy is configured without a resolver prefix,

```bash
$ etcd grpc-proxy start --endpoints=localhost:2379 \
  --listen-addr=127.0.0.1:23792 \
  --advertise-client-url=127.0.0.1:23792
```

The member list API to the grpc-proxy returns its own `advertise-client-url`:

```bash
ETCDCTL_API=3 etcdctl --endpoints=http://localhost:23792 member list --write-out table

+----+---------+--------------------------------+------------+-----------------+
| ID | STATUS  |              NAME              | PEER ADDRS |  CLIENT ADDRS   |
+----+---------+--------------------------------+------------+-----------------+
|  0 | started | Gyu-Hos-MBP.sfo.coreos.systems |            | 127.0.0.1:23792 |
+----+---------+--------------------------------+------------+-----------------+
```

## Namespacing 命名空间
假设一个应用程序希望完全控制整个key空间，但是etcd集群是和其他的应用共享的。
Suppose an application expects full control over the entire key space, but the etcd cluster is shared with other applications. 
让所有应用程序运行而互不干扰，代理可以划分etcd的key空间，因此客户端似乎可以访问完整的key空间。
To let all appications run without interfering with each other, the proxy can partition the etcd keyspace so clients appear to have access to the complete keyspace. 
当代理设置了`--namespace`标志，进入代理的所有客户端请求都被转换为用户定义的前缀的key上。
When the proxy is given the flag `--namespace`, all client requests going into the proxy are translated to have a user-defined prefix on the keys. 
对etcd集群的访问是有带有前缀的，来自代理的响应将去除前缀。
Accesses to the etcd cluster will be under the prefix and responses from the proxy will strip away the prefix; 
对客户端来说，似乎根本没有前缀。
to the client, it appears as if there is no prefix at all.

To namespace a proxy, start it with `--namespace`:

```bash
$ etcd grpc-proxy start --endpoints=localhost:2379 \
  --listen-addr=127.0.0.1:23790 \
  --namespace=my-prefix/
```

Accesses to the proxy are now transparently prefixed on the etcd cluster:

```bash
$ ETCDCTL_API=3 etcdctl --endpoints=localhost:23790 put my-key abc
# OK
$ ETCDCTL_API=3 etcdctl --endpoints=localhost:23790 get my-key
# my-key
# abc
$ ETCDCTL_API=3 etcdctl --endpoints=localhost:2379 get my-prefix/my-key
# my-prefix/my-key
# abc
```

## TLS termination TLS 终止

Terminate TLS from a secure etcd cluster with the gRPC proxy by serving an unencrypted local endpoint.
通过提供未加密的本地端点，使用gRPC代理从安全的etcd集群中终止TLS。

要尝试一下，使用客户端https启动单个成员etcd集群：
To try it out, start a single member etcd cluster with client https:

```sh
$ etcd --listen-client-urls https://localhost:2379 --advertise-client-urls https://localhost:2379 --cert-file=peer.crt --key-file=peer.key --trusted-ca-file=ca.crt --client-cert-auth
```
配置客户端端口的https服务：
Confirm the client port is serving https:

```sh
# fails
$ ETCDCTL_API=3 etcdctl --endpoints=http://localhost:2379 endpoint status
# works
$ ETCDCTL_API=3 etcdctl --endpoints=https://localhost:2379 --cert=client.crt --key=client.key --cacert=ca.crt endpoint status
```

接下来，通过使用客户端证书连接到etcd端点`https://localhost:2379`在`https://localhost:2379`上启动gRPC代理。
Next, start a gRPC proxy on `localhost:12379` by connecting to the etcd endpoint `https://localhost:2379` using the client certificates:

```sh
$ etcd grpc-proxy start --endpoints=https://localhost:2379 --listen-addr localhost:12379 --cert client.crt --key client.key --cacert=ca.crt --insecure-skip-tls-verify &
```
最后，通过http将key放入代理来测试TLS终止。
Finally, test the TLS termination by putting a key into the proxy over http:

```sh
$ ETCDCTL_API=3 etcdctl --endpoints=http://localhost:12379 put abc def
# OK
```

## Metrics and Health 指标和安全

gRPC代理通过定义`--endpoints`暴露了`/health`和`/metrics`。
The gRPC proxy exposes `/health` and Prometheus `/metrics` endpoints for the etcd members defined by `--endpoints`. 
另一种方法是定义一个额外的URL，该URL将使用`--metrics-addr`标志响应`/metrics` and `/health`访问点。
An alternative define an additional URL that will respond to both the `/metrics` and `/health` endpoints with the `--metrics-addr` flag.

```bash
$ etcd grpc-proxy start \
  --endpoints https://localhost:2379 \
  --metrics-addr https://0.0.0.0:4443 \
  --listen-addr 127.0.0.1:23790 \
  --key client.key \
  --key-file proxy-server.key \
  --cert client.crt \
  --cert-file proxy-server.crt \
  --cacert ca.pem \
  --trusted-ca-file proxy-ca.pem
 ```

### Known issue 已知的问题
代理的主接口服务于 HTTP2 和 HTTP/1.1。
The main interface of the proxy serves both HTTP2 and HTTP/1.1. 

If proxy is setup with TLS as show in the above example, when using a client such as cURL against the listening interface will require explicitly setting the protocol to HTTP/1.1 on the request to return `/metrics` or `/health`. By using the `--metrics-addr` flag the secondary interface will not have this requirement.
如果代理使用 TLS 设置，如上例所示，当对侦听接口使用诸如 cURL 之类的客户端时，将需要在请求返回 `/metrics` 或 `/health` 时将协议显式设置为 HTTP/1.1。 通过使用`--metrics-addr` 标志，辅助接口将没有这个要求。


```bash
 $ curl --cacert proxy-ca.pem --key proxy-client.key --cert proxy-client.crt https://127.0.0.1:23790/metrics --http1.1
```
