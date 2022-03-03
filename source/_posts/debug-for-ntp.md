---
layout: posts
title: 時鐘源的一次debug
mathjax: true
copyright: true
comment: true
date: 2022-03-02 23:27:15
categories: "Ops "
tags:
- "Linux "
- "NTP "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/ntp.png" width=50% /></center>

## 0x00 背景
ntp不能同步。简单记录一下如何debug。

<!-- more -->

## 0x01 信息收集
按照国际惯例，收一份sosreport检查。

### 系统信息
```bash
Red Hat Enterprise Linux Server release 7.6 (Maipo)
ntp-4.2.6p5-28.el7.x86_64

% cat dmidecode | grep -A 5 "System I"
System Information
	Manufacturer: XXXXXX
	Product Name: XXXXXX
	Version: RHEL XXXXXX
	Serial Number: XXXXXXXXXXXXXXXXX
	UUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```
### NTP侧的检查
```bash
% grep -v ^# etc/ntp.conf | grep -v ^$
driftfile /var/lib/ntp/drift
restrict default nomodify notrap nopeer noquery
restrict 127.0.0.1 
restrict ::1
server ntp01.xxx.com iburst
server ntp02.xxx.com iburst
server ntp03.xxx.com iburst 
includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
disable monitor
tinker panic 0

% cat systemctl_status_--all 
~~~
* ntpd.service - Network Time Service
   Loaded: loaded (/usr/lib/systemd/system/ntpd.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-02-17 17:16:59 HKT; 4 days ago
  Process: 208943 ExecStart=/usr/sbin/ntpd -u ntp:ntp $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 208944 (ntpd)
   CGroup: /system.slice/ntpd.service
           `-208944 /usr/sbin/ntpd -u ntp:ntp -4 -g

Feb 17 17:16:59 xxx ntpd[208944]: proto: precision = 0.065 usec
Feb 17 17:16:59 xxx ntpd[208944]: 0.0.0.0 c01d 0d kern kernel time sync enabled
Feb 17 17:16:59 xxx ntpd[208944]: restrict: error in address '::1' on line 15. Ignoring...
Feb 17 17:16:59 xxx ntpd[208944]: ntp_io: estimated max descriptors: 1024, initial socket boundary: 16
Feb 17 17:16:59 xxx ntpd[208944]: Listen and drop on 0 v4wildcard 0.0.0.0 UDP 123
Feb 17 17:16:59 xxx ntpd[208944]: Listen normally on 1 lo 127.0.0.1 UDP 123
Feb 17 17:16:59 xxx ntpd[208944]: Listen normally on 2 eth0 192.168.24.74 UDP 123
Feb 17 17:16:59 xxx ntpd[208944]: Listening on routing socket on fd #19 for interface updates
Feb 17 17:16:59 xxx ntpd[208944]: 0.0.0.0 c016 06 restart
Feb 17 17:16:59 xxx ntpd[208944]: 0.0.0.0 c012 02 freq_set kernel 0.919 PPM
~~~
```
### OS侧的NTP相关
```bash
% cat ntpq_-c_as 

ind assid status  conf reach auth condition  last_event cnt
===========================================================
  1 31017  9024   yes   yes  none    reject   reachable  2
  2 31018  90a4   yes   yes  none    reject   reachable 10
  3 31019  9024   yes   yes  none    reject   reachable  2

% cat ntpq_-pn 
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 192.168.8.127    192.168.8.80      2 u   34   64  377    0.507  675.400   0.125         
 192.168.8.1      192.168.8.127     3 u   40   64  377    0.826  675.575   0.087
 192.168.80.1     192.168.8.127     3 u   24   64  377    1.947  675.562   0.269

% cat ntpq_-c_rv_31017 
associd=31017 status=9024 conf, reach, sel_reject, 2 events, reachable,
srcadr=xxx.xxx.com, srcport=123, dstadr=192.168.24.74,
dstport=123, leap=00, stratum=2, precision=-23, rootdelay=0.000,
rootdisp=10139.175, refid=192.168.8.80,
reftime=e5be920d.1bac5634  Tue, Feb 22 2022  6:34:53.108,
rec=e5bec106.37073922  Tue, Feb 22 2022  9:55:18.214, reach=377,
unreach=0, hmode=3, pmode=4, hpoll=6, ppoll=6, headway=28,
flash=400 peer_dist, keyid=0, offset=675.400, delay=0.507,
dispersion=0.937, jitter=0.125, xleave=0.030,
filtdelay=     0.51    0.58    0.59    0.54    0.47    0.44    0.67    0.49,
filtoffset=  675.40  675.40  675.35  675.35  675.30  675.21  675.26  675.20,
filtdisp=      0.00    0.98    1.94    2.90    3.90    4.89    5.85    6.84

% cat ntpq_-c_rv_31018 
associd=31018 status=90a4 conf, reach, sel_reject, 10 events, reachable,
srcadr=xxx.xxx.com, srcport=123, dstadr=192.168.24.74,
dstport=123, leap=00, stratum=3, precision=-23, rootdelay=1.465,
rootdisp=10159.561, refid=192.168.8.127,
reftime=e5bebf79.036eb697  Tue, Feb 22 2022  9:48:41.013,
rec=e5bec100.429fa5b0  Tue, Feb 22 2022  9:55:12.260, reach=377,
unreach=0, hmode=3, pmode=4, hpoll=6, ppoll=6, headway=22,
flash=400 peer_dist, keyid=0, offset=675.575, delay=0.826,
dispersion=0.945, jitter=0.087, xleave=0.030,
filtdelay=     0.83    0.98    0.89    0.86    0.96    0.91    0.96    0.92,
filtoffset=  675.57  675.58  675.54  675.50  675.46  675.51  675.47  675.44,
filtdisp=      0.00    0.98    1.97    2.96    3.92    4.88    5.88    6.84

% cat ntpq_-c_rv_31019 
associd=31019 status=9024 conf, reach, sel_reject, 2 events, reachable,
srcadr=xxx.xxx.com, srcport=123,
dstadr=192.168.24.74, dstport=123, leap=00, stratum=3, precision=-23,
rootdelay=1.984, rootdisp=10148.712, refid=192.168.8.127,
reftime=e5bec10d.14a4aa6e  Tue, Feb 22 2022  9:55:25.080,
rec=e5bec110.3813c9ec  Tue, Feb 22 2022  9:55:28.219, reach=377,
unreach=0, hmode=3, pmode=4, hpoll=6, ppoll=6, headway=38,
flash=400 peer_dist, keyid=0, offset=675.562, delay=1.947,
dispersion=0.955, jitter=0.269, xleave=0.030,
filtdelay=     1.95    2.29    1.88    2.18    2.23    1.73    1.96    2.14,
filtoffset=  675.56  675.31  675.29  675.30  675.37  675.18  675.27  675.41,
filtdisp=      0.00    1.01    1.98    2.94    3.93    4.89    5.88    6.89

% cat ntptime 
ntp_gettime() returns code 5 (ERROR)
  time e5bec128.a9fe9000  Tue, Feb 22 2022  9:55:52.664, (.664041),
  maximum error 16000000 us, estimated error 16000000 us, TAI offset 0
ntp_adjtime() returns code 5 (ERROR)
  modes 0x0 (),
  offset 0.000 us, frequency 0.919 ppm, interval 1 s,
  maximum error 16000000 us, estimated error 16000000 us,
  status 0x4041 (PLL,UNSYNC,MODE),
  time constant 7, precision 1.000 us, tolerance 500 ppm,
```
### OS侧的时间检查
```bash
% cat netstat_-W_-neopa | grep ntp
udp        0      0 192.168.24.74:123        0.0.0.0:*                           0          58844017   208944/ntpd          off (0.00/0/0)
udp        0      0 127.0.0.1:123           0.0.0.0:*                           0          58844016   208944/ntpd          off (0.00/0/0)
udp        0      0 0.0.0.0:123             0.0.0.0:*                           0          58844013   208944/ntpd          off (0.00/0/0)
unix  2      [ ]         DGRAM                    58844008 208944/ntpd          

% cat timedatectl 
      Local time: Tue 2022-02-22 09:56:06 HKT
  Universal time: Tue 2022-02-22 01:56:06 UTC
        RTC time: Tue 2022-02-22 01:56:05
       Time zone: Asia/Hong_Kong (HKT, +0800)
     NTP enabled: yes
NTP synchronized: no
 RTC in local TZ: no
      DST active: n/a

~~~
Jan 14 03:01:51 xxx ntpd_intres[6850]: host name not found: ntp01.xxx.com
Jan 14 03:01:51 xxx ntpd_intres[6850]: host name not found: ntp02.xxx.com
Jan 14 03:01:51 xxx ntpd_intres[6850]: host name not found: ntp03.xxx.com
Jan 14 03:01:54 xxx ntpd[6832]: Listen normally on 2 eth0 192.168.24.74 UDP 123
Jan 14 03:01:54 xxx ntpd[6832]: new interface(s) found: waking up resolver
Jan 14 03:01:56 xxx ntpd_intres[6850]: DNS ntp01.xxx.com -> 192.168.8.127
Jan 14 03:01:56 xxx ntpd_intres[6850]: DNS ntp02.xxx.com -> 192.168.8.1
Jan 14 03:01:56 xxx ntpd_intres[6850]: DNS ntp03.xxx.com -> 192.168.80.1
...
Feb 17 17:16:59 xxx ntpd[208943]: ntpd 4.2.6p5@1.2349-o Mon Oct  9 16:33:12 UTC 2017 (1)
-- Subject: Unit ntpd.service has finished start-up
-- Unit ntpd.service has finished starting up.
Feb 17 17:16:59 xxx ntpd[208944]: proto: precision = 0.065 usec
Feb 17 17:16:59 xxx ntpd[208944]: 0.0.0.0 c01d 0d kern kernel time sync enabled
Feb 17 17:16:59 xxx ntpd[208944]: restrict: error in address '::1' on line 15. Ignoring...
Feb 17 17:16:59 xxx ntpd[208944]: ntp_io: estimated max descriptors: 1024, initial socket boundary: 16
Feb 17 17:16:59 xxx ntpd[208944]: Listen and drop on 0 v4wildcard 0.0.0.0 UDP 123
Feb 17 17:16:59 xxx ntpd[208944]: Listen normally on 1 lo 127.0.0.1 UDP 123
Feb 17 17:16:59 xxx ntpd[208944]: Listen normally on 2 eth0 192.168.24.74 UDP 123
Feb 17 17:16:59 xxx ntpd[208944]: Listening on routing socket on fd #19 for interface updates
Feb 17 17:16:59 xxx ntpd[208944]: 0.0.0.0 c016 06 restart
Feb 17 17:16:59 xxx ntpd[208944]: 0.0.0.0 c012 02 freq_set kernel 0.919 PPM
~~~
```
## 0x02 debug
汇总分析一下看到的报错error。

```bash
restrict ::1

Feb 17 17:16:59 xxx ntpd[208944]: restrict: error in address '::1' on line 15. Ignoring...
```
1. 系统的ipv6被关闭了，但配置文件还是开着ipv6监听。可以关闭或者忽略。

```bash
% cat ntptime 
ntp_gettime() returns code 5 (ERROR)
  time e5bec128.a9fe9000  Tue, Feb 22 2022  9:55:52.664, (.664041),
  maximum error 16000000 us, estimated error 16000000 us, TAI offset 0
ntp_adjtime() returns code 5 (ERROR)
  modes 0x0 (),
  offset 0.000 us, frequency 0.919 ppm, interval 1 s,
  maximum error 16000000 us, estimated error 16000000 us,
  status 0x4041 (PLL,UNSYNC,MODE),
  time constant 7, precision 1.000 us, tolerance 500 ppm,
```
2.  信息量较多
   * 这证明`ntp`和`chronyd`重启了，但依旧没有完成同步。有可能几分钟后消失，但如果没有，应该检查`ntpq`或者`chronyc`。
   * Linux 内核支持精确计时规则，ntptime 命令在启用此模式时提供与时间相关的内核变量信息，对`ntpd`守护进程使用`-x`选项会导致此模式被禁用，因此`ntptime`命令提供代码`5 ERROR`。
   * 由于某些原因（DNS、网络、不可靠的NTP服务器等），`ntpd/chronyd`没有进行同步。
   * 而`maxerror`和`esterror`可以参考 相关的`RFCs`: 
> Network Time Protocol (Version 3) Specification, Implementation and Analysis (RFC1305)   
> A  Kernel Model for Precision Timekeeping (1589) 

```bash
flash=400 peer_dist
```
3. 这个更是重量级信息。
   * 这意味着没有其他的NTP server提供可靠的时间。
   * NTP正在拒绝上游并且回显400。

## 0x03 解决方法
了解到了相应的问题所在，那解决办法有两条路，我建议都去做。

* 上调ntp的容忍度，通过设置maxdist直至timedatectl显示正常，比较高的值是30。

```bash
The synchronization distance, defined as one-half the delay plus the dispersion, represents the maximum error statistic. The jitter represents the expected error statistic. The maximum error and expected error calculated from 
the peer variables represents the quality metric for the server. The maximum error and expected error calculated from the system variables represents the quality metric for the client. If the root synchronization distance for 
any server exceeds 1.5 s, called the select threshold, the server is considered invalid
```
```bash
maxdist maxdistance
   Specify the synchronization distance threshold used by the clock selec‐
   tion algorithm. The default is 1.5 s. This determines both the  minimum
   number  of  packets  to  set the system clock and the maximum roundtrip
   delay. It can be decreased to improve reliability or increased to syn‐
   chronize clocks on the Moon or planets.
```
* 调查各个时钟源，并且尝试不要只用一个篮子里的时钟源作为依据。尤其是时钟敏感的交易，卫星原子钟更是必须。

## 0x04 总结
简单演示了如何从一个表面现象debug的过程，问题之初，要尽可能地广地收集信息，sosreport就是一个这样专业的工具。之后针对手上有的信息线索，一步一步地筛选出关联问题现象的证据，然后通过这些证据，结合现象去解读得出答案。

