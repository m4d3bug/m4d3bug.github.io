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
- "Context Switch "
photo:
---

*In this post, I am going to markdown what is "Context Switch".*

## *What is "Context Switch"*

*Linux allow processes more than CPUs, Context Switch can storing the state of a process or a thread when the multiple process to share a single CPU.*

Linux是一個多任務操作系統，允許任務數 >> CPU數，CPU上下文切換以滿足多任務共享CPU（造成負載升高）。

*Context switch includes the below hardware:*

CPU上下文涉及的硬件有以下：

- *CPU register: It is one of a small set of data holding places that are part of the computer processor.*

  CPU寄存器：CPU處理器內置的極小內存。

- *Program counter: It is a register in a computer processor that contains the address (location) of the instruction being executed at the current time.*

  程序計數器（指令地址寄存器）：用來存儲CPU正在執行的指令或即將執行的指令位置。

### *Process context switch*

進程上下文切換：內核態⇌用戶態

- *Ring 0: With the highest privileges, all resources can be accessed directly.*

  ​            具有最高權限，可以直接訪問所有資源。 （內核態）

- *Ring 3: Only limited resources can be accessed, and hardware devices (such as memory) cannot be directly accessed. They must be caught in the kernel through system calls to access these privileged resources.*

  ​            只能訪問受限資源，不能直接訪問硬件設備（例如內存），必須通過系統調用到內核中，才能訪問這些特權資源。 （用戶態）

![](https://i.loli.net/2019/09/05/7JGjSOnUPx5X4AK.png)

其一，为了保证所有进程可以得到公平调度，CPU 时间被划分为一段段的时间片，这些时间片再被轮流分配给各个进程。这样，当某个进程的时间片耗尽了，就会被系统挂起，切换到其它正在等待 CPU 的进程运行。

其二，进程在系统资源不足（比如内存不足）时，要等到资源满足后才可以运行，这个时候进程也会被挂起，并由系统调度其他进程运行。

其三，当进程通过睡眠函数 sleep 这样的方法将自己主动挂起时，自然也会重新调度。

其四，当有优先级更高的进程运行时，为了保证高优先级进程的运行，当前进程会被挂起，由高优先级进程来运行。

最后一个，发生硬件中断时，CPU 上的进程会被中断挂起，转而执行内核中的中断服务程序。

### *Thread context switch*



### *Task context switch*