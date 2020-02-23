---
title: tcp三次握手和四次挥手
categories: tcp学习
date: 2016-12-12 12:38:11
layout: post
# you can override the settings in _config.yml here !!
---
>

# 一. 预备工作

### 测试server代码

``` ruby
import socket
import sys
import os
addr = ('127.0.0.1', 9988)
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(addr)
server.listen(10)

while True:
        connection, address = server.accept()
        print 'connection ip:', address
```

### 测试client代码


``` ruby
from socket import *
import time

addr = ('127.0.0.1', 9988)
client = socket(AF_INET, SOCK_STREAM)
client.connect(addr)
```

### 抓包命令

``` ruby
tcpdump -i lo port 9988 -S
```

### 抓包数据

``` ruby
16:34:33.611019 IP localhost.41631 > localhost.nsesrvr: Flags [S], seq 2970024578, win 32792, options [mss 16396,sackOK,TS val 514254463 ecr 0,nop,wscale 7], length 0
16:34:33.611035 IP localhost.nsesrvr > localhost.41631: Flags [S.], seq 2032818397, ack 2970024579, win 32768, options [mss 16396,sackOK,TS val 514254463 ecr 514254463,nop,wscale 7], length 0
16:34:33.611045 IP localhost.41631 > localhost.nsesrvr: Flags [.], ack 2032818398, win 257, options [nop,nop,TS val 514254463 ecr 514254463], length 0
# == 三次握手结束==

# == 四次挥手开始＝＝
16:34:33.611150 IP localhost.nsesrvr > localhost.41474: Flags [F.], seq 507613731, ack 3023763844, win 256, options [nop,nop,TS val 514254463 ecr 514208493], length 0
16:34:33.611163 IP localhost.41474 > localhost.nsesrvr: Flags [.], ack 507613732, win 257, options [nop,nop,TS val 514254463 ecr 514254463], length 0
16:34:33.612059 IP localhost.41631 > localhost.nsesrvr: Flags [F.], seq 2970024579, ack 2032818398, win 257, options [nop,nop,TS val 514254464 ecr 514254463], length 0
16:34:33.612718 IP localhost.nsesrvr > localhost.41631: Flags [.], ack 2970024580, win 256, options [nop,nop,TS val 514254465 ecr 514254464], length 0
```

# 二. 三次握手

### 流程图

![tcp](/assets/img/tcp/tcp3.png)

### 中间的标志位

> S=SYN，发起连接标志。
> P=PUSH，传送数据标志。
> F=FIN，关闭连接标志。
> ack，表示确认包。
> RST=RESET，异常关闭连接。
> .，表示没有任何标志。


### 1. 三次握手具体分析

1. 第1行：16:34:33.611019，从localhost（client）的临时端口41631向 localhost.nsesrvr（server）的9988监听端口发起连接，client初始包序号seq为2970024578，滑动窗口大小为32792字节（滑动窗口即tcp接收缓冲区的大小，用于tcp拥塞控制），mss大小为16396（即可接收的最大包长度，通常为MTU减40字节，IP头和TCP头各20字节）。【seq= 2970024578，ack=0，syn=1】
2. 第2行：16:34:33.611035，server响应连接，同时带上第一个包的ack信息，为client端的初始包序号seq加1，即1944916151，即server端下次等待接受这个包序号的包，用于tcp字节流的顺序控制。Server端的初始包序号seq为2032818397，mss也是16396。【seq= 2032818397，ack= 2970024579，syn=1】
3. 第3行：16:34:33.611045 ，client再次发送确认连接，tcp连接三次握手完成，等待传输数据包。【ack= 2032818398，seq= 2970024579】

### 2. 为什么是三次握手而不是两次?

> 对于step3的作用，假设一种情况，客户端A向服务器B发送一个连接请求数据报，然后这个数据报在网络中滞留导致其迟到了，虽然迟到了，但是服务器仍然会接收并发回一个确认数据报。但是A却因为久久收不到B的确认而将发送的请求连接置为失效，等到一段时间后，接到B发送过来的确认，A认为自己现在没有发送连接，而B却一直以为连接成功了，于是一直在等待A的动作，而A将不会有任何的动作了。这会导致服务器资源白白浪费掉了，因此，两次握手是不行的，因此需要再加上一次，对B发过来的确认再进行一次确认，即确认这次连接是有效的，从而建立连接。

### 3. 对于双方，发送序号的初始化为何值?

> 有的系统中是显式的初始化序号是0，但是这种已知的初始化值是非常危险的，因为这会使得一些黑客钻漏洞，发送一些数据报来破坏连接。因此，初始化序号因为取随机数会更好一些，并且是越随机越安全。

# 二. 四次挥手

连接双方在完成数据传输之后就需要断开连接。由于TCP连接是属于全双工的，即连接双方可以在一条TCP连接上互相传输数据，因此在断开时存在一个半关闭状态，即有有一方失去发送数据的能力，却还能接收数据。因此，断开连接需要分为四次。主要过程如下：

![tcp](/assets/img/tcp/tcp4.png)

> step1. 主机A向主机B发起断开连接请求，之后主机A进入FIN-WAIT-1状态；
> step2. 主机B收到主机A的请求后，向主机A发回确认，然后进入CLOSE-WAIT状态；
> step3. 主机A收到B的确认之后，进入FIN-WAIT-2状态，此时便是半关闭状态，即主机A失去发送能力，但是主机B却还能向A发送数据，并且A可以接收数据。此时主机B占主导位置了，如果需要继续关闭则需要主机B来操作了；
> step4. 主机B向A发出断开连接请求，然后进入LAST-ACK状态；
> step5. 主机A接收到请求后发送确认，进入TIME-WAIT状态，等待2MSL之后进入CLOSED状态，而主机B则在接受到确认后进入CLOSED状态；

### 为何主机A在发送了最后的确认后没有进入CLOSED状态，反而进入了一个等待2MSL的TIME-WAIT?

* 第一，确保主机A最后发送的确认能够到达主机B。如果处于LAST-ACK状态的主机B一直收不到来自主机A的确认，它会重传断开连接请求，然后主机A就可以有足够的时间去再次发送确认。但是这也只能尽最大力量来确保能够正常断开，如果主机A的确认总是在网络中滞留失效，从而超过了2MSL，最后也无法正常断开；
* 第二，如果主机A在发送了确认之后立即进入CLOSED状态。假设之后主机A再次向主机B发送一条连接请求，而这条连接请求比之前的确认报文更早地到达主机B，则会使得主机B以为这条连接请求是在旧的连接中A发出的报文，并不看成是一条新的连接请求了，即使得这个连接请求失效了，增加2MSL的时间可以使得这个失效的连接请求报文作废，这样才不影响下次新的连接请求中出现失效的连接请求。

### 为什么断开连接请求报文只有三个，而不是四个?

因为在TCP连接过程中，确认的发送有一个延时（即经受延时的确认），一端在发送确认的时候将等待一段时间，如果自己在这段事件内也有数据要发送，就跟确认一起发送，如果没有，则确认单独发送。而我们的抓包实验中，由服务器端先断开连接，之后客户端在确认的延迟时间内，也有请求断开连接需要发送，于是就与上次确认一起发送，因此就只有三个数据报了。

# 整体的图

![tcp](/assets/img/tcp/tcp34.png)


# 参考文章：

[https://my.oschina.net/xianggao/blog/678644](https://my.oschina.net/xianggao/blog/678644)
[http://coolshell.cn/articles/11564.html](http://coolshell.cn/articles/11564.html)
[http://www.cnblogs.com/chobits/archive/2012/08/29/2662336.html](http://www.cnblogs.com/chobits/archive/2012/08/29/2662336.html)
