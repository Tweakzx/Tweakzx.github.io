---
title: "【Kubernetes】在k8s集群中挂载基于NFS的StorageClass"
author: "Tweakzx"
description: 
date: 2023-01-10T11:24:28+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories: kubernetes
tags: 
---

# 如何在k8s集群中挂载基于NFS的StorageClass

## 配置NFS服务器

### 查看系统版本

```bash
# cat /etc/redhat-release 
CentOS Linux release 7.7.1908 (Core)
```

CentOS 7.4 以后，支持 NFS v4.2 不需要 rpcbind 了，但是如果客户端只支持 NFC v3 则需要 rpcbind 这个服务

### 安装服务端

```bash
sudo yum install nfs-utils
```

> 只安装 nfs-utils 即可，rpcbind 属于它的依赖

### 配置服务端

设置 NFS 服务开机启动

```bash
sudo systemctl enable rpcbind
sudo systemctl enable nfs
```

启动 NFS 服务

```bash
sudo systemctl start rpcbind
sudo systemctl start nfs
```

防火墙需要打开 rpc-bind 和 nfs 的服务,  但是我的k8s集群已经关闭了防火墙， 所以这一步我掠过了。

```bash
sudo firewall-cmd --zone=public --permanent --add-service={rpc-bind,mountd,nfs}
success
sudo firewall-cmd --reload
success
```

### 配置共享目录

服务启动之后，我们在服务端配置一个共享目录

```bash
sudo mkdir /data
sudo chmod 755 /data
```

根据这个目录，相应配置导出目录

```bash
sudo vi /etc/exports
```

添加如下配置

```bash
/data/     192.168.0.0/24(rw,sync,no_root_squash,no_all_squash)
```

1. `/data`: 共享目录位置。
2. `192.168.0.0/24`: 客户端 IP 范围，`*` 代表所有，即没有限制。
3. `rw`: 权限设置，可读可写。
4. `sync`: 同步共享目录。
5. `no_root_squash`: 可以使用 root 授权。
6. `no_all_squash`: 可以使用普通用户授权。

`:wq` 保存设置之后，重启 NFS 服务。

```bash
sudo systemctl restart nfs
```

### 测试验证

在本机验证， 查看共享目录

```bash
showmount -e localhost
Export list for localhost:
/data 192.168.0.0/24
```

在其他节点上验证,  安装启动NFS服务， 查看共享目录

```
showmount -e 192.168.0.110
Export list for localhost:
/data 192.168.0.0/24
```

## 在K8s中配置并挂载StorageClass

### 配置Deployment

创建一个文件nfs-client.yaml， 内容如下，将环境变量 NFS_SERVER 和 NFS_PATH 替换，当然也包括下面的 nfs 配置。

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 10.151.30.57   # 需要替换成自己的ip地址
            - name: NFS_PATH
              value: /data/k8s      # 需要替换成自己的路径
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.151.30.57    # 需要替换成自己的ip地址
            path: /data/k8s         # 需要替换成自己的路径
```

### 配置SA

我们可以看到上面使用了一个名为 nfs-client-provisioner 的`serviceAccount`，所以我们也需要创建一个 sa，然后绑定上对应的权限。

创建文件nfs-client-sa.yaml， 内容如下

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```

我们新建了一个名为 nfs-client-provisioner 的`ServiceAccount`，然后绑定了一个名为 nfs-client-provisioner-runner 的`ClusterRole`，而该`ClusterRole`声明了一些权限，其中就包括对`persistentvolumes`的增、删、改、查等权限，所以我们可以利用该`ServiceAccount`来自动创建 PV。

### 配置StogrageClass对象

创建文件nfs-client-class.yaml， 内容如下

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: course-nfs-storage
provisioner: fuseim.pri/ifs
```

我们声明了一个名为 course-nfs-storage 的`StorageClass`对象，注意下面的`provisioner`对应的值一定要和上面的`Deployment`下面的 PROVISIONER_NAME 这个环境变量的值一样。

### 创建资源

```bash
kubectl create -f nfs-client.yaml
kubectl create -f nfs-client-sa.yaml
kubectl create -f nfs-client-class.yaml
```

创建完成之后我们就可以查看资源状态

```bash
# kubectl get pods
NAME                                       READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-7ddfdb5759-jthzf    1/1     Running   0          153m


# kubectl get storageclass
NAME                 PROVISIONER      AGE
course-nfs-storage   fuseim.pri/ifs   154m
```

## 参考链接

1. [CentOS 7 下 yum 安装和配置 NFS - Zhanming's blog (qizhanming.com)](https://qizhanming.com/blog/2018/08/08/how-to-install-nfs-on-centos-7)
2. [StorageClass · 从 Docker 到 Kubernetes 进阶手册 (qikqiak.com)](https://www.qikqiak.com/k8s-book/docs/35.StorageClass.html)