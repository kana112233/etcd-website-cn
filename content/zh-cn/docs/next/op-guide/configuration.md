---
title: Configuration flags 配置标志
weight: 4050
description: "etcd configuration: files, flags, and environment variables" “etcd配置： 文件，标志，和环境变量”
---

etcd通过配置文件，命令行标志和环境变量来配置。
etcd is configurable through a configuration file, various command-line flags, and environment variables.

可重用的配置文件是一个YAML文件，由一个或者多个命令行标志的名称和值组成，如下所述。
A reusable configuration file is a YAML file made with name and value of one or more command-line flags described below. 
要使用此文件，请将文件路径指定为`--config-file`标志或者`ETCD_CONFIG_FILE`环境变量。[sample configuration file][sample-config-file]可以用作起点，根据需要创建新的配置文件。
In order to use this file, specify the file path as a value to the `--config-file` flag or `ETCD_CONFIG_FILE` environment variable. The [sample configuration file][sample-config-file] can be used as a starting point to create a new configuration file as needed.

命令行上设置的选项优于环境中的选项。
Options set on the command line take precedence over those from the environment. 
如果提供了配置文件，其他的命令行标志和环境变量将被忽略。
If a configuration file is provided, other command line flags and environment variables will be ignored.
例如，`etcd --config-file etcd.conf.yml.sample --data-dir /tmp`将忽略`--data-dir`标志。
For example, `etcd --config-file etcd.conf.yml.sample --data-dir /tmp` will ignore the `--data-dir` flag.

标志`--my-flag`的环境变量格式为`ETCD_MY_FLAG`。它适用于所有标志。
The format of environment variable for flag `--my-flag` is `ETCD_MY_FLAG`. It applies to all flags.

[official etcd ports][iana-ports]是2379，这是给客户端用的，2380是peer交流的。
The [official etcd ports][iana-ports] are 2379 for client requests and 2380 for peer communication. 
etcd端口可以接设置为接受TLS流量，非TLS流量或TLS和非TLS流量。
The etcd ports can be set to accept TLS traffic, non-TLS traffic, or both TLS and non-TLS traffic.
要在Linux启动是使用自定义设置自动启动etcd，强烈建议使用[systemd][systemd-intro]单元。
To start etcd automatically using custom settings at startup in Linux, using a [systemd][systemd-intro] unit is highly recommended.

## Member flags 成员标记

### --name
+ Human-readable name for this member. 此成员的可读名称
+ default: "default" 默认值： "default"
+ env variable: ETCD_NAME 环境变量： ETCD_NAME
+ This value is referenced as this node's own entries listed in the `--initial-cluster` flag (e.g., `default=http://localhost:2380`). This needs to match the key used in the flag if using [static bootstrapping][build-cluster]. 

When using discovery, each member must have a unique name. `Hostname` or `machine-id` can be a good choice.
该值被引用为该节点自己的条目，列在`--initial-cluster`标记中（例如`default=http://localhost:2380`）。
如果使用[static bootstrapping][build-cluster]，这需要匹配标记中使用的key。
当使用发现，每一个成员必须有一个唯一的名字。`Hostname` or `machine-id`可能是一个好的选择。

### --data-dir
+ Path to the data directory. data文件夹的目录
+ default: "${name}.etcd" 默认： "${name}.etcd"
+ env variable: ETCD_DATA_DIR 环境变量： ETCD_DATA_DIR

### --wal-dir
专用wal目录的路劲。如果这个标记被设置，etcd将写入WAL文件到walDir而不是dataDir。
+ Path to the dedicated wal directory. If this flag is set, etcd will write the WAL files to the walDir rather than the dataDir. 
这个允许专用的磁盘使用，有助于避免日志记录和其他IO操作之间的io竞争。
This allows a dedicated disk to be used, and helps avoid io competition between logging and other IO operations.
+ default: ""
+ env variable: ETCD_WAL_DIR

### --snapshot-count 
+ Number of committed transactions to trigger a snapshot to disk. 去触发快照到磁盘的提交事物的数量
+ default: "100000" 默认： 100000
+ env variable: ETCD_SNAPSHOT_COUNT 环境变量： ETCD_SNAPSHOT_COUNT

### --heartbeat-interval
+ Time (in milliseconds) of a heartbeat interval. 心跳间隔的时间（单位毫秒）
+ default: "100" 默认： 100
+ env variable: ETCD_HEARTBEAT_INTERVAL 环境变量： ETCD_HEARTBEAT_INTERVAL

### --election-timeout
选举超时的时间。看[Documentation/tuning.md][tuning]了解细节。
+ Time (in milliseconds) for an election to timeout. See [Documentation/tuning.md][tuning] for details.
+ default: "1000" 默认： 1000
+ env variable: ETCD_ELECTION_TIMEOUT

### --listen-peer-urls
用于侦听客户端流量的 URL 列表。
+ List of URLs to listen on for peer traffic. 
该标志告诉etcd接收来自指定scheme://IP:port组合上的peers。
This flag tells the etcd to accept incoming requests from its peers on the specified scheme://IP:port combinations. 
Scheme可以是http或者https。或者`unix://<file-path>` or `unixs://<file-path>` 用与unix套接字.
Scheme can be http or https. Alternatively, use `unix://<file-path>` or `unixs://<file-path>` for unix sockets. 
如果指定IP为0.0.0.0，etcd监听所有接口上的给定端口。
If 0.0.0.0 is specified as the IP, etcd listens to the given port on all interfaces. 
如果给出了IP地址和端口，etcd将监听给定的访问点和接口。
If an IP address is given as well as a port, etcd will listen on the given port and interface. 
多个URL可用于指定要监听的多个地址和端口。
Multiple URLs may be used to specify a number of addresses and ports to listen on. 
etcd将响应来自任何列出的地址和端口的请求。
The etcd will respond to requests from any of the listed addresses and ports.
+ default: "http://localhost:2380"
+ env variable: ETCD_LISTEN_PEER_URLS
+ example: "http://10.0.0.1:2380"
+ invalid example: "http://example.com:2380" (domain name is invalid for binding)

### --listen-client-urls

用于侦听客户端流量的URL列表。这个标志告诉etcd接受来自客户端的请求
+ List of URLs to listen on for client traffic. 
该标志告诉etcd接收来自指定scheme://IP:port组合上的peers。
This flag tells the etcd to accept incoming requests from the clients on the specified scheme://IP:port combinations. Scheme can be either http or https. Alternatively, use `unix://<file-path>` or `unixs://<file-path>` for unix sockets. If 0.0.0.0 is specified as the IP, etcd listens to the given port on all interfaces. If an IP address is given as well as a port, etcd will listen on the given port and interface. Multiple URLs may be used to specify a number of addresses and ports to listen on. The etcd will respond to requests from any of the listed addresses and ports.
+ default: "http://localhost:2379"
+ env variable: ETCD_LISTEN_CLIENT_URLS
+ example: "http://10.0.0.1:2379"
+ invalid example: "http://example.com:2379" (domain name is invalid for binding)

### --max-snapshots
要保留的最大快照文件数
+ Maximum number of snapshot files to retain (0 is unlimited)
+ default: 5
+ env variable: ETCD_MAX_SNAPSHOTS
Windows用户的默认设置为无限制，并且手动设置为5是推荐的（或对安全的某种偏好）。
+ The default for users on Windows is unlimited, and manual purging down to 5 (or some preference for safety) is recommended.

### --max-wals
+ Maximum number of wal files to retain (0 is unlimited) 保留的最大的wal文件
+ default: 5
+ env variable: ETCD_MAX_WALS
+ The default for users on Windows is unlimited, and manual purging down to 5 (or some preference for safety) is recommended.

### --cors 
逗号分割的CORS（跨资源共享）来源的白名单
+ Comma-separated white list of origins for CORS (cross-origin resource sharing). 
+ default: ""
+ env variable: ETCD_CORS

### --quota-backend-bytes
当后端大小超过给定配额时发出警告。
+ Raise alarms when backend size exceeds the given quota (0 defaults to low space quota). 
+ default: 0
+ env variable: ETCD_QUOTA_BACKEND_BYTES

### --backend-batch-limit
BackendBatchLimit是提交后端事物之前最大的操作数。
+ BackendBatchLimit is the maximum operations before commit the backend transaction.
+ default: 0
+ env variable: ETCD_BACKEND_BATCH_LIMIT

### --backend-bbolt-freelist-type
etcd后端（bboltdb）使用的空闲列表类型（array and map支持的类型）
+ The freelist type that etcd backend(bboltdb) uses (array and map are supported types).
+ default: map
+ env variable: ETCD_BACKEND_BBOLT_FREELIST_TYPE

### --backend-batch-interval
BackendBatchInterval是提交后端事物之前最大时间。
+ BackendBatchInterval is the maximum time before commit the backend transaction.
+ default: 0
+ env variable: ETCD_BACKEND_BATCH_INTERVAL

### --max-txn-ops
事物中操作允许的最大数量。
+ Maximum number of operations permitted in a transaction.
+ default: 128
+ env variable: ETCD_MAX_TXN_OPS

### --max-request-bytes
服务器将接受的最大客户端请求大小（以字节为单位）
+ Maximum client request size in bytes the server will accept.
+ default: 1572864
+ env variable: ETCD_MAX_REQUEST_BYTES

### --grpc-keepalive-min-time
客户端ping服务器之前应该等待的最小持续时间间隔。
+ Minimum duration interval that a client should wait before pinging server.
+ default: 5s
+ env variable: ETCD_GRPC_KEEPALIVE_MIN_TIME

### --grpc-keepalive-interval
服务器到客户端ping检查连接是否处于活动状态的频率持续时间（0表示禁止）
+ Frequency duration of server-to-client ping to check if a connection is alive (0 to disable).
+ default: 2h
+ env variable: ETCD_GRPC_KEEPALIVE_INTERVAL

### --grpc-keepalive-timeout
关闭无响应连接之前的额外等待时间（0表示禁止）。
+ Additional duration of wait before closing a non-responsive connection (0 to disable).
+ default: 20s
+ env variable: ETCD_GRPC_KEEPALIVE_TIMEOUT

## Clustering flags

`--initial-advertise-peer-urls`, `--initial-cluster`, `--initial-cluster-state`, and `--initial-cluster-token` flags are used in bootstrapping ([static bootstrap][build-cluster], [discovery-service bootstrap][discovery] or [runtime reconfiguration][reconfig]) a new member, and ignored when restarting an existing member.

`--discovery` prefix flags need to be set when using [discovery service][discovery].

### --initial-advertise-peer-urls

此成员的peerURL列表，用于向集群的其余部分做通知。
+ List of this member's peer URLs to advertise to the rest of the cluster. 
这些地址用来在集群中沟通etcd数据。
These addresses are used for communicating etcd data around the cluster. 
至少一个必须路由到所有集群成员。这些URLs可以包含域名的名字。
At least one must be routable to all cluster members. These URLs can contain domain names.
+ default: "http://localhost:2380"
+ env variable: ETCD_INITIAL_ADVERTISE_PEER_URLS
+ example: "http://example.com:2380, http://10.0.0.1:2380"

### --initial-cluster
初始化集群启动配置
+ Initial cluster configuration for bootstrapping.
+ default: "default=http://localhost:2380"
+ env variable: ETCD_INITIAL_CLUSTER
+ The key is the value of the `--name` flag for each node provided. The default uses `default` for the key because this is the default for the `--name` flag.

### --initial-cluster-state
+ Initial cluster state ("new" or "existing"). Set to `new` for all members present during initial static or DNS bootstrapping. If this option is set to `existing`, etcd will attempt to join the existing cluster. If the wrong value is set, etcd will attempt to start but fail safely.
+ default: "new"
+ env variable: ETCD_INITIAL_CLUSTER_STATE

### --initial-cluster-token
+ Initial cluster token for the etcd cluster during bootstrap.
+ default: "etcd-cluster"
+ env variable: ETCD_INITIAL_CLUSTER_TOKEN

### --advertise-client-urls
+ List of this member's client URLs to advertise to the rest of the cluster. These URLs can contain domain names.
+ default: "http://localhost:2379"
+ env variable: ETCD_ADVERTISE_CLIENT_URLS
+ example: "http://example.com:2379, http://10.0.0.1:2379"
+ Be careful if advertising URLs such as http://localhost:2379 from a cluster member and are using the proxy feature of etcd. This will cause loops, because the proxy will be forwarding requests to itself until its resources (memory, file descriptors) are eventually depleted.

### --discovery
+ Discovery URL used to bootstrap the cluster.
+ default: ""
+ env variable: ETCD_DISCOVERY

### --discovery-srv
+ DNS srv domain used to bootstrap the cluster.
+ default: ""
+ env variable: ETCD_DISCOVERY_SRV

### --discovery-srv-name
+ Suffix to the DNS srv name queried when bootstrapping using DNS.
+ default: ""
+ env variable: ETCD_DISCOVERY_SRV_NAME

### --discovery-fallback
+ Expected behavior ("exit" or "proxy") when discovery services fails. "proxy" supports v2 API only.
+ default: "proxy"
+ env variable: ETCD_DISCOVERY_FALLBACK

### --discovery-proxy
+ HTTP proxy to use for traffic to discovery service.
+ default: ""
+ env variable: ETCD_DISCOVERY_PROXY

### --strict-reconfig-check
+ Reject reconfiguration requests that would cause quorum loss.
+ default: true
+ env variable: ETCD_STRICT_RECONFIG_CHECK

### --auto-compaction-retention
+ Auto compaction retention for mvcc key value store in hour. 0 means disable auto compaction.
+ default: 0
+ env variable: ETCD_AUTO_COMPACTION_RETENTION

### --auto-compaction-mode
+ Interpret 'auto-compaction-retention' one of: 'periodic', 'revision'. 'periodic' for duration based retention, defaulting to hours if no time unit is provided (e.g. '5m'). 'revision' for revision number based retention.
+ default: periodic
+ env variable: ETCD_AUTO_COMPACTION_MODE

### --enable-v2
+ Accept etcd V2 client requests
+ default: false
+ env variable: ETCD_ENABLE_V2

## Proxy flags

`--proxy` prefix flags configures etcd to run in [proxy mode][proxy]. "proxy" supports v2 API only.

### --proxy
+ Proxy mode setting ("off", "readonly" or "on").
+ default: "off"
+ env variable: ETCD_PROXY

### --proxy-failure-wait
+ Time (in milliseconds) an endpoint will be held in a failed state before being reconsidered for proxied requests.
+ default: 5000
+ env variable: ETCD_PROXY_FAILURE_WAIT

### --proxy-refresh-interval
+ Time (in milliseconds) of the endpoints refresh interval.
+ default: 30000
+ env variable: ETCD_PROXY_REFRESH_INTERVAL

### --proxy-dial-timeout
+ Time (in milliseconds) for a dial to timeout or 0 to disable the timeout
+ default: 1000
+ env variable: ETCD_PROXY_DIAL_TIMEOUT

### --proxy-write-timeout
+ Time (in milliseconds) for a write to timeout or 0 to disable the timeout.
+ default: 5000
+ env variable: ETCD_PROXY_WRITE_TIMEOUT

### --proxy-read-timeout
+ Time (in milliseconds) for a read to timeout or 0 to disable the timeout.
+ Don't change this value if using watches because use long polling requests.
+ default: 0
+ env variable: ETCD_PROXY_READ_TIMEOUT

## Security flags

The security flags help to [build a secure etcd cluster][security].

### --ca-file

**DEPRECATED**

+ Path to the client server TLS CA file. `--ca-file ca.crt` could be replaced by `--trusted-ca-file ca.crt --client-cert-auth` and etcd will perform the same.
+ default: ""
+ env variable: ETCD_CA_FILE

### --cert-file
+ Path to the client server TLS cert file.
+ default: ""
+ env variable: ETCD_CERT_FILE

### --key-file
+ Path to the client server TLS key file.
+ default: ""
+ env variable: ETCD_KEY_FILE

### --client-cert-auth
+ Enable client cert authentication.
+ default: false
+ env variable: ETCD_CLIENT_CERT_AUTH
+ CN authentication is not supported by gRPC-gateway.

### --client-crl-file
+ Path to the client certificate revocation list file.
+ default: ""
+ env variable: ETCD_CLIENT_CRL_FILE

### --client-cert-allowed-hostname
+ Allowed Allowed TLS name for client cert authentication.
+ default: ""
+ env variable: ETCD_CLIENT_CERT_ALLOWED_HOSTNAME

### --trusted-ca-file
+ Path to the client server TLS trusted CA cert file.
+ default: ""
+ env variable: ETCD_TRUSTED_CA_FILE

### --auto-tls
+ Client TLS using generated certificates
+ default: false
+ env variable: ETCD_AUTO_TLS

### --peer-ca-file

**DEPRECATED**

+ Path to the peer server TLS CA file. `--peer-ca-file ca.crt` could be replaced by `--peer-trusted-ca-file ca.crt --peer-client-cert-auth` and etcd will perform the same.
+ default: ""
+ env variable: ETCD_PEER_CA_FILE

### --peer-cert-file
+ Path to the peer server TLS cert file. This is the cert for peer-to-peer traffic, used both for server and client.
+ default: ""
+ env variable: ETCD_PEER_CERT_FILE

### --peer-key-file
+ Path to the peer server TLS key file. This is the key for peer-to-peer traffic, used both for server and client.
+ default: ""
+ env variable: ETCD_PEER_KEY_FILE

### --peer-client-cert-auth
+ Enable peer client cert authentication.
+ default: false
+ env variable: ETCD_PEER_CLIENT_CERT_AUTH

### --peer-crl-file
+ Path to the peer certificate revocation list file.
+ default: ""
+ env variable: ETCD_PEER_CRL_FILE

### --peer-trusted-ca-file
+ Path to the peer server TLS trusted CA file.
+ default: ""
+ env variable: ETCD_PEER_TRUSTED_CA_FILE

### --peer-auto-tls
+ Peer TLS using generated certificates
+ default: false
+ env variable: ETCD_PEER_AUTO_TLS

### --peer-cert-allowed-cn
+ Allowed CommonName for inter peer authentication.
+ default: ""
+ env variable: ETCD_PEER_CERT_ALLOWED_CN

### --peer-cert-allowed-hostname
+ Allowed TLS certificate name for inter peer authentication.
+ default: ""
+ env variable: ETCD_PEER_CERT_ALLOWED_HOSTNAME

### --cipher-suites
+ Comma-separated list of supported TLS cipher suites between server/client and peers.
+ default: ""
+ env variable: ETCD_CIPHER_SUITES

## Logging flags

### --logger

**Available from v3.4.**
**WARNING: `--logger=capnslog` to be deprecated in v3.5.**

+ Specify 'zap' for structured logging or 'capnslog'.
+ default: capnslog
+ env variable: ETCD_LOGGER

### --log-outputs
+ Specify 'stdout' or 'stderr' to skip journald logging even when running under systemd, or list of comma separated output targets.
+ default: default
+ env variable: ETCD_LOG_OUTPUTS
+ 'default' use 'stderr' config for v3.4 during zap logger migraion

### --log-level

**Available from v3.4.**

+ Configures log level. Only supports debug, info, warn, error, panic, or fatal.
+ default: info
+ env variable: ETCD_LOG_LEVEL
+ 'default' use 'info'.

### --debug

**WARNING: to be deprecated in v3.5.**

+ Drop the default log level to DEBUG for all subpackages.
+ default: false (INFO for all packages)
+ env variable: ETCD_DEBUG

### --log-package-levels

**WARNING: to be deprecated in v3.5.**

+ Set individual etcd subpackages to specific log levels. An example being `etcdserver=WARNING,security=DEBUG`
+ default: "" (INFO for all packages)
+ env variable: ETCD_LOG_PACKAGE_LEVELS

## Unsafe flags

Please be CAUTIOUS when using unsafe flags because it will break the guarantees given by the consensus protocol.
For example, it may panic if other members in the cluster are still alive.
Follow the instructions when using these flags.

### --force-new-cluster
+ Force to create a new one-member cluster. It commits configuration changes forcing to remove all existing members in the cluster and add itself, but is strongly discouraged. Please review the [disaster recovery][recovery] documentation for preferred v3 recovery procedures.
+ default: false
+ env variable: ETCD_FORCE_NEW_CLUSTER

## Miscellaneous flags

### --version
+ Print the version and exit.
+ default: false

### --config-file
+ Load server configuration from a file. Note that if a configuration file is provided, other command line flags and environment variables will be ignored.
+ default: ""
+ example: [sample configuration file][sample-config-file]
+ env variable: ETCD_CONFIG_FILE

## Profiling flags

### --enable-pprof
+ Enable runtime profiling data via HTTP server. Address is at client URL + "/debug/pprof/"
+ default: false
+ env variable: ETCD_ENABLE_PPROF

### --metrics
+ Set level of detail for exported metrics, specify 'extensive' to include server side grpc histogram metrics.
+ default: basic
+ env variable: ETCD_METRICS

### --listen-metrics-urls
+ List of additional URLs to listen on that will respond to both the `/metrics` and `/health` endpoints
+ default: ""
+ env variable: ETCD_LISTEN_METRICS_URLS

## Auth flags

### --auth-token
+ Specify a token type and token specific options, especially for JWT. Its format is "type,var1=val1,var2=val2,...". Possible type is 'simple' or 'jwt'. Possible variables are 'sign-method' for specifying a sign method of jwt (its possible values are 'ES256', 'ES384', 'ES512', 'HS256', 'HS384', 'HS512', 'RS256', 'RS384', 'RS512', 'PS256', 'PS384', or 'PS512'), 'pub-key' for specifying a path to a public key for verifying jwt, 'priv-key' for specifying a path to a private key for signing jwt, and 'ttl' for specifying TTL of jwt tokens.
+ For asymmetric algorithms ('RS', 'PS', 'ES'), the public key is optional, as the private key contains enough information to both sign and verify tokens.
+ Example option of JWT: '--auth-token jwt,pub-key=app.rsa.pub,priv-key=app.rsa,sign-method=RS512,ttl=10m'
+ default: "simple"
+ env variable: ETCD_AUTH_TOKEN

### --bcrypt-cost
+ Specify the cost / strength of the bcrypt algorithm for hashing auth passwords. Valid values are between 4 and 31.
+ default: 10
+ env variable: (not supported)

### --auth-token-ttl
+ Time (in seconds) of the auth-token-ttl. Support `--auth-token=simple` model only.
+ default: 300
+ env variable: (not supported)

## Experimental flags

### --experimental-corrupt-check-time
+ Duration of time between cluster corruption check passes
+ default: 0s
+ env variable: ETCD_EXPERIMENTAL_CORRUPT_CHECK_TIME

### --experimental-compaction-batch-limit
+ Sets the maximum revisions deleted in each compaction batch.
+ default: 1000
+ env variable: ETCD_EXPERIMENTAL_COMPACTION_BATCH_LIMIT

### --experimental-peer-skip-client-san-verification
+ Skip verification of SAN field in client certificate for peer connections. This can be helpful e.g. if
cluster members run in different networks behind a NAT.

  In this case make sure to use peer certificates based on
a private certificate authority using `--peer-cert-file`, `--peer-key-file`, `--peer-trusted-ca-file`
+ default: false
+ env variable: ETCD_EXPERIMENTAL_PEER_SKIP_CLIENT_SAN_VERIFICATION

[build-cluster]: clustering#static
[discovery]: clustering#discovery
[iana-ports]: http://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.txt
[proxy]: /docs/v2.3/proxy
[reconfig]: runtime-configuration
[recovery]: ../recovery
[restore]: /docs/v2/admin_guide#restoring-a-backup
[sample-config-file]: https://github.com/etcd-io/etcd/blob/master/etcd.conf.yml.sample
[security]: ../security
[static bootstrap]: clustering#static
[systemd-intro]: http://freedesktop.org/wiki/Software/systemd/
[tuning]: ../tuning#time-parameters
