# Android复习纲要 #
> 结构梳理参考：[https://blog.csdn.net/ClAndEllen/article/details/79257663](https://blog.csdn.net/ClAndEllen/article/details/79257663 "主要参考")

## Android基础 ##

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