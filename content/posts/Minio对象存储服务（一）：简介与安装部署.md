+++
title = "Minio对象存储服务（一）：简介与安装部署"
date = '2020-09-05'
tags = ["Minio"]
categories = ["Minio"]
id = "54e3ebb6-fbb9-41b0-97d9-96c5ba466ff"
+++
### 简介

&emsp;&emsp;MinIO是一个基于Apache License v2.0开源协议的对象存储服务。它兼容亚马逊S3云存储服务接口，非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从几kb到最大5T不等。

&emsp;&emsp;MinIO是一个非常轻量的服务,可以很简单的和其他应用的结合，类似 NodeJS, Redis 或者 MySQL。

### 环境准备

- Windows10操作系统
- Docker

### 单节点模式部署

&emsp;&emsp;Minio需要配置存储空间来让数据持久化，但我们目前只需要将Minio服务部署运行起来即可。

- 先从docker上拉取Minio的服务镜像

  ```shell
  docker pull minio/minio
  ```

- 运行以下示例命令，将运行一个供测试用的Minio服务，其中我们指定了一个目录名称，来告诉Minio将文件临时存储在此目录下，此目录会在容器启动后自动创建。需要注意的是，容器实例一旦停止运行，/data目录上的数据将随之消失。

  ```shell
  docker run -p 9000:9000 minio/minio server /data
  ```

- 在shell上启动服务成功后，将看到以下输出，其中的`'minioadmin:minioadmin'`是默认登录用的AK和AS

  ![](/image/Minio对象存储服务（一）：简介与安装部署/img-01.png)

### 基本操作

- 使用默认的密钥登录成功后就到了Minio管理面板首页

  ![](/image/Minio对象存储服务（一）：简介与安装部署/img-02.png)

- 需要先创建一个Bucket，右下角的`+`点击后可以创建一个新的Bucket，如下图示例创建了一个名为tmp-bucket的Bucket

  ![](/image/Minio对象存储服务（一）：简介与安装部署/img-03.png)

- 在显示当前文件路径的右侧有个`+`按钮，跳过这个按钮可以创建指定名称的目录，在上个步骤中右下角的`+`中提供了`upload file`的功能，可以让我们快速地上传一个文件，如以下示例，在路径为`tmp-bucket/tmp/`的目录下，上传了一个文件名为`新建文本文档.txt`的文件

  ![](/image/Minio对象存储服务（一）：简介与安装部署/img-04.png)

- 在每个文件的最右侧的`...`中还提供了文件分享、下载以及删除的功能

  ![](/image/Minio对象存储服务（一）：简介与安装部署/img-05.png)

&emsp;&emsp;以上就是Minio管理面板的简单操作示例，Minio提供的管理面板只适用于一些简单的应用场景或者Debug调试使用，正常的业务生产环境中一般是需要通过Minio官方或第三方的SDK来对文件进行操作。



