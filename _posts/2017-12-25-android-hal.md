---
layout:     post
title:      "Android硬件访问服务(二)"
subtitle:   " \"使用HAL操作硬件\""
date:       2017-12-25
author:     "Jason"
header-img: "img/android-jni.jpeg"
catalog: true
tags:
    - HAL
    - JNI
---

上一节介绍了使用JNI的方式访问硬件，但这样做有个缺点，如果修改了硬件访问接口，就需要重新编译整个Android系统，这显然非常不方便。那有其他办法吗？有的，Google公司在JNI和Linux Driver之间又添加了一层HAL，即硬件访问服务。由HAL层提供硬件访问的接口，JNI层加载HAL库，这样做的好处就是如果修改了硬件接口，只需要重新编译HAL库，而且有些厂家并不希望公开自己的具体硬件操作方法，从而起到硬件保密的作用。

那HAL层具体是怎么实现的呢？下面以一个led的硬件访问服务为例。

## 实现AIDL接口

首先创建一个AIDL接口文件

```java
package android.os;

/** {@hide} */
interface ILedService
{
    int ledCtrl(int which, int status);
}
```

这里就定义了一个硬件操作接口`ledCtrl`。修改frameworks/base/Android.mk，然后编译。会在

/out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/src/core/java/android/os/目录下生产ILedService.java接口文件。这是系统自动为我们生产的，以后讲解Binder的时候在细说。

## 实现LedService.java

```java
package com.android.server;
import android.os.ILedService;
public class LedService extends ILedService.Stub {
    private static final String TAG = "LedService";

    public LedService() {
        native_ledOpen();
    }

    public int ledCtrl(int which, int status) {
        return native_ledCtrl(which, status);//调用JNI本地方法
    }
    public native static int native_ledOpen();
    public native static void native_ledClose();
    public native static int native_ledCtrl(int which, int status);
}
```

LedService.java的作用就是作为Binder的服务端提供服务，由它来调用操作硬件的JNI接口函数。当然还需要将LedService.java注册进入SystemServer.java中。

```java
Slog.i(TAG, "Led Service");
led = new LedService();
ServiceManager.addService("led", led);//APP就是通过"led"来获得led服务的
```

## JNI层实现

```c
#define LOG_TAG "LedService"
#include "jni.h"
#include "JNIHelp.h"
#include "android_runtime/AndroidRuntime.h"
#include <utils/misc.h>
#include <utils/Log.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <hardware/led_hal.h>
namespace android
{
    static led_device_t* led_device;
    static jint led_open(JNIEnv *env, jobject cls)
    {
        jint err;
        hw_module_t* module;
        hw_device_t* device;

        ALOGI("native ledOpen ...");

        /* 1. hw_get_module */
        err = hw_get_module("led", (hw_module_t const**)&module);
        if (err == 0) {
            /* 2. get device : module->methods->open */
            err = module->methods->open(module, NULL, &device);
            if (err == 0) {
                /* 3. call led_open */
                led_device = (led_device_t *)device;
                return led_device->led_open(led_device);
            } else {
                return -1;
            }
        }

        return -1;
    }

    static jint led_ctrl(JNIEnv *env, jobject cls, jint which, jint status)
    {
        ALOGI("native ledCtrl %d, %d", which, status);
        return led_device->led_ctrl(led_device, which, status);
    }
    static void led_close(JNIEnv *env, jobject cls)
    {
        //ALOGI("native ledClose ...");
        //close(fd);
    }
    static JNINativeMethod method_table[] = {
        { "native_ledOpen", "()I", (void*)led_open},
        { "native_ledClose", "()V", (void*)led_close },
        { "native_ledCtrl", "(II)I", (void*)led_ctrl }
    };

    int register_android_server_LedService(JNIEnv *env)
    {
        return jniRegisterNativeMethods(env, "com/android/server/LedService",
                                        method_table, NELEM(method_table));
    }

};
```

然后修改services/core/jni/onload.cpp将我们的ledJNI层注册进去。我们前面创建了LedService.java，又修改了SystemServer.java，又添加了jni操作方法，还修改了onLoad.cpp。那么如何编译呢？通过分析frameworks/base/services/ Android.mk可知

```bash
...
include $(wildcard $(LOCAL_PATH)/*/jni/Android.mk)#包含所有目录下的JNI下面的Android.mk
...
include $(patsubst %,$(LOCAL_PATH)/%/Android.mk,$(services))#所有目录下的Android.mk
#而core目录下的Android.mk又会包含所有的JAVA文件
$(call all-java-files-under,java) \

```

所以只需要在frameworks/base/services下执行`mmm`命令就可以了。编译完成后系统启动后就已经包含了我们的led服务了，只不过我们还没有实现hal层，所以到led_open函数里面会返回`-1`。

##hw_get_module的过程

关于Java是如何调用到JNI层的，前面已经讲过，这里就不再啰嗦了。重点看`led_open`函数。通过`hw_get_module`找到一个名为`led`的module。JNI层调用HAL的实质就是加载so库的过程，而加载so库肯定会调用到`dlopen`函数，`dlopen`函数的第一个参数是路径，所以先看怎么构造这个路径。

```c
int hw_get_module(const char *id, const struct hw_module_t **module)
{
    return hw_get_module_by_class(id, NULL, module);
}

int hw_get_module_by_class(const char *class_id, const char *inst,
                           const struct hw_module_t **module)
{
  ...
 /* First try a property specific to the class and possibly instance */
    snprintf(prop_name, sizeof(prop_name), "ro.hardware.%s", name);
    if (property_get(prop_name, prop, NULL) > 0) {
        if (hw_module_exists(path, sizeof(path), name, prop) == 0) {
            goto found;
        }
    }

    /* Loop through the configuration variants looking for a module */
    for (i=0 ; i<HAL_VARIANT_KEYS_COUNT; i++) {
        if (property_get(variant_keys[i], prop, NULL) == 0) {
            continue;
        }
        if (hw_module_exists(path, sizeof(path), name, prop) == 0) {
            goto found;
        }
    }

    /* Nothing found, try the default */
    if (hw_module_exists(path, sizeof(path), name, "default") == 0) {
        goto found;
    }
  ...
    
    return load(class_id, path, module);
}

static int hw_module_exists(char *path, size_t path_len, const char *name,
                            const char *subname)
{
    snprintf(path, path_len, "%s/%s.%s.so",
             HAL_LIBRARY_PATH2, name, subname);
    if (access(path, R_OK) == 0)
        return 0;

    snprintf(path, path_len, "%s/%s.%s.so",
             HAL_LIBRARY_PATH1, name, subname);
    if (access(path, R_OK) == 0)
        return 0;

    return -ENOENT;
}
```

代码看起来不复杂，总结下过程。`hw_module_exists`在`"/vendor/lib/hw"， "/system/lib/hw"`目录下判断"name"."subname".so文件是否存在。而这里的name就是`led`，subname由一个结构体定义。

```c
static const char *variant_keys[] = {
    "ro.hardware",  /* This goes first so that it can pick up a different
                       file on the emulator. */
    "ro.product.board",
    "ro.board.platform",
    "ro.arch"
};
```

如果都不存在，再判断`led.default.so`是否存在。最后通过load加载模块。

```c

static int load(const char *id,
        const char *path,
        const struct hw_module_t **pHmi)
{
    int status = -EINVAL;
    void *handle = NULL;
    struct hw_module_t *hmi = NULL;

    /*
     * load the symbols resolving undefined symbols before
     * dlopen returns. Since RTLD_GLOBAL is not or'd in with
     * RTLD_NOW the external symbols will not be global
     */
    handle = dlopen(path, RTLD_NOW);
    if (handle == NULL) {
        char const *err_str = dlerror();
        ALOGE("load: module=%s\n%s", path, err_str?err_str:"unknown");
        status = -EINVAL;
        goto done;
    }

    /* Get the address of the struct hal_module_info. */
    const char *sym = HAL_MODULE_INFO_SYM_AS_STR;
    hmi = (struct hw_module_t *)dlsym(handle, sym);
    if (hmi == NULL) {
        ALOGE("load: couldn't find symbol %s", sym);
        status = -EINVAL;
        goto done;
    }

    /* Check that the id matches */
    if (strcmp(id, hmi->id) != 0) {
        ALOGE("load: id=%s != hmi->id=%s", id, hmi->id);
        status = -EINVAL;
        goto done;
    }

    hmi->dso = handle;

    /* success */
    status = 0;

    done:
    if (status != 0) {
        hmi = NULL;
        if (handle != NULL) {
            dlclose(handle);
            handle = NULL;
        }
    } else {
        ALOGV("loaded HAL id=%s path=%s hmi=%p handle=%p",
                id, path, *pHmi, handle);
    }

    *pHmi = hmi;

    return status;
}

```

下面再来看看HAL层的具体实现。首先需要知道一个`hw_module_t`里面可以包含多个`xx_device_t`结构体。比如后面会讲到的灯光子系统，它里面的`hw_module_t`就包含多个设备，比如背光灯，通知灯等。

##HAL层实现

```c
//led_hal.c
#define LOG_TAG "LedHal"
#include <hardware/led_hal.h>
#include <hardware/hardware.h>

#include <cutils/log.h>

#include <malloc.h>
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <math.h>

static int fd;

/** Close this device */
static int led_close(struct hw_device_t* device)
{
	close(fd);
	return 0;
}

static int led_open(struct led_device_t* dev)
{
	fd = open("/dev/gpiodev", O_RDWR);
	ALOGI("led_open : %d", fd);
	if (fd >= 0)
		return 0;
	else
		return -1;
}

static int led_ctrl(struct led_device_t* dev, int which, int status)
{
	int ret = ioctl(fd, status, which);
	ALOGI("led_ctrl : %d, %d, %d", which, status, ret);
	return ret;
}

static struct led_device_t led_dev = {
	.common = {
		.tag   = HARDWARE_DEVICE_TAG,
		.close = led_close,
	},
	.led_open  = led_open,
	.led_ctrl  = led_ctrl,
};

static int led_device_open(const struct hw_module_t* module, const char* id,
        struct hw_device_t** device)
{
	*device = &led_dev;
	return 0;
}

static struct hw_module_methods_t led_module_methods = {
    .open = led_device_open,
};

struct hw_module_t HAL_MODULE_INFO_SYM = {
    .tag = HARDWARE_MODULE_TAG,
    .hal_api_version = HARDWARE_HAL_API_VERSION,
    .id = "led",
    .name = "Default led HAL",
    .author = "The Android Open Source Project",
    .methods = &led_module_methods,
};

```

```c
//led_hal.h
#ifndef ANDROID_LED_INTERFACE_H
#define ANDROID_LED_INTERFACE_H
#include <stdint.h>
#include <sys/cdefs.h>
#include <sys/types.h>
#include <hardware/hardware.h>
__BEGIN_DECLS
struct led_device_t {
    struct hw_device_t common;

	int (*led_open)(struct led_device_t* dev);
	int (*led_ctrl)(struct led_device_t* dev, int which, int status);
};
__END_DECLS

#endif  // ANDROID_LED_INTERFACE_H
```



## APP如何使用服务

```java
 iLedService = ILedService.Stub.asInterface(ServiceManager.getService("led"));
 iLedService.ledCtrl(1, 1);
```

但是由于这里的ledCtrl接口属于hidden的，应用程序无法直接使用，需要导入classess.jar包。关于这部分，请参考另外一篇博文[AndroidStudio如何导入hidden类](https://jason--liu.github.io/2017/12/24/androidstudio-hidden/)。



