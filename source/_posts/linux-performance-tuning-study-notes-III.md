---
title: Linux Performance Tuning Study Notes III
mathjax: true
copyright: true
comment: true
date: 2019-09-03 22:57:23
categories:
- "Ops "
tags:
- "Linux "
- "Performance Tuning "
- "Performance Analysis "
photo:
---

CPU 上下文切换

Linux是一個多任務操作系統，允許任務數 >> CPU數， CPU上下文切換執行多任務，造成負載升高。

CPU上下文包括以下：

- CPU寄存器：CPU內置的極小內存

- 程序計數器：用來存儲CPU正在執行的指令位置，或即將執行的指令位置。

