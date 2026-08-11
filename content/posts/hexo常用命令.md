+++
title = "hexo常用命令"
date = '2018-10-06'
tags = ["Hexo"]
categories = ["Hexo"]
id = "10f38775-d3ba-4e88-a8dc-9842ff5f114e"
+++
中文版的官方命令文档如下：https://hexo.io/zh-cn/docs/commands.html

简单做下总结笔记，方便日常查阅。



### init

新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站。

```
$ hexo init [folder]
```

三种模式:

- 安全模式

在安全模式下，不会载入插件和脚本。当您在安装新插件遭遇问题时，可以尝试以安全模式重新执行。

```shell
$ hexo --safe
```

- 调试模式

在终端中显示调试信息并记录到 debug.log。当您碰到问题时，可以尝试用调试模式重新执行一次，并 提交调试信息到 GitHub。

```shell
$ hexo --debug
```

- 简洁模式

隐藏终端信息。

```shell
$ hexo --silent
```



### new

新建文章

```shell
$ hexo new [layout] <title>
```

缩写

```shell
$ hexo n <title>
```



### generate

将文章生成静态文件

```shell
$ hexo generate
```

缩写

```shell
$ hexo g
```

常用命令

```shell
$ hexo g
$ hexo g --watch  #监视文件变动
```



### publish

发布草稿

```shell
$ hexo publish [layout] <filename>
```

缩写

```shell
$ hexo p
```



### server

启动服务器，默认本地访问地址：localhost:4000或127.0.0.1:4000

```shell
$ hexo server
```

缩写

```shell
$ hexo s
```

常用命令

```shell
$ hexo s  #Hexo回监视文件变动并自动更新，无须重启服务
$ hexo s -s  #静态模式
$ hexo s -p 8888  #指定访问端口
$ hexo s -i 127.0.0.2  #指定ip
```



### deploy

部署网站

```shell
$ hexo deploy
```

缩写

```shell
$ hexo d
```

常用命令

```shell
$ hexo g -d  #生成静态文件后部署
$ hexo d -g  #同上
$ hexo s -g
```



### clean

清除缓存文件和已生成的静态文件

```shell
$ hexo clean
```



### list

列出网站的资料信息

```shell
$ hexo list <type>
```



### version

显示当前hexo版本

```shell
$ hexo --config custom.yml
```



### 自定义配置文件的路径

自定义配置文件的路径，执行后将不再使用 _config.yml。

```shell
$ hexo --config custom.yml
```



### 显示草稿

显示 source/_drafts 文件夹中的草稿文章。

```shell
$ hexo --draft
```