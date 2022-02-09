---
title: 深入剖析K8s筆記(特殊的Volume)
mathjax: true
copyright: true
comment: true
date: 2022-02-09 22:43:09
categories: "Ops "
tags:
- "Linux "
- "Kubernetes "
- "Pod "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/pod.png" width=50% /></center>

## 0x00 前言
先上定义，Projected Volume。译作“投射数据卷”

<!-- more -->

> 在 Kubernetes 中，有几种特殊的 Volume，其意义不是为了存放容器里的数据，也不是用来进行Container和宿主机之间的数据交换。这些特殊 Volume 的作用，是为容器提供预先定义好的数据。所以，从容器的角度来看，这些 Volume 里的信息就是仿佛是被 Kubernetes“投射”（Project）进入容器当中的。这正是 Projected Volume 的含义。

一共四种：

* Secret；
* ConfigMap；

* Downward API；
* ServiceAccountToken
好处：
* kubelet会通过etcd存储对该键值保持更新。（会有延迟
## 0x01 Secret
普通数据值以Secrect形式存入etcd，然后按需调用。

### 创建
```bash
方法一

$ cat ./username.txt
admin
$ cat ./password.txt
c1oudc0w!

$ kubectl create secret generic user --from-file=./username.txt
$ kubectl create secret generic pass --from-file=./password.txt

$ kubectl get secrets
NAME           TYPE                                DATA      AGE
user          Opaque                                1         51s
pass          Opaque                                1         51s

方法二

$ echo -n 'admin' | base64
YWRtaW4=
$ echo -n 'cloudc0w!' | base64
Y2xvdWRjMHch

$ cat secrect-create.yml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YWRtaW4=
  pass: Y2xvdWRjMHch
$ kubectl apply -f secrect-create.yml
```
### 调用
```bash
$ cat secrect-call.yml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume 
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass


$ kubectl exec -it test-projected-volume -- /bin/sh
$ ls /projected-volume/
username.txt
password.txt
$ cat /projected-volume/username.txt
root
$ cat /projected-volume/password.txt
cloudc0w!
```
## 0x02 ConfigMap
与Secret类似，ConfigMap存储的键值对在于无加密。

### 创建
```bash
方法一
$ cat configmap-create.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true  

$ kubectl apply -f configmap-create.yml

方法二
# .properties文件的内容
$ cat game.properties
enemy.types=aliens,monsters
player.maximum-lives=5
$ cat user-interface.properties 
color.good=purple
color.bad=yellow
allow.textmode=true

# 从.properties文件创建ConfigMap
$ kubectl create configmap game-demo --from-file=game.properties
$ kubectl create configmap game-demo --from-file=user-interface.properties 

# 以yaml模式查看这个ConfigMap里保存的信息(data)
$ kubectl get configmaps game-demo -o yaml
apiVersion: v1
data:
  game.properties: "enemy.types=aliens,monsters\nplayer.maximum-lives=5    \n"
  player_initial_lives: "3"
  ui_properties_file_name: user-interface.properties
  user-interface.properties: "color.good=purple\ncolor.bad=yellow\nallow.textmode=true
    \ \n"
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"game.properties":"enemy.types=aliens,monsters\nplayer.maximum-lives=5    \n","player_initial_lives":"3","ui_properties_file_name":"user-interface.properties","user-interface.properties":"color.good=purple\ncolor.bad=yellow\nallow.textmode=true  \n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"game-demo","namespace":"default"}}
  creationTimestamp: "2022-02-09T10:16:28Z"
  name: game-demo
  namespace: default
  resourceVersion: "4630695"
  uid: f8039b07-4c1c-47d4-aecf-09c2031b16c7
```
### 调用
```bash
$ cat configmap-create.yml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # You set volumes at the Pod level, then mount them into containers inside that Pod
    - name: config
      configMap:
        # Provide the name of the ConfigMap you want to mount.
        name: game-demo
        # An array of keys from the ConfigMap to create as files
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
[root@storage-k3sm configMap]# kubectl exec -it configmap-demo-pod -- /bin/sh
/ # cd /config/
/config # ls
game.properties            user-interface.properties
/config # cat game.properties 
enemy.types=aliens,monsters
player.maximum-lives=5    
/config # cat user-interface.properties 
color.good=purple
color.bad=yellow
allow.textmode=true 
```
## 0x03 Downward API
直接让Pod内获取到对象本身的信息。

### 创建+调用
```bash
$ cat dapi-create.yml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
  annotations:
    build: two
    builder: john-doe
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          if [[ -e /etc/podinfo/annotations ]]; then
            echo -en '\n\n'; cat /etc/podinfo/annotations; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI: #声明了前面要暴露给Pod的metadata.label信息给容器的/etc/podinfo/label
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations

$ kubectl apply -f dapi-create.yaml

$ kubectl logs kubernetes-downwardapi-volume-example
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"

build="two"
builder="john-doe"
```
除了metadata.labels 还有以下字段样例，最新官网自查。

```Plain Text
1. 使用fieldRef可以声明使用:
spec.nodeName - 宿主机名字
status.hostIP - 宿主机IP
metadata.name - Pod的名字
metadata.namespace - Pod的Namespace
status.podIP - Pod的IP
spec.serviceAccountName - Pod的Service Account的名字
metadata.uid - Pod的UID
metadata.labels['<KEY>'] - 指定<KEY>的Label值
metadata.annotations['<KEY>'] - 指定<KEY>的Annotation值
metadata.labels - Pod的所有Label
metadata.annotations - Pod的所有Annotation

2. 使用resourceFieldRef可以声明使用:
容器的CPU limit
容器的CPU request
容器的memory limit
容器的memory request
```
[官网对其作用说得很好。](https://kubernetes.io/zh/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)

> 对于容器来说，有时候拥有自己的信息是很有用的，可避免与 Kubernetes 过度耦合。 Downward API 使得容器使用自己或者集群的信息，而不必通过 Kubernetes 客户端或 API 服务器来获得。

> 一个例子是有一个现有的应用假定要用一个非常熟悉的环境变量来保存一个唯一标识。 一种可能是给应用增加处理层，但这样是冗余和易出错的，而且它违反了低耦合的目标。 更好的选择是使用 Pod 名称作为标识，把 Pod 名称注入这个环境变量中。

## 0x04 ServiceAccountToken
其实是Secret的变种，专门用于K8s系统内置的服务账户凭证。

## 调用(InClusterConfig
默认情况下会把相应的密钥信息挂载进容器里

```bash
# kubectl describe pod two-containers 
...
Containers:
  nginx-container:
    ...
    Mounts:
      /usr/share/nginx/html from shared-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fwlgc (ro)
  debian-container:
    ...
    Mounts:
      /pod-data from shared-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fwlgc (ro)
...
Volumes:
  ...
  kube-api-access-fwlgc:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true

# kubectl exec -it two-containers   -- /bin/sh                                                  
# cd /var/run/secrets/kubernetes.io
# ls
serviceaccount
# cd serviceaccount
# ls
ca.crt  namespace  token
# exit
```
## 0x05 总结
简单总结了几种Pod的Projected Volume的特点，适用于不同需求，与直接使用env传递最大的区别在于可以自动更新维护。做到了避免过度耦合和集群管理的平衡。
