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

## *ZeroTier* 部分

---

### 加入*ZeroTier* 組網

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

#### 寫入靜態*IP* 和*hostname*

```bash
# cat >> /etc/hosts << EOF
10.9.8.118 master.m4d3bug.com master 
10.9.8.65  node1.m4d3bug.com  node1
10.9.8.129 node2.m4d3bug.com  node2
EOF
# hostnamectl set-hostname xxx.m4d3bug.com
```

## *K8S* 部分

---

### 系統預設定

#### 確保*selinux* 為寬容模式

```bash
# setenforce 0
setenforce: SELinux is disabled
```

#### 關閉*firewalld*

云供應商們基本都關掉了，所以沒什麽回顯。

```bash
# systemctl disable firewalld
# systemctl stop firewalld
```

#### 选择性關閉*swap*

在*master* 節點以外操作。

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

#### 安裝*Docker* 軟體

```bash
# yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
# yum install docker-ce docker-ce-cli containerd.io
# docker -v 
Docker version 19.03.12, build 48a66213fe
# systemctl start docker
# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

#### 加入代理設定到Docker中

順便説一嘴，可以在*ZeroTier* 組網裏起一個代理。

```bash
# mkdir /usr/lib/systemd/system/docker.service.d
# cat >> /usr/lib/systemd/system/docker.service.d/http-proxy.conf << EOF
[Service]
Environment="HTTP_PROXY=http://10.9.8.10:1081/"
Environment="HTTPS_PROXY=http://10.9.8.10:1081/"
Environment="NO_PROXY=localhost,localhost.localdomain,localhost4,localhost4.localdomain4,10.0.0.0/8"
EOF
# cat > /etc/docker/daemon.json <<EOF
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
    "max-size": "100m"
    },
    "storage-driver": "overlay2"
}
EOF
# systemctl daemon-reload
# systemctl restart docker
```

#### 加入谷歌倉庫

同樣加入*ZeroTier* 中的代理地址。

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
#  rpm -qa |grep kube*
kubectl-1.18.8-0.x86_64
kubernetes-cni-0.8.6-0.x86_64
kubeadm-1.18.8-0.x86_64
kubelet-1.18.8-0.x86_64
# systemctl enable kubelet
# kubeadm config images pull
```

*master* 節點只是一隻小鷄鷄，所以就不關它的*swap* 了。

```nohighlight
# vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=--fail-swap-on=false
```

#### 安裝集群

在v1.8.0之後的版本，kubeadm提供了一種[分階段的構建方式](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/)，構建*etcd* 是其中的一個*phase* ，在啓動前我們需要對其中的參數進行修改。

- 定制該版本的*kubeadm-config.yml*

  ```bash
  # kubeadm config print init-defaults  > kubeadm-config.yaml
  # vim kubeadm-config.yaml
  apiVersion: kubeadm.k8s.io/v1beta2
  bootstrapTokens:
  - groups:
    - system:bootstrappers:kubeadm:default-node-token
    token: abcdef.0123456789abcdef
    ttl: 24h0m0s
    usages:
    - signing
    - authentication
  kind: InitConfiguration
  localAPIEndpoint:
    advertiseAddress: 10.9.8.118             <---網卡ip
    bindPort: 6443
  nodeRegistration:
    criSocket: /var/run/dockershim.sock
    name: master.m4d3bug.com
    taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
  ---
  apiServer:
    timeoutForControlPlane: 4m0s
  apiVersion: kubeadm.k8s.io/v1beta2
  certificatesDir: /etc/kubernetes/pki
  clusterName: kubernetes
  controllerManager: {}
  dns:
    type: CoreDNS
  etcd:
    local:
      dataDir: /var/lib/etcd
  imageRepository: k8s.gcr.io
  kind: ClusterConfiguration
  kubernetesVersion: v1.18.0
  networking:
    dnsDomain: cluster.local
    serviceSubnet: 10.96.0.0/12
    podSubnet: 10.244.0.0/16                 <---pod子網範圍
  scheduler: {}
  ```

-  *preflight* 階段

  ```bash
  # kubeadm init phase preflight --config kubeadm-config.yaml --ignore-preflight-errors=NumCPU --ignore-preflight-errors=Swap
  ```

-  *kubelet-start*  階段

  ```bash
  # kubeadm init phase kubelet-start --config kubeadm-config.yaml
  ```

-  *kubeconfig*  階段

  ```bash
  # kubeadm init phase kubeconfig all --config kubeadm-config.yaml
  ```

-  *control-plane*  階段

  ```bash
  # kubeadm init phase control-plane all --config kubeadm-config.yaml
  ```

-  etcd 階段 

  ```bash
  # kubeadm init phase etcd local --config kubeadm-config.yaml
  # vim /etc/kubernetes/manifests/etcd.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    annotations:
      kubeadm.kubernetes.io/etcd.advertise-client-urls: https://10.9.8.118:2379
    creationTimestamp: null
    labels:
      component: etcd
      tier: control-plane
    name: etcd
    namespace: kube-system
  spec:
    containers:
    - command:
      - etcd
      - --advertise-client-urls=https://10.9.8.118:2379
      - --cert-file=/etc/kubernetes/pki/etcd/server.crt
      - --client-cert-auth=true
      - --data-dir=/var/lib/etcd
      - --initial-advertise-peer-urls=https://10.9.8.118:2380
      - --initial-cluster=master.m4d3bug.com=https://10.9.8.118:2380
      - --key-file=/etc/kubernetes/pki/etcd/server.key
      - --listen-client-urls=https://0.0.0.0:2379        <--- 改爲0.0.0.0
      - --listen-metrics-urls=http://127.0.0.1:2381
      - --listen-peer-urls=https://0.0.0.0:2380          <--- 改爲0.0.0.0
      - --name=master.m4d3bug.com
      - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
      - --peer-client-cert-auth=true
      - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
      - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
      - --snapshot-count=10000
      - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
      image: k8s.gcr.io/etcd:3.4.3-0
      imagePullPolicy: IfNotPresent
      livenessProbe:
        failureThreshold: 8
        httpGet:
          host: 127.0.0.1
          path: /health
          port: 2381
          scheme: HTTP
        initialDelaySeconds: 15
        timeoutSeconds: 15
      name: etcd
      resources: {}
      volumeMounts:
      - mountPath: /var/lib/etcd
        name: etcd-data
      - mountPath: /etc/kubernetes/pki/etcd
        name: etcd-certs
    hostNetwork: true
    priorityClassName: system-cluster-critical
    volumes:
    - hostPath:
        path: /etc/kubernetes/pki/etcd
        type: DirectoryOrCreate
      name: etcd-certs
    - hostPath:
        path: /var/lib/etcd
        type: DirectoryOrCreate
      name: etcd-data
  status: {}
  
  # kubectl apply -f /etc/kubernetes/manifests/etcd.yaml
  # kubeadm init --skip-phases=preflight,certs,kubeconfig,kubelet-start,control-plane,etcd --config kubeadm-config.yaml
  ```

- 之後就如下：

  ```bash
  Your Kubernetes control-plane has initialized successfully!
  
  To start using your cluster, you need to run the following as a regular user:
  
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  You should now deploy a pod network to the cluster.
  Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/
  
  Then you can join any number of worker nodes by running the following on each as root:
  
  kubeadm join 10.9.8.118:6443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:f14e90eda52b285b41ddb5d34a4dcf21f55ed66831015c4ca1a996cf17754143 
  ```

- 部署*flannel* 

  ```bash
  # kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  ```

### 排障

```bash
查看pod日志
# kubectl --namespace kube-system logs kube-controller-manager-master.m4d3bug.com
查看pod的過程
# kubectl describe pod kube-controller-manager-master.m4d3bug.com --namespace=kube-system
打印加入的命令
# kubeadm token create --print-join-command
重置集群
# kubeadm reset
# ifconfig cni0 down
# ip link delete cni0
# ifconfig flannel.1 down
# ip link delete flannel.1
# rm -rf /var/lib/cni/
# rm -rf /var/lib/etcd
# rm -rf /etc/cni/net.d
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



