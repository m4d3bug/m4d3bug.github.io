---
title: Hello Kickstart + USB
date: 2019-08-20 15:47:23
categories:
- "Ops "
tags:
- "Kickstart "
- "USB "
- "RHEL 7 "
- "Linux "
---

*In this post, I am going to markdown how I tested unattended kickstart installation in vm15 by using USB.*

*And I will custom my own image before everything start.*

## *Environmental preparation*

### *Turn off the selinux*

``` nohighlight
~]# cat /etc/selinux/config |grep ^SELINUX=
SELINUX=disabled
```

 ## *Configure USB Install*

### *Use the environment's own image and modify the boot menu*

``` nohighlight
~]# mkdir -p /var/usb/rhel7 
============================
~]# mount /dev/cdrom /mnt/
~]# cp -rvf /mnt/* /var/usb/rhel7/
============================
~]# cd /var/usb/rhel7/isolinux
isolinux]# chmod +w isolinux.cfg
isolinux]# sed -i 's/timeout 600/timeout 100/g' isolinux.cfg
isolinux]# more isolinux.cfg|grep -A 9 "label linux"
label linux
  menu label ^Install Red Hat Enterprise Linux 7.4
  menu default
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=RHEL-7.4\x20Server.x86_64 inst.ks=hd:LABEL=RHEL-7.4\x20Server.x86_64:/ks.cfg quiet

label check
  menu label Test this ^media & install Red Hat Enterprise Linux 7.4
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=RHEL-7.4\x20Server.x86_64 rd.live.check quiet
# Change these section like this.
```

### *Custom my own image*

``` nohighlight
~]# cat >/etc/yum.repos.d/RHEL-Media.repo<<EOF
[r7-local]
name=RHEL-7-CD
baseurl=file:///var/usb/rhel7/
gpgcheck=1
enabled=1
priority=1
gpgkey=file:///var/usb/rhel7/RPM-GPG-KEY-RHEL-7
EOF
~]# yum clean all
~]# yum makecache
Metadata Cache Created
~]# cd /var/usb/rhel7/Packages
Packages]# wget http://mirror.centos.org/centos/7/os/x86_64/Packages/open-vm-tools-10.2.5-3.el7.x86_64.rpm
Packages]# wget http://mirror.centos.org/centos/7/os/x86_64/Packages/open-vm-tools-desktop-10.2.5-3.el7.x86_64.rpm
Packages]# yum install -y createrepo mkisofs isomd5sum syslinux
# Download iso tools
Packages]# cd ..
rhel7]# createrepo -g repodata/764ce0e938d43b3e9cb1bcca13cf71630aac0c44149f4a61548f642df3c70858-comps-Server.x86_64.xml .
c70858-comps-Server.x86_64.xml .
Spawning worker 0 with 2532 pkgs
Spawning worker 1 with 2531 pkgs
Workers Finished
Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete
rhel7]# yum clean all
rhel7]# yum install open-vm-tools
#Test it.
```

## *Configure Kickstart Install*

```nohighlight
~]# cd /var/usb/rhel7/isolinux
isolinux]# yum install -y system-config-kickstart
isolinux]# system-config-kickstart
```

### *Custom your ks.cfg*

![](https://i.loli.net/2019/08/18/NiJG2l7BSgHyvWe.png)

![](https://i.loli.net/2019/08/22/aSnE7IZXBuiNdPC.jpg)

![](https://i.loli.net/2019/08/18/K1WNP4OGlFILhCS.png)

![](https://i.loli.net/2019/08/18/GFX76hIsiwB9AzO.png)

![](https://i.loli.net/2019/08/18/VcaJwYrvNjkhDOF.png)

![](https://i.loli.net/2019/08/20/YzJ91PCfjvQFKs2.jpg)

![](https://i.loli.net/2019/08/18/KCMOzcEbmnIoT9t.png)

![](https://i.loli.net/2019/08/20/mXZ4EavRCtbV6F9.jpg)

### *Configure packages I need*

```nohighlight
]# cat ~/anaconda-ks.cfg |grep -A 19 %packages >> /var/usb/rhel7/ks.cfg
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

```nohighlight
]# sed -i '/bootloader --location=mbr/a autopart --type=lvm' /var/usb/rhel7/ks.cfg
```

## *Patch up everything*

### *Patch up the iso*

```nohighlight
rhel7]# mkisofs -o /tmp/My-RHEL-7.4.x86_64.iso -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -joliet-long -R -J -v -T /var/usb/rhel7/
Total translation table size: 1282022
Total rockridge attributes bytes: 563634
Total directory bytes: 860160
Path table size(bytes): 218
Done with: The File(s)                             Block(s)    2042346
Writing:   Ending Padblock                         Start Block 2043092
Done with: Ending Padblock                         Block(s)    150
Max brk space used 541000
2043242 extents written (3990 MB)
============================
rhel7]# isohybrid /tmp/My-RHEL-7.4.x86_64.iso 
============================
rhel7]# implantisomd5 /tmp/My-RHEL-7.4.x86_64.iso 
Inserting md5sum into iso image...
md5 = 00878c4f7f9ef89a92905cec5d5578e8
Inserting fragment md5sums into iso image...
fragmd5 = 8f4e15157268afdb9cd75367769a6ca5bf53b9c377c67f491dfb46be4a36
frags = 20
Setting supported flag to 0
# patch up the md5
============================
rhel7]# checkisomd5 /tmp/My-RHEL-7.4.x86_64.iso 
Press [Esc] to abort check.

The media check is complete, the result is: PASS.

It is OK to use this media.
```

### *Patch up the USB*

```nohighlight
~]# dd if=/tmp/My-RHEL-7.4.x86_64.iso of=/dev/sdb
8172968+0 records in
8172968+0 records out
4184559616 bytes (4.2 GB) copied, 94.6365 s, 44.2 MB/s
```

### *Verification results*

![](https://i.loli.net/2019/08/24/QvB4SdpCb9Y6D1c.png)

## *Done*

*Simply tried to complete the unattended installation of RHEL7 by using Kickstart with USB.*
