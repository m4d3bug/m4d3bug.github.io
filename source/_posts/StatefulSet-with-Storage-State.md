---
title: 深入剖析K8s筆記(StatefulSet之存儲狀態)
mathjax: true
copyright: true
comment: true
date: 2022-03-17 23:56:36
categories: "Ops "
tags: 
- "Linux "
- "Kubernetes "
- "StatefulSet "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/pvc&pv.png" width=50% /></center>

## 0x00 前言

书接前文，StatefulSet 专注处理应用的两类状态。

<!-- more -->
* 拓扑状态
* 存储状态

## 0x01 什么是PVC 与 PV
Q：意义何在？

A：1. 抽象了底层的存储类型（Ceph，GlusterFS[等](https://blog.madebug.net/ops/2022-01-31-why-we-need-pods#:~:text=path%3A%20%22%2Ftmp%22%20%23%E6%98%BE,%E5%A4%9A%E5%89%AF%E6%9C%AC%E5%AD%98%E6%B4%BB)）保证了信息的隔离。2. 降低用户生命和使用持久化存储 Volume 的门槛。

### 举两个例子：
1. Ceph RBD创建调用一条龙

```bash
$ cat >> rbd.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: rbd
spec:
  containers:
    - image: kubernetes/pause
      name: rbd-rw
      volumeMounts:
      - name: rbdpd
        mountPath: /mnt/rbd
  volumes:
    - name: rbdpd
      rbd:
        monitors:
        - '10.16.154.78:6789'
        - '10.16.154.82:6789'
        - '10.16.154.83:6789'
        pool: kube
        image: foo
        fsType: ext4
        readOnly: true
        user: admin
        keyring: /etc/ceph/keyring
        imageformat: "2"
        imagefeatures: "layering"
EOF
```
2. Persistent Volume Claim 创建

```bash
$ cat >> pvc.yml << EOF
#声明pv
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
# 单节点可读写模式 https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes
  - ReadWriteOnce 
  resources:
    requests:
      storage: 1Gi   # 请求1Gi大小的资源
EOF

$ cat >> pv-pod.yml << EOF
#声明pod
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-storage
  volumes:
    - name: pv-storage
      persistentVolumeClaim:
        claimName: pv-claim # 调用上面的pvc
EOF

$ kubectl apply -f pvc.yml
$ kubectl apply -f pv-pod.yml
$ kubectl get pvc pv-claim
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pv-claim   Bound    pvc-62e18a42-c9d0-42e1-bc87-543de2e88b79   1Gi        RWO            local-path     61m
$ kubectl edit pv pvc-62e18a42-c9d0-42e1-bc87-543de2e88b79
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: rancher.io/local-path
  creationTimestamp: "2022-03-15T13:40:49Z"
  finalizers:
  - kubernetes.io/pv-protection
  name: pvc-62e18a42-c9d0-42e1-bc87-543de2e88b79
  resourceVersion: "10135197"
  uid: a63f0944-c480-4295-8419-6b4f3ee76ab0
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: pv-claim
    namespace: default
    resourceVersion: "10135178"
    uid: 62e18a42-c9d0-42e1-bc87-543de2e88b79
  hostPath:
    path: /var/lib/rancher/k3s/storage/pvc-62e18a42-c9d0-42e1-bc87-543de2e88b79_default_pv-claim
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - storage-k3sw3.m4d3bug.com
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-path
  volumeMode: Filesystem
status:
  phase: Bound
```
可以看到，定义pvc和pod，关联的pv就会被一同创建，pvc关注的是上层pod调用的一些读写模式，大小。而pv关注的是下层与宿主机的联通定义。pvc更像是给开发用的一个接口，而pv是给运维针对pvc的具体实现。

大概就是这么一个感觉，运维<--- PV PVC ---> 开发。

## 0x02 如何服务 StatefulSet
### 准备
还是之前的[例子](https://blog.madebug.net/ops/2022-03-07-statefulset-with-topology-state#:~:text=%E5%85%B7%E4%BD%93Pod%20IP%E3%80%82-,%E4%B8%BE%E4%B8%AA%F0%9F%8C%B0%EF%BC%9A,default.svc.cluster.local,-Why)，稍作变更。

```bash
# 新建一个statefulset带pvc的
$ cat >> statefulset-pvc.yml << EOF
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
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  # 定义声明一个pvc
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
EOF

# 看一下新建结果
# kubectl get pvc -l app=nginx
NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0          Bound    pvc-84f2721a-2a20-4a31-9bae-8339352488d0   1Gi        RWO            local-path     22m
www-web-1          Bound    pvc-f0a00b36-f728-40b9-b5f6-209db2e10fce   1Gi        RWO            local-path     6m44s

# pv也跟着创建出来了
# kubectl get pv
NAME                                      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM STORAGECLASS                           REASON                   AGE
pvc-84f2721a-2a20-4a31-9bae-8339352488d0   1Gi        RWO            Delete           Bound    default/www-web-0                            local-path              6m16s
pvc-f0a00b36-f728-40b9-b5f6-209db2e10fce   1Gi        RWO            Delete           Bound    default/www-web-1                            local-path              6m13s

# 检测一下
# for i in 0 1; do kubectl exec web-$i -- sh -c 'echo hello $(hostname) > /usr/share/nginx/html/index.html'; done
# for i in 0 1; do kubectl exec -it web-$i -- apt update -y ;done
# for i in 0 1; do kubectl exec -it web-$i -- apt install curl -y; done
# for i in 0 1; do kubectl exec -it web-$i -- curl http://localhost; done
hello web-0
hello web-1
```
### 删除测试自重建
```bash
# kubectl delete pod -l app=nginx --force
warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "web-0" force deleted
pod "web-1" force deleted
# for i in 0 1; do kubectl exec -it web-$i -- apt update -y ;done
# for i in 0 1; do kubectl exec -it web-$i -- apt install curl -y; done
# for i in 0 1; do kubectl exec -it web-$i -- curl http://localhost; done
hello web-0
hello web-1
```
## 0x03 StatefulSet总结
* StatefulSet只管Pod，和deployment的编号形式不同，会更短小精简一些。

```bash
$ for i in  `kubectl get pod -l app=nginx |awk  '{print $1}' |grep -v NAME`; do kubectl exec -it $i -- hostname; done
web-0
web-1
nginx-deployment-848bcb569b-wscf6
nginx-deployment-848bcb569b-75fb9
nginx-deployment-848bcb569b-s8wq9
```
* headless service保证了解析记录的更新，维护了StatefulSet的拓扑状态。
* PVC作为永久存储的一种方式，维护了StatefulSet的存储状态。

