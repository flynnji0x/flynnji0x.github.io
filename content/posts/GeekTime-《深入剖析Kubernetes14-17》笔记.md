+++
title = "GeekTime -《深入剖析Kubernetes14-17》笔记"
date = '2021-03-22'
+++

### 14 深入解析Pod对象（一）：基本概念

- Pod和Container的关系：只是Pod属性里的一个普通的字段

- 凡是调度，网络，存储，以及安全相关的属性，基本上是Pod级别的

- Pod中几个重要的字段

  - NodeSelector，一个供用户将Pod与Node进行绑定的字段
  - NodeName，这个字段被赋值会被认为Pod已经经过了调度
  - HostAliases：定义了Pod的hosts文件里的内容，如：/etc/hosts
  - 凡是跟容器的Linux Namespace相关的属性，也一定是Pod级别的
  - 凡是Pod中的容器要共享宿主机的Namespace，也一定是Pod级别的定义

- Container中的重要字段

  - ImagePullPolicy字段，定义了镜像拉取的策略
  - Lifecycle字段，定义的是Container Lifecycle Hooks，例如postStart和preStop参数
  - Staus字段，包括以下几种可能的状态：
    - Pending
    - Running
    - Succeded
    - Failed
    - Unknown
  - Status还能细分出一组conditions，包括：
    - PodScheduled，Ready，Initialized，Unschedulable

  ### 15 深入解析Pod对象（二）：使用进阶

  - 特殊的Volumn，叫做Projected Volumn，可以翻译为，“投射数据卷”
  - kubernetes支持的Projected Volumn
    - Secret，典型场景，存放数据库的Credential信息
    - ConfigMap，用法与Secret几乎相同，保存的是不需要加密并且为应用所需的配置信息
    - Downward API，作用是让Pod里的容器能够直接获取到这个Pod API对象本身的信息，注意的是，获取的信息一定是Pod里的容器进程启动之前就能够确定下来的信息
    - ServiceAccountToken，只是一种特殊的Secret Volumn
  - PodPreset，即Pod预设置，可以用于对Pod批量化，自动化修改；其定义的内容只会在Pod API对象被创建之前追加在这个对象本身上，而不会影响任何Pod的控制器的定义

  ### 16 编排其实很简单：谈谈“控制器”模型

  - 控制器模式，control pattern
  - Pod这个看似复杂的API对象，实际上就是对容器的进一步抽象和封装而已
  - kubernetes的一个通用编排模式，控制循环，control loop，也叫调谐，reconcile，调谐的过程被称作调谐循环reconcile loop或同步循环sync loop
  - Deployment控制器，实际上都是由上半部分的控制器定义（包括期望状态），加上下半部分的被控制对象的模板组成的

  ### 17 经典PaaS的记忆：作业副本与水平扩展

  - Kubernetes中第一个重要的设计思想：控制器模式
  - Kubernetes中第一个控制器模式的完整实现：Deployment
  - 两个编排动作
    - 水平扩展/收缩
    - 滚动更新
      - 四个状态字段
        - DESIRED，用户期望的Pod副本个数
        - CURRENT，当前处于running状态的Pod的个数
        - UP-TO-DATE，当前处于最新版本的Pod的个数
        - AVAILABLE，当前已经可用的Pod的个数，既是running又是最新版本，处于Ready状态的Pod个数
      - 只有AVAILABLE字段描述的才是用户所期望的最终状态
  - Deployment只是在ReplicaSet的基础上，添加了UP-TO-DATE这个跟版本相关的状态字段
  - Deployment实际上是一个两层控制器，通过ReplicaSet的个数来描述应用的版本，再通过ReplicaSet的属性来保证Pod的副本数量
  - Deployment控制ReplicaSet（应用版本），ReplicaSet控制Pod（副本数）

