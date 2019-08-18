---
title: Hello Kickstart
date: 2019-08-18 12:21:48
categories:
- "Ops"
tags:
- "Kickstart"
- "DHCP"
- "TFTP"
- "PXE"
- "httpd"
- "RHEL 7"
---

In this post, I am going to markdown how I tested the unattended kickstart installation in vm15.

And we need to run the following service on the same machine: TFTP, DHCP & httpd.

## Environmental preparation

### Turn off the selinux ,firewalld and iptables

``` bash
~]# cat /etc/selinux/config |grep ^SELINUX=
SELINUX=disabled
~]# systemctl stop firewalld.service
~]# systemctl disable firewalld.service
~]# iptables -F
```

### Stop Virtual Network Editor's dhcp services

![](https://i.loli.net/2019/08/18/eKsX6nHdEc7bMB8.png)

### Configure custom ip

![](https://i.loli.net/2019/08/18/AkWE7qphu9iFcSI.png)

## Configure PXE Server

### Install required packages and copy boot menu program file

```bash
~]# yum -y install syslinux xinetd tftp-server dhcp
~]# mkdir /var/lib/tftpboot/pxelinux.cfg 
~]# cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/ 
```

### Start TFTP server

``` bash
~]# cat /etc/xinetd.d/tftp |grep disable
	disable			= no
~]# systemctl start xinetd 
~]# systemctl enable xinetd
```

### Start DHCP server and specify PXE server's IP for "next-server"

```bash
~]# cat /etc/dhcp/dhcpd.conf 

#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#
option domain-name  "srv.world";
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

## Configure Network Install 

### Use the environment's own image and create boot menu

``` bash
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

### Use httpd to provide http services

``` bash
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

### Create a  vm machines in the same LAN without iso.

![](https://i.loli.net/2019/08/18/bkgTQwx8hzvJp7f.png)

Enable network boot on BIOS settings of client computer and start it, then installation menu is shown, push Enter key to proceed to install.

![](https://i.loli.net/2019/08/18/3a6yE5s4xNDMZCX.png)

