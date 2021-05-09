---
title: System limits
weight: 3625
description: "etcd limits: requests and storage"
---

## 请求的大小限制

etcd是用来处理小的jkey value的。大的请求也会工作，但是会增加其他请求的延时。
默认最大的请求是1.5M。可以通过配置启动参数`--max-request-bytes`来修改它。

## 存储大小限制

默认的存储是2GB，可以通过`--quota-backend-bytes`来配置。正式环境推荐的最大大小是8GB，如果配置超过了8GB，etcd就会在启动是发出警告。

