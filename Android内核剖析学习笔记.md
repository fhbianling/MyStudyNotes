# Java基础 #
## ClassLoader ##
一般情况下通过import即可引入需要的类，在涉及到AndroidFramework层的情况下，通过ClassLoader扩充Framework层达到对原生该层的最小化修改。
Java的Runtime环境在初始化时，会自动创建一个ClassLoader的内部对象用于加载Runtime所需类。
每个ClassLader都必须有一个父ClassLoader，对于某个类的加载，只有父ClassLoder找不到该类时，子ClassLoader才会去	继承装载该类。
对于Android的主要影响体现在：Android的Class文件在应用程序中被打包为dex文件，而dex文件由AndroidSdk提供的DexClassLoader进行加载。

# FrameWork层 #
## 一个包含Activity的客户端至少有几个线程？ ##
在DDMS中看到的至少会有3个线程，包括1个main线程，Binder1和Binder2线程

APK的运行启动过程：



- ActivityThread执行Main方法
	- 创建主线程Looper,MessageQueue,ActivityThread对象
	- 创建一个H（Handler），一个ApplicationThread(Binder)对象	
		> ApplicationThread负责AmS的IPC调用，被调用后通过H将消息发送到消息队列，UI线程会从消息队列中取出消息并作出相应操作（Start,Stop,Pause等） 
	- UI线程调用Looper.loop()方法开始消息循环和处理
-  AmS发出消息<br/>->start某个Activity<br/>
	->创建Activity对象<br/>->创建PhoneWindow<br/>->DecorView<br/>->作为内容的View或ViewGroup<br/>显示内容 ：<br/>调用WindowManager类（WindowManager实际上用于和WmS交互，WmS不能直接和客户端交互）<br/>->创建一个ViewRoot对象（实际上创建了ViewRoot类和W类）<br/>ViewRoot对象创建完成后，由WindowManager调用WmS提供的远程接口添加窗口并显示在屏幕上
- WmS的内部类KeyQ不断把用户消息（UI操作，如触屏，滑动等）存储在QueueEvent队列中<br/>InputDispatcherThread线程逐个取出消息，调用WmS中相应函数处理消息<br/>若消息属于客户端某窗口则调用相应窗口W接口
- W类是一个Binder<br/>接到消息后，将其传递给ViewRoot<br/>ViewRoot将消息传递给ActivityThread<br/>ActivityThread解析消息并处理<br/>DecorVeiw首先处理消息，若不处理则传递给内部子View或ViewGroup<br/>若一直未被处理，则传递给给PhoneWindow并最终传递给Activity，因此Activity也有onTouchEvent方法

因此在一个包含Activity的客户端被启动后，至少会有3个线程，包括ActivityThread,ViewRoot.W和ApplicationThread对象的内部线程（Binder类会有一个隐藏线程用于发送和返回消息以实现IPC调用）。

> 为什么要在自定义线程中调用Looper.prepare()方法才能使用Handler？
><br/> 详见Handler相关源码，在ActivityThread中已经通过Looper.prepareMainLooper()为UI线程创建了Looper对象，而自定义线程中在sThreadLocal中是没有这个线程的Looper以及其MessageQueue的，因此需要调用.prepare()方法

## Activity间传递消息 ##
Activity间传递消息最大的问题是Activity引用的获得，由于Activity的实例化由FrameWork层完成，因此常规的通过引用传递消息的方式不一定适用。
<br/>可通过Application传递，因为对于同一个程序，其Application对象是唯一的。
<br/>FrameWork层也提供了Intent用于标准的Activity间传递消息的方法

## 相关概念在语义学上的某些差别和Android中的实际意义 ##
### 窗口 ###
泛指屏幕上的某个独立的界面，如对话框，Menu菜单，Activity界面。
<br/>从WmS角度来说，窗口是接收用户消息的最小单元，给WmS添加一个窗口是通过调用WindowManager类的addView方法。
<br/>既从WmS角度来看，所谓添加一个窗口就是添加一个View对象。WmS接收到用户消息后,首先判断其属于哪个窗口，然后通过IPC调用机制传递给ViewRoot.W
### Window类 ###
android.view包中的一个抽象类，抽象了客户端的基本操作，定义了一组回调接口，Activity实现了这些接口方法
### ViewRoot类 ###
android.view包中的一个类，其内部的W类在客户端创建一个窗口时作为其代理与WmS交互，每个窗口对应一个ViewRoot
### W类 ###
ViewRoot的内部类，继承至Binder,用于向WmS提供IPC接口，从而让WmS控制窗口客户端行为

实际上WmS所管理的窗口和Window类没有任何关系

# Context #
Activity的本质就是一个Context，extends是类的本质，而implement只是对类功能的扩充。
<br/>api24<br/>Activity **extends** ContextThemeWrapper 
<br/>**implements** Factory2, Callback, android.view.KeyEvent.Callback, OnCreateContextMenuListener, ComponentCallbacks2 
> Activity中OnTouchEvent方法是自身的方法，但这个方法会在onDispatchTouchEvent方法中被调用，而Activity的onDispatchTouchEvent方法来自于CallBack接口（第一个CallBack），CallBack接口是android.view.Window类中一个接口

<br/>ContextThemeWrapper **extends** ContextWrapper
<br/>ContextWrapper **extends** Context

一个应用内有多少个Activity或Service就有多少个Context对象。
## Context相关继承关系 ##
除上文Activity外，ContextWrapper的有一个子类为Service,而Context的真正实现类是ContextImpl.
ContextWrapper中包含了一个真正的Context实现类的引用，也即是ContextImpl的对象，而应用程序中所调用的各种来自Context的方法，其实现均来自与ContextImpl类。
> ContextWrapper仅有一个构造方法<br/>`public ContextWrapper(Context base) {
        mBase = base;
    }`
<br/>【其中的base在FrameWork层创建Activity时，通过Activity的attach方法传入（**这段存疑，网上的说法**）
[http://blog.csdn.net/z1074971432/article/details/12561369](http://blog.csdn.net/z1074971432/article/details/12561369)】

## Context的创建 ##