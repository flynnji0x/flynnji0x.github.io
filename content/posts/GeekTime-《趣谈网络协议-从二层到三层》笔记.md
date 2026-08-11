+++
title = "GeekTime -《趣谈网络协议-从二层到三层》笔记"
date = '2019-05-03'
tags = ["笔记", "极客时间", "网络协议"]
categories = ["极客时间"]
+++
# 趣谈网络协议-从二层到三层

### 05 从物理层到MAC层

- MAC，全称是Medium Access Control，即媒体访问控制
- 通过ARP协议，已知IP地址获取MAC地址

### 06 交换机与VLAN

- STP协议，Spanning Tree Protocol，最小生成树
  - Root Bridge，根交换机
  - Designated Bridges，指定交换机
  - Bridge Protocol Data Units（BPDU），网桥协议数据单元
  - Priority Vector，优先级向量
- 利用STP协议解决环路问题，将又环路的图变成没有环的树
- VLAN，虚拟局域网，用来解决广播和安全问题

### 07 ICMP与ping

- ICMP全称Internet Control Mesaage Protocol，互联网控制报文协议
- ping就是查询报文，是一种主动请求，并且获得主动应答的ICMP协议
- Traceroute，使用的是差错报文

### 08 世界那么大，我想出网关

- 网关往往是一个路由器，是一个三层转发的设备，理由如何寻找下一跳的规则
- NAT，Network Address Translation
- 不改变IP地址的网关。称为转发网关；改变IP地址的网关，称为NAT网关

### 09 路由协议

- 动态路由算法
- 距离矢量路由算法，distance vector routing
  - BGP，Border Gateway Protocol，外网路由协议
- 链路状态路由算法，link state routing
  - OSPF，Open Shortest Path First，开放式最短路径优先，称为IGP，Interior Gateway Protocol，内部网关协议