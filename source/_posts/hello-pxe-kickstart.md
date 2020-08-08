---
title: PXE+Kickstart搭建無人值守自動化安裝環境
date: 2020-08-01 15:16:19
categories:
- "Ops "
tags:
- "Kickstart "
- "DHCP "
- "TFTP "
- "PXE "
- "Httpd "
- "RHEL 7 "
- "Linux "
- "BIOS "
- "UEFI "
- "Anaconda "
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/PXE_diagram.png" width=50% /></center>

本文將簡單搭建一個基於*PXE* 網絡使用*Kickstart* 自動化安裝*BIOS* 或*UEFI* 架構的自動化安裝環境。其中*TFTP, DHCP & httpd* 將運行在同一個機器底下。

<!-- more -->

## 環境準備

### 關閉*selinux, firewalld* 和 *iptables*

排除不在本文涉及範圍内的内容影響。~<font color=#808080>會考慮單獨開坑記錄selinux的相關。 </font>~

``` nohighlight
~]# sed -i s/^SELINUX=.*$/SELINUX=disabled/ /etc/selinux/config
~]# setenforce 0
~]# cat /etc/selinux/config |grep ^SELINUX=
SELINUX=disabled
~]# systemctl stop firewalld.service
~]# systemctl disable firewalld.service
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
~]# iptables -F
```

### 確保局域網内無*DHCP* 服務器

1. 停用*vmnet* 的*DHCP* 功能。

   ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20191012215006.png?raw=true)

2. 新建一個*LAN* 區段。(本文采用方法)

   ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/setupLAN.png?raw=true)



### 設置自定義*IP*

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/PXEnetwork.png)

## 配置*PXE* 服務器

### 安裝必需包和調試工具

請在下面提取引導菜單程序文件步驟前，確認自己的*ISO* 挂載在*/dev/sr0* 下。

```bash
~]# yum install -y syslinux tftp-server dhcp tree tcpdump
~]# mkdir /var/lib/tftpboot/pxelinux
~]# mkdir -p /mnt/RHEL-7/7.8
~]# mount -o loop /dev/sr0 /mnt/RHEL-7/7.8
~]# df -TH
Filesystem            Type      Size  Used Avail Use% Mounted on
devtmpfs              devtmpfs  2.0G     0  2.0G   0% /dev
tmpfs                 tmpfs     2.0G     0  2.0G   0% /dev/shm
tmpfs                 tmpfs     2.0G   14M  2.0G   1% /run
tmpfs                 tmpfs     2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/rhel-root xfs        54G  8.6G   46G  16% /
/dev/mapper/rhel-home xfs        48G   34M   48G   1% /home
/dev/sda1             xfs       1.1G  192M  872M  19% /boot
tmpfs                 tmpfs     396M  8.2k  396M   1% /run/user/42
tmpfs                 tmpfs     396M   25k  396M   1% /run/user/0
/dev/sr0              iso9660   4.6G  4.6G     0 100% /run/media/root/RHEL-7.8 Server.x86_64
/dev/loop0            iso9660   4.6G  4.6G     0 100% /mnt/RHEL-7/7.8
~]# cp -pr /mnt/RHEL-7/7.8/Packages/syslinux-4.05-15.el7.x86_64.rpm /tmp/
~]# cd /var/lib/tftpboot/
tftpboot]# rpm2cpio /tmp/syslinux-4.05-15.el7.x86_64.rpm | cpio -dimv
tftpboot]# ls
pxelinux  usr
```

###  *BIOS* 的準備工作

```bash
~]# mkdir /var/lib/tftpboot/pxelinux/pxelinux.cfg
~]# cat >> /var/lib/tftpboot/pxelinux/pxelinux.cfg/default << EOF
default vesamenu.c32
prompt 1
timeout 100

label linux
  menu label ^Install system
  menu default
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.repo=http://172.16.7.1/RHEL-7/7.8/Server/x86_64 inst.ks=http://172.16.7.1/ks/bios-ks.cfg
label vesa
  menu label Install system with ^basic video driver
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.xdriver=vesa nomodeset inst.repo=http://172.16.7.1/RHEL-7/7.8/Server/x86_64/os
label rescue
  menu label ^Rescue installed system
  kernel vmlinuz
  append initrd=initrd.img rescue
label local
  menu label Boot from ^local drive
  localboot 0xffff
EOF
~]# cp /mnt/RHEL-7/7.8/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/pxelinux/
~]# cp /var/lib/tftpboot/usr/share/syslinux/{pxelinux.0,vesamenu.c32}  /var/lib/tftpboot/pxelinux/
```

### *UEFI* 的準備工作

在*UEFI* 中，我們用*grubx64.efi* 來代替*pxelinux* 作爲*UEFI* 的設定。

```bash
~]# cp /mnt/RHEL-7/7.8/EFI/BOOT/grubx64.efi /var/lib/tftpboot/
~]# cat >> /var/lib/tftpboot/grub.cfg << EOF
set timeout=9
menuentry 'Install Red Hat Enterprise Linux 7.8' {
        linuxefi pxelinux/vmlinuz ip=dhcp inst.repo=http://172.16.7.1/RHEL-7/7.8/Server/x86_64 inst.ks=http://172.16.7.1/ks/uefi-ks.cfg inst.gpt
        initrdefi pxelinux/initrd.img
}
EOF
```

## 配置*TFTP* 服務器

``` nohighlight
~]# sed -i '/disable/s/yes/no/g' /etc/xinetd.d/tftp
~]# cat /etc/xinetd.d/tftp |grep disable
	disable			= no
~]# systemctl start tftp
~]# systemctl enable tftp
Created symlink from /etc/systemd/system/sockets.target.wants/tftp.socket to /usr/lib/systemd/system/tftp.socket.
```

## 配置*DHCP* 服務器

```nohighlight
~]# cat >> /etc/dhcp/dhcpd.conf << EOF
# FILL THIS UP
#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#
option space pxelinux;
option pxelinux.magic code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;
option architecture-type code 93 = unsigned integer 16;

subnet 172.16.7.0 netmask 255.255.255.0 {
  option routers 172.16.7.1;
  range 172.16.7.200 192.168.188.253;

  class "pxeclients" {
      match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
      next-server 172.16.7.1;

      if option architecture-type = 00:07 {
        filename "grubx64.efi";
      } else {
        filename "pxelinux/pxelinux.0";
      }
  }
}
EOF
~]# systemctl start dhcpd 
~]# systemctl enable dhcpd 
Created symlink from /etc/systemd/system/multi-user.target.wants/dhcpd.service to /usr/lib/systemd/system/dhcpd.service.
~]# rm -rf /var/lib/tftpboot/usr
~]# tree -L 3 /var/lib/tftpboot/
/var/lib/tftpboot/
├── grub.cfg                     <--- UEFI的配置文件
├── grubx64.efi                  <--- UEFI的引導程序
└── pxelinux
    ├── initrd.img               <--- 啓動過程中的鏡像
    ├── pxelinux.0               <--- BIOS的引導程序
    ├── pxelinux.cfg             <--- 默認為BIOS的引導程序的配置文件夾
    │   └── default              <--- 默認為BIOS的引導程序的配置文件
    ├── vesamenu.c32             <--- 配置文件引用的頭文件，用於提供簡單菜單背景
    └── vmlinuz                  <--- BIOS的引導程序所需的内核文件

2 directories, 7 files

```

### 檢查*DHCP* 工作情況

在相同*LAN* 下啓動無盤*VM* 用於測試*DHCP*，測試期間檢查：開機提示、日志和抓包。

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/DHCPresult.png?raw=true)

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/DHCPlog.png?raw=true)

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/tcpdumpDHCP.png)

## 配置網絡安裝源

### 使用*http* 來提供*repo* 源

``` nohighlight
~]# yum -y install httpd
~]# rm -f /etc/httpd/conf.d/welcome.conf
~]# cat >> /etc/httpd/conf.d/iso.conf << EOF
# 新建該文件
Alias /RHEL-7/7.8/Server/x86_64 /mnt/RHEL-7/7.8
<Directory /mnt/RHEL-7/7.8>    
    Require all granted    
    Options Indexes FollowSymLinks    
    Require ip 0.0.0.0
</Directory>
EOF
~]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
~]# systemctl start httpd 
~]# curl -SsLv http://172.16.7.1/RHEL-7/7.8/Server/x86_64/
* About to connect() to 172.16.7.1 port 80 (#0)
*   Trying 172.16.7.1...
* Connected to 172.16.7.1 (172.16.7.1) port 80 (#0)
> GET /RHEL-7/7.8/Server/x86_64/ HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.16.7.1> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Sat, 01 Aug 2020 15:16:19 GMT
< Server: Apache/2.4.6 (Red Hat Enterprise Linux)
< Content-Length: 3715
< Content-Type: text/html;charset=ISO-8859-1
<
...

# TROUBLESHOOTING的方法如下
~]# tail -f /var/log/httpd/error_log
~]# journalctl -xefu httpd
~]# tcpdump -i ens38 port 80 and host 172.16.7.1 -vvv >> tcpdump.out
```

## 設定*Kickstart* 配置

### 創建相關目錄

```nohighlight
~]# mkdir /var/www/html/ks
```

### 創建密碼的加密字段
```nohighlight
~]# python -c 'import crypt,getpass;pw=getpass.getpass();print(crypt.crypt(pw) if (pw==getpass.getpass("Confirm: ")) else exit())'
Password: 
Confirm: 
$6$SXp9tsalYxyM41qQ$mG3TbO58L9m3.Hhlec.7aoAU2AeATpJ4p.5dmTXy1iKZkoqALFi9VOhFEWWJ7Tvk6bDYbTx4SRqHw14mVnbV2.
```

### 創建*BIOS*自應答文件

```nohighlight
~]# cat >> /var/www/html/ks/bios-ks.cfg << EOF
#platform=x86, AMD64, or Intel EM64T
#version=DEVEL
# Install OS instead of upgrade
install
# Keyboard layouts:
keyboard 'us'
# Root password
rootpw --iscrypted $6$SXp9tsalYxyM41qQ$mG3TbO58L9m3.Hhlec.7aoAU2AeATpJ4p.5dmTXy1iKZkoqALFi9VOhFEWWJ7Tvk6bDYbTx4SRqHw14mVnbV2.
# System language
lang en_US
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use graphical install
graphical
firstboot --disable
# SELinux configuration
selinux --disabled


# Firewall configuration
firewall --disabled
# Network information
network  --bootproto=dhcp --device=ens33
# Reboot after installation
reboot
# System timezone
timezone Africa/Abidjan
# Use network installation
url --url="http://172.16.7.1/RHEL-7/7.8/Server/x86_64"
# System bootloader configuration
bootloader --location=mbr --boot-drive=sda
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
part /boot --fstype=xfs --size=500
part pv.009009 --grow --size=1
volgroup VolGroup --pesize=4096 pv.009009
logvol / --fstype=xfs --name=lv_root --vgname=VolGroup --percent=100
logvol swap --fstype=swap --name=lv_swap --vgname=VolGroup --recommended

%packages
@^graphical-server-environment
@base
@core
@desktop-debugging
@dial-up
@fonts
@gnome-desktop
@guest-agents
@guest-desktop-agents
@hardware-monitoring
@input-methods
@internet-browser
@multimedia
@print-client
@x11
chrony
kexec-tools

%end

%addon com_redhat_kdump --enable --reserve-mb='auto'
%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
EOF
```

### 創建*UEFI*自應答文件

```nohighlight
~]# cat >> /var/www/html/ks/uefi-ks.cfg << EOF
#platform=x86, AMD64, or Intel EM64T
#version=DEVEL
# Install OS instead of upgrade
install
# Keyboard layouts
keyboard 'us'
# Root password
rootpw --iscrypted $6$SXp9tsalYxyM41qQ$mG3TbO58L9m3.Hhlec.7aoAU2AeATpJ4p.5dmTXy1iKZkoqALFi9VOhFEWWJ7Tvk6bDYbTx4SRqHw14mVnbV2.
# System language
lang en_US
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use graphical install
graphical
firstboot --disable
# SELinux configuration
selinux --disabled


# Firewall configuration
firewall --disabled
# Network information
network  --bootproto=dhcp --device=ens33
# Reboot after installation
reboot
# System timezone
timezone Africa/Abidjan
# Use network installation
url --url="http://172.16.7.1/RHEL-7/7.8/Server/x86_64"
# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
# Clear the Master Boot Record
# zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
part /boot/efi --fstype=efi --size=200 --asprimary
part /boot --fstype=xfs --size=500
part pv.009009 --grow --size=1
volgroup VolGroup --pesize=4096 pv.009009
logvol / --fstype=xfs --name=lv_root --vgname=VolGroup --percent=100
logvol swap --name=lv_swap --vgname=VolGroup --recommended

%packages
@^graphical-server-environment
@base
@core
@desktop-debugging
@dial-up
@fonts
@gnome-desktop
@guest-agents
@guest-desktop-agents
@hardware-monitoring
@input-methods
@internet-browser
@multimedia
@print-client
@x11
chrony
kexec-tools

%end

%addon com_redhat_kdump --enable --reserve-mb='128M'
%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
EOF
```

啓用之前的*VM*，并且確保其不包含*ISO*。

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20191012215904.png?raw=true)

有以下界面出現就代表成功。

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20191012215957.png?raw=true)

創建UEFI也可以更改以下設置。

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/UEFIVM.png?raw=true)

## 結語

簡單嘗試了基於*PXE*搭配*Kickstart*的無人值守安裝搭建。

## 鳴謝

- https://www.cnblogs.com/boowii/p/6475921.html
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/installation_guide/index#chap-installation-server-setup
- https://www.server-world.info/en/note?os=CentOS_7&p=pxe&f=1

