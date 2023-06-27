---
title: "【Kubernetes】ubuntu脚本安装k8s集群"
author: "Tweakzx"
description: 
date: 2023-04-21T16:06:05+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories: 
tag: script
---

## 安装k8s-1.18.9版本的脚本

- 下载之后最好修改

  - KUBE_VERSION="1.18.9" 以安装自己想要的版本
  - MASTER1_IP=    自己的master IP
  - NODE1_IP= 集群的节点IP

- 可以自定义修改， 但是也可以不动

  - POD_NETWORK="10.244.0.0/16"
  - SERVICE_NETWORK="10.96.0.0/16"

- 网络插件

  - 这个脚本默认使用flannel
  - 如果使用calico记得修改calico脚本里的pod network

- github

  - 项目地址：[Tweakzx/k8s-scripts (github.com)](https://github.com/Tweakzx/k8s-scripts)

  - ```shell 
    wget https://raw.githubusercontent.com/Tweakzx/k8s-scripts/main/k8s-install.sh
    ```

```shell
# !/bin/bash
KUBE_VERSION="1.18.9"
KUBE_VERSION2=$(echo $KUBE_VERSION |awk -F. '{print $2}')

DOCKER_VERSION=""

MASTER1_IP=10.10.0.25
NODE1_IP=10.10.0.26

MASTER1=master
NODE1=node1

POD_NETWORK="10.244.0.0/16"
SERVICE_NETWORK="10.96.0.0/16"

. /etc/os-release
IMAGES_URL="registry.aliyuncs.com/google_containers"
CRI_DOCKER_VERSION=0.2.6
CRI_DOCKER_URL="https://ghproxy.com/https://github.com/Mirantis/cri-dockerd/releases/download/v${CRI_DOCKER_VERSION}/cri-dockerd_${CRI_DOCKER_VERSION}.3-0.ubuntu-${UBUNTU_CODENAME}_amd64.deb"

LOCAL_IP=`hostname -I |awk '{print $1}'`

COLOR_SUCCESS="echo -e \\033[1;32m"
COLOR_FAILURE="echo -e \\033[1;31m"
END="\033[m"

color () {
    RES_COL=60
    MOVE_TO_COL="echo -en \\033[${RES_COL}G"
    RES_COL=50
    SETCOLOR_SUCCESS="echo -en \\033[1;32m"
    SETCOLOR_FAILURE="echo -en \\033[1;31m"
    SETCOLOR_WARNING="echo -en \\033[1;33m"
    SETCOLOR_NORMAL="echo -en \E[0m"
    echo -n "$1" && $MOVE_TO_COL
    echo -n "["
    if [ $2 = "success" -o $2 = "0" ] ;then
        ${SETCOLOR_SUCCESS}
        echo -n $"OK"
    elif [ $2 = "failure" -o $2 = "1" ] ;then
        ${SETCOLOR_FAILURE}
        echo -n $"FAILED"
    else
        ${SETCOLOR_WARNING}
        echo -n $"WARNING"
    fi
    ${SETCOLOR_NORMAL}
    echo -n "]"
    echo 
}

install_prepare () {
    ${COLOR_SUCCESS}"开始处理环境的基本配置..."${END}
    cat > /etc/hosts <<EOF
127.0.0.1 localhost
$MASTER1_IP $MASTER1
$NODE1_IP $NODE1
EOF
    hostnamectl set-hostname $(awk -v ip=$LOCAL_IP '{if($1==ip)print $2}' /etc/hosts)
    swapoff -a
    sed -i '/swap/s/^/#/' /etc/fstab
    cat >/etc/modules-load.d/k8s.conf <<EOF
modprobe overlay
modprobe br_netfilter
EOF
    cat >/etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
    sysctl --system
    color "安装前准备完成！" 0
    sleep 1
}

install_docker () {
    ${COLOR_SUCCESS}"开始安装docker..."${END}
    apt update
    apt -y install docker.io || { color "安装Docker失败！" 1; exit 1; }
    cat > /etc/docker/daemon.json << EOF
{
"registry-mirrors": [
"https://docker.mirrors.ustc.edu.cn",
"https://hub-mirror.c.163.com",
"https://reg-mirror.qiniu.com",
"https://registry.docker-cn.com"
],
"exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
    systemctl restart docker.service
    docker info && { color "安装 Docker 成功" 0; sleep 1; } || { color "安装 Docker 失败！" 1; exit; }
}

install_kubeadm () {
    ${COLOR_SUCCESS}"kubeadm依赖安装..."${END}
    apt-get update && apt-get install -y apt-transport-https curl
    curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
    cat << EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrorS.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
    apt-get update
    apt-cache madison kubeadm | head
    ${COLOR_FAILURE}"5秒后即将安装： kubeadm-"${KUBE_VERSION}" 版本......"${END}
    ${COLOR_FAILURE}"如果想安装其它版，请桜 ctrl + c 键退出，修改版本再执行"${END}
    sleep 6

    #安装指定版本
    apt install -y kubeadm=${KUBE_VERSION}-00 kubelet=${KUBE_VERSION}-00 kubectl=${KUBE_VERSION}-00
    [ $? -eq 0 ] && { color "安装 kubeadm 成功！" 0; sleep 1;} || { color "安装 kubeadm 失败！" 1; exit 2; }

}

# Kubernetes -v1.24之前版本无需安装 cri-dockerd 
install_cri_dockerd () {
    ${COLOR_SUCCESS}"开始安装cri-dockerd..."${END}
    [ $KUBE_VERSION2 -lt 24 ] && return
    if [ ! -e cri-dockerd_${CRI_DOCKER_VERSION}.3-0.ubuntu-${UBUNTU_CODENAME}_amd64.deb ] ;then
        curl -LO $CRI_DOCKER_URL || { color "下载cri-dockerd失败！" 1; exit 2; }
    fi
    dpkg -i cri-dockerd_${CRI_DOCKER_VERSION}.3-0.ubuntu-${UBUNTU_CODENAME}_amd64.deb
    sed -i -e 's#ExecStart=.*#ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --network-plugin=cni --pod-infra-container-image='"$IMAGES_URL"'/pause:3.7#g' /lib/systemd/system/cri-docker.service
    systemctl daemon-reload
    systemctl restart cri-docker.service

}

kubernetes_init () {
    ${COLOR_SUCCESS}"开始初始化k8s..."${END}
    kubeadm init \
--kubernetes-version=v${KUBE_VERSION} \
--apiserver-advertise-address=$(MASTER1_IP) \
--image-repository registry.aliyuncs.com/google_containers \
--service-cidr=${SERVICE_NETWORK} \
--pod-network-cidr=${POD_NETWORK} \
--v=5

    if [ $? -eq 0 ]; then
        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
        color "初始化 k8s 成功！" 0
    else
        color "初始化 k8s 失败！" 1
    fi
}

reset_kubernetes () {
    #To do
    kubeadm reset
    rm -rf /etc/kubernetes/*
    rm -rf $HOME/.kube/
}

configure_network () {
    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
}

remove_docker () {
    apt-get autoremove docker docker-ce docker-engine  docker.io  containerd runc
    dpkg -l |grep ^rc|awk '{print $2}' |sudo xargs dpkg -P
    rm -rf /etc/systemd/system/docker.service.d
    rm -rf /var/lib/docker
}

install_helm () {
    wget https://mirrors.huaweicloud.com/helm/v3.1.1/helm-v3.1.1-linux-amd64.tar.gz
    tar -zxvf helm-v3.1.1-linux-amd64.tar.gz
    mv linux-amd64/helm /usr/local/bin/helm
}

remove_kubernetes () {
    #To do
    apt remove -y kubeadm kubectl kubelet
    sudo rm -rf /etc/kubernetes /var/lib/kubernetes /var/lib/etcd /var/lib/kubelet /var/run/kubernetes ~/.kube
    sudo ip link delete cni0
    sudo ip link delete flannel.1
}

main () {
    PS3="请选择编号（1-8）"
    options=("初始化kubernetes集群" "加入kubernetes集群" "退出kubernetes集群" "配置网络" "安装helm" "删除kubernetes" "删除docker" "退出本程序")
    select item in "${options[@]}"; do
        case $item in
            "初始化kubernetes集群")
                install_prepare
                install_docker
                install_kubeadm
                install_cri_dockerd
                kubernetes_init
                break
                ;;
            "加入kubernetes集群")
                install_prepare
                install_docker
                install_kubeadm
                install_cri_dockerd
                ${COLOR_SUCCESS}"加入kubernetes前的准备工作已完成，请执行kubeadm join命令！"${END}
                break
                ;;
            "退出kubernetes集群")
                reset_kubernetes
                break
                ;;
            "配置网络")
                configure_network
                break
                ;;
            "安装helm")
                install_helm
                break
                ;;
            "删除kubernetes")
                reset_kubernetes
                remove_kubernetes
                break
                ;;
            "删除docker")
                remove_docker
                break
                ;;
            "退出本程序")
                exit
                ;;
            *) echo "invalid option $REPLY";;
        esac
    done
}

main
                                                                                                                 217,4         Bot
```

