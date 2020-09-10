---
title: 于PXE下的iPXE+Kickstart搭建無人值守自動化安裝環境
mathjax: true
copyright: true
comment: true
date: 2020-08-08 21:31:13
categories:
- "Ops "
tags:
- "Kickstart "
- "DHCP "
- "TFTP "
- "iPXE "
- "Httpd "
- "RHEL 7 "
- "Linux "
- "BIOS "
- "UEFI "
- "Anaconda "
- "Dnsmasq "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/iPXE.png" width=50% /></center>

本文將簡單搭建一個基於PXE網卡分發[iPXE](https://ipxe.org/)使用Kickstart自動化安裝BIOS或UEFI架構的自動化安裝環境。其中TFTP, DHCP&httpd將運行在同一個機器底下。

<!-- more -->

沒仔細看是不是以爲我一篇博客寫~<font color=#808080>水</font>~兩次。其實不然，iPXE作爲PXE的擴展版，在[Satellite](https://www.redhat.com/en/technologies/management/satellite), [OpenStack](https://www.redhat.com/en/technologies/linux-platforms/openstack-platform) 乃至於[Openshift](https://www.redhat.com/en/technologies/cloud-computing/openshift) 之中都有應用。因此來簡單學習一下。

## 環境準備

---

如果是物理服務器，請事先確認其[支持iPXE](https://ipxe.org/appnote/hardware_drivers) 。

重複的東西不再贅述，谷歌商店安裝Link to Text Fragment直接跳轉~<font color=#808080>騙點擊率</font>~即可。

### [關閉selinux, firewalld 和 iptables](https://blog.madebug.net/ops/2020-08-01-hello-pxe-kickstart#關閉selinux-firewalld-和-iptables:~:text=%E9%97%9C%E9%96%89selinux%2C%20firewalld%20%E5%92%8C%20iptables,-%E6%8E%92%E9%99%A4%E4%B8%8D%E5%9C%A8%E6%9C%AC%E6%96%87%E6%B6%89%E5%8F%8A%E7%AF%84%E5%9C%8D%E5%86%85%E7%9A%84%E5%86%85%E5%AE%B9%E5%BD%B1%E9%9F%BF%E3%80%82)

### [確保局域網内無DHCP 服務器](https://blog.madebug.net/ops/2020-08-01-hello-pxe-kickstart#確保局域網内無DHCP-服務器:~:text=%2DF-,%E7%A2%BA%E4%BF%9D%E5%B1%80%E5%9F%9F%E7%B6%B2%E5%86%85%E7%84%A1DHCP%20%E6%9C%8D%E5%8B%99%E5%99%A8)

### 設置自定義IP

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/iPXEnetwork.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/iPXEnetwork.png)

## 配置iPXE服務器

---

### 安裝必需包

這次選用dnsmasq來提供ftp和dhcp功能，借此學習一波。

```bash
~]# yum install -y ipxe-bootimgs dnsmasq tree tcpdump wireshark
```

提取引導程序文件undionly.kpxe 和ipxe.efi ，并自建菜單boot.ipxe。

```bash
~]# mkdir /var/lib/tftpboot
# 适用于 BIOS 硬件
~]# cp /usr/share/ipxe/undionly.kpxe /var/lib/tftpboot
# 适用于 UEFI 硬件
~]# cp /usr/share/ipxe/ipxe.efi /var/lib/tftpboot
~]# mkdir /var/lib/tftpboot/menu
~]# cat >> /var/lib/tftpboot/menu/boot.ipxe << EOF
#!ipxe
menu PXE Boot Options
item shell iPXE shell
item exit  Exit to BIOS

choose --default exit --timeout 10000 option && goto ${option}
:shell
shell
:exit
exit
EOF   
```

## 配置Dnsmasq服務器

### 修改配置

參考以下行號來查找内容進行添加/修改。

```bash
~]# vim /etc/dnsmasq.conf
    10  port=0                                            <---禁用dns功能
    106 interface=ens38                                   <---对应工作的网口
    111 listen-address=172.16.70.1                        <---本机用于监听的地址
    180 dhcp-range=172.16.70.200,static,255.255.255.0     <---设定相关的dhcp起始地址及掩码
    229 dhcp-host=00:0c:29:77:62:49,172.16.70.200,test1   <---加入客户机的MAC地址及分配的地址，和标记
    303 dhcp-vendorclass=BIOS,PXEClient:Arch:00000        <---Send extra options which are tagged as "BIOS" to any machine whose DHCP vendorclass string includes the substring "PXEClient"
    450 dhcp-match=set:ipxe,175                           <---iPXE sends a 175 option.
    451 dhcp-boot=tag:!ipxe,tag:BIOS,undionly.kpxe        <---BIOS标签的引导至undionly.kpxe 
    452 dhcp-boot=tag:!ipxe,tag:!BIOS,ipxe.efi            <---非BIOS标签的引导至ipxe.efi
    453 dhcp-boot=tag:ipxe,menu/boot.ipxe                 <---设置它们的菜单文件为boot.ipxe
    499 enable-tftp                                       <---启用tftp
    502 tftp-root=/var/lib/tftpboot                       <---设置tftp的目录
```

### 啓動服務

```bash
~]# systemctl enable dnsmasq
Created symlink from /etc/systemd/system/multi-user.target.wants/dnsmasq.service to /usr/lib/systemd/system/dnsmasq.service.
~]# systemctl start dnsmasq
```

### 檢查DHCP工作情況

在相同LAN下啓動無盤VM用於測試DHCP，測試期間檢查：開機提示、日志和抓包。

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/Dnsmasqresult.png?raw=true](https://img.madebug.net/m4d3bug/images-of-website/master/blog/Dnsmasqresult.png?raw=true)

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/Dnsmasqlog.png?raw=true](https://img.madebug.net/m4d3bug/images-of-website/master/blog/Dnsmasqlog.png?raw=true)

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/tcpdumpDnsmasq.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/tcpdumpDnsmasq.png)

## 配置網絡安裝源

---

### 使用HTTP來提供repo源

```bash
~]# yum -y install httpd
~]# rm -f /etc/httpd/conf.d/welcome.conf
~]# mkdir -p /var/www/html/media/rhel7/7.6
~]# mount -o loop /dev/sr0 /var/www/html/media/rhel7/7.6
~]# df -TH
Filesystem            Type      Size  Used Avail Use% Mounted on
/dev/mapper/rhel-root xfs        54G  8.4G   46G  16% /
devtmpfs              devtmpfs  937M     0  937M   0% /dev
tmpfs                 tmpfs     954M     0  954M   0% /dev/shm
tmpfs                 tmpfs     954M   11M  944M   2% /run
tmpfs                 tmpfs     954M     0  954M   0% /sys/fs/cgroup
/dev/sda1             xfs       1.1G  186M  878M  18% /boot
/dev/mapper/rhel-home xfs        50G   34M   50G   1% /home
tmpfs                 tmpfs     191M   50k  191M   1% /run/user/0
/dev/sr0              iso9660   4.5G  4.5G     0 100% /run/media/root/RHEL-7.6 Server.x86_64
/dev/loop0            iso9660   4.5G  4.5G     0 100% /var/www/html/media/rhel7/7.6
~]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
~]# systemctl restart httpd 
~]# curl -SsLv http://172.16.70.1/media/rhel7/7.6
* About to connect() to 172.16.70.1 port 80 (#0)
*   Trying 172.16.70.1...
* Connected to 172.16.70.1 (172.16.70.1) port 80 (#0)
> GET /media/rhel7/7.6 HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.16.70.1
> Accept: */*
>
< HTTP/1.1 301 Moved Permanently
< Date: Sat, 08 Aug 2020 11:53:20 GMT
< Server: Apache/2.4.6 (Red Hat Enterprise Linux)
< Location: http://172.16.70.1/media/rhel7/7.6/
< Content-Length: 243
< Content-Type: text/html; charset=iso-8859-1
<
* Ignoring the response-body
* Connection #0 to host 172.16.70.1 left intact
* Issue another request to this URL: 'http://172.16.70.1/media/rhel7/7.6/'
* Found bundle for host 172.16.70.1: 0x2648e90
* Re-using existing connection! (#0) with host 172.16.70.1
* Connected to 172.16.70.1 (172.16.70.1) port 80 (#0)
> GET /media/rhel7/7.6/ HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.16.70.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Sat, 08 Aug 2020 11:53:20 GMT
< Server: Apache/2.4.6 (Red Hat Enterprise Linux)
< Content-Length: 3691
< Content-Type: text/html;charset=ISO-8859-1
...

# TROUBLESHOOTING的方法如下
~]# tail -f /var/log/httpd/error_log
~]# journalctl -xefu httpd
~]# tcpdump -i ens38 port 80 and host 172.16.70.1 -vvv >> tcpdump.out
```

### 菜單加入HTTP鏈接

```bash
~]# cat /var/lib/tftpboot/menu/boot.ipxe
#!ipxe
:start
menu PXE Boot Options
item shell iPXE shell
# 增加菜单选项及其内容。
item rhel7-net RHEL 7 installation  
item exit  Exit to BIOS
# 设定安装选项为10s后默认启动项。
choose --default rhel7-net --timeout 10000 option && goto ${option} 
:shell
shell
:rhel7-net
# 设置server_root变量为网络源根目录
set server_root http://172.16.70.1/media/rhel7/7.6
# 指定启动镜像的网络地址        
initrd ${server_root}/images/pxeboot/initrd.images
# 指定相关kernel文件，源仓库根目录，kickstart的文件目录（按需加入，已略去新建存放该文件步骤）
kernel ${server_root}/images/pxeboot/vmlinuz inst.repo=${server_root} inst.ks=http://172.16.70.1/ks/bios-ks.cfg ip=dhcp ipv6.disable initrd=initrd.img inst.geoloc=0 devfs=nomount
boot
:exit
exit
```

## 結語

---

本文簡單嘗試在PXE的網卡環境下調用iPXE固件，除了本文的方法之外，還有以下有趣实现~<font color=#808080>水貼方向</font>~：

- [基於iSCSI的無盤工作站環境搭建](https://www.codenong.com/cs105547860/)

- [在支持iPXE的KVM上安裝你想要的系統](https://lala.im/4524.html)~<font color=#808080>例如kali</font>~，支持iPXE 的[vps](https://zhuanlan.zhihu.com/p/111206825):

>    这里特别说明下一项关键技术iPXE，iPXE是预引导执行环境（PXE）客户端固件和引导程序的开源实现，可用于启用没有内置PXE支持的计算机从网络引导。虽然标准化PXE客户端使用TFTP传输数据，但非标准化iPXE客户端固件增加了通过其他协议检索数据的功能，包括HTTP，iSCSI，以太网ATA（AoE）和以太网光纤通道（FCoE）。

>    为什么在有的云厂商裸金属云服务介绍上，能看到分钟级的裸金属服务器交付，就是使用了iPXE的技术，服务器启动，通过iPXE引导已经制作好的iSCSI系统镜像，这样就免去了安装操作系统的过程，并且服务器也不需要系统硬盘，节省了成本。并且这样更为灵活，通过这样的技术，可以实现用户根据需要制定自己的系统镜像，而且方便镜像虚实转换，镜像机可以用于启动云主机，也可以启动物理机！

另外以下是關於iPXE與PXE較爲[準確的解釋](https://groups.google.com/g/ustc_lug/c/P2jOQ5F4EKY?pli=1#c39:~:text=%E9%A6%96%E5%85%88%E9%9C%80%E8%A6%81%E7%90%86%E6%B8%85%E6%A5%9A%E5%87%A0%E4%B8%AA%E6%A6%82%E5%BF%B5%EF%BC%9Apxe%2C%20ipxe%2C%20pxelinux%EF%BC%8C%E4%BB%A5%E5%8F%8A%E5%87%A0%E4%B8%AA%E5%90%8D%E8%AF%8D%EF%BC%9Apxelinux.0%2C%20ipxelinux.0,%E7%94%B1%E4%BA%8Epxe%E4%BB%A3%E7%A0%81%E6%98%AF%E4%B8%BB%E6%9D%BF%E3%80%81%E7%BD%91%E5%8D%A1%E8%87%AA%E5%B8%A6%E7%9A%84%EF%BC%8C%E6%89%80%E4%BB%A5%E5%85%BC%E5%AE%B9%E6%80%A7%E6%9C%80%E5%A5%BD%EF%BC%88%E8%87%B3%E5%B0%91%E6%9C%AC%E6%9C%BA%E7%9A%84%E4%BB%A3%E7%A0%81%E5%85%BC%E5%AE%B9%E6%9C%AC%E6%9C%BA%E7%9A%84%E8%AE%BE%E5%A4%87%EF%BC%89%E3%80%82%E8%80%8Cipxe%E5%85%BC%E5%AE%B9%E6%80%A7%E7%95%A5%E5%B7%AE%EF%BC%88%E5%8F%AA%E6%98%AF%E7%9B%B8%E5%AF%B9%E6%9D%A5%E8%AF%B4%EF%BC%8C%E5%9B%A0%E4%B8%BA%E6%88%91%E4%BB%AC%E7%BC%96%E8%AF%91%E6%97%B6%E5%8F%AF%E8%83%BD%E4%BC%9A%E6%BC%8F%E6%8E%89%E4%B8%80%E4%BA%9B%E7%BD%91%E5%8D%A1%EF%BC%8C%E6%88%96%E8%80%85%E4%B8%80%E4%BA%9B%E7%89%B9%E6%AE%8A%E9%97%AE%E9%A2%98%E4%B8%8D%E5%A5%BD%E8%A7%A3%E5%86%B3%EF%BC%89%EF%BC%8C%E6%9B%BE%E7%BB%8F%E5%B0%9D%E8%AF%95%E8%BF%87%E7%9B%B4%E6%8E%A5%E4%B8%80%E6%AD%A5%20pxe%20%2D%3E%20ipxelinux.0%EF%BC%8C%E4%BD%86%E6%98%AF%E5%8F%91%E7%8E%B0%E6%9C%89%E4%B8%80%E4%BA%9B%E6%9C%BA%E5%99%A8%E6%97%A0%E6%B3%95%E5%90%AF%E5%8A%A8%EF%BC%8C%E5%8A%A0%E8%BD%BDipxe%E4%B9%8B%E5%90%8E%E5%B0%B1%E5%81%9C%E4%BD%8F%E4%BA%86%E3%80%82%E6%89%80%E4%BB%A5%E5%90%8E%E6%9D%A5%E9%80%80%E8%80%8C%E6%B1%82%E5%85%B6%E6%AC%A1%EF%BC%8C%E7%94%A8%E4%B8%A4%E6%AD%A5%E5%8A%A0%E8%BD%BD%EF%BC%8C%E5%AF%B9%E4%BA%8Eipxe%E4%B8%8D%E6%94%AF%E6%8C%81%E7%9A%84%E8%AE%BE%E5%A4%87%EF%BC%8C%E5%8F%AF%E4%BB%A5%E5%9C%A8%E7%AC%AC%E4%B8%80%E6%AD%A5pxe%2D%3Epxelinux.bin%E4%B9%8B%E5%90%8E%E6%89%8B%E5%BF%AB%E4%B8%80%E4%BA%9B%E6%8C%89%E4%BB%BB%E6%84%8F%E9%94%AE%E4%B8%AD%E6%96%AD%EF%BC%8C%E7%84%B6%E5%90%8E%E4%BB%8D%E7%84%B6%E5%8F%AF%E4%BB%A5%E4%BD%BF%E7%94%A8pxe%EF%BC%8C%E4%B8%8D%E8%BF%87%E4%B9%8B%E5%90%8E%E6%88%91%E4%BB%AC%E4%B8%80%E7%9B%B4%E6%B2%A1%E6%9C%89%E7%BB%B4%E6%8A%A4%E9%80%9A%E8%BF%87tftp%E5%8A%A0%E8%BD%BD%E7%9A%84pxelinux%E4%BB%A5%E5%8F%8A%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%EF%BC%8C%E6%89%80%E4%BB%A5%E9%82%A3%E9%83%A8%E5%88%86%E5%86%85%E5%AE%B9%E5%85%B6%E5%AE%9E%E7%8E%B0%E5%9C%A8%E9%83%BD%E5%B7%B2%E7%BB%8F%E4%B8%A5%E9%87%8D%E8%BF%87%E6%97%B6%E4%BA%86%E3%80%82)：

> * pxe是一个协议，跟mbr是一个性质的东西，它规定了CPU启动后通过什么方式获取引导代码并执行。
> * pxe的实现有许多，不同的厂商有不同的实现。并且pxe的实现代码主要有两种存放位置，一种是存在主板上，一种是存在网卡里，现在新的网卡一般都自带了pxe的实现代码。（去mbr查找引导代码的实现是在主板上的。）
> * 由于pxe协议比较“落后”，仅支持tftp传输数据，性能差，灵活性也差，于是有了gpxe这个项目。gpxe是一种兼容pxe的实现，并且在pxe之上增加了许多特性，例如通过http/ftp等协议传输数据。
> * gpxe原先使用的域名的拥有者突然收回了该域名的使用权，于是这些人fork出去做了ipxe，gpxe现在已经不再开发，ipxe开发非常活跃。
> * 一些较新的intel的网卡里都带了gpxe的实现代码，最新的可能会带ipxe代码。
> * pxelinux是syslinux项目的一个部分，syslinux主要有三个产出，syslinux、isolinux、pxelinux，分别用于硬盘、光盘、网络启动，它的角色与grub相同。

> * *.bin 和 *.0 文件一般是一样的，不过使用上有一些区别，下面解释
> * pxelinux.bin 是 pxelinux 编译后生成的文件
> * 由于大多数网卡、主板都不自带gpxe/ipxe的代码，所以通常引导时需要这样的途径： pxe -> ipxe -> pxelinux.bin，后面这两步可以合并，于是大家就把ipxe与pxelinux.bin​的代码合体，做成了 ipxelinux.0 （gpxe+pxelinux.bin = gpxelinux.0）。一般习惯上裸的pxelinux镜像用.bin后缀，加上gpxe/ipxe之后用.0后缀。此外还会有.lkrn后缀，这是ipxe的东西，ipxe的代码默认只能通过pxe协议的方式加载，他们搞了另外一个代码入口，使得可以通过像linux kernel的方式一样加载（就是可以通过grub引导），这种镜像的后缀是lkrn.
> * 所以可行的引导过程可以有这些：
>  - pxe(网卡) -> ipxe -> pxelinux.bin -> menu.c32
>  - pxe      -> ipxelinux.0 -> menu.c32
>  - pxe -> syslinux.bin -> ipxe -> pxelinux.bin -> menu.c32
>  - pxe -> syslinux.bin -> ipxelinux.0 -> menu.c32
>  - grub -> ipxe.lkrn -> pxelinux.bin -> menu.c32
>  - grub -> ipxelinux.lkrn -> menu.c32
>  - ipxe(烧入网卡) -> pxelinux.bin
>  - ...

> 由于pxe代码是主板、网卡自带的，所以兼容性最好（至少本机的代码兼容本机的设备）。而ipxe兼容性略差（只是相对来说，因为我们编译时可能会漏掉一些网卡，或者一些特殊问题不好解决），曾经尝试过直接一步 pxe -> ipxelinux.0，但是发现有一些机器无法启动，加载ipxe之后就停住了。所以后来退而求其次，用两步加载，对于ipxe不支持的设备，可以在第一步pxe->pxelinux.bin之后手快一些按任意键中断，然后仍然可以使用pxe，不过之后我们一直没有维护通过tftp加载的pxelinux以及配置文件，所以那部分内容其实现在都已经严重过时了。

> 

## 鸣谢

---

- [使用ipxe安装centos7](https://yangfeiffei.github.io/public/2019/08/12/net-install-centos7-with-ipxe.html)
- [gPXE/iPXE 初探 – HydricAcid](https://blog.hcl.moe/archives/2177)
- [CentOS7配置GRUB2+iPXE进行网络重装-荒岛](https://lala.im/4524.html)
- [一文读懂裸金属云 - 知乎](https://zhuanlan.zhihu.com/p/111206825)
- [原先的 PXE 引导中，两阶段的 PXELINUX 的区别，以及 iPXE 的用途？](https://groups.google.com/g/ustc_lug/c/P2jOQ5F4EKY?pli=1)
