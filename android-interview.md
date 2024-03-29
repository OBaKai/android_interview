## 四大组件

### Activity  ✅

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
总结：
1、Activity的生命stop/destory是依赖IdleHandler来回调，也就是在启动下一个Activity#onResume之后的那段空闲时间，才会执行的。
2、在Activity#onResume之后也会发出一个10s的兜底事件，防止stop/destory一直不执行。
3、如果在主线程的Handler消息一直很繁忙的话，是会影响stop/destory的回调。最严重的情况会出现10s才回调。

详细分析：
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



### Service ✅

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
1、应用进程调用startService，通知AMS要启动Service；
2、AMS构建好ServiceRecord之后，检查该Service是否已经创建（ServiceRecord#app、ServiceRecord#appThread != null）;
3、如果Service已经创建了，通知应用进程执行Service#onStartCommand；
4、如果Service没有创建，通知应用进程创建Service，应用进程创建Service对象并执行其onCreate，最后将Service对象缓存起来。紧接着AMS会通知进程执行Service#onStartCommand；
5、如果需要启动的Service的进程未启动，AMS会先叫Zygote孵化该进程，然后将创建Service动作延后处理。等到进程通知AMS自己已经启动完成后，AMS就会通知进程走创建Service的逻辑。


bindService
生命周期：
onCreate() -> onBind() -> onUnbind() -> onDestory()

特点：
1、绑定服务不会调用 onStartCommand() 方法。
2、绑定成功后，绑定者就能够持有服务的IBinder对象，双方就能够通信了。

绑定流程：（所有生命周期都是在主线程执行的。ServiceConnection回调也是在主线程执行的）
1、应用进程通知AMS要绑定Service，AMS记录绑定者的信息以及ServiceConnection对应的Binder对象（IServiceConnection）；
2、AMS构建好ServiceRecord之后，通知服务进程启动Service（跟startService启动流程一样），如果服务进程没启动就先启动进程，如果服务没启动就启动服务（服务进程会创建Service对象，并且调用其onCreate生命周期）。
3、服务启动后，AMS会向其请求回去Binder对象，这时服务进程就会回调服务的onBind，并且返回Binder对象给AMS。
4、AMS拿到服务的Binder对象后，就会给应用进程发布这个Binder对象，通过调用应用进程的IServiceConnection#connected发布到应用进程。
5、应用进程这边会在ServiceConnection#onServiceConnected中回调服务的IBinder对象，自此绑定流程就结束了。
```



### BroadcastReceiver ✅

##### 说说Broadcast的注册方式与区别。+3

```java
动态注册的接收者：
  1、创建好的接收者对象，在注册时会根据Context缓存到对应的Map中；
	2、然后会通知AMS自己要一个动态广播，AMS会根据传过来的IIntentReceiver对象也缓存到对应的一个Map中。

静态注册的接收者：
  1、在清单文件定义好的广播接收者，在app安装完成 或者 系统启动后清单文件会被PMS扫描；
  2、PMS将广播接收者的描述信息保存在PMS里边的mReceivers容器中。
 
广播分发流程：
  1 AMS根据广播的描述，收集符合条件的接收者；
  2 如果该广播是无序广播；
      2.1 查询出符合条件的动态接收者，将它们封装到一个BroadcastRecord，然后将BroadcastRecord加入到一个无序队列之后，走分发流程（并行分发）。
      2.2 通过PMS查询出符合条件的静态接收者，将它们封装到一个BroadcastRecord，然后将BroadcastRecord加入到一个有序队列之后，走分发流程（串行分发）。
  3 如果该广播是有序广播；
      会将符合条件的动态接收者 以及 静态接收者根据广播优先级合并到一起。将它们封装到一个BroadcastRecord，然后将BroadcastRecord加入到一个有序队列之后，走分发流程（串行分发）。
  4 分发机制在BroadcastQueue中实现；(mParallelBroadcasts（无序队列）、mOrderedBroadcasts（有序队列）)
      4.1 优先遍历无序队列，并行分发广播给所有的接收者。
          遍历接收者们，通通给它们每人发一次广播，发了就完事。也不需要它们告诉AMS自己有没有收到。
      4.2 接着遍历有序队列，串行分发广播给所有的接收者。
          4.2.1 如果该接收者是动态注册的，直接分发。
          4.2.2 如果该接收者是静态注册的，就先看进程是否已经启动，如果启动了直接分发。
          4.2.3 如果接收者的进程没有启动，先将广播标记为pending，进程启动后在attachApplication时继续处理这个pending的广播。
          4.2.4 串行机制：接收者收到广播后，告诉AMS已经收到了，AMS就会让BroadcastQueue继续分发广播给下一个接收者。
          4.2.5 超时机制：在分发广播前会发出一个延时Msg，用来防止分发广播时间过长。如果广播分发超时了，会放弃本次分发，处理下一个分发。

区别：
  1、动态注册接收者缓存在AMS中，静态注册接收者缓存在PMS中。
  2、无序广播分发过程中，动态注册接收者是优先分发的，并且分发方式是并行分发（更快收到广播），静态注册接收者是是在串行分发的。有序广播分发过程中，不管是动态注册接收者还是静态注册接收者，统一是串行分发。
```



##### 有序广播与无序广播的区别。

```java
1 AMS根据广播的描述，收集符合条件的接收者；
2 如果该广播是无序广播；
    2.1 查询出符合条件的动态接收者，将它们封装到一个BroadcastRecord，然后将BroadcastRecord加入到一个无序队列之后，走分发流程（并行分发）。
    2.2 通过PMS查询出符合条件的静态接收者，将它们封装到一个BroadcastRecord，然后将BroadcastRecord加入到一个有序队列之后，走分发流程（串行分发）。
3 如果该广播是有序广播；
    会将符合条件的动态接收者 以及 静态接收者根据广播优先级合并到一起。将它们封装到一个BroadcastRecord，然后将BroadcastRecord加入到一个有序队列之后，走分发流程（串行分发）。

4 分发机制在BroadcastQueue中实现；(mParallelBroadcasts（无序队列）、mOrderedBroadcasts（有序队列）)
    4.1 优先遍历无序队列，并行分发广播给所有的接收者。
        遍历接收者们，通通给它们每人发一次广播，发了就完事。也不需要它们告诉AMS自己有没有收到。
    4.2 接着遍历有序队列，串行分发广播给所有的接收者。
        4.2.1 如果该接收者是动态注册的，直接分发。
        4.2.2 如果该接收者是静态注册的，就先看进程是否已经启动，如果启动了直接分发。
        4.2.3 如果接收者的进程没有启动，先将广播标记为pending，进程启动后在attachApplication时继续处理这个pending的广播。
        4.2.4 串行机制：接收者收到广播后，告诉AMS已经收到了，AMS就会让BroadcastQueue继续分发广播给下一个接收者。
        4.2.5 超时机制：在分发广播前会发出一个延时Msg，用来防止分发广播时间过长。如果广播分发超时了，会放弃本次分发，处理下一个分发。
```



##### BroadcastReceiver 与 LocalBroadcastReceiver 有什么区别？

```java
LocalBroadcastManager（单例）
	维护着三个容器：
	private final HashMap<BroadcastReceiver, ArrayList<ReceiverRecord>> mReceivers = new HashMap<>();
  private final HashMap<String, ArrayList<ReceiverRecord>> mActions = new HashMap<>();
  private final ArrayList<BroadcastRecord> mPendingBroadcasts = new ArrayList<>();

registerReceiver方法：（注册接收者）
	1、根据 receiver 和 filter 创建ReceiverRecord，存入 mReceivers 里边对应的list中；
	2、根据 filter 中的 action 将上边创建的ReceiverRecord，存入 mActions 里边对应的list中。
  mReceivers中存放的是已注册的receiver以及其对应的ReceiverRecord，同一个receiver如果多次用相同的filter不会生效多次，但是如果filter不同注册多次，那么就会在mReceivers存储多个ReceiverRecord；
  mAction中存放的是已注册的所有action，以及这个action对应的ReceiverRecord，如果这个filter中有多个action，那么就会存储多个相同的ReceiverRecord。

unregisterReceiver方法：（反注册接收者）
	分别从mReceivers、mActions移除对应的缓存数据（与registerReceiver方法刚好相反的操作）

sendBroadcast方法：（发送广播）
	1、根据 action 从 mActions 取出对应的ArrayList<ReceiverRecord>；
	2、遍历ArrayList<ReceiverRecord>，用filter与广播的Intent信息进行匹配，匹配成功就加入分发队列mPendingBroadcasts；
	3、发送handle消息MSG_EXEC_PENDING_BROADCASTS，在主线程中执行发送广播；
	4、发送广播逻辑（executePendingBroadcasts方法）：遍历mPendingBroadcasts取出每个ReceiverRecord，轮询ReceiverRecord内的接收者，调用其onReceive方法。

sendBroadcastSync方法：（同步发送广播）
	public void sendBroadcastSync(@NonNull Intent intent) {
        if (sendBroadcast(intent)) {
            executePendingBroadcasts(); //从handler改为直接执行。
        }
    }


区别：
BroadcastReceiver是注册到AMS中的，能实现跨进程接收广播。广播是通过AMS分发过来的。
LocalBroadcastReceiver本身是一个单例，接收者注册到这个单例里边的一个容器中。广播发送也是通过这个单例实现的，遍历接收者容器中符合条件的对象调用其onReceive。
```



### ContentProvider ✅

##### ContentProvider是什么，作用是什么。

```java
ContentProvider的作用是为不同的应用之间数据共享，提供统一的接口。
安卓系统中应用内部的数据是对外隔离的，要想让其它应用能使用自己的数据（例如通讯录）这个时候就用到了ContentProvider。

通过 URI（统一资源标识符）来标识其它应用要访问的数据。
	每一个 ContentProvider 都拥有一个公共的URI，用于表示这个 ContentProvider 所提供的数据。
	例如：  content://com.xxx.xx/User/1
	content://：主题，ContentProvider的标准前缀
	com.xxx.xx：授权信息，URI的标识用于唯一标识这个 ContentProvider ，外部调用者可以根据这个标识来找到它。
	User：路径，通俗的讲就是你要操作的数据库中表的名字，
	1：记录，如果URI中包含表示需要获取的记录的 ID；则就返回该id对应的数据，如果没有 ID，就表示返回全部。
通过 ContentResolver（内容解析者）的增、删、改、查方法实现对共享数据的操作。
通过 ContentObserver（内容观察者）来监听数据是否发生了变化来对应的刷新页面。
```



##### ContentProvider,ContentResolver,ContentObserver之间的关系

```java
ContentProvider：管理数据，提供数据的增删改查操作，数据源可以是数据库、文件、XML、网络等。
ContentResolver：外部进程可以通过 ContentResolver 与 ContentProvider 进行交互。其他应用中
ContentResolver 可以不同 URI 操作不同的 ContentProvider 中的数据。
ContentObserver：观察 ContentProvider 中的数据变化，并将变化通知给外界。
```



##### 说说ContentProvider的优点

```java
封装
	采用ContentProvider方式，其 解耦了 底层数据的存储方式，使得无论底层数据存储采用何种方式，外界对数
	据的访问方式都是统一的，这使得访问简单 & 高效
	如一开始数据存储方式 采用 SQLite 数据库，后来把数据库换成 MongoDB，也不会对上层数据
	ContentProvider使用代码产生影响
提供一种跨进程数据共享的方式。
	应用程序间的数据共享还有另外的一个重要话题，就是数据更新通知机制了。因为数据是在多个应用程序中共享
	的，当其中一个应用程序改变了这些共享数据的时候，它有责任通知其它应用程序，让它们知道共享数据被修改了，
	这样它们就可以作相应的处理。
```



##### 说说ContentProvider启动流程

```java
1、调用者是通过ContentResolver来增删查改。
2、ContentResolver是个抽象类，ApplicationContentResolver继承了ContentResolver。
ApplicationContentResolver是在ContextImpl构造函数中被创建。可以通过Content#getContentResolver方法使用到它。
3、ContentResolver在调用insert、delete、update、query的时候，ContentProvider才会被启动。
4、调用者进程会向AMS请求ContentProviderHodler。
5、AMS收到调用者进程的请求，就会处理返回Hodler逻辑
  5.1 如果Provider对应的ContentProviderRecord存在，直接返回Hodler。
  5.2 如果Provider对应的ContentProviderRecord不存在，创建一个。
  	5.2.1 如果ContentProvider能跑在调用者进程，直接返回Hodler（该Holder不会带有Binder对象）。否则继续往下走。
  	5.2.2 如果ContentProvider所在进程没有启动，就先启动进程，然后等待发布Binder对象，完成的时候返回Hodler。
  	5.2.3 如果ContentProvider所在进程已经启动，请求发布Binder对象，然后等待发布Binder对象，完成的时候返回Hodler。
  	备注：AMS等待（wait()）的过程中会一直阻塞调用者进程的那条binder线程，直到Provider所在的进程发布Binder对象给AMS（发布完成后会notifyAll()）。
6、调用者进程安装ContentProviderHodler，获取IContentProvider。
  如果Hodler带有IContentProvider，则直接返回。
  如果Hodler没有IContentProvider，证明ContentProvider能跑在调用者进程，需要调用者进程自行安装。
  则直接创建一个ContentProvider对象，并让其走onCreate生命周期。IContentProvider则从ContentProvider#getIContentProvider方法中获取得到。


Provider进程发布Binder对象的两种方式：
  1、Provider所在进程启动时主动发布Provider的Binder对象。
  	AMS会在Provider所在进程启动后，通知它需要启动哪些Provider（AMS#attachApplicationLocked）。当启动完那些需要启动的Provider后，进程就会发布Binder对象到AMS。
  	备注：Provider启动是在Application对象创建之后执行的。（Application#attachBaseContext -> ContentProvider#onCreate -> Application#onCreate）
  
  2、AMS向Provider所在进程申请Binder对象。
  	Provider所在的进程，就会启动该Provider（Provider走onCreate）
```



##### 为什么ContentProvider#onCreate 比 Application#onCreate快

```java
app启动流程中的Application启动流程：
1、在app进程启动完成之后，会通知AMS自己进程已经启动完成了。
2、AMS会通知app进程执行启动Application流程，并且会把该app的所有在清单文件注册的Provider信息返回给app进程。
3、app进程在创建完Application实例，走了Application#attachBaseContext之后。会先遍历AMS传过来的Provider信息列表，先安装这些Provider，这时Provider就会走onCreate。并且会把Provider的binder对象发布给AMS。
4、完成安装Provider之后，Application才会走onCreate。

为什么安装ContentProvider要比Application启动要先走？
因为如果有调用者进程需要用到某个Provider，但是这个Provider无法在调用者进程安装的情况下。AMS会查找这个Provider有没有发布binder对象到AMS，如果没有并且Provider所在进程没有启动，AMS就会启动这个进程，等待binder对象的发布，并且阻塞调用者进程的这条binder线程。
如果Provider进程是在这种情况下被启动的，那么Provider组件的启动就显得尤为重要了，所以Provider的启动才会比Application要快。如果给Application流程先走完，Application#onCreate里边开发者会初始化一大堆东西，会耽误不少时间。

那为什么Application#attachBaseContext又比ContentProvider#onCreate先走？
Provider的启动需要用到Context。
在创建完Application实例后Application的Context就创建好了，并且立马会调用attachBaseContext告诉外部，我的Context已经可以用了。
```



##### 为什么ContentProvider可以有多实例

```java
因为Provider除了能在Provider所在进程创建的话，还可以在调用者进程创建。
只要符合这个条件：Provider所在进程的uid要跟调用者的uid相同。
```



## Android UI

### Fragment ✅

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
replace：替换Fragment，也就是先移除再添加。生命周期内获取数据，使用replace会重复获取。
add：添加Fragment，只是覆盖上一个Fragment。add一般会伴随hide()和show()，避免布局重叠。
添加相同Fragment的时候，用replace不会有任何变化，而add会抛异常。
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



### View ✅
##### 说说布局加载过程
```java
Activity#setContentView -> PhoneWindow#setContentView -> LayoutInflate#inflate（resId）

inflate的两步操作：
1、解析xml文件
	Resources#getLayout -> Resources#loadXmlResourceParser
	解析xml文件，IO操作。（耗时）
2、填充View树
	LayoutInflate#inflate（） -> LayoutInflate#createViewFromTag
		如果有设置Factory2，Facory，则通过Factory2、Facory生成View；（Factory2只能设置一次，第二次设置就会抛异常）
		如果没有设置，则通过默认方式（createView）生成View；
		默认方式（createView）：遍历布局文件中每个标签，反射创建View实例，填入View树。

	如果是AppCompatActivity，还会重写setContentView，通过AppCompatDelegate去篡改部分View。
		AppCompatDelegateImpl：
			解析xml：LayoutInflater#inflate（resId）解析xml
			填充View树：实现了Factory2，取代默认方式。并且在createView中篡改部分View。
				TextView -> AppCompatTextView，ImageView -> AppCompatImageView等等

总结：
解析xml文件是个IO操作，文件比较大会很耗时。
填充View树的过程，是通过反射创建View实例的，反射创建对象性能不够高。


统计界面布局耗时：
基本方式：setContentView前后增加时间统计。
	缺点：代码入侵性强；不能统一配置每个Activity都要加一次麻烦。
AOP方式：使用AspectJ，hook所有的setContentView。


优化方案：
AsyncLayoutInflater：异步加载xml
	原理：将resId加入到队列中，通过内部的一个线程解析xml，然后切换回主线程去填充View树。
	缺点：队列限长10，频繁加载layout可能会出现入队阻塞；内部的AsyncLayoutInflater不支持设置Factory；单线程工作不一定满足需求。
	解决：仿照AsyncLayoutInflater，自己写一个异步加载xml类。

减少xml布局的嵌套层级，避免过度绘制。
如果避免过度绘制
移除window的背景：AppTheme都默认带会有windowBackground，如果我们不用，可以移除掉。
减少给子控件设置背景。
使用约束布局，减少布局嵌套。
使用ViewStub来懒加载载View。（ViewStub是一个不可见的View类，当inflate或setVisible才会显示。当inflate View之后，ViewStub就会被移除。inflate只能用一次）
等
```

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



##### getDecorView 和 peekDecorView 有什么区别

```java
   //PhoneWindow里边这两个方法的实现的实现
   @Override
	 public final @NonNull View getDecorView() { //该方法确保了 DecorView 不为空
        if (mDecor == null || mForceDecorInstall) {
            installDecor(); //多调用一次installDecor
        }
        return mDecor;
    }

    @Override
    public final View peekDecorView() {
        return mDecor;
    }


installDecor()是在 setContentView(int layoutResID)方法内被调用，为windows添加decorview页面

installDecor执行流程
使用 generateDecor()创建一个DecorView对象,并赋值给mDecor变量。该变量并不完全等同于窗口修饰，窗口修饰是mDecor内部的唯一一个子视图。
根据用户指定的参数选择不用的窗口修饰，并把该窗口修饰作为mDecor的子窗口，这是在generateLayout()中调用mDecor.addView()完成的.
给mContentParent变量赋值，其值是通过调用ViewGroup.contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT)获得的，ID_ANDROID_CONTENT正是id = content 的 FrameLayout
```





#### RecycleView

#####  说说RecyclerView的回收复⽤机制

```java
回收复用的目标对象为ViewHolder
  
三级缓存 或 四级缓存
0、mChangedScrap & mAttachedScrap
mAttachedScrap：存放可见范围内的ViewHolder。（如果position或者id对应的上，无需重新绑定数据）
在onLayoutChildren时会先把当前可见的ViewHolder都移除掉，再重新添加进去。这些可见的ViewHolder就是临时缓存在mAttachedScrap的。
作用：在布局过程中，减少绑定数据次数

mChangedScrap：存放可见范围内且数据发生了变化的ViewHolder。（复用需重新绑定数据）
notifyItemChanged等方法调用之后，将发生变化的ViewHolder缓存到mChangedScrap。

1、mCachedViews：存放remove掉的ViewHolder。（如果position或者id对应的上，无需重新绑定数据）//默认值大小是2
我的理解：在用户重复短距离上下滑的场景，让RecyclerView进行快速回收与复用，提高性能。
作用：减少绑定数据次数

2、mViewCacheExtension (提供给用户自定义的)
  
3、RecycledViewPool：存放remove掉并且重置了数据的ViewHolder。（复用需重新绑定数据）
作用：减少ViewHolder的创建
	3.1 当mCachedViews缓存满了以后会根据FIFO（先进先出）的规则把ViewHolder移出并缓存到RecycledViewPool中。
	3.2 数据结构是SparseArray<ScrapData>，根据itemType将缓存分组，组的数据结构是ScrapData
	3.3 ScrapData对应的数据结构是ArrayList<ViewHolder>，每个itemType对应的ScrapData的缓存大小默认值是5，可以修改缓存大小
	3.4 可以提供给多个RecyclerView共享
  
从触摸事件分析回收复用原理：
RecyclerView#onTouchEvent（ACTION_MOVE）-> RecyclerView#scrollByInternal -> LayoutManager#scrollVerticallyBy（以垂直滑动为例）-> LinearLayoutManager#scrollVerticallyBy -> LinearLayoutManager#scrollBy -> LinearLayoutManager#fill  

int fill(RecyclerView.Recycler recycler, LayoutState layoutState, RecyclerView.State state, boolean stopOnFocusable) {
		...
        //判断当前是否为滚动状态，如果是则触发回收
        if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
            ...
            recycleByLayoutState(recycler, layoutState); //回收
        }
        ...
        while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
            ...
            layoutChunk(recycler, state, layoutState, layoutChunkResult); //复用
            ...
            if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
                ...
                recycleByLayoutState(recycler, layoutState); //回收
            }
            ...
        }
        ...
    }

复用原理：
LinearLayoutManager#layoutChunk -> LinearLayoutManager.LayoutState#next -> RecyclerView.Recycler#getViewForPosition -> RecyclerView.Recycler#tryGetViewHolderForPositionByDeadline

    void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state, LayoutState layoutState, LayoutChunkResult result) {
        View view = layoutState.next(recycler);
        ...
        RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) view.getLayoutParams();
        //addView方法、addDisappearingView方法最终都是走addViewInt方法，将View添加到RecyclerView
        //addViewInt -> ChildHelper#addView -> 将View添加到RecyclerView，并且调用dispatchChildAttached
        //dispatchChildAttached处理 Adapter#onViewAttachedToWindow、onChildViewAttachedToWindow回调
        if (layoutState.mScrapList == null) {
            if (mShouldReverseLayout == (layoutState.mLayoutDirection == LayoutState.LAYOUT_START)) {
                addView(view);
            } else {
                addView(view, 0);
            }
        } else {
            if (mShouldReverseLayout == (layoutState.mLayoutDirection == LayoutState.LAYOUT_START)) {
                addDisappearingView(view);
            } else {
                addDisappearingView(view, 0);
            }
        }
        ...
    }

    ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
            ...
            boolean fromScrapOrHiddenOrCache = false;
            ViewHolder holder = null;
            // 0) 我们调用notifyItemChanged方法时，先从mChangedScrap里找
            if (mState.isPreLayout()) {
                holder = getChangedScrapViewForPosition(position); //从mChangedScrap获取holder
                ...
            }
            // 1) 通过 position 在 mAttachedScrap / mHiddenViews / mCachedViews 中获取hodler
            if (holder == null) {
                holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
                ...
            }
            if (holder == null) {
                ...
                final int type = mAdapter.getItemViewType(offsetPosition);
                // 2) 通过id在 mAttachedScrap / mCachedViews 中获取holder
                if (mAdapter.hasStableIds()) {
                    holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition), type, dryRun);
                    ...
                }
                //尝试从mViewCacheExtension（自定义缓存）里边找
                if (holder == null && mViewCacheExtension != null) {
                    final View view = mViewCacheExtension.getViewForPositionAndType(this, position, type);
                    if (view != null) {
                        holder = getChildViewHolder(view);
                        ...
                    }
                }
                if (holder == null) { //根据item type从RecycledViewPool获取holder
                    holder = getRecycledViewPool().getRecycledView(type);
                    ...
                }
                if (holder == null) { //最终通过createViewHolder来创建一个holder
                    ...
                    holder = mAdapter.createViewHolder(RecyclerView.this, type);
                    ...
                }
            }
            ...
            return holder;
        }

回收原理：
LinearLayoutManager#recycleByLayoutState -> recycleViewsFromEnd 或 recycleViewsFromStart -> recycleChildren -> RecyclerView.LayoutManager#removeAndRecycleViewAt -> RecyclerView.Recycler#recycleView -> RecyclerView.Recycler#recycleViewHolderInternal

	public void removeAndRecycleViewAt(int index, @NonNull Recycler recycler) {
        final View view = getChildAt(index);
        removeViewAt(index); //移除View
        recycler.recycleView(view); //进行回收
    }

	public void recycleView(@NonNull View view) {
        ViewHolder holder = getChildViewHolderInt(view); //获取ViewHolder
        if (holder.isTmpDetached()) {
        	//调用Adapter#onViewDetachedFromWindow
        	//如果有注册ChildAttachStateChangeListener，则回调这些监听者onChildViewDetachedFromWindow
            removeDetachedView(view, false);
        }
        ...
        recycleViewHolderInternal(holder); //回收ViewHolder逻辑
        ...
    }

    void recycleViewHolderInternal(ViewHolder holder) {
            ...
            boolean cached = false;
            if (forceRecycle || holder.isRecyclable()) {
                if (mViewCacheMax > 0 && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID | ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_UPDATE | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
                    //判断是否满足放进mCachedViews
                    int cachedViewSize = mCachedViews.size();
                    //判断mCachedViews是否已满
                    if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                        //如果满了就将下标为0（即最早加入的）移除，同时将其加入到 RecyclerPool 中
                        recycleCachedViewAt(0);
                        cachedViewSize--;
                    }

                    ...
                    mCachedViews.add(targetCacheIndex, holder); //将ViewHolder加入mCachedViews
                    cached = true;
                }
                //如果没有满足上面的条件，则直接存进RecyclerPool中
                if (!cached) {
                    addViewHolderToRecycledViewPool(holder, true);
                    recycled = true;
                }
            }
            ...
        }
```



##### ViewHolder何时被缓存到RecycledViewPool中？

##### 说说CachedView和RecycledViewPool的关系

```java
在RecycledView滚动的过程中，当item离开屏幕后触发回收机制。
回收机制会优先把ViewHolder缓存到CachedView中，如果CachedView满了（默认大小2），按照先进先出，会先将最早加入CachedView的ViewHolder移除并且加入到RecycledViewPool中，然后再把回收的ViewHolder加入到CachedView。
```



##### 说说CachedView和RecycledViewPool两者区别

```java
1、数据结构
CachedView缓存：ArrayList<ViewHolder>  默认大小2
RecycledViewPool缓存：SparseArray<ScrapData>（key：viewType），ScrapData类（ArrayList<ViewHolder> 默认大小5）

2、缓存优先级
回收的ViewHolder都是进CachedView缓存的。当CachedView满了，会将CachedView最早的移除并放到到RecycledViewPool，腾出位置给回收的ViewHolder。

3、是否需要重新绑定数据
CachedView缓存：存放remove掉的ViewHolder。（如果position或者id对应的上，无需重新绑定数据）
RecycledViewPool缓存：存放remove掉并且重置了数据的ViewHolder。（复用需重新绑定数据）
```



##### 如何对RecycleView进⾏局部刷新的？

```java
方法1：notifyItemChanged(int position, Object payload)
背景：notifyItemChanged(int position)刷新某个item，出现图片闪一下的现象。
原因：其刷新是直接让该item重新走一次绑定数据。

notifyItemChanged(int position, Object payload)可以通过payload参数，实现item局部刷新
在payload传入需要刷新的部位表示。
然后还需要onBindViewHolder三个参数的方法来配合：onBindViewHolder(VH holder, int position, List<Object> payloads)

    public void onBindViewHolder(final RecyclerView.ViewHolder holder, final int position,List payloads) {
        if(payloads.isEmpty()){ //走整体刷新逻辑
           ...
        }else{ //走局部刷新逻辑
            int type= (int) payloads.get(0);
            switch(type){
                case 0:
                    userName.setText(mList.get(position).getName());//只刷name
                    break;
                case 1:
                    userId.setText(mList.get(position).getId());//只刷id
                    break;
            }    
        }
    }


方法2：Diffutil
背景：多个数据变化的情况（可能多个数据的局部变化），需要自己手动比对后才能知道哪些item需要刷新，较为麻烦。
使用：
1、实现自己的DiffUtil.Callback
public class ChatListCallback extends DiffUtil.Callback {
    private ArrayList<ChatItemBean> old_chats, new_chats;

    public ChatListCallback(ArrayList<ChatItemBean> old_chats, ArrayList<ChatItemBean> new_chats) {
        this.old_chats = old_chats;
        this.new_chats = new_chats;
    }

    @Override
    public int getOldListSize() { return old_chats.size(); }

    @Override
    public int getNewListSize() { return new_chats.size(); }

    /**
     * 判断此id的用户消息是否已存在
     */
    @Override
    public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
        String newId = old_chats.get(oldItemPosition).getUserId();
        String oldId = new_chats.get(newItemPosition).getUserId();
        if (oldId == null || newId == null) return false;
        else if(oldId.trim().equals("") || newId.trim().equals("") || (!oldId.equals(newId))) return false;
        else return true;
    }

    /**
     * 若此id的用户消息已存在，则判断内容是否一致
     */
    @Override
    public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
        String newAvatarPath = new_chats.get(newItemPosition).getAvatarPath();
        String oldAvatarPath = old_chats.get(oldItemPosition).getAvatarPath();

        String oldNickname = old_chats.get(oldItemPosition).getNickname();
        String newNickname = new_chats.get(newItemPosition).getNickname();

        if ((newAvatarPath == null && oldAvatarPath != null) || (oldAvatarPath == null && newAvatarPath != null)||( oldAvatarPath != null && !oldAvatarPath.equals(newAvatarPath)) ||
                        (newNickname == null && oldNickname != null) || (oldNickname == null && newNickname != null)||( oldNickname != null && !oldNickname.equals(newNickname))) return false;

        return true;
    }

    @Override
    public Object getChangePayload(int oldItemPosition, int newItemPosition) {
        String newAvatarPath = new_chats.get(newItemPosition).getAvatarPath();
        String oldAvatarPath = old_chats.get(oldItemPosition).getAvatarPath();

        String oldNickname = old_chats.get(oldItemPosition).getNickname();
        String newNickname = new_chats.get(newItemPosition).getNickname();

        Bundle bundle = new Bundle();

        if ((newAvatarPath == null && oldAvatarPath !=null) || (newAvatarPath != null && oldAvatarPath == null) || (oldAvatarPath != null && !oldAvatarPath.equals(newAvatarPath)))
            bundle.putString("avatarPath", newAvatarPath);
        if((newNickname == null && oldNickname != null) || (oldNickname == null && newNickname != null)||( oldNickname != null && !oldNickname.equals(newNickname)))
            bundle.putString("nickname", newNickname);

        if (bundle.size() == 0) return null;
        return bundle;
    }
}

2、通过DiffUtil.calculateDiff方法，比对数据
//注意：比对数据可能是耗时的，最好放到子线程
DiffUtil.DiffResult result = DiffUtil.calculateDiff(new ChatListCallback(oldChats, chats));
//刷新Adapter。注意：如果比对数据放在子线程，这里需切换回主线程
result.dispatchUpdatesTo(adapter);

3、Adapter需要实现onBindViewHolder(ChatItemViewHolder holder, int position, List<Object> payloads)
    @Override
    public void onBindViewHolder(ChatItemViewHolder holder, int position) {
        ...
        holder.mTvNickname.setText(chatItemBean.getNickname());
    }

    @Override
    public void onBindViewHolder(ChatItemViewHolder holder, int position, List<Object> payloads) {
        if (payloads.isEmpty())
            onBindViewHolder(holder, position);
        else {
            Bundle bundle = (Bundle) payloads.get(0);
            for (String key: bundle.keySet()){
                switch (key){
                    case "avatarPath":
                        ...
                        break;
                    case "nickname":
                        holder.mTvNickname.setText((CharSequence) bundle.get(key));
                        break;
                }
            }
        }
    }
```



##### 你是从哪些方面优化RecyclerView的？

```java
1、减少item布局的嵌套层级；
==========================================
2、减少notifyDataSetChanged刷新全部item，可改为使用单个item刷新 或者 局部刷新方案;
==========================================
3、onCreateViewHolder 和 onBindViewHolder 避免创建过多对象以及避减少不必要的操作。
	1）创建对象可以全局创建一个，例如在创建OnClickListener。
	2）减少onBindViewHolder中的操作。因为回收复用机制，会经常调到onBindViewHolder重新刷新数据。
	onBindViewHolder调用次数要比onCreateViewHolder多很多，所有可以将部分操作放到ViewHolder构造函数里边，例如setOnClickListener。
==========================================
4、重写onScroll事件，可在滑动停止后再加载。
对于大量图片的RecyclerView滑动暂停后再加载 或者 可以考虑对滑动速度、滑动状态进行判断，符合加载条件再加载（防止用户快速滑动过程中依旧不断在加载）。
==========================================
5、适当增大缓存大小（setItemViewCacheSize(int)）
RecyclerView可以设置自己所需要的ViewHolder缓存数量，默认大小是2。cacheViews中的缓存只能position相同才可得用，且不会重新bindView，CacheViews满了后移除到RecyclerPool中，并重置ViewHolder，如果对于可能来回滑动的RecyclerView，把CacheViews的缓存数量设置大一些，可以减少bindView的时间，加快布局显示。
	注：此方法是拿空间换时间，要充分考虑应用内存问题，根据应用实际使用情况设置大小。

	网上大部分设置CacheView大小时都会带上：
  setDrawingCacheEnabled(true)和setDrawingCacheQuality(View.DRAWING_CACHE_QUALITY_HIGH)
	setDrawingCacheEnabled这个是View本身的方法，意途是开启缓存。通过setDrawingCacheEnabled把cache打开，再调用getDrawingCache就可以获得view的cache图片，如果cache没有建立，系统会自动调用buildDrawingCache方法来生成cache。一般截图会用到，这里的设置drawingcache，可能是在重绘时不需要重新计算bitmap的宽高等，能加快dispatchDraw的速度，但开启drawingcache，肯定也会耗应用的内存，所以也慎用。
==========================================
6、recyclerView.setHasFixedSize(true);
当Item的高度是固定，设置这属性可以提高性能。尤其是当RecyclerView有插入、删除时性能提升更明显。
RecyclerView在条目数量改变，会重新测量、布局各个Item，如果设置了这个属性。
由于Item的宽高都是固定的，Adapter的内容改变时，RecyclerView不会整个布局都重绘。
具体可用以下伪代码表示：
void onItemsInsertedOrRemoved() {
   if (hasFixedSize) layoutChildren();
   else requestLayout();
}
==========================================
7、RecyclerView 数据预取（https://juejin.cn/post/6844903661382959118）
android sdk>=21时，支持渲染（Render）线程，RecyclerView数据显示分两个阶段：
1）在UI线程，处理输入事件、动画、布局、记录绘图操作，每一个条目在进入屏幕显示前都会被创建和绑定view；
2）渲染（Render）线程把指令送往GPU。
数据预取的思想就是：将闲置的UI线程利用起来，提前加载计算下一帧的Frame Buffer

如果使用系统提供的LayoutManager默认使用了这种优化。如果使用嵌套RecyclerView或者自定义LayoutManager，则需要在代码中设置。
1）对于嵌套 RecyclerView，要获取最佳的性能，在内部的 LayoutManager 中调用LinearLayoutManager.setInitialItemPrefetchCount()方法（25.1版本起可用）。
例如：如果竖直方向的list至少展示三个条目，调用 setInitialItemPrefetchCount(4)。
2）如果自己实现了LayoutManager，需要重写 LayoutManager.collectAdjacentPrefetchPositions()方法。该方法在数据预取开启时被 RecyclerView 调用（LayoutManager 的默认实现什么都不做）。在嵌套的内层 RecyclerView 中，如果想让LayoutManager 预取数据，同样应当实现 LayoutManager.collectInitialPrefetchPositions()。
==========================================
8、getExtraLayoutSpace为LayoutManager设置更多的预留空间
在RecyclerView的元素比较高，一屏只能显示一个元素的时候，第一次滑动到第二个元素会卡顿。  

RecyclerView (以及其他基于adapter的view，比如ListView、GridView等)使用了缓存机制重用子 view（即系统只将屏幕可见范围之内的元素保存在内存中，在滚动的时候不断的重用这些内存中已经存在的view，而不是新建view）。

这个机制会导致一个问题，启动应用之后，在屏幕可见范围内，如果只有一张卡片可见，当滚动的时 候，RecyclerView找不到可以重用的view了，它将创建一个新的，因此在滑动到第二个feed的时候就会有一定的延时，但是第二个feed之 后的滚动是流畅的，因为这个时候RecyclerView已经有能重用的view了。

如何解决这个问题呢，其实只需重写getExtraLayoutSpace()方法。根据官方文档的描述 getExtraLayoutSpace将返回LayoutManager应该预留的额外空间（显示范围之外，应该额外缓存的空间）。

LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this) {
    @Override
    protected int getExtraLayoutSpace(RecyclerView.State state) {
        return 300;
    }
};
==========================================
9、RecycledViewPool的复用
在TabLayout+ViewPager+RecyclerView的场景中，当多个RecyclerView有相同的item布局结构时，多个RecyclerView共用一个RecycledViewPool可以避免创建ViewHolder的开销，避免GC。RecycledViewPool对象可通过RecyclerView对象获取，也可以自己实现。

RecycledViewPool mPool = mRecyclerView1.getRecycledViewPool();
下一个RecyclerView可直接进行setRecycledViewPool
mRecyclerView2.setRecycledViewPool(mPool);
mRecyclerView3.setRecycledViewPool(mPool);

注意：
（1）RecycledViewPool是依据ItemViewType来索引ViewHolder的，必须确保共享的RecyclerView的Adapter是同一个，或view type 是不会冲突的。
（2）RecycledViewPool可以自主控制需要缓存的ViewHolder数量，每种type的默认容量是5，可通过setMaxRecycledViews来设置大小。mPool.setMaxRecycledViews(itemViewType, number); 但这会增大应用内存开销，所以也需要根据应用具体情况来使用。
（3）利用此特性一般建议设置layout.setRecycleChildrenOnDetach(true);此属性是用来告诉LayoutManager从RecyclerView分离时，是否要回收所有的item，如果项目中复用RecycledViewPool时，开启该功能会更好的实现复用。其他RecyclerView可以复用这些回收的item。
什么时候LayoutManager会从RecyclerView上分离呢，有两种情况：1）重新setLayoutManager()时，比如淘宝页面查看商品列表，可以线性查看，也可以表格形式查看，2）还有一种是RecyclerView从视图树上被remove时。但第一种情况，RecyclerView内部做了回收工作，设不设置影响不大，设置此属性作用主要针对第二种情况。
==========================================
10、RecyclerView中的一些方法
onViewRecycled()：当 ViewHolder 已经确认被回收，且要放进 RecyclerViewPool 中前，该方法会被回调。移出屏幕的ViewHolder会先进入第一级缓存ViewCache中，当第一级缓存空间已满时，会考虑将一级缓存中已有的ViewHolder移到RecyclerViewPool中去。在这个方法中可以考虑图片回收。

onViewAttachedFromWindow()： RecyclerView的item进入屏幕时回调
onViewDetachedFromWindow()：RecyclerView的item移出屏幕时回调

onAttachedToRecyclerView() ：当 RecyclerView 调用了 setAdapter() 时会触发，新的 adapter 回调 onAttached。
onDetachedFromRecyclerView()：当 RecyclerView 调用了 setAdapter() 时会触发，旧的 adapter 回调 onDetached

setHasStableIds()／getItemId()：setHasStableIds用来标识每一个itemView是否需要一个唯一标识，当stableId设置为true的时候，每一个itemView数据就有一个唯一标识。getItemId()返回代表这个ViewHolder的唯一标识，如果没有设置stableId唯一性，返回NO_ID=-1。通过setHasStableIds可以使itemView的焦点固定，从而解决RecyclerView的notify方法使得图片加载时闪烁问题。注意：setHasStableIds()必须在 setAdapter() 方法之前调用，否则会抛异常。因为RecyclerView.setAdapter后就设置了观察者，设置了观察者stateIds就不能变了。
```



##### RecyclerView预布局pre-layout是什么？

```java
RecyclerView pre-layout原理：
场景：列表只能显示两个item，当前显示是item1、item2。删除item2，需要item3平滑地移入并占据item2的位置。
执行动画轨迹，需要起点跟终点，终点是item2的位置，那么起点如何确定呢？由于LayoutManager只加载可见的item，删除item2之前，item3是处于不可见的，它并不会被layout。

pre-layout的生命周期：https://juejin.cn/post/6890288761783975950#heading-1
为动画执行前，先执行一次pre-layout，将item3加载到布局中。形成一张布局快照（item1、item2、item3）
再执行一次layout，形成一张布局快照（item1、item3）。对比两张快照，便知道item3的位置了。就知道它该如何做动画了。

//RecyclerView.State#mInPreLayout：pre-layout标志位
boolean mInPreLayout = false;

//RecyclerView#onLayout -> RecyclerView#dispatchLayout	
//在dispatchLayout中mInPreLayout的值标记了预布局的生命周期
    void dispatchLayout() {
        ...
        mState.mIsMeasuring = false;
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1(); //分发布局1
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2(); //分发布局2
        } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth() || mLayout.getHeight() != getHeight()) {
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else {
            mLayout.setExactMeasureSpecsFrom(this);
        }
        dispatchLayoutStep3(); //分发布局3
    }

    private void dispatchLayoutStep1() {
        ...
        mState.mInPreLayout = mState.mRunPredictiveAnimations; //pre-layout开始
        ...
        if (mState.mRunPredictiveAnimations) { //pre-layout，会走一次onLayoutChildren
            ...
            mLayout.onLayoutChildren(mRecycler, mState);
            ...
        }
        ...
    }

    private void dispatchLayoutStep2() {
        ...
        mState.mInPreLayout = false; //pre-layout结束
        mLayout.onLayoutChildren(mRecycler, mState); //开始正真的布局
        ...
    }

pre-layout的处理逻辑：
在预布局阶段，循环填充item时，若遇到被移除item，则会忽略它占用的空间，多余空间被用来加载额外的item，这些item在屏幕之外，本来不会被加载。

pre-layout与缓存机制：https://juejin.cn/post/6892809944702124045#heading-7
每次RecyclerView填充表项之前（onLayoutChildren执行的时候）都会先清空LayoutManager中现存的item，将它们detach并且加入到缓存列表中。然后再从缓存列表中取出item，进行填充。

当item是 不需要更新 或 被移除 或 item索引无效 的时候触发的onLayoutChildren，会缓存到mAttachedScrap列表中。
反之如果只是item需要更新数据的话，会缓存到mChangedScrap。


为什么要detach并缓存表项到 scrap 中，然后紧接着又在填充表项时从中取出？
因为 RecyclerView 要做item动画，
为了确定动画的种类和起终点，需要比对动画前和动画后的两张“item快照”，
为了获得两张快照，就得布局两次，分别是预布局和后布局（布局即是往列表中填充表项），
为了让两次布局互不影响，就不得不在每次布局前先清除上一次布局的内容（就好比先清除画布，重新作画），
但是两次布局中所需的某些表项大概率是一摸一样的，若在清除画布时，把item的所有信息都一并清除，那重新作画时就会花费更多时间（重新创建 ViewHolder 并绑定数据），
RecyclerView 采取了用空间换时间的做法：在清除画布时把表项缓存在 scrap 中，以便在填充表项可以命中缓存，以缩短填充表项耗时。
```



#### ViewPager

##### viewpager设置warp_content为什么会无效？

```java
场景：子View设置了高为100dp，ViewPager设置高为warp_content，根Layout高为match_parent。
实际：发现ViewPager高并不是100dp，而是占满了根Layout。
期望：ViewPager受子View高度影响，变成100dp。

原因：
常规的Layout在onMeasure的时候会先测量所有子View的宽高，然后才测量自己。
但是ViewPager不一样，ViewPager在onMeasure中是先测量自己。导致了子View无法影响到ViewPager。
再由于ViewPager是warp_content，那么它被根Layout测量的时候模式为 AT_MOST + 不超过父的剩余大小。
所以ViewPager沾满了根Layout。

解决：
重写ViewPager#onMeasure，先测量所有子View。然后将最大的子View宽高传给ViewPager。
```



##### 说说viewpager缓存机制？

```java
mOffscreenPageLimit：离屏缓存页数量。当设置为1时，会缓存当前页左右两边的1页。（默认值为1，传入<1的值也是强制设置为1）
populate()：该方法在 setAdapter、setOffscreenPageLimit、onMeasure、翻页 等都会被调用。
Adapter的所有方法都在populate方法里边有调用（populate方法管理着每一个item）。缓存逻辑也是在该方法里边实现的。

void populate(int newCurrentItem) {
        ...
        mAdapter.startUpdate(this); //开始更新

        //缓存范围 [startPos, endPos] == [mCurItem - pageLimit, mCurItem + pageLimit]
        final int pageLimit = mOffscreenPageLimit;
        final int startPos = Math.max(0, mCurItem - pageLimit);
        final int N = mAdapter.getCount();
        final int endPos = Math.min(N - 1, mCurItem + pageLimit);

        ...

        int curIndex = -1;
        ItemInfo curItem = null;
        for (curIndex = 0; curIndex < mItems.size(); curIndex++) {
            final ItemInfo ii = mItems.get(curIndex);
            if (ii.position >= mCurItem) {
                if (ii.position == mCurItem) curItem = ii;
                break;
            }
        }

        if (curItem == null && N > 0) {
            //addNewItem方法：调用mAdapter.instantiateItem 以及 构建ItemInfo添加到mItems缓存里边
            //ItemInfo缓存着Adapter#instantiateItem构建出来的对象（返回Fragment就缓存Fragment，返回View就缓存View）
            curItem = addNewItem(mCurItem, curIndex);
        }

        if (curItem != null) {
            //左边item进行缓存
            float extraWidthLeft = 0.f;
            int itemIndex = curIndex - 1;
            ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
            final int clientWidth = getClientWidth();
            final float leftWidthNeeded = clientWidth <= 0 ? 0 :
                    2.f - curItem.widthFactor + (float) getPaddingLeft() / (float) clientWidth;
            for (int pos = mCurItem - 1; pos >= 0; pos--) {
                if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
                    if (ii == null) {
                        break;
                    }
                    if (pos == ii.position && !ii.scrolling) {
                        mItems.remove(itemIndex);
                        mAdapter.destroyItem(this, pos, ii.object);
                        itemIndex--;
                        curIndex--;
                        ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                    }
                } else if (ii != null && pos == ii.position) {
                    extraWidthLeft += ii.widthFactor;
                    itemIndex--;
                    ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                } else {
                    ii = addNewItem(pos, itemIndex + 1);
                    extraWidthLeft += ii.widthFactor;
                    curIndex++;
                    ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                }
            }

            //右边item进行换粗
            float extraWidthRight = curItem.widthFactor;
            itemIndex = curIndex + 1;
            if (extraWidthRight < 2.f) {
                ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                final float rightWidthNeeded = clientWidth <= 0 ? 0 :
                        (float) getPaddingRight() / (float) clientWidth + 2.f;
                for (int pos = mCurItem + 1; pos < N; pos++) {
                    if (extraWidthRight >= rightWidthNeeded && pos > endPos) {
                        if (ii == null) {
                            break;
                        }
                        if (pos == ii.position && !ii.scrolling) {
                            mItems.remove(itemIndex);
                            mAdapter.destroyItem(this, pos, ii.object);
                            ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                        }
                    } else if (ii != null && pos == ii.position) {
                        extraWidthRight += ii.widthFactor;
                        itemIndex++;
                        ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                    } else {
                        ii = addNewItem(pos, itemIndex);
                        itemIndex++;
                        extraWidthRight += ii.widthFactor;
                        ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                    }
                }
            }

            calculatePageOffsets(curItem, curIndex, oldCurInfo);
            //FragmentPagerAdapter：调用了Fragment#setUserVisibleHint
            mAdapter.setPrimaryItem(this, mCurItem, curItem.object);
        }
        //FragmentPagerAdapter：调用了FragmentTransaction#commitNowAllowingStateLoss
        mAdapter.finishUpdate(this); //结束更新
        ...
    }
```





#### 事件分发

1. 触摸事件是如何传递的。+5
2. 事件分发是使用了什么设计模式？（责任链模式）
3. 如果解决滑动冲突？+2
4. 同时给一个view和viewgroup设置了点击事件优先响应那个，为什么  ？



#### 动画

1. 简单说明android中的几种动画，以及他们的特点和区别。



#### Drawable

##### 说说drawable 与 mipmap 的区别

```java
google建议 只把app启动图标放在 mipmap 目录中，其他图片资源仍然放在 drawable 下面。

mipmap是一种纹理映射技术，mipmap技术主要为了应对图片大小缩放的处理。为了提高缩小的速度和图片的质量，android会通过mipmap 技术提前对按缩小层级生成图片预先存储在内存中，这样来提高图片渲染的速度和质量。（使用mipmap技术会增加内存负担）

mipmap 和 drawable 的区别也就是这个设置是否开启的区别。
mipmap 目录下的图片默认 setHasMipMap 为 true
drawable 默认 setHasMipMap 为 false
```



##### 什么是drawable？

参考：https://blog.csdn.net/arnozhang12/article/details/52621191

```java
drawable：可简单理解为可绘制物，表示一些可以绘制在 Canvas 上的对象。在日常的工作开发中，我们为 UI 配置背景、图片、动画等等界面效果的时候，需要和众多的 Drawable 打交道。
  
Drawable
    |- createFromPath
    |- createFromResourceStream
    |- createFromStream
    |- createFromXml
    |
    |- inflate   : 从XML中解析属性，子类需重写
    |- setAlpha  : 设置绘制时的透明度
    |- setBounds : 设置Canvas为Drawable提供的绘制区域
    |- setLevel  : 控制Drawable的Level值，这个值在ClipDrawable、RotateDrawable、ScaleDrawable、AnimationDrawable等Drawable中有重要作用；区间为[0, 10000]
    |- draw(Canvas) : 绘制到Canvas上，子类必须重写
```



##### 说说drawable加载流程？

参考：https://blog.csdn.net/brian512/article/details/53363642

```java
//以View加载背景资源为例：	
public void setBackgroundResource(@DrawableRes int resid) {
        if (resid != 0 && resid == mBackgroundResource) {
            return;
        }

        Drawable d = null;
        if (resid != 0) {
            d = mContext.getDrawable(resid);
        }
        setBackground(d);

        mBackgroundResource = resid;
	}

//Drawable的加载：缓存机制（缓存Drawable.ConstantState） + 享元（每个Drawable都是新对象）
Resources#getDrawable -> Resources#getDrawableForDensity -> ResourcesImpl#loadDrawable -> ResourcesImpl#loadDrawableForCookie

ResourcesImpl#decodeImageDrawable -> ImageDecoder#decodeDrawable -> BitmapFactory#decodeBitmap -> BitmapFactory#decodeResourceStream

Drawable loadDrawable(@NonNull Resources wrapper, @NonNull TypedValue value, int id, int density, @Nullable Resources.Theme theme) throws NotFoundException {
        ...

        try {
            ...
            final Drawable.ConstantState cs;
            if (isColorDrawable) { //是否为ColorDrawable
                cs = sPreloadedColorDrawables.get(key); //从ColorDrawable缓存列表中获取
            } else {
                cs = sPreloadedDrawables[mConfiguration.getLayoutDirection()].get(key); //从BitmapDrawable缓存列表中获取
            }

            Drawable dr;
            if (cs != null) {
                dr = cs.newDrawable(wrapper);
            } else if (isColorDrawable) {
                dr = new ColorDrawable(value.data);
            } else {
                dr = loadDrawableForCookie(wrapper, value, id, density);
            }
            ...
            if (dr != null) {
                ...
                if (useCache) {
                    //将Drawable缓存起来
                    cacheDrawable(value, isColorDrawable, caches, theme, canApplyTheme, key, dr);
                    ...
                }
            }

            return dr;
        } catch (Exception e) { ... }
    }

private Drawable loadDrawableForCookie(@NonNull Resources wrapper, @NonNull TypedValue value, int id, int density) {
        ...
        final Drawable dr;
        ...
            try {
                if (file.endsWith(".xml")) { //xml读取
                    if (file.startsWith("res/color/")) {
                        dr = loadColorOrXmlDrawable(wrapper, value, id, density, file);
                    } else {
                        dr = loadXmlDrawable(wrapper, value, id, density, file);
                    }
                } else { //图片读取
                    //AssetManager#openNonAsset()是native方法，读取图片文件流
                    final InputStream is = mAssets.openNonAsset(value.assetCookie, file, AssetManager.ACCESS_STREAMING);
                    AssetInputStream ais = (AssetInputStream) is;
                    //将InputStream转成Drawable
                    dr = decodeImageDrawable(ais, wrapper, value);
                }
            } finally {
                stack.pop();
            }
        ...
        return dr;
    }

private Drawable decodeImageDrawable(@NonNull AssetInputStream ais, @NonNull Resources wrapper, @NonNull TypedValue value) {
        ImageDecoder.Source src = new ImageDecoder.AssetInputStreamSource(ais, wrapper, value);
        try {
            return ImageDecoder.decodeDrawable(src, (decoder, info, s) -> {
                decoder.setAllocator(ImageDecoder.ALLOCATOR_SOFTWARE);
            });
        } catch (IOException ioe) { ... }
    }


public static Drawable decodeDrawable(@NonNull Source src, @Nullable OnHeaderDecodedListener listener) throws IOException {
        Bitmap bitmap = decodeBitmap(src, listener);
        return new BitmapDrawable(src.getResources(), bitmap);
    }


public static Bitmap decodeBitmap(@NonNull Source src, @Nullable OnHeaderDecodedListener listener) throws IOException {
        TypedValue value = new TypedValue();
        value.density = src.getDensity();
        ImageDecoder decoder = src.createImageDecoder();
        if (listener != null) {
            listener.onHeaderDecoded(decoder, new ImageInfo(decoder), src);
        }
        return BitmapFactory.decodeResourceStream(src.getResources(), value,
                ((InputStreamSource) src).mInputStream, decoder.mOutPaddingRect, null);
    }


public static Bitmap decodeResourceStream(@Nullable Resources res, @Nullable TypedValue value, @Nullable InputStream is, @Nullable Rect pad, @Nullable Options opts) {
        validate(opts);
        if (opts == null) { opts = new Options(); }

        //设置屏幕密度，也就是说Drawable会根据屏幕密度来加载图片，所以资源图片放错位置，或者太大也是会导致OOM的
        if (opts.inDensity == 0 && value != null) {
            final int density = value.density;
            if (density == TypedValue.DENSITY_DEFAULT) {
                opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;
            } else if (density != TypedValue.DENSITY_NONE) {
                opts.inDensity = density;
            }
        }
        
        if (opts.inTargetDensity == 0 && res != null) {
            opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
        }
        
        return decodeStream(is, pad, opts);
    }


//Drawable的绘制：
View#setBackground（给全局变量mBackground赋值）
View#draw -> View#drawBackground -> Drawable#draw（BitmapDrawable重写了draw）-> 调用了canvas.drawBitmap绘制背景
```



##### 说说drawable跟屏幕密度的关系

```java
各个不同的文件夹对应的具体密度是多少？
0			nodpi
0-120 		ldpi
120-160		mdpi（48x48）
160-240		hdpi（72x72）
240-320		xhdpi（96x96）
320-480		xxhdpi（144x144）
480-640		xxxhdpi（192x192）
图片在低dpi的目录，那图片会被认为是为低密度设备需要的，现在要显示在高密度设备上，图片会进行放大。
图片在高dpi的目录，那图片会被认为是为高密度设备需要的，现在要显示在低密度设备上，图片会进行缩放。
图片在nodpi目录，则无论设备dpi为多少，保留原图片大小，不进行缩放


各个文件夹该放置多大分辨率的图片？
scale = 设备dpi / 图片所在drawable目录对应的最大dpi

假设当前的设备密度是480dpi, 72x72的图分别放在 xxhdpi 和 xhdp ，两种情况下图片各占多少内存？（假设现在是RGB565格式（2个字节））
xxhdpi:（72 * 72 * 2）*（480/480）= 10368字节
xhdpi:（72 * 72 * 2）*（480/320）= 15552字节
如果在开发时不恰当的将高分辨的图片放在低密度的文件夹里，很可能在高密度的设备上显示的时候会OOM，因为它们会进行放大！！

drawable文件夹的匹配规则
1.根据设备的密度，找到屏幕密度匹配的drawable文件夹，在里边找
2.屏幕密度匹配的drawable文件夹没找到，先更高密度的drawable中找，如果没找到再去更高密度drawable中找
3.如果高密度的drawable都没有，就去nodpi里边找
4.如果nodpi也没有，就去低密度drawable中找
5.如果所有drawable都找不到，就报错
```





#### Bitmap

##### 说说Bitmap变迁与原理解析

参考：https://www.jianshu.com/p/d5714e8987f3

```java
android2.1之前 - Bitmap的像素存储在Native上分配（生命周期不可控，需要用户自己释放）

android8.0之前 - Bitmap的像素存储：Dalvik的Java堆上
public final class Bitmap implements Parcelable {
    ...
    private byte[] mBuffer; //数据存储在Java堆中，Native层解析完数据后创建一个Java层的byte[]并返回
    ...
}

android8.0之后 - Bitmap的像素存储：又回到Native上分配，并且无需用户主动释放
public final class Bitmap implements Parcelable {
    ...
    private final long mNativePtr; //Java层只持有了Native层Bitmap对象地址，数据在Native层解析完之后通过calloc开辟内存直接存放
    ...
 }

Dalvik的Java堆上分配像素内存：
虚拟机内存是受限的，由系统配置dalvik.vm.heapgrowthlimit决定。
如果启用largeheap开关，虚拟机内存也只能增大到dalvik.vm.heapsize的大小。
dalvik.vm.heapgrowthlimit=192m
dalvik.vm.heapsize=512m

缺点：容易OOM
一旦Bitmap的内存占用达到虚拟机内存上限的时候，在虚拟机就会抛出OOM异常。

优点：OOM容易捕获
由于是在Java虚拟机抛出的异常，非常容易被捕获。


Native上分配像素内存：
Bitmap内存占用的无限增长，不受虚拟机内存限制。
直到App无法再从系统分配到内存，才会崩溃。

缺点：Native内存OOM难以捕获
Bitmap内存无限增长的情况下也会导致APP崩溃。但是这种崩溃已经不是OOM崩溃了，Java虚拟机也不会捕获。按道理说，应该属于linux的OOM了。
（这个时候崩溃并不为Java虚拟机控制，直接进程死掉，不会有Crash弹框）

优点：
不容易OOM；
降低Java虚拟机内存压力（降低虚拟机内存出现OOM）；
辅助回收Native内存（NativeAllocationRegistry）。
NativeAllocationRegistry是一种辅助自动回收native内存的一种机制，当Java对象被GC后，NativeAllocationRegistry可以辅助回收Java对象所申请的Native内存
  
  
  
Bitmap#createBitmap -> Bitmap#nativeCreate -> Bitmap.cpp#Bitmap_creator

===================== 8.0之前 ====================
static jobject Bitmap_creator(...) {
    ...
    Bitmap* nativeBitmap = GraphicsJNI::allocateJavaPixelRef(env, &bitmap, NULL);
    ...
    if (jColors != NULL) {
        GraphicsJNI::SetPixels(env, jColors, offset, stride, 0, 0, width, height, bitmap);
    }
    return GraphicsJNI::createBitmap(env, nativeBitmap, getPremulBitmapCreateFlags(isMutable));
}

//Graphics.cpp
android::Bitmap* GraphicsJNI::allocateJavaPixelRef(JNIEnv* env, SkBitmap* bitmap, SkColorTable* ctable) {
    ...
    //创建一个Java的Byte数组
    jbyteArray arrayObj = (jbyteArray) env->CallObjectMethod(gVMRuntime, gVMRuntime_newNonMovableArray, gByte_class, size);
    ...
    jbyte* addr = (jbyte*) env->CallLongMethod(gVMRuntime, gVMRuntime_addressOf, arrayObj);
    ...
    //Byte数组放到Native层的Bitmap对象
    android::Bitmap* wrapper = new android::Bitmap(env, arrayObj, (void*) addr, info, rowBytes, ctable);
    ...
    return wrapper;
}

//Graphics.cpp
jobject GraphicsJNI::createBitmap(JNIEnv* env, android::Bitmap* bitmap, int bitmapCreateFlags, jbyteArray ninePatchChunk, jobject ninePatchInsets, int density) {
	...
	//创建一个Java的Bitmap对象，通过其构造函数，将Byte数组赋值给它
    jobject obj = env->NewObject(gBitmap_class, gBitmap_constructorMethodID,
            reinterpret_cast<jlong>(bitmap), bitmap->javaByteArray(),
            bitmap->width(), bitmap->height(), density, isMutable, isPremultiplied,
            ninePatchChunk, ninePatchInsets);
    ...
    return obj; //返回这个Bitmap对象给Java层
}


===================== 8.0之后 ====================
static jobject Bitmap_creator(...) {
    ...
    sk_sp<Bitmap> nativeBitmap = Bitmap::allocateHeapBitmap(&bitmap);
    ...
    if (jColors != NULL) {
        GraphicsJNI::SetPixels(env, jColors, offset, stride, 0, 0, width, height, bitmap);
    }
    return createBitmap(env, nativeBitmap.release(), getPremulBitmapCreateFlags(isMutable));
}

//(hwui包下)Bitmap.cpp
static sk_sp<Bitmap> allocateHeapBitmap(size_t size, const SkImageInfo& info, size_t rowBytes) {
    void* addr = calloc(size, 1); //在native堆内存中申请内存空间
    if (!addr) {
        return nullptr;
    }
    return sk_sp<Bitmap>(new Bitmap(addr, size, info, rowBytes));
}

jobject createBitmap(JNIEnv* env, Bitmap* bitmap, int bitmapCreateFlags, jbyteArray ninePatchChunk, jobject ninePatchInsets, int density) {
    ...
    BitmapWrapper* bitmapWrapper = new BitmapWrapper(bitmap);
    //创建一个Java的Bitmap对象，通过其构造函数，将Native层Bitmap对象地址赋值给它
    jobject obj = env->NewObject(gBitmap_class, gBitmap_constructorMethodID,
            reinterpret_cast<jlong>(bitmapWrapper), bitmap->width(), bitmap->height(), density,
            isMutable, isPremultiplied, ninePatchChunk, ninePatchInsets);
    ...
    return obj; //返回这个Bitmap对象给Java层
}
```



##### 说说说Bitmap占用的内存是怎么释放的？

```java
释放的方式有两种，一种是主动释放，另外一种是系统帮我们释放。
//主动回收：recycle()
8.0之前：在Java层直接置空像素数据（byte[]）
8.0之后：调用native方法，在native层直接释放像素数据
recycle()只会给我们清除掉像素数据，但是Bitmap对象（Java层 以及 Native层）还是交由GC帮忙回收。
官方建议：平常开发中无需调用recycle()来释放内存，GC会帮我们释放的。

//系统帮我们释放：7.0前（BitmapFinalizer）、7.0后（NativeAllocationRegistry）
注意：在查阅了源码之后发现7.0之后就用了NativeAllocationRegistry，7.0之前才是用户BitmapFinalizer。并不是网上说的8.0（llk）

7.0之前：Java层Bitmap对象、像素数据依靠GC释放；Native层Bitmap对象依靠Object#finalize析构方法释放。
	像素数据存放在Java层，所有Java层Bitmap对象被GC释放了，像素数据自然也会被释放了。

	下面看看Native层Bitmap对象的释放：
    Bitmap(...) {
        ...
        mNativePtr = nativeBitmap;
        mFinalizer = new BitmapFinalizer(nativeBitmap);
        int nativeAllocationByteCount = (buffer == null ? getByteCount() : 0);
        mFinalizer.setNativeAllocationByteCount(nativeAllocationByteCount);
    }

    private static class BitmapFinalizer {
        private long mNativeBitmap;
        private int mNativeAllocationByteCount;

        BitmapFinalizer(long nativeBitmap) {
            mNativeBitmap = nativeBitmap;
        }

        public void setNativeAllocationByteCount(int nativeByteCount) {
            if (mNativeAllocationByteCount != 0) {
                VMRuntime.getRuntime().registerNativeFree(mNativeAllocationByteCount);
            }
            mNativeAllocationByteCount = nativeByteCount;
            if (mNativeAllocationByteCount != 0) {
                VMRuntime.getRuntime().registerNativeAllocation(mNativeAllocationByteCount);
            }
        }

        @Override
        public void finalize() {
            try {
                super.finalize();
            } catch (Throwable t) {
                // Ignore
            } finally {
                // finalize 这里是 GC 回收该对象时会调用
                setNativeAllocationByteCount(0);
                nativeDestructor(mNativeBitmap);
                mNativeBitmap = 0;
            }
        }
    }

    private static native void nativeDestructor(long nativeBitmap);

FinalizerDaemon：析构守护线程。
对于重写了finalize方法的对象，它们在被GC回收时并不会马上被回收。
而是被放入到一个队列中，等待FinalizerDaemon守护线程去调用它们的成员函数finalize然后再被回收。
（如果对象实现了finalize函数，不仅会使其生命周期至少延长一个GC过程，而且也会延长其所引用到的对象的生命周期，从而给内存造成了不必要的压力）

为什么Bitmap对象不直接实现finalize()方法呢？
因为8.0之前像素数据是存储在Java对象中的，如果直接实现finalize()方法会导致bitmap对象被延时回收，造成内存压力。
所以才会让BitmapFinalizer来实现finalize()方法，当Bitmap对象变成GCroot不可达时，会触发回收BitmapFinalizer放到延时回收队列中，调用它的finalize。


7.0之后：NativeAllocationRegistry（https://juejin.cn/post/6894153239907237902）
NativeAllocationRegistry总结：
1. 当Native内存增长过多的时候自动触发GC（告诉GCNative申请的内存长度，检查是否到达GC阀值）
2. 当GC回收Java对象时同时回收Native占用的内存（Cleaner）

Bitmap(...) {
    ...
    mNativePtr = nativeBitmap; //Native层Bitmap对象地址
    long nativeSize = NATIVE_ALLOCATION_SIZE + getAllocationByteCount(); //step1
    NativeAllocationRegistry registry = new NativeAllocationRegistry(Bitmap.class.getClassLoader(), nativeGetNativeFinalizer(), nativeSize); //step2
    registry.registerNativeAllocation(this, nativeBitmap); //step3
    ...
}

step1. 获取Native层需要的内存大小 
long nativeSize = NATIVE_ALLOCATION_SIZE + getAllocationByteCount();

//NATIVE_ALLOCATION_SIZE + getAllocationByteCount()
//固定的32字节 + getAllocationByteCount()
private static final long NATIVE_ALLOCATION_SIZE = 32;
public final int getAllocationByteCount() {
    ...
    //Native方法，返回Native层Bitmap对象所占用的字节数
    return nativeGetAllocationByteCount(mNativePtr);
}


step2. 创建NativeAllocationRegistry对象
//NativeAllocationRegistry registry = new NativeAllocationRegistry(Bitmap.class.getClassLoader(), nativeGetNativeFinalizer(), nativeSize);

public NativeAllocationRegistry(ClassLoader classLoader, long freeFunction, long size) {
        ...
        this.classLoader = classLoader;
        this.freeFunction = freeFunction;
        this.size = size;
}

//classLoader：这个参数并没有使用，不知道传进来是干啥用的
//freeFunction：释放Native层Bitmap对象以及其内存的函数地址 nativeGetNativeFinalizer()
//nativeSize：Native层需要的内存大小

//Bitmap.java
private static native long nativeGetNativeFinalizer();
//Bitmap.cpp
{"nativeGetNativeFinalizer", "()J", (void*)Bitmap_getNativeFinalizer },

//将函数转成内存地址返回
static jlong Bitmap_getNativeFinalizer(JNIEnv*, jobject) { return static_cast<jlong>(reinterpret_cast<uintptr_t>(&Bitmap_destruct)); }

//释放Bitmap对象函数
static void Bitmap_destruct(BitmapWrapper* bitmap) { delete bitmap;}


step3. 登记目标对象，实现内存的自动回收
registry.registerNativeAllocation(this, nativeBitmap);

    public Runnable registerNativeAllocation(Object referent, long nativePtr) {
        ...
        CleanerThunk thunk; //这个就是执行释放函数的Runnable，给Cleaner内部用的
        CleanerRunner result; //这个Runnable是提供给外部用的，方便外部调Cleaner#clean
        try {
            thunk = new CleanerThunk();
            //利用Cleaner机制来回收（使用虚引用得知对象被GC的时机，在GC前执行额外的回收工作）
            Cleaner cleaner = Cleaner.create(referent, thunk);
            result = new CleanerRunner(cleaner);
            registerNativeAllocation(this.size); //告诉GC本次Native层申请了多少内存
        } catch (VirtualMachineError vme /* probably OutOfMemoryError */) {
            applyFreeFunction(freeFunction, nativePtr); //如果发生异常，直接调用释放函数
            throw vme;
        }
        thunk.setNativePtr(nativePtr);
        return result;
    }

    private class CleanerThunk implements Runnable {
        private long nativePtr;

        public CleanerThunk() {
            this.nativePtr = 0;
        }

        public void run() {
            if (nativePtr != 0) {
                applyFreeFunction(freeFunction, nativePtr);
                registerNativeFree(size); //告诉GC本次Native申请的内存已经被释放
            }
        }

        public void setNativePtr(long nativePtr) {
            this.nativePtr = nativePtr;
        }
    }

    private static class CleanerRunner implements Runnable {
        private final Cleaner cleaner;

        public CleanerRunner(Cleaner cleaner) {
            this.cleaner = cleaner;
        }

        public void run() {
            cleaner.clean();
        }
    }

    //告诉GC本次Native申请的内存大小。检测下是否达到GC的触发条件。
    private static void registerNativeAllocation(long size) {
        VMRuntime.getRuntime().registerNativeAllocation((int)Math.min(size, Integer.MAX_VALUE));
    }

    //告诉GC本次Native申请的内存已经被释放
    private static void registerNativeFree(long size) {
        VMRuntime.getRuntime().registerNativeFree((int)Math.min(size, Integer.MAX_VALUE));
    }

    public static native void applyFreeFunction(long freeFunction, long nativePtr);

    //libcore_util_NativeAllocationRegistry.cpp
    static void NativeAllocationRegistry_applyFreeFunction(JNIEnv*, jclass, jlong freeFunction, jlong ptr) {
        //将内存地址强转为对象指针
        void* nativePtr = reinterpret_cast<void*>(static_cast<uintptr_t>(ptr));
        //将Bitmap_destruct函数地址强转为函数对象
        FreeFunction nativeFreeFunction
            = reinterpret_cast<FreeFunction>(static_cast<uintptr_t>(freeFunction));
        //调用该对象的Bitmap_destruct函数
        nativeFreeFunction(nativePtr);
    }

Cleaner：利用虚引用和ReferenceQueue实现对一个对象生命周期的监（https://juejin.cn/post/6891918738846105614#heading-7）

Cleaner继承自PhantomReference<Object>。
在构建Cleaner的时候需要传入一个强引用对象以及一个Runnable。
强引用对象就是监听的目标，Runnable就是用来执行具体逻辑。

当对象被GC后，PhantomReference就会加入到ReferenceQueue中，并且唤醒ReferenceQueueDaemon线程。
ReferenceQueueDaemon线程就会去遍历执行ReferenceQueue中的PhantomReference。如果是Cleaner，
就会执行其clean()方法（因此要保证Cleaner#clean方法中做的事情是快速的，防止阻塞其他Cleaner的清理动作）。

Cleaner#clean方法内就会执行Runnable#run，最终执行我们的释放逻辑。
```



##### 如何进行图⽚压缩？+2

```java
色深："色彩的深度"，表示一个像素点用多少bit来存储色值。（数字图像参数）
位深：跟硬件相关的对象一律使用“位深”来描述（物理硬件参数）
例如：某张图片100*100 色深32位(ARGB_8888)，保存时位深度为24位
加载到内存中占大小：100 * 100 * (32 / 8)
存储到文件中占大小：100 * 100 * (24 / 8) * 压缩率

压缩方法有两种：1、质量压缩；2、尺寸压缩。
质量压缩：保持图片宽高以及占用内存不变的情况下，改变Bitmap的质量。（https://cloud.tencent.com/developer/article/1006307）
ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
//format：压缩格式，JPEG（有损）、PNG（无损，quality参数会不起作用）、WEBP（有损或无损，有损比JPEG更加省空间）
//quality：为0～100，0表示最小体积，100表示最高质量
bitmap.compress(Bitmap.CompressFormat.JPEG, quality, outputStream);

实现原理：
Java层函数 -> Native函数 -> Skia函数 -> 对应第三库函数（例如libjpeg）
（Skia在Android中提供了基本的画图和简单的编解码功能。可以挂接其他的第三方编码解码库或者硬件编解码库，例如libpng、libjpeg、libgif 等。bitmap.compress(Bitmap.CompressFormat.JPEG...)，实际会调用libjpeg.so动态库进行编码压缩）


哈夫曼算法：
文件中可能会出现五个值a,b,c,d,e，其二进制为
a. 1010
b. 1011
c. 1100
d. 1101
e. 1110
定长算法下最优（最前面的一位数字是 1，其实是浪费掉了）
a. 010
b. 011
c. 100
d. 101
e. 110

哈夫曼算法对定长算法进行了改进。给信息赋予权重，即为信息加权。
假设 a 占据了 60%，b 占据了 20%， c 占据了 20%，d,e 都是 0%
a:010  (60%)
b:011  (20%)
c:100  (20%)
d:101  (0%)
e:110  (0%)
在这种情况下，我们可以使用哈夫曼树算法再次优化为：
a:1
b:01
c:00
就是出现频率高的字母使用短码，对出现频率低的使用长码，不出现的直接就去掉。最后abcde的哈夫曼编码就对应：1 01 00

所以这个算法一个很重要的思路是必须知道每一个元素出现的权重，能够知道每一个元素的权重那么就能够根据权重动态生成一个最优的哈夫曼表。
怎么去获取每一个元素，对于图片就是每一个像素中argb的权重。只能去循环整个图片的像素信息，这无疑是非常消耗性能的。

Android在使用libjpeg，压缩默认使用的是 默认哈夫曼表，而不是 图像数据计算哈弗曼表。
//标志位
optimize_coding=FALSE //使用默认哈夫曼表
optimize_coding=TRUE  //使用图像数据计算哈弗曼表（比默认哈夫曼表体积缩小2倍）
Google在初期考虑到手机的性能瓶颈，计算图片权重这个阶段非常占用CPU资源的同时也非常耗时，因为此时需要计算图片所有像素 argb 的权重，这也是 Android 的图片压缩率对比 iOS 来说差了一些的原因之一。

从Android7.0 版本开始，optimize_code标示已经设置为了TRUE，也就是默认使用图像生成哈夫曼表。



尺寸压缩：改变Bitmap的宽高以及占用内存（https://cloud.tencent.com/developer/article/1006352?from=article.detail.1006307）
也叫重采样，放大图像称为上采样（upsamping），缩小图像称为下采样（downsampling）。主要讨论缩小图像。
Android中图片重采样提供了两种方法，一种是邻近采样（Nearest Neighbour Resampling），另一种是双线性采样（Bilinear Resampling）

邻近采样：inSampleSize（inDensity搭配inTargetDensity使用，效果跟inSampleSize是一样的）
BitmapFactory.Options options = new BitmapFactory.Options();
//设置取样大小。它的作用是：设置int类型后，假如设为4。则宽和高都为原来的1/4，宽高都减少了自然内存也降低了。（google推荐用2的倍数）
options.inSampleSize = 2; 
Bitmap compress = BitmapFactory.decodeFile("xxx.png", options);

邻近采样采用邻近点插值算法
邻近采样比较粗暴，采样率设置为2。那么就是两个像素选一个留下，另一个直接抛弃。
例如：每个像素红绿相间的图片。邻近采样为2的时候，图片直接变成绿色了。


双线性采样：Matrix
//方式一
Bitmap bitmap = BitmapFactory.decodeFile("xxx.png");
//双线性采样使用的是双线性內插值算法，参考源像素相应位置周围2x2个点的值，根据相对位置取对应的权重，经过计算之后得到目标图像。
//双线性内插值算法在图像的缩放处理中具有抗锯齿功能, 是最简单和常见的图像缩放算法.
Matrix matrix = new Matrix();
matrix.setScale(0.5f, 0.5f);
bm = Bitmap.createBitmap(bitmap, 0, 0, bit.getWidth(), bit.getHeight(), matrix, true);

//方式二，最终也是走方式一的
Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/test.png");
Bitmap compress = Bitmap.createScaledBitmap(bitmap, bitmap.getWidth()/2, bitmap.getHeight()/2, true);

双线性采样 采用双线性内插值算法
参考了源像素相应位置周围2x2个点的值，根据相对位置取对应的权重，经过计算之后得到目标图像。
双线性内插值算法在图像的缩放处理中具有抗锯齿功能，是最简单和常见的图像缩放算法。
当对相邻2x2个像素点采用双线性內插值算法时，所得表面在邻域处是吻合的，但斜率不吻合，并且双线性内插值算法的平滑作用可能使得图像的细节产生退化，这种现象在上采样时尤其明显。

两个采样算法对比
邻近采样：最快的，效率最高；产生比较明显的锯齿（尤其是文字内容多的图片）；
双线性采样：抗锯齿。


双立方／双三次采样（Bicubic Resampling）
采用的是双立方／双三次插值算法
双立方／双三次插值算法更进一步参考了源像素某点周围4x4个像素。
Android中并没有原生支持，可以通过ffmpeg来支持，具体的实现在 libswscale/swscale.c 文件中：FFmpeg Scaler Documentation。
双立方/双三次插值算法经常用于图像或者视频的缩放，它能比双线性内插值算法保留更好的细节质量。
```



##### getByteCount() & getAllocationByteCount()的区别？

```java
在不复用Bitmap时，getByteCount（API12加入）和getAllocationByteCount（API19加入）返回的结果是一样的，都是Bitmap占用的内存大小。

在通过复用Bitmap来解码图片时：
getByteCount：图片像素占用的大小
getAllocationByteCount：Bitmap对象占用的大小
  
例如：某Bitmap内存占用20。Bitmap对象进行复用，放入一张小图占用了8。
那么getByteCount为8，getAllocationByteCount为20
```



##### 如何计算⼀张图⽚在内存中占⽤的⼤⼩？+2

```java
1、BitmapFactory#decodeResource（加载资源中的图片）
w * h * 一像素占的内存 * (设备dpi / 图片所在drawable目录对应的dpi)
  
2、BitmapFactory#decodeFile（加载图片文件）
w * h * 一像素占的内存
  
一像素占的内存大小：Bitmap.Config用来描述图片的像素是怎么被存储
ARGB_8888: 每个像素4字节. 共32位，默认设置。
Alpha_8: 只保存透明度，共8位，1字节。
ARGB_4444: 共16位，2字节。
RGB_565:共16位，2字节，只存储RGB值。
```



##### 说说LruCache & DiskLruCache原理 +2

```java
Lru：淘汰掉最近最少使用的元素。
LinkedHashMap实现，LinkedHashMap的构造函数里有个布尔参数accessOrder，当它为true时LinkedHashMap会以访问顺序为序排列元素，否则以插入顺序为序排序元素。

LruCache：内存缓存
int cacheMemorySize = (int)(Runtime.getRuntime().totalMemory() / 1024) / 8;
LruCache<String, Bitmap> lrucache = new LruCache<String, Bitmap>(cacheMemorySize) {

	//负责计算Bitmap大小。
    @Override
    protected int sizeOf(String key, Bitmap value) { 
        return getBitmapSize(value);
    }

    //key对应的缓存被删除后就会走这个方法
    @Override
    protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
        super.entryRemoved(evicted, key, oldValue, newValue);
    }

    //获取key对应的缓存时，如果已经被删掉了就会回调这个方法。是否需要再创建新的Bitmap
    @Override
    protected Bitmap create(String key) {
        return super.create(key);
    }
};

put(K key, V value)：
1、插入元素并相应增加当前缓存的容量。如果key已存有数据，将该数据移除并且回调entryRemoved。
2、如果缓存容量满了，开启循环 从表头删除数据，直到容量小于最大容量为止。被删掉的数据

get(K key)：
1、如果该数据存在直接返回。根据LinkedHashMap特性，将该数据移动到表尾.
2、如果取不到key对应的数据，就会调用create()方法，让开发者看需要来创建数据，如果create()有返回就会走类似put的流程。

remove(K key)：移除数据，并且减少对应的容量


DiskLruCache：磁盘缓存
//directory：存储缓存文件的目录；appVersion：应用版本号；valueCount：一个key对应的缓存文件的数目，如果我们传入的参数大于1，那么缓存文件后缀就是.0，.1等；maxSize：缓存容量上限。
DiskLruCache diskLruCache = DiskLruCache.open(directory, appVersion, valueCount, maxSize);

DiskLruCache.Editor editor = diskLruCache.edit(String.valueOf(System.currentTimeMillis()));
BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(editor.newOutputStream(0));
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.scenery);
bitmap.compress(Bitmap.CompressFormat.JPEG, 100, bufferedOutputStream);

editor.commit();
diskLruCache.flush();
diskLruCache.close();

原理：
//缓存目录中会有一个journal文件，用于记录磁盘缓存的文件描述信息 以及 操作记录
//map中记录的是文件的描述信息实体Entry，这些Entry与缓存目录中journal文件的记录会一一对应的
//当map发生变化了，会及时更新journal文件。以确保app下次启动读取回来的Entry是最新的
private final LinkedHashMap<String, Entry> lruEntries = new LinkedHashMap<String, Entry>(0, 0.75f, true);

DiskLruCache的实现两个部分：日志文件和具体的缓存文件。
每次对缓存存储的时候除了对缓存文件做相应的操作，还会在日志文件做相应的记录。
每条日志文件有四种情况：CLEAN（调用了DiskLruCache#edit()保存了缓存，并且调用了Edit#commit）、DIRTY（缓存正在编辑，调用DiskLruCache#edit函数）、REMOVE（缓存写入失败）、READ（读缓存）。
```



##### 如何去加载⼤图⽚、长图？

```java
BitmapRegionDecoder：区域解码器
https://blog.csdn.net/qq_21258529/article/details/90293388
```



## Android核心机制

### Context

##### 说说你对Context的了解。

```java
1、介绍Context
Context是个抽象类，实现类是ContextImpl，实现类由安卓系统提供。
有了Context之后就能访问应用特定的资源和类，并且还能发起一些应用层的调用（如启动Activity、发广播等）

2、说说Context的种类（继承Context的类）
应用中有三种不同的Context，分别是Application、Activity、Service
  
Application 继承自ContextWrapper（ContextWrapper继承自Context）
	Context创建过程：在Application创建过程中，会先创建一个ContextImpl对象，然后再创建Application对象，最后通过Application#attachBaseContext方法，将ContextImpl对象赋值给ContextWrapper#mBase
Activity 继承自ContextThemeWrapper，ContextThemeWrapper又继承自ContextWrapper
	ContextThemeWrapper：带了一些跟UI相关的变量
 	Context创建过程：在Activity创建过程中，先创建Activity对象，然后再创建一个ContextImpl对象，最后通过Activity#attachBaseContext方法，将ContextImpl对象赋值给ContextWrapper#mBase
Service 继承自ContextWrapper
	Context创建过程：跟Application、Activity过程差不多
  
//这里是一个典型的静态代理，ContextWrapper包装了一个Context对象（mBase），所有调用都委托给他了。
//mBase：就是ContextImpl对象
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }

    protected void attachBaseContext(Context base) {
        mBase = base;
    }

	public Context getBaseContext() {
        return mBase;
    }

    public Resources getResources() {
        return mBase.getResources();
    }

    ...
}
  
3、说明四大组件中广播接收者、内容提供者都不是Context。并且说明其内部的Context从哪里来。
广播接收者：onReceive的Context哪里来的？
动态注册：Context对象，就是注册接收者时候的Context对象（context.registerReceiver(xxxx)）
静态注册：Context对象，是以Application为mBase的一个ContextWrapper
	原理：
	ContextImpl context = application.getBaseContext();
	//ContextImpl#getReceiverRestrictedContext：new ReceiverRestrictedContext(getOuterContext())
	//ReceiverRestrictedContext也是继承ContextWrapper，那么我们只要知道构造函数中的 Context base 是谁，就知道mBase是谁了。
	//矛头指向getOuterContext方法，getOuterContext方法就是直接返回了一个mOuterContext对象。
	//mOuterContext：是在Context创建过程中，通过setOuterContext赋值的ContextImpl对象。也就是说上边用谁（Application、Activity、Service）的Context，这里的mOuterContext就是谁的了。
	receiver.onReceive(context.getReceiverRestrictedContext(), xxx);

    //ReceiverRestrictedContext也是继承ContextWrapper，那么我们只要知道构造函数中的 Context base 是谁，就知道mBase是谁了。
    //矛头只想 getOuterContext() 方法
	class ReceiverRestrictedContext extends ContextWrapper {
	    ReceiverRestrictedContext(Context base) {
	        super(base);
	    }
	}

内容提供者：成员变量mContext是哪里来的？
内容提供者创建的时候传入的Application Context对象。
内容提供者创建在Application#attachBaseContext之后就是为了获取Application Context对象。
```



##### 应用有多少个Context，不同Context有什么区别？

```java
Application个数（多进程）+ Activity个数 + Service个数。
  
Application 继承自ContextWrapper
	Context创建过程：在Application创建过程中，会先创建一个ContextImpl对象，然后再创建Application对象，最后通过Application#attachBaseContext方法，将ContextImpl对象赋值给ContextWrapper#mBase
Activity 继承自ContextThemeWrapper，ContextThemeWrapper又继承自ContextWrapper
	ContextThemeWrapper：带了一些跟UI相关的变量
 	Context创建过程：在Activity创建过程中，先创建Activity对象，然后再创建一个ContextImpl对象，最后通过Activity#attachBaseContext方法，将ContextImpl对象赋值给ContextWrapper#mBase
Service 继承自ContextWrapper
	Context创建过程：跟Application、Activity过程差不多
```



##### Activity this & getBaseContext 有什么区别？

```java
区别是两者返回对象不同：
this返回Activity对象自己（Activity也是继承Context的）。
getBaseContext返回的时候 mBase 这个Context对象。

两者使用上没什么区别：
Activity继承ContextWrapper，ContextWrapper实现的所有Context方法，都是委托给mBase的。
```



##### getApplication & getApplicationContext 有什么区别？

```java
getApplication 返回 Application对象
getApplicationContext 返回 Context对象
但是两者其实返回都是Application对象（Application也是继承自Context，可以强转的）

区别：
getApplicationContext 是Context抽象类的方法，哪里都可以用。
getApplication 是Activity、Service特有的方法。所有在广播接收者、内容提供者里边无法使用该方法。
```



## 性能优化

##### OOM可以被捕获吗？

```java
可以被捕获。但是不是万不得已最好别捕获它。

  
Java异常体系中所有异常都继承自Throwable，其中Throwable有两个直接子类Error和Exception。
Exception 一般指可以/应该捕获和处理的异常。
Error：一般指非正常状态的。比较严重的，不应该被捕获的系统错误。

如果你把捕获OOM当做处理OOM的一种手段，无疑是不合适的。
你无法保证你catch的代码就是导致OOM的原因，可能它只是压死骆驼的最后一根稻草，甚至你也无法保证你的catch代码块中不会再次触发OOM。

在你自己明确知道可能发生OOM的情况下设置一个兜底策略，这可能是捕获OOM的唯一意义了。
例如：View#buildDrawingCacheImpl（为View生成Bitmap缓存，如果OOM了就放弃生成）、NativeAllocationRegistry#registerNativeAllocation（在登记目标对象的时候，如果OOM了直接释放其native内存）

  
JVM中哪一块内存不会发生OutOfMemoryError？
Java虚拟机栈：每个方法被执行的时候，Java 虚拟机栈都会同步创建一个栈帧用于存储局部变量表、操作数栈、动态连接、方法出口等信息。每个方法被调用直到执行完毕的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。
如果线程请求的栈深度大于虚拟机所允许的深度，将抛出 StackOverflowError 异常。 如果 Java 虚拟机栈支持动态扩展，当栈扩展时无法申请到足够的内存会排抛出 OutOfMemoryError 异常。

本地方法栈：为虚拟机使用到的 Native 方法服务。《Java 虚拟机规范》对本地方法栈中方法使用的语言、使用方式和数据结构并没有任何强制规定，因此具体的虚拟机可以根据需要自由实现它。Hotspot 将本地方法栈和虚拟机栈合二为一。
本地方法栈也会在栈深度溢出和栈扩展失败时分别抛出 StackOverflowError 和 OutOfMemoryError 。

Java堆：所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，Java 世界里 “几乎” 所有的对象实例都在这里分配内存。在 《Java 虚拟机规范》中对 Java 堆的描述是：“所有的对象实例以及数组都应当在堆上分配”。

Java堆以处于物理上不连续的内存空间，但在逻辑上它应该被视为连续的。但对于大对象(典型的如数组对象)，多数虚拟机实现出于实现简单、存储高效的考虑，很可能会要求连续的内存空间。

Java堆既可以被实现成固定大小，也可以是扩展的。如果在 Java 堆中没有内存完成实例分配，并且堆无法再扩展时，Java 虚拟机将会抛出 OutOfMemoryError 。

方法区：方法区是各个线程共享的内存区域，它用于存储已被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据。
虽然《Java 虚拟机规范》中把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做“非堆”，目的是与 Java 堆分开来。
如果方法区无法满足新的内存分配的需求时，将抛出 OutOfMemoryError 。

运行时常量池：方法区的一部分。Class 文件的常量池表，用于存放编译期生成的各种字面量与符号引用，这部分内容将在类加载后方法方法去的运行时常量池。
运行时常量池具有动态性，运行期间也可以将新的常量放入池中，如 String.intern() 。
常量池受到方法区的限制，当无法再申请到内存时，会抛出 OutOfMemoryError 。

唯一一个在《Java虚拟机规范》中没有规定任何 OutOfMemoryError 情况的区域是 程序计数器。
程序计数器：是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是本地（Native）方法，这个计数器值则应为空（Undefined）。
```



##### Anr监控、死锁监控（灵感源自系统Watchdog）

```java
系统Watchdog原理：
1、Watchdog一个单例类，也是一个线程。在SystemServer中会启动它；
2、它维护着一个HandlerChecker列表，而HandlerChecker里边又维护了一个Monitor列表；
3、HandlerChecker是用来检查Handler是否有消息阻塞，Monitor是用来检测线程是否有死锁；
4、Watchdog会有一个专门检测线程死锁的HandlerChecker（mMonitorChecker），也会加入到HandlerChecker列表里边；
5、Watchdog线程会每30s遍历一次HandlerChecker列表发送检查事件。然后统计是否有检查未完成状态的HandlerChecker；
6、如果出现未检查完成的的HandlerChecker，超过60s之后就会dump堆栈日志以及重启SystemServer；
7、当Handler执行HandlerChecker的事件之后就认为检查完成，然后HandlerChecker就会对Monitor列表进行死锁检查；
8、如果出现死锁，那么HandlerChecker的Handler所在的线程就会阻塞。下一次检查就会无法完成走步骤6的逻辑。

源码分析：
class Watchdog extends Thread
    关键属性：
    ArrayList<HandlerChecker> mHandlerCheckers //HandlerChecker列表
    HandlerChecker mMonitorChecker; //在构造函数里边添加到HandlerChecker列表，这个Checker专门用来监听线程死锁

    //检查状态
    static final int COMPLETED = 0;   //检查完成
    static final int WAITING = 1;     //检查未完成，等待中（<30s）
    static final int WAITED_HALF = 2; //检查未完成，等待中（30s<time<60s）
    static final int OVERDUE = 3;     //检查未完成，不能忍了要炸了（>60s）

    关键方法：
    addThread(Handler thread) //根据Handler创建一个HandlerChecker，并且添加到mHandlerCheckers。
    addMonitor(Monitor monitor) //将Monitor添加到mMonitorChecker里边。
    run()方法：
        1、每30s执行发起一次检查，遍历HandlerChecker列表对每个HandlerChecker发起检查（HandlerChecker#scheduleCheckLocked）。
        2、然后检查所有HandlerChecker的状态（HandlerChecker#getCompletionStateLocked），看看是不是有处于检查未完成的HandlerChecker
        3、如果有未完成的HandlerChecker，根据状态做出对应的操作
            WAITED_HALF：dump这个HandlerChecker对应线程的堆栈日志
            OVERDUE：找出出问题的HandlerChecker，dump堆栈日志、eventlog、dropbox log。最后重启SystemServer。

     
class HandlerChecker implements Runnable
    Handler mHandler; //线程的Handler
    ArrayList<Monitor> mMonitors
    private boolean mCompleted; //检查是否已经完成，构造函数中会设为true
    private long mStartTime; //检查开始时间
    private Monitor mCurrentMonitor; //当前执行的Monitor（如果出现死锁，该属性就会有值）

    scheduleCheckLocked方法解析：发起检查
        public void scheduleCheckLocked() {
            //没有锁需要监听 同时 消息队列没有消息在休眠中，无需做检查
            if (mMonitors.size() == 0 && mHandler.getLooper().getQueue().isPolling()) {
                mCompleted = true;
                return;
            }
            //上一个检查还没结束
            if (!mCompleted) {
                return;
            }

            mCompleted = false;
            mCurrentMonitor = null;
            mStartTime = SystemClock.uptimeMillis(); //记录一下检查开始时间
            mHandler.postAtFrontOfQueue(this); //往消息队列发送消息（插入消息队列头部）
        }

    run方法解析：因为postRunnable，所有如果Handler执行消息会走到这里
    public void run() {
            final int size = mMonitors.size();
            for (int i = 0 ; i < size ; i++) { //遍历所有Monitor
                synchronized (Watchdog.this) {
                    //记录当前的Monitor
                    //如果出现阻塞超时，就会通过mCurrentMonitor，打印其实现类。从而知道哪个锁出现死锁

                    mCurrentMonitor = mMonitors.get(i); 
                }
                //Monitor接口的实现方法中，一般就是获取锁操作。如果一直获取不到锁，就会一直卡着（出现死锁）
                mCurrentMonitor.monitor();
            }

            synchronized (Watchdog.this) {
                mCompleted = true;
                mCurrentMonitor = null;
            }
        }


    getCompletionStateLocked方法解析：获取检查状态
        public int getCompletionStateLocked() {
            if (mCompleted) {
                return COMPLETED;
            } else { //没有检查完成
                long latency = SystemClock.uptimeMillis() - mStartTime;
                //mWaitMax 最大等待时长（默认60s）
                if (latency < mWaitMax/2) { //检查超时，但在容忍范围内
                    return WAITING;
                } else if (latency < mWaitMax) { //检查超时，已经超过一半的容忍范围了
                    return WAITED_HALF;
                }
            }
            return OVERDUE; //检查超时，已经无法容忍了
        }


interface Monitor {
    void monitor();
}


AMS中的使用
AMS构造函数中：
    Watchdog.getInstance().addMonitor(this);（AMS实现了Monitor接口）
    Watchdog.getInstance().addThread(mHandler);
AMS#monitor()：
    public void monitor() {
        synchronized (this) { } //获取一下AMS对象锁，看看能不能获取到
    }
```







1. 内存泄漏是如果产生的？如何解决？+3
2. 内存泄漏与内存抖动的区别。+3
3. 怎么app优化启动速度。
4. 如何监测内存泄漏。
5. 什么是ANR，如何避免它？
6. 如何进行app性能优化、内存优化、cpu使用率优化？
7. 内存泄漏的分类。如何分析内存泄漏问题。
8. native崩溃日志如何采集，怎么处理？



## 其他

### 进程优先级 - ADJ

```java
adj级别：进程优先级
NATIVE_ADJ				-1000		native进程（init进程fork出来的native进程）
SYSTEM_ADJ				-900		仅指system_server进程
PERSISTENT_PROC_ADJ		-800		系统进程（系统签名的应用，系统进程一般不会被杀即便被杀或者发生Crash系统会立即重新拉起）
PERSISTENT_SERVICE_ADJ	-700		关联着系统或关联persistent进程
FOREGROUND_APP_ADJ		0			前台进程（获得焦点）
VISIBLE_APP_ADJ			100			可见进程（失去焦点，但还可见）
PERCEPTIBLE_APP_ADJ		200			可感知进程（有前台service在运行，比如后台播放音乐）
BACKUP_APP_ADJ			300			备份进程
HEAVY_WEIGHT_APP_ADJ	400			重量级进程
SERVICE_ADJ				500			服务进程
HOME_APP_ADJ			600			Home进程（launcher）
PREVIOUS_APP_ADJ		700			上一个进程（在后台并且在最近任务栈顶的进程）
SERVICE_B_ADJ			800			B List中的Service
CACHED_APP_MIN_ADJ		900			不可见进程的adj最小值
CACHED_APP_MAX_ADJ		906			不可见进程的adj最大值

测试：
app在前台->0
app在前台，带个前台服务->0
app在后台->700（最近使用的进程）
app在后台，带个前台服务->50（0-100之间，可见进程）
app在后台，点开好几个app->900（危，第一个杀的就是你）

四大组件状态改变时会同步更新相应进程的adj优先级。当同进程有多个决定其优先级的组件时，取优先级最高的adj作为最终的adj。


LowMemoryKiller：防止物理内存剩余过低，当物理内存剩余到达阈值，就会根据adj将进程回收。
Memory		adj
332mb 	->  900+
221mb 	->  900
129mb 	->  300
110mb 	->  200
92 mb 	->  100
73 mb 	->  0
不用设备阈值可能不一样，当adj相同的情况下占内存大的进程会被优先回收。

跟adj相关的建议：
1、UI进程与Service进程一定要分离，因为对于包含activity的service进程，一旦进入后台就成为”cch-started-ui-services”类型的cache进程(ADJ>=900)，随时可能会被系统回收；而分离后的Service进程服务属于SERVICE_ADJ(500)，被杀的可能性相对较小。尤其是系统允许自启动的服务进程必须做UI分离，避免消耗系统较大内存。
2、只有真正需要用户可感知的应用，才调用startForegroundService()方法来启动前台服务，此时ADJ=PERCEPTIBLE_APP_ADJ(200)，常驻内存，并且会在通知栏常驻通知提醒用户，比如音乐播放，地图导航。切勿为了常驻而滥用前台服务，这会严重影响用户体验。
3、进程中的Service工作完成后，务必主动调用stopService或stopSelf来停止服务，避免占据内存，浪费系统资源；
4、不要长时间绑定其他进程的service或者provider，每次使用完成后应立刻释放，避免其他进程常驻于内存；
5、APP应该实现接口onTrimMemory()和onLowMemory()，根据TrimLevel适当地将非必须内存在回调方法中加以释放。当系统内存紧张时会回调该接口，减少系统卡顿与杀进程频次。
6、减少在保活上花心思，更应该在优化内存上下功夫，因为在相同ADJ级别的情况下，系统会选择优先杀内存占用的进程。


系统内存不足：系统内存直接就是物理内存了，当物理内存不足LMK根据进程adj等级来回收，如果严重不足可能连前台进程都杀。
    在内核层直接查杀进程，不会在Framework层还跟你叨逼叨看回收哪个Activity。
应用内存不足：使用的虚拟内存大于总的四分之三，并且如果如果此进程ActivityTask数>=3，会对不可见Task进行回收，每次回收1个Task。
ActivityThread#main -> ActivityThread#attach -> BinderInternal#addGcWatcher

//监听每次GC，GC完后都会走一次这Runnable
BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    ...
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) { //已用内存 > 3/4最大内存
                        ...
                        try {
                            //如果此进程ActivityTask数>=3，会对不可见Task进行回收，每次回收1个Task。
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) { throw e.rethrowFromSystemServer(); }
                    }
                }
            });
```



### 数据存储

1. SharedPreference能多进程访问吗？进程间数据共享有什么方式？
2. SharedPreference是如何存储的？存储位置在哪里？
3. 内部存储与外部存储的区别？
4. android 有哪些数据存储方式。+2



### 功能设计相关

1. 如何设计一个类似微信朋友圈首页功能，包括UI、数据等方面。
2. 如何设计一个无线数据的气泡显示聊天内容。
3. 大文件在传输过程中需要考虑哪些问题，如果保证一致性。
4. 如何做到单个信号源，多个页面响应。
5. 设计一个日志系统。

### 其他

1. 怎么终止一个app?
2. 说说进程如何保活。+2
3. 说说屏幕适配方案。+2
4. 说说android中进程优先级。
5. AsyncTask是什么，使用方法，使用时候需要注意什么？
6. 说说AsyncTask的原理，聊聊它的缺陷和问题。
7. 聊聊android各个版本的新特性。
8. android为每个应用程序分配的内存是多少。
9. JSbridge是如何实现js与native联通的。
