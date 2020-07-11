---
title: 基於Zerotier搭建NFS的TPOT蜜罐嘗試
mathjax: true
copyright: true
comment: true
date: 2020-07-11 18:23:31
updated: 2020-07-11 18:54:34
categories:
- "Sec "
tags:
- "Linux "
- "NFS "
- "TPOT "
- "Zerotier "
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/tpotsocial.png" width=50% /></center>

本文將通過[Zerotier](https://blog.madebug.net/ops/2020-06-15-use-zerotier-to-build-a-intranet-in-china)嘗試搭建nfs來給~~<font color=#808080>因爲沒錢</font>~~硬盤空間不足的雲服務器提供後端存儲以滿足TPOT的安裝需求。

<!-- more -->

## Zerotier部分

---

[略](https://blog.madebug.net/ops/2020-06-15-use-zerotier-to-build-a-intranet-in-china)

## NFS部分

---

### 服務端

```bash
~]# yum -y install nfs-utils
~]# vi /etc/idmapd.conf

# line 5: uncomment and change to your domain name
Domain = madebug.net

~]# vi /etc/exports
# write settings for NFS exports
/home 10.9.8.167/32(rw,no_root_squash)   <--- 雲服務器的zerotier ip，一碌柒

~]# systemctl start rpcbind nfs-server
~]# systemctl enable rpcbind nfs-server
~]# firewall-cmd --add-service=nfs --permanent
~]# firewall-cmd --reload
```

### 客戶端

```bash
~# apt -y install nfs-common
~# vi /etc/hosts

# last line: add local storage server ip
10.9.8.166 storage1.madebug.net

~# vi /etc/idmapd.conf

# line 6: uncomment and change to your domain name
Domain = madebug.net

~# mount -t nfs storage1.madebug.net:/home /data
~# df -TH
Filesystem                 Type      Size  Used Avail Use% Mounted on
udev                       devtmpfs  4.1G     0  4.1G   0% /dev
tmpfs                      tmpfs     824M  8.9M  815M   2% /run
/dev/vda1                  ext4       53G  2.5G   49G   5% /
tmpfs                      tmpfs     4.2G   25k  4.2G   1% /dev/shm
tmpfs                      tmpfs     5.3M     0  5.3M   0% /run/lock
tmpfs                      tmpfs     4.2G     0  4.2G   0% /sys/fs/cgroup
tmpfs                      tmpfs     824M     0  824M   0% /run/user/0
storage1.madebug.net:/home nfs4      169G   35M  169G   1% /data
~# vi /etc/fstab

# add to the end like follows
storage1.madebug.net:/home   /data  nfs     defaults        0       0

~# mount -a
```

## TPOT部分

---

設置好所需的代理。~~<font color=#808080>不知道怎麽設置的可以等我以後水一篇再補上。</font>~~

```bash
~# grep -vE '^#|^$' /etc/apt/sources.list
deb http://mirrors.tuna.tsinghua.edu.cn/debian/ buster main
deb-src http://mirrors.tuna.tsinghua.edu.cn/debian/ buster main        
deb http://security.debian.org/debian-security buster/updates main     
deb-src http://security.debian.org/debian-security buster/updates main 
deb http://mirrors.tuna.tsinghua.edu.cn/debian/ buster-updates main    
deb-src http://mirrors.tuna.tsinghua.edu.cn/debian/ buster-updates main
~# apt update
~# apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
~# curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
~# add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
~# apt update
~# apt install -y docker-ce docker-ce-cli containerd.io python3-pip zlib*
~# pip3 install docker-compose
~# pip3 install elasticsearch-curator
~# git clone https://github.com/dtag-dev-sec/tpotce
~# cd tpotce/iso/installer/
~# ./install.sh --type=user

# 重啓后來啓動所有容器
~# docker-compose -f ./standard.yml up -d
```

訪問  https://IP:64297/

## 結語

Zerotier組網天下第一。

以及鳴謝以下鏈接：

- [https://www.cnblogs.com/gucb/p/12612542.html](https://www.cnblogs.com/gucb/p/12612542.html)
- [https://www.freebuf.com/sectool/134504.html](https://www.freebuf.com/sectool/134504.html)
- [https://www.freebuf.com/sectool/178998.html](https://www.freebuf.com/sectool/178998.html)
