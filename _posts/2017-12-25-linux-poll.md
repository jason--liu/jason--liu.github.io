---
layout:     post
title:      "Linux字符设备驱动Poll机制"
subtitle:   ""
date:       2017-12-25
author:     "Jason"
header-img: "img/startup-photos.jpg"
catalog: true
tags:
    - Linux
    - poll
---



## Linux poll机制

在用户空间向驱动程序请求数据时，有以下几种方式：

- 不断查询，条件不满足的时候就是死循环，很耗CPU
- 休眠唤醒方式，如果条件不满足，应用程序则一直睡眠下去（缺点）
- poll机制，如果条件不满足，休眠指定的时间，休眠时间内条件满足则唤醒，条件一直不满足时间到达自动唤醒
- 异步通知，应用程序注册信号处理函数，驱动程序发送信号

看看调用过程

```sh
do_sys_poll
	poll_initwait(&table);
		init_poll_funcptr(&pwq->pt, __pollwait);
			pt->qproc = qproc;	//table->pt-qproc = __pollwait
			
	do_poll(nfds, head, &table, end_time);
		for (;;) {
			for (; pfd != pfd_end; pfd++) {
			
					if (do_pollfd(pfd, pt))// 这里会调用驱动函数的poll函数
					{
						count++;
						pt = NULL;
					}
				}
			}
			/*
			 * All waiters have already been registered, so don't provide
			 * a poll_table to them on the next loop iteration.
			 */
			pt = NULL;
			if (!count) {
				count = wait->error;
				if (signal_pending(current))
					count = -EINTR;
			}
			//如果count=0,或者超时则返回
			if (count || timed_out)	
				break;

			}
		}
```

驱动程序的poll函数一般实现方法

```c
static unsigned int xxx_poll(struct file *file, poll_table * wait)
{
	
	unsigned int mask = 0;
	poll_wait(file, &(cdev->recvwait), wait); //将当前进程挂入某个队列
	mask = POLLOUT | POLLWRNORM;	//设置要监听的事件类型
	if (!skb_queue_empty(&cdev->recvqueue)) //判断监听条件
		mask |= POLLIN | POLLRDNORM;
	return mask;
}
```

## Linux 异步通知机制

- 应用程序注册信号处理函数
- 驱动发送信号
- APP要告诉驱动PID
- 驱动使用kill_fasync发送信号

为了使设备支持异步通知机制，驱动程序中主要设计以下３项工作

１．支持F_SETOWN命令，能在这个控制命令处理中设置file->f_owner为对应进程PID.不过此项工作已由内核完成，设备驱动无需处理

２．支持F_SETFL命令的处理，每当FASYNC标志改变时，驱动程序中的fasync()函数将被执行．

３．在设备资源可获得时，调用kill_fasync()函数激发相应的信号

```c
.fasync = hpet_fasync,

static int hpet_fasync(int fd, struct file *file, int on)
{
  ...
	if (fasync_helper(fd, file, on, &devp->hd_async_queue) >= 0)
		return 0;
	else
		return -EIO;
  ...
}

static irqreturn_t hpet_interrupt(int irq, void *data)
{
  	kill_fasync(&devp->hd_async_queue, SIGIO, POLL_IN); //如果检测到IO，发送信号
	return IRQ_HANDLED;
}

/*APP
 * 告诉驱动程序它需要向哪个进程发送信号
 */
fcntl(fd, F_SETOWN, getpid());
oflags = fcntl(fd, F_GETFL);
fcntl(fd, FSETFL, oflags | FASYNC);//改变fasync标记，最终会调用到驱动fasync->fasync_helper，
//创建或释放fasync_struct结构
```

## Linux 同步互斥阻塞机制

### 信号量

信号量(semaphore)是用于保护临界区的一种常用方法．只有得到信号量的进程才能执行临界区的代码．当获取不到信号量时，进程进入休眠等待状态

定义信号量

```sh
struct semaphore sem;
```

初始化信号量

```c
void sema_init(struct semaphore *sem, int val);
void int_MUTEX(struct semaphore *sem,); //初始化为０

/* 上面两个函数的结合　*/
static DECLARE_MUTEX(button_lock);//定义并初始化互斥锁
```

获得信号量

```c
/* 获取信号量，　获取不到就休眠　*/
/* 比如一个进程open了，另外一个进程open的时候就获取不到信号量，就休眠　*/
/* 只有当第一个进程释放了信号量，这个进程才能获得信号量，才能唤醒　*/
void down(struct semaphore *sem);
/*　获取不到就休眠，且休眠状态可被打断　*/
int down_interruptible(struct semaphore *sem);
/* 试图获取信号量，如果获取不到，立即返回，就不会休眠　*/
int down_trylock(struct semaphore *sem);
```

释放信号量

```c
void up(struct semaphore *sem);
```

### 阻塞

阻塞操作

是值在执行设备操作时若不能获得资源则挂起进程，直到满足可操作的条件后再进行操作．

被挂起的进程进入休眠状态，被从调度器的运行队列移走，直到等待的条件被满足．

非阻塞操作

进程在不能进行设备操作时并不挂起，它或者放弃，或者不停地查询，直到可以进行操作为止．

举个例子，当读一个按键操作时，如果没有按下按键，驱动不返回，进程就休眠，这是阻塞．如果没有按下按键，驱动立即返回一个错误，这是非阻塞

```c
int open(struct inode *inode, struct file *file)
{
    if (file->f_flags & O_NONBLOCK)
      {
        if (down_trylock(&button_lock))
          return -EBUSY;
      }
}

int read()
{
 if (file->f_flags & O_NONBLOCK)
   {
     if (!ev_press)
       return -EAGAIN;
   }
 }
```

