# Android复习纲要 #
> 结构梳理参考：[https://blog.csdn.net/ClAndEllen/article/details/79257663](https://blog.csdn.net/ClAndEllen/article/details/79257663 "主要参考")

## 四大组件 ##

### Activity ###
[https://blog.csdn.net/clandellen/article/details/79257489](https://blog.csdn.net/clandellen/article/details/79257489 "参考")

![activity生命周期示意图](http://i2.tiimg.com/697392/7e09d93675aef06f.png)

对于一个Activity从对象创建到被GC回收，在它的整个生命周期中:

- onCreate/onStart *分别只执行一次*，对应Activity初次创建并显示到前台之前
- onResume 当onResume执行后，activity位于前台，且当前窗口焦点位于这个activity *可能执行多次*
- onPause/onStop 当activity从前台退往后台，或activity在前台但失去焦点时，则会触发onPause，并且如果在onPause执行后,activity位于后台的情况下，那么onStop也会接着执行 *可能执行多次*
- onDestroy 当activity即将走向生命周期终点时，onDestroy会被调用 *只会被执行一次*，但注意onDestroy被执行后并不代表activity对象一定会被回收，比如发生内存泄漏的情况下，任何无法从GC可达性算法中被判定为不可达的activity引用，此时都会使得对应activity即使在onDestroy被调用后，依然无法被回收。
- onRestart 当activity此时位于后台且为onStop，在它被再次调往前台时，即会执行onRestart，紧接着就是onResume *可能执行多次*
- onSaveInstanceState 当前activity处于异常状况，且即将被销毁的情况下，**可能而非必然**被调用，例如内存不足，横竖屏切换等情况下。且onSaveInstanceState确定会调用时，**一定** 会在onStop之前调用，但**无法确保一定**在onPause之前或之后被调用。

> 其他内容：进程优先级，启动模式等。

### Broadcast ###
[https://blog.csdn.net/clandellen/article/details/79279416](https://blog.csdn.net/clandellen/article/details/79279416)
android的一种用于在同一App不同进程不同组件间，或不同app间进行通信的机制。
#### 广播发送 ####
1. 无序广播 `context.sendBroadcast(intent)`
	<br>不可被拦截

2. 有序广播 `context.sendOrderBroadcast(intent)`
	<br>可以被拦截，可以被修改，可以添加新数据，下一个接受者接受到的数据，是上一个接受者传递过来的，所以有可能是被修改过的。
3. 本地广播 `LocalBroadcastManager.getInstance(context).sendBroadcast(intent)`
	<br>仅会在app内部传播
#### 广播接收 ####

1. 静态注册
	<br>实现 `xxxReceiver extends BroadcastReceiver`,重写 `onReceive(context,intent)`,并在AndroidManifest.xml中进行注册。
2. 动态注册
	<br>Receiver的创建方式同上，但注册方式通过调用`context.registerReceiver(多种重载)`来进行注册。对于本地广播通过`LocalBroadcastManager.getInstance(contenxt).registerReceiver(receiver,intentFilter)`来注册。
	<br>注意动态注册的广播接收器，需要在不使用的时候解注册unregister。
#### 实现 ####
1. 非本地广播(无序和有序广播)<br>通过Binder机制实现，并以AMS为桥梁。换言之，广播通过Binder机制发送到AMS，再由AMS查找到合适的接受者（即接收器实际上也是注册在AMS中），再将广播转发给合适的接受者。
2. 本地广播<br>内部实际上通过Handler实现。广播的真实表现形式就是Handler的message。`LocalBroadcastManager.getInstance(context)`这里的入参context实际上就是为了拿到在mainLooper上的handler。LocalBroadcastManager的单例中持有了所有通过该单例注册的接受者。通过该单例调用`sendBroadcast(intent)`时，实际上就是分析intent,找到合适的接受者，然后通过handler发送信号，向对应的接受者调用它的onReceive方法。

### ContentProvider ###
[https://blog.csdn.net/ClAndEllen/article/details/82765220](https://blog.csdn.net/ClAndEllen/article/details/82765220)
<br>跨程序数据共享的标准方式。
### Service ###
[https://blog.csdn.net/weixin_39460667/article/details/82770164](https://blog.csdn.net/weixin_39460667/article/details/82770164)
<br>可以在后台长时间运行，且没有用户界面的组件。运行于UI线程。
> 线程是进程的实际运作单位。一个进程可以并发多个线程。
> <br>这里的UI线程实际指的是App运行的主进程中的UI线程。
> <br>如果对service在注册时声明android:process=":xxx"属性，则该service会运行在包名.xxx对应的进程下的主线程中。
> <br>如果打印出来看的话，该主线程的名字一样叫main，但其hashCode和UI线程的hashCode不同(UI线程打印出来name也是main)。
#### 使用 ####
1. 在AndroidManifest.xml中声明。如果
2. 