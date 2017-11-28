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
规则4：阻塞所有的信号，保护对共享全局数据结构的访问
规则5：用`volatile`声明全局变量，高速编译器不要缓存变量在寄存器中，每次从内存中引用该变量，同时也应该暂时阻塞信号，保护每次对全局变量的访问
规则6：用`sig_atomic_t`声明标志。在处理程序中，处理程序写全局标志记录接收到了信号，主程序周期读标志，响应信号，最后清除标志。整型数据类型`sig_atomic_t`保证读写为原子操作，不可中断，因此可以安全读写`sig_atomic_t`变量，而不需要暂时阻塞信号。另外，原子性的保证只适合单个的读和写，不适用于`flag++`或者`flag=flag+10`这样的更新，因为它们需要多条指令

## 正确的信号处理
未处理信号不排队，所以每种类型最多只能有一个未处理的信号。如果正在处理，则第二个信号就简单丢弃，`不会排队`。如果存在一个未处理的信号就表明至少有一个信号到达了。
>不可以使用信号对来对其他进程中发生的事件计数

## 可移植的信号处理程序
Unix信号处理程序的缺陷在于不同的系统有不同的信号处理语义。
>signal函数的语义不同。有些系统在运行完处理程序后，需要显示重新设置
系统调用可以被中断，程序员必须手动重启被中断的系统调用

使用Posix标准定义的sigaction函数，明确指明需要的信号处理语义，Signal包装函数如下：
``` c
handler_t *Signal(int signum, handler_t *handler)
{
    struct sigaction action, old_action;

    action.sa_handler = handler;  
    sigemptyset(&action.sa_mask); /* Block sigs of type being handled */
    action.sa_flags = SA_RESTART; /* Restart syscalls if possible */

    if (sigaction(signum, &action, &old_action) < 0)
    {
        unix_error("Signal error");
    }
    return (old_action.sa_handler);
}
```
Signal包装函数语义如下：
>只有这个处理程序当前正在处理的那种类型的信号被阻塞
信号不会排队
被中断的系统调用会自动重启
一旦设置信号处理程序，会一直保持，直到Signal的handler参数为SIG_IGN或者SIG_DEF调用

## 同步流避免并发错误
在下面的代码情形中，main函数调用addjob和处理程序调用main函数调用addjob和处理程序调用deletejob间存在竞争。如果addjob首先获得调度，那么结果执行正确；如果没有那么结果就错误。另外，这样的错误非常难以调试，因为无法测试所有的交错情形。
``` c
void initjobs()
{
}

void addjob(int pid)
{
}

void deletejob(int pid)
{
}

/* WARNING: This code is buggy! */
void handler(int sig)
{
    int olderrno = errno;
    sigset_t mask_all, prev_all;
    pid_t pid;

    Sigfillset(&mask_all);
    while ((pid = waitpid(-1, NULL, 0)) > 0) { /* Reap a zombie child */
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
        deletejob(pid); /* Delete the child from the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    if (errno != ECHILD)
        Sio_error("waitpid error");
    errno = olderrno;
}

int main(int argc, char **argv)
{
    int pid;
    sigset_t mask_all, prev_all;

    Sigfillset(&mask_all);
    Signal(SIGCHLD, handler);
    initjobs(); /* Initialize the job list */

    while (1) {
        if ((pid = Fork()) == 0) { /* Child process */
            Execve("/bin/date", argv, NULL);
        }
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all); /* Parent process */  
        addjob(pid);  /* Add the child to the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);    
    }
    exit(0);
}
```
消除竞争的方法就是在调用fork之前，阻塞SIGCHLD信号，然后在调用addjob之后取消取消阻塞这些信号，从而保证子进程被添加到作业列表之后回收子进程。注意：`子进程继承了父进程的被阻塞集合`，所以小心解除子进程中阻塞的SIGCHLD信号。修正后的程序代码如下：
``` c
int main(int argc, char **argv)
{
    int pid;
    sigset_t mask_all, mask_one, prev_one;

    Sigfillset(&mask_all);
    Sigemptyset(&mask_one);
    Sigaddset(&mask_one, SIGCHLD);
    Signal(SIGCHLD, handler);
    initjobs(); /* Initialize the job list */

    while (1) {
        Sigprocmask(SIG_BLOCK, &mask_one, &prev_one); /* Block SIGCHLD */
        if ((pid = Fork()) == 0) { /* Child process */
            Sigprocmask(SIG_SETMASK, &prev_one, NULL); /* Unblock SIGCHLD */
            Execve("/bin/date", argv, NULL);
        }
        Sigprocmask(SIG_BLOCK, &mask_all, NULL); /* Parent process */  
        addjob(pid);  /* Add the child to the job list */
        Sigprocmask(SIG_SETMASK, &prev_one, NULL);  /* Unblock SIGCHLD */
    }
    exit(0);
}
```

## 显式地等待信号
有时候主程序需要显式地等待某个信号处理程序运行。
``` c
volatile sig_atomic_t pid;

void sigchld_handler(int s)
{
    int olderrno = errno;
    pid = Waitpid(-1, NULL, 0);
    errno = olderrno;
}

void sigint_handler(int s)
{
}

int main(int argc, char **argv)
{
    sigset_t mask, prev;

    Signal(SIGCHLD, sigchld_handler);
    Signal(SIGINT, sigint_handler);
    Sigemptyset(&mask);
    Sigaddset(&mask, SIGCHLD);

    while (1) {
        Sigprocmask(SIG_BLOCK, &mask, &prev); /* Block SIGCHLD */
        if (Fork() == 0) /* Child */
            exit(0);

        /* Parent */
        pid = 0;
        Sigprocmask(SIG_SETMASK, &prev, NULL); /* Unblock SIGCHLD */

        /* Wait for SIGCHLD to be received (wasteful) */
        while (!pid)
            ;

        /* Do some work after receiving SIGCHLD */
        printf(".");
    }
    exit(0);
}
```
上面代码执行正确，但由于while死循环导致CPU浪费。为了修复这个问题，在循环体内插入pause:
``` c
while(!pid)  /* Race! */
    pause();
```
需要采用while循环，是因为会收到一个或者多个SIGINT信号，pause会被中断。上述代码也存在严重的竞争条件：`如果在while测试后和pause之前收到SIGCHLD信号，pause会永远睡眠`。另外选择就是使用sleep替换pause，但由于无法确定休眠的间隔，如果间隔大小，循环浪费；如果太大，程序又太慢。
合适的解决办法是使用`sigsuspend`。
sigsuspend函数暂时用mask替换当前的阻塞集合，然后挂起该进程，直到收到一个信号，其行为要么是运行一个处理程序，要么是终止该进程。如果行为是终止，那么该进程不从sigsuspend返回就直接终止。如果行为是运行一个处理程序，那么sigsuspend从处理程序返回，恢复调用sigsuspend时原有的阻塞集合。`sigsuspend`函数等价于下述代码的`原子的`版本：
``` c
sigprocmask(SIG_SETMASK, &mask, &prev);
pause();
sigprocmask(SIG_SETMASK, &prev, NULL);
```
sigsuspend版本比原来的版本不浪费CPU，避免了引入pause带来的竞争，又比sleep更有效率。
``` c
volatile sig_atomic_t pid;

void sigchld_handler(int s)
{
    int olderrno = errno;
    pid = Waitpid(-1, NULL, 0);
    errno = olderrno;
}

void sigint_handler(int s)
{
}

int main(int argc, char **argv)
{
    sigset_t mask, prev;

    Signal(SIGCHLD, sigchld_handler);
    Signal(SIGINT, sigint_handler);
    Sigemptyset(&mask);
    Sigaddset(&mask, SIGCHLD);

    while (1) {
        Sigprocmask(SIG_BLOCK, &mask, &prev); /* Block SIGCHLD */
        if (Fork() == 0) /* Child */
            exit(0);

        /* Wait for SIGCHLD to be received */
        pid = 0;
        while (!pid)
            Sigsuspend(&prev);

        /* Optionally unblock SIGCHLD */
        Sigprocmask(SIG_SETMASK, &prev, NULL);

        /* Do some work after receiving SIGCHLD */
        printf(".");
    }
    exit(0);
}
```

# 非本地跳转
非本地跳转(nonlocal jump)是与本地跳转相对应的一个概念。本地跳转主要指的是类似于goto语句的一系列应用，当设置了标志之后，可以跳到所在函数内部的标号上。然而，本地跳转不能将控制权转移到所在程序的任意地点，不能跨越函数，因此也就有了`非本地跳转`。非本地跳转将控制直接从一个函数转移到另一个函数，而不需要经过正常的调用-返回序列。C语言里面提供了setjmp和longjmp函数来进行跨越函数之间的控制权的跳转，从而称之为非本地跳转。
``` c
#include <setjmp.h>
int setjmp(jmp_buf env);   /* setjmp返回0，longjmp返回非零 */
int sigsetjmp(sigjmp buf env, int savesigs); //可以被信号处理程序使用的版本
void longjmp(jmp_buf env, int value);   /* 从不返回 */
void siglongjmp(sigjmp buf env, int retval); //可以被信号处理程序使用的版本
```
setjmp函数在env缓冲区中保存当前调用环境，供后面的longjmp使用，并返回0。调用环境包括程序计数器、栈指针和通用目的寄存器。`setjmp`返回值不能被赋值给变量，不过可以安全地使用在switch和条件测试中。
``` c
rc = setjmp(env);  /* Wrong! */
```
longjmp函数从env缓冲区中恢复调用环境，然后触发一个从最近一次初始化env的setjmp调用的返回，并带有非零的返回值retval。
setjmp和longjmp之间的相互关系：
>setjmp函数只被调用一次，但返回多次：一次是当第一次调用setjmp，而调用环境保存在缓冲区env中时，一次是为每个相应的longjmp调用。另一方面，longjmp函数被调用一次，但从不返回。

longjmp如果跳过了中间释放已经分配了的某些数据结构，将会产生内存泄露。
非本地跳转的另一个重要应用是使一个信号处理程序分支到一个特殊的代码位置，而不是返回到被信号到达中断了的指令的位置。当用户在键盘上键入Ctrl+C时，程序使用信号和非本地跳转实现软重启。
``` c
sigjmp_buf buf;

void handler(int sig)
{
    siglongjmp(buf, 1);
}

int main()
{
    if (!sigsetjmp(buf, 1)) {
        Signal(SIGINT, handler);
        Sio_puts("starting\n");
    }
    else {
        Sio_puts("restarting\n");
    }

    while(1) {
        Sleep(1);
        Sio_puts("processing...\n");
    }
    exit(0); /* Control never reaches here */
}
```
当用户键入Ctrl+C时，内核发送SIGINT信号给进程，进程捕获信号，如果不使用非本地跳转，信号处理程序会将控制返回给被中断的循环，使用非本地跳转，控制返回到main函数的开始处。
为了避免竞争，必须在调用了sigsetjmp之后再设置信号处理程序，否则可能会在sigsetjmp为siglongjmp设置调用环境前运行处理程序；另外，在siglongjmp可达的代码中只调用安全函数。例如我们调用的安全函数Sio_puts和Sleep函数。不安全的函数exit是不可达的。

# 操作进程的工具
>strace 跟踪系统调用
ps 打印当前系统中的进程
top 打印当前进程资源使用
pmap 显示进程的内存映射
/proc 虚拟文件系统
