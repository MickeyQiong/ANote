
首先我们知道作为一个服务，肯定是需要启动的，那么就先从服务启动方式进行对比。<br>

其次就是服务如果要被应用所访问的到，就需要注册到一个共有的地方，所以就需要对比一下注册方式。<br>

最后就是服务就是拿来使用的，所以还要对比一下使用上的区别。<br>

接下来我们就具体分析一下这三种区别：
### 启动方式的区别
 
**系统服务**一般都是跑在SystemServer上的，也是在SystemServer中被启动起来的，在系统启动SystemServer时顺便就会把这些服务启动起来，比如说我们熟悉的AMS、WMS和PMS等。所谓的启动服务并不是一定要启动工作线程，大部分的服务都是跑在Binder线程里面的，只有少部分的服务会启动自己的工作线程。<br>

`SystemServer.java`
```
// 开启服务
try {
    traceBeginAndSlog("StartServices");
    // 开启系统所需的关键服务
    startBootstrapServices();
    // 开启与初始化系统进程没有关联并且很重要的服务
    startCoreServices();
    //开启其他服务
    startOtherServices();
    SystemServerInitThreadPool.shutdown();
} catch (Throwable ex) {
    Slog.e("System", "******************************************");
    Slog.e("System", "************ Failure starting system services", ex);
    throw ex;
} finally {
     traceEnd();
}
  ```

**那么启动服务到底指的的什么呢。主要是做服务的初始化工作**，比如说准备好服务的Binder实体对象，当有Client请求的时候，就会在Binder线程池中把具体的请求分发给具体的Binder实体对象进行处理，然后再把结果返给Client。<br>

还有一些系统的服务不是在SystemServer中启动的，他们开一个单独的进程，是通过native实现的，有自己的main函数入口，需要自己管理Binder机制和管理Binder通信，这样会复杂一些，但是还是和其他服务是类似的，还是会有Binder实体对象，还是会有Binder线程，Binder线程中会等待Client的请求。到此系统服务的启动大体上就介绍完了。<br>

**应用服务**的启动方式如下代码所示
```
private ComponentName startServiceCommon(Intent service, boolean requireForeground,
            UserHandle user) {
	try {
    	...
		// 跨进程调用AMS中的startService
        ComponentName cn = ActivityManager.getService().startService(
            mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
            getContentResolver()),requireForeground,getOpPackageName(), user.getIdentifier());
  		...
        return cn;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
可以看到上面的代码是通过Binder方式调用AMS中的startService方法。<br>
那么AMS中的startService又做了哪些事情呢？我们都知道AMS不是真正干事的，它只是负责管理和调度，这中间会经历一系列的调用，最后还是通过Binder IPC调用ActivityThread中的handleCreateService方法。又回到了我们应用进程，那么下面我们看看这个方法做了什么事情。

`ActivityThread#handleCreateService`
```
@UnsupportedAppUsage
private void handleCreateService(CreateServiceData data) {
    // 这里会简化一下代码，只留下比较关键的代码
	// 所以这不是完整的源码
    
    // 获取一个LoadedAPK对象
    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    
    try {
		// 获取ClassLoader
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
		// 创建Service实例
        service = packageInfo.getAppFactory()
                .instantiateService(cl, data.info.name, data.intent);
    } catch (Exception e) {
       ...
    }

    try {
        // 获取ContextImpl对象
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);

        // 获取Application对象
        Application app = packageInfo.makeApplication(false, mInstrumentation);
        // 调用attach()方法，加入进程、className等信息，初始化Service
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
        // 调用onCreate()方法
        service.onCreate();
        mServices.put(data.token, service);
        ...
    }
```
到这里Service实例就被创建出来了并调用了其onCreate方法。接下来还会调用它的onStartCommand方法，这里就很简单了，就不展开讲了。这样一个应用服务就启动起来了。
### 注册方式的区别
**系统服务**在启动的时候就会自动注册到ServiceManager上了，源码还是比较多的，这里就不看了，大多都是native层的。<br>

**应用服务**的注册是应用通过bindService方法向AMS请求对应Service的Binder对象，如果AMS有就直接返回，没有的话就向Service请求Binder对象，Service收到请求后就在ASM上注册并返回给应用。**应用服务是被动的注册**
### 使用方式的区别
**系统服务**我们平常的开发中接触还是比较多的。下面我们来举个例子。
```
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
	// 下载链接
    val downloadUri = Uri.parse("https://xxx")
	// 获取下载服务
    val downloadManager: DownloadManager = this.getSystemService(Context.DOWNLOAD_SERVICE) as DownloadManager
	// 把这个下载任务插入到ContentResolver中，等待执行
    downloadManager.enqueue(DownloadManager.Request(downloadUri))
}
```
在使用系统服务这方面还是非常简单的。

下面我们在来看看**应用服务**的使用是怎样的
```
bindService(intent,object: ServiceConnection {
	// 服务已连接
	override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
		// 通过Service的IBinder对象获取Service对象
		// 然后就可以愉快的使用了
    	mService = IMyService.Stub.asInterface(service)
    }

    override fun onServiceDisconnected(name: ComponentName?) {
		// 断开连接把Service对象设置null
        mService = null
    }
},Context.BIND_IMPORTANT)
```
## 简单总结

* 启动方式上的区别：<br>
**系统服务**是随着android系统的启动而自动启动的<br>
**应用服务**是需要我们手动调用启动方法启动的
* 注册方式上的区别：<br>
**系统服务**是自动注册到ServiceManager上的<br>
**应用服务**是手动注册到AMS上的
* 使用方式上的区别：<br>
**系统服务**通过context.getSystemService()方法回去服务来使用的<br>
**应用服务**通过调用startService()或bindService()方法来使用的

## 参考文献

[ActivityManagerService第二讲之Service启动流程](https://blog.csdn.net/zplxl99/article/details/104314587)