---
title: iPXE+Kickstart搭建無人值守自動化安裝環境
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

本文將簡單搭建一個基於*[iPXE](https://ipxe.org/)*網絡使用*Kickstart*自動化安裝*BIOS*或*UEFI*架構的自動化安裝環境。其中*TFTP, DHCP & httpd* 將運行在同一個機器底下。

<!-- more -->

沒仔細看是不是以爲我一篇博客寫~<font color=#808080>水</font>~兩次。其實不然，*iPXE* 作爲*PXE* 的擴展版，在[*Satellite*](https://www.redhat.com/en/technologies/management/satellite), [*OpenStack*](https://www.redhat.com/en/technologies/linux-platforms/openstack-platform) 乃至於[*Openshift*](https://www.redhat.com/en/technologies/cloud-computing/openshift) 之中都有應用。因此來簡單學習一下。

## 環境準備

---

如果是物理服務器，請事先確認其[支持*iPXE*](https://ipxe.org/appnote/hardware_drivers) 。

重複的東西不再贅述，谷歌商店安裝Link to Text Fragment直接跳轉~<font color=#808080>騙點擊率</font>~即可。

### [關閉*selinux, firewalld* 和 *iptables*](https://blog.madebug.net/ops/2020-08-01-hello-pxe-kickstart#關閉selinux-firewalld-和-iptables:~:text=%E9%97%9C%E9%96%89selinux%2C%20firewalld%20%E5%92%8C%20iptables,-%E6%8E%92%E9%99%A4%E4%B8%8D%E5%9C%A8%E6%9C%AC%E6%96%87%E6%B6%89%E5%8F%8A%E7%AF%84%E5%9C%8D%E5%86%85%E7%9A%84%E5%86%85%E5%AE%B9%E5%BD%B1%E9%9F%BF%E3%80%82)

### [確保局域網内無*DHCP* 服務器](https://blog.madebug.net/ops/2020-08-01-hello-pxe-kickstart#確保局域網内無DHCP-服務器:~:text=%2DF-,%E7%A2%BA%E4%BF%9D%E5%B1%80%E5%9F%9F%E7%B6%B2%E5%86%85%E7%84%A1DHCP%20%E6%9C%8D%E5%8B%99%E5%99%A8)

### 設置自定義IP

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/iPXEnetwork.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/iPXEnetwork.png)

## 配置*iPXE* 服務器

---

### 安裝必需包

這次選用*dnsmasq* 來提供*ftp* 和*dhcp* 功能，借此學習一波。

```bash
~]# yum install -y ipxe-bootimgs dnsmasq tree tcpdump wireshark
```

提取引導程序文件*undionly.kpxe* 和*ipxe.efi* ，并自建菜單*boot.ipxe* 。

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

### 檢查*DHCP* 工作情況

在相同*LAN* 下啓動無盤*VM* 用於測試*DHCP*，測試期間檢查：開機提示、日志和抓包。

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/Dnsmasqresult.png?raw=true](https://img.madebug.net/m4d3bug/images-of-website/master/blog/Dnsmasqresult.png?raw=true)

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/Dnsmasqlog.png?raw=true](https://img.madebug.net/m4d3bug/images-of-website/master/blog/Dnsmasqlog.png?raw=true)

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/tcpdumpDnsmasq.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/tcpdumpDnsmasq.png)

## 配置網絡安裝源

---

### 使用*HTTP* 來提供*repo* 源

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

### 菜單加入*HTTP* 鏈接

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

本文簡單搭建了iPXE的環境，除了本文的方法之外，還有以下有趣实现~<font color=#808080>水貼方向</font>~：

- [基於*iSCSI* 的無盤工作站環境搭建](https://www.codenong.com/cs105547860/)

- [在支持*iPXE* 的*KVM* 上安裝你想要的系統](https://lala.im/4524.html)~<font color=#808080>例如*kali*</font>~，支持*iPXE* 的*[vps](https://zhuanlan.zhihu.com/p/111206825):*

>    这里特别说明下一项关键技术iPXE，iPXE是预引导执行环境（PXE）客户端固件和引导程序的开源实现，可用于启用没有内置PXE支持的计算机从网络引导。虽然标准化PXE客户端使用TFTP传输数据，但非标准化iPXE客户端固件增加了通过其他协议检索数据的功能，包括HTTP，iSCSI，以太网ATA（AoE）和以太网光纤通道（FCoE）。

>    为什么在有的云厂商裸金属云服务介绍上，能看到分钟级的裸金属服务器交付，就是使用了iPXE的技术，服务器启动，通过iPXE引导已经制作好的iSCSI系统镜像，这样就免去了安装操作系统的过程，并且服务器也不需要系统硬盘，节省了成本。并且这样更为灵活，通过这样的技术，可以实现用户根据需要制定自己的系统镜像，而且方便镜像虚实转换，镜像机可以用于启动云主机，也可以启动物理机！

## 鸣谢

---

- [https://yangfeiffei.github.io/public/2019/08/12/net-install-centos7-with-ipxe.html](https://yangfeiffei.github.io/public/2019/08/12/net-install-centos7-with-ipxe.html)
- [https://blog.hcl.moe/archives/2177](https://blog.hcl.moe/archives/2177)
- [https://lala.im/4524.html](https://lala.im/4524.html)
- [https://zhuanlan.zhihu.com/p/111206825](https://zhuanlan.zhihu.com/p/111206825)
