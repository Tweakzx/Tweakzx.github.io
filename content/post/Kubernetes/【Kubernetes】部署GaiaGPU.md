---
title: "【Kubernetes】部署gaiaGPU（vCUDA）"
author: "Tweakzx"
date: 2023-06-27T14:57:42+08:00
description: 
categories: kubernetes
tags: 
- 容器
- k8s
- gaiagpu
image: 
draft: false
---



# 部署GaiaGPU

## 前置工作

- 配置好GPU环境
- 配置好k8s集群环境

## gpu-admission

### 部署deployment

创建文件```gpu-quota-admission.yaml```

```yaml
---
apiVersion: 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gpu-admission
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gpu-admission-as-kube-scheduler
subjects:
- kind: ServiceAccount
  name: gpu-admission
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:kube-scheduler
  apiGroup: rbac.authorization.k8s.io
  
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gpu-admission-as-volume-scheduler
subjects:
- kind: ServiceAccount
  name: gpu-admission
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:volume-scheduler
  apiGroup: rbac.authorization.k8s.io
  
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gpu-admission-as-daemon-set-controller
subjects:
- kind: ServiceAccount
  name: gpu-admission
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:controller:daemon-set-controller
  apiGroup: rbac.authorization.k8s.io

---
# 创建控制器
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: gpu-quota-admission
  name: gpu-quota-admission
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: gpu-quota-admission
  template:
    metadata:
      labels:
        k8s-app: gpu-quota-admission
      namespace: kube-system
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: node-role.kubernetes.io/master
                operator: Exists
            weight: 1
      containers:
      - env:
        - name: LOG_LEVEL
          value: "4"
        - name: EXTRA_FLAGS
          value: --incluster-mode=true
        image: ccr.ccs.tencentyun.com/tkeimages/gpu-quota-admission:latest
        imagePullPolicy: IfNotPresent
        name: gpu-quota-admission
        ports:
        - containerPort: 3456
          protocol: TCP
        resources:
          limits:
            cpu: "2"
            memory: 2Gi
          requests:
            cpu: "1"
            memory: 1Gi
        volumeMounts:
        - mountPath: /root/gpu-quota-admission/
          name: config
      dnsPolicy: ClusterFirstWithHostNet
      initContainers:
      - command:
        - sh
        - -c
        - ' mkdir -p /etc/kubernetes/ && cp /root/gpu-quota-admission/gpu-quota-admission.config
          /etc/kubernetes/'
        image: busybox
        imagePullPolicy: Always
        name: init-kube-config
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /root/gpu-quota-admission/
          name: config
      priority: 2000000000
      priorityClassName: system-cluster-critical
      restartPolicy: Always
      serviceAccount: gpu-admission
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      volumes:
      - configMap:
          defaultMode: 420
          name: gpu-quota-admission
        name: config

---
# 创建configMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: gpu-quota-admission
  namespace: kube-system
data:
  gpu-quota-admission.config: |
    {
         "QuotaConfigMapName": "gpuquota",
         "QuotaConfigMapNamespace": "kube-system",
         "GPUModelLabel": "gaia.tencent.com/gpu-model",
         "GPUPoolLabel": "gaia.tencent.com/gpu-pool"
     }

---
# 创建service
apiVersion: v1
kind: Service
metadata:
  name: gpu-quota-admission
  namespace: kube-system
spec:
  ports:
  - port: 3456
    protocol: TCP
    targetPort: 3456
  selector:
    k8s-app: gpu-quota-admission
  type: ClusterIP
```

``` shell
kubectl apply -f gpu-quota-admission.yaml
```

### 创建自定义调度文件

创建文件```/etc/kubernetes/scheduler-policy-config.json```

``` yaml
{
  "kind": "Policy",
  "apiVersion": "v1",
  "predicates": [
    {
      "name": "PodFitsHostPorts"
    },
    {
      "name": "PodFitsResources"
    },
    {
      "name": "NoDiskConflict"
    },
    {
      "name": "MatchNodeSelector"
    },
    {
      "name": "HostName"
    }
  ],
  "extenders": [
    {
      "urlPrefix": "http://gpu-quota-admission.kube-system:3456/scheduler",
      "apiVersion": "v1beta1",
      "filterVerb": "predicates",
      "enableHttps": false,
      "nodeCacheCapable": false,
      "managedResources": [
       {
         "name": "tencent.com/vcuda-memory",
         "ignoredByScheduler": false
       },
       {
         "name": "tencent.com/vcuda-core",
         "ignoredByScheduler": false
       }
      ]
    }
  ],
  "hardPodAffinitySymmetricWeight": 10,
  "alwaysCheckAllPredicates": false
}
```

### 修改默认调度器

```/etc/kubernetes/manifests/kube-scheduler.yaml```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --policy-config-file=/etc/kubernetes/scheduler-policy-config.json #增加项
    - --use-legacy-policy-config=true 								    #增加项
    image: registry.aliyuncs.com/google_containers/kube-scheduler:v1.23.17
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
    - mountPath: /etc/kubernetes/scheduler-policy-config.json #将文件挂载
      name: policyconfig
      readOnly: true
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet                        # 修改dns策略
  priorityClassName: system-node-critical
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  volumes:
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
  - hostPath:                                             #挂载到该 Pod 中的容器内
      path: /etc/kubernetes/scheduler-policy-config.json
      type: FileOrCreate
    name: policyconfig
status: {}
```

修改后wq保存， 保存后自动生效

## gpu-manager

### 给gpu节点打标签

``` shell
kubectl label node <your node> nvidia-device-enable=enable
```

### 部署deamonset

创建文件```gpu-manager.yaml```

```yaml
---
# 创建ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gpu-manager
  namespace: kube-system

---
# 创建ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gpu-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: gpu-manager
  namespace: kube-system
    
---
#创建DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-manager-daemonset
  namespace: kube-system
spec:
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      name: gpu-manager-ds
  template:
    metadata:
      # This annotation is deprecated. Kept here for backward compatibility
      # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
      labels:
        name: gpu-manager-ds
    spec:
      serviceAccount: gpu-manager
      tolerations:
        # This toleration is deprecated. Kept here for backward compatibility
        # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
        - key: CriticalAddonsOnly
          operator: Exists
        - key: tencent.com/vcuda-core
          operator: Exists
          effect: NoSchedule
      	- key: node-role.kubernetes.io/master ##增加容忍策略
          effect: NoSchedule
      # Mark this pod as a critical add-on; when enabled, the critical add-on
      # scheduler reserves resources for critical add-on pods so that they can
      # be rescheduled after a failure.
      # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
      priorityClassName: "system-node-critical"
      # only run node has gpu device
      nodeSelector:
        nvidia-device-enable: enable
      hostPID: true
      containers:
        - image: tkestack/gpu-manager:v1.1.5  #指定镜像版本
          imagePullPolicy: IfNotPresent       #使用本地镜像
          name: gpu-manager
          securityContext:
            privileged: true
          ports:
            - containerPort: 5678
          volumeMounts:
            - name: device-plugin
              mountPath: /var/lib/kubelet/device-plugins
            - name: vdriver
              mountPath: /etc/gpu-manager/vdriver
            - name: vmdata
              mountPath: /etc/gpu-manager/vm
            - name: log
              mountPath: /var/log/gpu-manager
            - name: checkpoint
              mountPath: /etc/gpu-manager/checkpoint
            - name: run-dir
              mountPath: /var/run
            - name: cgroup
              mountPath: /sys/fs/cgroup
              readOnly: true
            - name: usr-directory
              mountPath: /usr/local/host
              readOnly: true
          env:
            - name: LOG_LEVEL
              value: "4"
            - name: EXTRA_FLAGS        #增加启动参数⬇
              value: "--logtostderr=false --cgroup-driver=systemd"  
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
      volumes:
        - name: device-plugin
          hostPath:
            type: Directory
            path: /var/lib/kubelet/device-plugins
        - name: vmdata
          hostPath:
            type: DirectoryOrCreate
            path: /etc/gpu-manager/vm
        - name: vdriver
          hostPath:
            type: DirectoryOrCreate
            path: /etc/gpu-manager/vdriver
        - name: log
          hostPath:
            type: DirectoryOrCreate
            path: /etc/gpu-manager/log
        - name: checkpoint
          hostPath:
            type: DirectoryOrCreate
            path: /etc/gpu-manager/checkpoint
        # We have to mount the whole /var/run directory into container, because of bind mount docker.sock
        # inode change after host docker is restarted
        - name: run-dir
          hostPath:
            type: Directory
            path: /var/run
        - name: cgroup
          hostPath:
            type: Directory
            path: /sys/fs/cgroup
        # We have to mount /usr directory instead of specified library path, because of non-existing
        # problem for different distro
        - name: usr-directory
          hostPath:
            type: Directory
            path: /usr
    
---
# 创建service
apiVersion: v1
kind: Service
metadata:
  name: gpu-manager-metric
  namespace: kube-system
  annotations:
    prometheus.io/scrape: "true"
  labels:
    kubernetes.io/cluster-service: "true"
spec:
  clusterIP: None
  ports:
    - name: metrics
      port: 5678
      protocol: TCP
      targetPort: 5678
  selector:
    name: gpu-manager-ds
```

## 测试

### 创建测试文件

创建文件```tensorflow-mnist.yaml```

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: vcuda-test
    qcloud-app: vcuda-test
  name: vcuda-test
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: vcuda-test
  template:
    metadata:
      labels:
        k8s-app: vcuda-test
        qcloud-app: vcuda-test
    spec:
			tolerations:
      	- key: node-role.kubernetes.io/master  ##容忍策略
        	effect: NoSchedule
      containers:
      - command:
        - sleep
        - 360000s
        env:
        - name: PATH
          value: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        image: ccr.ccs.tencentyun.com/menghe/tensorflow-gputest:0.2
        imagePullPolicy: IfNotPresent
        name: tensorflow-test
        resources:
          limits:
            cpu: "4"
            memory: 8Gi
            tencent.com/vcuda-core: "50"
            tencent.com/vcuda-memory: "35"
          requests:
            cpu: "4"
            memory: 8Gi
            tencent.com/vcuda-core: "50"
            tencent.com/vcuda-memory: "35"
```

```shell
kubectl apply -f tensorflow-mnist.yaml
```

