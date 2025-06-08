---
title: 深入剖析K8s筆記(Controller模型)
mathjax: true
copyright: true
comment: true
date: 2022-02-19 23:03:58
categories: "Ops "
tags:
- "Linux "
- "Kubernetes "
- "Controller "
---

<center><img src="https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/operator-controller-reconciliation.jpeg" width=50% /></center>

## 0x00 前言
Q：何为Controller？

<!--more -->

A：把K8s想象成大吊臂，Container是一个个集装箱，Pod是对Container的吊环。Controller就是大吊臂的各种工作逻辑。例如：

```bash
它们都归kube-controller-manager管

$ cd kubernetes/pkg/controller/
$ ls -d */              
deployment/             job/                    podautoscaler/          
cloud/                  disruption/             namespace/              
replicaset/             serviceaccount/         volume/
cronjob/                garbagecollector/       nodelifecycle/          replication/            statefulset/            daemon/
...
```
## 0x01 Controller工作原理
```bash
for {
  实际状态 := 获取集群中对象X的实际状态（Actual State） <---kubelet汇报 / etcd / Controller自己收集
  期望状态 := 获取集群中对象X的期望状态（Desired State）<---Yaml文件的定义 / 
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
```
* Control loop: 整个Controller的工作过程其实分为三步。用Deployment举例
   * 获取实际状态，读etcd然后统计其个数。
   * 获取期望状态，关注Replicas字段。
   * 调谐（Reconcile），循环（Reconcile Loop / Sync Loop）确定编排动作（创建 / 删除）。

上述过程，充分体现“声明式”，“面向API对象编程”。

## 0x02 Controller的对象
Controller自身是个对象，但是对象也有对象。

画个楚河汉界。一图胜千言。

```bash
apiVersion: apps/v1                   
kind: Deployment                        
metadata:                               
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2           ^^^ Controller的定义（期望状态
----------------------------自制分割线用于界限-----------------------------
  template:             vvv Controller的对象（被控制对象模版PodTemplate
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
## 0x03 总结
简单介绍了Controller，它的概念，工作原理，还有它的对象。

