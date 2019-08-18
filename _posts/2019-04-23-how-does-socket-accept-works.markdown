---
layout: post
title:  "TIL: Socket Accept 是如何工作的"
date:   2019-04-23 09:00:53 +0800
categories: network til go
---

Socket 是由 **ClientIP : ClientPort + ServerIP : ServerPort + Protocol** 组成的。了解到这一点，就容易理解为什么 Web Server 能通过一个端口，与许多客户端通信。因为通信是在 connection 层面，不是在 port 层面的。

```go
ln, err := net.Listen("tcp", ":8080")
if err != nil {
  // handle error
}
for {
  conn, err := ln.Accept()
  if err != nil {
    // handle error
  }
  go handleConnection(conn)
}
```

[StackOverflow: How does the socket API accept() function work?](https://stackoverflow.com/questions/489036/how-does-the-socket-api-accept-function-work)

{% include /disqus.html %}