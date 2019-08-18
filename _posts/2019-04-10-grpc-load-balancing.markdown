---
layout: post
title:  "gRPC 负载均衡"
date:   2019-04-09 09:00:53 +0800
categories: gRPC network
---

## gRPC 负载均衡有什么特别的

gRPC 可以在 Layer 4 或者 Layer 7 做负载均衡。 但是如果在 Layer 4 上面做，会有负载不均衡的问题（见下文）。

考虑一下普通的 HTTP 负载均衡，一个 TCP 链接只有一对请求/响应：

```
1. 客户端与 Load Balancer 之间建立链接，发送请求
2. Load Balancer 运行负载均衡算法，选定后端服务器，转发请求/响应
4. 客户端收到响应后断开链接
```

当一个客户端同时发送大量请求的时候，会建立同样数量的链接 (假设客户端没有限制连接数)，Load Balancer 可以把链接分配到不同的后端服务器。

因为 gRPC 采取了 HTTP/2，在一个长链接中可以收发多个请求/响应。一旦客户端建立完链接，发送大量请求，这些请求都会通过同一个链接到达后端服务器，无法达到负载均衡的效果。

不同的客户端建立的链接可能会到达不同的后端服务器。这时 gRPC Layer 7 负载均衡的效果和 Layer 4 类似。


## gRPC 自身提供的方案
name resolver + balance policy:

* [grpc/blob/master/doc/load-balancing](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)
* [grpc blog](https://grpc.io/blog/loadbalancing)
* [另外一篇介绍 gRPC Load Balancing 的文章](https://itnext.io/on-grpc-load-balancing-683257c5b7b3)

需要部署一个额外的 Load Balancer, 例如 [grpclb](https://github.com/bsm/grpclb)

同时 Client 需要知晓 Load Balancer 的存在，通过 name resolver (例如 DNS) 查询后端服务器的地址，然后通过 Load Balancer 建立的 sub-channel 分发请求。

Sub-channel 的概念应该与 RabbitMQ Channel 的概念比较类似，通过复用 TCP 链接，Client 就好像同时链接上所有的后端服务器。

## 基于请求的负载均衡: Linkerd

参考 [Linkerd 使用方法](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/#grpc-load-balancing-on-kubernetes-with-linkerd)


#### Link Dump
* [Gist](https://gist.github.com/bojand/6a604f7e369d7c7d8c39eb77878a42c2)

{% include /disqus.html %}