---
title: Versioning
weight: 4900
description: Semantic versioning with etcd 使用etcd进行语义版本控制
---

## Service versioning 服务版本

etcd使用[semantic versioning](http://semver.org)
etcd uses [semantic versioning](http://semver.org)
New minor versions may add additional features to the API.

Get the running etcd cluster version with `etcdctl`:

```sh
ETCDCTL_API=3 etcdctl --endpoints=127.0.0.1:2379 endpoint status
```

## API versioning

The `v3` API responses should not change after the 3.0.0 release but new features will be added over time.

