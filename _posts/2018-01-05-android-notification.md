---
layout:     post
title:      "Android APP如何使用通知灯"
subtitle:   "从APP到JNI层"
date:       2018-01-05
author:     "Jason"
header-img: "img/iphone-jobs.jpeg"
catalog: true
tags:
    - notification
---

本文采用情景分析的方法来分析应用程序如何使用通知灯,所谓情景分析,就是抓住一条主线,只关心与我们相关的代码,忽略不相关的代码.这个方法是我从[LINUX内核源代码情景分析](https://www.amazon.cn/dp/B001CK2WQW/ref=sr_1_1?ie=UTF8&qid=1515120958&sr=8-1&keywords=linux%E6%83%85%E6%99%AF)中学到的,在此向两位前辈毛德操,胡希明致敬.

## 应用程序中如何使用通知灯

```JAVA
NotificationManager manager = (NotificationManager)getSystemService(NOTIFICATION_SERVICE);
Notification notification = new NotificationCompat.Builder(this)
	notification.ledARGB = Color.GREEN;// 控制 LED 灯的颜色，一般有红绿蓝三种颜色可选  
				notification.ledOnMS = 1000;// 指定 LED 灯亮起的时长，以毫秒为单位  
				notification.ledOffMS = 1000;// 指定 LED 灯暗去的时长，也是以毫秒为单位  
				notification.flags = Notification.FLAG_SHOW_LIGHTS;// 指定通知的一些行为，其																//中就包括显示LED 灯这一选项  
	manager.notify(1, notification)
```

首先可以看到应用程序中首先获得一个系统服务`NOTIFICATION_SERVICE`,然后构造一个`notification`,其中就包括了对led闪烁条件的初始化,最后调用manager.notifi将通知发出去.

## Framework中如何调用

那就先看看`NOTIFICATION_SERVICE`服务是在哪里注册的,搜索它.可以看到在ContextImpl.java中注册了`NOTIFICATION_SERVICE`.我查看的代码是Android L,Android M有些许不同,但大同小异.

```java
 registerService(NOTIFICATION_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    final Context outerContext = ctx.getOuterContext();
                    return new NotificationManager(
                        new ContextThemeWrapper(outerContext,
                                Resources.selectSystemTheme(0,
                                        outerContext.getApplicationInfo().targetSdkVersion,
                                        com.android.internal.R.style.Theme_Dialog,
                                        com.android.internal.R.style.Theme_Holo_Dialog,
                                        com.android.internal.R.style.Theme_DeviceDefault_Dialog,
                                        com.android.internal.R.style.Theme_DeviceDefault_Light_Dialog)),
                        ctx.mMainThread.getHandler());
                }});
```

这段代码是在ContextImpl.java中的静态代码块中,也就是说它会是这个文件中最先执行的部分.再来看看`registerService`具体是如何实现的.

```java
private static void registerService(String serviceName, ServiceFetcher fetcher) {
        if (!(fetcher instanceof StaticServiceFetcher)) {
            fetcher.mContextCacheIndex = sNextPerContextServiceCacheIndex++;
        }
        SYSTEM_SERVICE_MAP.put(serviceName, fetcher);
    }
```

可以看到这个函数会把我们上面传进去的匿名类`ServiceFetcher`放入一个Hash Map.放入哈希表后,下次就可以根据`serviceName`来取`fetcher`.也就是应用程序中getSystemService来获取服务的过程

```java
public Object getSystemService(String name) {
        ServiceFetcher fetcher = SYSTEM_SERVICE_MAP.get(name);
        return fetcher == null ? null : fetcher.getService(this);
    }
```

这里获得了一个`fetcher`并调用它的`getService`函数.这里的`fetcher`就是我们前面注册时传入的匿名类.看看fetcher实现.

```java
static class ServiceFetcher {
        public Object getService(ContextImpl ctx) {
         ...
           //这里只保存了关键代码
                service = createService(ctx);
                cache.set(mContextCacheIndex, service);
                return service;
            }
        }
```

这里会调用service的createService函数,而上面定义的createService会返回一个`NotificationManager`实例.

所以`getSystemService`会返回`NotificationManager`实例.

接先来再看看应用程序中的manager.notify是如何实现的.

```java
//NotificationManager.java
 public void notify(String tag, int id, Notification notification)
    {
        INotificationManager service = getService();
        ...
        try {
            service.enqueueNotificationWithTag(pkg, mContext.getOpPackageName(), tag, id,stripped, idOut, UserHandle.myUserId());//这里涉及到Binder远程调用
            }
        } catch (RemoteException e) {
        }
    }
```

看下getService函数实现

```java
static public INotificationManager getService()
    {
        if (sService != null) {
            return sService;
        }
        IBinder b = ServiceManager.getService("notification");
        sService = INotificationManager.Stub.asInterface(b);
        return sService;
    }
```

看到这里就需要对Android的Binder机制比较了解才行了,后面我会陆续推出讲解Binder机制的文章,也不知道有没有人看哈,没人看也没关系,反正就当给自己工作学习过程留个记录,(*^__^*) 嘻嘻……

回归正题,我们来看看"notification"是在哪里注册的.

```java
//SystemServer.java
mSystemServiceManager.startService(NotificationManagerService.class);
//startService会调用NotificationManagerService的onStart函数
publishBinderService(Context.NOTIFICATION_SERVICE, mService);//这里就把'notification'服务注册进去了.
```

来看看mService里面有什么.

```java
private final IBinder mService = new INotificationManager.Stub(){
  ...
  public void enqueueNotificationWithTag(String pkg, String opPkg, String tag, int id,
                Notification notification, int[] idOut, int userId) throws RemoteException {
            enqueueNotificationInternal(pkg, opPkg, Binder.getCallingUid(),
                    Binder.getCallingPid(), tag, id, notification, idOut, userId);
        }
  ...
}
```

再来看看enqueueNotificationInternal函数

```java
/*和Led操作相关的*/
buzzBeepBlinkLocked(r);
/*然后会调用到*/
updateLightsLocked();
/*进而调用到*/
mNotificationLight.setFlashing(ledARGB, Light.LIGHT_FLASH_TIMED,ledOnMS, ledOffMS);
```

那mNotificationLight是谁,以及setFlashing是如何实现的呢?

```java
final LightsManager lights = getLocalService(LightsManager.class);
        mNotificationLight = lights.getLight(LightsManager.LIGHT_ID_NOTIFICATIONS);
```

lights是从一个ArrayMap里面get到的,那在哪里put的呢,可以搜索'LightsManager.class'发现

```java
//LightsService.java
public void onStart() {
        publishBinderService("hardware", mLegacyFlashlightHack);
        publishLocalService(LightsManager.class, mService);
    }
```

在LightsService.java里面会注册本地服务.LightsService就实现了上面的setFlashing函数

```java
   @Override
        public void setFlashing(int color, int mode, int onMS, int offMS) {
            synchronized (this) {
                setLightLocked(color, mode, onMS, offMS, BRIGHTNESS_MODE_USER);
            }
        }


private void setLightLocked(int color, int mode, int onMS, int offMS, int brightnessMode) {
            if (color != mColor || mode != mMode || onMS != mOnMS || offMS != mOffMS) {
                if (DEBUG) Slog.v(TAG, "setLight #" + mId + ": color=#"
                        + Integer.toHexString(color));
                mColor = color;
                mMode = mode;
                mOnMS = onMS;
                mOffMS = offMS;
                Trace.traceBegin(Trace.TRACE_TAG_POWER, "setLight(" + mId + ", " + color + ")");
                try {
                    setLight_native(mNativePointer, mId, color, mode, onMS, offMS, brightnessMode);
                } finally {
                    Trace.traceEnd(Trace.TRACE_TAG_POWER);
                }
            }
```

终于看到setLight_native函数了,它在JNI层.

## JNI层中操作LED

JNI层其实也只是一层过渡,它向上为JAVA层提供操作硬件的接口,向下调用HAL层,然后在HAL层中具体操作硬件.

```java
    static void setLight_native(JNIEnv *env, jobject clazz, jlong ptr,
                                jint light, jint colorARGB, jint flashMode, jint onMS, jint offMS, jint brightnessMode)
    {
        Devices* devices = (Devices*)ptr;
        light_state_t state;

        if (light < 0 || light >= LIGHT_COUNT || devices->lights[light] == NULL) {
            return ;
        }

        memset(&state, 0, sizeof(light_state_t));
        state.color = colorARGB;
        state.flashMode = flashMode;
        state.flashOnMS = onMS;
        state.flashOffMS = offMS;
        state.brightnessMode = brightnessMode;

        {
            ALOGD_IF_SLOW(50, "Excessive delay setting light");
            devices->lights[light]->set_light(devices->lights[light], &state);
        }
    
```

其实JNI层,没什么好说的,主要是通过硬件访问服务来操作硬件,关于硬件访问服务,可以看我以前写的博文.