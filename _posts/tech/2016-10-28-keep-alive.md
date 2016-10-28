---
layout: post
title: http keep-alive测试
category: 技术
tags: Go
description: 
---
对于http的keep alive，我一直有一个疑问，即在开启keep alive下，同时多个请求进来，此时产生几个连接？为了逼真地模拟请求的同时性，这里我们可以在服务端进行一下处理，对于一个请求，sleep 10秒后再返回
而对于客户端，不同之处就是在for循环的过程中，使用goruoutine来同时请求，代码如下：

```go
	for i:=1;i<=10 ;i ++ {
		go Get("http://172.16.16.98:7777/test")
	}
```

上面测试的结果是，会产生10个不同的连接，也就是说在一个请求没有响应完成之前，有新的请求进来，还是会生成一个新的连接，如下：
[liziang@test-1 ~]$ netstat -an |grep 7777
tcp        0      0 192.168.1.15:50693      172.16.16.98:7777       ESTABLISHED
tcp        0      0 192.168.1.15:50696      172.16.16.98:7777       ESTABLISHED
tcp        0      0 192.168.1.15:50692      172.16.16.98:7777       ESTABLISHED
tcp        0      0 192.168.1.15:50694      172.16.16.98:7777       ESTABLISHED
tcp        0      0 192.168.1.15:50698      172.16.16.98:7777       ESTABLISHED
tcp        0      0 192.168.1.15:50695      172.16.16.98:7777       ESTABLISHED
tcp        0      0 192.168.1.15:50697      172.16.16.98:7777       ESTABLISHED
tcp        0      0 192.168.1.15:50701      172.16.16.98:7777       ESTABLISHED
tcp        0      0 192.168.1.15:50702      172.16.16.98:7777       ESTABLISHED
tcp        0      0 192.168.1.15:50699      172.16.16.98:7777       ESTABLISHED


然而在下面的情况下，只会产生一个链接
```go
	for i:=1;i<=10 ;i ++ {
		Get("http://172.16.16.98:7777/test")
	}
```
[liziang@test-1 ~]$ netstat -an |grep 7777
tcp        0      0 192.168.1.15:50760      172.16.16.98:7777       ESTABLISHED
[liziang@test-1 ~]$ netstat -an |grep 7777
tcp        0      0 192.168.1.15:50760      172.16.16.98:7777       ESTABLISHED