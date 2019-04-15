---
layout:     post
title:      "Linux信号编程，sigprocmask用法"
subtitle:   ""
date:       2019-04-15
author:     "Jason"
header-img: "img/android-jni.jpeg"
catalog: true
tags:
    - signal linux 
---

最近在学习nginx，看到里面有关于sigprocmask使用，查了些资料，顺便记录下。

## **sigset_t**

sigset_t是代表信号集合的数据结构。一个进程有一个信号集，这个信号集表示当前屏蔽(阻塞)了哪些信号。

如果把信号集中的某个位置置1，表示屏蔽了该信号，如果再来此信号，该进程是收不到的。

先介绍几个信号编程常用的系统API

```c
sigemptyset();//把信号集中的所有信号都清0，表示64个信号都没来  
sigfillset();//把信号集中所有信号都置1，即屏蔽所有信号，用得少  
sigaddset();//往信号集中增加信号  
sigdelset();//从信号集中删除信号(1变0)  
sigprocmask();//设置该进程对应的信号集中的内容  
sigismember();//测试一个指定信号是否被置位  
```

下面在着重讲下sigemptyset()和sigprocmask()。

## sigemptyset()

函数原型是

```c
int sigemptyset(sigset_t *set);
```

即初始化信号集并将所有信号清0。

## sigprocmask()

sigprocmask主要是设置该进程对应的信号集中的内容，这个函数稍微复杂点，先看看函数原型

```c
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```

参数how有如下取值

| how         | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| SIG_BLOCK   | SIG_BLOCK表明设置当前进程新的信号屏蔽字为 “当前信号屏蔽字" 和 第二个参数指向的信号集的并集，set包含了我们希望阻塞的附加信号 |
| SIG_UNBLOCK | 该进程新的信号屏蔽字是其当前信号屏蔽字和set所指向信号集补集的交集，set包含了我希望解除阻塞的信号 |
| SIG_SETMASK | 该进程新的信号屏蔽字将被set指向的信号集的值代替              |

## 实例

这些系统API直接讲解不容易理解，下面结合一个具体的例子看看。

```c
#include <stdio.h>
#include <stdlib.h>  //malloc
#include <unistd.h>
#include <signal.h>

//信号处理函数
void sig_quit(int signo)
{   
    printf("收到了SIGQUIT信号!\n");
    if(signal(SIGQUIT,SIG_DFL) == SIG_ERR)
    {
        printf("无法为SIGQUIT信号设置缺省处理(终止进程)!\n");
        exit(1);
    }
}

int main(int argc, char *const *argv)
{
    sigset_t newmask,oldmask; 
    if(signal(SIGQUIT,sig_quit) == SIG_ERR)  //注册信号对应的信号处理函数,"ctrl+\" 
    {        
        printf("无法捕捉SIGQUIT信号!\n");
        exit(1);   
        //退出程序，参数是错误代码，0表示正常退出，非0表示错误，但具体什么错误，没有特别规定
    }

    sigemptyset(&newmask); //newmask信号集中所有信号都清0（表示这些信号都没有来）；
    sigaddset(&newmask,SIGQUIT); //设置newmask信号集中的SIGQUIT信号位为1，也就是说，再来SIGQUIT信号时，进程就收不到，设置为1就是该信号被阻塞掉

    //sigprocmask()：设置该进程所对应的信号集
    if(sigprocmask(SIG_BLOCK,&newmask,&oldmask) < 0)  //第一个参数用了SIG_BLOCK表明设置 进程 新的信号屏蔽字 为 “当前信号屏蔽字 和 第二个参数指向的信号集的并集
    {                                                 //一个 ”进程“ 的当前信号屏蔽字，刚开始全部都是0的；所以相当于把当前 "进程"的信号屏蔽字设置成 newmask（屏蔽了SIGQUIT)；
                                                      //第三个参数不为空，则进程老的(调用本sigprocmask()之前的)信号集会保存到第三个参数里，用于后续，这样后续可以恢复老的信号集给线程
        printf("sigprocmask(SIG_BLOCK)失败!\n");
        exit(1);
    }
    printf("我要开始休息10秒了--------begin--，此时我无法接收SIGQUIT信号!\n");
    sleep(10);   //这个期间无法收到SIGQUIT信号的；
    printf("我已经休息了10秒了--------end----!\n");
    if(sigismember(&newmask,SIGQUIT))  //测试一个指定的信号位是否被置位(为1)，测试的是newmask
    {
        printf("SIGQUIT信号被屏蔽了!\n");
    }
    else
    {
        printf("SIGQUIT信号没有被屏蔽!!!!!!\n");
    }
    if(sigismember(&newmask,SIGHUP))  //测试另外一个指定的信号位是否被置位,测试的是newmask
    {
        printf("SIGHUP信号被屏蔽了!\n");
    }
    else
    {
        printf("SIGHUP信号没有被屏蔽!!!!!!\n");
    }

    //把信号集还原回去
    if(sigprocmask(SIG_SETMASK,&oldmask,NULL) < 0) 
    //第一个参数用了SIGSETMASK表明设置进程新的信号屏蔽字为第二个参数指向的信号集，第三个参数没用
    {
        printf("sigprocmask(SIG_SETMASK)失败!\n");
        exit(1);
    }

    printf("sigprocmask(SIG_SETMASK)成功!\n");
    
    if(sigismember(&oldmask,SIGQUIT))  //测试一个指定的信号位是否被置位,这里测试的当然是oldmask
    {
        printf("SIGQUIT信号被屏蔽了!\n");
    }
    else
    {
        printf("SIGQUIT信号没有被屏蔽，您可以发送SIGQUIT信号了，我要sleep(10)秒钟!!!!!!\n");
        int mysl = sleep(10);
        if(mysl > 0)
        {
            printf("sleep还没睡够，剩余%d秒\n",mysl);
        }
    }
    printf("end main!\n");
    return 0;
}
```

