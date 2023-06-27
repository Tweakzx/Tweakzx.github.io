---
title: "【Docker】Docker用法"
description: 
date: 2022-11-01T16:44:33+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories: 
    - docker
tags: 
    - docker
---

# docker 的使用方法

##  docker的安装

docker的版本比较多， 大家可以自行搜索安装方式， 以下只是参考。

### Ubuntu

安装docker.io比较方便

#### 查询可安装版本

```shell
apt-cache madison docker
```

#### 安装docker

```shell
sudo apt install docker.io
指定安装版本
sudo apt install docker.io=18.09.1-0ubuntu1 
```

#### 启动docker

```shell
sudo systemctl start docker
```

#### 重启docker

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### 设置开机自启动

```shell
$ sudo systemctl enable docker
或
$ sudo systemctl enable docker.service --now
```

#### 验证docker

``` shell
systemctl status docker
docker --version
```

### centOS

需要安装docker-ce， docker社区版

#### 卸载旧版本

```shell
yum remove docker  docker-common docker-selinux docker-engine
```

#### 安装依赖

``` shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```

#### 设置源

阿里源

``` shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

清华源

```shell
yum-config-manager --add-repo https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo
```

#### 安装

查看可安装版本

```shell
yum list docker-ce --showduplicates | sort -r
```

选择版本安装

```shell
yum -y install docker-ce-18.03.1.ce
```

### 启动docker

```shell
systemctl start docker
systemctl enable docker
```

## docker 的基本命令

### 本地镜像管理

- [images](https://www.runoob.com/docker/docker-images-command.html)
- [rmi](https://www.runoob.com/docker/docker-rmi-command.html)
- [tag](https://www.runoob.com/docker/docker-tag-command.html)
- [build](https://www.runoob.com/docker/docker-build-command.html)
- [history](https://www.runoob.com/docker/docker-history-command.html)
- [save](https://www.runoob.com/docker/docker-save-command.html)
- [load](https://www.runoob.com/docker/docker-load-command.html)
- [import](https://www.runoob.com/docker/docker-import-command.html)

### 容器生命周期管理

- [run](https://www.runoob.com/docker/docker-run-command.html)
- [start/stop/restart](https://www.runoob.com/docker/docker-start-stop-restart-command.html)
- [kill](https://www.runoob.com/docker/docker-kill-command.html)
- [rm](https://www.runoob.com/docker/docker-rm-command.html)
- [pause/unpause](https://www.runoob.com/docker/docker-pause-unpause-command.html)
- [create](https://www.runoob.com/docker/docker-create-command.html)
- [exec](https://www.runoob.com/docker/docker-exec-command.html)

### 容器操作

- [ps](https://www.runoob.com/docker/docker-ps-command.html)
- [inspect](https://www.runoob.com/docker/docker-inspect-command.html)
- [top](https://www.runoob.com/docker/docker-top-command.html)
- [attach](https://www.runoob.com/docker/docker-attach-command.html)
- [events](https://www.runoob.com/docker/docker-events-command.html)
- [logs](https://www.runoob.com/docker/docker-logs-command.html)
- [wait](https://www.runoob.com/docker/docker-wait-command.html)
- [export](https://www.runoob.com/docker/docker-export-command.html)
- [port](https://www.runoob.com/docker/docker-port-command.html)
- [stats](https://www.runoob.com/docker/docker-stats-command.html)

### 容器rootfs命令

- [commit](https://www.runoob.com/docker/docker-commit-command.html)
- [cp](https://www.runoob.com/docker/docker-cp-command.html)
- [diff](https://www.runoob.com/docker/docker-diff-command.html)

### 镜像仓库

- [login](https://www.runoob.com/docker/docker-login-command.html)
- [pull](https://www.runoob.com/docker/docker-pull-command.html)
- [push](https://www.runoob.com/docker/docker-push-command.html)
- [search](https://www.runoob.com/docker/docker-search-command.html)

- https://www.runoob.com/docker/docker-import-command.html)

### info|version

- [info](https://www.runoob.com/docker/docker-info-command.html)
- [version](https://www.runoob.com/docker/docker-version-command.html)

## docker 部署实例

### Hello World

```
runoob@runoob:~$ docker run ubuntu:15.10 /bin/echo "Hello world"
Hello world
```

### Redis

```
docker pull redis:latest

docker run -itd --name redis-test -p 6379:6379 redis

docker exec -it redis-test /bin/bash
```

## 相关问题

### 1. docker.io 和docker-ce 的区别

> 维护者：
>
> - docker-ce 是 docker 官方维护的
> - docker.io 是 Debian 团队维护的
>
> 依赖管理：
>
> - docker.io 采用 apt 的方式管理依赖
> - docker-ce 用 go 的方式管理依赖，会自己管理所有的依赖。



### 2.

