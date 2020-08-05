---
title: 基於Zerotier搭建"跨供應商"的K8S集群带阻嘗試
mathjax: true
copyright: true
comment: true
date: 2020-06-21 00:02:51
updated: 2020-07-11 18:33:33
categories:
- "Ops "
tags:
- "Linux "
- "Network "
- "UDP "
- "P2P "
- "Zerotier "
- "Kubernetes "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/kubernetes%26zerotier.png" width=50% /></center>

本文旨在嘗試驗證自己的一個~~<font color=#808080>奇葩</font>~~想法。

<!-- more -->

## 機器情況

---

| 供應商 | 規格 | 機房位置   |
| ------ | ---- | ---------- |
| 騰訊雲 | 2C8G | 中國上海   |
| 搬瓦工 | 2C2G | 美國洛杉磯 |
| Ucloud | 1C1G | 中國香港   |

## ZeroTier部分

---

### 加入ZeroTier組網

#### 安裝軟體

```bash
# curl -s https://install.zerotier.com/ | sudo bash
```

#### 加入網絡

```bash
# zerotier-cli join xxxxxxxxxxxxxxxx
```

#### 機器網絡狀況

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/machinesNetworks.png](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/machinesNetworks.png)

#### 寫入靜態IP和hostname

```bash
# cat >> /etc/hosts << EOF
10.9.8.118 master.m4d3bug.com master 
10.9.8.65  node1.m4d3bug.com  node1
10.9.8.129 node2.m4d3bug.com  node2
EOF
# hostnamectl set-hostname xxx.m4d3bug.com
```

## K8S部分

---

### 系統預設定

#### 確保selinux為寬容模式

```bash
# setenforce 0
setenforce: SELinux is disabled
```

#### 關閉firewalld

云供應商們基本都關掉了，所以沒什麽回顯。

```bash
# systemctl disable firewalld
# systemctl stop firewalld
```

#### 选择性關閉swap

在master節點以外操作

```bash
# swapoff -a
# vi /etc/fstab
...
# disable swap line
#/swap none swap sw 0 0
# mount -a
```

#### 設置并啓用内核參數

```bash
# cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# modprobe br_netfilter
# sysctl -p /etc/sysctl.d/k8s.conf
```

### 開始安裝

#### 安裝Docker軟體

```bash
# yum -y install docker
# docker -v 
Docker version 1.13.1, build 64e9980/1.13.1
# systemctl start docker
# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

#### 加入代理設定到Docker中

順便説一嘴，可以在ZeroTier組網裏起一個代理。

```bash
# mkdir /usr/lib/systemd/system/docker.service.d
# cat >> /usr/lib/systemd/system/docker.service.d/http-proxy.conf << EOF
[Service]
Environment="HTTP_PROXY=http://10.9.8.10:1081/"
Environment="HTTPS_PROXY=http://10.9.8.10:1081/"
Environment="NO_PROXY=localhost,localhost.localdomain,localhost4,localhost4.localdomain4,10.0.0.0/8"
EOF
# systemctl daemon-reload
# systemctl restart docker
```

#### 加入谷歌倉庫

同樣加入ZeroTier中的代理地址。

```bash
# cat <<'EOF' > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
proxy=http://10.9.8.10:1081
EOF
```

#### 獲得必須的軟體及鏡像

```bash
# yum -y install kubeadm kubelet kubectl
# rpm -qa |grep kube*
kubeadm-1.18.4-0.x86_64
kubectl-1.18.4-0.x86_64
kubelet-1.18.4-0.x86_64
# systemctl enable kubelet
# kubeadm config images pull
```

master節點只是一隻小鷄鷄，所以就不關它的swap了。

```nohighlight
# vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=--fail-swap-on=false
```

#### 附加設置

鑒於這是node為最小單位的部署方法，爲了dns可控，每個機器加裝dnsmasq并且指定本機解析。

```bash
# yum install -y dnsmasq
# vim /etc/dnsmasq.conf
strict-order    #在解析域名地址时严格按照 DNS 服务器列表的从上到下的顺序去解析
# vim /etc/resolv.conf
search local
....
# systemctl enable dnsmasq.service 
# systemctl start dnsmasq.service
```

#### 安裝集群

注意master節點的ZeroTier IP

```bash
# kubeadm init --apiserver-advertise-address=10.9.8.118 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU --ignore-preflight-errors=Swap 
```

## 結語

---

一套下來，UDP的通信可靠性還是名不虛傳，除非等待HTTP3.0/quic協議普及吧，這樣子運營商也許就不會對UDP那麽狠了，所以奉勸各位還是別折騰這條路了，後面或許會嘗試使用[GRE方式](https://feisky.xyz/posts/2015-03-02-setting-up-gre-for-kubernetes/)來再嘗試一次。以下是部署后情況：

可以見到，即使加入成功也都是充斥著大量因爲timeout造成的failed的信息在其中。

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/failedzerotier.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/failedzerotier.png)

其後，通過睡了一覺，白天時分，QOS緩和的時候，順利將剩下搬瓦工節點加入。

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/k8s-status.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/k8s-status.png)

但也證明，SDN跨運營商，以node為最小單位組建K8S集群是可行的，但是需要money💰。因此不難理解爲什麽現在混合雲架構都是傾向于以一個帶master節點集群為最小單位組建集群。~<font color=#808080>或許可以試試每個節點都是單master的去污點化部署。</font>~

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/k8scurl.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/k8scurl.png)



