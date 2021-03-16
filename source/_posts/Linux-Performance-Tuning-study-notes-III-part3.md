# Linux性能調優實戰筆記III（CPU上下文切換-下）

本文旨在剖析CPU上下文切换的一个H。

## 0x00 工具准备

---

- 分析：
    - vmstat、pidstat、watch

        ```bash
        # yum install procps-ng sysstat -y
        ```

- 压测：sysbench

    ```bash
    # yum-config-manager --enable epel
    # yum install sysbench -y 
    # yum-config-manager --disable epel
    ```

- 监控对象：进程、线程、in列

## 0x01 如何分析(how?)

---

### 事前介紹

- vmstat（系统总体の上下文切换情况）

    ```bash
    # vmstat -wSm 1 6
    procs -----------------------memory---------------------- ---swap-- -----io---- -system-- --------cpu--------
     r  b         swpd         free         buff        cache   si   so    bi    bo   in   cs  us  sy  id  wa  st
     3  0            0         3697            2         2732    0    0    10    12   43   87   0   0 100   0   0
     1  0            0         3684            2         2745    0    0     0     0 4518  717  26  21  53   0   0
     1  0            0         3673            2         2757    0    0     0     0 4041  553  27  18  55   0   0
     0  0            0         3659            2         2769    0    0     0     0 4605  609  27  19  54   0   0
     1  0            0         3640            2         2785    0    0     0     0 1552  308  39  12  48   0   0
     1  0            0         3603            2         2822    0    0     0     0 1182  273  43   9  48   0   0
    ```

    - r列：就绪队列，**运行&等待的CPU**进程数。
    - b列：不可中斷睡眠狀態的進程數。
    - in列：每秒中断次数。
    - cs列：每秒content switch次数变化。
    - us列、sy列：user及system层面的系统CPU使用率。
- pidstat（进程の上下文切换情况）

    ```bash
    # pidstat -w -u 5
    Average:      UID       PID    %usr %system  %guest    %CPU   CPU  Command
    Average:        0       845    0.00    0.20    0.00    0.20     -  vmtoolsd
    Average:        0      1053    0.20    0.20    0.00    0.40     -  NetworkManager
    Average:        0      1366    0.00    0.20    0.00    0.20     -  rsyslogd
    Average:        0     20518    0.00    0.20    0.00    0.20     -  pidstat
    Average:        0     76678    0.20    0.20    0.00    0.40     -  tmux

    Average:      UID       PID   cswch/s nvcswch/s  Command
    Average:        0         1      1.00      0.00  systemd
    Average:        0         6      1.79      0.00  ksoftirqd/0
    Average:        0         7      1.59      0.00  migration/0
    Average:        0         9     33.86      0.00  rcu_sched
    Average:        0        11      0.40      0.00  watchdog/0
    Average:        0        12      0.40      0.00  watchdog/1
    Average:        0        13      2.59      0.00  migration/1
    Average:        0        14      2.79      0.00  ksoftirqd/1
    Average:        0        37      0.20      0.00  khugepaged
    Average:        0       511     19.92      0.00  xfsaild/dm-0
    Average:        0       595      0.20      0.00  systemd-journal
    Average:        0       845     11.95      0.00  vmtoolsd
    Average:        0       848      1.59      0.00  rngd
    Average:        0      1053      4.18      0.00  NetworkManager
    Average:        0      9445      0.20      0.00  goa-identity-se
    Average:        0      9535      1.00      0.00  gsd-color
    Average:        0      9670     10.16      0.00  vmtoolsd
    Average:        0      9766      0.20      0.00  fwupd
    Average:        0     16537      1.59      0.00  kworker/1:1
    Average:        0     20518      0.20      0.20  pidstat
    Average:        0     76190      1.20      0.00  kworker/u256:2
    Average:        0     76678      9.36      0.00  tmux
    Average:        0    107281      1.79      0.00  kworker/0:1
    ```

    - cswch/s：每秒自愿上下文切换，进程（资源耗尽）自愿进而上下文切换。
    - nvcswch/s：每秒非自愿上下文切换，进程因（优先级、时间片）因素被强制调度进而上下文切换。
- watch（监控interrupts详情）
- sysbench（以10个线程运行5分钟的基准测试，模拟多线程切换的问题）

    ```bash
    # sysbench --threads=10 --max-time=300 threads run
    WARNING: --max-time is deprecated, use --time instead
    sysbench 1.0.17 (using system LuaJIT 2.0.4)

    Running the test with following options:
    Number of threads: 10
    Initializing random number generator from current time

    Initializing worker threads...

    Threads started!
    ```

### 开始分析

- **vmstat**

    ```bash
    -w, --wide
       Wide  output mode (useful for systems with higher amount of memory, where the default output mode suffers from unwanted column breakage).  The output is wider than 80 characters per
       line.
    -S, --unit character
       Switches outputs between 1000 (k), 1024 (K), 1000000 (m), or 1048576 (M) bytes.  Note this does not change the swap (si/so) or block (bi/bo) fields.
    -m, --slabs
       Displays slabinfo.

    # vmstat -wSm 1 6
    procs -----------------------memory---------------------- ---swap-- -----io---- -system-- --------cpu--------
     r  b         swpd         free         buff        cache   si   so    bi    bo   in   cs  us  sy  id  wa  st
     7  0            0          302            2         6118    0    0    10    36   44  103   0   0 100   0   0
    11  0            0          282            2         6138    0    0     0     0 2475 2262933  27  73   0   0   0
     8  0            0          266            2         6159    0    0     0     0 2173 1915515  35  65   1   0   0
     5  0            0          266            2         6159    0    0     0     0 2264 1983326  37  63   0   0   0
     6  0            0          265            2         6159    0    0     0     0 2284 1899297  33  67   1   0   0
     7  0            0          266            2         6159    0    0     0     0 2342 2126275  33  68   0   0   0
    ```

    - r列：远超实际CPU个数。
    - cs列：上升。
    - us列、sy列：系統cpu使用率飙升，user的cpu使用率无变化。
    - in列：同样飙升（本节最后分析）

    結論：r↑ 就绪越多，cs↑上下文切换频繁，所以sy↑，CPU使用率飙升。

- **pidstat**

    ```bash
    -u     Report CPU utilization.
    -w     Report task switching activity (kernels 2.6.23 and later only).  
    -t     Also display statistics for threads associated with selected tasks.

    # pidstat -w -u 1
    Linux 3.10.0-1160.el7.x86_64 (learning79.madebug.net) 03/14/2021 _x86_64_ (2 CPU)

    10:10:58 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
    10:10:59 AM     0       845    0.00    1.00    0.00    1.00     1  vmtoolsd
    10:10:59 AM     0    114598   19.00  100.00    0.00  100.00     1  sysbench
    10:10:59 AM     0    114613    0.00    1.00    0.00    1.00     0  pidstat

    10:10:58 AM   UID       PID   cswch/s nvcswch/s  Command
    10:10:59 AM     0         1      1.00      0.00  systemd
    10:10:59 AM     0         6      2.00      0.00  ksoftirqd/0
    10:10:59 AM     0         7      1.00      0.00  migration/0
    10:10:59 AM     0         9     60.00      0.00  rcu_sched
    10:10:59 AM     0        13      1.00      0.00  migration/1
    10:10:59 AM     0       511     20.00      0.00  xfsaild/dm-0
    10:10:59 AM     0       845     11.00      0.00  vmtoolsd
    10:10:59 AM     0       848      1.00      0.00  rngd
    10:10:59 AM     0      1053      5.00      0.00  NetworkManager
    10:10:59 AM     0      9025      1.00      0.00  wpa_supplicant
    10:10:59 AM     0      9445      1.00      0.00  goa-identity-se
    10:10:59 AM     0      9535      1.00      0.00  gsd-color
    10:10:59 AM     0      9670      9.00      0.00  vmtoolsd
    10:10:59 AM     0     76190      1.00      0.00  kworker/u256:2
    10:10:59 AM     0     76483      6.00      0.00  sshd
    10:10:59 AM     0     76678     24.00      0.00  tmux
    10:10:59 AM     0    107281    157.00      0.00  kworker/0:1
    10:10:59 AM     0    112877      1.00      0.00  kworker/1:1
    10:10:59 AM     0    114613      1.00    143.00  pidstat
    10:10:59 AM     0    114627      3.00      1.00  byobu-status
    ```

    - sysbench的CPU使用率100%
    - pidstat的非自愿上下文切换频率到143
    - kworker/0:1的自愿上下文切换去到了157
        - 加上-t参数输出进程下线程的数据

            ```bash
            10:44:12 PM     0     58152         -      6.00      0.00  kworker/1:0
            10:44:12 PM     0         -     58152      6.00      0.00  |__kworker/1:0
            10:44:12 PM     0         -     59224  67134.00 218954.00  |__sysbench
            10:44:12 PM     0         -     59225  54921.00 227870.00  |__sysbench
            10:44:12 PM     0         -     59226  60706.00 217821.00  |__sysbench
            10:44:12 PM     0         -     59227  52608.00 237218.00  |__sysbench
            10:44:12 PM     0         -     59228  45081.00 224735.00  |__sysbench
            10:44:12 PM     0         -     59229  48077.00 232958.00  |__sysbench
            10:44:12 PM     0         -     59230  64332.00 212784.00  |__sysbench
            10:44:12 PM     0         -     59231  40541.00 234529.00  |__sysbench
            10:44:12 PM     0         -     59232  54334.00 223209.00  |__sysbench
            10:44:12 PM     0         -     59233  47245.00 246611.00  |__sysbench
            ```

- **watch**

    ```bash
    # watch -d cat /proc/interrupts
    Every 2.0s: cat /proc/interrupts      
                CPU0       CPU1
    ...
    RES:     976811    1022850   Rescheduling interrupts
    ```

    - 从/proc/interrupts中读取中断内容。
    - /proc是虚拟文件系统，用于内核空间与用户空间之间的通信。
    - 重调度中断（RES）占据极大份额，正在因为多任务频繁唤醒空闲CPU调度新任务执行。

## 0x02 总结

- 上下文切换正常值取决于CPU性能。
- 10000≥x ≥100+算正常，＞10000或切换次数跨度很大，则可能出现性能问题。
- cswch/s变大，进程等待资源，I /O问题。
- nvcswch/s变大，进程被强制调度，CPU争抢，CPU问题。
- in变大，CPU被中断处理程序占用，需查看/proc/interrupts来确定中断类型。