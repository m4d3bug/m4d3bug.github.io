---
title: Hello Kickstart + EFI
date: 2019-08-18 12:21:48
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
---

*In this post, I am going to markdown how I tested unattended kickstart installation in vm15 by using PXE.

And we need to run the following service on the same machine: TFTP, DHCP & httpd.*

## *Environmental preparation*

### *Turn off the selinux ,firewalld and iptables*

``` nohighlight
~]# cat /etc/selinux/config |grep ^SELINUX=
SELINUX=disabled
~]# systemctl stop firewalld.service
~]# systemctl disable firewalld.service
~]# iptables -F
```

### *Stop Virtual Network Editor's dhcp services*

![](https://i.loli.net/2019/08/26/buNconEfrlS68J3.jpg)

### *Configure custom ip*

![](https://i.loli.net/2019/08/18/AkWE7qphu9iFcSI.png)

## *Configure PXE Server*

### *Install required packages and copy boot menu program file*

```nohighlight
[root@localhost ~]# yum -y install syslinux xinetd tftp-server dhcp
[root@localhost ~]# mkdir /var/lib/tftpboot/pxelinux.cfg 
[root@localhost ~]# cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/ 
```

### *Start TFTP server*

``` nohighlight
[root@localhost ~]# cat /etc/xinetd.d/tftp |grep disable
	disable			= no
[root@localhost ~]# systemctl start xinetd 
[root@localhost ~]# systemctl enable xinetd
```

### *Start DHCP server and specify PXE server's IP for "next-server"*

```nohighlight
[root@localhost ~]# cat /etc/dhcp/dhcpd.conf 

#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#
option domain-name-servers 192.168.188.174;
default-lease-time 300;
max-lease-time 7200;
authoritative;

subnet 192.168.188.0 netmask 255.255.255.0 {
    range	dynamic-bootp 192.168.188.200 192.168.188.254;
    option	broadcast-address 192.168.188.255;
    option	routers 192.168.188.2;
    filename	"pxelinux.0";
    next-server	192.168.188.174;
}
~]# systemctl start dhcpd 
~]# systemctl enable dhcpd 
```

## *Configure Network Install*

### *Use the environment's own image and create boot menu*

``` nohighlight
[root@localhost ~]# mkdir -p /var/pxe/rhel7 
[root@localhost ~]# mkdir -p /var/lib/tftpboot/pxelinux.cfg/{5,6,7}/{i386,x86_64}
============================
[root@localhost ~]# mount /dev/cdrom /mnt/
[root@localhost ~]# cp -rvf /mnt/* /var/pxe/rhel7/
============================
[root@localhost ~]# cp /var/pxe/rhel7/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/pxelinux.cfg/7/x86_64/
[root@localhost ~]# cp /usr/share/syslinux/menu.c32 /var/lib/tftpboot/
[root@localhost ~]# cp /var/pxe/rhel7/isolinux/{splash.png,memtest,boot.msg} /var/lib/tftpboot/pxelinux.cfg
[root@localhost tftpboot]# tree 
.
├── menu.c32     ----- > Use for draw PXE menu.(Or use vmsamenu.c32 from syslinux-tftpboot)
├── pxelinux.0   ------> Bootloader for Bios.  (Provider by syslinux)
└── pxelinux.cfg ------> Default dir of pxeliunx.0's config.
    ├── 5
    │   ├── i386
    │   └── x86_64
    ├── 6
    │   ├── i386
    │   └── x86_64
    ├── 7
    │   ├── i386
    │   └── x86_64
    │       ├── initrd.img
    │       └── vmlinuz
    ├── boot.msg
    ├── default ------> Config file.
    ├── memtest
    └── splash.png

10 directories, 8 files

~]# cat /var/lib/tftpboot/pxelinux.cfg/default
timeout 100
default menu.c32

menu title ########## PXE Boot Menu ##########
label 1
   menu label ^1) Install RHEL 7
   kernel rhel7/vmlinuz
   append initrd=rhel7/initrd.img inst.stage2=http://192.168.188.174/rhel7 inst.ks=http://192.168.188.174/ks/rhel-ks.cfg quiet

label 2
   menu label ^2) Boot from local drive
   localboot
```

### *Use httpd to provide http services*

``` nohighlight
[root@localhost ~]# yum -y install httpd
[root@localhost ~]# rm -f /etc/httpd/conf.d/welcome.conf
[root@localhost ~]# vi /etc/httpd/conf.d/pxeboot.conf
# create new
Alias /rhel7 /var/pxe/rhel7
<Directory /var/pxe/rhel7>
    Options Indexes FollowSymLinks
    Require ip 127.0.0.1 192.168.188.0/24
</Directory>
[root@localhost ~]# systemctl enable httpd 
[root@localhost ~]# systemctl start httpd 
```

### *Create a  vm machines in the same LAN without iso.*

![](https://i.loli.net/2019/08/19/w8lkvhm1aFeg5ur.jpg)

*Enable network boot on BIOS settings of client computer and start it, then installation menu is shown, push Enter key to proceed to install.*

![](https://i.loli.net/2019/08/18/3a6yE5s4xNDMZCX.png)

## *Configure Kickstart Install*

```nohighlight
[root@localhost ~]# mkdir /var/www/html/ks 
[root@localhost ~]# yum install -y system-config-kickstart
[root@localhost ~]# system-config-kickstart
```

### *Custom your ks.cfg*

![](https://i.loli.net/2019/08/18/NiJG2l7BSgHyvWe.png)

![](https://i.loli.net/2019/08/18/Ukyx7uJDowYIAVQ.png)

![](https://i.loli.net/2019/08/18/K1WNP4OGlFILhCS.png)

![](https://i.loli.net/2019/08/18/GFX76hIsiwB9AzO.png)

![](https://i.loli.net/2019/08/18/VcaJwYrvNjkhDOF.png)

![](https://i.loli.net/2019/08/18/8y5D3huAg2siJ6L.png)

![](https://i.loli.net/2019/08/20/YzJ91PCfjvQFKs2.jpg)

![](https://i.loli.net/2019/08/18/2wWiAxO8HvVClK6.png)

### *Configure packages I need*

```nohighlight
[root@localhost ~]# cat ~/anaconda-ks.cfg |grep -A 19 %packages >> /var/www/html/ks/rhel-ks.cfg
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
```

### *Configure disk*

``` nohighlight
[root@localhost ~]# sed -i '/bootloader --location=mbr/a autopart --type=lvm' /var/www/html/ks/rhel-ks.cfg
```

*Start the vm machine which is enabled network booting on BIOS settings, then PXE boot menu is shown. After 10 seconds later, installation process starts and will finish and reboot automatically.*

## *Done*

*Simply tried to complete the unattended installation of RHEL7 by using Kickstart with PXE.*

### *Acknowledgements*

- https://www.cnblogs.com/boowii/p/6475921.html
- https://www.server-world.info/en/note?os=CentOS_7&p=pxe&f=1

