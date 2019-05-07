---
layout:     post
title:      "Android开机启动优化"
subtitle:   ""
date:       2019-03-29
author:     "Jason"
header-img: "img/android-jni.jpeg"
catalog: true
tags:
    - Boot
---

最近在搞安卓开机优化，很不好搞，把遇到的问题先记录一下。

## 使用bootchart分析开机启动时间

### 编译bootchart

在/system/core/init中的Android.mk中加入`INIT_BOOTCHART:true`

```makefile
 LOCAL_PATH:= $(call my-dir)
 include $(CLEAR_VARS)
 INIT_BOOTCHART := true
```

使用`mmm .-B`强制编译下，会在`out/target/product/sabresd_6dq/root/`下生成init文件，因为init是在root目录下，所以需要重新烧写boot.img

### 生成开机log文件

在data目录下面，创建bootchart-start文件，然后echo 一个数字进去，这个数字是开机启动一直进行bootchart的时间。

```bash
echo 90 > bootchart-start
```

重启系统，data目录下会自动创建bootchart文件夹，里面有相关的log。需要注意的是要先`echo 1 > bootchart-stop`文件否则抓取下来的文件不能生成图片。

打包/data/bootchart/目录下的文件

```bash
tar -czf bootchart.tar *
```

### 生成png图片

首先需要在Ubuntu系统上装bootchart工具

```bash
sudo apt-get install bootchart
```

执行`bootchart bootchart.tar`生成bootchart.png图片

![](https://raw.githubusercontent.com/jason--liu/jason--liu.github.io/master/img/bootchart1.png)可以看到bootanimation结束的时间大约是16S的时候。因为之前系统已经优化过了，可以看到zygote和system_server里面IO wait已经很少了，可以优化的空间应该不大了。

## readahead提升开机速度

read调用过程，经过虚拟文件系统，具体文件系统层，page cache，块设备层，IO调度层，具体物理设备层

![read调用过程](https://www.ibm.com/developerworks/cn/linux/l-cn-read/images/image001.gif)

而readahead的功能就是提前将数据读到Page Cache中

注意要添加`CREATE_TRACE_POINTS`，但是只有一个文件能包含这个宏，否则编译不过

内核的解释

>/*Any file that uses trace points, must include the header.
>
>But only one file, must include the header by defining
>
>CREATE_TRACE_POINTS first.  This will make the C code that
>
>creates the handles for the trace points.
>*/
>#define CREATE_TRACE_POINTS

## 参考资料

[android bootchart 分析开机启动时间](https://www.twblogs.net/a/5b8653162b71775d1cd4ef9c/zh-cn)

[Android开机速度优化简单回顾](https://blog.csdn.net/freshui/article/details/53700771)

[添加readahead trace event](https://blog.csdn.net/jscese/article/details/46415531)

