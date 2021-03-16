---
title: Linux性能調優實戰筆記II
mathjax: true
copyright: true
comment: true
date: 2019-09-02 20:11:46
updated: 2020-07-11 18:33:33
categories:
- "Ops "
tags:
- "Linux "
- "Performance Analysis "
- "Performance Tuning "
- "Load Averages "
- "Sysstat "
- "Stress "
- "Linux性能調優實戰筆記"
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/linux-tux-minimalism-4k-42-1280x800.jpg" width=50% /></center>       

本文旨在阐述，关于平均负载的知识点

<!-- more -->

## 0x00 什么是平均负载(Load Averages)

---

平均负载依赖于有多少空闲核心，运行态，不可中断态。我们可以从uptime的manual中查看到以下讯息：

>man uptime
>...
>System load averages is the average number of processes that are either in a runnable or uninterruptable state. (平均负载：单位时间内，系统处于运行态和不可中断态的进程数。)
>A process in a runnable state is either using the CPU or waiting to use the CPU.(运行态指正使用CPU或等待CPU。)
>A process in uninterruptable state is waiting for some I/O access, eg waiting for disk.(不可中断态指正处于内核态关键流程，万不可打断，诸如等待磁盘I/O响应。“不可中断态指系统对进程和硬件设备的保护机制。”)
>The averages are taken over the three time intervals.(参数取自三个时间间隔：1min、5min、15min)
>Load averages are not normalized for the number of CPUs in a system, so a load average of 1 means a single CPU system is loaded all the time while on a 4 CPU system it means it was idle 75% of the time.
>##一直稳定：1min≈5min≈15min
>##过去高负：1min  <<  15min  
>##目前高负：1min  >>  15min

- **运行态(runnable)+不可中断态(uninterruptable)**

  以下算式直观描述影响平均负载的可能因素(CPU占用，CPU等待，IO等待)：

  - *System Load Averages ↑ = Using CPU ↑ + Waiting CPU + Waiting I/O*
  - *System Load Averages ↑ = Using CPU + Waiting CPU  ↑ + Waiting I/O*
  - *System Load Averages ↑ = Using CPU + Waiting CPU  + Waiting I/O ↑*

- **R+ = running↓ D+ = Disk Sleep(uninterruptable sleep)↓**

  ```nohighlight
  [root@localhost ~]# ps -aux
  USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
  root      22417  0.0  0.3  39008  3628 pts/0    R+   21:40   0:00 ps -aux
  root      22418  0.0  0.1  22016  1624 pts/0    D+   21:40   0:00 -bash
  ```

- **当平均负载 = 1时：**

  这需要结合CPU数来进行判断CPU NUM = grep 'model name' /proc/cpuinfo | wc -l

  - 1个CPU系统：满载
  - 4个CPU系统：1/4满载

- **注意：平均负载应该小于CPU数的70%。** 

## 0x01 案例模拟

---

### 测试説明

- 测试工具：[**stress**](https://linux.die.net/man/1/stress)
- 分析工具：[**sysstat**](https://man7.org/linux/man-pages/man5/sysstat.5.html) (仅使用 mpstat[CPU] 和 pidstat[pid])

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
[root@localhost ~]# wget http://download-ib01.fedoraproject.org/pub/fedora/linux/releases/30/Everything/x86_64/os/Packages/s/sysstat-11.7.3-3.fc30.x86_64.rpm
[root@localhost ~]# yum install -y stress sysstat-11.7.3-3.fc30.x86_64.rpm
```

### CPU占用

#### Windows 1

  施加一个持续**10min的一个CPU占用**。
  
  ```nohighlight
  [root@localhost ~]# stress --cpu 1 --timeout 600
  stress: info: [19751] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hd
  ```

#### Windows 2

  每两秒**输出uptime**命令的结果。

  ```nohighlight
  [root@localhost ~]# watch -d uptime
  Every 2.0s: uptime                                      Mon Sep  2 23:39:47 2019
  
  23:39:47 up  1:09,  4 users,  load average: 0.85, 0.68, 0.36
  ```

#### Windows 3

  使用mpstat来每5s输出，可以看到**%usr列显着升高。**
  
  ```nohighlight
    [root@localhost ~]# mpstat -P ALL 5
    11:41:18 PM  CPU %usr  %nice  %sys %iowait %irq  %soft  %steal  %guest  %gnice  %idle
    11:41:23 PM  all 50.15  0.00  0.20  0.10   0.00   0.10    0.00    0.00    0.00  49.45
  >>11:41:23 PM    0 100.00 0.00  0.00  0.00   0.00   0.00    0.00    0.00    0.00   0.00
    11:41:23 PM    1  0.20  0.00  0.40  0.00   0.00   0.00    0.00    0.00    0.00  99.40
    
    11:41:23 PM  CPU %usr  %nice  %sys %iowait %irq  %soft  %steal  %guest  %gnice  %idle
    11:41:28 PM  all 50.25  0.00  0.30  0.00   0.00   0.00    0.00    0.00    0.00  49.45
  >>11:41:28 PM    0 94.81  0.00  0.20  0.00   0.00   0.00    0.00    0.00    0.00   4.99
    11:41:28 PM    1  5.61  0.00  0.40  0.00   0.00   0.00    0.00    0.00    0.00  93.99
    
    11:41:28 PM  CPU %usr  %nice  %sys %iowait %irq  %soft  %steal  %guest  %gnice  %idle
    11:41:33 PM  all 50.25  0.00  0.20  0.10   0.00   0.00    0.00    0.00    0.00  49.45
  >>11:41:33 PM    0 98.60  0.00  0.00  0.00   0.00   0.00    0.00    0.00    0.00   1.40
    11:41:33 PM    1  1.80  0.00  0.40  0.20   0.00   0.00    0.00    0.00    0.00  97.60             
  ```

#### Windows 4

  现在我们可以看见平均负载的**升高是因为CPU占用**。

  ```nohighlight
    [root@localhost ~]# pidstat -u 5
    10:40:09 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
    10:40:14 PM     0      2853    0.20    0.20    0.00    0.40    0.40     0  YDService
    10:40:14 PM     0      3303    0.00    0.20    0.00    0.00    0.20     0  sshd
    10:40:14 PM     0      3750    0.40    0.40    0.00    0.00    0.80     0  barad_agent
    10:40:14 PM     0      4179    0.20    0.00    0.00    0.00    0.20     0  watch
  >>10:40:14 PM     0     18043  100.00    0.00    0.00    0.00  100.00     1  stress
  ```

### IO等待

#### Windows 1

  施加一个持续**10分钟的io写入。**
  
  ``` nohighlight
  [root@localhost ~]# stress -i 1 --timeout 600
  stress: info: [15703] dispatching hogs: 0 cpu, 1 io, 0 vm, 0 hdd
  ```

#### Windows 2

  Load Average**一分钟内数值飙升至1.06。**

  ``` nohighlight
  [root@localhost ~]# watch -d uptime
  Every 2.0s: uptime                                       Tue Sep  3 21:34:11 2019
  
   21:34:11 up 6 min,  4 users,  load average: 1.05, 0.62, 0.27
  ```

#### Windows 3

  只有一个CPU的**%iowait上升。**

  ```nohighlight
    [root@localhost ~]# mpstat -P ALL 5 3
    09:36:05 PM  CPU %usr %nice  %sys %iowait %irq  %soft  %steal  %guest  %gnice  %idle
    09:36:10 PM  all 0.50  0.00 29.03  21.27  0.00   0.00    0.00    0.00    0.00  49.19
    09:36:10 PM    0 0.81  0.00 25.15  17.44  0.00   0.00    0.00    0.00    0.00  56.59
  >>09:36:10 PM    1 0.20  0.00 32.73  25.10  0.00   0.00    0.00    0.00    0.00  41.97
    
    09:36:10 PM  CPU %usr %nice  %sys %iowait %irq  %soft  %steal  %guest  %gnice  %idle
  >>09:36:15 PM  all 0.30  0.00 29.43  21.42  0.00   0.00    0.00    0.00    0.00  48.85
    09:36:15 PM    0 0.40  0.00 20.61  15.15  0.00   0.00    0.00    0.00    0.00  63.84
    09:36:15 PM    1 0.00  0.00 38.29  27.58  0.00   0.00    0.00    0.00    0.00  34.13
    
    09:36:15 PM  CPU %usr %nice  %sys %iowait %irq  %soft  %steal  %guest  %gnice  %idle
    09:36:20 PM  all 0.30  0.00  28.61 21.54  0.00   0.00    0.00    0.00    0.00  49.54
    09:36:20 PM    0 0.41  0.00  13.18  9.13  0.00   0.00    0.00    0.00    0.00  77.28
  >>09:36:20 PM    1 0.20  0.00  43.75 33.87  0.00   0.00    0.00    0.00    0.00  22.18
  ```

#### Windows 4

  Load Average**上升是因为等待I/O。**

  ``` nohighlight
    [root@localhost ~]# pidstat -u 5 1
    10:37:29 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
    10:37:34 PM     0      1232    0.00    0.40    0.00    0.00    0.40     0 kworker/0:1H
    10:37:34 PM     0      2853    0.40    0.40    0.00    0.60    0.80     1  YDService
    10:37:34 PM     0      3750    0.20    0.00    0.00    0.00    0.20     0  barad_agent
    10:37:34 PM     0     16458    0.00    0.20    0.00    0.00    0.20     0  pidstat
  >>10:37:34 PM     0     17460    0.00   85.20    0.00    0.40   85.20     1  stress 
  ```

### CPU等待

#### Windows 1

  ``` nohighlight
  [root@localhost ~]# stress --cpu 8 --timeout 600
  stress: info: [9168] dispatching hogs: 8 cpu, 0 io, 0 vm, 0 hdd
  ```

#### Windows 2

  Load Average在**一分钟内逼近10.00。**
  ``` nohighlight
  [root@localhost ~]# watch -d uptime
  Every 2.0s: uptime                                        Tue Sep  3 22:08:46 2019
  
   22:08:46 up 41 min,  4 users,  load average: 9.34, 8.13, 4.66
  ```

#### Windows 3

  全体CPU的%usr都在上升。

  ``` nohighlight
  [root@localhost ~]# mpstat -P ALL 5 3
                      v
  09:59:51 PM CPU   %usr %nice  %sys %iowait %irq %soft  %steal  %guest  %gnice  %idle
  09:59:57 PM  all 99.70  0.00  0.20  0.10   0.00  0.10    0.00    0.00    0.00   0.00
  09:59:57 PM    0 99.80  0.00  0.20  0.00   0.00  0.00    0.00    0.00    0.00   0.00
  09:59:57 PM    1 99.80  0.00  0.20  0.00   0.00  0.00    0.00    0.00    0.00   0.00
  
  09:59:57 PM  CPU %usr  %nice  %sys %iowait %irq %soft  %steal  %guest  %gnice  %idle
  10:00:02 PM  all 99.60  0.00  0.40  0.00   0.00  0.10    0.00    0.00    0.00   0.00
  10:00:02 PM    0 99.80  0.00  0.20  0.00   0.10  0.00    0.00    0.00    0.00   0.00
  10:00:02 PM    1 99.40  0.00  0.60  0.00   0.00  0.00    0.00    0.00    0.00   0.00
  
  10:00:02 PM  CPU %usr  %nice  %sys %iowait %irq %soft  %steal  %guest  %gnice  %idle
  10:00:07 PM  all 99.70  0.00  0.30  0.00   0.00  0.00    0.00    0.00    0.00   0.00
  10:00:07 PM    0 99.80  0.00  0.20  0.00   0.00  0.00    0.00    0.00    0.00    0.00
  10:00:07 PM    1 99.60  0.00  0.40  0.00   0.00  0.00    0.00    0.00    0.00    0.00
  ```

#### Windows 4

  Load Average上升时因为等待CPU，也即是%wait。

  ```nohighlight
  [root@localhost ~]# pidstat -u 5 1
                                                          v
  10:33:44 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
  10:33:49 PM     0      2774    0.00    0.20    0.00    0.00    0.20     1  auditd
  10:33:49 PM     0      2853    0.20    0.20    0.00   74.45    0.40     1  YDService
  10:33:49 PM     0     16458    0.00    0.20    0.00    0.20    0.20     0  pidstat
  10:33:49 PM     0     16488   24.55    0.00    0.00   75.65   24.55     0  stress
  10:33:49 PM     0     16489   24.55    0.00    0.00   75.05   24.55     0  stress
  10:33:49 PM     0     16490   24.75    0.00    0.00   75.45   24.75     0  stress
  10:33:49 PM     0     16491   24.75    0.00    0.00   74.65   24.75     1  stress
  10:33:49 PM     0     16492   24.95    0.00    0.00   75.45   24.95     1  stress
  10:33:49 PM     0     16493   24.95    0.00    0.00   75.05   24.95     1  stress
  10:33:49 PM     0     16494   24.35    0.00    0.00   75.05   24.35     0  stress
  10:33:49 PM     0     16495   24.75    0.00    0.00   75.65   24.75     1  stress
  ```

## 0x02 结语

---

### Load Averages = running(运行态) + uninterruptable(不可中断态)

- *System Load Averages ↑ = Using CPU ↑ + Waiting CPU + Waiting I/O*
- *System Load Averages ↑ = Using CPU + Waiting CPU  ↑ + Waiting I/O*
- *System Load Averages ↑ = Using CPU + Waiting CPU  + Waiting I/O ↑*

### 平均负载可以通过以下公式进行计算。

>load(t) = n+((load(t-1)-n)/e^(interval/(min*60)))
>load(t): 平均负载的时间.
>n: 运行态和不可中断态的线程数
>interval: 计算间隔，RHEL是5秒
>min: 负载的时长(分钟数)
