+++
title = "GeekTime -《网络编程实战-提高篇》笔记"
date = '2019-06-22'
tags = ["笔记", "极客时间", "网络编程"]
categories = ["极客时间"]
+++

### 10 TIME-WAIT

- TCP连接的四次挥手

![Untitled2.png](/image/GeekTime-《网络编程实战-提高篇》笔记/Untitled2.png)

- TIME_WAIT停留程序时间是固定的，是最长分节生命期MSL，maximum segment lifetime的两倍，一般称之为2MSL
- 只有发起连接终止的一方会进入TIME_WAIT状态（面试常问）
- TIME_WAIT的作用
  - 为了确保最后的ACK能让被动关闭方接收，从而帮助其正常关闭
  - 为了让旧连接的重复分节在网络中自然消失。经过2MSL时间，足以让两个方向上的分组都被丢弃，使得原来连接的分组在网络中都消失，再出现的分组一定都是新化身所产生的
- 2MSL的时间是从主机1接收到FIN后发送ACK开始计时的
- TIME_WAIT的危害
  - 内存资源占用
  - 对端口资源的占用，一个TCP连接至少消耗一个本地端口
- 优化TIME_WAIT
  - 调低TCP_TIMEWAIT_LEN，重新编译系统
  - SO_LINGER的设置，设置调用close或shutdown关闭连接时的行为
  - net.ipv4.tcp_tw_reuse，可以复用处于TIME_WAIT的套接字为新的连接所用，需要打开对TCP时间戳的支持，即net.ipv4.tcp_timestamps=1

### 11 关闭连接

- close函数

```c
int close(int sockfd)
```

- close函数会对套接字引用计数减一，引用计数到0，则会对套接字进行彻底释放，并且会关闭TCP两个方向的数据流
- shutdown函数

```c
int shutdown(int sockfd, int howto)
```

- shutdown函数可以关闭单方向的连接
- 期望关闭连接其中一个方向时，应该使用shutdown函数

### 12 使用Keep-Alive还是应用心跳来检测？

- TCP的保持活跃的机制，keep-alive
- 机制的原理，定义一个时间段，在这个时间段内，如果没有任何连接相关的活动，TCP保活机制会开始作用，每隔一个时间间隔，发送一个探测报文，报文包含的数据非常少，如果连续几个探测报文都没有得到响应，则认为当前的TCP连接已经死亡，系统内核将错误信息通知给上层应用程序
- 可以通过在应用程序设计一个ping-pong机制，来模拟TCP keep-alive机制，完成在应用层的连接探活

### 13 TCP协议中的动态数据传输

- 发送窗口用来控制发送和接收端的流量；阻塞窗口用来控制多条连接公平使用的有限带宽
- 小数据包加剧了网络带宽的浪费，为了解决这个问题，引入了如Nagle算法，延时ACK等机制
- 在程序设计层面，不要多次频繁地发送小报文，可以考虑使用writev批量发送

### 14 UDP也可以是“已连接”

- UDP套接字的connect操作和TCP的connect操作作用不同
- 如果UDP套接字不调用connect函数，则发送端不会报错，只会阻塞在recvfrom上，等待返回或超时
- UDP套接字调用connect方法，可以绑定本地地址和端口，为了让程序可以快速获取异步错误信息的通知，同时也可以获得一定性能上的提升

### 15 “地址已经被使用”？

- 出现`address already in use`一般是因为服务器端主动关闭连接，导致地址+端口的TCP连接处于TIME_WAIT的状态。服务器重启时继续绑定在原地址+端口上
- 最佳实践：在所有TCP服务器程序中，调用bind之前请设置SO_REUSEADDR套接字选项

### 16 如何理解TCP的“流”?

- readn函数

  - 读取报文预设大小的字节，readn调用会一直循环，尝试读取预设大小的字节，如果接收缓冲区数据为空，readn函数会阻塞在那里，直到有数据到达

- read_message函数

  - 调用多次readn函数，用于读取消息中每个固定长度的数据内容

- 两种报文格式

  - 发送端把要发送的报文长度预先通过报文告知给接收端
  - 通过一些特殊字符来进行边界的划分

- 显式编码报文长度

  ![Untitled3.png](/image/GeekTime-《网络编程实战-提高篇》笔记/Untitled3.png)

- 特殊字符作为边界

  ![Untitled4.png](/image/GeekTime-《网络编程实战-提高篇》笔记/Untitled4.png)

  - HTTP就是通过设置回车符，换行符作为HTTP报文协议的边界

### 17 TCP并不总是“可靠”的？

- 故障模式总结
  - 第一类，对端无FIN包发送出来的情况
    - 网络中断造成的对端无FIN包
    - 系统崩溃造成的对端无FIN包
  - 第二类，是对端有FIN包发送出来
    - read直接感知FIN包
    - 通过write产生RST，read调用感知RST
    - 向一个已关闭连接连续写，最终导致SUGPIPE

### 19 如何理解TCP的四次挥手？

- TCP协议栈会为FIN包插入一个文件结束符EOF到接收缓冲区中，EOF会被放在已排队等候的其他已接收的数据之后
- MSL是任何IP数据报能够在因特网存活的最长时间
- 每个数据报都包含一个被称为TTL，time to live的8位段，最大值255，每经过一跳就减一，减为0时所在路由器会将其丢弃

