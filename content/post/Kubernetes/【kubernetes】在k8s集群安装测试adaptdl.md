---
title: "【Kubernetes】在k8s集群安装测试adaptdl"
author: "Tweakzx"
description: 
date: 2023-01-09T21:15:48+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories: kubernetes
tags: 
---

# 在k8s集群上部署测试Adaptdl

## 环境配置

- Centos Linux realease 7.9.2009 三台（GPU节点），CentOS Linux release 7.7.1908 一台（Master节点）
- 内核：3.10.0-1062.18.1.el7.x86_64
- Docker-ce：18.09.7
- Kubernetes：1.16.15
- Nvidia Driver： 460.32.03两台，510.47.03一台
- Helm: 3.1.1
- 安装配置Nvidia-docker2， 集群可以使用**nvidia.com/gpu**
- 配置NFS，在集群中**配置StorageClass**

## 配置docker私有仓库

Adaptdl提供的安装方法[Installing the AdaptDL Scheduler — AdaptDL documentation](https://adaptdl.readthedocs.io/en/latest/installation/install-adaptdl.html#install-the-adaptdl-helm-chart)里面提到， 使用`--set docker-registry.enabled=true` 可以安装一个Docker Registry， 但是这是不安全的， 建议使用别的docker registry来替代， 在安装的时候将`--set docker-registry.enabled=true` 省略掉。

我自己尝试使用adaptdl自带的registry结果在submit job的时候无法拉去到镜像， 试了很多别的方法， 发现没有效果， 只能自己搭建私有仓库来尝试解决， 万幸确实解决了。 

### 在[91]节点上配置registry

拉取镜像

```bash
docker pull registr
```

搭建私有仓库

```
docker run -d --name registry -p 5000:5000 -v /opt/registry:/var/lib/registry   registry:latest
```

查看是否建成

```bash
# ls  /opt/registry
docker
# docker ps |grep registry
4375a31c83e3 registry:latest  "/entrypoint.sh /etc…"   44 minutes ago Up 44 minutes 0.0.0.0:5000->5000/tcp registry
# netstat  -antup | grep 5000
tcp6  0   0 :::5000  :::*   LISTEN  10552/docker-proxy 
```

查看镜像列表， 你可以发现此刻仓库镜像为空

```bash
# curl http://10.10.108.91:5000/v2/_catalog 
{"repositories":[]} 
```

### 在其他节点[10, 14, 15]上使用私有仓库

```bash
vim  /etc/docker/daemon.json   
```

修改daemon.json， 写入以下内容

```
"insecure-registries": [ "10.10.108.91:5000" ],
```

之后重启docker

```bash
systemctl daemon-reload
systemctl restart docker 
```

### 从其他节点上上传镜像到仓库

```
docker pull busybox
docker tag busybox:latest 10.10.108.91:5000/busybox:latest
docker push 10.10.108.91:5000/busybox:latest
```

先下载busybox， 改名之后再上传到私有仓库， 之后查看镜像列表， 发现私有仓库里多了busybox

```bash
# curl http://10.10.108.91:5000/v2/_catalog 
{"repositories":["adaptdl-submit","busybox"]}
```

私有仓库配置成功

## 安装Helm

建议下载helm3， 具体版本支持策略可以看[Helm | Helm版本支持策略](https://helm.sh/zh/docs/topics/version_skew/)， 安装方法参考[Helm | 安装Helm](https://helm.sh/zh/docs/intro/install/)

```bash
wget https://get.helm.sh/helm-v3.1.1-linux-amd64.tar.gz
tar -zxvf helm-v3.1.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm

# 查看是否安装成功
helm version
```

## Helm安装Adaptdl

### Adaptdl官方提供的安装策略

```
helm install adaptdl adaptdl-sched --repo https://github.com/petuum/adaptdl/raw/helm-repo --namespace adaptdl --create-namespace --set docker-registry.enabled=true
```

我执行这个命令， 出现了很多问题， 例如

- 获取https://github.com/petuum/adaptdl/raw/helm-repo 超时
- 无法识别--create-namespace 参数
- validator-webhook.yaml报错Invalid value: []string{"v1"}: must include at least one of v1beta1
- 安装成功后， 仓库拉取镜像超时

### 我的安装过程

本地下载adpatdl-sched

```
wget https://raw.githubusercontent.com/petuum/adaptdl/helm-repo/adaptdl-sched-0.2.11.tgz
```

解包修改一点内容

```bash
tar -zxvf adaptdl-sched-0.2.11.tgz
vim adaptdl-sched/templates/validator-webhook.yaml
```

将admissionReviewVersions由v1改为v1beta1

```bash
webhooks:
  - name: {{ .Release.Name }}-validator.{{ .Release.Namespace }}.svc.cluster.local
    clientConfig:
      caBundle: {{ $cert.Cert | b64enc }}
      service:
        name: {{ .Release.Name }}-validator
        namespace: {{ .Release.Namespace }}
        path: "/validate"
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["adaptdl.petuum.com"]
        apiVersions: ["v1"]
        resources: ["adaptdljobs"]
    admissionReviewVersions:
      - v1beta1
    sideEffects: None
```

重新打包

```bash
helm package adaptdl-sched
```

创建namespace

```
kubectl create namespace adaptdl
```

安装adaptdl

```bash
helm install adaptdl adaptdl-sched-0.2.11.tgz --namespace adaptdl
```

使用私有仓库

```bash
export ADAPTDL_SUBMIT_REPO=10.10.108.91:5000/adaptdl-submit
export ADAPTDL_SUBMIT_REPO_CREDS=mysecret
```

查看效果

```bash
# kubectl get pods -A 
NAMESPACE     NAME                                       READY   STATUS      RESTARTS   AGE
adaptdl       adaptdl-adaptdl-sched-67c844b5b6-zpqk8     3/3     Running     0          56m
adaptdl       adaptdl-validator-5f48976d4d-kr9cf         1/1     Running     0          56m
```

## 测试

### Hello-world

这一部分参考[Submitting a Simple Job — AdaptDL documentation](https://adaptdl.readthedocs.io/en/latest/commandline/simple-job.html#writing-a-simple-program)

```
mkdir hello_world
cd hello_world
```

#### 创建文件

一共要创建三个文件

- hello_world.py
- Dockerfile
- adaptdljob.yaml

hello_world/hello_world.py

```python
import adaptdl.env
import os
import time

print("Hello, world!")

with open(os.path.join(adaptdl.env.share_path(), "foo.txt"), "w") as f:
    f.write("Hello, world!")

time.sleep(100)
```

hello_world/Dockerfile

```dockerfile
FROM python:3.7-slim
RUN python3 -m pip install adaptdl

COPY hello_world.py /root/hello_world.py

ENV PYTHONUNBUFFERED=true
```

hello_world/adaptdljob.yaml

```yaml
apiVersion: adaptdl.petuum.com/v1
kind: AdaptDLJob
metadata:
  generateName: hello-world-
spec:
  template:
    spec:
      containers:
      - name: main
        command:
        - python3
        - /root/hello_world.py
```

#### 提交任务

```bash
adaptdl submit hello_world/ --checkpoint-storage-class=course-nfs-storage
```

查看效果

```bash
# kubectl get pods -A 
default hello-world-vbfp9-3f6dd75e-cf18-447a-a49a-3e1f39234b7a-0-0 1/1 Running 0  6s
# adaptdl ls
Name                Status     Start(UTC)    Runtime  Rplc  Rtrt    
hello-world-vbfp9   Running    Jan-10 07:29  0 min    1     0
# kubectl logs hello-world-6r2h6-57779eff-089b-47d9-ab44-fb23d7433518-0-0 
Hello, world!
```

### BERT

这部分的代码在adaptdl/examples/BERT中

docker build的时候， 只能读取与Dockerfile同目录以下的内容， 目录外的内容无法使用， 所以你可能要对Dockerfile做一下修改， 然后将adaptdl的源码文件复制一份到BERT/目录下， 然后

```
adaptdl submit examples/BERT/ --checkpoint-storage-class=course-nfs-storage
```

查看运行状态，以及输出

```bash
# kubectl logs mlm-task-bnqxw-390a44d3-ca34-4cb9-9ea7-091b1d4ec4db-0-0
INFO:root:Downloading file wikitext-2-v1.zip to .data/wikitext-2-v1.zip.
wikitext-2-v1.zip: 100%|██████████| 4.48M/4.48M [00:05<00:00, 811kB/s] 
INFO:root:File .data/wikitext-2-v1.zip downloaded.
INFO:root:Opening zip file .data/wikitext-2-v1.zip.
INFO:root:Creating train data
INFO:root:Creating test data
INFO:root:Creating valid data
1lines [00:00,  3.76lines/s]
INFO:root:File .data/wikitext-2-v1.zip already exists.
INFO:root:Opening zip file .data/wikitext-2-v1.zip.
INFO:root:.data/wikitext-2/ already extracted.
INFO:root:.data/wikitext-2/wiki.test.tokens already extracted.
INFO:root:.data/wikitext-2/wiki.valid.tokens already extracted.
INFO:root:.data/wikitext-2/wiki.train.tokens already extracted.
INFO:root:Creating train data
INFO:root:Creating test data
INFO:root:Creating valid data
```

## 问题

> Error: No Docker registry could be found!

```bash
export ADAPTDL_SUBMIT_REPO=10.10.108.91:5000/adaptdl-submit
export ADAPTDL_SUBMIT_REPO_CREDS=mysecret
```

解决：记得在Master节点上设置仓库的环境变量



> Unsupported storageclass from available storageclasses []

https://github.com/petuum/adaptdl/issues/112#issue-1115791420

解决：配置StorageClass， 在提交job的时候指定`--checkpoint-storage-class=...`



## 参考链接

1. [Installing the AdaptDL Scheduler — AdaptDL documentation](https://adaptdl.readthedocs.io/en/latest/installation/install-adaptdl.html#install-the-adaptdl-helm-chart)
2. [Submitting a Simple Job — AdaptDL documentation](https://adaptdl.readthedocs.io/en/latest/commandline/simple-job.html#writing-a-simple-program)
3. [docker私有镜像仓库的配置和使用 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/400598575)
4. [AdaptDL](https://adaptdl.readthedocs.io/_/downloads/en/latest/pdf/)
5. [hello_world can not run · Issue #112 · petuum/adaptdl (github.com)](https://github.com/petuum/adaptdl/issues/112)
6. [Helm | Helm版本支持策略](https://helm.sh/zh/docs/topics/version_skew/)
7. [Helm | 安装Helm](https://helm.sh/zh/docs/intro/install/)
8. [Submitting a Simple Job — AdaptDL documentation](https://adaptdl.readthedocs.io/en/latest/commandline/simple-job.html#writing-a-simple-program)

