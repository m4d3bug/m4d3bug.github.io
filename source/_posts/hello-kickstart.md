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

## Setup The PXE server

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

## Start DHCP server and specify PXE server's IP for "next-server"

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







