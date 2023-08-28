---
title: 使用kube-vip+zerotier+k3s搭建去中心化异地高可用的集群
mathjax: true
copyright: true
comment: true
date: 2022-12-03 19:52:12
updated: 2022-12-03 19:52:12
categories:
- "Ops "
tags:
- "Zerotier "
- "K3s "
- "Kube-vip "
- "Kubernetes "
- "HomeLab "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221203203605.png" width=50% /></center>



## 0x00️⃣ 前言 

本文记述了我谋划已久的去中心化异地高可用k3s单集群搭建，[以前试过用K8s搭建](https://blog.madebug.net/ops/2020-06-21-kubernetes-dislikes-zerotier)，那是一个单master的集群。

<!-- more -->

是的，你没听错。在zerotier打通网络之上，使用kube-vip在zerotier网卡广播virtual ip，从而构建起一套异地容灾k3s多master单集群，之后买再多的vps，都可以直接作为worker加入了。后面再对接备份，存储，爽歪歪。

- master高可用
- worker随时加入
- k3s应用生态丰富
- p2p组网，完全的去中心化
- worker随时可以离开

## 0x01️⃣ 機器情況



---

| 供應商 |  角色  |  規格   | 機房位置 |
| :----: | :----: | :-----: | :------: |
| rack01 | master |  6C32G  |  某地A   |
| rack02 | master | 20C128G |  某地B   |
| rack03 | master | 24C128G |  某地C   |
| rack04 | worker | 32C256G |  某地C   |
|  vps   | worker |  1C1G   |  某地D   |
|  ...   |  ...   |   ...   |   ...    |

## 0x02️⃣ zerotier部分

---

### 加入ZeroTier組網

### 自建zerotier planet

~~~
# git clone https://github.com/m4d3bug/docker-zerotier-planet
# cd docker-zerotier-planet
# chmod +x ./deploy.sh
# ./deploy.sh
~~~

#### 安裝軟體

```bash
# curl -s https://install.zerotier.com/ | sudo bash
```

#### 加入網絡

```bash
# zerotier-cli join xxxxxxxxxxxxxxxx
```

## 0x03️⃣ k3s部分

