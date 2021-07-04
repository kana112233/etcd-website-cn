---
title: etcd gateway etcd 网关
weight: 4300
description: etcd gateway, when to use it, and how to set it up etcd网关，何时使用它，以及如何设置它
---

## What is etcd gateway 什么是etcd gateway

etcd网关是简单的转发网络数据到etcd集群的TCP代理.
etcd gateway is a simple TCP proxy that forwards network data to the etcd cluster. 
gateway是无状态和透明的；它既不检查客户端请求也不干扰集群响应。
The gateway is stateless and transparent; it neither inspects client requests nor interferes with cluster responses. 
它不会终止TLS连接，代表其客户端进行TLS握手或验证连接是否安全。
It does not terminate TLS connections, do TLS handshakes on behalf of its clients, or verify if the connection is secured.

gateway支持多个etcd服务器节点并且使用循环策略工作。
The gateway supports multiple etcd server endpoints and works on a simple round-robin policy. 
它只路由到可用的访问点，并向客户端隐藏故障的访问点。
It only routes to available endpoints and hides failures from its clients. 
其他的重试策略，例如加权循环，也许在未来支持。
Other retry policies, such as weighted round-robin, may be supported in the future.

## When to use etcd gateway 什么时候使用etcd gateway

每一个访问etcd的应用访问etcd必须先有etcd集群客户端的访问点的地址。
Every application that accesses etcd must first have the address of an etcd cluster client endpoint. 
If multiple applications on the same server access the same etcd cluster, every application still needs to know the advertised client endpoints of the etcd cluster. 
如果同一服务器上的多个应用程序访问同一个etcd集群，每个应用程序仍然需要知道etcd集群的通告客户端端点。

如果etcd集群被重新配置为具有不同的端点，每个应用程序可能还需要更新其端点列表。这种大规模的重新配置既乏味又容易出错。
If the etcd cluster is reconfigured to have different endpoints, every application may also need to update its endpoint list. This wide-scale reconfiguration is both tedious and error prone.

etcd网关提供稳定的本地访问点来解决这个问题。
etcd gateway solves this problem by serving as a stable local endpoint. 
典型的etcd网关配置让每台机器运行一个网关，监听本地地址，每个etcd应用程序都连接到其本地网关。
A typical etcd gateway configuration has each machine running a gateway listening on a local address and every etcd application connecting to its local gateway. 
结果是只有网关需要更新其端点，而不是更新每个应用程序。
The upshot is only the gateway needs to update its endpoints instead of updating each and every application.

总之，为了自动传播集群端点更改，etcd网关在每台集群上运行，为访问同一个etcd集群的多个应用程序提供服务。
In summary, to automatically propagate cluster endpoint changes, the etcd gateway runs on every machine serving multiple applications accessing the same etcd cluster.

## When not to use etcd gateway 什么时候不使用etcd gateway

- Improving performance 提高性能

网关不是为提高etcd集群性能而设计的。
The gateway is not designed for improving etcd cluster performance. 
没有提供缓存，watch合并或者批处理。
It does not provide caching, watch coalescing or batching. 
etcd团队正在开发一个缓存代理，旨在提高集群的可扩展性。
The etcd team is developing a caching proxy designed for improving cluster scalability.

运行一个集群管理系统
- Running on a cluster management system

高级集群管理系统像Kubernetes原生支持的服务发现。
Advanced cluster management systems like Kubernetes natively support service discovery. 
应用程序可以使用系统管理的DNS名称或虚拟IP地址访问etcd集群。
Applications can access an etcd cluster with a DNS name or a virtual IP address managed by the system. 
例如，kube-proxy相当于etcd网关。
For example, kube-proxy is equivalent to etcd gateway.

## Start etcd gateway 开始etcd gateway
考虑一个具有以下静态端点的etcd集群：
Consider an etcd cluster with the following static endpoints:

|Name|Address|Hostname|
|------|---------|------------------|
|infra0|10.0.1.10|infra0.example.com|
|infra1|10.0.1.11|infra1.example.com|
|infra2|10.0.1.12|infra2.example.com|

使用命令启动etcd网关以使用这些静态端点
Start the etcd gateway to use these static endpoints with the command:

```bash
$ etcd gateway start --endpoints=infra0.example.com,infra1.example.com,infra2.example.com
2016-08-16 11:21:18.867350 I | tcpproxy: ready to proxy client requests to [...]
```

Alternatively, if using DNS for service discovery, consider the DNS SRV entries:

```bash
$ dig +noall +answer SRV _etcd-client._tcp.example.com
_etcd-client._tcp.example.com. 300 IN SRV 0 0 2379 infra0.example.com.
_etcd-client._tcp.example.com. 300 IN SRV 0 0 2379 infra1.example.com.
_etcd-client._tcp.example.com. 300 IN SRV 0 0 2379 infra2.example.com.
```

```bash
$ dig +noall +answer infra0.example.com infra1.example.com infra2.example.com
infra0.example.com.  300  IN  A  10.0.1.10
infra1.example.com.  300  IN  A  10.0.1.11
infra2.example.com.  300  IN  A  10.0.1.12
```

Start the etcd gateway to fetch the endpoints from the DNS SRV entries with the command:

```bash
$ etcd gateway start --discovery-srv=example.com
2016-08-16 11:21:18.867350 I | tcpproxy: ready to proxy client requests to [...]
```

## Configuration flags 配置标志

### etcd cluster

#### --endpoints
 用于转发客户端连接的etcd服务器目标的逗号分隔列表。
 * Comma-separated list of etcd server targets for forwarding client connections.
 * Default: `127.0.0.1:2379`
 * Invalid example: `https://127.0.0.1:2379` (gateway does not terminate TLS). 
 注意网关不验证HTTP模式或检查请求，它只会转发被给的请求。
 Note that the gateway does not verify the HTTP schema or inspect the requests, it only forwards requests to the given endpoints.

#### --discovery-srv 

用于通过SRV记录引导集群端点的DNS域。
 * DNS domain used to bootstrap cluster endpoints through SRV recrods.
 * Default: (not set)

### Network

#### --listen-addr

 * Interface and port to bind for accepting client requests.
 * Default: `127.0.0.1:23790`

#### --retry-delay

 * Duration of delay before retrying to connect to failed endpoints.
 * Default: 1m0s
 * Invalid example: "123" (expects time unit in format)

### Security

#### --insecure-discovery

 * Accept SRV records that are insecure or susceptible to man-in-the-middle attacks.
 * Default: `false`

#### --trusted-ca-file

 * Path to the client TLS CA file for the etcd cluster to verify the endpoints returned from SRV discovery. Note that it is ONLY used for authenticating the discovered endpoints rather than creating connections for data transferring. The gateway never terminates TLS connections or create TLS connections on behalf of its clients.
 * Default: (not set)
