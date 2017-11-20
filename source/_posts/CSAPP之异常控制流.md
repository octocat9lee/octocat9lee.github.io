---
title: CSAPP之异常控制流
tags:
  - CSAPP
toc: true
date: 2017-11-20 11:40:58
---
# 前言

# 进程控制
对于fork关系复杂的进程，建议使用`进程图`画出各个进程的关系，梳理所有调用时序的可能性。
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
