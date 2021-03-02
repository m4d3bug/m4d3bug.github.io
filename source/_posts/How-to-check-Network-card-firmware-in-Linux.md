---
title: 如何檢查Linux網卡驅動
mathjax: true
copyright: true
comment: true
date: 2020-12-16 22:31:27
categories:
- "Ops "
tags:
- "Linux "
- "RHEL "
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/PCIENetworkcard.png" width=50% /></center>

本文簡單記錄檢查Linux驅動來源以及相關信息。

<!-- more -->

## 0x00 前言

衆所周知，每個不同Linux發行版維持著不同版本的Kernel分支，而其内部驅動的集成度，在不同時期也不一樣。當問題出現時，檢查驅動的信息，尋求相應的支持/答案，顯得十分必要。本文演示的例子是RHEL 7。

## 0x01 正文

首先使用sosreport收集系統信息。

```bash
# sosreport

sosreport (version 3.9)

This command will collect diagnostic and configuration information from
this Red Hat Enterprise Linux system and installed applications.   

An archive containing the collected information will be generated in
/var/tmp/sos.pd1lwm and may be provided to a Red Hat support
representative.

Any information provided to Red Hat will be treated in accordance with
the published support policies at:
 
  https://access.redhat.com/support/

The generated archive may contain data considered sensitive and its
content should be reviewed by the originating organization before being
passed to any third party.

No changes will be made to system configuration.

Press ENTER to continue, or CTRL-C to quit.

Please enter the case id that you are generating this report for []:

 Setting up archive ...
 Setting up plugins ...

Creating compressed archive...

Your sosreport has been generated and saved in:
  /var/tmp/sosreport-rhel7-2020-12-16-teqlkzl.tar.xz

 Size   13.69MiB
 Owner  root
 md5    73bf64534498f5e97959e52a59217233

Please send this file to your support representative.
```

然後解壓

```bash
# cd /var/tmp/
# tar -Jxvf sosreport-rhel7-2020-12-16-teqlkzl.tar.xz
```

從PCI總綫上去檢查設備型號。

```bash
# grep -EH 'Ethernet|Network' sosreport-*/lspci                                                                               
sosreport-rhel7-2020-12-16-teqlkzl/lspci:03:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [10ec:8168] (rev 15)
sosreport-rhel7-2020-12-16-teqlkzl/lspci:04:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [10ec:8168] (rev 15)
sosreport-rhel7-2020-12-16-teqlkzl/lspci:05:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [10ec:8168] (rev 15)
sosreport-rhel7-2020-12-16-teqlkzl/lspci:06:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [10ec:8168] (rev 15)
sosreport-rhel7-2020-12-16-teqlkzl/lspci:08:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX200 [8086:2723] (rev 1a)
sosreport-rhel7-2020-12-16-teqlkzl/lspci:09:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller [10ec:8125] (rev 04)
```

接著檢查它們的驅動情況。

```bash
# grep -E 'driver|version|bus-info' sosreport*/sos_commands/networking/ethtool_-i_e*
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp3s0:driver: r8168
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp3s0:version: 8.048.00-NAPI
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp3s0:firmware-version:
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp3s0:expansion-rom-version:
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp3s0:bus-info: 0000:03:00.0
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp4s0:driver: r8168
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp4s0:version: 8.048.00-NAPI
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp4s0:firmware-version:
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp4s0:expansion-rom-version:
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp4s0:bus-info: 0000:04:00.0
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp5s0:driver: r8168
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp5s0:version: 8.048.00-NAPI
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp5s0:firmware-version:
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp5s0:expansion-rom-version:
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp5s0:bus-info: 0000:05:00.0
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp6s0:driver: r8168
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp6s0:version: 8.048.00-NAPI
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp6s0:firmware-version:
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp6s0:expansion-rom-version:
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp6s0:bus-info: 0000:06:00.0
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp9s0:driver: r8125
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp9s0:version: 9.003.05-NAPI
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp9s0:firmware-version:
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp9s0:expansion-rom-version:
sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/networking/ethtool_-i_enp9s0:bus-info: 0000:09:00.0
# grep -E 'driver|version|bus-info' sosreport*/sos_commands/networking/ethtool_-i_w*
driver: iwlwifi
version: 3.10.0-1160.6.1.el7.x86_64
firmware-version: 48.4fa0041f.0
expansion-rom-version:
bus-info: 0000:08:00.0
```

然後檢查驅動的來源情況。

```bash
# grep -E 'r8168|iwlwifi|8125' -A9 sosreport-rhel7-2020-12-16-teqlkzl/sos_commands/kernel/modinfo_ALL_MODULES
filename:       /lib/modules/3.10.0-1160.6.1.el7.x86_64/kernel/drivers/net/wireless/intel/iwlwifi/iwlwifi.ko.xz
license:        GPL
author:         Copyright(c) 2003- 2015 Intel Corporation <linuxwifi@intel.com>
description:    Intel(R) Wireless WiFi driver for Linux
firmware:       iwlwifi-100-5.ucode
firmware:       iwlwifi-1000-5.ucode
firmware:       iwlwifi-135-6.ucode
firmware:       iwlwifi-105-6.ucode
firmware:       iwlwifi-2030-6.ucode
firmware:       iwlwifi-2000-6.ucode
firmware:       iwlwifi-5150-2.ucode
firmware:       iwlwifi-5000-5.ucode
firmware:       iwlwifi-6000g2b-6.ucode
firmware:       iwlwifi-6000g2a-6.ucode
firmware:       iwlwifi-6050-5.ucode
firmware:       iwlwifi-6000-6.ucode
firmware:       iwlwifi-7265D-29.ucode
firmware:       iwlwifi-7265-17.ucode
firmware:       iwlwifi-3168-29.ucode
firmware:       iwlwifi-3160-17.ucode
firmware:       iwlwifi-7260-17.ucode
firmware:       iwlwifi-8265-36.ucode
firmware:       iwlwifi-8000C-36.ucode
firmware:       iwlwifi-9260-th-b0-jf-b0-46.ucode
firmware:       iwlwifi-9000-pu-b0-jf-b0-46.ucode
firmware:       iwlwifi-ty-a0-gf-a0-48.ucode
firmware:       iwlwifi-so-a0-gf-a0-48.ucode
firmware:       iwlwifi-so-a0-hr-b0-48.ucode
firmware:       iwlwifi-so-a0-jf-b0-48.ucode
firmware:       iwlwifi-cc-a0-48.ucode
firmware:       iwlwifi-QuQnj-b0-jf-b0-48.ucode
firmware:       iwlwifi-QuZ-a0-jf-b0-48.ucode
firmware:       iwlwifi-QuZ-a0-hr-b0-48.ucode
firmware:       iwlwifi-Qu-b0-jf-b0-48.ucode
firmware:       iwlwifi-Qu-c0-hr-b0-48.ucode
firmware:       iwlwifi-QuQnj-a0-hr-a0-48.ucode
firmware:       iwlwifi-QuQnj-b0-hr-b0-48.ucode
firmware:       iwlwifi-Qu-b0-hr-b0-48.ucode
firmware:       iwlwifi-QuQnj-f0-hr-a0-48.ucode
firmware:       iwlwifi-Qu-a0-jf-b0-48.ucode
firmware:       iwlwifi-Qu-a0-hr-a0-48.ucode
retpoline:      Y
rhelversion:    7.9
srcversion:     7AF1DC6F8F3C429A27DEBB1
alias:          pci:v00008086d00007AF0sv*sd00000A10bc*sc*i*
alias:          pci:v00008086d00007AF0sv*sd00000510bc*sc*i*
alias:          pci:v00008086d00007AF0sv*sd00000310bc*sc*i*
alias:          pci:v00008086d00007AF0sv*sd00000090bc*sc*i*
alias:          pci:v00008086d00007A70sv*sd00000A10bc*sc*i*
alias:          pci:v00008086d00007A70sv*sd00000510bc*sc*i*
--
filename:       /lib/modules/3.10.0-1160.6.1.el7.x86_64/weak-updates/r8125/r8125.ko
version:        9.003.05-NAPI
license:        GPL
description:    Realtek RTL8125 2.5Gigabit Ethernet driver
author:         Realtek and the Linux r8125 crew <netdev@vger.kernel.org>
retpoline:      Y
rhelversion:    7.8
srcversion:     1F956BA08266DA84517213A
alias:          pci:v000010ECd00003000sv*sd*bc*sc*i*
alias:          pci:v000010ECd00008125sv*sd*bc*sc*i*
depends:
vermagic:       3.10.0-1127.el7.x86_64 SMP mod_unload modversions
signer:         The ELRepo Project (http://elrepo.org): ELRepo.org Secure Boot Key
sig_key:        F3:65:AD:34:81:A7:B2:0E:34:27:B6:1B:2A:26:63:5B:83:FE:42:7B
sig_hashalgo:   sha256
sig_hashalgo:   sha256
parm:           speed_mode:force phy operation. Deprecated by ethtool (8). (uint)
parm:           duplex_mode:force phy operation. Deprecated by ethtool (8). (uint)
parm:           autoneg_mode:force phy operation. Deprecated by ethtool (8). (uint)
parm:           advertising_mode:force phy operation. Deprecated by ethtool (8). (uint)
--
filename:       /lib/modules/3.10.0-1160.6.1.el7.x86_64/weak-updates/r8168/r8168.ko
version:        8.048.00-NAPI
license:        GPL
description:    RealTek RTL-8168 Gigabit Ethernet driver
author:         Realtek and the Linux r8168 crew <netdev@vger.kernel.org>
retpoline:      Y
rhelversion:    7.7
srcversion:     5ECBBFD13C1086105CAF750
alias:          pci:v00001186d00004300sv00001186sd00004B10bc*sc*i*
alias:          pci:v000010ECd00002600sv*sd*bc*sc*i*
alias:          pci:v000010ECd00002502sv*sd*bc*sc*i*
alias:          pci:v000010ECd00008161sv*sd*bc*sc*i*
alias:          pci:v000010ECd00008168sv*sd*bc*sc*i*
depends:
--
filename:       /lib/modules/3.10.0-1160.6.1.el7.x86_64/kernel/drivers/net/wireless/intel/iwlwifi/mvm/iwlmvm.ko.xz
license:        GPL
author:         Copyright(c) 2003- 2015 Intel Corporation <linuxwifi@intel.com>
description:    The new Intel(R) wireless AGN driver for Linux
retpoline:      Y
rhelversion:    7.9
srcversion:     C54E61FAAB55BA2A52ED85E
depends:        iwlwifi,mac80211,cfg80211
intree:         Y
vermagic:       3.10.0-1160.6.1.el7.x86_64 SMP mod_unload modversions
signer:         Red Hat Enterprise Linux kernel signing key
sig_key:        3F:FB:02:6D:AD:EF:6E:0B:C4:0
...
```

從上述的singer就可以得知它們的倉庫來源。

## 0x02 總結

事實上，安裝來自第三方驅動的方法林林總總（二進制，RPM或自定義脚本），而紅帽提供的驅動内置于Kernel，因此卸載它們的方法顯而易見就是安裝全新内核。
