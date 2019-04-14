---
layout: post
title:  "使用 tcpdump 与 wireshark 诊断网络问题"
date:   2019-04-13 09:00:00 +0800
categories: network wireshark
---

Wireshark 是一款很强大的网络分析软件。一个最常见的使用场景是判断客户端与服务器连接中哪一端出了问题。

公司有一个行情收集的应用，会从 Binance 交易所的 Websocket 接口获取价格更新。最近一次部署后，在 AWS 美区的 Staging 环境工作正常，新加坡区的 Production 环境却一直收不到数据。

我们怀疑是 Binance 的接口屏蔽了特定地理区域的访问。

用 tcpdump 在应用上收集了一段时间的流量: `tcpdump -w output.pcap`，把文件拷贝到本地，通过 Wireshark 打开。

1. Binance 的 Websocket 端口是 9443，所以先在 Wireshark 的筛选框内选出包含这个端口的流量: `tcp.port == 9443`

2. 右击其中的一个 packet，选择 Conversation Filter -> TCP。这个菜单帮你做的，就是在筛选框内填上 src/dest 的 ip + port，查询这个 TCP 会话的所有 packet。

![wireshark-conversation-filter](/assets/wireshark-conversation-filter.png)

筛选后的结果:

![wireshark-filtered](/assets/wireshark-filtered.png)

注意看 time 这一列，从第 14 秒到第 24 秒之间没有数据包，然后我们的客户端 (100.xxx.xxx.xxx) 主动断开了链接，不是 Binance 服务器拒绝。

而且这个模式看起来是客户端超时造成的，调查代码确实有设置 `ReadTimeout`。

明确了责任方，进一步调查的范围就缩小了很多。

---
#### 参考链接
* [Julia Evans: How I use Wireshark](https://jvns.ca/blog/2018/06/19/what-i-use-wireshark-for/)
