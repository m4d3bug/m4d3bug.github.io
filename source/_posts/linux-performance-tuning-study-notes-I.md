---
title: Linux Performance Tuning Study Notes I
mathjax: true
copyright: true
comment: true
date: 2019-09-01 20:18:15
categories:
- "Ops "
tags:
- "Linux "
- "Performance tuning "
- "Performance analysis "
photo:
---

*The nature of performance problems is that the system resources have reached the bottleneck, but the processing requests are not fast enough to support more requests.*

性能問題的本質，就是系統資源已經達到瓶頸，但請求的處理卻還不夠快，無法支撐更多的請求。

*Performance analysis is to find out the bottlenecks of applications or systems and try to avoid or mitigate them.*

性能分析，其實就是找出應用或系統的瓶頸，并設法去避免或者緩解它們。

## *Analyse problems from different side*



<center><font size="5"><B>↓<i>From application side(throughput, delay...)</i>↓</B></font></center>


![](https://i.loli.net/2019/09/01/wRrC6fNJWbMFgQd.png)



<center><font size="5"><B>↑<i>From system side(CPU, memory...)</i>↑</B></font></center>



## *Linux performance tools*

![](https://i.loli.net/2019/09/01/9UxpGEs4hHkValJ.png)

## *Mind mapping of linux tuning*

![](https://i.loli.net/2019/09/01/WLM1DYxAJuFkZVE.png)

## *Done*

*High concurrency means a big throughput, and fast response means a small delay.*

高並發就是吞吐大，響應快就是延時小。

