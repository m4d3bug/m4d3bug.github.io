---
title: 如何為KVM限速
mathjax: true
copyright: true
comment: true
date: 2020-11-30 21:09:21
categories:
- "Ops "
tags:
- "Linux "
- "KVM "
- "Network "
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/linux-kvm-1280x720.png" width=50% /></center>

本文簡單實踐KVM的限速設置並檢驗。

<!--more -->

## 0x00 前言

最開始時老生常談的網速轉換，本次目標是限速150Mbps，遵循以下公式：

```
b is bit, B is Byte
1 Mbps = 0.125 MB/s = 1024 kb/s = 128 KB/s
150 Mbps = 18.75 MB/s = 153600 kb/s = 19200 KB/s
```

爲什麽需要特別提及網速，因爲KVM的XML使用的是KB/s，測速軟件一般是 MB/s。

```
The units for average and peak attributes are kilobytes per second

23.17. Devices Red Hat Enterprise Linux 7 | Red Hat Customer Portal
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-Manipulating_the_domain_xml-Devices#sect-Network_interfaces-Quality_of_service:~:text=The%20units%20for%20average%20and%20peak%20attributes%20are%20kilobytes%20per%20second
```

## 0x01 設置

修改流程如下:

```
[root@rhel7 ~]# virsh list
 Id    Name                           State
----------------------------------------------------
 4     rhel7.6                        running
[root@rhel7 ~]# virsh edit rhel7.6
....
<bandwidth>
    <inbound average='19200' peak='20000'/>
    <outbound average='19200' peak='20000'/>
</bandwidth>
:wq
[root@rhel7 ~]# virsh stop rhel7.6
[root@rhel7 ~]# virsh start rhel7.6
```

## 0x02 測試

### 使用iperf3測速

```
From the KVM host
[root@rhel7 ~]# iperf3 -i 5 -t 60 -c 192.168.122.224                                                                                                         
------------------------------------------------------------
Client connecting to 192.168.122.224, TCP port 5001
TCP window size:  204 KByte (default)
------------------------------------------------------------
[  3] local 192.168.122.1 port 56482 connected with 192.168.122.224 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 5.0 sec  89.1 MBytes   150 Mbits/sec
[  3]  5.0-10.0 sec  87.2 MBytes   146 Mbits/sec
[  3] 10.0-15.0 sec  87.8 MBytes   147 Mbits/sec
[  3] 15.0-20.0 sec  87.2 MBytes   146 Mbits/sec
[  3] 20.0-25.0 sec  87.9 MBytes   147 Mbits/sec
[  3] 25.0-30.0 sec  87.2 MBytes   146 Mbits/sec
[  3] 30.0-35.0 sec  87.8 MBytes   147 Mbits/sec
[  3] 35.0-40.0 sec  87.9 MBytes   147 Mbits/sec
[  3] 40.0-45.0 sec  87.2 MBytes   146 Mbits/sec
[  3] 45.0-50.0 sec  87.8 MBytes   147 Mbits/sec
[  3] 50.0-55.0 sec  87.2 MBytes   146 Mbits/sec
[  3] 55.0-60.0 sec  87.8 MBytes   147 Mbits/sec
[  3]  0.0-60.0 sec  1.03 GBytes   147 Mbits/sec

To the KVM guest
[root@rhel76 ~]# iperf3 -i 5 -s                                                                                                                              
------------------------------------------------------------
Server listening on TCP port 5001
TCP window size: 85.3 KByte (default)
------------------------------------------------------------
[  4] local 192.168.122.224 port 5001 connected with 192.168.122.1 port 56482
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0- 5.0 sec  87.7 MBytes   147 Mbits/sec
[  4]  5.0-10.0 sec  87.6 MBytes   147 Mbits/sec
[  4] 10.0-15.0 sec  87.6 MBytes   147 Mbits/sec
[  4] 15.0-20.0 sec  87.6 MBytes   147 Mbits/sec
[  4] 20.0-25.0 sec  87.6 MBytes   147 Mbits/sec
[  4] 25.0-30.0 sec  87.6 MBytes   147 Mbits/sec
[  4] 30.0-35.0 sec  87.6 MBytes   147 Mbits/sec
[  4] 35.0-40.0 sec  87.6 MBytes   147 Mbits/sec
[  4] 40.0-45.0 sec  87.6 MBytes   147 Mbits/sec
[  4] 45.0-50.0 sec  87.6 MBytes   147 Mbits/sec
[  4] 50.0-55.0 sec  87.6 MBytes   147 Mbits/sec
[  4] 55.0-60.0 sec  87.6 MBytes   147 Mbits/sec
[  4]  0.0-60.1 sec  1.03 GBytes   147 Mbits/sec
```

### 使用qperf

```
From the KVM guest
[root@rhel76 ~]# qperf -ip 19766 -t 60 --use_bits_per_sec 192.168.122.1 tcp_bw
tcp_bw:
	  bw  =  147 Mb/sec

To the KVM host
[root@rhel7 ~]# qperf 
```

## 0x03 結語

XML修改后使用virsh reboot是不生效的。
