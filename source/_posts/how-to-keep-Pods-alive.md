---
title: 深入剖析K8s筆記(如何保證的Pod的健康)
mathjax: true
copyright: true
comment: true
date: 2022-02-10 20:20:36
categories: "Ops "
tags:
- "Linux "
- "Kubernetes "
- "Pod "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/pod.png" width=50% /></center>

## 0x00 前言

Q: `healthy check`除了监控`Pod/Container`状态外，有没有更细致，监控`Container`内服务状态的方法呢？

<!-- more -->

A: 有，可以为`Container`定义一个“探针” `Probe`，`kubelet`就可以根据这个`Probe`的返回值断定`Container`的状态，而不只以`Container`运行为标准。而且探针有三种：

* `livenessProbe`存活探针
* `readinessProbe`就绪探针
* `startupProbe`启动探针

## 0x01 健康检查-探针
### livenessProbe存活探针
```bash
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    #30秒自动删除/tmp/healthy探针
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    #使用livenessProbe定义检查命令
    livenessProbe: 
      exec:
        command:
        - cat
        - /tmp/healthy
      #启动后5秒开始检查
      initialDelaySeconds: 5
      #5秒一次
      periodSeconds: 5
```
观察一下，`kubelet`的恢复机制（pod.spec.restartPolicy）在不停地拉起保活。

```bash
$ kubectl get pod test-liveness-exec -o wide --watch                                                                                 
NAME                 READY   STATUS             RESTARTS       AGE   IP            NODE                        NOMINATED NODE   READINESS GATES
test-liveness-exec   0/1     CrashLoopBackOff   13 (80s ago)   35m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   1/1     Running            14 (5m7s ago)   39m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   1/1     Running            15 (<invalid> ago)   40m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   0/1     CrashLoopBackOff   15 (<invalid> ago)   42m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   1/1     Running            16 (5m1s ago)        47m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   1/1     Running            17 (<invalid> ago)   48m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   0/1     CrashLoopBackOff   17 (<invalid> ago)   49m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   1/1     Running            18 (5m ago)          54m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   1/1     Running            19 (0s ago)          56m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   0/1     CrashLoopBackOff   19 (<invalid> ago)   57m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   1/1     Running            20 (4m57s ago)       62m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   1/1     Running            21 (<invalid> ago)   63m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>
test-liveness-exec   0/1     CrashLoopBackOff   21 (<invalid> ago)   64m   10.42.2.178   storage-k3sw2.m4d3bug.com   <none>           <none>

$ kubectl describe pod test-liveness-exec
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  16m                   default-scheduler  Successfully assigned default/test-liveness-exec to storage-k3sw2.m4d3bug.com
  Normal   Pulled     16m                   kubelet            Successfully pulled image "busybox" in 2.340333537s
  Normal   Pulled     15m                   kubelet            Successfully pulled image "busybox" in 2.281807839s
  Normal   Pulled     14m                   kubelet            Successfully pulled image "busybox" in 2.366382447s
  Normal   Created    14m (x3 over 16m)     kubelet            Created container liveness
  Normal   Started    14m (x3 over 16m)     kubelet            Started container liveness
  Warning  Unhealthy  13m (x9 over 16m)     kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  Normal   Killing    13m (x3 over 15m)     kubelet            Container liveness failed liveness probe, will be restarted
```
这就是K8s的Pod的恢复机制，但为什么总是在同一节点恢复呢？这就是`kind: Pod`和`kind: Deployment`的区别。

## 0x02 恢复机制-pod.spec.restartPolicy
谈到恢复机制，pod.spec.restartPolicy的状态有以下三种。
* Always：在任何情况下，只要Container不在运行状态，就自动重启Container；
* OnFailure: 只在Container异常时才自动重启Container；
* Never: 从来不重启Container。
另外需要注意：
* Container被重新自动创建，内容可能丢失。
* 默认的恢复策略就是Always。
* Pod内所有的容器都异常才会显示为Failed。
* [更多](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
## 0x03 多一重保险-预设字段
Q: 那有没有办法可以从运维的角度去预设这些健康检查的字段呢？

A: 有，PodPresent

```bash
$ kubectl apply -f preset.yaml 
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```
先后apply一下对象

```bash
$ kubectl apply -f pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
spec:
  containers:
    - name: website
      image: nginx
      ports:
        - containerPort: 80
```
观察

```bash
$ kubectl get pod website -o yaml
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
  annotations:
    podpreset.admission.kubernetes.io/podpreset-allow-database: "resource version"
spec:
  containers:
    - name: website
      image: nginx
      volumeMounts:
        - mountPath: /cache
          name: cache-volume
      ports:
        - containerPort: 80
      env:
        - name: DB_PORT
          value: "6379"
  volumes:
    - name: cache-volume
      emptyDir: {}
```
> PodPreset 里定义的内容，只会在 Pod API 对象被创建之前追加在这个对象本身上，而不会影响任何 Pod 的控制器的定义。而多个PodPreset冲突以外的字段都将被合并。

## 0x04 总结
从健康检查，到探针，再到恢复（保活）机制。最后到预设字段PodPreset，填了不少坑也埋了不少坑，未完待续。
