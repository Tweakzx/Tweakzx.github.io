---
title: "【Kubernetes】ubuntu安装k8s集群"
author: "Tweakzx"
date: 2022-06-13T14:57:42+08:00
description: 
categories: kubernetes
tags: 
- 容器
- k8s
image: https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/20220823230439.png
draft: false
---

# 在ubuntu上k8s集群部署实践

> centos 上安装建议看 [后端 - CentOS 搭建 K8S，一次性成功，收藏了！_个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000037682150)

## 一、机器配置

### 配置主机名

```shell
sudo hostnamectl set-hostname "k8s-master"    // Run this command on masternode
cat /etc/hostname
sudo hostnamectl set-hostname "k8s-node1"     // Run this command on node-0
sudo hostnamectl set-hostname "k8s-node2"     // Run this command on node-1
```

### 配置/etc/hosts

```shell
sudo vi /etc/hosts 

10.1.13.106    k8s-master

#10.1.13.107    k8s-node1

#10.1.13.108    k8s-node1
```

### 配置免密登录（可以省略）

想让机器 A 访问机器 B，就把机器 A 的公钥放到机器 B 的`~/.ssh/authorized_keys` 文件里就行了。

首先我们在`master`上生成一个密钥，输入下述命令后**一路回车**即可：

```shell
ssh-keygen
```

然后登录`worker`，并依次输入下述两条命令将其复制并写入到`worker`的`authorized_keys`中，注意我下面的`scp`命令中使用了`worker`别名，要提前进行配置：

```bash
# 复制到 worker
scp root@k8s-master:~/.ssh/id_rsa.pub /home
# 写入到 authorized_keys 中
cat /home/id_rsa.pub >> ~/.ssh/authorized_keys
```

然后在master上使用`ssh worker`登录就可以发现直接连接上而不需要密码了。

### 关闭防火墙

```shell
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

### 禁用swap

这个swap其实可以类比成 windows 上的虚拟内存，它可以让服务器在内存吃满的情况下可以保持低效运行，而不是直接卡死。但是 k8s 的较新版本都要求关闭swap。所以咱们直接动手，修改/etc/fstab文件：

```shell
sudo vi /etc/fstab
```

你应该可以看到如下内容，把第二条用#注释掉就好了，注意第一条别注释了，不然重启之后系统有可能会报file system read-only错误。

```shell
UUID=e2048966-750b-4795-a9a2-7b477d6681bf /   ext4    errors=remount-ro 0    1
# /dev/fd0        /media/floppy0  auto    rw,user,noauto,exec,utf8 0       0
```

然后输入reboot重启即可，重启后使用top命令查看任务管理器，如果看到如下KiB Swap后均为 0 就说明关闭成功了。

关闭swap之后使用top命令看到 KiB Swap 全部为0

![image-20220613000133107](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/image-20220613000133107.png)

上面说的是永久关闭swap内存，其实也可以暂时关闭，使用swapoff -a命令即可，效果会在重启后消失。

### 禁用Selinux

```shell
sudo apt install selinux-utils
setenforce 0
```

### 确保时区和时间正确

```shell
sudo timedatectl set-timezone Asia/Shanghai
sudo systemctl restart rsyslog
sudo apt-get install ntpdate –y
sudo ntpdate time.windows.com
```

### 配置net.bridge.bridge-nf-call-iptables

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

### 设置rp_filter的值

```shell
sudo vi /etc/sysctl.d/10-network-security.conf

# 将下面两个参数的值从2修改为1
# net.ipv4.conf.default.rp_filter=1
# net.ipv4.conf.all.rp_filter=1

sudo sysctl --system
```

## 二、安装docker

docker 是 k8s 的基础，在安装完成之后也需要修改一些配置来适配 k8s ，所以本章分为 **docker 的安装** 与 **docker 的配置** 两部分。如果你已经安装并使用了一段时间的 docker 了话，建议使用`docker -v`查看已安装的 docker 版本，并在 k8s 官网上查询适合该版本的 k8s 进行安装。这一步两台主机都需要进行安装。

### 安装docker

```shell
sudo apt install docker.io

sudo apt install docker.io=18.09.1-0ubuntu1 
```

### 启动docker

```shell
sudo systemctl start docker
```

### 配置docker

```shell
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://n73pm3wf.mirror.aliyuncs.com"],
  "exec-opts": [ "native.cgroupdriver=systemd" ]
}
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 设置开机自启动

```shell
$ sudo systemctl enable docker
或
$ sudo systemctl enable docker.service --now
```

### 验证docker

``` shell
systemctl status docker
docker --version
docker info | grep Cgroup
```

修改后的 docker cgroup 状态，发现变为`systemd`即为修改成功。

### 重启docker（当有配置改动后执行）

```shell
sudo pkill -SIGHUP dockerd
sudo systemctl restart docker
```

### 卸载docker

```shell
sudo apt-get remove docker.io
```



## 三、安装k8s

### 安装k8s

```shell
# 使得 apt 支持 ssl 传输
apt-get update && apt-get install -y apt-transport-https
# 下载 gpg 密钥   这个需要root用户否则会报错
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
# 添加 k8s 镜像源 这个需要root用户否则会报错
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
# 更新源列表
apt-get update

# 下载 kubectl，kubeadm以及 kubelet
# 安装最新版本
apt-get install -y kubelet kubeadm kubectl
# 安装指定版本
apt-get install -y kubelet=1.15.1-00 kubeadm=1.15.1-00 kubectl=1.15.1-00
```

### 删除已安装的k8s

```shell
sudo apt-get remove -y --allow-change-held-packages kubeadm kubectl kubelet
```

### 查询可安装版本

```bash
apt-cache madison kubeadm
apt-cache madison kubelet
apt-cache madison kubectl
apt-cache madison kubernetes-cni
```

### 设置不随系统更新而更新

```shell
sudo apt-mark hold kubelet kubeadm kubectl
```

## 四、初始化master

### kubeadm init

```shell
kubeadm init \
--apiserver-advertise-address=10.10.0.2 \
--kubernetes-version=1.15.1 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=10.244.0.0/16
```

参数解释：

- --kubernetes-version：指定kubernetes的版本，与上面kubelet，kubeadm，kubectl工具版本保持一致。
- --apiserver-advertise-address：apiserver所在的节点(master)的ip。
- --image-repository=registry.aliyuncs.com/google_containers：由于国外地址无法访问，所以使用阿里云仓库地址
- --server-cidr：service之间访问使用的网络ip段
- --pod-network-cidr：pod之间网络访问使用的网络ip,与下文部署的CNI网络组件yml中保持一致

初始化成功之后你将看到

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.10.0.2:6443 --token juojiv.q4gwfmkj4cwgscnt \
        --discovery-token-ca-cert-hash sha256:381b61262d3b174cfce0e5cfbd0b4b171e270c506d82c4d82334d0a4e2c2ac47
```

### 初始化失败

重置之后重新初始化

```shell
kubeadm reset
```

### 记得保存kubeadm join 命令

```shell
kubeadm join 10.10.0.2:6443 --token qc7drh.47mloh5ierij84xi \
        --discovery-token-ca-cert-hash sha256:90f147ef85ae06d9a46bed91d64109abbd697b6878ab9ca16a376e11816f9a0d 
```

如果忘记可以使用下面的命令重新生成

```shell
kubeadm token create --print-join-command
```

### 配置kubectl

按照上图中的命令配置，可以直接从命令行中复制

- 非root用户

```shell
 mkdir -p $HOME/.kube
 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 查看已加入的节点
kubectl get nodes
# 查看集群状态
kubectl get cs
```

- root用户

```shell
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### 部署网络

网络插件可以选择calico或flannel。这里使用Flannel作为Kubernetes容器网络方案，解决容器跨主机网络通信。

- 部署flannel网络

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

也可以把这个文件下载到本地，注意修改下图中的Network，与kubeadm inti中的--pod-network-cidr 10.244.0.0/16参数保持一致，然后

````shell
kubectl apply -f kube-flannel.yml
````



![image-20220613013117789](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/image-20220613013117789.png)

将会看到

```bash
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

查看pod状态，应该全部为running。可能需要等待一段时间

```shell
kubectl get pod --all-namespaces -o wide
```

![image-20220613012658569](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/image-20220613012658569.png)

- 部署calico网络

下载yml文件

```shell
wget  https://docs.projectcalico.org/v3.11/manifests/calico.yaml

vi calico.yaml
```

\#修改CALICO_IPV4POOL_CIDR，为10.244.0.0/16（要与kubeadm inti中的--pod-network-cidr 10.244.0.0/16参数保持一致，默认为192.168.0.0/16

```shell
kubectl apply -f calico.yaml
```

## 五、添加worker节点

使用之前保存的kubeadm join命令

```shell
kubeadm join 10.10.0.2:6443 --token ryyy8k.pr8u7cjm4btqdx4v --discovery-token-ca-cert-hash sha256:6b596c66745b162190475b63686ca46c60cc4f39473
```

然后在master节点上，查看

```shell
kubectl get nodes
```

可以看到

![image-20220613010009663](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/image-20220613010009663.png)

## 六、验证

### 创建nginx

```shell
kubectl create deployment nginx-web --image=nginx
```

### 获取pod信息

```shell
kubectl get pod -o wide
```

![image-20220613013920336](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/image-20220613013920336.png)

可以看到该节点的ip为10.244.1.2

### 测试nginx

```shell
root@k8s-master:/home# curl http://10.244.1.2
```

可以看到

```shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 删除pod

```shell
kubectl delete pod nginx-web-xxxxxx -n nginx-web
```

## 七、删除节点

首先，确定想要清空的节点的名称。可以用以下命令列出集群中的所有节点:

```shell
kubectl get nodes
```

接下来，告诉 Kubernetes 清空节点：

```bash
kubectl drain <node name>
```

一旦它返回（没有报错）， 你就可以下线此节点（或者等价地，如果在云平台上，删除支持该节点的虚拟机）。 如果要在维护操作期间将节点留在集群中，则需要运行：

```bash
kubectl uncordon <node name>
```

然后告诉 Kubernetes，它可以继续在此节点上调度新的 Pods。
