---
title: 趣談Linux操作系統筆記II（之剖析系統綫程）
mathjax: true
copyright: true
comment: true
date: 2020-06-29 19:03:36
categories:
- "Ops "
tags:
- "Linux "
- "趣談Linux操作系統筆記"
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/linux-tux-minimalism-4k-42-1280x800.jpg" width=50% /></center>

<!-- more -->

本文旨在剖析线程的五個W和一個H。

## 什麽是綫程*(what and why?)*

---

綫程常常與進程一起談論，可以通過以下解釋它們之間關係：

- **進程相當於一個項目，而綫程則是爲了完成項目需求，而建立的一個個開發任務。**

因此，儅開發任務間無先後次序時，適當拆分出支綫來并發，可顯著提升效率（而科學的分工，符合[**宇宙法則**](https://zh.wikipedia.org/zh-tw/%E5%B8%95%E7%B4%AF%E6%89%98%E6%B3%95%E5%88%99)）。

## 如何創建綫程*(how?)*

---

要創建我們就要預先瞭解其函數的調用過程。

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/howtaskwork.jpg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/howtaskwork.jpg)

有了骨架，我們可以構建與之對應的測試代碼。

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM_OF_TASKS 5

void *downloadfile(void *filename) //聲明綫程函數
{
   printf("I am downloading the file %s!\n", (char *)filename);
   sleep(10);
   long downloadtime = rand()%100;
   printf("I finish downloading the file within %d minutes!\n", downloadtime);
   pthread_exit((void *)downloadtime); //次綫程結束並返回隨機生成的downloadtime
}

int main(int argc, char *argv[])
   char files[NUM_OF_TASKS][20]={"file1.avi","file2.rmvb","file3.mp4","file4.wmv","file5.flv"}; //聲明數組
   pthread_t threads[NUM_OF_TASKS];                                    //聲明該數組為綫程對象
   int rc;
   int t;
   int downloadtime;

   pthread_attr_t thread_attr;                                         //設置綫程屬性
   pthread_attr_init(&thread_attr);                                    //初始化綫程
   pthread_attr_setdetachstate(&thread_attr,PTHREAD_CREATE_JOINABLE);  //設置其屬性為主綫程等待子綫程，並獲取其退出狀態方便調試

   for(t=0;t<NUM_OF_TASKS;t++){
     printf("creating thread %d, please help me to download %s\n", t+1, files[t]);  //打印
     rc = pthread_create(&threads[t], &thread_attr, downloadfile, (void *)files[t]); //創建並開啓綫程&thread_attr，然後指定子綫程的函數downloadfile
     if (rc){
       printf("ERROR; return code from pthread_create() is %d\n", rc);
       exit(-1);
     }
   }

   pthread_attr_destroy(&thread_attr);  //銷毀綫程屬性

   for(t=0;t<NUM_OF_TASKS;t++){
     pthread_join(threads[t],(void**)&downloadtime);   //等待綫程結束，並取得返回的downloadtime數值
     printf("Thread %d downloads the file %s in %d minutes.\n",t+1,files[t],downloadtime);
   }

   pthread_exit(NULL); //主綫程結束
}
```

對代碼進行編譯運行。多次運行我們可以見到綫程的隨機創建（皆因綫程創建需要資源動態分配）。

```bash
[root@learn ~]# gcc download.c -lpthread -o download.out
[root@learn ~]# ./download.out 
creating thread 1, please help me to download file1.avi
creating thread 2, please help me to download file2.rmvb
creating thread 3, please help me to download file3.mp4
creating thread 4, please help me to download file4.wmv
creating thread 5, please help me to download file5.flv
I am downloading the file file2.rmvb!
I am downloading the file file4.wmv!
I am downloading the file file5.flv!
I am downloading the file file3.mp4!
I am downloading the file file1.avi!
I finish downloading the file within 83 minutes!
I finish downloading the file within 86 minutes!
I finish downloading the file within 77 minutes!
I finish downloading the file within 15 minutes!
I finish downloading the file within 93 minutes!
Thread 1 downloads the file file1.avi in 93 minutes.
Thread 2 downloads the file file2.rmvb in 86 minutes.
Thread 3 downloads the file file3.mp4 in 15 minutes.
Thread 4 downloads the file file4.wmv in 83 minutes.
Thread 5 downloads the file file5.flv in 77 minutes.
[root@learn ~]# ./download.out 
creating thread 1, please help me to download file1.avi
creating thread 2, please help me to download file2.rmvb
I am downloading the file file1.avi!
creating thread 3, please help me to download file3.mp4
I am downloading the file file2.rmvb!
creating thread 4, please help me to download file4.wmv
I am downloading the file file3.mp4!
creating thread 5, please help me to download file5.flv
I am downloading the file file4.wmv!
I am downloading the file file5.flv!
I finish downloading the file within 83 minutes!
I finish downloading the file within 86 minutes!
I finish downloading the file within 77 minutes!
I finish downloading the file within 15 minutes!
I finish downloading the file within 93 minutes!
Thread 1 downloads the file file1.avi in 83 minutes.
Thread 2 downloads the file file2.rmvb in 86 minutes.
Thread 3 downloads the file file3.mp4 in 93 minutes.
Thread 4 downloads the file file4.wmv in 15 minutes.
Thread 5 downloads the file file5.flv in 77 minutes.
```

## 綫程的數據*(when and where?)*

---

在瞭解了并发多綫程的運作模式及優勢，緊接要解決綫程對三種不同位置類別數據的處理方式。

### 綫程棧上的本地數據

默認被賦予了每個綫程8MB的綫程棧空間，用於存儲其中的局部變量。

```bash
[root@learn ~]# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 7144
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192            <<<
cpu time               (seconds, -t) unlimited
max user processes              (-u) 7144
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

而内存中綫程棧間有用小塊區域來隔離各自空間，儅被踏入則會引發[段錯誤](https://baike.baidu.com/item/%E6%AE%B5%E9%94%99%E8%AF%AF)***[(Segmentation fault)](https://en.wikipedia.org/wiki/Segmentation_fault)***。

可以通過函數pthread_attr_t來修改單個綫程棧大小，或者通過ulimit -s來進行全局修改。

```bash
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);

ulimit -s 16384
```

### 進程中共享的全局數據

綫程在讀取進程中共享的全局數據時，爲避免數據不一，需要**等待互斥鎖[*(pthread_mutex_lock()]*** 或**嘗試互斥鎖*[pthread_mutex_trylock()]*** 來保證獨占。 前者需要一直等待，後者則可以根據沒搶到的提示佛系搶佔。

- 無條件變量的等待互斥鎖

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/howmutexwork.jpg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/howmutexwork.jpg)

無互斥鎖下，受多綫程影響的全局變量***(money_of_tom + money_of_jerry)***：

```bash
[root@learn ~]# cat mutex.c 

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM_OF_TASKS 5

int money_of_tom = 100;
int money_of_jerry = 100;
//暂不启用mutex
//pthread_mutex_t g_money_lock;

void *transfer(void *notused)   //子綫程用於執行金額轉賬
{
  pthread_t tid = pthread_self();
  printf("Thread %u is transfering money!\n", (unsigned int)tid);
  //暂不启用mutex
  //pthread_mutex_lock(&g_money_lock);
  sleep(rand()%10);
  money_of_tom+=10;
  sleep(rand()%10);
  money_of_jerry-=10;
  //暂不启用mutex
  //pthread_mutex_unlock(&g_money_lock);
  printf("Thread %u finish transfering money!\n", (unsigned int)tid);
  pthread_exit((void *)0);
}

int main(int argc, char *argv[])  //使用主綫程來輸出總額方便觀察
{
  pthread_t threads[NUM_OF_TASKS];
  int rc;
  int t;
  //暂不启用mutex
  //pthread_mutex_init(&g_money_lock, NULL);

  for(t=0;t<NUM_OF_TASKS;t++){
    rc = pthread_create(&threads[t], NULL, transfer, NULL); //創建子綫程開啓轉賬
    if (rc){
      printf("ERROR; return code from pthread_create() is %d\n", rc);
      exit(-1);
    }
  }
  
  for(t=0;t<18;t++){
    //暂不启用mutex
    //pthread_mutex_lock(&g_money_lock);
    printf("money_of_tom + money_of_jerry = %d\n", money_of_tom + money_of_jerry);
    sleep(rand()%3);  //隨機輸出時間
    //暂不启用mutex
    //pthread_mutex_unlock(&g_money_lock);
  }
  //暂不启用mutex
  //pthread_mutex_destroy(&g_money_lock);
  pthread_exit(NULL);
}
[root@learn ~]# gcc mutex.c -lpthread -o before.out
[root@learn ~]# ./before.out 
Thread 4143912704 is transfering money!
Thread 4135520000 is transfering money!
Thread 4152305408 is transfering money!
Thread 4127127296 is transfering money!
money_of_tom + money_of_jerry = 200
Thread 4118734592 is transfering money!
money_of_tom + money_of_jerry = 200
money_of_tom + money_of_jerry = 210
money_of_tom + money_of_jerry = 210
money_of_tom + money_of_jerry = 210
Thread 4118734592 finish transfering money!
Thread 4143912704 finish transfering money!
money_of_tom + money_of_jerry = 220
money_of_tom + money_of_jerry = 230
money_of_tom + money_of_jerry = 230
money_of_tom + money_of_jerry = 230
money_of_tom + money_of_jerry = 230
money_of_tom + money_of_jerry = 230
Thread 4127127296 finish transfering money!
money_of_tom + money_of_jerry = 220
Thread 4152305408 finish transfering money!
money_of_tom + money_of_jerry = 210
money_of_tom + money_of_jerry = 210
money_of_tom + money_of_jerry = 210
Thread 4135520000 finish transfering money!
money_of_tom + money_of_jerry = 200
money_of_tom + money_of_jerry = 200
money_of_tom + money_of_jerry = 200
[root@learn ~]# ./before.out 
money_of_tom + money_of_jerry = 200
Thread 4152305408 is transfering money!
Thread 4135520000 is transfering money!
Thread 4118734592 is transfering money!
Thread 4143912704 is transfering money!
Thread 4127127296 is transfering money!
money_of_tom + money_of_jerry = 200
money_of_tom + money_of_jerry = 200
money_of_tom + money_of_jerry = 200
money_of_tom + money_of_jerry = 200
money_of_tom + money_of_jerry = 210
money_of_tom + money_of_jerry = 210
Thread 4143912704 finish transfering money!
money_of_tom + money_of_jerry = 230
money_of_tom + money_of_jerry = 230
money_of_tom + money_of_jerry = 230
money_of_tom + money_of_jerry = 240
Thread 4127127296 finish transfering money!
money_of_tom + money_of_jerry = 230
money_of_tom + money_of_jerry = 230
money_of_tom + money_of_jerry = 230
money_of_tom + money_of_jerry = 230
Thread 4152305408 finish transfering money!
Thread 4135520000 finish transfering money!
money_of_tom + money_of_jerry = 210
Thread 4118734592 finish transfering money!
money_of_tom + money_of_jerry = 200
money_of_tom + money_of_jerry = 200
```

有互斥鎖下，受多綫程影響的全局變量***(money_of_tom + money_of_jerry)***：

```bash
[root@learn ~]# cat mutex.c 

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM_OF_TASKS 5

int money_of_tom = 100;
int money_of_jerry = 100;
//第一次运行去掉下面这行
pthread_mutex_t g_money_lock;

void *transfer(void *notused)
{
  pthread_t tid = pthread_self();
  printf("Thread %u is transfering money!\n", (unsigned int)tid);
  pthread_mutex_lock(&g_money_lock);   //開始轉賬前，等待互斥鎖獨占共享變量。
  sleep(rand()%10);
  money_of_tom+=10;
  sleep(rand()%10);
  money_of_jerry-=10;
  pthread_mutex_unlock(&g_money_lock);  //完成轉賬後，釋放互斥鎖。
  printf("Thread %u finish transfering money!\n", (unsigned int)tid);
  pthread_exit((void *)0);
}

int main(int argc, char *argv[])
{
  pthread_t threads[NUM_OF_TASKS];
  int rc;
  int t;
  pthread_mutex_init(&g_money_lock, NULL); //開始創建綫程前，初始化一個互斥鎖&g_money_lock。

  for(t=0;t<NUM_OF_TASKS;t++){
    rc = pthread_create(&threads[t], NULL, transfer, NULL);
    if (rc){
      printf("ERROR; return code from pthread_create() is %d\n", rc);
      exit(-1);
    }
  }
  
  for(t=0;t<100;t++){
    pthread_mutex_lock(&g_money_lock);   //開始打印共享變量前，等待互斥鎖獨占共享變量。
    printf("money_of_tom + money_of_jerry = %d\n", money_of_tom + money_of_jerry);
    sleep(rand()%10);
    pthread_mutex_unlock(&g_money_lock);  //完成輸出后解鎖。
  }
  pthread_mutex_destroy(&g_money_lock); //全部完成後，銷毀鎖。
  pthread_exit(NULL);
}
[root@learn ~]# gcc mutex.c -lpthread -o after.out
[root@learn ~]# ./after.out 
money_of_tom + money_of_jerry = 200
Thread 4152305408 is transfering money!
Thread 4143912704 is transfering money!
Thread 4118734592 is transfering money!
Thread 4135520000 is transfering money!
Thread 4127127296 is transfering money!
money_of_tom + money_of_jerry = 200
money_of_tom + money_of_jerry = 200
...
money_of_tom + money_of_jerry = 200
Thread 4135520000 finish transfering money!
Thread 4152305408 finish transfering money!
Thread 4143912704 finish transfering money!
Thread 4118734592 finish transfering money!
money_of_tom + money_of_jerry = 200
money_of_tom + money_of_jerry = 200
...
money_of_tom + money_of_jerry = 200
Thread 4127127296 finish transfering money!
```

- 有條件變量的等待互斥鎖

```bash
[root@learn ~]# cat varmutex.c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM_OF_TASKS 3
#define MAX_TASK_QUEUE 11

char tasklist[MAX_TASK_QUEUE]="ABCDEFGHIJ";  //聲明含十個任務的數組
int head = 0;
int tail = 0;

int quit = 0;

pthread_mutex_t g_task_lock;                // 聲明互斥鎖 g_task_lock
pthread_cond_t g_task_cv;                   // 聲明條件變量 g_task_cv

void *coder(void *notused)                  // 定義子綫程工作内容
{
  pthread_t tid = pthread_self();           // 聲明子綫程的tid

  while(!quit){                            //  反運算符，不爲零則假，爲零則真

    pthread_mutex_lock(&g_task_lock);       // 等待互斥鎖
    while(tail == head){                    // 僅所有task完成後，tail==head
      if(quit){                             // 如果收到quit信號退出
        pthread_mutex_unlock(&g_task_lock); // 釋放互斥鎖
        pthread_exit((void *)0);            // 退出綫程
      }
      printf("No task now! Thread %u is waiting!\n", (unsigned int)tid);  // 打印等待綫程
      pthread_cond_wait(&g_task_cv, &g_task_lock);     // 一直等待條件變量和互斥鎖的入參將自動搶鎖。
      printf("Have task now! Thread %u is grabing the task !\n", (unsigned int)tid); //綫程id開始執行綫程
    }
    char task = tasklist[head++];          // 推進已完成head
    pthread_mutex_unlock(&g_task_lock);    // 釋放整個子綫程的互斥鎖
    printf("Thread %u has a task %c now!\n", (unsigned int)tid, task); //  獲得任務
    sleep(5);                              // 摸魚五秒后打印完成    
    printf("Thread %u finish the task %c!\n", (unsigned int)tid, task);
  }

  pthread_exit((void *)0);
}

int main(int argc, char *argv[])            // 定義主綫程工作内容 
{
  pthread_t threads[NUM_OF_TASKS];          // 聲明綫程。其中包含十個任務數組
  int rc;
  int t;

  pthread_mutex_init(&g_task_lock, NULL);   // 初始化互斥鎖
  pthread_cond_init(&g_task_cv, NULL);      // 初始化條件變量

  for(t=0;t<NUM_OF_TASKS;t++){       
    rc = pthread_create(&threads[t], NULL, coder, NULL); // 創建無屬性綫程，調用coder之後就繼續往下。
    if (rc){
      printf("ERROR; return code from pthread_create() is %d\n", rc);
      exit(-1);
    }
  }

  sleep(5);  // 等待五秒避免與子進程請求互斥鎖的衝突。

  for(t=1;t<=4;t++){
    pthread_mutex_lock(&g_task_lock);      // 等待互斥鎖
    tail+=t;                               // 標志待完成位為1開始標記，2，3，4
    printf("I am Boss, I assigned %d tasks, I notify all coders!\n", t); // 打印發佈任務數
    pthread_cond_broadcast(&g_task_cv);    // 使用pthread_cond_broadcast通知所有綫程的條件變量
    pthread_mutex_unlock(&g_task_lock);    // 之後吊銷發佈任務的互斥鎖
    sleep(20);                             // 並等待20秒后繼續發佈任務
  }

  pthread_mutex_lock(&g_task_lock);        // 請求互斥鎖
  quit = 1;                                // 不爲零，停止子綫程創建
  pthread_cond_broadcast(&g_task_cv);      // 並通知現有綫程
  pthread_mutex_unlock(&g_task_lock);      // 釋放互斥鎖

  pthread_mutex_destroy(&g_task_lock);     // 銷毀互斥鎖
  pthread_cond_destroy(&g_task_cv);        // 銷毀條件變量
  pthread_exit(NULL);
}

[root@learn ~]# gcc varmutex.c -lpthread -o varmutex.out
[root@learn ~]# ./varmutex.out
//招聘三个员工，一开始没有任务，大家睡大觉
No task now! Thread 3491833600 is waiting!
No task now! Thread 3483440896 is waiting!
No task now! Thread 3475048192 is waiting!
//老板开始分配任务了，第一批任务就一个，告诉三个员工醒来抢任务
I am Boss, I assigned 1 tasks, I notify all coders!
//员工一先发现有任务了，开始抢任务
Have task now! Thread 3491833600 is grabing the task !
//员工一抢到了任务A，开始干活
Thread 3491833600 has a task A now! 
//员工二也发现有任务了，开始抢任务，不好意思，就一个任务，让人家抢走了，接着等吧
Have task now! Thread 3483440896 is grabing the task!
No task now! Thread 3483440896 is waiting!
//员工三也发现有任务了，开始抢任务，你比员工二还慢，接着等吧
Have task now! Thread 3475048192 is grabing the task !
No task now! Thread 3475048192 is waiting!
//员工一把任务做完了，又没有任务了，接着等待
Thread 3491833600 finish the task A !
No task now! Thread 3491833600 is waiting!
//老板又有新任务了，这次是两个任务，叫醒他们
I am Boss, I assigned 2 tasks, I notify all coders!
//这次员工二比较积极，先开始抢，并且抢到了任务B
Have task now! Thread 3483440896 is grabing the task !
Thread 3483440896 has a task B now! 
//这次员工三也聪明了，赶紧抢，要不然没有年终奖了，终于抢到了任务C
Have task now! Thread 3475048192 is grabing the task !
Thread 3475048192 has a task C now! 
//员工一上次抢到了，这次抢的慢了，没有抢到，是不是飘了
Have task now! Thread 3491833600 is grabing the task !
No task now! Thread 3491833600 is waiting!
//员工二做完了任务B，没有任务了，接着等待
Thread 3483440896 finish the task B !
No task now! Thread 3483440896 is waiting!
//员工三做完了任务C，没有任务了，接着等待
Thread 3475048192 finish the task C !
No task now! Thread 3475048192 is waiting!
//又来任务了，这次是三个任务，人人有份
I am Boss, I assigned 3 tasks, I notify all coders!
//员工一抢到了任务D，员工二抢到了任务E，员工三抢到了任务F
Have task now! Thread 3491833600 is grabing the task !
Thread 3491833600 has a task D now! 
Have task now! Thread 3483440896 is grabing the task !
Thread 3483440896 has a task E now! 
Have task now! Thread 3475048192 is grabing the task !
Thread 3475048192 has a task F now! 
//三个员工都完成了，然后都又开始等待
Thread 3491833600 finish the task D !
Thread 3483440896 finish the task E !
Thread 3475048192 finish the task F !
No task now! Thread 3491833600 is waiting!
No task now! Thread 3483440896 is waiting!
No task now! Thread 3475048192 is waiting!
//公司活越来越多了，来了四个任务，赶紧干呀
I am Boss, I assigned 4 tasks, I notify all coders!
//员工一抢到了任务G，员工二抢到了任务H，员工三抢到了任务I
Have task now! Thread 3491833600 is grabing the task !
Thread 3491833600 has a task G now! 
Have task now! Thread 3483440896 is grabing the task !
Thread 3483440896 has a task H now! 
Have task now! Thread 3475048192 is grabing the task !
Thread 3475048192 has a task I now! 
//员工一和员工三先做完了，发现还有一个任务开始抢
Thread 3491833600 finish the task G !
Thread 3475048192 finish the task I !
//员工三没抢到，接着等
No task now! Thread 3475048192 is waiting!
//员工一抢到了任务J，多做了一个任务
Thread 3491833600 has a task J now! 
//员工二这才把任务H做完，黄花菜都凉了，接着等待吧
Thread 3483440896 finish the task H !
No task now! Thread 3483440896 is waiting!
//员工一做完了任务J，接着等待
Thread 3491833600 finish the task J !
No task now! Thread 3491833600 is waiting!
```

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/howmutexworkwithcondtionvar.jpeg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/howmutexworkwithcondtionvar.jpeg)

### 綫程的私有數據

與本地數據的區別在於，前者只是局部變量，私有數據是于綫程創建的全局變量，亦非進程的全局變量。

創建方式

```c
int pthread_setspecific(pthread_key_t key, const void *value)
```

獲得方式

```c
void *pthread_getspecific(pthread_key_t key)
```

## 結語

---

1. 主要闡述了三部分，what and why，how，when and where 。
2. 當中how通過代碼樣例，示範了創建綫程的步驟，創建並運行了一個多綫程樣例隨機運行的情況。
3. 在when and where中，通過比較，得出互斥鎖對於保護綫程操作數據的必要性。
4. 之後在三種不同的綫程數據中，重點解剖了有無變量控制下的互斥鎖工作過程。
5. 以下是總結貼圖。

    ![https://img.madebug.net/m4d3bug/images-of-website/master/blog/summarytasks.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/summarytasks.png)

6. 個人學習筆記，歡迎斧正。
