## 前言

作为一个安卓程序猿不知道Android启动一样可以开发出来一个应用。那么了解Android系统的启动有什么好处呢？毕竟没有好处的事大家是不会去做的。

* 对Android系统有个整体的了解
* 可以学习到一些架构的思想
* 当然还可以应对一些高级面试升职加薪
* ···

下面就一起探索Android启动的方方面面吧，

就让我们从zygote谈起吧

## Zygote - 系统启动的灵魂

### 1、首先要考虑的就是zygote是什么，它的作用是什么

Zygote是所有Java进程的父进程，Zygote进程本身是由init进程孵化而来的。

它的第一个作用就是fork System Server进程，System Server是Zygote fork的第一个进程。第二个作用就是孵化其他进程。

`！Zygote fork要单线程`

`！Zygote的ipc没有采用binder而是socket`

### 2、其次它的启动流程是什么

1. init进程fork出zygote进程
2. 启动虚拟机，注册jni函数
3. 预加载系统资源
4. 启动SystemServer(下面会介绍)
5. 进入Socket Loop

### 3、最后它的工作原理

在最后一步进入loop循环会等待消息的到来，一旦有消息来了，就会执行runOnce函数，首先会调用readArgumentList方法读取消息发过来的参数列表。然后通过Zygote.forkAndSpecialize创建子进程，这个函数会返回两次，一次是从子进程返回的pid，一个是从父进程返回的pid并相应执行他们的逻辑。


## System Server启动

1. 通过Zygote.forkSystemService()启动systemserver进程，然后进行基本的初始化工作。
2. 接下来启动binder线程
3. 最后执行SystemServer().run()，这个run函数里面首先会创建主线程的looper，然后创建上下文，然后分批启动Service（startBootstrapService，startCoreService，startOtherService），最后Looper.loop()。
> 这里启动的Service指的是系统的服务，但是我们自己也可以创建Service，那么它们有什么区别呢？<br>
> 可以看这篇文章[Android系统服务和应用服务的区别](https://github.com/MickeyQiong/ANote/blob/main/note/Android%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1%E5%92%8C%E5%BA%94%E7%94%A8%E6%9C%8D%E5%8A%A1%E7%9A%84%E5%8C%BA%E5%88%AB.md)

## Launcher的启动

当服务都启动完了之后，就会调用systemReady函数，这里面又会调用startHomeActivityLocked函数，这个函数会启动一个LoaderTask，它的作用就是去PMS中查询所有已经安装的应用，然后把她们的图标展示出来。这样Android系统就启动起来了。

## ServiceManager的启动

### 启动流程

servicemanager的启动主要做了3件事：

* binder_open(128*1024) // 打开binder驱动，申请128k内存空间
* binder_become_context_manager // 注册成上下文
* binder_loop // 等待请求

### 怎么获取ServiceManager的Binder对象

根据0号handle值获取了一个BpBinder对象

### 怎么向ServiceManger添加服务

获取ServiceManger的Binder对象，然后发起一个add_service的调用，需要传两个参数一个事服务的名称，另一个事服务的binder对象。

### 怎么从ServiceManager获取服务

获取ServiceManger的Binder对象，然后调用find_service的binder调用，传入服务的名称。

## 参考文献

[Android 操作系统架构开篇](http://gityuan.com/android/)



