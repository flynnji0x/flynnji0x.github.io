+++
title = "GeekTime -《网络编程实战-基础篇》笔记"
date = '2019-06-07'
tags = ["笔记", "极客时间", "网络编程"]
categories = ["极客时间"]
+++
# 网络编程实战-基础篇

### 02 网络编程模型

- 需要强调的是，无论是客户端，还是服务端，它们运行的单位都是进程，process，而不是机器。

- 在tcp/ip协议栈中，ip用来表示网络世界的地址。

- 端口号是一个16位的整数，最多为2^16，65536

- 客户端的端口是由操作系统内核临时分配的，称为临时端口；而服务端的端口通常是一个众所周知的端口

- 一个连接可以通过客户端-服务端的ip和端口号唯一确定，叫做套接字对，用下面的四元组表示：

  ```jsx
  (clientaddr:clientport,serveraddr:serverport)
  ```

  - TCP，又被叫做字节流套接字，Stream Socket；UDP也有类似叫法，数据报套接字，Datagram Socket
  - 一般分别以SOCK_STRAM与SOCK_DGRAM分别表示TCP和UDP套接字

  ### 03 套接字和地址

  - 网络编程中客户端和服务端的核心逻辑（重点掌握）

  ![Untitled.png](/image/GeekTime-《网络编程实战-基础篇》笔记/Untitled.png)

  - TCP三次握手，three-way handshake

  - 一旦连接建立，数据的传输就不再是单向的，而是双向的，这也是tcp的一个显著特性

  - socket是我们用来建立连接，传输数据的唯一途径

  - 套接字地址格式

    - 通用地址格式

    ```c
    1 /* POSIX.1g 规范规定了地址族为 2 字节的值.*/
    2 typedef unsigned short int sa_family_t; 
    3 /* 描述通用套接字地址 */ 
    4 struct sockaddr{ 
    5 sa_family_t sa_family; /* 地址族. 16-bit*/ 
    6 char sa_data[14]; /* 具体的地址值 112-bit */ 
    7 };
    ```

    - 地址族有以下几种：
      - AF_LOCAL，表示本地地址，一般用于本地socket通信，也可以写成，AFZ_UNIX,AF_FILE
      - AF_INET，因特网使用的ipv4地址
      - AF_INET6，因特网使用的ipv6地址
    - AF_表示，address family，地址族；PF_表示protocol family，协议族

### 04 怎么使用套接字格式建立连接？

- 创建套接字

```c
int socket(int domain, int type, int protocol)
```

- domain，指的是PF_INET,PF_INET6和PF_LOCAL等，表示什么样的套接字

- type可用值如下：

  - SOCK_STREAM，表示的是字节流，对应TCP
  - SOCKZ_DGRAM，表示的是数据报，对应UDP
  - SOCK_RAW，表示的是原始套接字

- protocol，基本废弃，一般写成0即可

- bind函数

  ```c
  bind(int fd, sockaddr * addr, socklen_t len)
  ```

  - 对于使用者，需要将IPv4，IPv6或本地套接字格式转化为通用套接字格式：

    ```c
    struct sockaddr_in name;
    bind (sock, (struct sockaddr *) &name, sizeof (name)
    ```

    - 对于实现者，可工具地址结构的前两个字节判断出是哪种地址
    - 通配地址的配置：

    ```c
    struct sockaddr_in name;
    name.sin_addr.s_addr = htonl (INADDR_ANY); /* IPV4 通配地址 */
    ```

- listen函数

  - 初始化创建的套接字，可以认为是一个”主动“套接字，其目的是之后主动发起请求（通过connect函数），通过listen函数可以转换为”被动“套接字

  ```c
  int listen (int socketfd, int backlog)
  ```

- accept函数

  - 作用是连接建立之后，操作系统内核和应用程序之间的桥梁：

  ```c
  int accept(int listensockfd, struct sockaddr *cliaddr, socklen_t *addrlen)
  ```

  - 完成了TCP三次握手之后，操作系统内核会为客户生成一个已连接套接字，应用程序使用这个已连接套接字和客户进行通信。而监听套接字一直都处于”监听“状态

- connect函数

  - 客户端与服务端的连接建立，是通过connect函数完成的：

  ```c
  int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen)
  ```

  - 调用connect前不必非得调用bind函数，因为内核会确定源ip地址，并按照一定的算法选择一个临时端口作为源端口
  - 调用connect函数会激发TCP三次握手过程

  ![Untitled1.png](/image/GeekTime-《网络编程实战-基础篇》笔记/Untitled1.png)

### 05 使用套接字进行读写

- 发送数据
  - 常用的三个发送数据的函数

```c
ssize_t write (int socketfd, const void *buffer, size_t size)
ssize_t send (int socketfd, const void *buffer, size_t size, int flags)
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags)
```

- 发送缓存区

  - 当应用程序调用write函数时，实际做的事情是把数据从应用程序中拷贝到操作系统内核的发送缓冲区中，并不一定是把数据通过套接字写出去

- 读取数据

  ```c
  ssize_t read (int socketfd, void *buffer, size_t size)
  ```

  - read函数要求操作系统内核从套接字描述字socketfd读取最多多少个字节，并将结果存储到buffer中，返回值告诉我们实际读取的字节数目

- 在UNIX的世界里万物都是文件

- 阻塞式套接字最终发送返回的实际写入字节数和请求字节数是相等的

- 发送成功仅仅表示的是数据被拷贝到了发送缓冲区中，并不意味着连接对端已经收到所有的数据，至于什么时候发送到对端的接受缓冲区，或者更进一步说，什么时候被对方应用程序缓存所接收，对我们而言完全都是透明的

- 对于send来说，返回成功仅仅表示数据写到发送缓冲区成功，并不表示对端已经成功收到

- 对于read来说，需要循环读取数据，并且需要考虑EOF等异常条件

### 06 UDP编程

- recvfrom和sendto是UDP用来接收和发送报文的两个主要函数：

```c
#include <sys/socket.h>
 
ssize_t recvfrom(int sockfd, void *buff, size_t nbytes, int flags, 
　　　　　　　　　　struct sockaddr *from, socklen_t *addrlen); 
 
ssize_t sendto(int sockfd, const void *buff, size_t nbytes, int flags,
                const struct sockaddr *to, socklen_t *addrlen);
```

- UDP是无连接的数据报程序，和TCP不同，不需要三次握手建立一条连接
- UDP程序通过recvfrom和sendto函数直接收发数据报报文

### 07 本地套接字

- 实现进程间通信的常用方法，如本地套接字，管道，共享消息队列等

### 08 工具

- ping，用来完成对网络连通性的探测

- ifconfig，用来显示当前系统中的所有网络设备，也就是网卡列表
- netstat，了解当前系统的网络连接状况
- lsof，找出在指定的IP地址hhuoz端口上打开套接字的进程
- tcpdump，抓包工具，用于查看报文，排查问题
