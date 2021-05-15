---
title: Set up a local cluster
weight: 3125
description: Configuring local clusters for testing and development
---
为了测试和开发，最快和最简单的方法是配置本地集群。对于线上产品开发，参考这里[clustering][clustering]。

## Local standalone cluster

### Starting a cluster

运行下面的命令开启一个只有一个节点的集群:

```
$ ./etcd
...
```

如果`etcd`程序不在当前的工作目录，它也许在`$GOPATH/bin/etcd`或者`/usr/local/bin/etcd`地方，你要在合适的地方运行。

etcd运行后监听`localhost:2379`

### Interacting with the cluster

使用`etcdctl`来和集群交互:

1. 创建一个key-value对:

    ```
      $ ./etcdctl put foo bar
      OK
    ```

    如果打印了OK就是创建成功了。

2. 获取 `foo` 的value:

    ```
    $ ./etcdctl get foo
    bar
    ```

    如果返回`bar`，说明你又对了。

## 本地多成员集群

### 开始一个集群


在etcd的git仓库里提供了一个文件[`Procfile`](https://github.com/etcd-io/etcd/blob/main/Procfile)，可以轻松配置本地多成员集群。为了开始一个多成员集群，导航到etcd根目录然后执行下面的操作：

1. 安装`goreman`去控制基于Procfile的程序：

    ```
    $ go get github.com/mattn/goreman
    ```

2. 使用`goreman`程序加etcd给的文件Procfile来开始一个集群：
    

    ```
    $ goreman -f Procfile start
    ```

    所有的成员开始运行。他们分别在`localhost:2379`,`localhost:22379`和`localhost:32379`上监听客户端请求。
 
### 和集群交互

Use `etcdctl` to interact with the running cluster:
使用`etcdctl`和集群交互：

1. 打印成员列表：

    ```
    $ etcdctl --write-out=table --endpoints=localhost:2379 member list
    ```
    etcd的成员显示在下面：

    ```
    +------------------+---------+--------+------------------------+------------------------+
    |        ID        | STATUS  |  NAME  |       PEER ADDRS       |      CLIENT ADDRS      |
    +------------------+---------+--------+------------------------+------------------------+
    | 8211f1d0f64f3269 | started | infra1 | http://127.0.0.1:2380  | http://127.0.0.1:2379  |
    | 91bc3c398fb3c146 | started | infra2 | http://127.0.0.1:22380 | http://127.0.0.1:22379 |
    | fd422379fda50e48 | started | infra3 | http://127.0.0.1:32380 | http://127.0.0.1:32379 |
    +------------------+---------+--------+------------------------+------------------------+
    ```

2. 在集群中保存key-value对的例子：

    ```
    $ etcdctl put foo bar
    OK
    ```

    如果打印了OK，那么存储的key-value对就成功了。

### 测试容错机制

为了练习etcd的容错，kill一个成员然后读取key的value。

1. 找出想要停止的进程的成员名称。

    The `Procfile` lists the properties of the multi-member cluster. For example, consider the member with the process name, `etcd2`.
    `Procfile`文件列出了多成员集群的属性。例如，考虑进程名字叫`etcd2`的成员。

2. 停止这个成员：

    ```
    # kill etcd2
    $ goreman run stop etcd2
    ```

3. 保存key：

    ```
    $ etcdctl put key hello
    OK
    ```

4. 获取上一步存储的key的value：

    ```
    $ etcdctl get key
    hello
    ```

5. 从停止的成员获取key：

    ```
    $ etcdctl --endpoints=localhost:22379 get key
    ```

    这个命令应该会显示一个连接失败的错误：

    ```
    2017/06/18 23:07:35 grpc: Conn.resetTransport failed to create client transport: connection error: desc = "transport: dial tcp 127.0.0.1:22379: getsockopt: connection refused"; Reconnecting to "localhost:22379"
    Error:  grpc: timed out trying to connect
    ```
6. 重启停止的成员：

    ```
    $ goreman run restart etcd2
    ```

7. 从重启的成员上get这个key：

    ```
    $ etcdctl --endpoints=localhost:22379 get key
    hello
    ```

    重启这个成员来重建这个连接。 现在使用`etcdctl`就能成功获取key的value。为了更多的学习与etcd交互，请读取[与etcd部分交互][interacting]。

[clustering]: ../op-guide/clustering
[interacting]: ./interacting_v3
