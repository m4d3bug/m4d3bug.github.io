---
title: Linux性能調優實戰筆記I
mathjax: true
copyright: true
comment: true
date: 2019-09-01 20:18:15
updated: 2020-07-11 18:33:33
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

## 0x00 什么是性能问题

---

性能问题的本质，就是系统资源已经达到瓶颈，但请求的处理却还不够快，无法支撑更多的请求。

性能分析，其实就是找出应用或系统的瓶颈，并设法去避免或者缓解它们。

---

## 0x01 不同侧分析问题的次序

---

<center><font size="5"><B>↓从应用侧（吞吐量，延迟……）↓</B></font></center>

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20200411112804.png" width=50% /></center>

<center><font size="5"><B>↑从系统侧（CPU, 内存……）↑</B></font></center>

---

## 0x02 Linux各方面相关的工具

---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20200411112954.png" width=50% /></center>

---

## 0x03 Linux调优的脑图

---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20200411113050.png" width=80% /></center>

---

## 0x04 结语

---

- *High concurrency means a big throughput, and fast response means a small delay.* 高并發就是吞吐大，响应快就是延时小。

---

