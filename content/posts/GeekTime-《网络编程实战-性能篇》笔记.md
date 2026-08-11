+++
title = "GeekTime -《网络编程实战-性能篇》笔记"
date = '2019-07-26'
tags = ["笔记", "极客时间", "网络编程"]
categories = ["极客时间"]
+++

### 20 select

- I/O事件类型

  - 标准输入文件描述符准备好可以读
  - 监听套接字准备好，新的连接已经建立成功
  - 已连接套接字准备好可以写
  - I/O事件超时

- select函数

  ```c
  int select(int maxfd, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);
   
  返回：若有就绪描述符则为其数目，若超时则为 0，若出错则为 -1
  ```

  - readset，读描述符集合，在指定描述符上检测数据可以读
  - writeset，写描述符集合，在指定描述符上检测数据可以写
  - exceptset，异常描述符集合，检测有异常发生
  - 宏
    - FD_ZERO，将描述符都设置为0
    - FD_SET，将a[fd]设置为1
    - FD_CLR，将a[fd]设置为0
    - FD_ISSET，判断a[fd]是0还是1
  - 0代表不需要处理，1代表需要处理
  - 内核通知套接字有数据可以读，使用read函数不会被阻塞
  - 内核通知套接字可以往里写，使用write函数不会阻塞
  - 套接字可写，完全是基于套接字本身的特性来说的
    - 套接字发送缓冲区足够大，进行write操作，不会阻塞，直接返回
    - 写的半边已关闭，继续写操作会产生SIGPIPE信号
    - 套接字上有错误待处理，进行write操作，不会阻塞，返回-1

### 21 poll

- select有一个缺点，就是所支持的文件描述符个数是有限的，在linux中select默认最大值为1024

- poll是除了select之外，另一种普遍使用的I/O多路复用技术，和select相比，它和内核交互的数据结构有所变化，另外也突破了文件描述符的个数限制

- poll函数

  ```c
  int poll(struct pollfd *fds, unsigned long nfds, int timeout); 
  　　　
  返回值：若有就绪描述符则为其数目，若超时则为 0，若出错则为 -1
  ```

- pollfd结构

  ```c
  struct pollfd {
      int    fd;       /* file descriptor */
      short  events;   /* events to look for */
      short  revents;  /* events returned */
   };
  ```

- events类型的事件可以分为两大类

  - events可以表示多个不同事件，可以通过使用二进制掩码位与操作来完成，如：

  ```c
  #define    POLLIN    0x0001    /* any readable data available */
  #define    POLLPRI   0x0002    /* OOB/Urgent readable data */
  #define    POLLOUT   0x0004    /* file descriptor is writeable */
  ```

  - 可读事件

    - 可读事件和select的readset基本一致，是系统内核通知应用程序有数据可以读，用read函数执行操作不会被阻塞，有以下几种：

      ```c
      #define POLLIN     0x0001    /* any readable data available */
      #define POLLPRI    0x0002    /* OOB/Urgent readable data */
      #define POLLRDNORM 0x0040    /* non-OOB/URG data available */
      #define POLLRDBAND 0x0080    /* OOB/Urgent readable data */
      ```

  - 可写事件

    - 可写事件和select的writeset基本一致，是系统内核通知套接字缓冲区已准备好，用write函数执行写操作不会被阻塞，以下几种：

      ```c
      #define POLLOUT    0x0004    /* file descriptor is writeable */
      #define POLLWRNORM POLLOUT   /* no write type differentiation */
      #define POLLWRBAND 0x0100    /* OOB/Urgent data can be written */
      ```

  - 错误事件

    - 只能通过revents加以检测，如下几种：

      ```c
      #define POLLERR    0x0008    /* 一些错误发送 */
      #define POLLHUP    0x0010    /* 描述符挂起 */
      #define POLLNVAL   0x0020    /* 请求的事件无效 */
      ```

  - 在poll函数中，如果不想对每个pollfd结构进行事件检测，就把对应pollfd结构的fd成员设置为一个负值

### 22 非阻塞I/O

- 非阻塞I/O

  - 读操作。非阻塞模式下read函数调用会立即返回，一般是EWOULDBLOCK或EAGAIN出错信息
  - 写操作。write函数返回当前函数调用有多少数据被拷贝到了发送缓冲区

  ![Untitled5.png](/image/GeekTime-《网络编程实战-性能篇》笔记/Untitled5.png)

- 一般非阻塞I/O下，使用轮询的方式引起CPU占用率高，所以一般将非阻塞I/O和I/O多路复用技术select，poll等倒赔使用，是linux下高性能网络编程的首选

### 23 epoll

- epoll是一种I/O多路复用技术，通过监控注册的多个描述字，来进行I/O事件的分发处理

- epoll_create函数

  ```c
  int epoll_create(int size);
  int epoll_create1(int flags);
          返回值: 若成功返回一个大于 0 的值，表示 epoll 实例；若返回 -1 表示出错
  ```

  - size参数已不再被需要，因为内核可以动态分配需要的内核数据结构，只需要设置为大于0的整数即可

- epoll_ctl函数

  ```c
  int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
          返回值: 若成功返回 0；若返回 -1 表示出错
  ```

  - 用于往这个epoll实例增加或删除监控的事件

- epoll_wait函数

  ```c
  int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
    返回值: 成功返回的是一个大于 0 的数，表示事件的个数；返回 0 表示的是超时时间到；若出错返回 -1.
  ```

- 对比

  - edge-triggered，边缘触发，只有第一次满足条件才触发，之后不再传递同样的事件
  - level-triggered，条件触发，只要满足事件的条件，就会一直不断把这个事件传递给用户
  - 一般认为，边缘触发的效率比条件触发的效率要高，是epoll的杀手锏之一

### 24 C10K问题

- C10K：如何在一台物理机上同时服务10000个用户，C表示并发，10K=10000
- 如何解决C10K问题
  - 应用程序如何和操作系统配合，调度处理上万个套接字上的处理I/O事件
  - 应用程序如何分配进程，线程资源来服务上万个连接
- 解决方案
  - 阻塞I/O+进程
  - 阻塞I/O+线程
  - 非阻塞I/O+readines notification + 单线程
  - 非阻塞I/O+readiness notification + 多线程
  - 异步I/O+多线程
- 在linux下，解决高性能问题的利器是非阻塞I/O加上epoll机制，再利用多线程能力

### 25 阻塞I/O和进程模型

- 进程是程序执行的最小单位

- 创建新的进程使用函数fork

  ```c
  pid_t fork(void)
  返回：在子进程中为 0，在父进程中为子进程 ID，若出错则为 -1
  ```

- 为每一个连接创建一个独立的子进程来进行服务，实现简单，但无法满足高性能程序的需求

### 26 阻塞I/O和线程模型

- 线程有自己的上下文context，有一个唯一标识线程的ID，thread ID，还有栈，程序计数器，寄存器等，同个进程内的线程共享改进程的整个虚拟地址空间，包括代码，数据，堆，共享库等
- 线程的上下文：程序计数器告诉CPU代码执行到哪，寄存器存了当前计算的一些中间值，内存内有一些当前用到的变量等
- 线程的上下文切换：从一个计算场景，切换到另外一个计算场景，程序计数器，寄存器等这些值重新载入新场景的值
- POSIX线程是现代UNIX系统提供的处理线程的标准接口，有60多个定义函数

### 27 I/O多路复用和线程模型

- 事件驱动模型，也被叫做反应堆模型reactor，或者是Event loop模型，核心有两点：
  - 存在一个无限循环的事件分发线程，或者叫做reactor线程，Event loop线程
  - 所有的I/O操作都可以抽象成事件，每个事件必须有回调函数来处理
- 任何的网络程序所作的事情都可以总结成以下几种：
  - read，从套接字收取数据
  - decode，解析，解码数据
  - compute，堆数据进行计算和处理
  - encode，将结果按照约定的格式进行编码
  - send，通过套接字把结果发送出去
- 支持多并发的网络编程技术
  - fork
  - pthread
  - single reactor thread
  - single reactor thread+worker threads

### 28 I/O多路复用进阶

- 将连接建立事件和已建立连接上的I/O事件分离，形成主-从reactor模式
  - 主reactor只负责分发Acceptor连接建立，已连接套接字上的I/O事件交给sub-reactor负责分发，而sub-reactor的数量可以根据CPU的核数来灵活配置
- 主-从reactor+worker threads模式

### 29 epoll和多线程模型

- 与poll相比，linux提供的epoll是一种更为高效的事件分发机制
- epoll的性能分析
  - 事件集合，epoll维护了一个全局的事件集合，通过epoll句柄操作整个事件集合
  - 就绪列表，epoll返回的是活的的事件列表，减少了大量的扫描时间

### 30 异步I/O

- 阻塞I/O

![Untitled6.png](/image/GeekTime-《网络编程实战-性能篇》笔记/Untitled6.png)

- 非阻塞I/O

![Untitled7.png](/image/GeekTime-《网络编程实战-性能篇》笔记/Untitled7.png)

- 多路复用

![Untitled8.png](/image/GeekTime-《网络编程实战-性能篇》笔记/Untitled8.png)

- 异步I/O

![Untitled9.png](/image/GeekTime-《网络编程实战-性能篇》笔记/Untitled9.png)

- 在read调用时，内核将数据从内核空间拷贝到应用程序空间，这个过程是在read函数中同步进行的，如果内核实现的拷贝效率很差，read调用就会在这个同步过程中消耗比较长的时间
- 在linux下的aio异步操作支持很少，故一般都是使用epoll这样的I/O分发技术

### 31 epoll源码深度剖析

- epoll中使用的数据结构
  - eventpoll
    - 调用epoll_create之后内核侧创建的一个句柄，表示一个epoll实例
    - epoll_ctl和epoll_wait都是对这个eventpoll实例数据进行操作
  - epitem
    - 每次调用epoll_ctl增加一个fd时，会创建一个新的epitem实例，加入到eventpoll结构体中的红黑树中，红黑树对应字段是rbr
    - 查找fd是否有就绪事件都是通过红黑树上的epitem来操作的
  - eppoll_entry
    - 每当一个fd关联到epoll实例，就会有一个eppoll_entry产生
- 基于epoll的事件分发能力，是linux下高性能网络编程的不二之选
- epoll维护了一棵红黑树来跟踪所有待检测的文件描述符，红黑树的使用减少了内核和用户空间之间大量的数据拷贝和内存分配，大大提高了性能
