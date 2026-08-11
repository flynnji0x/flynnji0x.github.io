+++
title = "Dremio数据湖引擎（二）：在win10环境下的安装部署"
date = '2019-09-28'
tags = ["Dremio"]
categories = ["Dremio"]
id = "29c5c69a-f6a1-4b89-9b43-81c9b11d56b0"
+++
&emsp;&emsp;由于博主日常使用的OS为windows10，故本文将简单展示如何在win10基于Docker容器安装部署Dremio。另外，Dremio的官网也给出诸如AWS版本、Azure版本等的安装部署包，有兴趣的话可通过以下链接前往了解：[dremio deploy](https://www.dremio.com/deploy/)

### 环境准备

- win10环境下的Docker容器服务
  
### 拉取dremio-oss的docker镜像

  这里拉取下来的社区版本的Dremio镜像，商业版本的Dremio需要联系Dremio官方了。当然，作为个人开发使用，社区版本的Dremio已完全够用了。

  ```shell
  docker pull dremio/dremio-oss
  ```

### 单节点Dremio服务的部署

  ```shell
  docker run -p 9047:9047 -p 31010:31010 -p 45678:45678 dremio/dremio-oss
  ```

  利用以上命令可在Docker上启用一个单节点的Dremio服务，节点还包括以下服务：

  - Embedded Zookeeper
  - Master Coordinator
  - Executor

- 注册账号

  Dremio服务启动成功后，在浏览器访问本地9047端口，http://localhost:9047/。

  将出现以下账号注册界面，如下图：

  ![这是代替图片的文字，随便写](/image/Dremio数据湖引擎（二）：在win10环境下的安装部署/register.png)

  填写完信息并注册成功后会自动跳转到Dremio的管理面板首页，如下图：

  ![这是代替图片的文字，随便写](/image/Dremio数据湖引擎（二）：在win10环境下的安装部署/index.png)

- 创建一个内置简单数据源

  点击左下角的`Add Data Lake`，，可以看到Dremio已支持的基于表的存储源以及基于文件的存储源，这里我们先直接新增一个内置的数据源即可

  ![这是代替图片的文字，随便写](/image/Dremio数据湖引擎（二）：在win10环境下的安装部署/sample.png)

  点击文件路径一直到如下图的界面，可以看到这个数据源已经内置了csv、json、parquet等文件格式的多个数据文件或目录（对应基于表存储系统中的表）

  ![这是代替图片的文字，随便写](/image/Dremio数据湖引擎（二）：在win10环境下的安装部署/sample2.png)

- 选择其中一个文件，点击文件名旁边的小加号复制文件的访问路径，然后到查询编辑器执行类似以下的查询SQL，但是此时会抛出错误，Dremio提示找不到此文件

  ```sql
  select * from Samples."samples.dremio.com"."SF weather 2018-2019.csv"
  ```

  ![](/image/Dremio数据湖引擎（二）：在win10环境下的安装部署/sample3.jpg)

- 其实原因不难找，Dremio引擎其实也是通过类似其他数据库中的元数据机制来管理数据集的，只需要让我们要查询数据的文件注册到Dremio的元数据中就可以正常查询了。

  通过以下SQL可以查询到Dremio当前已知的元数据

  ```sql
  select * from INFORMATION_SCHEMA."TABLES"
  ```

  通过点击如下图按钮，将文件转换为指定的文件格式，就会将文件注册到Dremio的元数据中

  ![](/image/Dremio数据湖引擎（二）：在win10环境下的安装部署/sample4.jpg)

  再回到查询器页面执行查询SQL，就能正常查询出数据了

  ![](/image/Dremio数据湖引擎（二）：在win10环境下的安装部署/sample5.jpg)

&emsp;&emsp;到这里就算完成了Dremio的基本部署，并能简单使用Dremio提供的内置数据源查询到数据。



