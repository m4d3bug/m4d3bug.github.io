---
title: 趣談Linux操作系統筆記II（之剖析系統綫程）
mathjax: true
copyright: true
comment: true
date: 2020-06-29 19:03:36
updated: 2020-07-11 18:33:33
categories:
- "Ops "
tags:
- "Linux "
- "趣談Linux操作系統筆記"
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/linux-tux-minimalism-4k-42-1280x800.jpg" width=50% /></center>

本文旨在剖析线程的5个W和1个H。

<!-- more -->

## 0x00 什么是线程(what and why?)

---

线程常常与进程一起谈及，可通过以下解释它们的关系：

> **进程相当于一个项目，而线程则是为了完成项目需求，而建立的一个个开发任务。**

因此，当开发任务间无先后次序时，拆分出支线来方便并发，可以使效率up up（科学的分工，符合[**宇宙法則**](https://zh.wikipedia.org/zh-tw/%E5%B8%95%E7%B4%AF%E6%89%98%E6%B3%95%E5%88%99)）。

## 0x01 如何创建线程(how?)

---
创建前，先了解调用线程的必经步骤。

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/howtaskwork.jpg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/howtaskwork.jpg)

根据流程图，构建与之对应的测试代码。

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM_OF_TASKS 5

//声明函数
void *downloadfile(void *filename) 
{
   printf("I am downloading the file %s!\n", (char *)filename);
   sleep(10);
   long downloadtime = rand()%100;
   printf("I finish downloading the file within %d minutes!\n", downloadtime);
   //次线程结束并返回随机生成的downloadtime
   pthread_exit((void *)downloadtime); 
}

int main(int argc, char *argv[])
   //声明数组
   char files[NUM_OF_TASKS][20]={"file1.avi","file2.rmvb","file3.mp4","file4.wmv","file5.flv"}; 
   //声明该数组为线程对象
   pthread_t threads[NUM_OF_TASKS];                                    
   int rc;
   int t;
   int downloadtime;

   
   //设置线程属性
   pthread_attr_t thread_attr;                                         
   //初始化线程
   pthread_attr_init(&thread_attr);                                   
   //设置其属性为主线程等待子线程，并获取其退出状态方便调试
   pthread_attr_setdetachstate(&thread_attr,PTHREAD_CREATE_JOINABLE);  

   for(t=0;t<NUM_OF_TASKS;t++){
     //打印
     printf("creating thread %d, please help me to download %s\n", t+1, files[t]);
	 //创建并开启线程 &thread_attr, 然后指定子线程的函数 downloadfie
     rc = pthread_create(&threads[t], &thread_attr, downloadfile, (void *)files[t]); 
     if (rc){
       printf("ERROR; return code from pthread_create() is %d\n", rc);
       exit(-1);
     }
   }

   //回收线程属性
   pthread_attr_destroy(&thread_attr);  

   for(t=0;t<NUM_OF_TASKS;t++){
     //等待线程结束，并取得返回的downloadtime数值
     pthread_join(threads[t],(void**)&downloadtime);  
     printf("Thread %d downloads the file %s in %d minutes.\n",t+1,files[t],downloadtime);
   }

   //主线程结束
   pthread_exit(NULL); 
}
```

对代码进行编译，多次运行观察线程的随机创建（线程创建全靠资源动态分配）。
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

## 0x02 线程的数据(when and where?)

---

了解完并发多线程的运作模式和优势，接着是线程对**三种不同位置类别数据**的处理方式。

### 线程栈上的本地数据

默认被赋予了每个线程8MB的线程栈空间，用于存储其中的局部变量。

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

而内存中线程栈间有用小块区域来隔离各自空间，当被踏入则会引发[段错误](https://baike.baidu.com/item/%E6%AE%B5%E9%94%99%E8%AF%AF)***[(Segmentation fault)](https://en.wikipedia.org/wiki/Segmentation_fault)***。

可以通过函数pthread_attr_t来修改单个线程栈大小，或者通过ulimit -s来进行全局修改。

```bash
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);

ulimit -s 16384
```

### 进程中共享的全局数据

线程在读取进程中共享的全局数据时，引入互斥锁**Mutex(Mutual Exclusion)**来避免数据不一，需要**等待互斥锁[(pthread_mutex_lock()]** 或**尝试互斥锁[pthread_mutex_trylock()]** 來保证独占。 前者需要一直等待，后者则可以根据没抢到的提示佛系抢占。

- 无条件变量的等待互斥锁

    ![https://img.madebug.net/m4d3bug/images-of-website/master/blog/howmutexwork.jpg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/howmutexwork.jpg)

    **有无锁的情况对比，可帮助明白锁的重要性。**

    **无互斥锁**，受多线程影响的全局变量**(money_of_tom + money_of_jerry)**：

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
    
	//创建子线程用于执行金额转账
    void *transfer(void *notused)   
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
    
	//主线程负责输出总额，子线程负责交易细节
    int main(int argc, char *argv[])  
    {
      pthread_t threads[NUM_OF_TASKS];
      int rc;
      int t;
      //暂不启用mutex
      //pthread_mutex_init(&g_money_lock, NULL);
    
      for(t=0;t<NUM_OF_TASKS;t++){
	    //创建子线程开启转账
        rc = pthread_create(&threads[t], NULL, transfer, NULL); 
        if (rc){
          printf("ERROR; return code from pthread_create() is %d\n", rc);
          exit(-1);
        }
      }
      
      for(t=0;t<18;t++){
        //暂不启用mutex
        //pthread_mutex_lock(&g_money_lock);
        printf("money_of_tom + money_of_jerry = %d\n", money_of_tom + money_of_jerry);
		//随机输出时间
        sleep(rand()%3);  
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

    **有互斥锁**，受多线程影响的全局变量**(money_of_tom + money_of_jerry)**：

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
	  //开始转账前，等待互斥锁独占共享变量
      pthread_mutex_lock(&g_money_lock);   
      sleep(rand()%10);
      money_of_tom+=10;
      sleep(rand()%10);
      money_of_jerry-=10;
	  //完成转账后，释放互斥锁
      pthread_mutex_unlock(&g_money_lock);  
      printf("Thread %u finish transfering money!\n", (unsigned int)tid);
      pthread_exit((void *)0);
    }
    
    int main(int argc, char *argv[])
    {
      pthread_t threads[NUM_OF_TASKS];
      int rc;
      int t;
	  //开始创建线程前，初始化一个互斥锁 &g_money_lock
      pthread_mutex_init(&g_money_lock, NULL); 
    
      for(t=0;t<NUM_OF_TASKS;t++){
        rc = pthread_create(&threads[t], NULL, transfer, NULL);
        if (rc){
          printf("ERROR; return code from pthread_create() is %d\n", rc);
          exit(-1);
        }
      }
      
      for(t=0;t<100;t++){
        //开始打印共享变量前，等待互斥锁独占共享变量
        pthread_mutex_lock(&g_money_lock);   
        printf("money_of_tom + money_of_jerry = %d\n", money_of_tom + money_of_jerry);
        sleep(rand()%10);
        //完成输出后解锁
        pthread_mutex_unlock(&g_money_lock);  
      }
      //全部完成后，销毁锁
      pthread_mutex_destroy(&g_money_lock); 
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

- 有条件变量的等待互斥锁

    ![https://img.madebug.net/m4d3bug/images-of-website/master/blog/howmutexworkwithcondtionvar.jpeg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/howmutexworkwithcondtionvar.jpeg)

    根据流程图，构建与之对应的测试代码。

```bash
[root@learn ~]# cat varmutex.c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM_OF_TASKS 3
#define MAX_TASK_QUEUE 11

char tasklist[MAX_TASK_QUEUE]="ABCDEFGHIJ";  //声明含十个任务的数组
int head = 0;
int tail = 0;

int quit = 0;

pthread_mutex_t g_task_lock;                // 声明互斥锁 g_task_lock
pthread_cond_t g_task_cv;                   // 声明条件变量 g_task_cv

void *coder(void *notused)                  // 定义子线程工作内容
{
  pthread_t tid = pthread_self();           // 声明子线程的tid

  while(!quit){                            //  反运算符，≠0为假，=0则真

    pthread_mutex_lock(&g_task_lock);       // 等待互斥锁
    while(tail == head){                    // 仅所有task完成后，tail==head
      if(quit){                             // 如果收到quit信号退出
        pthread_mutex_unlock(&g_task_lock); // 释放互斥锁
        pthread_exit((void *)0);            // 退出线程
      }
      // 打印等待线程
      printf("No task now! Thread %u is waiting!\n", (unsigned int)tid);  
      // 一直等待条件变量和互斥锁的入参将自动抢锁。
      pthread_cond_wait(&g_task_cv, &g_task_lock); 
      //线程id开始执行线程
      printf("Have task now! Thread %u is grabing the task !\n", (unsigned int)tid); 
    }
    char task = tasklist[head++];          // 推进已完成head
    pthread_mutex_unlock(&g_task_lock);    // 释放整个子线程的互斥锁
    printf("Thread %u has a task %c now!\n", (unsigned int)tid, task); //  获得任务
    sleep(5);                              // 摸鱼五秒后打印完成    
    printf("Thread %u finish the task %c!\n", (unsigned int)tid, task);
  }

  pthread_exit((void *)0);
}

int main(int argc, char *argv[])            // 定义主线程工作内容 
{
  pthread_t threads[NUM_OF_TASKS];          // 声明线程。其中包含十个任务数组
  int rc;
  int t;

  pthread_mutex_init(&g_task_lock, NULL);   // 初始化互斥锁
  pthread_cond_init(&g_task_cv, NULL);      // 初始化条件变量

  for(t=0;t<NUM_OF_TASKS;t++){  
    // 创建无属性线程，调用coder之后就继续往下。
    rc = pthread_create(&threads[t], NULL, coder, NULL); 
    if (rc){
      printf("ERROR; return code from pthread_create() is %d\n", rc);
      exit(-1);
    }
  }

  sleep(5);  // 等待五秒避免与子进程请求互斥锁的冲突。

  for(t=1;t<=4;t++){
    pthread_mutex_lock(&g_task_lock);      // 等待互斥锁
    tail+=t;                               // 标志待完成位为1开始标记，2，3，4
    printf("I am Boss, I assigned %d tasks, I notify all coders!\n", t); // 打印发布任务数
    pthread_cond_broadcast(&g_task_cv);    // 使用pthread_cond_broadcast通知所有线程的条件变量
    pthread_mutex_unlock(&g_task_lock);    // 吊销发布任务的互斥锁
    sleep(20);                             // 并等待20秒后继续发布任务
  }

  pthread_mutex_lock(&g_task_lock);        // 请求互斥锁
  quit = 1;                                // 不为0，停止子线程创建
  pthread_cond_broadcast(&g_task_cv);      // 并通知现有线程
  pthread_mutex_unlock(&g_task_lock);      // 释放互斥锁

  pthread_mutex_destroy(&g_task_lock);     // 销毁互斥锁
  pthread_cond_destroy(&g_task_cv);        // 销毁条件变量
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

### 线程的私有数据

与本地数据的区别在于，前者只是局部变量，私有数据是于线程创建的全局变量，亦非进程的全局变量。

#### 創建key对应的value

```c
int pthread_setspecific(pthread_key_t key, const void *value)
```

#### 获得key对应的value

```c
void *pthread_getspecific(pthread_key_t key)
```

## 0x03 结语

---

1. 主要阐述了三部分，what and why，how，when and where 。
2. 当中how通过代码样例，示范了创建线程的步骤，创建并运行了一个多线程样例随机运行的情况。
3. 在when and where中，通过比较，得出互斥锁对于保护线程操作数据的必要性。
4. 之后在三种不同的线程数据中，重点解剖了有无变量控制下的互斥锁工作过程。
5. 以下是总结贴图。

    ![https://img.madebug.net/m4d3bug/images-of-website/master/blog/summarytasks.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/summarytasks.png)

6. 个人学习笔记，欢迎斧正。
