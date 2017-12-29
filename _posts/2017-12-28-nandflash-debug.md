---
layout:     post
title:      "NandFlash ECC软校验错误调试"
subtitle:   ""
date:       2017-12-29
author:     "Jason"
header-img: ""
catalog: true
tags:
    - Driver
    - NandFlash
---
## 问题描述

公司的的板子要换NandFlash,所以需要先验证下.以前NandFlash一直使用的是软校验的方法,按理说这部分不用动,系统就应该正常启动,并挂载文件系统.因为U-boot里面的软校验算法就是移植内核里面的.

可实际情况确实,内核能起来,但挂载文件系统的时候报ECC ERROR.

## 问题解决

由于文件系统是u-boot烧写进去的,那就先比较下文件系统在内核里面和在u-boot里面的ecc码.

> nanddump -l 2028 -p -o /dev/mtd5
>
> dump出来的OOB数据
>
> ```
>   OOB Data: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
>   OOB Data: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
>   OOB Data: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
>   OOB Data: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
>   OOB Data: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
>   OOB Data: a5 a9 9b ff ff ff ff ff ff ff ff ff ff ff ff ff
>   OOB Data: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
>   OOB Data: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 
> ```

然后再在uboot中dump

>nand dump.oob 0x00d80000
>
>OOB:
>
>        ff ff ff ff ff ff ff ff
>        ff ff ff ff ff ff ff ff
>        ff ff ff ff ff ff ff ff
>        ff ff ff ff ff ff ff ff
>        ff ff ff ff ff ff ff ff
>        a9 95 57 ff ff ff ff ff
>        ff ff ff ff ff ff ff ff
>        ff ff ff ff ff ff ff ff

先不说数据对不对,结果的位数都不同,内核打印出来的是128kb的,而uboot打印出来的是64kb的.我们使用的新nandflash的oob大小也的确是128kb的.那怎么该呢?为了兼容以前的版本,我们的uboot不能修改,所有只能修改内核.修改也很简单,内核里面使用ONFI的方式来获取oobsize的大小,那就不管ONFI读出来的大小是多少,我都定死为64kb,那问题就解决了.驱动调试就是这样,把结果直接说出来好像没什么技术含量,但发现问题的过程还是很复杂的.比如首先得知道ECC校验码是存在OOB里面的,如果使用硬件校验的方式,如果OOB大小不够,还可以在驱动里面把NandFlash划分一块区域出来用来指定存放ECC校验数据.

那u-boot里面为什么会认为OOB为64Kb呢?本着追究问题本质的原则,查看了uboot的源码,原来uboot里面没用使用ONFI的方式来获取一些nandflash信息,比如oob大小,那它就使用默认的64kb大小.其实只需要打开`CONFIG_SYS_NAND_ONFI_DETECTION`,就可以使用ONFI的方式来获取flash信息了.

## 参考资料

[如何编写Linux下的NandFlash驱动](https://www.crifan.com/files/doc/docbook/linux_nand_driver/release/webhelp/ch01s03s01s01.html)

[ONFI官方介绍](http://www.onfi.org/specifications/)






