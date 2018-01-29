# View事件传递流程 #
> View树的任意子结点，要么是View要么是ViewGroup，因此IO消息的传递，所调用的接口应该是由View提供的，因为ViewGroup是View的子类。
> <br>以TouchEvent为例，当接受到事件后，首先做出处理的是位于View树根的ViewRootImpl，事件的进一步分发将由它进行处理。
> <br>更具体的说，该ViewRootImpl会通过调用mView(这里就是DecorView)的dispatchPointerEvent()方法来处理事件。
## View中的TouchEvent ##
对于View的dispatchPointerEvent()方法，其源码如下

     	if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }

在View的dispatchTouchEvent实现中，会首先判断是否设置了onTouchListener，如果设置交给它处理，如果这个Listener不存在或者它没有处理则交给onTouchEvent()(View的方法)处理。
当然在这个方法中还包括对滚动停止，可点击性检查等等其他处理。
## ViewGroup中的TouchEvent ##
ViewGroup重写了View的dispatchTouchEvent()（*注意不是dispatchPointerEvent()*）。

在该方法的核心逻辑中：

- 在disallowIntercept为false时，会去获取onInterceptTouchEvent()这个方法的返回值，也即是否需要拦截事件。如果不允许拦截，则直接视该方法返回值为false,即不拦截。
- 上述代码的逻辑是发生在ACTION_DOWN,或者上一次的Touch事件被某个View处理了(mFirstTouchTarget!=null)，这两种情况下。（**存疑**）
- 假设该事件没有被ViewGroup拦截的情况下，事件将进一步被分发给子类。
	> 这个分发逻辑实质上是一个for循环，这个循环按照索引递减的方式去遍历该ViewGroup的子View,同时，不可见的或者点击事件坐标位于View范围以外的View将首先被过滤。
	> 在此条件下，越接近用户的或者说Z轴越高的越优先满足它对Touch事件的消耗。这样也是符合人对点击事件的抽象的。
- 具体的分发上依然是通过调用子View的dispatchTouchEvent()，这里又分为子View是View和ViewGroup的两种情况。如果是ViewGroup，则这个子View还要再去对它的子View重复上述过程。