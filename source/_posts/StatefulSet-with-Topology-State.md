---
title: 深入剖析K8s筆記(StatefulSet之拓撲狀態)
mathjax: true
copyright: true
comment: true
date: 2022-03-07 00:36:32
categories: "Ops "
tags: 
- "Linux "
- "Kubernetes "
- "StatefulSet "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/statefulset&topology.png" width=50% /></center>

## 0x00 前言
Q：有Deployment不就够了吗？StatefulSet 又是什么？

<!-- more -->

A：Deployment: “无状态”应用√，主从 / 主备关系分布式应用x。

而StatefulSet专注“有状态”（Stateful Application）应用，其不外乎两类状态。

* 拓扑状态：应用的按序创建 / 启动。
* 存储状态：应用内多个实例，与不同存储的关系。

而过程就是：记录状态 ---> 重建pod ---> 恢复状态。

## 0x01 Headless Service
### What
Q：什么是 Service ？

A：先上[定义](https://kubernetes.io/zh/docs/concepts/services-networking/service/)：

> Service 是 Kubernetes 项目中用来将一组 Pod 暴露给外界访问的一种机制。比如，一个 Deployment 有 3 个 Pod，那么我就可以定义一个 Service。然后，用户只要能访问到这个 Service，它就能访问到某个具体的 Pod。

### How
Q：如何访问 ？

A：Two ways

* 通过 Service 的 VIP 
* 通过 Service 的 DNS 解析记录，有两种处理方法。
   * Normal Service直接访问域名获得其 VIP，例如：my-svc.my-namespace.svc.cluster.local 
   * Headless Service直接查询不同域名获得具体Pod IP。

举个🌰：

```bash
$ cat >> svc.yml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None # 不为其分配VIP
  selector:
    app: nginx
EOF

$ kubectl apply -f svc.yml && kubectl get service nginx 
NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   None         <none>        80/TCP    104s

$ cat >> statefulset.yml << EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
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
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web


$ kubectl apply -f statefulset.yml && kubectl get pods -w -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     0/1       Pending   0          0s
web-0     0/1       Pending   0         0s
web-0     0/1       ContainerCreating   0         0s
web-0     1/1       Running   0         19s
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         20s

$ kubectl get statefulset web
NAME   READY   AGE
web    2/2     9m18s

$ kubectl exec web-0 -- sh -c 'hostname'
web-0
$ kubectl exec web-1 -- sh -c 'hostname'
web-1

#尝试解释一下
$ kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh
/ # nslookup web-0.nginx
Server:    10.43.0.10
Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.42.3.239 web-0.nginx.default.svc.cluster.local
/ # nslookup web-1.nginx
Server:    10.43.0.10
Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.42.1.198 web-1.nginx.default.svc.cluster.local
```
### Why
Q：有何用？

A：StatefulSet通过原先的顺序（web-0，web-1），名字，重新创造出它们，连解析记录都保持（而非IP结果）。

```bash
$ kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted


$ kubectl get pod -w -l app=nginx
NAME    READY   STATUS        RESTARTS   AGE
web-1   0/1     Terminating   0          139m
web-0   1/1     Running       0          39s
web-0   1/1     Terminating   0          42s
web-0   0/1     Terminating   0          43s
web-0   0/1     Terminating   0          43s
web-0   0/1     Terminating   0          43s
web-0   0/1     Pending       0          0s
web-0   0/1     Pending       0          0s
web-0   0/1     ContainerCreating   0          0s
web-0   1/1     Running             0          1s
web-1   0/1     Terminating         0          140m
web-1   0/1     Terminating         0          140m
web-1   0/1     Pending             0          0s
web-1   0/1     Pending             0          0s
web-1   0/1     ContainerCreating   0          0s
web-1   1/1     Running             0          1s

# kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh
If you don't see a command prompt, try pressing enter.
/ #  nslookup web-0.nginx &&  nslookup web-1.nginx
Server:    10.43.0.10
Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.42.3.244 web-0.nginx.default.svc.cluster.local
Server:    10.43.0.10
Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.42.1.199 web-1.nginx.default.svc.cluster.local
```
## 0x02 总结
本文阐述了，在headless Service的场景里，通过StatefulSet的配合，完成对有状态应用的创建，删除，自恢复。这种记录的保存，正是StatefulSet保存有拓扑状态的办法。

