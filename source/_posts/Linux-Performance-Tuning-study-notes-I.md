---
title: Linux性能調優實戰筆記I
mathjax: true
copyright: true
comment: true
date: 2019-09-01 20:18:15
categories:
- "Ops "
tags:
- "Linux "
- "Performance Tuning "
- "Performance Analysis "
- "Linux性能調優實戰筆記"
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/linux-tux-minimalism-4k-42-1280x800.jpg" width=50% /></center>

<!-- more -->

# 什麽是性能問題

性能問題的本質，就是系統資源已經達到瓶頸，但請求的處理卻還不夠快，無法支撐更多的請求。

性能分析，其實就是找出應用或系統的瓶頸，并設法去避免或者緩解它們。

# 不同側分析問題的次序

<center><font size="5"><B>↓從應用側（吞吐量，延遲……）↓</B></font></center>

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20200411112804.png" width=50% /></center>

<center><font size="5"><B>↑從系統側（CPU, 内存……）↑</B></font></center>

# Linux各方面相關的工具

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20200411112954.png" width=50% /></center>

# Linux調優的腦圖

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20200411113050.png" width=50% /></center>

# 結語

*High concurrency means a big throughput, and fast response means a small delay.*

高並發就是吞吐大，響應快就是延時小。

