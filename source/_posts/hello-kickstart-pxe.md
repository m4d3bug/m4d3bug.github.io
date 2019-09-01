---
title: Hello Kickstart + PXE
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
~]# yum -y install syslinux xinetd tftp-server dhcp
~]# mkdir /var/lib/tftpboot/pxelinux.cfg 
~]# cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/ 
```

### *Start TFTP server*

``` nohighlight
~]# cat /etc/xinetd.d/tftp |grep disable
	disable			= no
~]# systemctl start xinetd 
~]# systemctl enable xinetd
```

### *Start DHCP server and specify PXE server's IP for "next-server"*

```nohighlight
~]# cat /etc/dhcp/dhcpd.conf 

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
~]# mkdir -p /var/pxe/rhel7 
~]# mkdir /var/lib/tftpboot/rhel7
============================
~]# mount /dev/cdrom /mnt/
~]# cp -rvf /mnt/* /var/pxe/rhel7/
============================
~]# cp /var/pxe/rhel7/images/pxeboot/vmlinuz /var/lib/tftpboot/rhel7/
~]# cp /var/pxe/rhel7/images/pxeboot/initrd.img /var/lib/tftpboot/rhel7/
~]# cp /usr/share/syslinux/menu.c32 /var/lib/tftpboot/
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
~]# yum -y install httpd
~]# rm -f /etc/httpd/conf.d/welcome.conf
~]# vi /etc/httpd/conf.d/pxeboot.conf
# create new
Alias /rhel7 /var/pxe/rhel7
<Directory /var/pxe/rhel7>
    Options Indexes FollowSymLinks
    Require ip 127.0.0.1 192.168.188.0/24
</Directory>
~]# systemctl enable httpd 
~]# systemctl start httpd 
```

### *Create a  vm machines in the same LAN without iso.*

![](https://i.loli.net/2019/08/19/w8lkvhm1aFeg5ur.jpg)

*Enable network boot on BIOS settings of client computer and start it, then installation menu is shown, push Enter key to proceed to install.*

![](https://i.loli.net/2019/08/18/3a6yE5s4xNDMZCX.png)

## *Configure Kickstart Install*

```nohighlight
~]# mkdir /var/www/html/ks 
~]# yum install -y system-config-kickstart
~]# system-config-kickstart
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
]# cat ~/anaconda-ks.cfg |grep -A 19 %packages >> /var/www/html/ks/rhel-ks.cfg
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
]# sed -i '/bootloader --location=mbr/a autopart --type=lvm' /var/www/html/ks/rhel-ks.cfg
```

*Start the vm machine which is enabled network booting on BIOS settings, then PXE boot menu is shown. After 10 seconds later, installation process starts and will finish and reboot automatically.*

## *Done*

*Simply tried to complete the unattended installation of RHEL7 by using Kickstart with PXE.*

