---
title: Docker安装
date: 2017-06-11 17:28:43
categories:
  Server
tags: 
  - Docker
  - Technology
---

# 1.Docker安装
1. 在ubuntu上安装Docker，用下面命令：
```bash
$ sudo apt-get install apt-transport-https
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
$ sudo bash -c "echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"
$ sudo apt-get update
$ sudo apt-get install lxc-docker
```
2. 查看Docker版本
```bash
$ sudo docker version
```
3. 查看镜像
```bash
$ sudo docker images
```
4. 运行容器：
```bash
$ sudo docker hello-world
```