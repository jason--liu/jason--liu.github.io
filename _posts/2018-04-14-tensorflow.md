---
layout:     post
title:      "Google 神经网络TensorFlow填坑日志"
subtitle:   ""
date:       2018-04-14
author:     "Jason"
header-img: "img/ai-robot.jpg"
catalog:
tags:
- kernel
---

## Tensorflow简介

TensorFlow™ 是一个使用数据流图进行数值计算的开源软件库。图中的节点代表数学运算， 而图中的边则代表在这些节点之间传递的多维数组（张量）。这种灵活的架构可让您使用一个 API 将计算工作部署到桌面设备、服务器或者移动设备中的一个或多个 CPU 或 GPU。 TensorFlow 最初是由 Google 机器智能研究部门的 Google Brain 团队中的研究人员和工程师开发的，用于进行机器学习和深度神经网络研究， 但它是一个非常基础的系统，因此也可以应用于众多其他领域。这段话是复制[官网](https://www.tensorflow.org/?hl=zh-cn)的。



## Python更新

首先TensorFlow对应Python2.x和Pytahon3.x都有，但由于Python3.x更普及，所以我决定还是安装Python3.x。

Python安装过程很简单，首先到[官网](https://www.python.org/)下载安装包 ，我安装的是Python最新版3.6。

然后就是

```shell
 ./configure
  make
  make test
  sudo make install
```

简单吧，就3条命令，更多详情请参考安装包里的README。

但有个问题，由于我用的是Ubuntu14.04 LTS系统，里面很多软件都依赖Python2.x，所以版本切换是个问题。不过问题解决起来也容易。

```shell
#首先对两个版本的Python设置优先级，值越大，优先级越高。
sudo update-alternatives --install /usr/bin/python python /usr/bin/python2 100
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3 150
```

配置好了，就可以一条命令选择了。

```shell
sudo update-alternatives --config python
```

会出现下面的选项

```shell
There are 3 choices for the alternative python (providing /usr/bin/python).

  Selection    Path                    Priority   Status
------------------------------------------------------------
  0            /usr/local/bin/python3   200       auto mode
  1            /usr/bin/python2         100       manual mode
  2            /usr/bin/python3         150       manual mode
* 3            /usr/local/bin/python3   200       manual mode

Press enter to keep the current choice[*], or type selection number:
```

这里我选3，对于python3.6。好了，Python更新好了，就可以下载安装Tensorflow了。

## Tensorflow安装

较新版的Tensorflow直接支持pip安装，非常方便，而pip在Python3.x中在安装Python的时候已经安装好了，所以也不用管了。但由于国情原因，TensorFlow在国内下载不是很方便，所以使用镜像站。这里我使用的清华大学的[镜像](https://mirror.tuna.tsinghua.edu.cn/help/tensorflow/)站，然后就是安装了。

```shell
pip3 install \
  -i https://pypi.tuna.tsinghua.edu.cn/simple/ \
  https://mirrors.tuna.tsinghua.edu.cn/tensorflow/linux/cpu/tensorflow-1.7.0-cp36-cp36m-linux_x86_64.whl
```

好了，坑来了，总是报ssl module in Python is not available的错误。

```shell
pip is configured with locations that require TLS/SSL, however the ssl module in Python is not available.  
Collecting bottle  
```

在网上Google了好久，才找到解决方法，留个记号。

```shell
#首先安装ssl,其实我已经安了
sudo apt-get install openssl   
sudo apt-get install libssl-dev  

#到Python安装包目录下，修改Modules/Setup
#找到
# Socket module helper for socket(2)
#_socket socketmodule.c

# Socket module helper for SSL support; you must comment out the other
# socket line above, and possibly edit the SSL variable:
#SSL=/usr/local/ssl
#_ssl _ssl.c \
#       -DUSE_SSL -I$(SSL)/include -I$(SSL)/include/openssl \
#       -L$(SSL)/lib -lssl -lcrypto
#####################################################################修改为
# Socket module helper for socket(2)
_socket socketmodule.c

# Socket module helper for SSL support; you must comment out the other
# socket line above, and possibly edit the SSL variable:
#SSL=/usr/local/ssl
_ssl _ssl.c \
        -DUSE_SSL -I$(SSL)/include -I$(SSL)/include/openssl \
        -L$(SSL)/lib -lssl -lcrypto
#然后重新安装一次
 ./configure
  make
  sudo make install
```

再执行上面的Tensorflow安装命令就可以了，不过可能需要加sudo权限。

## 测试

测试是否安装成功

```shell
#打开终端，启动Python.
Python 3.6.5 (default, Apr 11 2018, 21:12:06) 
[GCC 4.8.4] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> tf.__version__
'1.7.0'
>>> tf.__path__
['/usr/local/lib/python3.6/site-packages/tensorflow']
```

可以看到，tensorflow版本和路径都能查到了，所以应该是安装成功了。
