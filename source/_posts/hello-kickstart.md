---
title: Hello Kickstart
date: 2019-08-18 12:21:48
categories:
- "Ops"
tags:
- "Kickstart"
- “DHCP”
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
# cat /etc/selinux/config |grep ^SELINUX=
SELINUX=disabled
# systemctl stop firewalld.service
# systemctl disable firewalld.service
# iptables -F
```

## Setup The PXE server

### Install required packages and copy boot menu program file

```bash
# yum -y install syslinux xinetd tftp-server dhcp
# mkdir /var/lib/tftpboot/pxelinux.cfg 
# cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/ 
```

### Start TFTP server

``` bash
# cat /etc/xinetd.d/tftp |grep disable
	disable			= no
# systemctl start xinetd 
# systemctl enable xinetd 
```







