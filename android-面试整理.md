## Android

#### Context

1. 说说你对Context的了解。

2. Activity、Context、Application三者有什么不同。+2

3. Intent的作用。

4. 创建dialog所需的上下文为什么必须是Activity

   

#### Activity

##### 说说bundle机制。Bunder传输对象为什么需要序列化？+2

```java
定义：一个支持序列化（内部实现了Parcelable）的key-value容器类（内部使用ArrayMap存储数据）。
作用：包装数据，用于组件之间的传递 或者 Activity与Fragment之间传递。
  
为什么Bundle需要实现序列化呢？
序列化：将对象转换为可存储传输的字节序列。
因为Bundle需要支持对象的传输。而对象是无法存储或传输的，需要转换成可存储传输的状态，所有才需要序列化。
组件之间的数据传递是跨进程的，比如ActA在启动ActB时给它传递数据，是需要先给AMS进程提交申请并注册ActB的，AMS同意之后告诉app进程可以启动ActB了。这个进程间传递数据的过程，数据是需要实现序列化的。
```



##### 为什么Intent无法传大图片。+2

```java
1、讲清楚一个进程的Binder内存大小限制
#define BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)//这里的限制是1MB-4KB*2
ProcessState::ProcessState(const char *driver)
{
    if (mDriverFD >= 0) {
        // 调用mmap接口向Binder驱动中申请内核空间的内存
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        ...
    }
}
如果一个进程使用ProcessState这个类来初始化Binder服务，这个进程的Binder内核内存上限就是BINDER_VM_SIZE，也就是1MB-8KB。
对于普通APP来说都是由Zygote进程孵化出来的，而Zygote进程的初始化Binder服务的时候提前调用了ProcessState这个类，所以普通的APP进程上限就是1MB-8KB。


2、讲清楚Binder调用中同步、异步调用的内存限制
并且我们在使用Binder的时候，使用同步、异步（oneway）调用也是有内存限制的。
同步：1MB-8KB
异步：(1MB-8KB)/2
Binder调用中同步调用优先级大于异步的调用，为了充分满足同步调用的内存需要，所以异步调用的内存需要做限制。


3、讲清楚startActivty(Intent)是异步调用
Intent传输过大抛出的异常：
android.os.TransactionTooLargeException: data parcel size 821976 bytes       
  at android.os.BinderProxy.transactNative(Native Method)        
  at android.os.BinderProxy.transact(BinderProxy.java:540)       
  at android.app.IApplicationThread$Stub$Proxy.scheduleTransaction(IApplicationThread.java:2504)

由于IApplicationThread是被oneway修饰，所以其内部所有方法都是异步方法。
也就是说Intent传输的最大内存限制是(1MB-8KB)/2。


4、讲清楚Binder内存是公用的
一个进程内Binder内存缓存区是公用的。并且异步调用的内存限制也是累加的。
也就是说进程中所有的异步调用总内存要是超过(1MB-8KB)/2也会抛异常。

实验：传一个 ((1MB - 8KB)/2)-1 的东西
//警告日志
E/ActivityTaskManager: Transaction too large, intent: Intent { cmp=com.melody.test/.SecondActivity (has extras) }, extras size: 520236, icicle size: 0

//崩溃信息
Exception when starting activity com.melody.test/.SecondActivity    
  android.os.TransactionTooLargeException: data parcel size 522968 bytes        
    at android.os.BinderProxy.transactNative(Native Method)       
    at android.os.BinderProxy.transact(BinderProxy.java:540)        
    at android.app.IApplicationThread$Stub$Proxy.scheduleTransaction(IApplicationThread.java:2504)

警告的打印：extras size: 520236。 崩溃的打印：data parcel size: 522968。
大小相差：2732 约等于 2.7KB。
说明进程内有其他的异步调用占用了2.7KB空间，如果想要不崩溃，传的大小应该是 ((1MB - 8KB)/2) -1 - 2.7KB。

```



##### Intent传递数据是否有大小限制？如果数据量偏大有哪些方案。

```java
有限制。传输的大小限制是(1MB-8KB)/2

解决方案：
如果对效率没有要求并且数据不敏感的话可以选：FileProvider来实现应用间共享文件。

如果对效率有要求 或者 数据比较敏感：
匿名共享内存 - MemoryFile（使用起来好麻烦）
MemoryFile开辟内存空间，获得FileDescriptor。将FileDescriptor传给其他进程，往共享内存写入数据。
MemoryFile需要与Binder配合使用，通过Binder将FileDescriptor传给其他线程。

Linux mmap
这个用起来就很爽了。不过需要在native层写。
```



##### 说说Activity的启动模式，分别在什么场景使用？+3

```java
Standard：标准模式。每次启动都会创建一个全新的实例。
	场景：最常用的。
  
  
SingleTop：栈顶复用模式。这模式下如果Activity位于栈顶不会新建实例。onNewIntent会被调用接收新的请求信息，不再调用onCreate和onStart。
	场景：推送详情页。手机收到多条推送，点击一条查看推送详情，再点击一条查看这时会非常快得到推送详情页。
  
  
SingleTask（FLAG_ACTIVITY_NEW_TASK）：栈内复用模式。如果栈内有实例则复用，并会将该实例之上的Activity全部清除。
  特殊技能：可使用taskAffinity开辟新任务栈。
  一般情况下一个应用中的所有activity具有相同的taskAffinity，也就所有activity都会在同一个任务栈中。taskAffinity是可以影响activity（singleTask）在哪个任务栈创建的。
在activity启动的时候，AMS会检查这个activity（singleTask）是否已经有任务栈，如果没有会根据taskAffinity创建任务栈（taskAffinity默认是包名）
	同进程设置不同的taskAffinity：activity就会在不同的任务栈中，AMS会根据不同taskAffinity创建不同的任务栈。并且在哪个任务栈启动activity，activity就会呆在哪个任务栈。
  不同进程设置相同的taskAffinity：就会出现不同进程的activity出现在同一个任务栈中。
  
	场景：app首页。用户点击多次页面的相互跳转后，在点击回到主页，再次点击退出，这时他的实际需求就是要退出程序。而不是一次一次关闭刚才跳转过的页面最后才退出。
  
  
SingleInstance：系统会为它创建一个单独的任务栈，并且这个任务栈只有这个activity，不允许有别的activity存在。
  特殊技能：单独享用一个任务栈。
  该模式下的activity，如果它里边启动另一个activity，这个activity会去到该进程其他的任务栈中。如果是配置了taskAffinity，那么AMS会给这个activity加上FLAG_ACTIVITY_NEW_TASK，并且为其开辟新的任务栈。
  
	场景：电话拨号页，通过自己的应用或者其他应用打开拨打电话页面；支付页，类似支付宝、微信那种；
```



##### Activity横竖屏切换的生命周期。+3

```java
不设置android:configChanges：
切横屏：
onPause->onSaveInstanceState->onStop->onDestory->onCreate->onStart->onRestoreInstanceState->onResume
切回竖屏：（3.2之前的系统会走两次生命周期，bug！！！）
onPause->onSaveInstanceState->onStop->onDestory->onCreate->onStart->onRestoreInstanceState->onResume

android:configChanges="orientation"：生命周期与上边不设置的是一样的。
android:configChanges="orientation|keyboardHidden"：生命周期与上边不设置的是一样的。
android:configChanges="orientation|keyboardHidden|screenSize"：不销毁Activity，只调用 onConfigurationChanged 方法。
```



##### 说说一些常见的操作对Activity周期的影响。

```java
横竖屏切换：
不设置android:configChanges：
切横屏：onPause->onSaveInstanceState->onStop->onDestory->onCreate->onStart->onRestoreInstanceState->onResume
切回竖屏：（3.2之前的系统会走两次生命周期，bug！！！）onPause->onSaveInstanceState->onStop->onDestory->onCreate->onStart->onRestoreInstanceState->onResume
android:configChanges="orientation"：生命周期与上边不设置的是一样的。
android:configChanges="orientation|keyboardHidden"：生命周期与上边不设置的是一样的。
android:configChanges="orientation|keyboardHidden|screenSize"：不销毁Activity，只调用 onConfigurationChanged 方法。

按下HOME键：onPause->onStop->onRestart->onStart->onResume

按下BACK键： onPause->onStop->onDestroy->onCreate->onStart->onResume

锁屏：锁屏时只会调用onPause()，开屏后则调用onResume()。

弹出 Dialog：是通过 WindowManager.addView 显示的（没有经过 AMS），所以不会影响生命周期。（不过会触发焦点变化回调onWindowFocusChanged）
下拉状态栏：状态栏是一个Window，跟Dialog一样也是不经过AMS的。所以不会影响生命周期。（不过会触发焦点变化回调onWindowFocusChanged）

启动theme为DialogActivity、跳转透明Activity：A.onPause -> B.onCrete -> B.onStart -> B.onResume（ Activity 不会回调 onStop，因为只有在 Activity 切到后台不可见才会回调 onStop）
onPause表示当前页面失去焦点，onStop表示当前页面不可见。dialog的主题页面，就只会执行onPause，而不会执行onStop。
```



##### 如何保存Activity的状态？

```java
如何保存Activity的状态？
安卓3.0之后应用等到 onStop 返回之后才可以被杀死，也就是说在执行完 onStop 之前应用都不可能被杀死。
所有官方建议在onPause 方法中去存储那些持久性的数据，比如用户的输入等。

onSaveInstanceState 将在Activity转入“后台状态”之前被调用，能让我们存储一些activity的动态的状态值到Bundle对象中，以便在之后调用onCreate方法时用到。
但是 onSaveInstanceState 并不是生命周期的方法，所以在activity被杀死的时候，它是不能保证百分百的被执行的。

进程状态：
可视状态：可以看得见(没有被完全遮挡)，但是没有焦点，不可以触摸操作
前台状态：可以操作(有焦点)的状态
后台状态：已经看不到了，系统可以将这个进程杀死来回收内存
空进程状态：一个没有持有任何Activity和任何应用组件的进程，比如Services或者广播接受者，当内存不足的时候，它们将会被先杀死并回收。
```



##### onSaveInstanceState()与onRestoreIntanceState()作用是什么。

```java
当系统“未经你许可”时销毁了你的activity，则onSaveInstanceState会被系统调用，给你机会让你保存你的数据。
onRestoreInstanceState是让你恢复你所保存的数据。只有在activity确实是被系统回收，重新创建情况下才会被调用。

onSaveInstanceState 只适合保存瞬态数据, 比如UI控件的状态, 成员变量的值等，而不应该用来保存持久化数据。

onSaveInstanceState 在 onPause 或 onStop 之前执行，onRestoreInstanceState 会在 onStart 和onResume 之间执行。

onRestoreInstanceState 的bundle参数也会传递到onCreate方法中，你也可以选择在 onCreate 中做数据还原。
```



##### onCreate和onRestoreInstance方法中恢复数据时的区别。

```java
因为 onSaveInstanceState 不一定会被调用，所以 onCreate 里的Bundle参数可能为空，如果使用 onCreate 来恢复数据，一定要做非空判断。
而 onRestoreInstanceState 的Bundle参数一定不会是空值，因为它只有在上次activity被回收了才会调用。

而且 onRestoreInstanceState 是在 onStart 之后被调用的。有时我们需要 onCreate 中做的一些初始化完成之后再恢复数据，用 onRestoreInstanceState 会比较方便。

用 onRestoreInstanceState 恢复数据，你可以决定是否在方法里调用父类的 onRestoreInstanceState 方法，即是否调用super.onRestoreInstanceState(savedInstanceState);
而用 onCreate 恢复数据，你必须调用super.onCreate(savedInstanceState); 
```



##### onPause 和 onSaveInstanceState 都可以存储数据，用途有何不同

```java
onPause()方法适合去存储那些持久性的数据。
onSaveInstanceState方法只适合保存瞬态数据, 比如UI控件的状态, 成员变量的值等。因为只有当系统“未经你许可”时销毁了你的activity，onSaveInstanceState才会被调用。
```



##### onNewIntent 什么时候执行？

```java
启动模式为singleTask，并且触发栈内复用的时候。（就会回调onNewIntent方法，而不走onCreate方法。）
生命周期为 onNewIntent -> onRestart -> onStart -> onResume

启动模式为singleTop，并且触发栈顶复用的时候。（就会回调onNewIntent方法，而不走onCreate方法。）
生命周期为 onPause -> onNewIntent -> onStart -> onResume

注意：当在onNewIntent用到intent的时候，需要在用intent之前，使用setIntent(intent)。不然getIntent()都是得到老的intent。
```



##### onRestart 什么时候执行？

```java
1. activity由不可见变为可见
  按下home键之后，然后切换回来，会调用onRestart()；
  从本Activity跳转到另一个Activity之后，按back键返回原来Activity，会调用onRestart()；
  从本Activity切换到其他的应用，然后再从其他应用切换回来，会调用onRestart()。
2. 已经存在activity实例，再次启动activity时
  singleTask->触发栈内复用的时候，会调用onRestart()；
```



##### 为什么finish之后会10s才执行onDestory？

```java
为什么finish之后会10s才执行onDestory？
finish()执行流程：
(app)finish -> AMS#finishActivity -> ActivityStack#finishActivityLocked -> ActivityStack#startPausingLocked -> 
IApplicationThread#schedulePauseActivity -> (app)ActivityThread#handlePauseActivity（回调onPause） -> AMS#activityPaused -> ActivityStack#activityPausedLocked（移除pause兜底事件）-> ActivityStack#completePauseLocked
  
ActivityStack#startPausingLocked逻辑：
1、prev.app.thread.schedulePauseActivity（调用app进程的Binder接口schedulePauseActivity）
2、增加pause流程兜底机制，发送500ms延时事件，防止第1步的pause流程不执行。

ActivityThread#handlePauseActivity逻辑：
1、调用onPause生命周期方法
2、调用ActivityManagerNative.getDefault().activityPaused(token) 告诉AMS，我执行了onPause

ActivityStack#completePauseLocked逻辑：
1、若Activity变为不可见时，调用addToStopping函数，将中断的Activity加入到mStoppingActivities（ArrayList<ActivityRecord>）中；
2、进入启动目标Activity的流程（ActivityStackSuperVisor#resumeFocusedStackTopActivityLocked）。

也就是说，Activity执行完onPause生命周期之后，AMS并不会立即给它走onStop，而是加到一个缓存列表里边。那什么时候才会执行呢？


  
Activity onResume执行流程：
ActivityThread#handleResumeActivity（回调onResume）-> Looper.myQueue().addIdleHandler(new Idler()) -> IActivityManager#activityIdle -> AMS#activityIdle -> ActivityStackSupervisor#activityIdleInternalLocked

ActivityStackSupervisor#activityIdleInternalLocked逻辑：
遍历mStoppingActivities，然后对item们进行调用
if (r.finishing) {
    stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false);
} else {
    stack.stopActivityLocked(r);
}

ActivityStack#finishCurrentActivityLocked -> ActivityStack#destroyActivityLocked -> 回到app走onDestory
ActivityStack#stopActivityLocked -> 回到app走onStop -> 通知AMS -> 一顿操作之后 -> 回到app走onDestory

stop/destory流程兜底机制：下一个Activity#onResume回调之后，发送一个10s的兜底事件。
ActivityStackSupervisor#resumeFocusedStackTopActivityLocked -> ActivityStack#resumeTopActivityUncheckedLocked -> ActivityStack.resumeTopActivityInnerLocked -> ActivityRecord.completeResumeLocked -> ActivityStackSupervisor.scheduleIdleTimeoutLocked

void scheduleIdleTimeoutLocked(ActivityRecord next) {
  Message msg = mHandler.obtainMessage(IDLE_TIMEOUT_MSG, next);
  mHandler.sendMessageDelayed(msg, IDLE_TIMEOUT); //IDLE_TIMEOUT的值是10，延时10s
}

case IDLE_TIMEOUT_MSG: {
    activityIdleInternal((ActivityRecord) msg.obj, true);
} 



总结：
Activity的生命stop/destory是依赖IdleHandler来回调，也就是在启动下一个Activity#onResume之后的那段空闲时间，才会执行的。
在Activity#onResume之后也会发出一个10s的兜底事件，防止stop/destory一直不执行。
如果在主线程的Handler消息一直很繁忙的话，是会影响stop/destory的回调。最严重的情况会出现10s才回调。
```



##### activity.startActivity与context.startActivity的区别

```java
context.startActivity如果Intent的flag没有配置 FLAG_ACTIVITY_NEW_TASK ，就会抛异常。
context中startActivity方法真正实现是在ContextImpl.java
    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();
        if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                    + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }

activity.startActivity并没有这个限制。
Activity继承ContextThemeWrapper，ContextThemeWrapper继承ContextWrapper，ContextWrapper继承Context
而Activity重写了startActivity方法，解除了这个限制。

  
为什么activity.startActivity不加FLAG_ACTIVITY_NEW_TASK可以被允许的？
因为activity里边调用startActivity，已经是能确保有任务栈了，所以可以无需加FLAG_ACTIVITY_NEW_TASK。
而application、service、静态注册的broadcastReceiver这些地方用startActivity就必须要加，因为有可能任务栈还未建立。
所以官方才在context.startActivity增加了这个限制。


FLAG_ACTIVITY_NEW_TASK 也就是 启动模式中的 singleTask

一般情况下一个应用中的所有activity具有相同的taskAffinity，也就所有activity都会在同一个任务栈中。taskAffinity是可以影响activity（singleTask）在哪个任务栈创建的。
在activity启动的时候，AMS会检查这个activity（singleTask）是否已经有任务栈，如果没有会根据taskAffinity创建任务栈（taskAffinity默认是包名）

同进程设置不同的taskAffinity：activity就会在不同的任务栈中，AMS会根据不同taskAffinity创建不同的任务栈。并且在哪个任务栈启动activity，activity就会呆在哪个任务栈。
不同进程设置相同的taskAffinity：就会出现不同进程的activity出现在同一个任务栈中。
```



##### 跨App启动Activity需要注意什么

```java
启动Activity有显式启动和隐式启动两种方式。官方的文档中说startActivity可能会报NotFoundException，表示被start的Activity不存在。
因此，我们很容易忽略另一个可能的Exception，Permission Denial。因为startActivity是会去检验权限的。
在跨App启动Activity的时候，也许你什么Exception都不会得到，也可能会直接被Force Close掉。
  
  
如下情况，可以成功startActivity：
同一个application下；
相同UID；（Android会给每个程序分配一个UID，在相同的UID下可进行数据共享）
permission匹配；
目标Activity的属性Android:exported=”true”；
目标Activity具有相应的IntentFilter，存在Action动作或其他过滤器并且没有设置exported=false；
启动者的Pid是一个系统服务（System Server）的Pid【也就是系统服务前来调用普通App的Activity等】；
启动者的Uid是一个System Uid（Android规定android.system.uid=1000，具有该Uid的application，我们称之为获得Root权限）；
如果上述调节，满足一条，一般即可（与其他几条不发生强制设置冲突），否则，将会得到Permission Denial的Exception而导致Force Close。

  
让你的App将它里面含有的某些activity、service、provider等的数据进行共享：
1、开放允许被其他进程启动：android:exported=”true”；

2、设置自定义权限：android:permission=”xxx.xxx.xx”；（如果其他进程想使用我的组件，就必须在清单中也设置对应的android:permission=”xxx.xxx.xx”）

3、私有暴露，配置相同UID：使用sharedUserId。（两个程序可以在清单里边配置相同的sharedUserId，来让他们两个的UID相同）
```



##### 显式启动和隐式启动的区别。

```java
显式启动：明确知道了要启动的目标，直接使用目标的Class类启动。
    构造方法传入Component来启动，最常用的方式
    1、setComponent(componentName)方法
    2、setClass/setClassName方法

隐私启动：启动的目标并不清楚，只能通过使用其在清单中的配置信息来启动（前提是目标有在清单中配置对应的action）。隐式启动常用于不同应用之间的跳转（例如打开支付宝、微信的支付页面等）。
    通过在AndroidManifest中配偶action、data、category，让系统来筛选出合适的Activity
    action的匹配规则
        Intent-filter action可以设置多条
        intent中的action只要与intent-filter其中的一条匹配成功即可，且intent中action最多只有一条
        Intent-filter内必须至少包含一个action。
    category的匹配规则
        Intent-filter内必须至少包含一个category，android:name为android.intent.category.DEFAULT。
        intent-filter中，category可以有多条
        intent中，category也可以有多条
        intent中所有的category都可以在intent-filter中找到一样的（包括大小写）才算匹配成功
    data的匹配规则
        intent-filter中可以设置多个data
        intent中只能设置一个data
        intent-filter中指定了data，intent中就要指定其中的一个data
```



##### 说说Activity的启动流程。

```JAVA
总结：冷启动。
1、Launcher向AMS请求启动Activity；
Launcher会给Intent加上 FLAG_ACTIVITY_NEW_TASK（singleTask）（要启动程序的根Activity，需要创建任务栈）。

2、AMS检查应用进程是否存在，不存在则向Zygote请求孵化一个进程；
AMS与Zygote之间会建立一个Socket连接，将启动参数发送给Zygote。

3、Zygote收到启动参数后会执行孵化子进程，孵化后在子进程中反射 ActivityThread 并调用其main方法；
Zygote作为Socket的服务端，会一直轮询等待客户端连接。
孵化子进程使用的是fork方式。
子进程中会反射 ActivityThread 并调用其main方法。

4、应用进程启动并且与AMS绑定；（IActivityManager：应用进程持有的AMS binder接口；IApplicationThread：AMS持有的应用进程binder接口）
ActivityThread#main 执行，与AMS绑定并获得 IActivityManager 接口。
应用进程也创建 IApplicationThread 接口，通过 IActivityManager#attachApplication 方法传给AMS。
AMS持有应用进程 IApplicationThread 接口。

5、AMS通知应用进程创建Application以及Activity实例；
AMS通过调用 IApplicationThread#bindApplication 通知应用进程创建Application（Application#onCreate）。
紧接着AMS会创建一个启动Activity事务（ClientTransaction(LaunchActivityItem)）通过 IApplicationThread#scheduleTransaction 通知应用进程创建Activity。(如果是热启动，前面所有操作都不走，直接走这一步)
应用进程根据事务类型执行操作，启动Activity事务则是执行创建Activity后然后执行 Activity#onCreate。

6、最后走Activity的onStart、onResume生命周期，完成启动流程。
根据启动Activity事务里的LifecycleItem，执行分别执行生命周期轨迹onStart、onResume。


  
详细分析：
1、Launcher向AMS请求启动Activity
Launcher#startActivitySafely -> Activity#startActivity -> Instrumentation#execStartActivity（调用ActivityManager.getService().startActivity）-> 跑到AMS...

关键点：
Launcher#startActivitySafely 解析：
    给要启动的Intent加上 Intent.FLAG_ACTIVITY_NEW_TASK（singleTask）（要启动程序的根Activity，需要创建任务栈）;

关键点：
ActivityManager.getService 解析：获取AMS的代理对象；
    //得到activity的service引用，即IBinder类型的AMS引用
    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE); 
    //转换成IActivityManager对象
    final IActivityManager am = IActivityManager.Stub.asInterface(b);

ActivityManager.getService().startActivity 解析：
    通过Binder接口，调用AMS方法



2、AMS发送创建应用进程请求
第一步：AMS调用Process来进行进程启动
AMS#startActivity -> AMS#startActivityAsUser -> ActivityStarter#execute -> ActivityStarter#startActivityMayWait（根据Intent寻找合适的activity，如果存在多个符合条件会弹ResolverActivity让用户选择）-> ActivityStarter#startActivityUnchecked（根据启动模式做对应操作，由于是singleTask这里会创建一个任务栈） -> ActivityStackSupervisor#resumeFocusedStackTopActivityLocked -> ActivityStack#resumeTopActivityUncheckedLocked -> ActivityStackSupervisor#startSpecificActivityLocked（关键方法，普通Activity和根Activity启动流程的分岔路口）-> AMS#startProcessLocked -> AMS#startProcess（启动进程）

关键点：
ActivityStackSupervisor#startSpecificActivityLocked 解析：
        //获取将要启动的Activity的所在的进程
        ProcessRecord app = mService.getProcessRecordLocked(r.processName, r.info.applicationInfo.uid, true);
        if (app != null && app.thread != null) { //如果进程已存在
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0 || !"android".equals(r.info.packageName)) {
                    app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode, mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {}
        }
        //应用进程还未创建，则通过AMS调用startProcessLocked启动进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0, "activity", r.intent.getComponent(), false, false, true);

热启动：要启动的进程以及存在，走 realStartActivityLocked 然后直接返回了。
冷启动：进程并未启动，那么就先走启动进程流程。

AMS#startProcess 解析：
    //调用Process.start方法来为应用创建进程
    //final String entryPoint = "android.app.ActivityThread"; 创建进程后，主线程入口
    startResult = Process.start(entryPoint,
        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
        app.info.dataDir, invokeWith,
        new String[] {PROC_START_SEQ_IDENT + app.startSeq});


第二步：Process向Zygote进程发送创建应用进程请求
Process#start -> Process.ProcessStartResult#start -> Process.ProcessStartResult#startViaZygote

关键点：
Process.ProcessStartResult#startViaZygote 解析：
    // --runtime-args, --setuid=, --setgid=,
    //创建字符串列表，并将启动应用进程的启动参数保存到列表中
    argsForZygote.add("--runtime-args");
    argsForZygote.add("--setuid=" + uid);
    argsForZygote.add("--setgid=" + gid);
    argsForZygote.add("--runtime-flags=" + runtimeFlags);
    ...

    //openZygoteSocketIfNeeded：与Zygote建立Socket连接（这里连接的address是根据abi来传的），ZygoteState类型的对象。
    //zygoteSendArgsAndGetResult：由于已经与Zygote建立了Socket连接，这方法就是将进程的启动参数通过写入ZygoteState传给Zygote。
    return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), useBlastulaPool, argsForZygote);



3、Zygote进程孵化应用进程
第一步、fork出应用进程
ZygoteInit#main（创建Server端，并且等待Client连接）-> ZygoteServer#runSelectLoop（死循环不停的监听着Socket连接）-> ZygoteConnection#processOneCommand（fork进程）

关键点：
ZygoteInit#main 解析：
    public static void main(String argv[]) {
        ZygoteServer zygoteServer = new ZygoteServer();
        Runnable caller;
        try {
            ...
            //创建名为zygote的Socket
            zygoteServer.createZygoteSocket(socketName);
            ....
            //由于在init.rc中设置了start-system-server参数,因此
            //这里将启动SystemServer,可见SystemServer由Zygote创建的第一个进程
            if (startSystemServer) {
                Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
                if (r != null) {
                    r.run();
                    return;
                }
            }
           
            caller = Zygote.initBlastulaPool();
            if (caller == null) {
                //等待AMS的请求
                caller = zygoteServer.runSelectLoop(abiList);
            }
        } catch (Throwable ex) {
        } finally {
            zygoteServer.closeServerSocket();
        }

        //执行AMS请求返回的Runnable
        if (caller != null) {
            caller.run();
        }
    }



第二步、在应用进程反射ActivityThread，并调用其main方法
ZygoteConnection#handleChildProc -> ZygoteInit#zygoteInit -> RuntimeInit#applicationInit -> RuntimeInit#findStaticMain（反射ActivityThread拿到main方法。传给一个Runnable就返回了）

关键点：

ZygoteConnection#processOneCommand 解析：
    获取应用程序进程的启动参数；
    fork当前进程创建一个子进程。
        在子进程执行 handleChildProc（pid=0）
        在父进程执行 handleParentProc（pid不为0）

RuntimeInit#findStaticMain 解析：
    根据AMS传传过来的“android.app.ActivityThread”反射拿到其main方法；
    然后创建一个Runnable，在其run方法里边执行反射 ActivityThread#main；
    这个Runnable通过层层返回最终回到了 ZygoteInit#main，在ZygoteInit#main里边调用了run方法。



4、应用进程绑定AMS（IActivityManager：应用进程持有的AMS binder接口；IApplicationThread：AMS持有的应用进程binder接口）
第一步、AMS初始化应用进程的Application
（app）ActivityThread#main -> ActivityThread#attach -> AMS#attachApplication -> AMS#attachApplicationLocked 
-> (app)ApplicationThread#bindApplication（sendMsg BIND_APPLICATION） -> ActivityThread#handleBindApplication -> Instrumentation#callApplicationOnCreate -> Application#onCreate

关键点：

ActivityThread#main 解析：
    public static void main(String[] args) {
        //创建主线程的消息队列
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        //开启主线程的消息循环（保证主线程一直存活的关键）
        Looper.loop();
    }

ActivityThread#attach 解析：
    final ApplicationThread mAppThread = new ApplicationThread(); //实现了IApplicationThread接口
    private void attach(boolean system, long startSeq) {
        if (!system) {
            ...
            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread, startSeq); //AMS绑定ApplicationThread对象
            } catch (RemoteException ex) { }

            //垃圾回收观察者
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    ...
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    //每当系统触发GC，自己就计算下使用了多少内存，如果超过总量的3/4就，就告诉AMS叫它帮忙释放下。
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        try {
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) { }
                    }
                }
            });
        } 
        ...
    }

AMS#attachApplicationLocked 解析：
    private final boolean attachApplicationLocked(IApplicationThread thread, int pid, int callingUid, long startSeq) {
        //AMS调用客户端的binder对象IApplicationThread
        //Application#onCreate就是在这里走的
        thread.bindApplication(...一大波传参);
        ...
        if (normalMode) {
            try {
                if (mStackSupervisor.attachApplicationLocked(app)) { //启动Activity
                    didSomething = true;
                }
            } catch (Exception e) {}
        }
        ...
    }

第二步、AMS创建ClientTransaction传递给应用进程
AMS#attachApplicationLocked（除了初始化Application，另一个重要逻辑就是启动根Activity）-> ActivityStackSupervisor#attachApplicationLocked -> ActivityStackSupervisor#realStartActivityLocked -> ClientLifecycleManager#scheduleTransaction（LaunchActivityItem）-> （app）ApplicationThread#scheduleTransaction -> ActivityThread#scheduleTransaction（ActivityThread继承自ClientTransactionHandler）-> ClientTransactionHandler#scheduleTransaction（sendMsg EXECUTE_TRANSACTION）-> ActivityThread.H#handleMessage（EXECUTE_TRANSACTION）-> TransactionExecutor#execute

关键点：

ActivityStackSupervisor#realStartActivityLocked 解析：封装ClientTransaction，给应用进程执行
    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
                ...
                final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread, r.appToken);
                //添加callback
                clientTransaction.addCallback(LaunchActivityItem.obtain(...一大波传参));

                //判断此时的生命周期是resume还是pause
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                //设置当前的生命周期
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
                ...
            } 
            ...
        return true;
    }

ClientLifecycleManager.scheduleTransaction 解析：将ClientTransaction传给应用进程
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            transaction.recycle();
        }
    }

    //ClientTransaction#schedule
    public void schedule() throws RemoteException {
        //mClient就说IApplicationThread接口
        mClient.scheduleTransaction(this);
    }

TransactionExecutor#execute 解析：执行callback以及更新生命周期状态
    public void execute(ClientTransaction transaction) {
        executeCallbacks(transaction);
        executeLifecycleState(transaction);
    }

    public void executeCallbacks(ClientTransaction transaction) {
        final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
        if (callbacks == null) {
            return;
        }
        ...
        final int size = callbacks.size(); //执行callback
        for (int i = 0; i < size; ++i) {
            final ClientTransactionItem item = callbacks.get(i);
            ...
            //这个item为LaunchActivityItem
            //由于LaunchActivityItem没有实现postExecute，所以只需要分析execute
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
            ...
        }
    }



第三步、应用进程创建实例Activity，走onCreate生命周期。

执行LaunchActivityItem：
TransactionExecutor#executeCallbacks -> LaunchActivityItem#execute -> ClientTransactionHandler#handleLaunchActivity -> ClientTransactionHandler#performLaunchActivity -> ... -> onCreate()

关键点：

ClientTransactionHandler#performLaunchActivity 解析：
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        ContextImpl appContext = createBaseContextForActivity(r); //创建要启动Activity的上下文环境
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //用类加载器来创建该Activity的实例
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            ...
        }
        ...
        try {
            //创建Application,makeApplication会调用Application的onCreate方法
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            ...
            if (activity != null) {
                ...
                //初始化Activity
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
                ...
                //回调onCreate生命周期
                if (r.isPersistable()) { 
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ....
            }
            //设置生命周期状态为onCreate
            r.setState(ON_CREATE);
        } 
        ...
        return activity;
    }


第四步、走onStart、onResume生命周期
TransactionExecutor#executeLifecycleState -> TransactionExecutor#cycleToPath -> TransactionExecutor#performLifecycleSequence -> ActivityThread#handleStartActivity -> ... -> onStart()

TransactionExecutor#executeLifecycleState -> ActivityLifecycleItem#execute -> ActivityThread#handleResumeActivity -> ... -> onResume()

关键点：
TransactionExecutor#executeLifecycleState 解析：
    private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        if (lifecycleItem == null) { return; }
        ...
        //cycleToPath方法作用：根据生命周期轨迹，走接下来的生命周期
        //由于这个ActivityLifecycleItem是ResumeActivityItem，所以getTargetState为ON_RESUME
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */);

        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }

    private void cycleToPath(ActivityClientRecord r, int finish, boolean excludeLastState) {
        final int start = r.getLifecycleState();
        //根据起终点，获取生命周期轨迹的路线。添加onCreate - onResume之间的生命周期
        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
        performLifecycleSequence(r, path);
    }

TransactionExecutor#performLifecycleSequence 解析：
    private void performLifecycleSequence(ActivityClientRecord r, IntArray path) {
        final int size = path.size();
        //遍历生命周期轨迹的路线，一个个按顺序执行
        for (int i = 0, state; i < size; i++) {
            state = path.get(i);
            switch (state) {
                ...
                case ON_START:
                    mTransactionHandler.handleStartActivity(r, mPendingActions);
                    break;
                ...
            }
        }
    }
```



##### 说说Activity任务栈。

```java
ActivityRecord、TaskRecord、ActivityStack 之间的关系。
每个 ActivityStack 中包含有若干个 TaskRecord 对象；
每个 TaskRecord 包含如果若干个 ActivityRecord 对象；
每个 ActivityRecord 记录一个 Activity 所有信息。(一个Activity可以有多个ActivityRecord,因为Activity可以被多次启动，这主要取决于其启动模式。)

  
AMS#mStackSupervisor：
负责管理ActivityStack，内部管理了mHomeStack、mFocusedStack和mLastFocusedStack三个ActivityStack。
mHomeStack管理的是Launcher相关的Activity栈，stackId为0；
mFocusedStack管理的是当前显示在前台Activity的Activity栈；
mLastFocusedStack管理的是上一次显示在前台Activity的Activity栈。
  
  
1、ActivityStack中，HomeStack是被Launcher占用，另外的均是我们启动的应用占用的。点击Home键其实就是ActivityStack们的交替。
2、每个Activity都有一个affinity，默认会是所在应用的包名。
3、启动一个Activity，首先有一个当前的Task，然后依据启动模式，选择是在当前Task添加，还是寻找新的Task。
4、standard:新建实例。当前Task能添加则添加。如：当前Task中的Activity如果是singleInstance则会依据affinity寻找对应Task添加。
	singleTop：和standard一样的步骤找到可添加的Task，然后看顶部的Activity是不是要启动的Activity。
	singleTask：依据affinity找到可添加的Task，然后看Task中是不是有要启动的Activity实例。
	singleInstance：依据affinity查找，是否存在只有要启动的activity的实例的Task，切换到该Task。
```





#### Service

##### Service和Activity怎么进行数据交互 +4

```java
Activity->Service：
直接在Intent里边传递数据就行了。

Service->Activity：
1、通过Binder对象
   定义一个类继承自Binder，然后在Service#onBind中实例化这个类并且返回。
   Activity通过bindService的方式启动service，ServiceConnection#onServiceConnected里边将IBinder强转成那个自定义的类。

2、通过LocalBroadcastReceiver
	在Activity里边动态注册一个本地广播接受者，然后Service发送本地广播。

3、EventBus
```



##### IntentService 的应用场景和内部实现原理？+2

```java
使用场景：后台执行几个耗时的任务，并且任务需要顺序执行。
原理分析：Service + HandlerThread 的结合体

IntentService#onCreate：创建了一个HandlerThread对象，并且把Looper丢给一个Handler。
IntentService#onStartCommand：Handler发送消息（消息id：startId，消息内容：Intent（启动者传过来的Intent））。
IntentService需要重写onHandleIntent方法，根据Intent执行具体的操作，执行完后会自动调用stopSelf(startId)。
当执行完所有任务后，便会自动停止服务。

stopSelf(startId)和stopSelf()区别：
stopSelf()会立刻停止服务，这个时候还有可能有其他消息未处理；
stopSelf(int startId)会等待所有的消息都处理完毕后才会终止服务。（会在执行停止服务之前判断最近启动服务的次数是否和startId相等，相等则停止，不等就等待）。

顺序执行的实现原理：startService会启动服务，如果服务已经启动了，会直接走onStartCommand。借助这个机制，给内部Handler按顺序的发消息，由于Handler有消息队列，所以任务变坏顺序执行。
耗时操作的实现原理：由于Service本身是不能做耗时操作的，所以必须使用线程。HandlerThread内部实现有线程，并且线程内部运行有自己的Looper，将这个Looper丢给一个Handler，便能在这个线程里边执行任务了。
```



##### startService、bindService的区别？生命周期一样吗? +5

```java
startService：
生命周期：
onCreate() -> onStartCommand() -> onDestory()

特点：
1、如果服务已经开启，不会执行 onCreate()， 而是会调用onStartCommand()；
2、一旦服务开启跟启动者就没有任何关系了。


bindService：
生命周期：
onCreate() -> onBind() -> onUnbind() -> onDestory()

特点：
1、绑定服务不会调用 onStartCommand() 方法。
2、绑定成功后，绑定者就能够持有服务的IBinder对象，双方就能够通信了。
```



##### Service与Activity是在同一个线程吗？为什么？

```java
是的，都是在主线程。应该说是Activity、Service的所有生命周期，都是在主线程调用的。
因为Service、Activity的启动都是通过向AMS发出申请，然后AMS那边创建好对应的Record之后，再通知对应的进程的ActivityThread，
而ActivityThread的处理都是在主线程的Handler中发对应的消息。所有的生命周期回调都是在handeMessage里边处理的，所以都是在主线程。
```



##### 说说Service的启动流程

```java
总结：
  
startService
生命周期：
onCreate() -> onStartCommand() -> onDestory()

特点：
1、如果服务已经开启，不会执行 onCreate()， 而是会调用onStartCommand()；
2、一旦服务开启跟启动者就没有任何关系了。

启动流程：（所有生命周期都是在主线程执行的）
1、应用进程通知AMS要启动Service；
2、AMS构建好ServiceRecord之后，通知服务进程（有可能是跨进程启动）创建Service实例，再执行通知服务进程走onStartCommand逻辑；
3、服务进程通过反射创建Service对象，并且调用其onCreate生命周期；
4、服务进程通知AMS，Service启动完成。


bindService
生命周期：
onCreate() -> onBind() -> onUnbind() -> onDestory()

特点：
1、绑定服务不会调用 onStartCommand() 方法。
2、绑定成功后，绑定者就能够持有服务的IBinder对象，双方就能够通信了。

启动流程：（所有生命周期都是在主线程执行的。ServiceConnection回调也是在主线程执行的）
1、应用进程通知AMS要绑定Service，并记录绑定者的信息以及ServiceConnection对象；
2、AMS构建好ServiceRecord之后，通知服务进程（有可能是跨进程启动）创建Service实例，再执行通知服务进程走onBind逻辑；
3、服务进程通过反射创建Service对象，并且调用其onCreate生命周期；
4、随后服务进程走onBind生命周期，将返回的IBinder对象回传给AMS；
5、AMS根据绑定者的信息，找到绑定者的ServiceConnection，并将IBinder对象以及ServiceConnection对象传回绑定者进程；
6、绑定者进程post runnable回主线程后调用ServiceConnection对象的onServiceConnected方法传入IBinder对象。

  
详细分析：
startService：

onCreate:
ContextImpl#startService -> ContextImpl#startServiceCommon -> AMS#startService -> ActiveServices#startServiceLocked -> ActiveServices#startServiceInnerLocked -> ActiveServices#bringUpServiceLocked -> ActiveServices#realStartServiceLocked ->（app）ActivityThread#scheduleCreateService（sendMsg CREATE_SERVICE）-> ActivityThread#handleCreateService -> onCreate

onStartCommand:
ActiveServices#realStartServiceLocked -> ActiveServices#sendServiceArgsLocked -> app）ActivityThread#scheduleServiceArgs（sendMsg SERVICE_ARGS）-> ActivityThread.H#handleMessage（SERVICE_ARGS）-> ActivityThread#handleServiceArgs -> onStartCommand

关键点：
ActiveServices类：AMS里边有一个对象mServices（ActiveServices），ActiveServices类是负责Service相关的逻辑，包括启动，停止，和绑定，以及重启生命周期的调用等。

ActivityThread#handleCreateService 解析：反射创建Service对象，并且调用onCreate方法
	private void handleCreateService(CreateServiceData data) {
	    ...
	    LoadedApk packageInfo = getPackageInfoNoCheck(data.info.applicationInfo, data.compatInfo);
	    Service service = null;
	    try {
	        java.lang.ClassLoader cl = packageInfo.getClassLoader(); //反射创建Service对象
	        service = (Service) cl.loadClass(data.info.name).newInstance();
	    } catch (Exception e) {}

	    try {
	        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
	        ...
	        //如果Application没有创建，则创建一个Application对象。
	        Application app = packageInfo.makeApplication(false, mInstrumentation);

	        service.attach(context, this, data.info.name, data.token, app, ActivityManager.getService());

	        //调用Service的onCreate方法
	        service.onCreate();
	        ...
	        try {
	            ActivityManager.getService().serviceDoneExecuting(
	                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
	        } catch (RemoteException e) {}
	    } catch (Exception e) {}
	}



bindService：

ContextImpl#bindService -> ContextImpl#bindServiceCommon -> AMS#bindService -> ActiveServices#bindServiceLocked -> ActiveServices#bringUpServiceLocked（这里开始与startService流程一直了）-> ActiveServices#realStartServiceLocked ->（app）ActivityThread#scheduleCreateService（sendMsg CREATE_SERVICE）-> ActivityThread.H#handleMessage（CREATE_SERVICE）-> ActivityThread#handleCreateService -> onCreate

onBind生命周期执行流程：
ActiveServices#realStartServiceLocked -> ActiveServices#requestServiceBindingsLocked（只有bindService才执行） -> ActiveServices#requestServiceBindingLocked -> （app）ActivityThread#scheduleBindService（sendMsg BIND_SERVICE） -> ActivityThread#handleBindService -> onBind或onRebind

关键点：
ActiveServices#realStartServiceLocked 解析：
	private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app, boolean execInFg) throws RemoteException {
	    ...
	    try {
	        ...
	        app.thread.scheduleCreateService(r, r.serviceInfo, mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo), app.repProcState);
	    } catch (DeadObjectException e) { } finally { }
	    ...
	    requestServiceBindingsLocked(r, execInFg); //bindService会在该方法里边执行逻辑

	    updateServiceClientActivitiesLocked(app, null, true);

	    if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
	        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
	                null, null, 0));
	    }

	    sendServiceArgsLocked(r, execInFg, true); //如果是startService，这里会走onStartCommand

	    if (r.delayed) {
	        getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
	        r.delayed = false;
	    }
	   	...
	}

	private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)throws TransactionTooLargeException {
		//只有bindService的时候，r.bindings才有值
	    for (int i=r.bindings.size()-1; i>=0; i--) {
	        IntentBindRecord ibr = r.bindings.valueAt(i);
	        if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
	            break;
	        }
	    }
	}

	private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i, boolean execInFg, boolean rebind) throws TransactionTooLargeException {
	    ...
	    if ((!i.requested || rebind) && i.apps.size() > 0) {
	        try {
	            bumpServiceExecutingLocked(r, execInFg, "bind");
	            r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
	            //绑定Service对象
	            r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind, r.app.repProcState);
	            if (!rebind) {
	                i.requested = true;
	            }
	            i.hasBound = true;
	            i.doRebind = false;
	        } catch (TransactionTooLargeException e) {} catch (RemoteException e) {}
	    }
	    return true;
	}


ActivityThread#handleBindService 解析：
	private void handleBindService(BindServiceData data) {
	    //获取Service对象
	    Service s = mServices.get(data.token); 
	    if (s != null) {
	        try {
	           	...
	            try {
	                if (!data.rebind) { //判断是否为重新绑定
	                    //走onBind生命周期，并且获取返回值IBinder对象
	                    IBinder binder = s.onBind(data.intent);
	                    //将IBinder对象，传给AMS
	                    ActivityManager.getService().publishService(data.token, data.intent, binder);
	                } else { //重新绑定走onRebind生命周期
	                    s.onRebind(data.intent);
	                    ActivityManager.getService().serviceDoneExecuting(data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
	                }
	                ...
	            } catch (RemoteException ex) {}
	        } catch (Exception e) {}
	    }
	}

给启动者执行ServiceConnection回调：
在onBind生命周期回调之后，会调用AMS#publishService。
AMS#publishService -> ActiveServices#publishServiceLocked -> （app）ConnectionRecord.InnerConnection（继承自IServiceConnection）#connected -> LoadedApk.ServiceDispatcher#connected（post runnable） -> LoadedApk.ServiceDispatcher#doConnected -> ServiceConnection#onServiceConnected

void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    final long origId = Binder.clearCallingIdentity();
    try {
        if (r != null) {
            Intent.FilterComparison filter
                    = new Intent.FilterComparison(intent);
            IntentBindRecord b = r.bindings.get(filter);
            if (b != null && !b.received) {
                b.binder = service;
                b.requested = true;
                b.received = true;
                //ServiceRecord的connections是一个ArrayMap对象。
                //遍历该ArrayMap对象的Value值，Value值是一个ArrayList对象
                //遍历ArrayList对象，获取每一个ConnectionRecord对象，通过filter找到
                //相应的ConnectionRecord对象
                for (int conni=r.connections.size()-1; conni>=0; conni--) {
                    ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                    for (int i=0; i<clist.size(); i++) {
                        ConnectionRecord c = clist.get(i);
                        if (!filter.equals(c.binding.intent.intent)) {
                            continue;
                        }
                        try {
                            //c是ConnectionRecord对象，其中的conn是InnerConnection
                            //对象，这里实际上是调用了InnerConnection的connected方法。
                            c.conn.connected(r.name, service, false);
                        } catch (Exception e) {
                        }
                    }
                }
            }

            serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
        }
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}

public void doConnected(ComponentName name, IBinder service, boolean dead) {
    ServiceDispatcher.ConnectionInfo old;
    ServiceDispatcher.ConnectionInfo info;

    synchronized (this) {
        old = mActiveConnections.get(name);
        //Service已经绑定过了
        if (old != null && old.binder == service) {
            return;
        }
        //将新的Service信息存储起来
        if (service != null) {
            info = new ConnectionInfo();
            info.binder = service;
            info.deathMonitor = new DeathMonitor(name, service);
            try {
                service.linkToDeath(info.deathMonitor, 0);
                mActiveConnections.put(name, info);
            } catch (RemoteException e) {
                mActiveConnections.remove(name);
                return;
            }
        } else {
            mActiveConnections.remove(name);
        }
        if (old != null) {
            old.binder.unlinkToDeath(old.deathMonitor, 0);
        }
    }

    ///将旧的Service进行解绑的操作
    if (old != null) {
        mConnection.onServiceDisconnected(name);
    }
    if (dead) {
        mConnection.onBindingDied(name);
    }
    //调用mConnection对象的onServiceConnected方法绑定新的Service
    //这里的mConnection就是ServiceConnection对象，也即是我们最开始调用bindService方法时
    //传进来的参数。这里的service参数就是Service服务的onBind方法返回的IBinder对象。
    if (service != null) {
        mConnection.onServiceConnected(name, service);
    }
}
```



1. startService、bindService的区别？生命周期一样吗? +3

4. Service与Activity 是在同一个线程吗？为什么？+2

   

#### BroadcastReceiver

1. 说说Broadcast的注册方式与区别。+3

2. 有序广播与无序广播的区别。

3. BroadcastReceiver 与 LocalBroadcastReceiver 有什么区别？

   

#### Fragment

##### 说说你对Fragment的了解。

**Fragment作用**：

1. Fragment 可以将 Activity 视图拆分为多个区块进行模块化地管理，避免了 Activity 视图代码过度臃肿混乱。
2. 虽然自定义 View 也可以实现，但 Fragment 是在更高的层次封装，具有完善的生命周期管理。

**Fragment源码分析**：
https://www.jianshu.com/p/c86b6a77a43f
https://www.jianshu.com/p/79445d860bd3

FragmentController：它是 FragmentActivity 和 FragmentManager 的中间桥接者，对 Fragment 的操作最终是分发到 FragmentManager 来处理；

FragmentManager：实现了 Fragment 的核心逻辑，负责对 Fragment 执行添加、移除或替换等操作，以及添加到返回堆栈。它的实现类是 FragmentManagerImpl；

FragmentHostCallback：是 FragmentManager 向 Fragment 宿主的回调接口，Activity 和 Fragment 中都有内部类实现该接口，所以 Activity 和 Fragment 都可以作为另一个 Fragment 的宿主（Fragment 宿主和 FragmentManager 是 1 : 1 的关系）；

FragmentTransaction：是 Fragment 事务抽象类，它的实现类 BackStackRecord 是事务管理的主要分析对象。
事务的作用：
可以动态改变 Fragment 状态，使得 Fragment 在一定程度脱离宿主的状态。不过，事务依然受到宿主状态约束，例如：当前 Activity 处于 STARTED 状态，那么 addFragment 不会使得 Fragment 进入 RESUME 状态。只有将来 Activity 进入 RESUME 状态时，才会同步 Fragment 到最新状态。

**Fragment的生命周期实现**：
Fragment只定义了5种生命周期状态，内部是通过状态的升级、降级来分发Fragment生命周期。
static final int INITIALIZING = 0;     初始状态，Fragment 未创建
static final int CREATED = 1;          已创建状态，Fragment 视图未创建
static final int ACTIVITY_CREATED = 2; 已视图创建状态，Fragment 不可见
static final int STARTED = 3;          可见状态，Fragment 不处于前台
static final int RESUMED = 4;          前台状态，可接受用户交互

FragmentManagerImpl#moveToState() //生命周期分发的核心方法
f.mState < newState //状态升级
f.mState > newState //状态降级

| Activity生命周期 | FragmentManager | Fragment状态变化 | Fragment生命周期 |
| :----:| :---: | :----: | :----:|
|     onCreate      |            dispatchCreate             |           INITIALIZING -> CREATE            | onAttach -> onCreate |
|  onStart（首次）  | dispatchActivityCreated dispatchStart |    CREATE -> ACTIVITY_CREATED -> STARTED    | onCreateView -> onViewCreated -> onActivityCreated -> onStart |
| onStart（非首次） |             dispatchStart             |         ACTIVITY_CREATED -> STARTED         |                           onStart                            |
|     onResume      |            dispatchResume             |    STARTED -> RESUMED（Fragment 可交互）    |                           onResume                           |
|      onPause      |             dispatchPause             |             RESUMED -> STARTED              |                           onPause                            |
|      onStop       |             dispatchStop              |         STARTED -> ACTIVITY_CREATED         |                            onStop                            |
|     onDestroy     |            dispatchDestroy            | ACTIVITY_CREATED -> CREATED -> INITIALIZING | onDestroyView -> onDestroy -> onDetach |

**事务的操作**：
add & remove：Fragment 状态在 INITIALIZING 与 RESUMED 之间转移；
detach & attach：Fragment 状态在 CREATE 与 RESUMED 之间转移；
replace： 先移除所有 containerId 中的实例，再 add 一个 Fragment；
show & hide： 只控制 Fragment 隐藏或显示，不会触发状态转移，也不会销毁 Fragment 视图或实例；
hide & detach & remove 的区别： hide 不会销毁视图和实例、detach 只销毁视图不销毁实例、remove 会销毁实例（自然也销毁视图）。不过，如果 remove 的时候将事务添加到回退栈，那么 Fragment 实例就不会被销毁，只会销毁视图。

**事务的提交**：
commit：异步提交，不允许状态丢失	异步（handler.post）
commitAllowingStateLoss：异步提交，允许状态丢失	异步（handler.post）
commitNow：同步提交，不允许状态丢失	同步（事务不允许加入回退栈，因为无法确认事务的顺序）
commitNowAllowingStateLoss：同步提交，允许状态丢失	同步	(事务不允许加入回退栈)
executePendingTransactions：同步执行事务队列中的全部事务



##### Fragment与Activity的生命周期。+5

```java
onAttach()：在Activity.onCreate方法之前调用，可以获取除了View之外的资源
onCreate()：当Fragment第一次与Activity产生关联时就会调用，以后不再调用
onCreateView()：创建Fragment中显示的view,其中inflater用来装载布局文件
onViewCreated()：onCreateView执行完后就会调用
onActivityCreated()：当Activity完成onCreate()时调用。
============================================================
onStart()：当Fragment可见时调用。
============================================================
onResume()：当Fragment可见且可交互时调用。
============================================================  
onPause()：当Fragment不可交互但可见时调用。
============================================================  
onStop()：当Fragment不可见时调用。
============================================================  
onDestroyView()：当Fragment的UI从视图结构中移除时调用。
onDestroy()：销毁Fragment时调用。
onDetach()：当Fragment和Activity解除关联时调用。
  
用===分割开的就是与Activity对应的周期
```



##### 说说Fragment如何进⾏懒加载 +2

```java
https://juejin.cn/post/6844904050698223624#heading-0

show + hide的老方案：onHiddenChanged()
先将所有Fragment添加，需要显示的Fragment调用show()，其他都调hide()。
Fragment显示、隐藏会触发onHiddenChanged回调。可以通过在这个回调里边做懒加载。
abstract class LazyFragment:Fragment(){
    override fun onResume() {
        super.onResume()
        judgeLazyInit()
    }
    override fun onHiddenChanged(hidden: Boolean) {
        super.onHiddenChanged(hidden)
        isVisibleToUser = !hidden
        judgeLazyInit()
    }

    private fun judgeLazyInit() {
        if (!isLoaded && !isHidden) {
            //加载数据.........
            isLoaded = true
        }
    }
}

ViewPager + Fragment的老方案：setUserVisibleHint()
ViewPager切换到Fragment会调Fragment#setUserVisibleHint(true)，被切走的那个Fragment会调setUserVisibleHint(false)
abstract class LazyFragment : Fragment() {
    private var isVisibleToUser = false
    private var isCallResume = false

    override fun onResume() {
        super.onResume()
        isCallResume = true
        judgeLazyInit()
    }

    private fun judgeLazyInit() {
        if (!isLoaded && isVisibleToUser && isCallResume) {
            //加载数据.........
            isLoaded = true
        }
    }

    override fun setUserVisibleHint(isVisibleToUser: Boolean) {
        super.setUserVisibleHint(isVisibleToUser)
        this.isVisibleToUser = isVisibleToUser
        judgeLazyInit()
    }
}

老方案缺陷：就是不可见的Fragment还是执行了onResume()方法，违反了onResume()方法设计的初衷。

AndroidX的新方案：FragmentTransaction#setMaxLifecycle方法。（设置活跃状态下Fragment最大的状态，如果该Fragment超过了设置的最大状态，那么会强制将Fragment降级到正确的状态）。
如果某Fragment，setMaxLifecycle设置了Lifecycle.State.STARTED，那么在隐藏状态下生命周期只能走到onStart。

show + hide的新方案：
fragmentTransaction.hide(aFragment);
fragmentTransaction.setMaxLifecycle(aFragment, Lifecycle.State.STARTED);

abstract class LazyFragment : Fragment() {
    private var isLoaded = false
    override fun onResume() {
        super.onResume()
        if (!isLoaded && !isHidden) {
            //加载数据.........
            isLoaded = true
        }
    }
}

ViewPager + Fragment的新方案：FragmentPagerAdapter构造函数，新参数behavior。
BEHAVIOR_SET_USER_VISIBLE_HINT：当Fragment对用户的可见状态发生改变时，setUserVisibleHint方法会被调用。
BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT：选中的Fragment在Lifecycle.State#RESUMED状态，其他不可见的Fragment会被限制在Lifecycle.State#STARTED 状态。
（实现原理：Adapter在setPrimaryItem方法内部实现了，如果设置了BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT，就给Fragment设置上setMaxLifecycle(Lifecycle.State.STARTED)）

ViewPager2的方案：默认实现了显示的Fragment才执行onResume的逻辑。
```



##### Fragment与Activity如何传输数据？+2

```java
Activity -> Fragment
AFragment fragment = new AFragment(); 
Bundle bundle = new Bundle(); 
fragment.setArguments(bundle);

Fragment -> Activity
Fragment定义个接口，Activity实现这个接口
Fragment#onAttach里边通过强转context获取到接口。
  
其他方式：EventBus；ViewModel（Activity、Fragment公用一个ViewModel）；
```



##### Fragment之间如何进⾏通信？+2

```java
通过Activity作为中间人：
Fragment通过getActivity获取到Activity，Activity通过findFragmentByTag||findFragmentById获取Fragment，Fragment实现接口。
  
其他方式：EventBus；ViewModel。
```



##### 为啥Fragment要用setArguments(Bundle)传参？

```java
Activity重新创建的时候：
Activity#onSaveInstanceState -> FragmentManager#saveAllState -> 将当前Frgment的信息保存起来（包括 传递的Bundle）
  
Activtiy#onCreate（savedInstanceState != null）-> FragmentManager#restoreAllState -> FragmentState#instantitate

//反射调用了Fragment的无参构造函数
Fragment f = (Fragment)clazz.newInstance();  
if (args != null) {  
    args.setClassLoader(f.getClass().getClassLoader());  
    f. mArguments = args; //将数据重新丢回给frament
}  

Fragment重新创建的时候，由于什么原因（例如横竖屏切换）导致你的Fragment重新创建。Activity重创的时候会反射Fragment无参构造函数来创建Fragment。如果在Fragment的构造函数里边传数据，那么传的数据都会不见了。因此推荐使用Fragment.setArguments。
```



##### Fragment的replace和add方法的区别。

```java
replace：替换Fragment，也就是先移除再添加。生命周期内获取获取数据，使用replace会重复获取。
add：添加Fragment，只是覆盖上一个Fragment。add一般会伴随hide()和show()，避免布局重叠。
添加相同Fragment的时候，用replace不会有任何变化，而add会抛移除。
使用add如果应用放在后台或以其他方式被系统销毁，再打开时，hide()中引用的fragment会销毁，所以依然会出现布局重叠bug，可以使用replace或使用add时，添加一个tag参数。
```



##### getFragmentManager，getChildFragmentManager之间的区别？

```java
getFragmentManager：Activity 嵌套 Fragment 时选⽤
getChildFragmentManager：Fragment 嵌套 Fragment 时选⽤

getFragmentManager()所得到的是所在fragment 的父容器的管理器。
getChildFragmentManager()所得到的是在fragment  里面子容器的管理器。
```



##### FragmentPageAdapter和FragmentStatePageAdapter区别及使用场景。

```java
FragmentStatePagerAdapter：
FragmentStatePagerAdapter会销毁不需要的fragment，事务提交后activity的FragmentManager中的fragment会被彻底移除。
类名中的“State”表名：在销毁fragment时，可在onSaveInstanceState(Bundle)方法中保存fragment的Bundle信息。用户切换回来时，保存的实例状态可用来恢复生产新的fragment。

FragmentPageAdapter：
对于不再需要的fragment，FragmentPageAdapter会选择调用事务的detach(Fragment)方法来处理它，而非remove(Fragment)方法，也就是说Fragment只是销毁了fragment视图，Fragment实例还保留在FragmentManager中。因此FragmentPagerAdapter创建的fragment永远不会销毁。

FragmentStatePageAdapter适用于Fragment较多的情况。较多的Fragment都保存在FragmentManager中的话会对应用的性能会造成很大的影响。
FragmentPageAdapter则适用于固定的，少量的Fragment情况。
```



##### commit与commitAllowingStateLoss的区别。

```java
commit()：需要状态保持。即只能使用在在activity的状态存储之前，即在onSaveInstanceState(Bundle outState)之前调用，否则会抛异常。
commitAllowingStateLoss()：允许状态丢失。
```



##### Fragment在Activity中replace的生命周期。+2

```java
消失的Fragment：onPause，onStop，onDestroyView，onDestroy，onDetach
显示的Fragment：onAttach，onCreate，onCreateView，onViewCreated，onStart，onResume
```



##### Fragment在Activity中replace，并addToBackStack的生命周期。

```java
消失的Fragment：onPause，onStop，onDestroyView
显示的Fragment：onAttach，onCreate，onCreateView，onViewCreated，onStart，onResume

如果按返回键会返回到上一个Fragment，其生命周期为：onCreateView，onStart，onResume

使用replace加载fragment，增加addToBackStack()，原来Fragment不会销毁，但是会销毁视图和重新创建视图（回调onDestroyView和onCreateView)
使用replace加载fragment，不增加addToBackStack，fragment会销毁（回调onDestroy)
```



##### Fragment在ViewPager中切换的生命周期。+2

```java
在ViewPager中Fragment在切换时，会优先触发setUserVisibleHint(boolean isVisibleToUser)。可以通过该方法监听Fragment页面的显示与隐藏。
由于ViewPager#setOffscreenPageLimit预加载的存在会同时加载多个Fragment
这些加载的Fragment会先走setUserVisibleHint，当前显示的那个Fragment为true，其余为false。
然后这些加载的Fragment就走生命周期：onAttach，onCreate，onCreateView，onViewCreated，onStart，onResume。
当翻页之后，被反走的Fragment会走销毁的生命周期，不过设置不同的Adapter会走不同的销毁周期。
  
FragmentStatePagerAdapter：直接移除Fragment
销毁的周期：onPause，onStop，onDestroyView，onDestroy，onDetach
恢复的周期：onAttach，onCreate，onCreateView，onViewCreated，onStart，onResume
  
FragmentPageAdapter：分离Fragment，但会缓存其实例
销毁的周期：onPause，onStop，onDestroyView
恢复的周期：onCreateView，onViewCreated，onStart，onResume
```



#### Handler

（handler的更多问题：[面试中 - Handler引发的那些灵魂拷问](https://blog.csdn.net/yudan505/article/details/113716381)）

1. Looper死循环为什么不会触发ANR？+4

2. 说说android消息机制。

3. 两个线程能使用handler通讯吗？为什么？

4. HandlerThread是什么？

5. 单线程模式中Message、Handler、MessageQueue、Looper之间的关系。

6. Handler的post(runnable)是如何实现的。callback、runnable、msg的优先级。

7. Handler的阻塞是如何实现的。

   

#### RecycleView

1. RecycleView与ListView的区别。+2
2. 说说对RecycleView的了解。



#### View

##### Activity、Window、DecorView的关系。+2

```java
android系统上所有的界面都是靠Window机制显示的。每个Activity、Dialog背后也都是Window。

PhoneWindow是Window唯一的实现类，Activity持有一个PhoneWindow，而PhoneWindow持有一个DecorView，而我们写的界面全都是在DecorView上显示的。
PhoneWindow在addView时会创建一个ViewRootImpl，让ViewRootImpl持有DecorView。
ViewRootImpl与WMS通讯，ViewRootImpl是真正绘制DecorView的地方。
```



##### 说说view的绘制流程。+4

```java
PhoneWindow对象创建：
startActivity -> AMS -> ActivityThread#handleLaunchActivity -> ActivityThread#performLaunchActivity -> Activity#attach -> new PhoneWindow()

DecorView对象的创建：
Activity#onCreate() -> setContentView -> new DecorView()

addView DecorView：
Activity#onResume() -> PhoneWindow#addView -> new ViewRootImpl() -> ViewRootImpl#setView -> ViewRootImpl#requestLayout

requestLayout -> scheduleTraversals -> mChoreographer#postCallback -> TraversalRunnable -> doTraversal -> performTraversals
ViewRootImpl#performTraversals()：
performMeasure -> DecorView#measure -> DecorView#onMeasure -> ViewGroup#measure
performLayout -> DecorView#layout -> DecorView#onLayout -> ViewGroup#layout
performDraw -> DecorView#draw -> DecorView#onDraw -> ViewGroup#draw

总结：
startActivity->onCreate->完成DecorView的创建-
>onResume()->DecorView添加到ViewRootImpl->ViewRootImpl.performTraversals()
方法，测量（measure）,布局（layout）,绘制（draw）, 从DecorView自上而下遍历整个View树。

Measure：测量视图宽高。 单一View:measure() -> onMeasure() -> getDefaultSize() 计算View的宽/高值 -> setMeasuredDimension存储测量后的View宽 / 高 
ViewGroup: -> measure() -> 需要重写onMeasure( ViewGroup没有定义测量的具体过程，因为ViewGroup是一个抽象类，其测量过程的onMeasure方法需要各个子类去实现。
如：LinearLayout、RelativeLayout、FrameLayout等等，这些控件的特性都是不一样的，测量规则自然也都不一
样。)遍历测量ViewGroup中所有的View -> 根据父容器的MeasureSpec和子View的LayoutParams等信息计算子
View的MeasureSpec -> 合并所有子View计算出ViewGroup的尺寸 -> setMeasuredDimension 存储测量后的宽 / 高
从顶层父View向子View的递归调用view.layout方法的过程，即父View根据上一步measure子View所得到的布局大小
和布局参数，将子View放在合适的位置上。

Layout：先通过 measure 测量出 ViewGroup 宽高，ViewGroup 再通过 layout 方法根据自身宽高来确定自身
位置。当 ViewGroup 的位置被确定后，就开始在 onLayout 方法中调用子元素的 layout 方法确定子元素的位置。子元素如果是 ViewGroup 的子类，又开始执行 onLayout，如此循环往复，直到所有子元素的位置都被确定，整个
View 树的 layout 过程就执行完了。

Draw：绘制视图。ViewRoot创建一个Canvas对象，然后调用OnDraw()。六个步骤：①、绘制视图的背景；
②、保存画布的图层（Layer）；③、绘制View的内容；④、绘制View子视图，如果没有就不用；⑤、还原图层
（Layer）；⑥、绘制View的装饰(例如滚动条等等)。
```



##### 聊聊ViewGroup的测量、布局过程。

```java
ViewGroup怎么执行测量的：
以LinearLayout为例，在onMeasure方法里边会先区分横竖方向的测量。以竖向的测量为例子。
第1次遍历子View：
如果Layout是精确模式高度已经固定了，子View的大小就随意了反正超不过父。并且带权重的View能分到的高度也已经固定了，在第2次遍历的时候再给带权重的View分配高度。
如果Layout是AT_MOST模式，需要测量子View才能确定父的高度，所以需要调用子View的measure来自测量，带权重的View也需要并且让它按WRAP_CONTENT标准来测量。
第2次遍历子View：
如果有带权重的View，给带权重的View分配高度。制造出一个带精确值的MeasureSpec，然后丢给有带权重的View让其自测量。
总结：
第1次遍历：算高度、宽度。跳过带权重的子View。
第2次遍历：补充给带权重的子View测量高度。
  

View的布局过程：
以LinearLayout为例，在onLayout方法里边也是分横竖布局的。还是以竖向为例子。
根据自身的重力配置（Gravity）计算子View的top应该从哪个位置开始。
然后开始遍历子View，通过子View的测量宽高，然后计算出每个View对应的left、top，然后调用view的layout方法。
```



##### View必须在主线程刷新吗？

```JAVA
ViewRootImpl#requestLayout 都会先执行 ViewRootImpl#checkThread（检查当前线程）
  
怎么才能在子线程刷新ui？
1. 在onResume()之前刷新，因为ViewRootImpl还没创建。
2. 不触发requestLayout。比如设置textView固定宽度，然后setText。
```



##### 如何自定义控件。+3

```java
自定义View: 只需要重写onMeasure()和onDraw()
自定义ViewGroup: 则只需要重写onMeasure()和onLayout()
```



##### 说说MeasureSpace。+2

```java
测量规格，View是根据它来决定自己的大小。
32位int：高2位mode，低30位size。
3种mode：
未知（UNSPECIFIED）：父视图不约束子视图View，一般我们用不上（系统内部，ScrollView）
精确（EXACTLY）：父视图为子视图指定一个确切的尺寸（match_parent、具体值）
尽可能大（AT_MOST）：父视图为子视图指定一个最大尺寸，子视图必须确保自身 & 所有子视图可适应在该尺寸内（wrap_content）

MeasureSpec从哪里来的：
由于绘制流程从DecorView开始，而DecorView的MeasureSpec是通过
ViewRoomImpl#getRootMeasureSpec制造出来的，ViewRoomImpl会根据屏幕宽高以及DecorView的测量模式制造出对应的MeasureSpec。
一般Activity的DecorView宽高都是MATCH_PARENT，所以测量模式为EXACTLY。而Dialog的DecorView宽高可能都是WRAP_CONTENT，所以测量模式为AT_MOST。
ViewGroup以及View的MeasureSpec都是依靠父布局的MeasureSpec加上自身测量得到的。
```



##### 子View创建MeasureSpace的规则是什么。

```java
根据父容器的MeasureSpec和子View的LayoutParams等信息计算子View的MeasureSpec

当View是具体值：精确模式 + 具体值
当View是match_parent：父容器测量模式 +（a.父为精确模式，子大小为父的剩余大小；b.父为AT_MOST，子大小不会超过父的剩余大小）
当View是wrap_content：AT_MOST + 不超过父的剩余大小
```



##### 为什么自定义View设置wrap_content不起作用的？

```java
getDefaultSize方法：
在测量过程中setMeasuredDimension的时候会用到getDefaultSize方法。
getDefaultSize的默认实现中，当View的测量模式是AT_MOST或EXACTLY时，View的大小都会被设置成子View MeasureSpec的specSize。
因为AT_MOST对应wrap_content；EXACTLY对应match_parent，所以，默认情况下，wrap_content和match_parent是具有相同的效果的。

specSize从哪里来：
在计算子View测量尺寸方法getChildMeasureSpec()中，子View MeasureSpec在属性被设置为wrap_content或match_parent情况下，子View MeasureSpec的specSize被设置成parenSize=父容器当前剩余空间大小
  
所以：wrap_content起到了和match_parent相同的作用：等于父容器当前剩余空间大小
```



##### getWidth与getMeasureWidth区别。+2

```java
getWidth与getMeasureWidth区别: 在大多数情况下两者的值是相同的。但是值生成时机不同。
getMeasureWidth在measure之后就是有效值了，而getWidth要在layout之后才能是有效值。
getWidth -> mRight - mLeft
getMeasureWidth -> mMeasuredWidth & MEASURED_SIZE_MASK
在布局过程中，调用child.layout的时候传入的right就是用 left + child.getMeasuredWidth

值不同的情况：重写了View#layout，手动地改了right的值。
```



##### Activity中获取某个View的宽高有几种方法？

```java
1. Activity/View#onWindowFocusChanged：此时View已经初始化完毕，当Activity的窗口得到焦点和失去焦
点时均会被调用一次，如果频繁地进行onResume和onPause，那么onWindowFocusChanged也会被频繁
地调用。
2. view.post(runnable)：post的消息能保证在绘制三大流程走完之后才会被执行。
3. ViewTreeObserver#addOnGlobalLayoutListener：当View树的状态发生改变或者View树内部的View的可
见性发生改变时，onGlobalLayout方法将被回调。
```



##### 为什么onCreate、onResume获取不到View的宽高？

```java
因为View的绘制流程是在onResume之后才进行的。
在onResume之后，DecorView才被add到WindowManger，这时才与ViewRoomImpl关联，才开始走绘制流程。
只有走完测量流程，View才能获取到宽高。
```



##### requestLatout()与invalidate()的区别。

```java
requestLatout：走三大流程(某些情况，不一定走draw)
	设置该view自己的flag（mPrivateFlags）为PFLAG_FORCE_LAYOUT，并且调用父布局的requestLayout。
一直会去到ViewRoomImpl的requestLayout，然后就走绘制的三大流程。
measure：
  如果自己的flag为PFLAG_FORCE_LAYOUT，执行onMeasure。并且设置flag为PFLAG_LAYOUT_REQUIRED。
layout：
  如果自己的flag为PFLAG_LAYOUT_REQUIRED，执行onLayout。
 
===============================================================
        //View是否不需要绘制,两种情况下不需要绘制
        //view设置了OnPreDrawListener并在onPreDraw里返回了false,或者view的Visibility属性不为View.VISIBLE
        boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
        //View不需要绘制跟Surface没有创建的时候，不需要调度绘制函数
        if (!cancelDraw && !newSurface) {
						...           
            performDraw();
        }
===============================================================

invalidate：只走draw流程
  设置自己的flag为PFLAG_DIRTY，并且调用父布局的invalidateChild。
一直会去到ViewRoomImpl的invalidateChildInParent，其实最终也是执行绘制三大流程。
不过由于flag值不同，view最终只会走了draw流程。
```



##### invalidate()和 postInvalidate() 的区别。

```java
invalidate()只能在ui线程调用。
  
postInvalidate()可以在子线程也可以在主线程调用。
其内部就是多了个handler切回主线程的操作，最终也是调用invalidate()。
```



##### 说说view.post跟handler.post的区别。

```java
public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }
        getRunQueue().post(action);
        return true;
    }

handler.post：
是直接add到MessageQueue里边的，loop到它就会执行。
 
view.post：
如果View已经attach到window，直接调用UI线程的handler.post。
如果View还未attach到window，将Runnable放入View内部的一个队列HandlerActionQueue中，等到执行attach的时候在从队列拿出Runnable，再调用UI线程的handler.post。
  
并且view.post能够在Runnable里边获取到View的有效宽高。
ViewRoomImpl#performTraversals在走三大流程前，会调用getRunQueue().executeActions(mAttachInfo.mHandler)
由Handler.post到消息队列，这些消息必定是在performTraversals方法执行完之后才有机会出队被执行的。
```



##### 说说绘制和屏幕刷新机制原理。

```java
帧率：GPU/CPU生成帧的速率（单位fps）
屏幕刷新率：设备刷新屏幕的频率（单位hz），该值对于特定的设备来说是个常量。
帧率和刷新频率没有必然的大小关系。

单缓存：CPU/GPU向Buffer中生成图像，屏幕从Buffer中取图像，刷新后显示。
缺陷：如果帧率比刷新率大的情况，当屏幕还没有刷新第n帧的时候，GPU/CPU已经在生成第n+1帧了，从上往下开始覆盖第n帧的数据。
当屏幕开始刷新第n帧的时候，Buffer中的数据上半部分是已经是第n+1帧数据，而下半部分是第n帧的数据。导致图像会出现上半部分和下半部分明显偏差的现象
出现这个现象，叫做“屏幕撕裂”。

双重缓存：VSync信号 与 BackBuffer、FrameBuffer
GPU向BackBuffer中写数据，屏幕从FrameBuffer中读数据。
VSync信号负责调度从BackBuffer到FrameBuffer的复制操作。

当屏幕刷新周期完成后便会产生VSync信号，然后VSync会先完成复制操作之后再通知CPU/GPU绘制下一帧图像。
复制操作完成后屏幕开始下一个刷新周期，将刚复制到FrameBuffer的数据显示到屏幕上。
在这种模型下，只有当VSync信号产生时，CPU/GPU才会开始绘制。这样当帧率大于刷新频率时，帧率就会被迫跟刷新频率保持同步，从而避免“画面撕裂”。

缺陷：
遇到比较复杂的界面B，CPU/GPU处理时间较长（超过16.6ms），屏幕显示完A界面后发出Vsync信号，CPU/GPU还没把B的数据更新到BackBuffer。
而Vsync从BackBuffer拷贝出来的还是A界面的数据，屏幕继续绘制了A界面的数据。这种现象就是“掉帧”

三重缓存
在双重缓存基础上增加了一个GraphicBuffer缓冲区，让这个Buffer给CPU用，让它提前忙起来。
三缓冲有效利用了等待Vysnc的时间，减少了“掉帧”，保证画面的连续性，提高柔韧性

安卓系统中有2种VSync信号：屏幕产生的硬件VSync和由SurfaceFlinger将其转成的软件Vsync信号。
后者经由Binder传递给Choreographer，来执行刷新界面。
```



##### 说说Choreography原理。

```java
Choreographer：用于与Vsync机制配合，统一动画、输入和绘制时机。

FrameHandler - 发送同步消息、处理消息用
CallbackQueue - 用于保存不同类型的Callback(输入、绘制、动画、提交)
FrameDisplayEventReceiver - 用来接收VSync信号

postCallback方法：将Ruunable加入到CallbackQueue。并且调用一个native方法，是用来向底层注册监听下一个屏幕刷新信号，当下一个屏幕刷新信号发出时，底层就会回调Choreographer的onVsync()方法来通知上层app。

onVsync回调方法：根据返回的下一帧时间sendMessageAtTime() -> FrameDisplayEventReceiver#run -> doFrame
-> 执行CallbackQueue里各个类型的Callback。

总结：Choreographer支持4总类型事件：输入、绘制、动画、提交。并可通过postCallback保存起来，等待Vsync信号到来的时候刷新。
Choreographer有监听Vsync信号，一旦收到信号就会执行doFrame方法，去执行所有类型事件。
```



##### 每隔16.6ms刷新一次屏幕到底指的是什么意思?

```java
安卓系统屏幕刷新率为60hz，就是说屏幕一秒内会刷新60次，也就是16.6ms会刷新一次屏幕。
```



##### 如果界面一直保持没变那么还会每隔16.6ms刷新一次屏幕吗?

```java
如果app界面保持不变app是不会一直刷新的。只有当app向底层注册监听下一个屏幕刷新信号之后，才能接收到下一个屏幕刷新信号的通知，只有收到通知app才回去执行绘制流程。而只有当某个View发起了刷新请求时，app才会去向底层注册监听下一一个屏幕刷新信号。
但是系统屏幕会每隔16.6ms刷新一次，界面保持不变的话一直刷新的是相同帧。
```



##### 为什么会出现丢帧的情况？

```java
1、过度绘制 - 由于界面布局嵌套太多，导致遍历绘制View树计算屏幕数据的时间超过了16.6ms。
2、主线程有耗时操作 - 主线程在处理耗时的操作，导致遍历绘制View树的工作迟迟不能开始从而超过了16.6ms底层切换下一帧画面的时机。
```



##### 如果避免过度绘制？

```java
1.尽量减少布局层级 - 推荐使用ConstraintLayout（约束布局）
2.层级一样的情况下，优先考虑用LinearLayout，其性能要高于RelativeLayout
3.重复的布局，请用 include 标签
4.用 ViewStub 标签来加载一些不常用的布局（比如网络连接异常等等）
5.使用 merge 标签减少布局的嵌套层次
6.不要写多余的background
7.去掉 window 的默认背景
8.对于使用 Selector 当背景的 Layout（比如 ListView 的 Item，会使用 Selector 来标记点击，选择等不同的状态），可以将 normal 状态的 color 设置成”@android:color/transparent”，来解决对应的问题
9.使用包含 layout_weight 属性的 LinearLayout 会在绘制时花费昂贵的系统资源，因为每一个子组件都需要被测量两次。在使用 ListView 与 GridView 的时候，这个问题显得尤为重要，因为子组件会重复被创建，所以要尽量避免使用 layout_weight
10.优化自定义View的计算 - clipRect
学会裁剪掉View的覆盖部分，增加cpu的计算量，来优化GPU的渲染，这个API可以很好的帮助那些有多组重叠组件的自定义View来控制显示的区域。同时clipRect方法还可以帮助节约GPU资源，在clipRect区域之外的绘制指令都不会被执行，那些部分内容在矩形区域内的组件，仍然会得到绘制。并且在onDraw方法中减少View的重复绘制。
```



#### 动画

1. 简单说明android中的几种动画，以及他们的特点和区别。



#### Drawable与Bitmap

1. 如果对bitmap进行压缩。
2. 图片资源放在不同文件夹中，加载出来的内存占用分别是多少，为什么会这样？



#### 事件分发

1. 触摸事件是如何传递的。+5
2. 事件分发是使用了什么设计模式？（责任链模式）
3. 如果解决滑动冲突？+2
4. 同时给一个view和viewgroup设置了点击事件优先响应那个，为什么  ？



#### 数据存储

1. SharedPreference能多进程访问吗？进程间数据共享有什么方式？
2. SharedPreference是如何存储的？存储位置在哪里？
3. 内部存储与外部存储的区别？
4. android 有哪些数据存储方式。+2



#### 性能优化

1. 内存泄漏是如果产生的？如何解决？+3

2. 内存泄漏与内存抖动的区别。+3

3. 怎么app优化启动速度。

4. 如何监测内存泄漏。

5. 什么是ANR，如何避免它？

6. 如何进行app性能优化、内存优化、cpu使用率优化？

7. 内存泄漏的分类。如何分析内存泄漏问题。

8. native崩溃日志如何采集，怎么处理？

   

#### Apk与打包流程

1. apk如何脱壳。

2. 如何进行多渠道打包？

3. 打包app如何进行加固与混淆？

4. 如何进行apk瘦身？

5. 讲述下android的数字签名。

6. apk打包过程中aar中是否包含R文件。

7. jar、aar的区别。

8. V1、V2、V3签名有什么区别。

   

#### 进程间通讯

1. android进程间通讯方式有哪些？
2. binder优势是什么？
3. 说说aidl生成java类的细节。
4. 进程间通讯遇到过哪些问题？



#### Framework

1. 说说view的渲染过程（WMS）。+2
2. 说说app的启动流程。+2
3. launcher启动程序 跟 另一个程序跳转过去两者有什么区别？
4. Activity是在哪里创建的。Application是在哪里创建的。与AMS是如何交互的。
5. 说说类加载器的双亲委托机制。



#### 功能设计相关

1. 如何设计一个类似微信朋友圈首页功能，包括UI、数据等方面。
2. 如何设计一个无线数据的气泡显示聊天内容。
3. 大文件在传输过程中需要考虑哪些问题，如果保证一致性。
4. 如何做到单个信号源，多个页面响应。
5. 设计一个日志系统。



#### 其他

1. 怎么终止一个app?
3. 说说进程如何保活。+2
4. 说说屏幕适配方案。+2
4. 说说android中进程优先级。
5. AsyncTask是什么，使用方法，使用时候需要注意什么？
6. 说说AsyncTask的原理，聊聊它的缺陷和问题。
7. 聊聊android各个版本的新特性。
8. android为每个应用程序分配的内存是多少。
9. JSbridge是如何实现js与native联通的。



## Android第三方库

#### RxJava

#### OKHttp

#### Retrofit

1. 说说你对retrofit的了解。

   

#### Glide



#### Databinding

1. 说说databinding的原理。

   

#### 插件化

1. 启动Activity的hook方式。taskAffity。

   

#### 其他

1. 组件化中module和app之间的区别。module通信是如何实现的。



## Kotlin

#### 协程

1. 说说kotlin的协程。+3
2. 概括一下kotlin协程的上下文。
3. 协程是如何挂起的。



#### 语法相关

1. 高阶函数是什么？
2. ==与===的区别。
3. lateinit与lazy的区别。



#### 其他

1. kotlin与java有什么区别。+3
2. kotlin有什么优缺点？+2
3. kotlin的lambda与java的lambda有什么区别。



## NDK

#### JNI

1. jni的注册方法有哪些。+2

2. JNIEnv *env是什么？有什么用？

3. extern "C"的作用

   ```java
   是让被作用的代码块采用c语言的编译规则编译
   
   如果没加extern "C"：
   运行的时候就会抛异常，找不到函数
   定义的函数：Java_com_kobe_MainActivity_stringFromJNI
   编译之后的函数：Z40Java_com_kobe_MainActivity_stringFromJNIP7_JNIEnvP8_jobject
   
   C不支持函数的重载，编译之后函数名不变
   C++支持函数的重载，编译之后函数名会变
   静态注册的JNI接口，需要考虑C++编译之后函数名变化的问题，所以需要加上extern "C"的关键字。
   动态注册的JNI接口，就不会有这个问题
   
   
   源码是怎么使用的：头文件有可能被C语言或者C++语言使用，怎么做兼容
   使用宏定义判断当前的编译器是不是c++编译器，这种使用技巧，可以说在android源码中随处可见
   #if defined(__cplusplus)
   extern "C" {
   #endif
   void *memset(void *s, int c, size_t n);
   #if defined(__cplusplus)
   }
   #endif
   ```

   

   

#### C++相关

1. 函数指针如何写？

2. c++中构造函数的调用顺序，析构函数是否需要virtual。

3. . 与 -> 的区别。哪个性能更高。

4. :: 与 : 的区别。

   

#### 其他

1. 视频编解码是怎么做的。
