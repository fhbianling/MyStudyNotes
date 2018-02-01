# InputManagerService与输入事件 #
## 事件的分类 ##
- KeyEvent:按键事件
- TouchEvent:触摸事件
- MouseEvent:鼠标事件
- TrackBallEvent:轨迹球事件（淘汰）

### 结构 ###

|InputEvent（抽象接口）
|
|KeyEvent（按键）|MotionEvent(所有能产生Movement的事件)

## 派发 ##
输入事件的派发由WindowManagerService开始，至于事件的采集来自于Linux内核层

## InputManagerService(IMS) ##
Java层的InputManagerService只是对NativeInputManager的一层包装。该项服务在SystemServer的run()方法中创建实例。


//Native层略

## ViewRootImpl对事件的派发 ##
WMS->ViewRootImpl(InputEvent)->enqueueInputEvent()

在enqueueInputEvent()中，如果为紧急事件->doProcessInputEvents()(立即处理),否则scheduleProcessInputEvents()(将事件入队，按顺序处理)。

