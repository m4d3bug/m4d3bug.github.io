---
title: Linux Performance Tuning Study Notes II
mathjax: true
copyright: true
comment: true
date: 2019-09-02 20:11:46
categories:
- "Ops "
tags:
- "Linux "
- "Performance Analysis "
- "Performance Tuning "
- "Load Averages "
photo:
---

*In this post, I am going to markdown what is "load averages"*

## *What is "load averages"*

*We can see the following information in the uptime manual:*

```nohighlight
[root@localhost ~]# man uptime
...
  System load averages is the average number of processes that are either in a runnable or uninterruptable state. 
###平均負載是指單位時間內，系統處於可運行態和不可中斷態的平均進程數。
  A process in a runnable state is either using the CPU or waiting to use the CPU. 
###可運行態則正使用CPU或等待CPU。
  A process in uninterruptable state is waiting for some I/O access, eg waiting for disk. 
###不可中斷態則正處於內核態關鍵流程，萬不可打斷，諸如等待磁盤I/O響應。
###“不可中斷態實際上是系統對進程和硬件設備的一種保護機制。”
  The averages are taken over the three time intervals.
###參數取自三個時間間隔：1min、5min、15min
  Load averages are not normalized for the number of CPUs in a system, so a load average of 1 means a single CPU system is loaded all the time while on a 4 CPU system it means it was idle 75% of the time.
###一直穩定：1min≈5min≈15min
###過去高負：1min  <<  15min  
###目前高負：1min  >>  15min
```

### *load averages = runnable(可運行態) + uninterruptable(不可中斷態)*

- *System load averages ↑ = Using CPU ↑ + Waiting CPU + Waiting I/O*

- *System load averages ↑ = Using CPU + Waiting CPU  ↑ + Waiting I/O*

- *System load averages ↑ = Using CPU + Waiting CPU  + Waiting I/O ↑*

  因此，平均負載和 CPU 使用率不一定掛鉤。

###  *R+ = runnable  ↓  D+ = Disk Sleep(uninterruptable)   ↓*

```nohighlight
[root@localhost ~]# ps -aux
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      22417  0.0  0.3  39008  3628 pts/0    R+   21:40   0:00 ps -aux
root      22418  0.0  0.1  22016  1624 pts/0    D+   21:40   0:00 -bash
```

### *what if load average = 1*：

 *CPU NUM = grep 'model name' /proc/cpuinfo | wc -l*

- *1 CPU system: loaded all the time.*

- *4 CPU system: 1/4 loaded all the time.*

#### *PS: Load averages should be smaller than the number of CPUs 70%.*

## *Case simulation*

### *Prerequisites*

- *test tools              >   stress - tool to impose load on and stress test systems.*  
- analyzing tools   >   *sysstat - sysstat configuration file.(only use mpstat[CPU] and pidstat[pid])*

```nohighlight
[root@localhost ~]# screenfetch 
                   ..                    root@localhost
                 .PLTJ.                  OS: CentOS 
                <><><><>                 Kernel: x86_64 Linux 3.10.0-957.21.3.el7.x86_64
       KKSSV' 4KKK LJ KKKL.'VSSKK        Uptime: 48m
       KKV' 4KKKKK LJ KKKKAL 'VKK        Packages: 494
       V' ' 'VKKKK LJ KKKKV' ' 'V        Shell: bash 4.2.46
       .4MA.' 'VKK LJ KKV' '.4Mb.        CPU: Intel Xeon E5-26xx v4 @ 2x 2.394GHz
     . KKKKKA.' 'V LJ V' '.4KKKKK .      GPU: cirrusdrmfb
   .4D KKKKKKKA.'' LJ ''.4KKKKKKK FA.    RAM: 147MiB / 7821MiB
  <QDD ++++++++++++  ++++++++++++ GFD>  
   'VD KKKKKKKK'.. LJ ..'KKKKKKKK FV    
     ' VKKKKK'. .4 LJ K. .'KKKKKV '     
        'VK'. .4KK LJ KKA. .'KV'        
       A. . .4KKKK LJ KKKKA. . .4       
       KKA. 'KKKKK LJ KKKKK' .4KK       
       KKSSA. VKKK LJ KKKV .4SSKK       
                <><><><>                
                 'MKKM' 
                   ''
[root@VM_0_15_centos ~]# yum install -y stress sysstat
```

### *Using CPU*

- #### *Windows 1*

  ```nohighlight
  [root@localhost ~]# stress --cpu 1 --timeout 600
  stress: info: [19751] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hd
  ```

  *Will stop util cpu 1 up to 100%.*

- #### *Windows 2*

  ```nohighlight
  [root@localhost ~]# watch -d uptime
  Every 2.0s: uptime                                            Mon Sep  2 23:39:47 2019
  
   23:39:47 up  1:09,  4 users,  load average: 0.85, 0.68, 0.36
  ```

  *load average of 1 min will close to 1.00.*

- #### *Windows 3*

  ```nohighlight
  [root@localhost ~]# mpstat -P ALL 5
  11:41:18 PM  CPU %usr  %nice  %sys %iowait %irq  %soft  %steal  %guest  %gnice  %idle
  11:41:23 PM  all 50.15  0.00  0.20  0.10   0.00   0.10    0.00    0.00    0.00  49.45
  11:41:23 PM    0 100.00 0.00  0.00  0.00   0.00   0.00    0.00    0.00    0.00    0.00
  11:41:23 PM    1  0.20  0.00  0.40  0.00   0.00   0.00    0.00    0.00    0.00   99.40
  
  11:41:23 PM  CPU %usr  %nice  %sys %iowait %irq  %soft  %steal  %guest  %gnice  %idle
  11:41:28 PM  all 50.25  0.00  0.30  0.00   0.00   0.00    0.00    0.00    0.00   49.45
  11:41:28 PM    0 94.81  0.00  0.20  0.00   0.00   0.00    0.00    0.00    0.00    4.99
  11:41:28 PM    1  5.61  0.00  0.40  0.00   0.00   0.00    0.00    0.00    0.00   93.99
  
  11:41:28 PM  CPU %usr  %nice  %sys %iowait %irq  %soft  %steal  %guest  %gnice  %idle
  11:41:33 PM  all 50.25  0.00  0.20  0.10   0.00   0.00    0.00    0.00    0.00   49.45
  11:41:33 PM    0 98.60  0.00  0.00  0.00   0.00   0.00    0.00    0.00    0.00    1.40
  11:41:33 PM    1  1.80  0.00  0.40  0.20   0.00   0.00    0.00    0.00    0.00   97.60
  ```

  *Only one cpu's %usr is rising.*

- #### *Windows 4*

  ```nohighlight
  [root@localhost ~]# pidstat -u 5
  11:50:48 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
  11:50:53 PM     0      6099    0.00    0.20    0.00    0.20     0  barad_agent
  11:50:53 PM     0      6100    0.20    0.00    0.00    0.20     0  barad_agent
  11:50:53 PM     0     12526    0.20    0.00    0.00    0.20     1  YDService
  11:50:53 PM     0     12760    0.00    0.20    0.00    0.20     1  YDLive
  11:50:53 PM     0     20334    0.00    0.20    0.00    0.20     1  pidstat
  11:50:53 PM     0     22927  100.00    0.00    0.00  100.00     0  stress
  ```

  *Now we can clearly see load average up to 1 because of using CPU.*



