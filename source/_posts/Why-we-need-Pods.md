---
title: 深入剖析K8s筆記(爲什麽我們需要Pod)
mathjax: true
copyright: true
comment: true
date: 2022-01-31 01:17:49
categories: "Ops "
tags: 
- "Linux "
- "Kubernetes "
- "Pod "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/pod.png" width=50% /></center>

## 0x00 前言
Pod，是Kubernetes项目的原子调度单位。但为什么是呢？

<!--more -->
## 0x01 Pod的第一层意义
**容器其本质是特殊的进程**，而K8s身为一个Paas领军产物，设计之初就借鉴了OS。

来，先观察一下Linux的进程树结构：

```bash
# pstree -g |more
systemd(1)-+-acpid(680)
           |-agetty(1924)
           |-agetty(1925)
           |-atd(1462)
           |-auditd(614)---{auditd}(614)
           |-barad_agent(10479)-+-barad_agent(10479)
           |                    `-barad_agent(10479)-+-{barad_agent}(10479)
           |                                         |-{barad_agent}(10479)
           |                                         `-{barad_agent}(10479)
           |-containerd-shim(860)-+-entrypoint.sh(15567)---sleep(15567)
           |                      |-pause(878)
           |                      |-sleep(12230)
           |                      |-{containerd-shim}(860)
           |                      |-{containerd-shim}(860)
           |                      |-{containerd-shim}(860)
           |                      |-{containerd-shim}(860)
           |                      |-{containerd-shim}(860)
           |                      |-{containerd-shim}(860)
           |                      |-{containerd-shim}(860)
           |                      |-{containerd-shim}(860)
           |                      |-{containerd-shim}(860)
           |                      |-{containerd-shim}(860)
           |                      |-{containerd-shim}(860)
           |                      `-{containerd-shim}(860)
```
那跟K8s有什么关系？

用[krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install/)装个[tree](https://github.com/ahmetb/kubectl-tree)看看。

```bash
# kubectl tree deployment nginx-deployment
NAMESPACE  NAME                                       READY  REASON  AGE  
default    Deployment/nginx-deployment                -              24h  
default    ├─ReplicaSet/nginx-deployment-5d59d67564   -              24h  
default    ├─ReplicaSet/nginx-deployment-64c9d67564   -              7h48m
default    ├─ReplicaSet/nginx-deployment-66db4f9b59   -              7h32m
default    └─ReplicaSet/nginx-deployment-848bcb569b   -              7h30m
default      ├─Pod/nginx-deployment-848bcb569b-7zk57  True           7h30m
default      └─Pod/nginx-deployment-848bcb569b-w9mzm  True           7h30m
# kubectl tree deployment csi-resizer -n longhorn-system
NAMESPACE        NAME                                  READY  REASON  AGE  
longhorn-system  Deployment/csi-resizer                -              11d  
longhorn-system  └─ReplicaSet/csi-resizer-6dd8bd4c97   -              11d  
longhorn-system    ├─Pod/csi-resizer-6dd8bd4c97-gvwm9  True           2d21h
longhorn-system    ├─Pod/csi-resizer-6dd8bd4c97-nxg5n  True           2d21h
longhorn-system    └─Pod/csi-resizer-6dd8bd4c97-xjcwn  True           10d
```
至此，对Pod的理解又加深一层，Pod于K8s的意义，跟进程于Linux的意义，跃然纸上。

## 0x02 Pod的第二层意义
为什么不是container，而是Pod呢？

**Pod其实只是逻辑概念**，K8s管的其实是各node的经脉（namespace、Cgroups），**Pod更重要的意义，使容器可以共享Network Namespace，以及Volume。**

而container需要共享时，还得先按序启动。

```bash
# docker run --net=B --volumes-from=B --name=A imgae
```
不像Pod，靠Pause容器，就可保证各container相互对等。

```bash
├─containerd-shim(10811)─┬─minio(10859)─┬─{minio}(10859)
           │                        │              ├─{minio}(10859)
           │                        │              ├─{minio}(10859)
           │                        │              ├─{minio}(10859)
           │                        │              └─{minio}(10859)
           │                        ├─pause(10831)
           │                        ├─{containerd-shim}(10811)
           │                        ├─{containerd-shim}(10811)
           │                        ├─{containerd-shim}(10811)
           │                        ├─{containerd-shim}(10811)
           │                        ├─{containerd-shim}(10811)
```
有什么好处呢？

* localhost可以直接通信。
* container内网络设备 == pause容器（Infra容器）
* Pod生命周期关联pause container（Infra container）
* 一个Pod一个IP，也就是一个Network Namespace一个IP

```bash
# kubectl get pods  -l app=nginx -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE                        NOMINATED NODE   READINESS GATES
nginx-deployment-848bcb569b-7zk57   1/1     Running   0          13h     10.42.2.166   storage-k3sw2.m4d3bug.com   <none>           <none>
nginx-deployment-848bcb569b-w9mzm   1/1     Running   0          13h     10.42.5.242   storage-k3sw4.m4d3bug.com   <none>           <none>
nginx-deployment-848bcb569b-ztt4r   1/1     Running   0          3h38m   10.42.3.32    storage-k3sw3.m4d3bug.com   <none>           <none>
nginx-deployment-848bcb569b-dvc7t   1/1     Running   0          3h38m   10.42.1.42    storage-k3sw1.m4d3bug.com   <none>           <none>
```
* Pod内资源所有containers共享，例如存储

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:
      path: /var/tmp. # 宿主机路径
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html && sleep 999"] 
                                                                              #sleep续命，pod才会是ready
```
* (题外话) 不共享的Volumes也可以这样写

```yaml
apiVersion: apps/v1
kind: Deployment # 控制器用于控制pod行为 controller pattern
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 4 #保证自动调动
  template:
    metadata: #存放要监管的对象
      labels: #强行关联Deployment纳管app：nginx为管理对象
        app: nginx
     #annotations: 通告给Kubernetes组件本身，而不是用户
    spec: #存放该对象的独有定义
      containers:
      - name: nginx
        image: nginx:1.8
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html" #指定容器内路径
          name: nginx-vol
      volumes:
      - name: nginx-vol
        # emptyDir: {} #隐式声明自动创建宿主机的临时目录
        hostPath:
          path: "/tmp" #显式声明宿主机的永久目录，搭配nfs挂载可以做到多副本存活
```
## 0x03 Pod的最终意义
有了前面的铺垫，我们可以得到以下的对照关系。

|线程|进程组|操作系统OS|
| ----- | ----- | ----- |
|容器|Pod|Kubernetes|

凭借Pod的特性，我们打通了不同容器、不同软件的耦合关系，并且像进程组一样共享了各种resource。

而这种“组合”，也正是sidecar的工作思路，通过共享目录不染指业务容器达到协管（我更喜欢译成随航而不是边车。

## 0x04 总结
简单探讨了Pod于K8s的含义，帮助自己理解消化总结。
