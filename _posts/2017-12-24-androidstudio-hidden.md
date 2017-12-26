---
layout:     post
title:      "AndroidStudio如何使用系统hidden类"
subtitle:   ""
date:       2017-12-24
author:     "Jason"
header-img: "img/android-jni.jpeg"
catalog: true
tags:
    - AndroidStudio
---

在应用开发过程中，可能会需要使用到系统的方法。比如自己在系统中添加了自己的服务,而自己添加的接口都是hidden的.那么AndrodStuido怎么使用呢?
##生成classess

首先重新编译系统,不知道为什么,我直接在frameworks/base/services/目录下执行`mmm`命令,classess始终不会更新,所以需要重新编译系统.然后把classess.jar拷贝出来,AndroidStudio就可以导入了,现在介绍如何导入jar包.

## AS导入JAR包

点击Files,然后点击project files.

![project_structure](https://raw.githubusercontent.com/jason--liu/jason--liu.github.io/master/img/pj-structure.png)

点击左上角的+号.然后选择红色箭头指示的方框.

![](https://raw.githubusercontent.com/jason--liu/jason--liu.github.io/master/img/new-module.png)

成功导入过后,会在app下面出现classess.如下图所示.

![](https://raw.githubusercontent.com/jason--liu/jason--liu.github.io/master/img/classess.png)

点击Denpencies.如下图,依次点击

![](https://raw.githubusercontent.com/jason--liu/jason--liu.github.io/master/img/add-modules.png)

点击+号后选择modules.重新同步Graddle,就可以在AndroidStudio使用Android系统的hidden方法了.

## 错误解决

在编译过程中,可能会出现错误.比如

> ava.lang.OutOfMemoryError: GC overhead limit exceeded

只需要在app/build.gradle中加入如下代码,我使用的是android studio 2.3.3

```bash
dexOptions { 
		  javaMaxHeapSize "4g" 
}
```

然后重新编译就可以了.