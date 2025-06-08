---
title: iPXE+Kickstart搭建無人值守自動化安裝環境
mathjax: true
copyright: true
comment: true
date: 2020-09-12 00:37:40
categories:
- "Ops "
tags:
- "Kickstart "
- "TFTP "
- "iPXE "
- "Httpd "
- "RHEL 7 "
- "Linux "
- "BIOS "
- "UEFI "
- "Anaconda "
- "Dnsmasq "
- "VMware "
---
<center><img src="https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/iPXE.png" width=50% /></center>

本文將實踐通過燒錄iPXE到相應的VMware虛擬機中，來模擬一個無PXE的自動化安裝環境。

<!-- more -->
在閲讀本文前，請確保您有足夠瞭解[iPXE和PXE的~~情情愛愛~~恩恩怨怨。](https://groups.google.com/g/ustc_lug/c/P2jOQ5F4EKY?pli=1)

## 0x00 相關背景

iPXE有以下的使用方法，不像PXE那麽死板：

- pxe(網卡) -> ipxe -> pxelinux.bin -> menu.c32
- pxe -> ipxelinux.0 -> menu.c32
- pxe -> syslinux.bin -> ipxe -> pxelinux.bin -> menu.c32
- pxe -> syslinux.bin -> ipxelinux.0 -> menu.c32
- pxe -> ipxe.lkrn(ipxe.efi/undionly.kpxe)  -> boot.ipxe
- grub -> ipxe.lkrn(ipxe.efi/undionly.kpxe) -> pxelinux.bin -> menu.c32
- grub -> ipxelinux.lkrn -> menu.c32
- ipxe(燒入網卡) -> pxelinux.bin
- ipxe(燒入網卡) -> boot.ipxe
- ...

本文實踐的則是最後一種方法。ipxe(燒入網卡) -> boot.ipxe，相較於前文，你將會看見Dnsmasq日志的明顯區別。

## 0x01 創建VM

VMware虛擬機的vmx文件，支持自定義rom及mrom，這些[固件的替換](https://ipxe.org/howto/vmware)，使其具有ipxe的功能。

### 下載並編譯

```bash
# git clone git://git.ipxe.org/ipxe.git
# cd ipxe/
# make bin/8086100f.mrom bin/808610d3.mrom bin/10222000.rom bin/15ad07b0.rom
```

### 啓用iPXE固件

圖形化創建一個無CD的空VM，並確保其與iPXE主機相同LAN。

隨後移動相關固件到該空VM目錄下。

```bash
# mv bin/{8086100f.mrom,808610d3.mrom,10222000.rom,15ad07b0.rom} /mnt/g/Virtual Machines/iPXE-test-7
# ls -alt /mnt/g/Virtual Machines/iPXE-test-7                                                                                                                                                                          22:01:05
total 8436196
-rwxrwxrwx 1 m4d3bug m4d3bug 4337106944 Sep 11 22:02 iPXE-test-7.vmdk*
-rwxrwxrwx 1 m4d3bug m4d3bug     577406 Sep 11 21:36 vmware.log*
drwxrwxrwx 1 m4d3bug m4d3bug        512 Sep 10 22:35 ../
drwxrwxrwx 1 m4d3bug m4d3bug        512 Sep 10 20:52 ./
-rwxrwxrwx 1 m4d3bug m4d3bug       3035 Sep 10 20:52 iPXE-test-7.vmx*
-rwxrwxrwx 1 m4d3bug m4d3bug       8684 Sep 10 20:52 iPXE-test-7.nvram*
drwxrwxrwx 1 m4d3bug m4d3bug        512 Sep 10 14:04 iPXE-test-7.vmdk.lck/
-rwxrwxrwx 1 m4d3bug m4d3bug 4294967296 Sep 10 14:04 564dc457-c101-4841-4883-ecf173ce702f.vmem*
drwxrwxrwx 1 m4d3bug m4d3bug        512 Sep 10 14:04 564dc457-c101-4841-4883-ecf173ce702f.vmem.lck/
-rwxrwxrwx 1 m4d3bug m4d3bug     198523 Sep 10 13:05 vmware-0.log*
-rwxrwxrwx 1 m4d3bug m4d3bug      66560 Sep 10 11:14 15ad07b0.rom*
-rwxrwxrwx 1 m4d3bug m4d3bug      67584 Sep 10 11:14 10222000.rom*
-rwxrwxrwx 1 m4d3bug m4d3bug      68608 Sep 10 11:14 808610d3.mrom*
-rwxrwxrwx 1 m4d3bug m4d3bug      68608 Sep 10 11:14 8086100f.mrom*
drwxrwxrwx 1 m4d3bug m4d3bug        512 Sep 10 11:07 ipxe/
-rwxrwxrwx 1 m4d3bug m4d3bug        266 Sep 10 10:51 iPXE-test-7.vmxf*
-rwxrwxrwx 1 m4d3bug m4d3bug          0 Sep 10 10:51 iPXE-test-7.vmsd*
drwxrwxrwx 1 m4d3bug m4d3bug        512 Sep 10 10:51 iPXE-test-7.vmx.lck/
```

然後修改.vmx文件增加以下：

```bash
# cd  /mnt/g/Virtual Machines/iPXE-test-7
# vim iPXE-test-7.vmx
……
ethernet0.opromsize = 262144
e1000bios.filename = "8086100f.mrom"
e1000ebios.filename = "808610d3.mrom"
nbios.filename = "10222000.rom"
nx3bios.filename = "15ad07b0.rom"
```

## 0x02 搭建iPXE环境

[不再赘述](https://blog.madebug.net/Ops/2020-08-08-hello-iPXE-over-PXE-Kickstart)

## 0x03 檢查日誌

從日誌中可以清晰看到僅調用了boot.ipxe，相較於上篇文章，少了iPXE引導程序的加載。

![https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/Dnsmasqlogs.jpg](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/Dnsmasqlogs.jpg)

## 0x04 結語

本文簡單嘗試了iPXE的加載順序，模擬驗證了iPXE的硬件條件。現在是容器大行其道的時代，批量部署環境只是幾秒鐘的事。但是在與之對應的物理機時代，iPXE仍有它的地位與話語權。
