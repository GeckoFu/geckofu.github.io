---
layout: post
title:  "Redis 协议简介"
date:   2019-04-18 09:00:53 +0800
categories: network redis load-balance
---



在看[有赞 KV 架构](https://tech.youzan.com/shi-yong-kai-yuan-ji-zhu-gou-jian-you-zan-fen-bu-shi-kvcun-chu-fu-wu/)的时候，对其中的 Redis Proxy 产生了兴趣。

Proxy 可以做流控，负载均衡，可以屏蔽底层存储的细节；为了实现 Proxy，首先需要能够理解接收到的字节流数据，即通信协议。

Redis 的协议在简洁，可读性和性能之间达到了很好的平衡。

其中有一个很好的设计选择，是在 String, Array 类型的数据信息前，添加长度信息:

```bash
# `$` 是 bulk string 的标识前缀
# `13` 是字符串的字节数
# `\r\n` 是字符串的结束符
$13\r\nHello, Word!\r\n
```

有了长度信息，客户端只需要读取对应的字节数据就完事儿了。不需要做 Escaping。解析器因此也不需要复杂的模型设计。

Dead Simple.

在没有长度信息的时候，你需要约定结束符，例如 `\r\n`，然后解析每个字符，判断是否匹配到结束符。

Array 的长度信息与 String 表达的含义不一样，它表达的是数组的长度：

```bash
*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
```

只需要按照长度信息，依次识别其中的 String 字节就行了，也不需要复杂的模型。

---

**参考🔗**
* [RedisGreen](https://www.redisgreen.net/blog/beginners-guide-to-redis-protocol/)
* [Redis Protocol](https://redis.io/topics/protocol)