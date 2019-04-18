---
layout: post
title:  "Kubernetes Pod 优雅退出"
date:   2019-04-15 09:01:53 +0800
categories: Go Kubernetes
---

## 可以采取的措施
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

**参考🔗**
* [Gruntwork Zero Downtime 系列](https://blog.gruntwork.io/zero-downtime-server-updates-for-your-kubernetes-cluster-902009df5b33)
* [Go: make ctrl-c cancel the context](https://medium.com/@matryer/make-ctrl-c-cancel-the-context-context-bd006a8ad6ff)

---
<br/>

#### sidenote: Graceful Restart in Go
应用的 Graceful Restart 比 Terminate 更加复杂一些。
在 Ruby Web 应用中，你基本上不需要考虑这些，因为 Puma 等 webserver 已经帮你处理了；但 Go 中这些情况需要自己处理。

> 主要要解决两个问题：
>
>  * 进程重启不需要关闭监听的端口
>
>  * 既有请求应当完全处理或者超时
>
>  https://colobu.com/2015/10/09/Linux-Signals/

具体方法：
1. Fork 一个子进程，与父进程侦听同一个端口
2. 通知父进程，做 graceful terminate

[endless](https://github.com/fvbock/endless) 是这个思路的具体实现

**参考🔗**
* [Graceful Restart in Golang](https://grisha.org/blog/2014/06/03/graceful-restart-in-golang/)

---
<br/>

#### sidenote: PodDisruptionBudget 与 RollingUpdate 的区别
* 两者都不能防止 delete 操作同时删除所有 replica
* Deployment Controller 关心 RollingUpdate, Eviction API 关心 PodDisruptionBudget

**参考🔗**
* [revolgy blog](https://www.revolgy.com/blog/kubernetes-in-production-poddisruptionbudget)

---
<br/>

#### sidenote: 不同信号之间的区别
* `SIGINT` Interruption: 在终端输入 Ctrl-C 时，系统发送的信号，可以被捕获
* `SIGTERM` Terminate: `kill` 命令默认发送的信号，与 `SIGINT` 相似，主要不同的就是一个在终端输入，另一个通过 `kill` 调用
* `SIGKILL`: `kill -9` 不能被捕获，没有办法做 Graceful Termination
* `SIGSTOP`: 在终端输入 Ctrl-Z，暂停进程执行

**参考🔗**
* [Quora 漫画](https://www.quora.com/What-is-the-difference-between-the-SIGINT-and-SIGTERM-signals-in-Linux-What%E2%80%99s-the-difference-between-the-SIGKILL-and-SIGSTOP-signals)