---
title: CSAPP之异常控制流
tags:
  - CSAPP
toc: true
date: 2017-11-20 11:40:58
---
# 前言
从开机到关机，处理器做的工作其实很简单，就是不断读取并执行指令，每次执行一条，整个指令执行的序列，称为处理器的控制流。对于程序内部本身的控制，可以通过跳转、调用和返回等程序指令实现；然而，如果需要对系统状态变化做出反应，因为系统状态不能被程序变量捕获，有时甚至与程序的执行无关。比如说定时器产生信号，数据包到达网卡，程序向磁盘请求数据等情况发生时，需要采用另外的机制对上述的异常情形做出反应，这些突变就称为异常控制流(Exception Control Flow,ECF)。
ECF存在于系统的每个层级，最底层的机制称为异常(Exception)，用以改变控制流以响应系统事件，通常是由硬件的操作系统共同实现的。更高层次的异常控制流包括进程切换(Process Context Switch)、信号(Signal)和非本地跳转(Nonlocal Jumps)，也可以看做是一个从硬件过渡到操作系统，再从操作系统过渡到语言库的过程。进程切换是由硬件计时器和操作系统共同实现的，而信号则只是操作系统层面的概念了，到了非本地跳转就已经是在C运行时库中实现的了。

# 进程控制
进程是计算机科学中最为重要的思想之一，进程是运行实例。进程给每个应用提供了两个非常关键的抽象：一是逻辑控制流，二是私有地址空间。逻辑控制流通过称为上下文切换(context switching)的内核机制让每个程序都感觉自己在独占处理器。私有地址空间则是通过称为虚拟内存(virtual memory)的机制让每个程序都感觉自己在独占内存。这样的抽象使得具体的进程不需要操心处理器和内存的相关适宜，也保证了在不同情况下运行同样的程序能得到相同的结果。对于fork关系复杂的进程，建议使用`进程图`画出各个进程的关系，梳理所有调用时序的可能性。

## 僵尸进程
当一个进程由于某种原因终止时，内核并不立即从系统中删除。而是进程被保持在一种已终止的状态，直到被它的父进程`回收`。所以，当一个进程完成它的工作终止之后，它的父进程需要调用`wait()或者waitpid()`系统调用取得子进程的终止状态。<strong>一个终止了但未被回收的进程称为“僵尸进程”。</strong>系统进程表是一项有限资源，如果系统进程表被僵尸进程耗尽的话，系统就可能无法创建新的进程。模拟产生僵尸进程示例：
<!--more-->
``` c
#include "csapp.h"
int main()
{
    if(Fork() == 0)
    {
        printf("child pid %d\n", getpid());
        exit(0);
    }
    while(1)
    {
        sleep(1);
        printf("parent pid %d\n", getpid());
    }
    exit(0);
}
```
僵尸进程查看命令：
``` bash
ps -A -o stat,ppid,pid,cmd | grep -e '^[Zz]'
```
使用Kill -HUP 僵尸进程ID来杀死僵尸进程，往往无法杀死僵尸进程，此时就需要杀死僵尸进程的父进程。因为，杀掉僵尸进程的父进程后，僵尸进程就变成了孤儿进程，孤儿进程会被init进程接管，init进程会wait()这些孤儿进程，释放它们占用的系统进程表中的资源，这样，这些已经“僵尸”的孤儿进程所占用的资源就被系统回收了。kill僵尸进程父进程命令：
``` bash
ps -A -o stat,ppid,pid,cmd | grep -e '^[Zz]' | awk '{print $2}' | xargs kill -9
```

## 孤儿进程
在子进程还未退出之前，它的父进程就已经退出了，一个没有了父进程的子进程就是一个孤儿进程（orphan）。如果一个父进程终止了，内核会安排init进程成为孤儿进程的养父。init进程的PID为1，是在系统启动时由内核创建的，它不会终止，是所有进程的祖先。如果父进程没有回收它的僵尸子进程就终止了，那么内核会安排init进程去回收它们。模拟产生孤儿进程示例：
``` c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main()
{
    if (!fork()) {
        while(1)
        {
            sleep(2);
            printf("child pid [%d], ppid [%d]\n", getpid(), getppid());
        }
    }
    exit(0);
}

```

## 回收子进程
程序并不会按照特定的顺序回收子进程。子进程回收的顺序是计算机系统的属性。下面的回收程序，表现出`非确定性`，从而执行结果表现出不同的时序。
``` c
#include "csapp.h"
#define N 2

int main()
{
    int status, i;
    pid_t pid;
    /* Parent creates N children */
    for (i = 0; i < N; i++)
    {
        if ((pid = Fork()) == 0)  /* Child */
        {
            exit(100+i);
        }    
    }

    /* Parent reaps N children in no particular order */
    while ((pid = waitpid(-1, &status, 0)) > 0)
    {
    	if (WIFEXITED(status))
        {
            printf("child %d terminated normally with exit status=%d\n",
               pid, WEXITSTATUS(status));
        }
    	else
        {
            printf("child %d terminated abnormally\n", pid);
        }
    }

    /* The only normal termination is if there are no more children */
    if (errno != ECHILD)
    {
    	unix_error("waitpid error");    
    }
    exit(0);
}

```

如果需要按照指定的顺序回收子进程，需要使用子进程ID来等待每个子进程的结束。

``` c
#include "csapp.h"
#define N 2

int main()
{
    int status, i;
    pid_t pid[N], retpid;

    /* Parent creates N children */
    for (i = 0; i < N; i++)
    {
        if ((pid[i] = Fork()) == 0)  /* Child */
        {
            exit(100+i);
        }
    }


    /* Parent reaps N children in no particular order */
    i = 0;
    while ((retpid = waitpid(pid[i++], &status, 0)) > 0)
    {
    	if (WIFEXITED(status))
        {
            printf("child %d terminated normally with exit status=%d\n",
               pid, WEXITSTATUS(status));
        }
    	else
        {
            printf("child %d terminated abnormally\n", pid);
        }
    }

    /* The only normal termination is if there are no more children */
    if (errno != ECHILD)
    {
    	unix_error("waitpid error");    
    }
    exit(0);
}
```

# 信号
信号是操作系统中提供的一种进程间通讯机制。它是一种异步的通知机制，用来提醒进程一个事件已经发生。当一个信号发送给一个进程，操作系统中断了进程正常的控制流程，此时，任何非原子操作都将被中断。如果进程定义了信号的处理函数，那么它将被执行，否则就执行默认的处理函数。
如果信号已被发送但是未被接收，那么处于等待状态(pending)，同类型的信号至多只会有一个待处理信号(pending signal)。比如说进程有一个 SIGCHLD 信号处于等待状态，那么之后进来的 SIGCHLD 信号都会被直接扔掉。
进程也可以阻塞特定信号的接收，但信号的发送并不受控制，所以被阻塞的信号仍然可以被发送，不过直到进程取消阻塞该信号之后才会被接收。内核用等待(pending)位向量和阻塞(blocked)位向量来维护每个进程的信号相关状态。
一个待处理信号最多只能被接收一次。

## 发送信号
>使用/bin/kill向程序发送信号，例如`/bin/kill -9 pid`发送信号9(SIGKILL)到指定的pid进程。一个为负的pid会导致信号发送到进程组pid中的每个进程。
通过键盘让内核向每个前台进程发送SIGINT(SIGTSTP)信号，SIGINT(ctrl+c)默认终止进程；SIGTSTP(ctrl+z)默认挂起进程
用kill函数发送信号
用alarm函数发送信号

## 接收信号
信号预定义的默认行为：
>进程终止
进程终止并转储内存
进程停止直到被SIGCONT信号重启
进程忽略该信号

## 信号阻塞
内核会阻塞与当前在处理的信号同类型的其他正待等待的信号，也就是说，一个SIGINT信号处理程序是不能被另一个SIGINT信号中断的。如果想要显式阻塞，就需要使用sigprocmask函数以及其他一些辅助函数。临时阻塞SIGINT信号处理程序示例：
``` c
#include "csapp.h"

void sigint_handler(int sig)
{
    printf("Caught SIGINT!\n");
}

unsigned int snooze(unsigned int secs)
{
    unsigned int rc = sleep(secs);
    printf("Slept for %d of %d secs.\n", secs-rc, secs);
    return rc;
}

int main()
{
    /* Install the SIGINT handler */
    if (signal(SIGINT, sigint_handler) == SIG_ERR)
    {
        unix_error("signal error");
    }
    (void)snooze(5);

    sigset_t mask, prev_mask;
    Sigemptyset(&mask);
    Sigaddset(&mask, SIGINT);
    Sigprocmask(SIG_BLOCK, &mask, &prev_mask);

    (void)snooze(5); /* Wait for the receipt of a signal */

    Sigprocmask(SIG_SETMASK, &prev_mask, NULL);

    return 0;
}
```

## 编写信号处理程序
信号处理程序很麻烦，因为它们和主程序以及其信号处理程序并发运行，并且共享相同的全局数据结构，那么结果可能就不可预知。因此，我们要注意因为并行访问可能导致的数据损坏的问题。为了使信号处理程序安全的运行，需要遵守的原则：
>规则1：信号处理程序越简单越好，例如简单设置全局变量返回，其他信号处理相关的由主程序执行
规则2：只使用异步信号安全(可重入或者不能被信号处理程序中断)的函数，使用`man 7 signal`可以获得信号安全函数列表
规则3：在进入和退出的时候保存和恢复errno，从而信号处理程序就不会覆盖原有的errno值

# 总结
