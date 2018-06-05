---
title: Docker常用命令(二)
date: 2017-06-11 17:28:43
categories:
  Server
tags: 
  - Docker
  - Technology
---

# ps 命令
> ps命令用于输出容器列表
> docker ps <选项>

-a、--all=false 列出所有容器。不带 -a 只输出在运行的容器
--before="" 列出特定容器创建前的容器，包含停止的容器。
-f、--filter=[] 设置输出过滤。如 "exited=0"
-l、--latest=false 列出最后创建的容器，包含停止的容器
-q、--quiet=false 只输出容器的id

# push 命令
> push命令用于将镜像推送到Docker注册服务器
> docker push <注册名>/<镜像名>:<标签>

注册名中既可以设置Docker Hub 的用户名，也可以设置注册地址
若不设置标签，则推送所有标签的镜像
```sh
$ docker pull user/hello:latest
```
如下推送到个人仓库
```sh
$ docker pull 192.168.0.33:6666/hello:latest
$ docker pull yourset.com:6666/hello:latest
```
# restart 命令
> restart命令用户重启容器
> docker restart <选项><容器名称，id>

-t、time=10 设置从容器停止到重启的等待时间，单位为秒
```sh
$ docker restart hello
```
# rm 命令
> rm 命令用于删除容器
> docker rm <选项><容器名称，id>

-f、--force=false 强制停止容器后删除（使用SIGKILL信号）
-l、--link=false 在docker run 命令中使用--link 选项，只删除连接，不删除容器。
-v、--volumes=false 删除连接到容器的数据卷
若要一次删除所有容器，可在docker ps：命令中使用 -a -q 选项获取容器id只有传给docker rm 命令
```sh
$ docker rm `docker ps -aq`
$ docker rm $(docker ps -aq)
```
# rmi 命令
> rmi命令用于删除镜像。若不指定标签，则删除latest标签
> docker rim <注册名称>/<镜像名称，id>:<标签>

-f、--force=false 强制删除镜像
--no-prune=false 不删除不带标签的父级镜像
```sh
$ docker rmi hello
$ docker rmi user/hello:latest
$ docker rmi 192.168.0.33:6666/hello:latest  #远程仓库镜像
$ docker pull yourset.com:6666/hello:latest  #远程仓库镜像
```
删除所有镜像与删除容器类似
```sh
$ docker rmi `docker images -aq`
```

# run 命令
> run命令用于指定镜像创建容器
> docker run <选项><镜像名称，id><命令><参数>

docker run 命令 与 docker create 基本类似 唯一的不同是 run命令在创建容器后会启动容器，所以参数基本类似，只是多了关于启动后的设置，一下是多出来的命令：
-d、--detach Detach模式，一般为守护进程模式，容器以后台方式运行
--rm=false 若容器内的进程终止，则自动删除容器，此选项不能与-d选项一起使用
--sig-proxy=true 将所有信号传递给进程（非TTY模式时也一样），但不传递SIGCHLD、SIGKILL、SIGSTOP信号
# save 命令
> save命令用于将镜像保存为tar包文件
> docker save <选项><镜像名称>:<标签>

-o、--output="" 设置保存的文件名
若不设置-o选项，tar文件会输出到标准输出，所以必须设置重定向。如果仅指定镜像名称而未指定标签，则将所有标签保存到一个tar文件。
# search 命令
> search命令用与在docker hub 中搜索镜像
> docker search <选项><搜索词>

--automated=false 只显示由docker hub 的automated build 创建的镜像
--no-trunc=false 显示所有因因为内容过长而省略的部分
-s、--stars=0 显示滴啊有特定星级以上的镜像
# start 命令
> start命令用于启动容器
>docker start <选项><容器名称，id>

-a、--attrch=false 将标准输入、标准输出、标准错误连接到容器，传递所有信号
-i、--interactive=false 激活标准输入
# stop 命令
> stop命令用于终止容器
> docker stop <选项><容器名称，id>

-t、--time=10 设置终止容器前的等待时间，单位为秒
# tag 命令
> tag命令用于设置镜标签
> docker tag <选项><镜像名称>:<标签><注册地址，用户名>/<镜像名称>:<标签>

-f、--force=false 即使已拥有标签也强制设置
如将远程仓库设置标签
```sh
$ docker tag hello:latest user/hello:0.1  #设置docker hub上的
$ docker tag hello:latest youset:6666/hello:0.1  #私人仓库
```
# top 命令
> top命令用于显示容器中正在运行的进程信息
> docker top <容器名称，id><ps选项>

在<ps选项>中设置 Linux ps 命令的选项
```sh
$ docker top hello aux
```
# unpause 命令
> unpause命令用于重启 pause 命令暂停的容器
> docker unpause <容器名称，id>

# version 命令
> version命令用于输出docker的版本信息

```sh
docker version
```
# wait 命令
> wait 命令等待容器终止，然后输出 Exit Code
> docker wait <容器名称，id>

# 后记
单一的容器一般不能满足业务需要，需要一个编排的工具。Docker Compose和Docker Swarm 正是负责快速在集群中部署分布式应用。漫漫长路，学的还有好多，工作虽不是负责这方面的，我想做的只是将自己的想法运行在代码是而已。
