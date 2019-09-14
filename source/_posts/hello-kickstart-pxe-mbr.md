---
title: Hello Kickstart + PXE +MBR
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
- "MBR "
- "Anaconda "
---

*In this post, I am going to markdown how I tested unattended kickstart installation in vm15 by using PXE.*

*And we need to run the following service on the same machine: TFTP, DHCP & httpd.*

## *Environmental preparation*

### *Turn off the selinux ,firewalld and iptables*

``` nohighlight
~]# cat /etc/selinux/config |grep ^SELINUX=
SELINUX=disabled
~]# systemctl stop firewalld.service
~]# systemctl disable firewalld.service
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
~]# iptables -F
```

### *Stop Virtual Network Editor's dhcp services*

![](https://i.loli.net/2019/08/26/buNconEfrlS68J3.jpg)

### *Configure custom ip*

![](https://i.loli.net/2019/08/18/AkWE7qphu9iFcSI.png)

## *Configure PXE Server*

### *Install required packages and copy boot menu program file*

```nohighlight
[root@localhost ~]# yum -y install syslinux tftp-server dhcp
[root@localhost ~]# mkdir /var/lib/tftpboot/pxelinux
[root@localhost ~]# mount /dev/cdrom /mnt/
mount: /dev/sr0 is write-protected, mounting read-only
[root@localhost ~]# cp -pr /mnt/Packages/syslinux-4.05-13.el7.x86_64.rpm /tmp
[root@localhost ~]# cd /var/lib/tftpboot/
[root@localhost tftpboot]# rpm2cpio /tmp/syslinux-4.05-13.el7.x86_64.rpm |cpio -dimv
[root@localhost tftpboot]# ls
pxelinux  usr
[root@localhost ~]# mkdir /var/lib/tftpboot/pxelinux/pxelinux.cfg
[root@localhost ~]# cat /var/lib/tftpboot/pxelinux/pxelinux.cfg/default
default vesamenu.c32
prompt 1
timeout 100

label linux
  menu label ^Install system
  menu default
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.repo=http://192.168.188.174/RHEL-7/7.4/Server/x86_64/
label vesa
  menu label Install system with ^basic video driver
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.xdriver=vesa nomodeset inst.repo=http://192.168.188.174/RHEL-7/7.x/Server/x86_64/os/
label rescue
  menu label ^Rescue installed system
  kernel vmlinuz
  append initrd=initrd.img rescue
label local
  menu label Boot from ^local drive
  localboot 0xffff
[root@localhost ~]# cp /mnt/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/pxelinux/
[root@localhost ~]# cp /var/lib/tftpboot/usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/pxelinux/
[root@localhost ~]# cp /var/lib/tftpboot/usr/share/syslinux/vesamenu.c32 /var/lib/tftpboot/pxelinux/
[root@localhost tftpboot]# umount /mnt/
[root@localhost tftpboot]# tree -L 2 .
.
├── pxelinux
│   ├── initrd.img
│   ├── pxelinux.0        ------> Bootloader for Bios.
│   ├── pxelinux.cfg      ------> Default dir of pxeliunx.0's config.
│   ├── vesamenu.c32      ----- > Use for draw PXE menu.
│   └── vmlinuz           ----- > Kernel
└── usr
    ├── bin
    └── share

5 directories, 4 files
```

### *Start TFTP server*

``` nohighlight
[root@localhost ~]# cat /etc/xinetd.d/tftp |grep disable
	disable			= no
[root@localhost ~]# systemctl start tftp
[root@localhost ~]# systemctl enable tftp
Created symlink from /etc/systemd/system/sockets.target.wants/tftp.socket to /usr/lib/systemd/system/tftp.socket.
```

### *Start DHCP server and specify PXE server's IP for "next-server"*

```nohighlight
[root@localhost ~]# cat /etc/dhcp/dhcpd.conf 

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

subnet 192.168.188.0 netmask 255.255.255.0 {
  option routers 192.168.188.2;
  range 192.168.188.200 192.168.188.253;

  class "pxeclients" {
      match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
      next-server 192.168.188.174;

      if option architecture-type = 00:07 {
        filename "uefi/shim.efi";
      } else {
        filename "pxelinux/pxelinux.0";
      }
  }
}

~]# systemctl start dhcpd 
~]# systemctl enable dhcpd 
Created symlink from /etc/systemd/system/multi-user.target.wants/dhcpd.service to /usr/lib/systemd/system/dhcpd.service.
```

*The dhcpd working should get these:*

![dhcplog.png](https://i.loli.net/2019/09/14/r62YRjlST4QObtg.png)

![dhcpok.png](https://i.loli.net/2019/09/14/aPyrBv4pixcd6mL.png)

## Configure Network Install*

### *Use httpd to provide http repo*

``` nohighlight
[root@localhost ~]# mkdir -p /mnt/RHEL-7/7.4
[root@localhost ~]# mount /dev/sr0 /mnt/RHEL-7/7.4
mount: /dev/sr0 is write-protected, mounting read-only

[root@localhost ~]# yum -y install httpd
[root@localhost ~]# rm -f /etc/httpd/conf.d/welcome.conf
[root@localhost ~]# cat /etc/httpd/conf.d/iso.conf
# Create this
Alias /RHEL-7/7.4/Server/x86_64 /mnt/RHEL-7/7.4
<Directory /mnt/RHEL-7/7.4>
    Options Indexes FollowSymLinks
    Require ip 127.0.0.1 192.168.188.0/24
</Directory>
[root@localhost ~]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
[root@localhost ~]# systemctl start httpd 
[root@localhost ~]# curl -v http://192.168.188.174/RHEL-7/7.4/Server/x86_64
...
* Connection #0 to host 192.168.188.174 left intact
[root@localhost ~]# tcpdump -i ens33 port 80 and host 172.16.1.100 -vvv >> tcpdump.out
```

### *Create a  vm machines in the same LAN without iso.*

![](https://i.loli.net/2019/08/19/w8lkvhm1aFeg5ur.jpg)

*Enable network boot on BIOS settings of client computer and start it, then installation menu is shown, push Enter key to proceed to install.*

![ancanda.png](https://i.loli.net/2019/09/14/5iecmlxNtYS3jyk.png)

## *Configure Kickstart Install*

```nohighlight
[root@localhost ~]# mkdir /var/www/html/ks 
[root@localhost ~]# yum install -y system-config-kickstart
[root@localhost ~]# system-config-kickstart
```

### *Custom your ks.cfg*

*Also can generate from [here](https://access.redhat.com/labs/kickstartconfig/).*

![](https://i.loli.net/2019/08/18/NiJG2l7BSgHyvWe.png)

![](https://i.loli.net/2019/08/18/Ukyx7uJDowYIAVQ.png)

![](https://i.loli.net/2019/08/18/K1WNP4OGlFILhCS.png)

![](https://i.loli.net/2019/08/18/GFX76hIsiwB9AzO.png)

![](https://i.loli.net/2019/08/18/VcaJwYrvNjkhDOF.png)

![](https://i.loli.net/2019/08/20/YzJ91PCfjvQFKs2.jpg)

![](https://i.loli.net/2019/08/18/8y5D3huAg2siJ6L.png)

![mbr-ks.png](https://i.loli.net/2019/09/14/8aAY9sFUrqibzCp.png)

### *Configure packages I need*

```nohighlight
[root@localhost ~]# cat ~/anaconda-ks.cfg |grep -A 19 %packages >> /var/www/html/ks/mbr-ks.cfg
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
[root@localhost ~]# sed -i '/bootloader --location=mbr/a autopart --type=lvm' /var/www/html/ks/mbr-ks.cfg
```

### *Edit the boot menu*

```nohighlight
[root@localhost ~]# more /var/lib/tftpboot/pxelinux/pxelinux.cfg/default 
default vesamenu.c32
prompt 1
timeout 100

label linux
  menu label ^Install system
  menu default
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.repo=http://192.168.188.174/RHEL-7/7.4/Server/x86_64/ inst.ks=http://192.168.188.174/ks/mbr-ks.cfg
label vesa
  menu label Install system with ^basic video driver
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.xdriver=vesa nomodeset inst.repo=http://192.168.188.174/RHEL-7/7.4/Server/x86_64/
label rescue
  menu label ^Rescue installed system
  kernel vmlinuz
  append initrd=initrd.img rescue
label local
  menu label Boot from ^local drive
  localboot 0xffff
```

*Start the vm machine which is enabled network booting on BIOS settings, then PXE boot menu is shown. After 10 seconds later, installation process starts and will finish and reboot automatically.*

## *Done*

*Simply tried to complete the unattended installation of RHEL7 by using Kickstart with PXE.*

## *Acknowledgements*

- https://www.cnblogs.com/boowii/p/6475921.html
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/installation_guide/index#chap-installation-server-setup
- https://www.server-world.info/en/note?os=CentOS_7&p=pxe&f=1

