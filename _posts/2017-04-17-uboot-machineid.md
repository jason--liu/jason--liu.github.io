---
layout: post
title: "uboot和内核machine id匹配"
modified:
categories: blog
excerpt:
tags: [uboot]
image:
  feature:
modified:
---
在调试新版uboot和内核时,可能出现machine id不匹配的问题,解决办法有两种.

```shell
uboot版本:2011.09
kernel版本:3.2.0
```



### 修改uboot源码

```c
/* 目录:board/ti/am335x/evm.c */
 
gd->bd->bi_arch_number = MACH_TYPE_TIAM335EVM	/* MACH_TYPE_TIAM335EVM 就是machine id */
  /* uboot中所有的machine id都定义在
  *  arch/arm/include/asm/mach-types.h文件中
  */
```

uboot中的bi_arch_number需要与内核中的id匹配,因为内核就是根据machine id来确定使用哪个单板.

### 修改kernel源码

```c
/* 目录:arch/arm/mach-omap2/board-am335x.c */

MACHINE_START(AM335XEVM, "yt3358")
	/* Maintainer: Texas Instruments */
	.atag_offset	= 0x100,
	.map_io		= am335x_evm_map_io,
	.init_early	= am33xx_init_early,
	.init_irq	= ti81xx_init_irq,
	.handle_irq     = omap3_intc_handle_irq,
	.timer		= &omap3_am33xx_timer,
	.init_machine	= am335x_evm_init,
MACHINE_END
```

这里的宏AM335XEVM就是machine id. 在内核目录中搜索发现它定义在arch/arm/tools/mach-types文件中,arm架构所支持的所有单板machine id都在这个文件中.

