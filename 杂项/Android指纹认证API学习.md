# Android指纹识别 #
> 参考博客：<br>[https://www.cnblogs.com/popfisher/p/6063835.html](https://www.cnblogs.com/popfisher/p/6063835.html "Android指纹识别深入浅出分析到实战（6.0以下系统适配方案）")
> <br>[http://blog.csdn.net/zhangdong305/article/details/52438050](http://blog.csdn.net/zhangdong305/article/details/52438050)
> <br>[https://www.jianshu.com/p/f0d48b3d8478](https://www.jianshu.com/p/f0d48b3d8478)
> 
> 参考Github:[https://github.com/PopFisher/FingerprintRecognition](https://github.com/PopFisher/FingerprintRecognition)

## 概述 ##

相关类：

- android.hardware.fingerprint.**FingerprintManager**
- android.support.v4.hardware.fingerprint.**FingerprintManagerCompat**
- android.support.v4.hardware.fingerprint.**FingerprintManagerCompatApi23**

类结构：

- **FingerprintManager**。在api 23(即Android M,6.0)及以上提供的一项系统服务System Service，可以通过Context.getSystemService拿到这个manager在系统注册的实例。
- **FingerprintManagerCompatApi23**。在v4包中提供的上述系统服务的包装类。当调用其与该系统服务相对应的方法时，事实上最终调用到的还是FingerprintManager的相关方法。**但是这个类的相关方法基本是包权限的，不能由开发者调用到**。这个类封装系统服务后，主要是提供了一些对硬件设备不存在服务下获取SystemService的判断和方法实现。（考虑这种情况，在api23以上系统的手机，厂商并没有提供指纹相关的硬件，当使用Compat时，由于Compat调用到了CompatApi23，我们开发者就不需要再去判断这种情况）
- **FingerprintManagerCompat**。该类即是提供了兼容性处理的系统服务包装类，上述CompatApi23类的包权限方法既由该类来进行调用，更准确的说，当api大于等于23时，该Compat会通过CompatApi23去处理相关的方法调用，小于api23时由该类的内部类LegacyFingerprintManagerCompatImpl来处理相关调用，至于这个内部类的处理逻辑就是什么都不处理。也就是说，如果小于23，通过该Compat进行的调用，无论手机是否有指纹相关的硬件和厂商自己提供的SDK，都会视为没有指纹相关的系统服务。

综上，一般情况下，我们只需要FingerprintManagerCompat这个类就可以满足需求。

> 当compileSdkVersion小于23时在IDE中是访问不到FingerprintManager这个类的，另外appcompat或v4包的对应版本小于23也是访问不到下面两个类的。

## 使用 ##
需要相关权限：**"android.permission.USE_FINGERPRINT"**也即**Manifest.permission.USE_FINGERPRINT**，可以在manifest中直接申请。

这里针对FingerprintManagerCompat的相关方法进行分析

1.

    FingerprintManagerCompat.from(Context context)
该方法用于获取FingerprintMangerCompat对象。

2.

    public boolean isHardwareDetected()；
判断硬件设备是否支持指纹识别。

3.

	public boolean hasEnrolledFingerprints()；
当前设备是否包含已经登记的指纹。
> 两个需要注意的点，1，如果要进行指纹识别，要求设备中至少有一个已经登记的指纹。2，如果设备有已经登记的指纹，那么一般来说设备也一定有锁屏密码（否则假设设备使用了指纹锁屏功能，未设锁屏密码，在手指受伤或其他情况导致登记指纹失效的情况下，用户就不能解锁手机了）。

4.

	public void authenticate(@Nullable CryptoObject crypto,
							int flags,
            				@Nullable CancellationSignal cancel, 
							@NonNull AuthenticationCallback callback,
            				@Nullable Handler handler)；

参数说明:

- **CryptoObject crypto** <br>一个加密对象，这个对象在认证成功后会在AuthenticationCallback的result中回传回来。可以传null
- **int flags** <br>可选flags，不知道什么用处，源码注释说"should be 0"。则传0就可以。
- **CancellationSignal cancel**<br>这个对象可以用于取消指纹识别。
- **AuthenticationCallback callback**<br>识别结果回调。
- **Handler handler**<br>该handler的用处在于线程同步，主要是为它内部创建的handler提供Looper,除此之外不会用这个handler做更多的事。AuthenticationCallback的各种回调发生在该handler对应的线程，如无需求传null即可，内部会默认使用主线程的mainLooper来创建它需要的handler来投递回调。

需要注意的若干点：

1. CancellationSignal cancel该引用只能使用一次，即如果一次认证已经被该引用取消了，这时再次调用authenticate()方法，入参CancellationSignal依然是上次的该对象引用，这次方法调用会直接走到回调的error中，不会再次激活传感器。<br>假设传感器开启，但此时用户没有录入指纹，那么传感器将会一直开启知道超时才会关闭，这个超时时限是由厂商决定的，因此在某些情况下应该由我们手动cancel来关闭传感器，比如离开需要指纹认证的页面时。
	
2. 当authenticate()该方法调用后，指纹传感器即被激活，AuthenticationCallback，有如下四个回调:
	-  void onAuthenticationError(int errMsgId, CharSequence errString)
	-  void onAuthenticationHelp(int helpMsgId, CharSequence helpString) 
	-  void onAuthenticationSucceeded(AuthenticationResult result)
	-  void onAuthenticationFailed()  
		- 一旦error或succeeded这两个回调被触发，指纹传感器就会停止。
		- failed代表的是认证失败，即传感器正常工作，但指纹不符合。
		- error情况下，传感器没有正常工作，比如多次failed的情况下，会直接进入error回调，并停止传感器，需要等待一段时间才能再次重试。设备没有录入指纹或未开启指纹的情况下，用手指触摸传感器也会进入该回调。
		- help情况下，这种情况下传感器工作异常但不会停止，比如传感器表面有污垢，影响到识别过程，就会进入到help状态，应该将helpString展示给用户，让其根据提示调整设备状态。

3. CryptoObject 这个参数主要是用来做安全性处理的，前面提到这个对象在认证成功时会在AuthenticationResult中被传递回来，这时就可以效验数据是否在认证过程中篡改过。**//todo**
