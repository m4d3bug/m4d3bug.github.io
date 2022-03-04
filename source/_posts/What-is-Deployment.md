---
layout: posts
title: 深入剖析K8s筆記(什麽是Deployment)
mathjax: true
copyright: true
comment: true
date: 2022-03-05 00:38:57
categories: "Ops "
tags:
- "Linux "
- "Kubernetes "
- "Deployment "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/deployment.png" width=50% /></center>

## 0x00 前言

开门见山，这篇blog的主角是Deployment。

<!-- more -->

## 0x01 扩展/收缩 --- ReplicaSet
摘抄官网 [ReplicaSet | Kubernetes](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/)

> ReplicaSet 确保任何时间都有指定数量的 Pod 副本在运行。 然而，Deployment 是一个更高级的概念，它管理 ReplicaSet，并向 Pod 提供声明式的更新以及许多其他有用的功能。 因此，我们建议使用 Deployment 而不是直接使用 ReplicaSet，除非 你需要自定义更新业务流程或根本不需要更新。

`ReplicaSet`也是`Deployment`管控的API[对象之一](https://blog.madebug.net/ops/2022-02-19-k8s-controller-explaination#0x02-Controller%E7%9A%84%E5%AF%B9%E8%B1%A1)，主要负责Pod 的“水平扩展 / 收缩”（horizontal scaling out/in）。

不信？举个🌰：

```bash
# cat replicaset.yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-set
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9

# kubectl apply -f deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

# kubectl get rs -l app=nginx
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-848bcb569b   3         3         3       5m15s
# kubectl get deployment nginx-deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           6m55s
# kubectl get pods -l app=nginx 
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-848bcb569b-rhhlt   1/1     Running   0          8m16s
nginx-deployment-848bcb569b-wp29j   1/1     Running   0          8m16s
nginx-deployment-848bcb569b-8nw82   1/1     Running   0          8m16s
```
apply了Deployment，ReplicaSet也被创建了，也不难推出它们的关系。

[drawio](gao8Do6yj1f-q2DHNmohGz9hkhON_8pTlhcLGZsiYRU.svg)

实现水平扩展 / 收缩的核心，就需要改改yml，或者使用`kubectl scale` 

```bash
$ kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
```
## 0x02 滚动更新
### 实验一：何为滚动更新？
一个执行

```bash
# 用户提交一个deployment对象，启用--record并使用rollout实时监控。
# kubectl apply -f deployment.yml --record && kubectl rollout status deployment/nginx-deployment
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/nginx-deployment created
Waiting for deployment "nginx-deployment" rollout to finish: 0 of 3 updated replicas are available...
Waiting for deployment "nginx-deployment" rollout to finish: 1 of 3 updated replicas are available...
Waiting for deployment "nginx-deployment" rollout to finish: 2 of 3 updated replicas are available...
deployment "nginx-deployment" successfully rolled out
```
一个监控`deployment`

```bash
# kubectl get deployments --watch
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/3     0            0           0s
nginx-deployment   0/3     0            0           0s
nginx-deployment   0/3     0            0           0s
nginx-deployment   0/3     3            0           0s
nginx-deployment   1/3     3            1           1s
nginx-deployment   2/3     3            2           2s
nginx-deployment   3/3     3            3           2s
```
一个监控R`eplicaSet`

```bash
# kubectl get rs --watch
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-848bcb569b   3         0         0       0s
nginx-deployment-848bcb569b   3         0         0       0s
nginx-deployment-848bcb569b   3         3         0       0s
nginx-deployment-848bcb569b   3         3         1       2s
nginx-deployment-848bcb569b   3         3         2       2s
nginx-deployment-848bcb569b   3         3         3       2s
```
解释一下字段，
* DESIRED：用户期望的 Pod 副本个数（spec.replicas 的值）；
* CURRENT：当前处于 Running 状态的 Pod 的个数；
* UP-TO-DATE：当前处于最新版本的 Pod 的个数，所谓最新版本指的是 Pod 的 Spec 部分与 Deployment 里 Pod 模板里定义的完全一致；**（Deployment独有，用于监控其版本是否一致。）**
* AVAILABLE：当前已经可用的 Pod 的个数，即：既是 Running 状态，又是最新版本，并且已经处于 Ready（健康检查正确）状态的 Pod 的个数。

### 实验二：观察UP-TO-DATE
一个执行

```bash
$ kubectl edit deployment/nginx-deployment
... 
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1 # 1.7.9 -> 1.9.1
        ports:
        - containerPort: 80
...
:wq
deployment.extensions/nginx-deployment edited
```
一个监控`deployment` 

```bash
# kubectl get deployments --watch
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           15m
nginx-deployment   3/3     0            3           15m
nginx-deployment   3/3     0            3           15m
nginx-deployment   3/3     0            3           15m
nginx-deployment   3/3     1            3           15m
nginx-deployment   4/3     1            4           15m
nginx-deployment   3/3     1            3           15m
nginx-deployment   3/3     2            3           15m
nginx-deployment   4/3     2            4           16m
nginx-deployment   3/3     2            3           16m
nginx-deployment   3/3     3            3           16m
nginx-deployment   4/3     3            4           16m
nginx-deployment   3/3     3            3           16m
```
一个监控`ReplicaSet`

```bash
# kubectl get rs --watch
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-848bcb569b   1         1         1       15m
nginx-deployment-69d9dddfd6   3         3         2       32s
nginx-deployment-69d9dddfd6   2         3         2       59s
nginx-deployment-69d9dddfd6   2         3         2       59s
nginx-deployment-69d9dddfd6   2         2         2       59s
nginx-deployment-848bcb569b   0         1         1       15m
nginx-deployment-848bcb569b   0         1         1       15m
nginx-deployment-848bcb569b   0         0         0       15m
nginx-deployment-69d9dddfd6   3         2         2       32s
nginx-deployment-69d9dddfd6   3         3         2       32s
nginx-deployment-69d9dddfd6   3         3         3       64s
```
See?  UP-TO-DATE正在变化！交替地逐一升级，正是滚动升级。

```bash
$ kubectl describe deployment nginx-deployment
...
Events:
  Type    Reason             Age                    From                   Message
  ----    ------             ----                   ----                   -------
  Normal  ScalingReplicaSet  21m                    deployment-controller  Scaled up replica set nginx-deployment-848bcb569b to 3 (原先的3个ReplicaSet 848bcb569b)
  Normal  ScalingReplicaSet  6m58s                  deployment-controller  Scaled up replica set nginx-deployment-69d9dddfd6 to 1 (新建了一个ReplicaSet 69d9dddfd6)
  Normal  ScalingReplicaSet  6m42s                  deployment-controller  Scaled down replica set nginx-deployment-848bcb569b to 2 (删除了一个ReplicaSet 848bcb569b)
  Normal  ScalingReplicaSet  6m42s                  deployment-controller  Scaled up replica set nginx-deployment-69d9dddfd6 to 2 (又新建了一个ReplicaSet 69d9dddfd6)
  Normal  ScalingReplicaSet  6m26s                  deployment-controller  Scaled down replica set nginx-deployment-848bcb569b to 1  (又删除了一个ReplicaSet 848bcb569b)
  Normal  ScalingReplicaSet  6m26s                  deployment-controller  Scaled up replica set nginx-deployment-69d9dddfd6 to 3  (又新建了一个ReplicaSet 69d9dddfd6)
```
而滚动升级默认会保证每次只更新25%，这个值可以按需设定，最好可以搭配前面的[探针](https://blog.madebug.net/ops/2022-02-10-how-to-keep-pods-alive)，严格制定滚动更新时存活标准和比例。

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1 #每次操作多少个pod，可用%
      maxUnavailable: 1 #每次删除多少pod，可用%
```
[drawio](7Tq_rxh9T4kY-ftxc-0ElrOOJPg9qzP1u9BNvku4NU8.svg)

### 更新后悔药
kubectl也提供了后悔药。

```bash
$ kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment


$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl create -f nginx-deployment.yaml --record
2           kubectl edit deployment/nginx-deployment
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91


$ kubectl rollout history deployment/nginx-deployment --revision=2
```
ReplicaSet 的个数可以通过`kubectl rollout pause deployment/nginx-deployment` 控制，pause状态下的deployment不会因多次edit操作创建新的ReplicaSet（有点git为了控制comment次数，把多个add合并到一个comment的感觉。）之后，可以通过`kubectl rollout resume deployment/nginx-deployment` 合并当前所有的edit为一次更新，但老实说，这有悖每次变更操作的最小化原则。介意的话，或者可以试试`spec.revisionHistoryLimit` ，但我更倾向于都不做～

## 0x03 总结
这篇难产了几天，终于啃完，也加深了理解，希望可以继续坚持。



