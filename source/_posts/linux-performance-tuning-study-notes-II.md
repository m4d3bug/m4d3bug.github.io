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

``` nohighlight
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

+ *System load averages ↑ = Using CPU ↑ + Waiting CPU + Waiting I/O*

+ *System load averages ↑ = Using CPU + Waiting CPU  ↑ + Waiting I/O*

+ *System load averages ↑ = Using CPU + Waiting CPU  + Waiting I/O ↑*

  因此，平均負載和 CPU 使用率不一定掛鉤。

###  *R+ = runnable  ↓  D+ = Disk Sleep(uninterruptable)   ↓*

``` nohighlight
[root@localhost ~]# ps -aux
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      22417  0.0  0.3  39008  3628 pts/0    R+   21:40   0:00 ps -aux
root      22418  0.0  0.1  22016  1624 pts/0    D+   21:40   0:00 -bash
```

### *what if load average = 1*：

 *CPU NUM = grep 'model name' /proc/cpuinfo | wc -l*

+ *1 CPU system: loaded all the time.*

+ *4 CPU system: 1/4 loaded all the time.*

#### *Load averages should be smaller than the number of CPUs 70%.*