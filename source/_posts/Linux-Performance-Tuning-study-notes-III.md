---
title: Linux性能調優實戰筆記III（CPU上下文切換-上）
mathjax: true
copyright: true
comment: true
date: 2021-02-22 22:15:53
categories:
- "Ops "
tags:
- "Linux "
- "Linux性能調優實戰筆記 "
- "CPU Content Switch "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/linux-tux-minimalism-4k-42-1280x800.jpg" width=50% /></center>       

本文旨在剖析CPU上下文切換的五個W。

<!-- more -->

## 0x00 概念(what and why?)

- Linux在支持遠超CPU數的任務時，背後在**頻繁為任務輪流切換CPU**。
- 每次切換均需Linux為CPU（**保存/預設**）**CPU寄存器和程序計數器**
    - **CPU寄存器：**CPU内置高速小内存。
    - **程序計數器：**CPU内置指令暫存器。
- 誘發場景：**[進程](https://blog.madebug.net/ops/2020-06-25-funny-with-linux-i-about-progress#0x02-%E5%9F%B7%E8%A1%8C%E5%87%BD%E6%95%B8%E7%9A%84%E4%B8%BB%E9%AB%94%EF%BC%9A%E9%80%B2%E7%A8%8B)の上下文切換、[綫程](https://blog.madebug.net/ops/2020-06-29-funny-with-linux-ii-about-multi-tasks)の上下文切換以及中斷の上下文切換。**

## 0x01 類型(who, when and where?)

### 進程の上下文切換

- [進程通過執行函數去執行程序，是資源擁有的基本單位](https://blog.madebug.net/ops/2020-06-25-funny-with-linux-i-about-progress)，不再贅述。（who）
- 誘發進程“被”切換/釋放CPU上下文：（when）
    - 預定*時間*片耗盡。
    - 系統*資源*不足。
    - 調用sleep將*自*己主動*挂*起。
    - *優先更高*優先級進程。
    - *硬件中斷*，釋放CPU，轉向内核執行中斷。
- 時而内核態，時而用戶態。（where）
- 進程内切換（特權模式切換）：（where）
    - 執行多個系統調用，一次系統調用，兩次CPU上下文切換（**保存/預設**）。
- 進程間切換（進程上下文切換）：（where）
    - 進程切換前/後，需保存/預設【刷新】（内核狀態【堆棧、寄存器】、虛擬内存、棧、全局變量）。
    - 虛擬内存刷新→TLB刷新其映射物理内存位置→内存變慢→其他共享緩存處理器。
    - 如下所示：

    ![https://img.madebug.net/m4d3bug/images-of-website/master/blog/switchcontentbetweenprogress.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/switchcontentbetweenprogress.png)

### 綫程の上下文切換

- [綫程是調度的基本單位](https://blog.madebug.net/ops/2020-06-29-funny-with-linux-ii-about-multi-tasks)，不再贅述。（who）
- 誘發綫程“被”切換/釋放CPU上下文：（when, where）
    - 前後綫程分屬**不同進程切換≈進程間上下文切換**
    - 前後綫程均屬**同一進程切換=一次CPU上下文切换**
- 多綫程消耗＜<多進程消耗

### 中斷の上下文切換

- 同一CPU内，中斷處理比進程優先級更高。
- 中斷，先保存前面進程，后中斷目的進程。（一次保存，一次CPU上下文切換）

## 0x02 小結

- 基本都是概念性東西。
- 切換過多往往是性能下降的元凶之一，也是負載升高的元凶。
