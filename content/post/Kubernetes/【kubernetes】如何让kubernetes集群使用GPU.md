---
title: "【Kubernetes】如何让kubernetes集群使用GPU"
author: "Tweakzx"
description: 
date: 2023-01-09T20:06:52+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories: kubernetes
tags: 
    - GPU
    - GPU 集群
---

## 在Kubernetes集群中使用GPU资源

在前文中， 我们搭建了k8s集群， 但是k8s原生不支持GPU资源， 需要使用各大GPU厂商开发的插件才能使用。

这一部分大家可以参考[NVIDIA/k8s-device-plugin: NVIDIA device plugin for Kubernetes (github.com)](https://github.com/NVIDIA/k8s-device-plugin#quick-start)

本文就记录我在k8s集群中配置插件的过程。

## 安装前的环境配置

- Centos Linux realease 7.9.2009
- 内核：3.10.0-1062.18.1.el7.x86_64
- Docker-ce：18.09.7
- Kubernetes：1.16.15
- Nvidia Driver： 460.32.03两台，510.47.03一台

## 安装`nvidia-docker2`

> 如果你使用的是新版本的docker (>=19.03)，则推荐使用nvidia-container-toolkit包来代替nvidia-docker2包：

由于我的docker版本是18.09.7， 所以我需要安装nvidia-docker2

```bash
##设置仓库
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo | \
sudo tee /etc/yum.repos.d/nvidia-docker.repo

##更新仓库中的key
DIST=$(sed -n 's/releasever=//p' /etc/yum.conf)
DIST=${DIST:-$(. /etc/os-release; echo $VERSION_ID)}
sudo yum makecache

##安装nvidia-docker2
sudo yum install nvidia-docker2
```

这时候， 查看/etc/docker/daemon.json，你会发现内容已经改了， 内容如下

```bash
{
  "runtimes": {
     "nvidia": {
         "path": "nvidia-container-runtime",
         "runtimeArgs": []
      }
   }
}
```

但是， 我还有一些其他的设置， 例如cgroupdriver， 镜像等等也需要设置， 所以我的修改了daemon.json，内容如下

```bash
{
  "default-runtime": "nvidia",
  "registry-mirrors": ["https://n73pm3wf.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "runtimes": {
     "nvidia": {
         "path": "nvidia-container-runtime",
         "runtimeArgs": []
      }
   }
}
```

修改完之后，记得重启docker

```bash
##重启docker
systemctl restart docker
```

特别注意一个字段`"default-runtime": "nvidia",` 这个default-runtime需要加上才可以在k8s中找到设备。

加了之后重启docker， 使用

```bash
docker run --security-opt=no-new-privileges --cap-drop=ALL --network=none -it -v /var/lib/kubelet/device-plugins:/var/lib/kubelet/device-plugins nvidia/k8s-device-plugin:1.11
```

如果成功的话会出现

```
2023/01/09 12:56:36 Loading NVML
2023/01/09 12:56:36 Fetching devices.
2023/01/09 12:56:37 Starting FS watcher.
2023/01/09 12:56:37 Starting OS watcher.
2023/01/09 12:56:37 Starting to serve on /var/lib/kubelet/device-plugins/nvidia.sock
2023/01/09 12:56:37 Registered device plugin with Kubelet
```

如果不加的话，使用如下命令也可以验证docker是否可以使用GPU

```bash
docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi
```

等待几秒，下载完成后出现会以下结果

```
# docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi
Mon Jan  9 12:33:59 2023       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 510.47.03    Driver Version: 510.47.03    CUDA Version: 11.6     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla V100-PCIE...  Off  | 00000000:02:00.0 Off |                    0 |
| N/A   28C    P0    24W / 250W |      0MiB / 16384MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   1  Tesla V100-PCIE...  Off  | 00000000:03:00.0 Off |                    0 |
| N/A   29C    P0    25W / 250W |      0MiB / 16384MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   2  Tesla V100-PCIE...  Off  | 00000000:82:00.0 Off |                    0 |
| N/A   30C    P0    24W / 250W |      0MiB / 16384MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   3  Tesla V100-PCIE...  Off  | 00000000:83:00.0 Off |                    0 |
| N/A   28C    P0    23W / 250W |      0MiB / 16384MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## 在k8s上启动nvidia-device-plugin

在Master节点上运行

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.11/nvidia-device-plugin.yml
```

然后使用以下命令查看

```bash
kubectl get pods -A -owide
```

可以发现， 相关的pod已经起来了，对应每个修改了docker运行时的GPU节点都有一个pod

```
kube-system   nvidia-device-plugin-daemonset-2bhzz  1/1     Running     0          59m
kube-system   nvidia-device-plugin-daemonset-l5djd  1/1     Running     0          59m
kube-system   nvidia-device-plugin-daemonset-lt756  1/1     Running     0          60m
```

使用

```
kubectl describe nodes
```

可以发现可分配资源里出现了nvidia.com/gpu的字段

```
Capacity:
 cpu:                56
 ephemeral-storage:  2148327Mi
 hugepages-1Gi:      0
 hugepages-2Mi:      0
 memory:             65678476Ki
 nvidia.com/gpu:     4
 pods:               110
Allocatable:
 cpu:                56
 ephemeral-storage:  2027415715761
 hugepages-1Gi:      0
 hugepages-2Mi:      0
 memory:             65576076Ki
 nvidia.com/gpu:     4
 pods:               110
System Info:
```

## 测试

我们使用官方文档里的测试方法

```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  restartPolicy: Never
  containers:
    - name: cuda-container
      image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda10.2
      resources:
        limits:
          nvidia.com/gpu: 1 # requesting 1 GPU
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
EOF
```

可以得到

```
# kubectl logs gpu-pod 
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

## 遇到的问题

> Failed to initialize NVML: could not load NVML library.If this is a GPU node, did you set the docker default runtime to `nvidia`?

你可能没有设置`"default-runtime": "nvidia"字段， 设置之后重启docker， 将pod全部删除重启，一定要注意操作顺序， 安装docker， 配置/etc/docker/daemon.json， 重启docker， 启动k8s-device-plugin, 进行测试。

```
kubectl delete -n kube-system <POD NAME>
```

## 如果需要给不同型号GPU打Label

```
# Label your nodes with the accelerator type they have.
kubectl label nodes <node-with-k80> accelerator=nvidia-tesla-k80
kubectl label nodes <node-with-p100> accelerator=nvidia-tesla-p100
```

调用的时候， 指定nodeSelector条件

```
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vector-add
spec:
  restartPolicy: OnFailure
  containers:
    - name: cuda-vector-add
      # https://github.com/kubernetes/kubernetes/blob/v1.7.11/test/images/nvidia-cuda/Dockerfile
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100 # or nvidia-tesla-k80 etc.
```

