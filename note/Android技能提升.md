## Binder ##
### **1、为什么Android要采用Binder作为IPC机制**？ ###
- **从性能的角度 数据拷贝次数：**Binder数据拷贝只需要一次，而管道、消息队列、Socket都需要2次，但共享内存方式一次内存拷贝都不需要；从性能角度看，Binder性能仅次于共享内存。
- **从稳定性的角度：**Binder是基于C/S架构的，简单解释下C/S架构，是指客户端(Client)和服务端(Server)组成的架构，Client端有什么需求，直接发送给Server端去完成，架构清晰明朗，Server端与Client端相对独立，稳定性较好；而共享内存实现方式复杂，没有客户与服务端之别， 需要充分考虑到访问临界资源的并发同步问题，否则可能会出现死锁等问题；从这稳定性角度看，Binder架构优越于共享内存。
- **从安全的角度：**传统Linux IPC的接收方无法获得对方进程可靠的UID/PID，从而无法鉴别对方身份；而Android作为一个开放的开源体系，拥有非常多的开发平台，App来源甚广，因此手机的安全显得额外重要；Android为每个安装好的应用程序分配了自己的UID，故进程的UID是鉴别进程身份的重要标志，前面提到C/S架构，Android系统中对外只暴露Client端，Client端将任务发送给Server端，Server端会根据权限控制策略，判断UID/PID是否满足访问权限，目前权限控制很多时候是通过弹出权限询问对话框，让用户选择是否运行。
### **2、Binder是什么？作用是什么？** ###
Binder通信采用C/S架构，从组件视角来说，包含Client、Server、ServiceManager以及binder驱动，其中ServiceManager用于管理系统中的各种服务。<br>

Binder使用过程中3个重要步骤中相应的C端和S端：
- 1、**注册服务：**首先AMS注册到ServiceManager。改过程：AMS所在的进程（system server）是客户端，ServiceManager是服务端。
- 2、**获取服务：**Client进程使用AMS前，须先向ServiceManager中获取AMS的代理类AMP。该过程：AMP所在进程(app process)是客户端，ServiceManager是服务端。
- 3、**使用服务：**app进程根据得到的代理类AMP,便可以直接与AMS所在进程交互。该过程：AMP所在进程(app process)是客户端，AMS所在进程(system_server)是服务端。
## Handler ##
- 每个 Handler 都会和一个 Looper 实例关联在一起，可以在初始化 Handler 时通过构造函数主动传入实例，否则就会默认使用和当前线程关联的 Looper 对象
- 每个 Looper 都会和一个 MessageQueue 实例关联在一起，每个线程都需要通过调用 Looper.prepare()方法来初始化本线程独有的 Looper 实例，并通过调用Looper.loop()方法来使得本线程循环向 MessageQueue 取出消息并执行。Android 系统默认会为每个应用初始化和主线程关联的 Looper 对象，并且默认就开启了 loop 循环来处理主线程消息
- MessageQueue 按照链接结构来保存 Message，执行时间早（即时间戳小）的 Message 会排在链表的头部，Looper 会循环从链表中取出 Message 并回调给  Handler，取值的过程可能会包含阻塞操作
- Message、Handler、Looper、MessageQueue 这四者就构成了一个生产者和消费者模式。Message 相当于产品，MessageQueue 相当于传输管道，Handler 相当于生产者，Looper 相当于消费者
- Handler 对于 Looper、Handler 对于 MessageQueue、Looper 对于 MessageQueue、Looper 对于 Thread ，这几个之间都是一一对应的关系，在关联后无法更改，但 Looper 对于 Handler、MessageQueue 对于 Handler 可以是一对多的关系
- Handler 能用于更新 UI 包含了一个隐性的前提条件：Handler 与主线程 Looper 关联在了一起。在主线程中初始化的 Handler 会默认与主线程 Looper 关联在一起，所以其 handleMessage(Message msg) 方法就会由主线程来调用。在子线程初始化的 Handler 如果也想执行 UI 更新操作的话，则需要主动获取 mainLooper 来初始化 Handler
- 对于我们自己在子线程中创建的 Looper，当不再需要的时候我们应该主动退出循环，否则子线程将一直无法得到释放。对于主线程 Loop 我们则不应该去主动退出，否则将导致应用崩溃
- 我们可以通过向 MessageQueue 添加 IdleHandler 的方式，来实现在 Loop 线程处于空闲状态的时候执行一些优先级不高的任务。例如，假设我们有个需求是希望当主线程完成界面绘制等事件后再执行一些 UI 操作，那么就可以通过 IdleHandler 来实现，这可以避免拖慢用户看到首屏页面的速度。
### **1、Handler、Looper、MessageQueue、Thread的对应关系** ###
- Looper 中的 MessageQueue 和 Thread 两个字段都属于常量，且 Looper 实例是存在 ThreadLocal 中，这说明了 Looper 和 MessageQueue 之间是一对一应的关系，且一个 Thread 在其整个生命周期内都只会关联到同一个 Looper 对象和同一个 MessageQueue 对象
- Handler 中的 Looper 和 MessageQueue 两个字段也都属于常量，说明 Handler 对于 Looper 和 MessageQueue 都是一对一的关系。
- Looper 和 MessageQueue 对于 Handler 却可以是一对多的关系，例如，多个子线程内声明的 Handler 都可以关联到 mainLooper。
### **2、Handler的同步机制** ###
- MessageQueue 在保存 Message 的时候，enqueueMessage方法内部已经加上了同步锁，从而避免了多个线程同时发送消息导致竞态问题。
- next()方法内部也加上了同步锁，所以也保障了 Looper 分发 Message 的有序性。
- 最重要的一点是，Looper 总是由一个特定的线程来执行遍历，所以在消费 Message 的时候也不存在竞态。
## HandlerThread ##
HandlerThread 是 Android SDK 中和 Handler 在同个包下的一个类，从其名字就可以看出来它是一个线程，而且使用到了 Handler其用法类似于以下代码。

通过 HandlerThread 内部的 Looper 对象来初始化 Handler，同时在 Handler 中声明需要执行的耗时任务，主线程通过向 Handler 发送消息来触发 HandlerThread 去执行耗时任务。
## IntentService ##
IntentService 是系统提供的 Service 子类，用于在后台串行执行耗时任务，在处理完所有任务后会自动停止，不必来手动调用 stopSelf() 方法。而且由于IntentService 是四大组件之一，拥有较高的优先级，不易被系统杀死，因此适合用于执行一些高优先级的异步任务。
> Google 官方以前也推荐开发者使用 IntentService，但是在 Android 11 中已经被标记为废弃状态了，但这也不妨碍我们来了解下其实现原理

IntentService 内部依靠 HandlerThread 来实现，其 onCreate()方法会创建一个 HandlerThread，拿到 Looper 对象来初始化 ServiceHandler。ServiceHandler 会将其接受到的每个 Message 都转交由抽象方法 onHandleIntent来处理，子类就通过实现该方法来声明耗时任务