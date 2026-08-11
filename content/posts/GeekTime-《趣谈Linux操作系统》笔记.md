+++
title = "GeekTime -《趣谈Linux操作系统》笔记"
date = '2020-12-10'
tags = ["笔记", "极客时间", "linux", "操作系统"]
categories = ["极客时间"]
+++

### 02 学习路径

- Linux的学习过程，总结下来，对应有6个坡，即6个阶段

  - 抛弃旧的思维习惯，熟练使用Linux命令行

    全面学习Linux命令，推荐《鸟哥的Linux私房菜》，更深入推荐《Linux系统管理技术手册》

  - 通过系统调用或者glibc，学会自己进行程序设计

    《UNIX环境高级编程》

  - 了解Linux内核机制，反复研习重点突破

    《深入理解LINUX内核》

  - 阅读Linux内核代码，聚焦核心逻辑和场景

    《LINUX内核源代码情景分析》

  - 实验定制化Linux组件，已经没人能阻挡你成为内核开发工程师了

  - 面向真实场景的开发，实践没有终点

### 03 Linux内核体系结构

- 操作系统内核体系结构组成
  - 系统调用子系统
    - 操作系统的功能调用的统一入口
  - 进程管理子系统
    - 堆执行的程序进行生命周期和资源管理
  - 内存管理子系统
    - 对操作系统的内存进行管理，分配，挥手，隔离
  - 文件子系统
    - 对文件进行管理
  - 设备子系统
    - 对输入输出设备进行管理
  - 网络子系统
    - 网络协议栈和收发网络包

### 04 快速上手几个Linux命令

- passwd

- useradd

- chown，chgrp

- centos：rpm，ubuntu：deb

- centos：yum，ubuntu：apt-get

- Linux执行程序的方式：

  - 通过shell在交互命令行里面运行

  - 后台运行

    - nohup，no hang up，最后加一个&，表示后台运行

    ```bash
    nohup command > out.file 2>&1 &
    ```

  - 以服务的方式运行

    - systemctl

### 05 学会几个系统调用

- 进程管理
  - 创建进程的系统调用叫fork
  - 父进程，Parent Process，子进程，Child Process
- 内存管理
  - 对于进程内的内存空间，放程序代码的地方，称为代码段， Code Segment
  - 对于经常内的内存空间，放进程运行中产生数据的这部分，称为数据段，Data Segment
  - 局部变量在函数切换时会被释放，而动态分配需要较长时间保存，指明才销毁的，称为堆，Heap
  - 堆里面分配内存的系统调用：
    - 调整内存数据段，brk
    - 内存映射，mmap
- 文件管理
  - Linux中，一切皆文件
  - 每个文件，Linux都会分配一个文件描述符，File Descriptor，是一个非负整数
- 信号处理
  - 进程通过kill函数，将一个用户信号发送给另一个进程
  - sigaction系统调用，可以注册一个信号处理函数
- 进程间通信
  - 消息队列，Message Queue
  - 共享内存
- 网络通信
  - TCP/IP网络协议栈
- Glibc，是Linux下使用的开源的标准C库，提供了丰富的API，如字符串处理，数据运算等用户态服务，还有封装了系统提供的系统服务，即系统调用的封装

### 06 x86架构

- 计算机的核心，CPU，central processing unit，中央处理器
- CPU和其他设备的连接，需要总线，Bus
- CPU本身没办法保存太多的中间结果，需要依赖内存，Memory
- CPU由三部分组成：
  - 运算单元，只管运算
  - 数据单元，包括CPU内部缓存和寄存器组，空间小，速度飞快
  - 控制单元，可以获得下一条指令，然后执行这条指令，指令会指导运算单元去除数据单元中的某些数据，计算出结果，然后放在数据单元的某个地方
- 控制单元
  - 指令指针寄存器，存放下一条指令在内存中的地址，指令分为两部分，一部分是做什么操作，一部分是操作哪些数据
- 总线的分类
  - 地址总线，Address Bus
  - 数据总线，Data Bus
- 23位的系统架构下，旧模式称为实模式，Real Pattern，新模式称为保护模式，Protected Pattern

### 07 从BIOS到bootloader

- ROM，Read Only Memory，只读存储器
- 内存RAM，Random Access Memory，随机存取存储器
- BIOS，Basic Input and Output System，基本输入输出系统
- Grub2，Grand Unified Bootloader Version 2，系统启动工具
- - 安装boot.img，到第一个扇区，称为MBR，Master Boot Record，主引导记录/扇区
  - 加载grub2的另一个镜像，core.img
  - 控制权交到core.img的第一个扇区，diskboot.img
  - doskboot.img加载，lzma_decompress.img，kernel.img，modules&others
  - lzma_decompress.img调用real_to_prot，切换到保护模式
    - 启动分段，建立段描述符表，将寄存器中的段寄存器变成段选择子，指向段描述符，旧能实现不同进程的切换了
    - 启动分页，将内存分成相等大小的块
- 操作系统启动过程梳理
  - BIOS
  - 引导扇区，boot.img
  - diskboot.img
  - lzma_decompress.img，实模式到保护模式
  - 建立分段分页，打开地址线
  - kernel.img，选择一个操作系统
  - 启动内核

### 08 内核初始化

- 内核的启动从入口函数start_kernel()开始，相当于内核的main函数

  - se_task_stack_end_magic(&init_task)，创建0号进程，这是唯一一个没有通过fork或者kernel_thread产生的进程，是进程列表的第一个
  - trap_init()，设置各种中断门，Interrupt Gate，用于处理中断事件
  - mm_init()，初始化内存管理模块
  - sched_init()，初始化调度模块
  - vfs_caches_init()，初始化基于内存的文件系统rootfs
  - rest_init()，其他的初始化工作
    - 第一大作用是初始化1号进程，kernel_thread()，是所有用户态进程的祖先
    - 创建2号进程，kernel_thread()，是所有内核态进程运行的祖先

- x86提供了分层的权限机制，把区域分成了四个Ring

  ![Untitled.png](/image/GeekTime-《趣谈Linux操作系统》笔记/Untitled.png)

  - 能够访问关键资源的代码放在Ring0，称为内核态，Kernel Mode
  - 将普通的程序代码放在Ring3，称为用户态，User Mode
  - 当程序发送网络包时，用户态 - 系统调用 - 保存寄存器 - 内核态执行系统调用 - 恢复寄存器 - 返回用户态，程序接着运行
