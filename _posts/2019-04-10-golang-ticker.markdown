---
layout: post
title:  "Go 定时器 (time.NewTicker)"
date:   2019-04-09 23:01:53 +0800
categories: Go
---


今天看到 "肥朝" 公众号一篇介绍 SpringBoot 中定时器的[__文章__](https://mp.weixin.qq.com/s/1IyXrkhCvG1hR21Vr1ttkA)，感觉挺有意思。

文章列举了 SpringBoot 定时器的三种模式: `fixedDelay`, `cron`, `fixedRate`, 那么 Go 语言中 `time.Ticker` 是哪一种呢？

```go
func main() {
	jobs := []int{60, 3, 6, 2, 2}

	logger.Infof("Ticker Created.")
	ticker := time.NewTicker(3 * time.Second)
	defer ticker.Stop()
	index := 0

	for t := range ticker.C {
		interval := jobs[index]
		logger.Infof("第 %v 个任务开始执行，收到的时间戳 %v, 时长 %v s", index, t, interval)
		time.Sleep(time.Duration(interval) * time.Second)
		index = index + 1
		if index == len(jobs) {
			break
		}
	}
}
// [INFO]:  2019/04/10 14:30:34 Ticker Created.
// [INFO]:  2019/04/10 14:30:37 第 0 个任务开始执行，收到的时间戳 2019-04-10 14:30:37.843782 +0800 CST m=+3.001299889, 时长 60 s
// [INFO]:  2019/04/10 14:31:37 第 1 个任务开始执行，收到的时间戳 2019-04-10 14:30:40.847492 +0800 CST m=+6.005127442, 时长 3 s
// [INFO]:  2019/04/10 14:31:40 第 2 个任务开始执行，收到的时间戳 2019-04-10 14:31:40.840862 +0800 CST m=+66.000836811, 时长 6 s
// [INFO]:  2019/04/10 14:31:46 第 3 个任务开始执行，收到的时间戳 2019-04-10 14:31:43.845576 +0800 CST m=+69.005667986, 时长 2 s
// [INFO]:  2019/04/10 14:31:49 第 4 个任务开始执行，收到的时间戳 2019-04-10 14:31:49.841825 +0800 CST m=+75.002150625, 时长 2 s
```

可以看到 Go 中的定时器相当于 SpringBoot 中的 `fixedRate` 模式。

而要达到 `fixedDelay` 效果，直接阻塞调用 `time.Sleep` 就可以了。

还有一个值得注意的点，ticker 创建完之后，不是马上就有一个 tick，第一个 tick 在 3 秒之后。Github 上有相关的[讨论](https://github.com/golang/go/issues/17601)。

查看 `NewTicker` 的源码，就比较容易理解为什么 Go 表现为 `fixedRate` 模式：

```go
func NewTicker(d Duration) *Ticker {
  if d <= 0 {
      panic(errors.New("non-positive interval for NewTicker"))
  }
  // Give the channel a 1-element time buffer.
  // If the client falls behind while reading, we drop ticks
  // on the floor until the client catches up.
  c := make(chan Time, 1)
  t := &Ticker{
      C: c,
      r: runtimeTimer{
          when:   when(d),
          period: int64(d),
          f:      sendTime,
          arg:    c,
      },
  }
  startTimer(&t.r)
  return t
}
```

因为 Channel 只能包含一个元素，runtime 往其中发送时间戳的时候，如果上一个没有消费完，发送方就会 block。

