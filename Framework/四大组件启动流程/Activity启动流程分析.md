## 注意

### 为什么Zygote与AMS之间是用Socket通信，而不是用Binder？

```java
1、Socket比Binder性能更好？更安全？
反驳：
性能：Binder数据拷贝只需要拷贝一次，而Socket是要两次的。
安全：Binder：系统会为应用分配UID，可以同时支持实名和匿名。实名：系统服务是实名的。匿名：自定义的Service（其他进程拿不到）
	   Socket：依赖上层协议，访问接入点是开放性的，不安全。

  
2、Zygote进程没办法用Binder？
观点：
  Zygote是Linux层就有的，而Binder是android层才有的。会不会Zygote启动的时候，还没有Binder呢？
反驳：
  ServiceManager是一个守护进程，它维护着系统服务和客户端的Binder通信。Binder机制是基于   ServiceManager的。
  ServiceManager在init进程启动后启动，Zygote进程在init之后。ServiceManager进程的启动远比zygote要 早。
  在启动Zygote进程是需要用到ServiceManager进程的服务，那么显然Zygote是可以使用Binder的。

  
3、fork函数限制（多线程程序里不准使用fork）？
观点：
  怕父进程binder线程有锁，然后子进程的主线程一直在等其子线程(从父进程拷贝过来的子进程)的资源，但是其实父	 进程的子进程并没有被拷贝过来，造成死锁。所以fork不允许存在多线程。
  而非常巧的是Binder通讯偏偏就是多线程，所以干脆父进程（Zygote）这个时候就不使用binder线程。

反驳：
  应该是有办法让Zygote主线程直接作为一个唯一的Binder线程。
  让init启动Zygote时直接将Zygote主线程注册成Binder线程并且是唯一线程。

  
所有我感觉，Zygote应该是可以用Binder替换Socket。
网上很多都是偏向于 -> Binder是多线程的所以不能用在有fork的Zygote中。
```





## Activity启动流程

#### 总结

```java
App启动流程：
1、Launcher向AMS请求启动Activity；
Launcher会给Intent加上 FLAG_ACTIVITY_NEW_TASK（singleTask）（要启动程序的根Activity，需要创建任务栈）。

2、AMS检查应用进程是否存在，不存在则向Zygote请求孵化一个进程；
AMS与Zygote之间会建立一个Socket连接，将启动参数发送给Zygote。

3、Zygote会执行孵化子进程，孵化后在子进程中反射 ActivityThread 并调用其main方法；
Zygote作为Socket的服务端，会一直轮询等待客户端连接。当AMS与Zygote建立连接后，Zygote根据AMS传过来的启动参数，孵化子进程（fork方式孵化子进程）。
在子进程中会反射 ActivityThread 并调用其main方法。

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



#### 源码分析

##### Launcher向AMS请求启动Activity

```java
Launcher#startActivitySafely -> Activity#startActivity -> Instrumentation#execStartActivity（调用ActivityManager.getService().startActivity）-> 跑到AMS...
  
关键点：
Launcher#startActivitySafely 解析：
    给要启动的Intent加上 Intent.FLAG_ACTIVITY_NEW_TASK（singleTask）（要启动程序的根Activity，需要创建任务栈）;

ActivityManager.getService 解析：获取AMS的代理对象；
    //得到activity的service引用，即IBinder类型的AMS引用
    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE); 
    //转换成IActivityManager对象
    final IActivityManager am = IActivityManager.Stub.asInterface(b);

ActivityManager.getService().startActivity 解析：通过Binder接口，调用AMS方法
```



##### AMS检查应用进程是否存在，不存在则向Zygote请求孵化一个进程

###### 第一步：AMS调用Process来进行进程启动

```java
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
```



###### 第二步：Process向Zygote进程发送创建应用进程请求

```java
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
```



##### Zygote执行孵化子进程，孵化后在子进程中反射 ActivityThread 并调用其main方法

###### 第一步、fork出应用进程

```java
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
        } catch (Throwable ex) { ... } finally { zygoteServer.closeServerSocket(); }

        //执行AMS请求返回的Runnable
        if (caller != null) {
            caller.run();
        }
    }
```



###### 第二步、在应用进程反射ActivityThread，并调用其main方法

```java
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
```



##### 应用进程启动并且与AMS绑定

IActivityManager：应用进程持有的AMS binder接口；

IApplicationThread：AMS持有的应用进程binder接口。



###### 第一步、AMS初始化应用进程的Application

```java
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
```



###### 第二步、AMS创建ClientTransaction传递给应用进程

```java
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
```



###### 第三步、应用进程创建实例Activity，走onCreate生命周期。

```java
执行LaunchActivityItem：TransactionExecutor#executeCallbacks -> LaunchActivityItem#execute -> ClientTransactionHandler#handleLaunchActivity -> ClientTransactionHandler#performLaunchActivity -> ... -> onCreate()
  
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
```



###### 第四步、走onStart、onResume生命周期

```java
onStart：
TransactionExecutor#executeLifecycleState -> TransactionExecutor#cycleToPath -> TransactionExecutor#performLifecycleSequence -> ActivityThread#handleStartActivity -> ... -> onStart()

onResume：
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



## Activity 销毁流程

#### 总结

```java
1、Activity的生命stop/destory是依赖IdleHandler来回调，也就是在启动下一个Activity#onResume之后的那段空闲时间，才会执行的。
2、在Activity#onResume之后也会发出一个10s的兜底事件，防止stop/destory一直不执行。
3、如果在主线程的Handler消息一直很繁忙的话，是会影响stop/destory的回调。最严重的情况会出现10s才回调。
```



#### 源码分析

##### finish()执行流程

```java
(app)finish -> AMS#finishActivity -> ActivityStack#finishActivityLocked -> ActivityStack#startPausingLocked -> IApplicationThread#schedulePauseActivity -> (app)ActivityThread#handlePauseActivity（回调onPause） -> AMS#activityPaused -> ActivityStack#activityPausedLocked（移除pause兜底事件）-> ActivityStack#completePauseLocked
  
关键点：
ActivityStack#startPausingLocked 解析：
1、prev.app.thread.schedulePauseActivity（调用app进程的Binder接口schedulePauseActivity）
2、增加pause流程兜底机制，发送500ms延时事件，防止第1步的pause流程不执行。

ActivityThread#handlePauseActivity 解析：
1、调用onPause生命周期方法
2、调用ActivityManagerNative.getDefault().activityPaused(token) 告诉AMS，我执行了onPause

ActivityStack#completePauseLocked 解析：
1、若Activity变为不可见时，调用addToStopping函数，将中断的Activity加入到mStoppingActivities（ArrayList<ActivityRecord>）中；
2、进入启动目标Activity的流程（ActivityStackSuperVisor#resumeFocusedStackTopActivityLocked）。

也就是说，Activity执行完onPause生命周期之后，AMS并不会立即给它走onStop，而是加到一个缓存列表里边。那什么时候才会执行呢？
```



##### onStop/onDestory执行流程

```java
Activity在onResume执行完后会调用addIdleHandler（一般情况下onResume之后app都会相对空闲）。当消息队列空闲的时候，告诉AMS我空闲了。AMS就会帮我们处理一下事情，其中就包括给finish的Activity走销毁的生命周期。
  
ActivityThread#handleResumeActivity（回调onResume）-> Looper.myQueue().addIdleHandler(new Idler()) -> IActivityManager#activityIdle -> AMS#activityIdle -> ActivityStackSupervisor#activityIdleInternalLocked

关键点：
ActivityThread#handleResumeActivity 解析：
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward, String reason) {
        ...
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason); //调用onResume
        ...
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            ...
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l); //DecorView add to Window
                } else {
                    a.onWindowAttributesChanged(l);
                }
            }
        } 
        ...
        Looper.myQueue().addIdleHandler(new Idler()); //关键！！！
    }

ActivityThread.Idler 解析：
private class Idler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            ActivityClientRecord a = mNewActivities;
            ...
            if (a != null) {
                mNewActivities = null;
                IActivityManager am = ActivityManager.getService();
                ActivityClientRecord prev;
                do {
                    if (a.activity != null && !a.activity.mFinished) {
                        try {
                            am.activityIdle(a.token, a.createdConfig, stopProfiling);
                            a.createdConfig = null;
                        } catch (RemoteException ex) { ... }
                    }
                    prev = a;
                    a = a.nextIdle;
                    prev.nextIdle = null;
                } while (a != null);
            }
            if (stopProfiling) { mProfiler.stopProfiling(); }
            ensureJitEnabled();
            return false;
        }
    }

ActivityStackSupervisor#activityIdleInternalLocked 解析：给上一个Activity个痛快，给它走onStop、onDestory
final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout, boolean processPausingActivities, Configuration config) {
        ...
        if (r != null) {
            mHandler.removeMessages(IDLE_TIMEOUT_MSG, r); //移除兜底机制消息
            ...
        }

        ...

        final ArrayList<ActivityRecord> stops = processStoppingActivitiesLocked(r, true /* remove */, processPausingActivities);
        NS = stops != null ? stops.size() : 0;
        if ((NF = mFinishingActivities.size()) > 0) {
            //mFinishingActivities是全局变量，记录着在finish中的Activity
            finishes = new ArrayList<>(mFinishingActivities); 
            mFinishingActivities.clear();
        }

        ...

        //给Activity执行onStop
        for (int i = 0; i < NS; i++) {
            r = stops.get(i);
            final ActivityStack stack = r.getStack();
            if (stack != null) {
                if (r.finishing) {
                    stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false, "activityIdleInternalLocked");
                } else {
                  /* stopActivityLocked逻辑：
                mService.getLifecycleManager().scheduleTransaction(r.app.thread, r.appToken, 
	StopActivityItem.obtain(r.visible, r.configChangeFlags));

                  */
                    stack.stopActivityLocked(r);
                }
            }
        }

        //给Activity执行onDestory
        for (int i = 0; i < NF; i++) {
            r = finishes.get(i);
            final ActivityStack stack = r.getStack();
            if (stack != null) {
               /* destroyActivityLocked逻辑：
                 mService.getLifecycleManager().scheduleTransaction(r.app.thread, r.appToken, DestroyActivityItem.obtain(r.finishing, r.configChangeFlags));
               */
                activityRemoved |= stack.destroyActivityLocked(r, true, "finish-idle");
            }
        }
        ...
    }

在特殊情况，app的消息队列一直处理繁忙，那岂不是finish的Activity就一直不走销毁的生命周期了？大可放心，AMS也是有兜底机制的。
stop/destory流程兜底机制：下一个Activity#onResume回调之后，发送一个10s的兜底事件。
ActivityStackSupervisor#resumeFocusedStackTopActivityLocked -> ActivityStack#resumeTopActivityUncheckedLocked -> ActivityStack.resumeTopActivityInnerLocked -> ActivityRecord.completeResumeLocked -> ActivityStackSupervisor.scheduleIdleTimeoutLocked

void scheduleIdleTimeoutLocked(ActivityRecord next) {
  Message msg = mHandler.obtainMessage(IDLE_TIMEOUT_MSG, next);
  mHandler.sendMessageDelayed(msg, IDLE_TIMEOUT); //IDLE_TIMEOUT的值是10*1000，延时10s
}

case IDLE_TIMEOUT_MSG: {
    activityIdleInternal((ActivityRecord) msg.obj, true);
} 
```

