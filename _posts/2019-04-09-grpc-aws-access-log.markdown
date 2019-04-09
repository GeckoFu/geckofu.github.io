---
layout: post
title:  "使用 AWS ELB Access Log 定位网络问题"
date:   2019-04-09 09:00:53 +0800
categories: gRPC network
---


我们在 Kubernetes 集群上有一个 gRPC 服务, 通过 ELB 四层代理提供外网白名单访问。
最近这个服务一直频繁重启，初步发现的问题是应用建立的连接数过多，导致的 OOM。

通过 `netstat -n` 可以查看连接的客户端 IP，绝大部分 IP 都属于 Kubernetes CIDR 网段，但是这些地址都不属于 Pod/Service。

我们猜测这些链接都来自集群外部，`netstat` 显示的 IP 属于 load balancer。

那么如何通过这些内网 IP 找到调用方？ 答案是 ELB Access Log。

在 AWS EC2 Web Console 中，可以打开 ELB 的 Access Log 功能，日志会输出到指定的 S3 Bucket 中。
日志包含了很多有用的信息，比如: 
  * ELB 转发到 Target 花费的时间
  * Target 处理请求消耗的时间
  * **Client ip 与 port**

详细日志格式信息可以参看 [AWS 文档](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html)

通过日志我们找到了异常流量来自于另一个集群的机器，随即定位到了这台机器上的应用。

---
### 事后分析
* **还可以通过 gRPC 记录客户端 IP**

```go
import("google.golang.org/grpc/peer")
p, _ := peer.FromContext(ctx)
fmt.Println(p.Addr)
```

* **Kubernetes gRPC Load Balance**

gRPC 使用基于 HTTP2 的长链接，之前 connection-based load balance 需要转换成 request-based load balance
[Kubernetes gRPC Load Balance with Linkerd](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/)

* **客户端限流 (Rate Limit)**

应该限制每个客户端能建立的最大连接数
[gRPC Support with Nginx](https://www.nginx.com/blog/nginx-1-13-10-grpc/)