---
layout: post
title:  "Kubernetes Pod 优雅退出"
date:   2019-04-15 09:01:53 +0800
categories: Go Kubernetes
---

**可以采取的措施**
* Pod 设置 `preStop` lifecycle hook
* 如果应用通过 Service 提供服务，在 `preStop` 中 sleep 一段时间，让 Endpoint Controller 有时间删除 Pod IP，切断流量
* 应用程序捕获 SIGTERM 信号，清理资源
* 设置 `PodDisruptionBudget`，确保服务在线

```go
// 应用程序捕获 SIGTERM 举例: Golang context
//
func main() {
  ctx := context.Background()
  ctx, cancel := context.WithCancel(ctx)

	c := make(chan os.Signal, 1)
  signal.Notify(c, os.Interrupt)
  
  go func() {
		select {
		case <-c:
			cancel()
		case <-ctx.Done():
		}
  }()

  doSomethingAwesome(ctx)
}
```

---


**参考🔗**
* [Gruntwork Zero Downtime 系列](https://blog.gruntwork.io/zero-downtime-server-updates-for-your-kubernetes-cluster-902009df5b33)
* [Go: make ctrl-c cancel the context](https://medium.com/@matryer/make-ctrl-c-cancel-the-context-context-bd006a8ad6ff)