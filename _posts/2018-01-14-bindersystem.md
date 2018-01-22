---
layout:     post
title:      "Android Binder系统分析"
subtitle:   ""
date:       2018-1-14
author:     "Jason"
header-img: "img/android-jni.jpeg"
catalog: true
tags:
    - Binder
---

毫不夸张地说，Binder是Android系统中最重要的特性之一；正如其名“粘合剂”所喻，它是系统间各个组件的桥梁，Android系统的开放式设计也很大程度上得益与这种及其方便的跨进程通信机制。

## Binder模型

Binder对用户来说主要分为注册服务，获取服务和使用服务，后面我会分别分析这三个过程的C++实现和JACA实现。

![](http://upload-images.jianshu.io/upload_images/944365-1630c69e48cb1deb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###addService

这里我先不具体分析Binder驱动代码，只是先简要介绍下Binder驱动在整个服务注册，服务获取和服务使用过程中是如何工作的。Server端addService过程。

- 为每一个服务构造flat_binder_object结构体
- 调用ioctl发送数据

  - 数据中有flat_binder_object
  - 服务的名字
  - 目的handle = 0(ServiceManager)
- 驱动程序为每一个flat_binder_object构造binder_node节点，一个binder_node节点代表一个服务。binder_node节点里面包含

  - ptr 对应函数处理方法
  - cookie
  - proc　指向当前进
- 驱动程序根据handle = 0找到SM，并把数据发送给SM的todo链表，并构造binder_ref节点，即服务的引用。

### ServiceManager

驱动程序会在SM的内核态构造binder_ref节点，里面包含

binder_ref:

- desc 一个数字，依据服务注册的顺序递增
- node  指向上面讲的binder_node节点

同时SM在用户态会构造一个serverlist链表，serverlist里面包含

- name 服务的名字
- handle 等于上面的desc

### getService

Client端获取服务过程

- 构造数据，数据中包含想要获取的服务名称和目的即handle＝０
- 调用ioctl发送数据
- 根据handle = 0找到SM
- SM从serverlist中根据name找到相应的handle,并返回flat_binder_object结构体，里面包含handle(想要获取服务的handle)
- 驱动程序会创建一个binder_ref节点，里面的node指向SM中binnder_ref的node

### 服务的使用

服务的使用过程，同样是先构造数据，code(想要调用哪个函数)，handle(对应服务的引用)，再调用ioctl发送数据

驱动程序会设置ptr和cookie,驱动程序根据handle找到binder_ref-->binder_node-->binder_proc.Server端接收到数据后根据prt和cookie调用不用的服务。

**小结**

- 最核心的函数ioctl
- client端最核心的数据handle
- server端最核心的数据ptr/cookie

下面就分别来分析C++和JAVA对这三个过程的具体实现，先来看C++实现。

## C++实现

下面以HelloService为例来看下服务的注册过程。

### addService

```C++
defaultServiceManager()->addService(String16("hello"), new BnHelloService());
//IserviceManager.cpp
 virtual status_t addService(const String16& name, const sp<IBinder>& service,
            bool allowIsolated)
    {
        Parcel data, reply;
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        data.writeString16(name);
        data.writeStrongBinder(service);//重点是这个
        data.writeInt32(allowIsolated ? 1 : 0);
        status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
        return err == NO_ERROR ? reply.readExceptionCode() : err;
    }

status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}

status_t flatten_binder(const sp<ProcessState>& /*proc*/,
    const sp<IBinder>& binder, Parcel* out)
{
 	 IBinder *local = binder->localBinder();//local指向了BnHelloService，下面单独分析
  ...
   		obj.type = BINDER_TYPE_BINDER;
        obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());//binder实体
        obj.cookie = reinterpret_cast<uintptr_t>(local);//把local转换为整数
  ...
}
```

单独分析下

```c
IBinder *local = binder->localBinder();
	BpBinder* IBinder::remoteBinder()
{
    return NULL;
}
BpBinder* BpBinder::remoteBinder()
{
    return this;
}
```

因为BpBinder是继承自IBinder的，并重写了remoteBinder方法，根据“父类引用指向子类对象”可知会调用BpBinder的remoteBinder函数，所以返回this,即BnHelloService.

### getService

客户端getService的过程实质上是获取BpBinder和handle的过程。

```c
//getService
//checkService
return reply.readStrongBinder();

sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    unflatten_binder(ProcessState::self(), *this, &val);
    return val;
}

status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
  		case BINDER_TYPE_HANDLE:
              *out = proc->getStrongProxyForHandle(flat->handle);//重点
               return finish_unflatten_binder(
               static_cast<BpBinder*>(out->get()), *flat, in);
}
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{	...
   b = new BpBinder(handle); 
	...
}
```

### 代理类如何发送数据

代理类如何发送数据，调用ioctl, 数据里含有handl, 含有其他构造的参数

```c
status_t err = remote()->transact(Hello_SERVICE, data, &reply);
```

这个remote是什么?

```c
inline  IBinder*        remote()                { return mRemote; }
IBinder* const          mRemote
```

也就是调用mRemote的transact函数，它是IBinder指针，而它的子类BpBinder实现了transact函数。

```c
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }
}
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
  if ((err=talkWithDriver()) < NO_ERROR) break;
  {if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)}//调用ioctl发送数据
}
```



### 服务的使用过程

Server如何分辨client想使用哪一个服务？

server收到的数据里含有flat_binder_object结构体，它可以根据.binder/.cookie分析client想使用哪一个服务。

```c
	IPCThreadState::self()->joinThreadPool();
joinThreadPool()
do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);
            abort();
        }
        
        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

status_t IPCThreadState::executeCommand(int32_t cmd)
{
  ...
    if (tr.target.ptr) {
                sp<BBinder> b((BBinder*)tr.cookie);
                error = b->transact(tr.code, buffer, &reply, tr.flags);
  ...
}
  status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
  {
    err = onTransact(code, data, reply, flags);
  }
  //因为BnHelloService继承BBinder，所以就调用到了它里面的onTransact函数
```

**小结**

如果要自己实现C++系统服务的话，需要实现

- C/S端(应用层)
- BpXXX.cpp和BnXX.cpp（RPC）
- IPC层系统实现

后面讲JAVA的时候会发现使用起来简单很多，只需要实现想要的服务，RPC和IPC层由来系统实现

## JAVA实现

