---
layout:     post
title:      "Android ADB 常用命令"
subtitle:   ""
date:       2017-12-25
author:     "Jason"
header-img: ""
catalog: false
tags:
    - Android
    - adb
---



**adb logcat的过滤基本用法**

1. adb logcat   //打印默认所有日志
2. adb logcat -s tag //打印带有tag标签的所有日志
3. adb logcat -v time //打印所有日志并带上时间
4. adb logcat -s tag -v time //打印带有tag标签的所有日志并带上时间
5. adb logcat > D:/log.txt  //把所有日志都打印到D盘目录下的log.txt中（Window系统下）
6. adb logcat -s tag > D:/log.txt //同上
7. adb logcat -s tag -v > D:/log.txt //同上

**adb 日志优先级过滤日志**

> 过滤项格式 : <tag>[:priority] , 标签:日志等级, 默认的日志过滤项是 ” *:I ” ;
>
> — V : Verbose (明细);
>
> — D : Debug (调试);
>
> — I : Info (信息);
>
> — W : Warn (警告);
>
> — E : Error (错误);
>
> — F: Fatal (严重错误);
>
> — S : Silent(Super all output) (最高的优先级, 可能不会记载东西);

比较常用的
adb logcat “*:W”
或者
adb logcat *:W

**使用管道过滤日志**

adb logcat | grep abc //过滤含有abc字符串的所有日志
adb logcat | grep -i abc //忽略大小写
