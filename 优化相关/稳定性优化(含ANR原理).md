稳定性优化包括：Java crash；Native crash；ANR；业务逻辑的正常运行。



## Java crash

```java
① bugly：crash收集以及上报（release环境记得上传符号表）
		优点：统计信息多而且详细
		缺点：经常出现崩溃堆栈不完整的情况
② rxjava全局捕获：捕获在rxjava调用链中发生的异常（设置阀门，超过阀门值自动上报）
		使用场景：可以进行集中收集，必要情况下可以保存本地上报到后台
		debug环境：开启RxJavaExtensions库，可以打印出完整的堆栈

③ 接管Looper：捕获主线程中出现的carsh（黑科技）（设置阀门，超过阀门值自动上报）
		如果某些场景在工作时的崩溃，这些场景不影响正常流程的话，可以在接管的Looper中捕获，提高App稳定性。 
		handler.post(() -> {
            while (true){
                try{
                    Looper.loop(); //接管Looper
                }catch (Exception e){
                    //do something 可收集崩溃信息

                    //忽略某些场景出现的崩溃
                    String stack = Log.getStackTraceString(e);
                    if (stack.contains("某View") || ...){
                        ...
                    }else{
                        throw e;
                    }
                }
            }
        });
```



## Native crash

```java
① ndk工具 - objdump：输出so的函数地址，根据崩溃信息中的地址找到对应出错的地方。根据arm的brk指令，定位问题原因。
② google breakpad：收集native crash日志。
```



## ANR

### ANR产生原因

#### 系统资源不足

```java
系统资源不足的时候（cpu资源不足，内存不足，binder线程不足等）

1. 受整机负载影响（看logcat信息）
	① Load: 31.7 / 33.43 / 30.98（前1，前5，前15分钟的平均负载，我们通常看变化趋势）
	② 1.7% 3589/com.xxx.xxx: 1% user + 0.6% kernel / faults: 3365 minor 5 major（自己进程cpu使用）
	③ 85% TOTAL: 42% user + 33% kernel + 0% iowait + 0% softirq（看整体的负载情况）
     自己进程的cpu使用率正常，但是系统整体负载很高
       
2. 受低内存影响（内存紧张的时候，kswapd线程会活跃起来进行回收内存）
	① kswapd的CPU占据排进top3的话，这个问题受系统低内存影响比较大。
	② 低内存往往会伴随着IO升高，内存回收线程如kswapd、HeapTaskDaemon会变得活跃。
		低内存时系统往往会竭尽可能的回收内存，可能触发的fast path 回收 \ kswapd 回收 \ direct reclaim 回收 \ LMK杀进程回收等行为，都会造成do_page_fault 高频发生。
	③  另外当出现大量的 IO 操作的时候，应用主线程的 Uninterruptible Sleep 也会变多，此时涉及到 io 操作（比如 view，读文件，读配置文件、读 odex 文件），都会触发 Uninterruptible Sleep ，导致整个操作的时间变长，这就解释了为何低内存下为何应用响应卡顿。
```



#### 生命周期耗时

```java
组件生命周期内做了耗时操作，触发了AMS的定时炸弹。
  
ANS通知app走每一个生命周期之前都会安放定时炸弹，app走完生命周期就会通知ANS，ANS就会拆炸弹。如果定时炸弹超时了就会anr。
```



#### Input事件处理耗时

```java
触摸事件周期内做了耗时操作，触发了IuputDispather的检测机制。

① 如果在dispatchTouchEvent, onTouchEvent等处于consumeEvents调用链中执行耗时操作，即使未达到5s，也可能因为事件累计导致ANR。导致ANR的条件是在有PendingEvent时，WaitQueue中累积事件的总处理时长大于5s。
② 如果在ButtonClicked等非consumeEvents调用链中执行耗时操作，一定是大于5s（且有PendingEvent）才会触发ANR。
```



##### Input原理

```java
Input原理：https://juejin.cn/post/6956500920108580878#heading-1

system_server在启动时会启动InputManagerService，InputManagerService会创建InputReader、InputDispatcher对象。
并且开启这两个对象的native轮询线程。

InputReader：负责获取驱动中的触摸事件、传给InputDispatcher。
	它利用inotify和epoll机制监听/dev/input目录下的input设备事件节点，通过EventHub的getEvents接口获取到事件。

InputDispatcher：负责寻找目标窗口、发送事件给目标窗口
	1、寻找目标窗口
	系统屏幕上同一时间会存在多个可见窗口，如状态栏、导航栏、应用窗口、应用界面Dialog弹框窗口等。
	查看命令：adb shell dumpsys SurfaceFlinger
	寻找函数：InputDispatcher::findTouchedWindowTargetsLocked
	寻找逻辑：根据读取到的事件所属的屏幕displayid、x、y坐标位置等属性，遍历从mWindowHandlesByDisplay中匹配找到目标窗口
	mWindowHandlesByDisplay集合：InputDispatcher#setInputWindows 赋值而来的（SurfaceFlinger会在每一帧合成任务SurfaceFlinger::OnMessageInvalidate中搜集每个可见窗口Layer的信息通过，Binder调用InputDispatcher::setInputWindows同步可见窗口信息给InputDispatcher（android11之前是由WindowManagerService传给InputDispatcher的））

	2、发送事件给目标窗口
	2.1 双方通过什么方式通讯？
	InputDispatcher使用InputChannel封装的Socket（InputChannel内携带这socket fd）发送事件给目标应用窗口所属的应用进程。

	2.2 应用进程是怎么获得socket fd的？（InputDispatcher的InputChannel注册与监听）
	当应用界面启动的时候（ViewRootImpl#setView）添加窗口到WMS的过程中，会new一个空的InputChanel并且传给WMS（addToDisplayAsUser），让WMS为其赋值。
	在WMS addToDisplayAsUser的过程中，会创建一对InputChanel，client端赋值给那个空的InputChanel丢回给应用进程，service端的注册到InputDispatcher（InputDispatcher会缓存到一个map中，key为窗口token，value为InputChanel）。

	自此双方各持有了fd，各种会将fd加入到自己的looper里边进行监听（通过epoll进行监听）

	2.3 InputDispatcher发送事件
	寻找目标窗口，判断窗口是否就绪，如果已就绪
	通过窗口token获取到对应窗口的InputChanel。通过InputChanel内的socket fd写入事件

	2.4 应用进程接收事件
	应用进程的主线程的looper进行监听，且监听事件为ALOOPER_EVENT_INPUT，并在接收到事件之后会触发回调NativeInputEventReceiver::handleEvent函数，然后反调用java层方法。
	ViewRootImpl::WindowInputDispatcherReceiver::onInputEvent -> ViewRootImpl#enqueueInputEvent -> ViewPostImeInputState#deliver -> DecorView ... 责任链 ...（消费事件）-> ViewPostImeInputState#finishInputEvent（最终通过自己的InputChannel通知 InputDispatcher 事件已经处理完成）



ViewRootImpl 事件分发的过程
setView -> new WindowInputEventReceiver -> new InputEventReceiver -> new NativeInputEventReceiver

有事件来的时候JNI调用到 WindowInputEventReceiver#dispatchInputEvent
WindowInputEventReceiver并没有实现这个方法，因此会调用父类InputEventReceiver#dispatchInputEvent
内部会真正调用到 WindowInputEventReceiver#onInputEvent

-> doProcessInputEvents -> deliverInputEvent -> InputStage#deliver（这里的InputStage是 ViewPostImeInputStage ）-> apply(q, onProcess(q)) （先执行onProcess，再执行apply）-> ViewPostImeInputStage#onProcess -> ViewPostImeInputStage#processPointerEvent执行DecorView#dispatchPointerEvent）

ViewPostImeInputStage#processPointerEvent分析：
		private int processPointerEvent(QueuedInputEvent q) {
            final MotionEvent event = (MotionEvent)q.mEvent;
            ...
            boolean handled = mView.dispatchPointerEvent(event);
            ...
            return handled ? FINISH_HANDLED : FORWARD;
        }

ViewRootImpl 处理完事件后，通知InputDispatcher
apply(q, onProcess(q) 不管返回什么result，最终都是走到finishInputEvent) -> ViewPostImeInputStage#apply（InputStage#apply）-> InputStage#finish -> ... -> InputStage#onDeliverToNext -> InputStage#finishInputEvent -> QueuedInputEvent#InputEventReceiverfinishInputEvent -> ... 通知InputDispatcher

	
	
Input查看命令：adb shell dumpsys input
只关注 Input Dispatcher State:
  FocusedApplications: //获取焦点的进程
    displayId=0, name='AppWindowToken{a781bdd token=Token{82383b4 ActivityRecord{174d887 u0 aaa.llk.trace/com.llk.trace.MainActivity t868}}}', dispatchingTimeout=5000.000ms
  FocusedWindows: //获取焦点的窗口
    displayId=0, name='Window{d711981 u0 aaa.llk.trace/com.llk.trace.MainActivity}'
  PendingEvent:  //PendingEvent
    MotionEvent, age=2971.4ms
  InboundQueue: length=23 //从InputReader获取到的事件
    MotionEvent, age=2907.3ms
    MotionEvent, age=2654.5ms
    ...
  Connections: //已注册的InputChannel
    0: ...
    1: ...
    ...
  	12: channelName='d711981 aaa.llk.trace/com.llk.trace.MainActivity (server)', windowName='d711981 aaa.llk.trace/com.llk.trace.MainActivity (server)', status=NORMAL, monitor=false, inputPublisherBlocked=false
      OutboundQueue: <empty> //等待发送的应用的事件
      WaitQueue: length=20   //已发送给应用的事件
        MotionEvent, targetFlags=0x00000105, resolvedAction=0, age=3478.7ms, wait=3476.1ms
        MotionEvent, targetFlags=0x00000105, resolvedAction=2, age=3475.3ms, wait=3474.2ms
        ...
```



##### Input事件产生ANR原因

```java
Input ANR分析：https://zhuanlan.zhihu.com/p/53331495?from_voters_page=true

InputDispatcher涉及到3个队列 以及 一个变量：
		InboundQueue：存放从InputReader发送过来的事件
		OutboundQueue：存放即将发送给目标app的事件
		WaitQueue：已经发送给app的事件，等待app处理完成通知移除事件
		PendingEvent变量：从InboundQueue队头拿到事件赋值给自己，如果窗口就绪会将事件丢到OutboundQueue。

InputDispatcher轮询逻辑：
1、获取要处理的事件
如果PendingEvent为空
	从InboundQueue队头取出事件，赋值给PendingEvent
如果PendingEvent不为空
	继续拿PendingEvent手上的事件执行

2、检查窗口是否就绪
checkWindowReadyForMoreInputLocked：!connection->waitQueue.isEmpty() && currentTime >= connection->waitQueue.head->deliveryTime + STREAM_AHEAD_EVENT_TIMEOUT
检查窗口就绪逻辑：(WaitQueue不为空 && 当前时间 >= waitQueue队头事件时间 + 500ms) ---> 未就绪（证明目标进程有段事件没回应了）

如果窗口就绪
	将PendingEvent的事件加入到OutboundQueue
	发送给目标窗口对应的应用进程，并且将事件加入WaitQueue。
	进入下一次轮询
如果窗口未就绪
	当前事件是否是Pending（是不是上一轮还处理掉，还留在PendingEvent里的事件）
	如果不是，设置pending超时时长（当前时间+5s）
	如果是，判断其pending超时时长，看看是否已经超时（当前时间>pending超时时长，就是超时）
		如果超时：触发anr
		如果未超时：进入下一次轮询
	进入下一次轮询

3、当应用进程处理完事件，会通知InputDispatcher，从WaitQueue移除对应的事件


实验分析：

实验1 dispatchTouchEvent sleep10000s
非点击事件：在执行完dispatchTouchEvent后，才会通知WaitQueue移除。

第一次点击：事件发送成功，进入WaitQueue，等待app回应。（app阻塞了不会有回应的，但是如果等到5s后也是不会发生anr的，因为InputDispatcher没有新事件进来，没有继续工作了）
第二次点击：PendingEvent拿到down事件， InboundQueue留下up事件。但是发现目标进程并未就绪，给自己设置5s超时之后继续进入轮询。直到轮询5s后，发现不对劲就抛出anr


实验2 dispatchTouchEvent sleep10000s

点击事件触发anr（onClick里边sleep），第三次点击5s后才会触发anr？？？
点击事件：它是在dispatchTouchEvent执行完成后才丢到主线程执行的。也就是当进入onClick时，第一次点击的down/up事件已经执行完成，并从WaitQueue移除了。

第一次点击：移除WaitQueue成功（但其实app已经进入阻塞了）
第二次点击：事件发送成功，进入WaitQueue，等待app回应。（app阻塞了不会有回应的，但是如果等到5s后也是不会发生anr的，因为InputDispatcher没有新事件进来，没有继续工作了）
第三次点击：取出事件到PendingEvent，但是发现目标进程并未就绪，给自己设置5s超时之后继续进入轮询。直到轮询5s后，发现不对劲就抛出anr

实验...参考文章

总结：
dispatchTouchEvent, onTouchEvent等处于非点击事件的调用链中，执行耗时操作即使未达到5s，也可能因为事件累计导致ANR。
导致ANR的条件是在有PendingEvent时，WaitQueue中累积事件的总处理时长大于5s。
如果是点击事件中，执行耗时操作一定是大于5s（且有PendingEvent）才会触发ANR。
```





### ANR分析

相关文章：

https://cdn.modb.pro/db/68635
https://cloud.tencent.com/developer/article/1969659
https://www.jianshu.com/p/18f16aba79dd



#### logcat分析

```java
查看logcat日志（logcat可以分析大概是什么原因导致的ANR）
	ANR in aaa.llk.trace (aaa.llk.trace/com.llk.trace.MainActivity)
    PID: 24791  //发生的进程pid
    Reason: Input dispatching timed out...   //产生原因
    Parent: aaa.llk.trace/com.llk.trace.MainActivity  //ANR产生时的焦点窗口
    Load: 0.71 / 0.68 / 0.66 //Cpu的负载（三个数字分别是1分钟、5分钟、15分钟内系统的平均负荷）
    CPU usage from 21205ms to 0ms ago (2022-07-19 17:13:06.985 to 2022-07-19 17:13:28.190): //负载信息抓取在ANR发生之前的21205ms-0ms。同时指明ANR的时间点：17:13:06 - 17:13:28

      //各个进程cpu占用情况
      14% 894/surfaceflinger: 6.9% user + 8% kernel / faults: 299 minor 6 major
      8.8% 1864/com.android.systemui: 5.5% user + 3.3% kernel / faults: 1184 minor 5 major
      7.9% 1376/system_server: 4.1% user + 3.7% kernel / faults: 6509 minor 41 major
      3.3% 24366/com.google.android.googlequicksearchbox:search: 2.7% user + 0.5% kernel / faults: 19071 minor 273 major
      2.3% 1996/com.android.launcher3: 1.8% user + 0.4% kernel / faults: 9973 minor 76 major
      1.7% 12309/com.tencent.android.qqdownloader:tools: 1.5% user + 0.1% kernel / faults: 1706 minor 33 major
      1.3% 20846/com.tencent.mm: 0.7% user + 0.6% kernel / faults: 1936 minor 59 major

  当CPU完全空闲的时候，平均负荷为0；
  当CPU工作量饱和的时候，平均负荷为1，通过Load可以判断系统负荷是否过重。
  有一个形象的比喻：个CPU想象成一座大桥，桥上只有一根车道，所有车辆都必须从这根车道上通过，系统负荷为0，意味着大桥上一辆车也没有，系统负荷为0.5，意味着大桥一半的路段有车，系统负荷为1.0，意味着大桥的所有路段都有车，也就是说大桥已经"满"了，系统负荷为2.0，意味着车辆太多了，大桥已经被占满了（100%），后面等着上桥的车辆还有一倍。大桥的通行能力，就是CPU的最大工作量；桥梁上的车辆，就是一个个等待CPU处理的进程（process）。

  经验法则是这样的：
  当系统负荷持续大于0.7，你必须开始调查了，问题出在哪里防止情况恶化。
  当系统负荷持续大于1.0，你必须动手寻找解决办法，把这个值降下来。
  当系统负荷达到5.0，就表明你的系统有很严重的问题

  a. user:用户态，kernel:内核态
	b. faults:内存缺页，minor——轻微的，major——重度，需要从磁盘拿数据
	c. iowait:IO使用（等待）占比
	d. irq:硬中断，softirq:软中断

	注意：
	iowait占比很高，意味着有很大可能，是io耗时导致ANR，具体进一步查看有没有进程faults major比较多。
	单进程CPU的负载并不是以100%为上限，而是有几个核，就有百分之几百，如4核上限为400%。

  CPU负荷：查看命令 adb shell uptime
  14:56:18 up 22 days, 40 min,  0 users,  load average: 0.29, 0.40, 0.54


  
	内存信息（不一定有）
	Total number of allocations 476778　　进程创建到现在一共创建了多少对象
	Total bytes allocated 52MB　进程创建到现在一共申请了多少内存
	Total bytes freed 52MB　　　进程创建到现在一共释放了多少内存
	Free memory 777KB　　　 不扩展堆的情况下可用的内存
	Free memory until GC 777KB　　GC前的可用内存
	Free memory until OOME 383MB　　OOM之前的可用内存
	Total memory 当前总内存（已用+可用）
	Max memory 384MB  进程最多能申请的内存

	从含义可以得出结论：Free memory until OOME 的值很小的时候，已经处于内存紧张状态。应用可能是占用了过多内存。

	查看eventlog（不一定有）
	I am_meminfo: [350937088,41086976,492830720,427937792,291887104]
	五个值：Cached，Free，Zram，Kernel，Native
	Cached+Free的内存代表着当前整个手机的可用内存，如果值很小，意味着处于内存紧张状态。
	一般低内存的判定阈值为：4G内存手机以下阀值：350MB，以上阀值则为：450MB
	ps: 如果ANR时间点前后，日志里有打印onTrimMemory，也可以作为内存紧张的一个参考判断
```

#### trace分析

```java
trace内容解析
  "main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x71560df8 self=0x7a80246c00
  | sysTid=24791 nice=-10 cgrp=default sched=0/0 handle=0x7b06305ed0
  | state=S schedstat=( 309084171 15243024 335 ) utm=24 stm=6 core=1 HZ=100
  | stack=0x7fd43ad000-0x7fd43af000 stackSize=8192KB
  | held mutexes=

"main"：线程名称，prio：线程优先级，tid：不是线程id是一个虚拟机中实现线程锁的变量，Native：Linux线程状态

group：线程组名，sCount：线程被挂起的次数，dsCount：线程被调试器挂起的次数，odj：线程java对象的地址，self：这个线程本身的地址
（当进程开始调试后sCount会变为0，调试结束判断是否被正常挂起进行增长，但是dsCount不会变为0，所以dsCount可以用来判断这个线程是否被调试过）

sysTid：Linux下内核线程ID（主线程的话，与进程号相同），nice：Linux线程的优先级（nice值越小优先级越高。-10优先级就已经很高了），sched：分别标志线程的调度策略和优先级，cgrp：调度数组，handle：线程的处理函数地址

state：调度状态，utm：线程用户态下使用的时间值，stm：内核态下的调度时间值，core：最后执行这个线程的cpu核的序号


Linux线程状态：
ZOMBIE                              线程死亡，终止运行
RUNNING/RUNNABLE                    线程可运行或正在运行
TIMED_WAIT                          执行了带有超时参数的wait、sleep或join函数
MONITOR                             线程阻塞，等待获取对象锁
WAIT                                执行了无超时参数的wait函数
INITIALIZING                        新建，正在初始化，为其分配资源
STARTING                            新建，正在启动
NATIVE                              正在执行JNI本地函数
VMWAIT                              正在等待VM资源
SUSPENDED                           线程暂停，通常是由于GC或debug被暂停



案例1：主线程无卡顿，处于正常状态堆栈
	"main" prio=5 tid=1 Native（处于JNI调用）
	...
	at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:330)
  at android.os.Looper.loop(Looper.java:169)
  at android.app.ActivityThread.main(ActivityThread.java:7073)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:536)
  ...

  分析：堆栈是很正常的空闲堆栈，表明主线程正在等待新的消息。如果ANR日志里主线程是这样一个状态。
  那可能有两个原因：
	1、该ANR是CPU抢占或内存紧张等其他因素引起 -> 分析CPU、内存的情况
	2、这份ANR日志抓取的时候，主线程已经恢复正常 -> 关注trace抓日志的时间和ANR发生的时间是否相隔过久，太久这个堆栈就没分析的意义了。


案例2：主线程执行耗时操作
	 "main" prio=5 tid=1 Runnable（处于运行状态）
	 ...
	 at com.example.test.MainActivity$onCreate$2.onClick(MainActivity.kt:20)
   at android.view.View.performClick(View.java:7187)
   at android.view.View.performClickInternal(View.java:7164)
   at android.view.View.access$3500(View.java:813)
   at android.view.View$PerformClick.run(View.java:27640)
   ...

   分析：主线程正处于执行状态，堆栈并不是处于空闲状态，发生ANR是因为一处click监听函数里执行了耗时操作。


案例3：主线程被锁阻塞
  "main" prio=5 tid=1 Blocked（处于阻塞状态）
  ...
	at com.example.test.MainActivity$onCreate$1.onClick(MainActivity.kt:15)
  - waiting to lock <0x01aed1da> (a java.lang.Object) held by thread 3
  at android.view.View.performClick(View.java:7187)
  at android.view.View.performClickInternal(View.java:7164)
  ...

  "WQW TEST" prio=5 tid=3 TimeWating（处于等待状态）
  ...
  at java.lang.Thread.sleep(Native method)
  - sleeping on <0x043831a6> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:440)
  - locked <0x043831a6> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:356)
  at com.example.test.MainActivity$onCreate$2$thread$1.run(MainActivity.kt:22)
  - locked <0x01aed1da> (a java.lang.Object)
  at java.lang.Thread.run(Thread.java:919)

  分析：主线程等待tid为3的线程的一个锁对象（0x01aed1da），但是tid为3的线程处于等待状态，导致主线程阻塞从而导致ANR的发生。
  注意：常见因锁而ANR的场景是SharePreference写入。


案例4：CPU被抢占
	CPU usage from 0ms to 10625ms later (2020-03-09 14:38:31.633 to 2020-03-09 14:38:42.257):
  543% 2045/com.alibaba.android.rimet: 54% user + 89% kernel  faults: 4608 minor 1 major
  99% 674/android.hardware.camera.provider@2.4-service: 81% user + 18% kernel  faults: 403 minor
  24% 32589/com.wang.test: 22% user + 1.4% kernel  faults: 7432 minor 1 major
  ...

  分析：钉钉的进程，占据CPU高达543%，抢占了大部分CPU资源，因而导致发生ANR

案例5：内存紧张导致ANR
	CPU和堆栈都很正常，仍旧发生ANR了
	系统日志里搜索am_meminfo， 这个没有搜索到
	再次搜索onTrimMemory，果然发现了很多条记录

	10-31 22:37:19.749 20733 20733 E Runtime : onTrimMemory level:80,pid:com.xxx.xxx:Launcher0
	10-31 22:37:33.458 20733 20733 E Runtime : onTrimMemory level:80,pid:com.xxx.xxx:Launcher0
	10-31 22:38:00.153 20733 20733 E Runtime : onTrimMemory level:80,pid:com.xxx.xxx:Launcher0
	...

	分析：在发生ANR的时间点前后，内存都处于紧张状态，level等级是80。等级是很严重的，应用马上就要被杀死，被杀死的这个应用从名字可以看出来是桌面，连桌面都快要被杀死，那普通应用能好到哪里去呢？
	一般来说，发生内存紧张，会导致多个应用发生ANR，所以在日志中如果发现有多个应用一起ANR了，可以初步判定，此ANR与你的应用无关。

案例6：系统服务超时导致ANR
	"main" prio=5 tid=1 Native
	...
	at android.os.BinderProxy.transactNative(Native method)
  at android.os.BinderProxy.transact(Binder.java:804)
  at android.net.IConnectivityManager$Stub$Proxy.getActiveNetworkInfo(IConnectivityManager.java:1204)—关键行！
  at android.net.ConnectivityManager.getActiveNetworkInfo(ConnectivityManager.java:800)
  at com.xiaomi.NetworkUtils.getNetworkInfo(NetworkUtils.java:2)
  at com.xiaomi.frameworkbase.utils.NetworkUtils.getNetWorkType(NetworkUtils.java:1)
  ...

  分析：从堆栈可以看出获取网络信息发生了ANR：getActiveNetworkInfo。
  一个进程最多能使用16个binder线程。如果当systemserver的全部Binder线程都被阻塞，故不能再分配Binder线程处理其他的Binder消息，导致其他进程给systemserver发送Binder消息时是会被阻塞，且没有对端binder线程的。

  可进一步搜索：blockUntilThreadAvailable关键字：
  at android.os.Binder.blockUntilThreadAvailable(Native method)

  如果有发现某个线程的堆栈，包含此字样，可进一步看其堆栈，确定是调用了什么系统服务。
  此类ANR也是属于系统环境的问题，如果某类型机器上频繁发生此问题，应用层可以考虑规避策略。

  如果是Binder call产生的ANR，需要进一步确认下对端的情况；
  如果是耗时操作，直接修改成异步。怀疑系统执行慢可以看看binder_sample，dvm_lock等信息，其次gc多不多，lmk杀进程是不是很频繁，都可以看出系统的健康状态。

  Logcat中寻找蛛丝马迹：
  binder_sample：
  说明: 监控每个进程的主线程的binder transaction的耗时情况，当超过阈值时，则输出相应的目标调用信息，默认1000ms打开
  格式: 52004 binder_sample (descriptor|3),(method_num|1|5),(time|1|3),(blocking_package|3),(sample_percent|1|6)

  例子: 2754 2754 I binder_sample: [android.app.IActivityManager,35,2900,android.process.media,5]
  从上面的log中可以得出
  1、主线程 2754
  2、执行android.app.IActivityManager接口
  3、所对应方法code=35(即STOP_SERVICE_TRANSACTION),
  4、所花费时间为2900ms
  5、该block所在package为 android.process.media 
  最后一个参数是sample比例(没有太大价值)


  dvm_lock_sample
  说明: 当某个线程等待lock的时间blocked超过阈值,则输出当前的持锁状态
  格式: 20003 dvm_lock_sample (process|3),(main|1|5),(thread|3),(time|1|3),(file|3),(line|1|5),(ownerfile|3),(ownerline|1|5),(sample_percent|1|6)

  例子: dvm_lock_sample: [system_server,1,Binder_9,1500,ActivityManagerService.java,6403,-,1448,0]
  1、system_server: Binder_9在执行到ActivityManagerService.java的6403行代码
  2、一直在等待AMS锁，而该锁所同一文件的1448行代码所持有，从而导致Binder_9线程被阻塞1500ms


  binder starved
  说明: 当system_server等进程的线程池使用完, 无空闲线程时, 则binder通信都处于饥饿状态, 则饥饿状态超过一定阈值则输出信息
  参数: persist.sys.binder.starvation (默认值16ms)
  例子: 1232 1232 "binder thread pool (16 threads) starved for 100 ms"
  system_server进程的线程池已满的持续长达100ms，一般有了这些信息之后，可以辅助我们确定问题原因归属是系统原因还是App原因
```



### ANR监控

#### WatchDog

```java
anr监控：(本地阀门，超过阀门值自动上报日志到后台。比如：设备累计出现6次 或 一天内出现3次 自动上报到后台。不一定每次都会anr但是这也是非常危险的信号，需要我们重点关注，为什么会出现那么长的耗时)
  1) 自定义Watchdog：监控线程耗时（主线程就是监控anr了）、监控死锁
  //监控线程耗。如果是主线程监控anr的话，休眠时间可以设置小于5s的。
  //比如每4s 走一次检查，如果4s还没检查完就dump堆栈。
  addThread(Handler thread)
  //监控死锁
  addMonitor(Monitor monitor) 

  2) Looper#setMessageLogging：重写Looper的日志打印logging（监控Handler消息执行是否有耗时，用于统计操作耗时、分析潜在anr风险）
  执行消息前：logging.println(">>>>> Dispatching to " + msg.target + " " + msg.callback + ": " + msg.what);
  执行消息后：logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);


缺点：
1、检测不够准确，存在误判、漏判
2、无法抓取trace文件

抓trace只能够通过：
① adb bugreport > bugreport.txt：打包整个系统平台和某个app在运行一段时间之内的所有信息。
	battery historian的分析工具，图形化解析bugreport.txt文件。（https://github.com/google/battery-historian）
② 去到系统分区取 data\anr\traces.txt
③ dropbox日志（系统运行中所发生的crash、anr、eventlog都会保存在这）：目录/data/system/dropbox（需要root）；可用DropBoxManager取出（需要系统权限）。
  adb shell dumpsys dropbox --print  >  crash.txt  
```



#### Matrix - TraceCannary

```java
ANR的过程：
当应用发生ANR之后，AMS会收集许多相关进程的pid，用来dump堆栈从而生成ANR trace文件。
（为了保证系统可用，dump堆栈的时间不能超过20s，超过主动放弃dump）。收集的第一个进程，也是一定会被收集到的进程，就是发生ANR的进程。

接着系统开始向这些应用进程发送SIGQUIT信号，应用进程收到SIGQUIT信号后开始dump堆栈生成trace文件（这个dump过程是发生在应用进程的）。
每一个应用进程都会有一个SignalCatcher线程，专门处理SIGQUIT信号。（WaitForSignal方法调用了sigwait方法，这是一个阻塞方法。这里的死循环，就会一直不断的等待监听SIGQUIT和SIGUSR1这两个信号的到来。）

然后再回到AMS完成剩余的ANR流程。

接收SIGQUIT信号：
1、sigwait（该方案不可靠）
由于Signal Catcher线程在sigwait，我们再写一个线程sigwait。由于两个线程通过sigwait方法监听同一个信号时，具体哪一个线程收到信号时不能确定的。


2、Signal Handler
同时有sigwait和signal handler的情况下，信号没有走signal handler而是依然被系统的Signal Catcher线程捕获到了。

原因是Android默认把SIGQUIT设置成了BLOCKED，通过pthread_sigmask或者sigprocmask把SIGQUIT设置为UNBLOCK，那么再次收到SIGQUIT时，就一定会进入到我们的handler方法中。

需要注意，通过Signal Handler抢到了SIGQUIT后，原本的Signal Catcher线程中的sigwait就不再能收到SIGQUIT了，原本的dump堆栈的逻辑就无法完成了，为了ANR的整个逻辑和流程跟原来完全一致，需要在Signal Handler里面重新向Signal Catcher线程发送一个SIGQUIT。（如果缺少了重新向SignalCatcher发送SIGQUIT的步骤，AMS就一直等不到ANR进程写堆栈，直到20秒超时后，才会被迫中断，而继续之后的流程。直接的表现就是ANR弹窗非常慢（20秒超时时间），并且/data/anr目录下无法正常生成完整的 ANR Trace文件。）


3、完善Signal Handler

收到SIGQUIT信号不一定是发生了ANR?
收到SIGQUIT信号并不一定是发生了anr，因为有可能是其他程序或系统发出的用作其他用途的（发送信号方法：java层调用android.os.Process.sendSignal方法；Native层调用kill或者tgkill方法）。


解决方法1：检查NOT_RESPONDING的flag。
在ANR弹窗前会执行到makeAppNotRespondingLocked方法中，会给发生ANR进程标记一个NOT_RESPONDING的flag。而这个flag可以通过ActivityManager来获取：ActivityManager#getProcessesInErrorState，判断下是否为ActivityManager.ProcessErrorStateInfo.NOT_RESPONDING。监控到SIGQUIT后，在20秒内不断轮询自己是否有NOT_RESPONDING，一旦发现有这个flag那么马上就可以认定发生了一次ANR。

漏报情况：发生ANR了但是进程查不到NOT_RESPONDING的情况
后台ANR：ANR被标记为了后台ANR（即SilentAnr），那么杀死进程后就会直接return，并不会走到产生进程错误状态的逻辑。后台ANR没办法捕捉到，而后台ANR的量同样非常大，并且后台ANR会直接杀死进程。
前台ANR：部分厂商系统定制化，前台的ANR也并不会弹窗而是直接杀死进程。


解决方法2：检查主线程消息队列，头消息的入队时间（减去当前时间，得到是等待时间，如果等待时间过长就说明主线程是处于卡住的状态）。

最好是这两个方法结合！！！



4、trace信息的捕获
截获SIGQUIT信号之后，会再次发送SIGQUIT信号出去。因为需要唤醒Signal Catcher线程的sigwait，让该线程工作起来。
会执行写trace文件的操作，它会先connet方法链接一个path为“/dev/socket/tombstoned_java_trace”的socket，然后执行write，生成trace。

我们需要hook connet，判断path是否为“/dev/socket/tombstoned_java_trace”
然后再对Signal Catcher线程的第一次write，write的内容就是我们所需要的。

hook是用爱奇艺的xHook（PLT hook）

5、总结
这个方案不仅仅可以用来查ANR问题，任何时候我们都可以手动向自己发送一个SIGQUIT信号，从而hook到当时的Trace。Trace的内容对于我们排查线程死锁，线程异常，耗电等问题都非常有帮助。


matrix相关代码（所有逻辑在 模块：matrix-trace-canary）

SignalHandler.cc#signalHandler（实现signalHandler，一直轮询信号）
-> AndrDumper.cc#handleSignal（判断信号是否为SIGQUIT）
-> AndrDumper.cc#siUserCallback（hook connet/write（hookAnrTraceWrite），再次发送SIGQUIT信号（sendSigToSignalCatcher））

等到Signal Catcher线程执行 connet /dev/socket/tombstoned_java_trace
-> MatrixTracer.cc#my_connect（判断是否为 /dev/socket/tombstoned_java_trace）
等到Signal Catcher线程执行 write
-> MatrixTracer.cc#my_write（执行写入（writeAnr），执行回调上层（printTraceCallback））
-> MatrixTracer.cc#writeAnr（执行un hook，将内容写入到文件）

java层 主要是 SignalAnrTracer
SignalAnrTracer#isMainThreadBlocked 检查主线程是否处于卡住的状态
SignalAnrTracer#checkErrorStateCycle 检查NOT_RESPONDING的flag
```



## 业务逻辑的正常运行

```java
例如无法正常登录，问题怎么定位？到底出问题是在客户端？服务端？还是第三方sdk？
  qq -> 业务后台 -> im sdk -> 直播间sdk -> 登录成功
  优化后：qq -> 业务后台 -> im sdk -> 登录成功 -> 直播间sdk
  
日志收集器：推拉结合，实现业务日志上报（用户在平台反馈问题上报；业务后台主动发起拉取等）
	封装日志打印工具，在业务代码中统一用这个日志工具打印，日志将写入本地。（日志工具：）
 （约定：不打频繁日志；不打敏感日志；注意日志等级；单次打印注意数据量等）
	通过长链接场景的消息推送（例如：云信），当后台需要拉取指定用户的日志时发送消息给客户端，客户端收到消息就打包日志上传。
```



## 其他  

```java
分析event log：通过EventLog来分析Activity、Process、CPU、Window等相关信息
支持的事件清单：/system/etc/event-log-tags
events日志获取：adb logcat -b events
```



## 稳定性优化量化

```java
1、bugly崩溃率统计；
3、Rxjava捕获、Looper捕获上报统计；
2、anr监控上报统计；
  比对优化前版本 与 优化后版本。（一般都是大版本才做比对，例如 2.0.0 -> 3.0.0，2.1.0->2.2.0这种参考价值不高）
```

