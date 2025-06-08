---
title: 趣談Linux操作系統筆記I（之剖析系統進程）
mathjax: true
copyright: true
comment: true
date: 2020-06-25 21:21:52
updated: 2020-07-11 18:33:33
categories:
- "Ops "
tags:
- "Linux "
- "趣談Linux操作系統筆記"
---

<center><img src="https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/linux-tux-minimalism-4k-42-1280x800.jpg" width=50% /></center>

本文旨在從三個方面（被執行的主體[Executable and Linkable Format]，執行程序的主體，執行函數的主體）闡述系統調用的全過程。

<!-- more -->

## 0x00 被執行的主體：ELF

---

### ELF的第一種類型：可重定位文件

可重定位文件全程：Relocatable File，環境準備，安裝軟體。

```bash
[root@learn ~]# yum -y groupinstall "Development Tools"
```

新建代碼工程文件。

```bash
[root@learn ~]# cat >> process.c << EOF
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

extern int create_process (char* program, char** arg_list);

int create_process (char* program, char** arg_list)
{
        pid_t child_pid;                  # 初始化child_pid
        child_pid = fork ();              # 調用fork獲取進程號
        if (child_pid != 0)               # 如果獲取成功
            return child_pid;             # 則返回獲取的進程號
        else {                            # 否則
            execvp (program, arg_list);   # 調用執行，輸入程序名及其參數
            abort ();                     # 之後中斷                  
        }
}
EOF
[root@learn ~]# cat >> createprocess.c << EOF
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

extern int create_process (char* program, char** arg_list);

int main ()
{
    char* arg_list[] = {                  # 參數列表
        "ls",                             
        "-l",
        "/etc/yum.repos.d/",
        NULL                              # 完成輸入
    };
    create_process ("ls", arg_list);      # 調用process.c中自定義的create_process函數
    return 0;
}
EOF
```

進行編譯，**獲得ELF的第一種類型: 可重定位文件(Relocatable File)。**

```bash
[root@learn ~]# gcc -c -fPIC process.c
[root@learn ~]# gcc -c -fPIC createprocess.c
```

對其内部sections進行檢視。

![https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/relocatableFileFormat.jpg](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/relocatableFileFormat.jpg)

```bash
[root@learn ~]# readelf -S ./createprocess.o                             
There are 13 section headers, starting at offset 0x368:

Section Headers:
  [Nr] Name              Type             Address           Offset    ⚠局部變量運行時存在
       Size              EntSize          Flags  Link  Info  Align    棧，運行前的.o文件不
  [ 0]                   NULL             0000000000000000  00000000  包含。
       0000000000000000  0000000000000000           0     0     0
  [ 1] **.text**             PROGBITS     0000000000000000  00000040 <=== 編譯后的二進制代碼
       000000000000004b  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000270 <=== 標注需重定位的函數
       0000000000000078  0000000000000018   I      10     1     8
  [ 3] **.data**             PROGBITS     0000000000000000  0000008b <=== 已初始化的全局變量
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] **.bss**              NOBITS       0000000000000000  0000008b <=== 未初始化的全局變量
       0000000000000000  0000000000000000  WA       0     0     1
  [ 5] **.rodata**           PROGBITS     0000000000000000  0000008b <=== 只讀數據，常量變量
       0000000000000018  0000000000000000   A       0     0     1       （這正是process.o
  [ 6] .comment          PROGBITS         0000000000000000  000000a3     所沒有的）
       000000000000002e  0000000000000001  MS       0     0     1
  [ 7] .note.GNU-stack   PROGBITS         0000000000000000  000000d1
       0000000000000000  0000000000000000           0     0     1
  [ 8] .eh_frame         PROGBITS         0000000000000000  000000d8
       0000000000000038  0000000000000000   A       0     0     8
  [ 9] .rela.eh_frame    RELA             0000000000000000  000002e8
       0000000000000018  0000000000000018   I      10     8     8
  [10] .symtab           SYMTAB           0000000000000000  00000110 <=== 符號表，函數變量
       0000000000000120  0000000000000018          11     9     8
  [11] .strtab           STRTAB           0000000000000000  00000230 <=== 字符串表，字符串常量
       000000000000003b  0000000000000000           0     0     1        及變量名
  [12] .shstrtab         STRTAB           0000000000000000  00000300
       0000000000000061  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

### ELF的第二種類型：可執行文件

可执行文件全稱：Executable file，先通過ar，將.o文件歸檔為.a文件，可作爲靜態鏈接庫。

```bash
[root@learn ~]# man ar
c   Create the archive.  The specified archive is always created if it did not exist, when you request an update.  But a warning is issued unless you specify in advance that you expect to create        
           it, by using this modifier.
r   Insert the files member... into archive (with replacement). This operation differs from q in that any previously existing members are deleted if their names match those being added.

           If one of the files named in member... does not exist, ar displays an error message, and leaves undisturbed any existing members of the archive matching that name.

           By default, new members are added at the end of the file; but you may use one of the modifiers a, b, or i to request placement relative to some existing member.

           The modifier v used with this operation elicits a line of output for each file inserted, along with one of the letters a or r to indicate whether the file was appended (no old member deleted)       
           or replaced.
[root@learn ~]# ar cr libstaticprocess.a process.o
[root@learn ~]# readelf -S libstaticprocess.a 

File: libstaticprocess.a(process.o)   <---   （只是歸檔，并非新文件）
There are 12 section headers, starting at offset 0x328:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       000000000000003d  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000268
       0000000000000048  0000000000000018   I       9     1     8
  [ 3] .data             PROGBITS         0000000000000000  0000007d
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .bss              NOBITS           0000000000000000  0000007d
       0000000000000000  0000000000000000  WA       0     0     1
  [ 5] .comment          PROGBITS         0000000000000000  0000007d
       000000000000002e  0000000000000001  MS       0     0     1
  [ 6] .note.GNU-stack   PROGBITS         0000000000000000  000000ab
       0000000000000000  0000000000000000           0     0     1
  [ 7] .eh_frame         PROGBITS         0000000000000000  000000b0
       0000000000000038  0000000000000000   A       0     0     8
  [ 8] .rela.eh_frame    RELA             0000000000000000  000002b0
       0000000000000018  0000000000000018   I       9     7     8
  [ 9] .symtab           SYMTAB           0000000000000000  000000e8
       0000000000000138  0000000000000018          10     8     8
  [10] .strtab           STRTAB           0000000000000000  00000220
       0000000000000042  0000000000000000           0     0     1
  [11] .shstrtab         STRTAB           0000000000000000  000002c8
       0000000000000059  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

后將.a文件鏈入目標文件**(Relocatable File)**中去編譯即可得到**ELF的第二種形態，可執行文件(Executable file)**。

```bash
                                          目標文件  -L 查找當前目錄     
[root@learn ~]# gcc -o staticcreateprocess createprocess.o -L . -l staticprocess
             編譯       輸出文件                               -l 指定library並補全lib和.a
```

對其内部sections進行檢視。

> ELF Header，e_entry虛擬地址入口。

> Section Header Table（段頭表，struct elf32_phdr 和 struct elf64_phdr兩個描述代碼運行于32/64位。還有p_vaddr加載内存地址）

![https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/executableFileFormat.jpg](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/executableFileFormat.jpg)

```bash
[root@learn ~]# ./staticcreateprocess 
total 228
-rw-r--r--. 1 root root 231449 Jun 22 05:36 redhat.repo
[root@learn ~]# readelf -S ./staticcreateprocess 
There are 31 section headers, starting at offset 0x19e8:            ⚠ 準備加載到内存執行，因
                                                                       此包含需要加載的代碼塊，
Section Headers:                                                       數據段和不需要加載的部
  [Nr] Name              Type             Address           Offset     分。
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298
       000000000000001c  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002b8  000002b8
       0000000000000090  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000400348  00000348
       000000000000004a  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           0000000000400392  00000392
       000000000000000c  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          00000000004003a0  000003a0
       0000000000000020  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             00000000004003c0  000003c0
       0000000000000018  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             00000000004003d8  000003d8
       0000000000000060  0000000000000018  AI       5    24     8
  [11] .init             PROGBITS         0000000000400438  00000438
       000000000000001a  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         0000000000400460  00000460
       0000000000000050  0000000000000010  AX       0     0     16
  [13] .plt.got          PROGBITS         00000000004004b0  000004b0
       0000000000000008  0000000000000000  AX       0     0     8
  [14] .text             PROGBITS         00000000004004c0  000004c0  <===  代碼段
       00000000000001f2  0000000000000000  AX       0     0     16
  [15] .fini             PROGBITS         00000000004006b4  000006b4
       0000000000000009  0000000000000000  AX       0     0     4
  [16] .rodata           PROGBITS         00000000004006c0  000006c0  <===  代碼段
       0000000000000028  0000000000000000   A       0     0     8
  [17] .eh_frame_hdr     PROGBITS         00000000004006e8  000006e8
       000000000000003c  0000000000000000   A       0     0     4
  [18] .eh_frame         PROGBITS         0000000000400728  00000728
       0000000000000114  0000000000000000   A       0     0     8
  [19] .init_array       INIT_ARRAY       0000000000600e10  00000e10
       0000000000000008  0000000000000008  WA       0     0     8
  [20] .fini_array       FINI_ARRAY       0000000000600e18  00000e18
       0000000000000008  0000000000000008  WA       0     0     8
  [21] .jcr              PROGBITS         0000000000600e20  00000e20
       0000000000000008  0000000000000000  WA       0     0     8
  [22] .dynamic          DYNAMIC          0000000000600e28  00000e28
       00000000000001d0  0000000000000010  WA       6     0     8
  [23] .got              PROGBITS         0000000000600ff8  00000ff8
       0000000000000008  0000000000000008  WA       0     0     8
  [24] .got.plt          PROGBITS         0000000000601000  00001000
       0000000000000038  0000000000000008  WA       0     0     8
  [25] .data             PROGBITS         0000000000601038  00001038  <===  數據段  
       0000000000000004  0000000000000000  WA       0     0     1
  [26] .bss              NOBITS           000000000060103c  0000103c  <===  數據段 
       0000000000000004  0000000000000000  WA       0     0     1
  [27] .comment          PROGBITS         0000000000000000  0000103c
       000000000000002d  0000000000000001  MS       0     0     1
  [28] .symtab           SYMTAB           0000000000000000  00001070  <===  不加載到内存段 
       0000000000000660  0000000000000018          29    48     8
  [29] .strtab           STRTAB           0000000000000000  000016d0  <===  不加載到内存段
       0000000000000208  0000000000000000           0     0     1
  [30] .shstrtab         STRTAB           0000000000000000  000018d8
       000000000000010c  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

運行結果

```bash
[root@learn ~]# ./staticcreateprocess 
total 228
-rw-r--r--. 1 root root 231449 Jun 22 05:36 redhat.repo
```

### ELF的第三種類型：共享對象文件

共享對象文件全稱：Shared Object，先通過gcc，由.o文件編譯為.so文件，可得到**動態鏈接庫，亦就是ELF第三種類型**。

```bash
[root@learn ~]# gcc -shared -fPIC -o libdynamicprocess.so process.o
```

對其内部sections進行檢視。

```bash
[root@learn ~]# readelf -S libdynamicprocess.so 
There are 28 section headers, starting at offset 0x18c0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .note.gnu.build-i NOTE             00000000000001c8  000001c8
       0000000000000024  0000000000000000   A       0     0     4
  [ 2] .gnu.hash         GNU_HASH         00000000000001f0  000001f0
       000000000000003c  0000000000000000   A       3     0     8
  [ 3] .dynsym           DYNSYM           0000000000000230  00000230
       0000000000000168  0000000000000018   A       4     1     8
  [ 4] .dynstr           STRTAB           0000000000000398  00000398
       00000000000000c4  0000000000000000   A       0     0     1
  [ 5] .gnu.version      VERSYM           000000000000045c  0000045c
       000000000000001e  0000000000000002   A       3     0     2
  [ 6] .gnu.version_r    VERNEED          0000000000000480  00000480
       0000000000000020  0000000000000000   A       4     1     8
  [ 7] .rela.dyn         RELA             00000000000004a0  000004a0
       00000000000000c0  0000000000000018   A       3     0     8
  [ 8] .rela.plt         RELA             0000000000000560  00000560
       0000000000000048  0000000000000018  AI       3    22     8
  [ 9] .init             PROGBITS         00000000000005a8  000005a8
       000000000000001a  0000000000000000  AX       0     0     4
  [10] .plt              PROGBITS         00000000000005d0  000005d0
       0000000000000040  0000000000000010  AX       0     0     16
  [11] .plt.got          PROGBITS         0000000000000610  00000610
       0000000000000010  0000000000000000  AX       0     0     8
  [12] .text             PROGBITS         0000000000000620  00000620
       0000000000000122  0000000000000000  AX       0     0     16
  [13] .fini             PROGBITS         0000000000000744  00000744
       0000000000000009  0000000000000000  AX       0     0     4
  [14] .eh_frame_hdr     PROGBITS         0000000000000750  00000750
       000000000000001c  0000000000000000   A       0     0     4
  [15] .eh_frame         PROGBITS         0000000000000770  00000770
       0000000000000064  0000000000000000   A       0     0     8
  [16] .init_array       INIT_ARRAY       0000000000200df8  00000df8
       0000000000000008  0000000000000008  WA       0     0     8
  [17] .fini_array       FINI_ARRAY       0000000000200e00  00000e00
       0000000000000008  0000000000000008  WA       0     0     8
  [18] .jcr              PROGBITS         0000000000200e08  00000e08
       0000000000000008  0000000000000000  WA       0     0     8
  [19] .data.rel.ro      PROGBITS         0000000000200e10  00000e10
       0000000000000008  0000000000000000  WA       0     0     8
  [20] .dynamic          DYNAMIC          0000000000200e18  00000e18
       00000000000001c0  0000000000000010  WA       4     0     8
  [21] .got              PROGBITS         0000000000200fd8  00000fd8
       0000000000000028  0000000000000008  WA       0     0     8
  [22] .got.plt          PROGBITS         0000000000201000  00001000
       0000000000000030  0000000000000008  WA       0     0     8
  [23] .bss              NOBITS           0000000000201030  00001030
       0000000000000008  0000000000000000  WA       0     0     1
  [24] .comment          PROGBITS         0000000000000000  00001030
       000000000000002d  0000000000000001  MS       0     0     1
  [25] .symtab           SYMTAB           0000000000000000  00001060
       0000000000000570  0000000000000018          26    44     8
  [26] .strtab           STRTAB           0000000000000000  000015d0
       00000000000001f5  0000000000000000           0     0     1
  [27] .shstrtab         STRTAB           0000000000000000  000017c5
       00000000000000f5  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

之後.so文件同樣可鏈入目標文件（可重定位文件）中去編譯出來。

```bash
                             形態2                形態1                形態3
[root@learn ~]# gcc -o dynamiccreateprocess createprocess.o -L. -l dynamicprocess
```

運行結果，運行時需要手動指定load的路徑。

```bash
[root@learn ~]# export LD_LIBRARY_PATH=.
[root@learn ~]# ./dynamiccreateprocess 
total 228
-rw-r--r--. 1 root root 231449 Jun 22 05:36 redhat.repo
[root@learn ~]# readelf -S dynamiccreateprocess 
There are 31 section headers, starting at offset 0x1940:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238 <--- ld-linux.so
       000000000000001c  0000000000000000   A       0     0     1         負責運行時進行鏈接
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298
       0000000000000038  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002d0  000002d0
       00000000000000d8  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           00000000004003a8  000003a8
       0000000000000080  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           0000000000400428  00000428
       0000000000000012  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          0000000000400440  00000440
       0000000000000020  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             0000000000400460  00000460
       0000000000000018  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             0000000000400478  00000478
       0000000000000030  0000000000000018  AI       5    24     8
  [11] .init             PROGBITS         00000000004004a8  000004a8
       000000000000001a  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         00000000004004d0  000004d0  <---過程鏈接表 PLT
       0000000000000030  0000000000000010  AX       0     0     16   Procedure Linkage Table，
  [13] .plt.got          PROGBITS         0000000000400500  00000500
       0000000000000008  0000000000000000  AX       0     0     8
  [14] .text             PROGBITS         0000000000400510  00000510
       00000000000001b2  0000000000000000  AX       0     0     16
  [15] .fini             PROGBITS         00000000004006c4  000006c4
       0000000000000009  0000000000000000  AX       0     0     4
  [16] .rodata           PROGBITS         00000000004006d0  000006d0
       0000000000000028  0000000000000000   A       0     0     8
  [17] .eh_frame_hdr     PROGBITS         00000000004006f8  000006f8
       0000000000000034  0000000000000000   A       0     0     4
  [18] .eh_frame         PROGBITS         0000000000400730  00000730
       00000000000000f4  0000000000000000   A       0     0     8
  [19] .init_array       INIT_ARRAY       0000000000600e00  00000e00
       0000000000000008  0000000000000008  WA       0     0     8
  [20] .fini_array       FINI_ARRAY       0000000000600e08  00000e08
       0000000000000008  0000000000000008  WA       0     0     8
  [21] .jcr              PROGBITS         0000000000600e10  00000e10
       0000000000000008  0000000000000000  WA       0     0     8
  [22] .dynamic          DYNAMIC          0000000000600e18  00000e18
       00000000000001e0  0000000000000010  WA       6     0     8
  [23] .got              PROGBITS         0000000000600ff8  00000ff8
       0000000000000008  0000000000000008  WA       0     0     8
  [24] .got.plt          PROGBITS         0000000000601000  00001000  <--- 全局偏移量表 GOT
       0000000000000028  0000000000000008  WA       0     0     8      Global Offset Table    
  [25] .data             PROGBITS         0000000000601028  00001028
       0000000000000004  0000000000000000  WA       0     0     1
  [26] .bss              NOBITS           000000000060102c  0000102c
       0000000000000004  0000000000000000  WA       0     0     1
  [27] .comment          PROGBITS         0000000000000000  0000102c
       000000000000002d  0000000000000001  MS       0     0     1
  [28] .symtab           SYMTAB           0000000000000000  00001060
       0000000000000600  0000000000000018          29    47     8
  [29] .strtab           STRTAB           0000000000000000  00001660
       00000000000001cf  0000000000000000           0     0     1
  [30] .shstrtab         STRTAB           0000000000000000  0000182f
       000000000000010c  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

工作流程（編譯時建立PLT代替去找，PLT因爲GOT告知，因此會問GOT，GOT引路讓PLT去加載ld-linux.so直接在内存中找），之所以必須PLT去找因爲其作用為存放stub代碼，GOT存放so對應的真實代碼地址。

dynamiccreateprocess→ PLT⇋GOT→ld-linux.so→libdynamicprocess.so(create_process)

>它们是怎么工作的，使得程序运行的时候，可以将 so 文件动态链接到进程空间的呢？dynamiccreateprocess 这个程序要调用 libdynamicprocess.so 里的 create_process 函数。由于是运行时才去找，编译的时候，压根不知道这个函数在哪里，所以就在 PLT 里面建立一项 PLT[x]。这一项也是一些代码，有点像一个本地的代理，在二进制程序里面，不直接调用 create_process 函数，而是调用 PLT[x]里面的代理代码，这个代理代码会在运行的时候找真正的 create_process 函数。去哪里找代理代码呢？这就用到了 GOT，这里面也会为 create_process 函数创建一项 GOT[y]。这一项是运行时 create_process 函数在内存中真正的地址。如果这个地址在 dynamiccreateprocess 调用 PLT[x]里面的代理代码，代理代码调用 GOT 表中对应项 GOT[y]，调用的就是加载到内存中的 libdynamicprocess.so 里面的 create_process 函数了。但是 GOT 怎么知道的呢？对于 create_process 函数，GOT 一开始就会创建一项 GOT[y]，但是这里面没有真正的地址，因为它也不知道，但是它有办法，它又回调 PLT，告诉它，你里面的代理代码来找我要 create_process 函数的真实地址，我不知道，你想想办法吧。PLT 这个时候会转而调用 PLT[0]，也即第一项，PLT[0]转而调用 GOT[2]，这里面是 ld-linux.so 的入口函数，这个函数会找到加载到内存中的 libdynamicprocess.so 里面的 create_process 函数的地址，然后把这个地址放在 GOT[y]里面。下次，PLT[x]的代理函数就能够直接调用了。

## 0x01 執行程序的主體：load_elf_binary等函數

---

系統中大體通過以下的調用順序執行：

```bash
sys_execve->do_execve->do_execveat_common->exec_binprm->search_binary_handler。
```

而exec函數的命名還遵循以下規則：

>- 包含 p 的函数（execvp, execlp）会在 PATH 路径下面寻找程序；
>- 不包含 p 的函数需要输入程序的全路径；
>- 包含 v 的函数（execv, execvp, execve）以数组的形式接收参数；
>- 包含 l 的函数（execl, execlp, execle）以列表的形式接收参数；
>- 包含 e 的函数（execve, execle）以数组的形式接收环境变量。

![https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/execnamerule.jpg](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/execnamerule.jpg)

## 0x02 執行函數的主體：進程

---

```bash
[root@deployer ~]# ps -ef
           進程 父進程        發起地             無括號，用戶態，父進程為1，systemd init進程
                             問號為後臺          中括號，内核態，父進程為2，kthreadd内核綫程
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0  2018 ?        00:00:29 /usr/lib/systemd/systemd --system --deserialize 21
root         2     0  0  2018 ?        00:00:00 [kthreadd]
root         3     2  0  2018 ?        00:00:00 [ksoftirqd/0]
root         5     2  0  2018 ?        00:00:00 [kworker/0:0H]
root         9     2  0  2018 ?        00:00:40 [rcu_sched]
......
root       337     2  0  2018 ?        00:00:01 [kworker/3:1H]
root       380     1  0  2018 ?        00:00:00 /usr/lib/systemd/systemd-udevd
root       415     1  0  2018 ?        00:00:01 /sbin/auditd
root       498     1  0  2018 ?        00:00:03 /usr/lib/systemd/systemd-logind
......
root       852     1  0  2018 ?        00:06:25 /usr/sbin/rsyslogd -n
root      2580     1  0  2018 ?        00:00:00 /usr/sbin/sshd -D
root     29058     2  0 Jan03 ?        00:00:01 [kworker/1:2]
root     29672     2  0 Jan04 ?        00:00:09 [kworker/2:1]
root     30467     1  0 Jan06 ?        00:00:00 /usr/sbin/crond -n
root     31574     2  0 Jan08 ?        00:00:01 [kworker/u128:2]
......
root     32792  2580  0 Jan10 ?        00:00:00 sshd: root@pts/0
root     32794 32792  0 Jan10 pts/0    00:00:00 -bash
root     32901 32794  0 00:01 pts/0    00:00:00 ps -ef
```

## 0x03 結語

---

終於啃完進程了，耗時好幾天，果然寫博客方便整理思路。

1. ELF的三種形態，.o , 二進制文件, .so ，.a只是.o的簡單歸檔。
2. 既然ELF是被執行的主體，自然系統調用到最後需要load_elf_binary函數。
3. 而在系統層面，用戶態的祖宗進程和内核態的祖宗進程分別以自己的方式進行函數的調用。

個人學習筆記，如果有誤，歡迎指正。
