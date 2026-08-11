+++
title = "GeekTime -《深入剖析Kubernetes09-13》笔记"
date = '2021-03-18'
+++
# 深入剖析kubernetes

### 09 从容器到容器云：谈谈kubernetes的本质

- 最具代表性的容器编排工具
  - docker compose + Swarm
  - kubernetes
- Kubernetes是依托google的Borg项目的理论优势，由master和node两种节点组成，分别对应着控制节点和计算节点
- 控制节点，即Master节点，由三个独立组件组合而成，负责API服务的kube-apiserver，负责调度的kube-scheduler，负责容器编排的kube-controller-manager，整个集群的持久化数据，则由kube-apiserver处理后保存在Ectd中
- 计算节点上最核心的部分，叫做kubelet的组件。主要负责同容器运行时打交道
- kubelet还通过gRPC协议同一个叫做Device Plugin的插件进行交互，用来隔离GPU等宿主机物理设备的主要组件
- kubelet的另一个重要功能，调用网络插件和存储插件为容器配置网络和持久化存储
- kubernetes项目最主要的设计思想是，从更宏观的角度，以统一的方式来定义任务之间的各种关系，并且为将来支持更多种类的关系留有余地
- 几个在kubernetes中的对象
  - Pod
  - Job，用来描述一次性运行的pod
  - DaemonSet，用来描述每个宿主机上必须且只能运行一个副本的守护进程服务
  - CronJob，用于描述定时任务
  - Deployment，pod的多实例管理器
  - Service，主要作用，就是作为Pod的代理入口，从而代替Pod对外暴露一个固定的网络地址
  - Secret，保存在Etcd中的键值对数据，用于pod间的访问鉴权
- 在kubernetes项目中，推崇的使用方法是：
  - 通过“编排对象”，Pod，Job，CronJob等，来描述你试图管理的应用
  - 再为它定义一些“服务对象”，比如Service，Secret等，这些对象，会负责具体的平台级功能

### 10 kubernetes一键部署利器：kubeadm

- kubeadm的工作原理
  - kubeadm的妥协方案，把kubelet直接运行在宿主机上，然后使用容器部署其他的kubernetes组件
- kubeadm init的工作流程
  - 首先要做的是，一系列的检查工作，以确定这台机器可以用来部署kubernetes，称为“Preflight Checks”
  - 然后，生成kubernetes对外提供服务所需的各种证书和对应的目录
  - 证书生成后，kubeadm接下来会为其他组件生成访问kube-apiserver所需的配置文件
  - 接下来，kubeadm会为Master组件生成Pod配置文件
  - 然后，kubeadm就会为集群生成一个bootstrap token
  - token生成之后，kubeadm会将ca.crt等Master节点的重要信息，通过ConfigMap的方式保存在Etcd中，供后续部署Node节点使用
  - 最后一步，安装默认插件

### 12 第一个容器化应用

- 一个YAML文件，在kubernetes中，就是一个API Objetc，API对象
- 每一个API对象，都有一个叫做Metadata的字段，就是API对象的“标识”，即元数据，是我们从kubernetes中找到这个对象的主要依据
- 一个API对象，大多可以分为Metadata和Spec两个部分，前者是这个对象的元数据，后者存放的是属于这个对象独有的定义，用来描述它所要表达的功能
- 一些常用的命令
  - kubectl apply
  - kubectl exec
  - kubectl delete
  - kubectl get
  - kubectl create -f 我的配置文件

### 13 为什么我们需要Pod？

- Pod，是kubernetes项目的原子调度单位，即最小的编排单位
- Docker容器的本质：Namespace做隔离，Cgroups做限制，rootfs做文件系统
- Pod，最重要的一个事实是，它只是一个逻辑概念
- Pod里的所有容器，共享的是同一个Network Namespace，并且可以声明共享同一个Volume
- Pod的实现需要使用一个中间容器，这个容器叫做Infra容器，永远是第一个被创建的容器
