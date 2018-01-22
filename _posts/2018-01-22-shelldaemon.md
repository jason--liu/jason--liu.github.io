---
layout:     post
title:      "利用shell脚本实现守护进程"
subtitle:   ""
date:       2018-01-22
author:     "Jason"
header-img: ""
catalog: 
tags:
- shell
---



在程序开发过程中,经常需要使用一个进程来监视另一个进程,如过这个进程挂了,那么监视它的进程就会把它重新拉起来.这就是守护进程的作用,守护进程可以用C实现,下面介绍一种简单的实现方式,使用shell脚本.

```bash
#!/bin/sh
#添加本地执行路径
export LD_LIBRARY_PATH=./

while true; do
{
#启动一个循环，定时检查进程是否存在
# ps aux --> a 为显示其他用户启动的进程；
#            u 为显示启动进程的用户名与时间；
#            x 为显示系统属于自己的进程；
# ps aux | grep 可执行程序名 --> 在得到的当前启动的所有进程信息文本中，过滤出包含有指定文本        #  (即可执行程序名字)的信息文本行
# 注：假设 ps aux | grep 可执行程序名 有输出结果，但输出不是一条信而是两条，
# 一个为查找到的包含有指定文本(即可执行程序名字)的信息文本行(以换行符0x10结尾的文本为一行)，
# 一个为 grep 可执行程序名 ，即把自己也输出来了，
# 所这条信息是我们不需要的，因为我们只想知指定名字的可执行程序是否启动了
# grep -v 指定文本 --> 输出不包含指定文本的那一行文本信息
target=`ps aux | grep appdemo | grep -v grep`
if [ ! "$target" ]; then
#如果不存在就重新启动
./appdemo  &     
sleep 5
fi
#每次循环休眠5s
sleep 5
} 2>> error.log #把错误日志导入error.log
done
```

