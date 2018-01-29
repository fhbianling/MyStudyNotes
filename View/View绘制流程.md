# View的组织形式 #
## 相关类 ##
- **View**,View本身不是任何类的子类，其继承了三个接口，其中两个为Android IO系统输入事件的相关接口，另一个为Drawable.Callback，主要用于Drawable改变时通知UI样式改变(比如StateDrawable，选择器这种)。

- **ViewParent**，该接口分别由ViewRootImpl和ViewGroup继承，是对View父节点的一种抽象。

- **ViewGroup**，继承自View，同时实现了ViewParent接口

- **ViewRootImpl**,该类是真正用于和UI绘制底层实现的surface进行交互的类，它承担着和WMS进行交互的职责。当WindowManager进行addView操作时，就会生成一个与被add的view相绑定的ViewRootImpl对象。
	> 对于Activity和Dialog,Toast等在这里的实现最终都可以指向这个方法

## View树的结构 ##
ViewRootImpl始终是这颗抽象出来的树结构的根。

它的子节点只有一个，举例来说，比如Activity的DecorView，从这个子节点开始，下面的结构就是典型的树结构。

任何叶子结点都可能是一个ViewGroup(实质上也可当作View),或是一个View。其他子结点则必然是一个ViewGroup。

## UI刷新的触发点 ##
涉及到两个重要方法，View#requestLayout(),View#invalidate(),**这两个方法在ViewParent中也被声明了。**

当调用View本身的setAlpha(),setTranslationX()等等可能导致UI需要刷新的方法时，最终都会调用到这两个方法，当然自定义View控件时，也需要根据业务逻辑调用这两个方法刷新UI。

- requestLayout，请求布局。
- invalidate,无效，意指当前UI失效(比如控件中内容的改变)，需要重新绘制。

ViewGroup并没有重写这两个方法。例如对ViewGroup addView时也会调用这个方法。此时其方法体对应View中的实现。

对于View的requestLayout和invalidate,核心逻辑其实都是通过getParent拿到这个View的ViewParent，然后调用它的这两个对应方法。
换句话说对于任何需要刷新的UI，当调用该方法时都会自下而上的一直向树的顶层调用。也就是说最终会走入ViewRootImpl的该方法中。

## UI刷新的具体实现 ##
这两个方法在ViewRootImpl中的实现，其核心逻辑都是经过一系列判断后，在满足条件的情况下走入到scheduleTraversals()中，并最终走入到performTraversal()该方法。

traversal的意思是遍历。

performTraversal()中最终会走入三个核心方法，

- performLayout()。
- performMeasure()。
- performDraw().

前两个方法的实现实际上就是对View树根中ViewRootImpl的子结点调用其layout和measure这两个方法。

这两个方法来自于View，并最终会分别调用到View的onLayout和onMeasure，也就是自定义UI时通常会重写的两个方法。
对于ViewGroup，它需要重写onLayout，onMeasure，根据子控件的宽高和margin(虽然这个属性是对view声明的，但实际上是在ViewGroup内部才会生效)等属性，调整子控件的布局位置以及宽高最大测量值等。对于View，它需要在onMeasure中确定自身所需要的宽高。

总之在这两个方法之后，View树中各个UI元素都确定好了自身绘制的位置和大小等信息。

在进入performDraw()后，实际上调用的是对ViewRootImpl的子结点view调用draw(Canvas)方法.

> draw(Canvas)方法,在view的实现中，会分别调用到dispatchDraw(Canvas)和onDraw(Canvas)，View仅仅是空实现dispatchDraw(Canvas)，这个方法的意义在于对其子View分发draw(Canvas)的操作。
> 这个方法在ViewGroup中被实现了。 在之前测量时获得的各种信息，此时就会在ViewGroup的dispatchDraw中用于裁剪canvas并分配给子View。

> 在draw(Canvas)方法的View实现中，如下源码注释很好的解释了这个方法的工作流程
> 
>  Draw traversal performs several drawing steps which must be executed
>  in the appropriate order:
>  
>  *  1. Draw the background (绘制背景)
>  *  2. If necessary, save the canvas' layers to prepare for fading (如果需要，保存layer，以备后续fading效果)
>  *  3. Draw view's content (绘制View内容区)
>  *  4. Draw children (绘制View的子View)
>  *  5. If necessary, draw the fading edges and restore layers (如果必要，则绘制fading效果，并restore layer)
>  *  6. Draw decorations (scrollbars for instance) (绘制装饰)
>  */

onDraw方法的入参canvas，就是在对view调用draw()之前在由ViewRootImpl中持有的surface创建出来的(`mSurface.lockCanvas(dirty);`)。之后的过程就是自顶向下的，由View树中的各个UI元素对Canvas作画，当这一过程结束后，由`surface.unlockCanvasAndPost(canvas)`这一调用真正的将此时的新的UI信息刷新到界面之上。
# Surface的绘制操作 #

//todo