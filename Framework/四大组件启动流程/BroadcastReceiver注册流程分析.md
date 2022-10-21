## 动态注册广播

#### 总结

1、创建好的接收者对象，在注册时会根据Context缓存到对应的Map中；
2、然后会通知AMS自己要一个动态广播，AMS会根据传过来的IIntentReceiver对象也缓存到对应的一个Map中。



#### 调用链

```java
ContextImpl#registerReceiver -> ContextImpl#registerReceiverInternal -> AMS#registerReceiver
```



#### 源码解析

##### ContextImpl#registerReceiverInternal

```java
//ContextImpl#registerReceiverInternal：一个BroadcastReceiver对应一个IIntentReceiver
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId, IntentFilter filter, String broadcastPermission, Handler scheduler, Context context, int flags) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                ...
                //通过LoadedApk获取IIntentReceiver，
                //LoadedApk#mReceivers（ArrayMap<Context, ArrayMap<BroadcastReceiver, ReceiverDispatcher>>）缓存着所有动态注册广播接收者
                //key为Context：每个Activity、Service组件都是继承自Context，用Context去查找这个组件所有的广播接收者。
                //key为BroadcastReceiver：通过BroadcastReceiver查找该接收者是否已经注册了，已经注册了就直接返回ReceiverDispatcher。
                //ArrayMap<Context, ArrayMap<BroadcastReceiver, ReceiverDispatcher>>：实现了一个Context能创建多个BroadcastReceiver
                rd = mPackageInfo.getReceiverDispatcher(receiver, context, scheduler, mMainThread.getInstrumentation(), true);
            } else {
                ...
                rd = new LoadedApk.ReceiverDispatcher(receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            final Intent intent = ActivityManager.getService().registerReceiver(mMainThread.getApplicationThread(), mBasePackageName, rd, filter,broadcastPermission, userId, flags);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) { }
    }
```



##### AMS#registerReceiver

```java
//AMS#registerReceiver：一个IIntentReceiver对应着一个ArrayList<IntentFilter>
public Intent registerReceiver(IApplicationThread caller, String callerPackage, IIntentReceiver receiver, IntentFilter filter, String permission, int userId, int flags) {
        ... //找一下有没有要发给这个接收者的粘性广播
        synchronized (this) {
            ...
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid, userId, receiver);
                if (rl.app != null) {
                    final int totalReceiversForApp = rl.app.receivers.size();
                    if (totalReceiversForApp >= MAX_RECEIVERS_ALLOWED_PER_APP) {
                        throw new IllegalStateException(...); //如果接收者超过了最大注册数（1000），就抛异常。
                    }
                    rl.app.receivers.add(rl);
                } else {
                    ...
                }
              	//ReceiverList extends ArrayList<BroadcastFilter>
                //BroadcastFilter extends IntentFilter
              	//加入到mRegisteredReceivers
                mRegisteredReceivers.put(receiver.asBinder(), rl); 
            } 
            ... //如果有要发给这个接收者的粘性广播，就给他发吧（BroadcastQueue#enqueueParallelBroadcastLocked）
        }
    }
```



## 静态注册广播

#### 总结

1、在清单文件定义好的广播接收者，在app安装完成 或者 系统启动后清单文件会被PMS扫描；

2、PMS将广播接收者的描述信息保存在PMS里边的mReceivers容器中。



#### 调用链

```java
系统启动 -> PackageManagerService创建（扫描系统路径apk、应用路径apk） -> PMS#scanDirLI -> PMS#scanPackageLI -> 解析清单文件，将“receiver”节点的配置信息封装好保存到mReceivers（IntentResolver<PackageParser.ActivityIntentInfo, ResolveInfo>）
```



#### 源码解析

##### PMS#scanPackageLI

```java
...
N = pkg.receivers.size();
r = null;
for (i=0; i<N; i++) {
   PackageParser.Activity a = pkg.receivers.get(i);
   a.info.processName = fixProcessName(pkg.applicationInfo.processName,
           a.info.processName, pkg.applicationInfo.uid);
   mReceivers.addActivity(a, "receiver");
   ...
}
...
```



## 广播分发

#### 总结
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



#### 调用链

```java
ContextImpl#broadcastIntent -> AMS#broadcastIntentLocked -> AMS#broadcastIntentLocked -> BroadcastQueue#scheduleBroadcastsLocked（sendMsg BROADCAST_INTENT_MSG）-> BroadcastHandler# case BROADCAST_INTENT_MSG -> BroadcastQueue#processNextBroadcast -> BroadcastQueue#processNextBroadcastLocked
 
BroadcastQueue#processNextBroadcastLocked拆分：
逻辑1：轮询mParallelBroadcasts队列（先把无序广播发送给动态注册的接收者）
轮询mParallelBroadcasts队列（并行执行）：
  	BroadcastQueue#deliverToRegisteredReceiverLocked -> BroadcastQueue#performReceiveLocked -> IApplicationThread#scheduleRegisteredReceiver -> IIntentReceiver#performReceive -> ReceiverDispatcher#performReceive（post Runnable）-> 反射调用onReceive，调ReceiverData#finish -> BroadcastReceiver.PendingResult#finish -> BroadcastReceiver.PendingResult#sendFinished -> AMS#finishReceiver
  
逻辑2：轮询mOrderedBroadcasts队列（再把无序广播发送给静态注册的接收者们 或 把有序广播发给所有接收者）
轮询mOrderedBroadcasts队列（串行执行）：
    情况1：发给动态注册的广播接收者：与无序广播发送给动态注册的接收者的逻辑一样
    情况2：发给静态注册的广播接收者（进程已存在）：BroadcastQueue#processCurBroadcastLocked -> IApplicationThread#scheduleReceiver -> ActivityThread#scheduleReceiver(sendMsg RECEIVER)) -> ActivityThread#handleReceiver（反射创建Receiver对象，并调onReceive后还再调AMS#finishReceiver）-> AMS#finishReceiver（判断是否需要继续执行分发，如果是） -> BroadcastQueue#processNextBroadcastLocked -> ...给下一个接收者发广播
    情况3：发给静态注册的广播接收者（进程不存在）：AMS#startProcessLocked -> 初始化app进程后，调用AMS#attachApplication -> AMS#sendPendingBroadcastsLocked -> BroadcastQueue#sendPendingBroadcastsLocked -> BroadcastQueue#processCurBroadcastLocked -> ...（跟情况2一样了）
```



#### 源码解析

##### AMS#broadcastIntentLocked

```java
final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {
        ...

        //该flag作用：让stopped状态的应用不能接收到该广播
        //有个与之相反的FLAG_INCLUDE_STOPPED_PACKAGES
        intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);

        ...
        
        //系统应用会在Manifest里标记一些广播action为受保护，这里通过PMS判断该广播是否为受保护广播
        final boolean isProtectedBroadcast;
        try {
            isProtectedBroadcast = AppGlobals.getPackageManager().isProtectedBroadcast(action);
        } catch (RemoteException e) {}

        final boolean isCallerSystem;
        switch (UserHandle.getAppId(callingUid)) {
            case ROOT_UID:
            case SYSTEM_UID:
            case PHONE_UID:
            case BLUETOOTH_UID:
            case NFC_UID:
            case SE_UID:
                isCallerSystem = true;
                break;
            default:
                isCallerSystem = (callerApp != null) && callerApp.persistent;
                break;
        }

        if (!isCallerSystem) { //通过uid判断是否为系统进程
            //不是系统进程，你又发了受保护的广播，那就给你一抛一个安全异常。
            if (isProtectedBroadcast) {
                ...
                throw new SecurityException(msg); //Permission Denial: not allowed to send broadcast
            }
            ...
        }

        if (action != null) { //对发送的一些特殊广播做处理
            ...
            switch (action) {
                //检查特定action是否有申请权限
                case Intent.ACTION_UID_REMOVED:
                case Intent.ACTION_PACKAGE_REMOVED:
                case Intent.ACTION_PACKAGE_CHANGED:
                case Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE:
                case Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE:
                case Intent.ACTION_PACKAGES_SUSPENDED:
                case Intent.ACTION_PACKAGES_UNSUSPENDED:
                    if (checkComponentPermission(android.Manifest.permission.BROADCAST_PACKAGE_REMOVED,callingPid, callingUid, -1, true)
                            != PackageManager.PERMISSION_GRANTED) {
                        ...
                        throw new SecurityException(msg); //Permission Denial: 缺少权限 BROADCAST_PACKAGE_REMOVED
                    }
                    ...
                    break;
                ...
                case "com.android.launcher.action.INSTALL_SHORTCUT":
                    //安卓O之后不支持这快照广播了，要使用ShortcutManager.pinRequestShortcut().了
                    Log.w(TAG, "Broadcast " + action + " no longer supported. It will not be delivered.");
                    return ActivityManager.BROADCAST_SUCCESS;
            }
            ...
        }

        if (sticky) { //处理粘性广播
            if (checkPermission(android.Manifest.permission.BROADCAST_STICKY,callingPid, callingUid)
                    != PackageManager.PERMISSION_GRANTED) {
                ...
                throw new SecurityException(msg); //Permission Denial: 缺少BROADCAST_STICKY权限
            }
            ...
            if (intent.getComponent() != null) { //粘性广播不能指定target
                throw new SecurityException("Sticky broadcasts can't target a specific component");
            }
            ...
            //所有粘性广播都会保存在mStickyBroadcasts（SparseArray<ArrayMap<String, ArrayList<Intent>>>）
            //根据userId取出该应用发送的所有粘性广播
            ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(userId);
            if (stickies == null) {
                stickies = new ArrayMap<>();
                mStickyBroadcasts.put(userId, stickies);
            }

            //根据action取出对应未发送的粘性广播Intent（因为有可能一个广播发送多次，所以是要用List存储）
            ArrayList<Intent> list = stickies.get(intent.getAction());
            if (list == null) {
                list = new ArrayList<>();
                stickies.put(intent.getAction(), list);
            }
            //遍历list，符合替换条件就替换intent，不符合替换条件就添加到list末尾
            //替换条件：action、data、type、pkg、component、categories这些全部相同才能替换
            //我的理解：符合替换条件，但是并不代表两个intent是完全相同的。flags有可能不同的，所以才需要替换更新
            ...
        }

        int[] users; //判断这个广播是发给所有用户，还是指定用户
        if (userId == UserHandle.USER_ALL) {
            users = mUserController.getStartedUserArray();
        } else {
            users = new int[] {userId};
        }


        List receivers = null; //存储静态注册的接收者
        List<BroadcastFilter> registeredReceivers = null; //存储动态注册的接收者
        //FLAG_RECEIVER_REGISTERED_ONLY：只有动态注册的接收者才能接收
        //如果没有这个flag，才会去PMS那收集静态注册的接收者
        if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)== 0) {
            //从PMS那收集静态注册的接收者（按照广播优先级排序返回）
            receivers = collectReceiverComponents(intent, resolvedType, callingUid, users); 
        }

        //匹配符合接收条件的动态注册接收者（按照广播优先级排序返回），并且存储在registeredReceivers
        if (intent.getComponent() == null) {
            if (userId == UserHandle.USER_ALL && callingUid == SHELL_UID) {
                for (int i = 0; i < users.length; i++) {
                    ...
                    List<BroadcastFilter> registeredReceiversForUser = mReceiverResolver.queryIntent(intent,resolvedType, false /*defaultOnly*/, users[i]);
                    if (registeredReceivers == null) {
                        registeredReceivers = registeredReceiversForUser;
                    } else if (registeredReceiversForUser != null) {
                        registeredReceivers.addAll(registeredReceiversForUser);
                    }
                }
            } else {
                registeredReceivers = mReceiverResolver.queryIntent(intent,resolvedType, false /*defaultOnly*/, userId);
            }
        }

        ...

        int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
        if (!ordered && NR > 0) { //发送无序广播给动态注册的接收者
            ...
            //根据flag返回对应的队列，FLAG_RECEIVER_FOREGROUND：返回前台队列；FLAG_FROM_BACKGROUND：返回后台队列(默认是后台队列)。
            //广播配置了FLAG_RECEIVER_FOREGROUND，优先让在前台应用接收。这个flag可以降低前台应用接收的广播延迟
            final BroadcastQueue queue = broadcastQueueForIntent(intent);
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
                    requiredPermissions, appOp, brOptions, registeredReceivers, resultTo,
                    resultCode, resultData, resultExtras, ordered, sticky, false, userId);
            final boolean replaced = replacePending
                    && (queue.replaceParallelBroadcastLocked(r) != null);
            if (!replaced) {
                queue.enqueueParallelBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
            registeredReceivers = null;
            NR = 0;
        }

        int ir = 0;
        if (receivers != null) {
            ...

            //如果是有序广播才会走这个逻辑（如果是无需广播上边 NR = 0了，所有ir < NR是不成立的。）
            //动态注册的接收者 与 静态注册的接收者 按照广播优先级合并到receivers这个列表里
            int NT = receivers != null ? receivers.size() : 0;
            int it = 0;
            ResolveInfo curt = null;
            BroadcastFilter curr = null;
            while (it < NT && ir < NR) {
                if (curt == null) curt = (ResolveInfo)receivers.get(it);
                if (curr == null) curr = registeredReceivers.get(ir);
                if (curr.getPriority() >= curt.priority) {
                    receivers.add(it, curr);
                    ir++;
                    curr = null;
                    it++;
                    NT++;
                } else {
                    it++;
                    curt = null;
                }
            }
        }
        //如果是有序广播才会走这个逻辑（如果是无需广播上边 NR = 0了，所有ir < NR是不成立的。）
        //防止没有静态注册的接收者，这里将动态注册的接收者放进receivers这个列表里
        while (ir < NR) {
            if (receivers == null) {
                receivers = new ArrayList();
            }
            receivers.add(registeredReceivers.get(ir));
            ir++;
        }

        ...

        //发送无序广播给静态注册的接收者 或者 发送有序广播给排序好的接收者（按照广播优先级排序好了动态注册、静态注册的接收者）
        if ((receivers != null && receivers.size() > 0) || resultTo != null) {
            BroadcastQueue queue = broadcastQueueForIntent(intent);
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
                    requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                    resultData, resultExtras, ordered, sticky, false, userId);
            final BroadcastRecord oldRecord = replacePending ? queue.replaceOrderedBroadcastLocked(r) : null;
            if (oldRecord != null) {
                
            } else {
                queue.enqueueOrderedBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
        } else {
            if (intent.getComponent() == null && intent.getPackage() == null
                    && (intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
                addBroadcastStatLocked(intent.getAction(), callerPackage, 0, 0, 0);
            }
        }

        return ActivityManager.BROADCAST_SUCCESS;
    }
```



##### BroadcastQueue#processNextBroadcastLocked

```java
final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
        BroadcastRecord r;
        ...
        while (mParallelBroadcasts.size() > 0) { //处理无序队列
            r = mParallelBroadcasts.remove(0);
            r.dispatchTime = SystemClock.uptimeMillis();
            r.dispatchClockTime = System.currentTimeMillis();
            ...
            final int N = r.receivers.size();
            for (int i=0; i<N; i++) { //并行分发广播给所有的接收者
                Object target = r.receivers.get(i);
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
            }
            ...
        }

        ...

        do { //处理有序队列
            ...
            r = mOrderedBroadcasts.get(0);
            boolean forceReceive = false;

            int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
            if (mService.mProcessesReady && r.dispatchTime > 0) {
                long now = SystemClock.uptimeMillis();
                //r.dispatchTime：广播刚开始分发的时间
                //mTimeoutPeriod：配置了前台接受（Intent.FLAG_RECEIVER_FOREGROUND）的话是10s，反之是60s
                //这里判断是不是分发超时了，如果是超时了直接废掉当前这个BroadcastRecord。（一个广播发那么久都没发不完，老子不发了。）
                if ((numReceivers > 0) && (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                    /* broadcastTimeoutLocked方法逻辑：
                    ...
                    BroadcastRecord r = mOrderedBroadcasts.get(0);
                    ...
                    //当前就是因为等待分发这个广播而导致的超时
                    //需要pending置空，由于是串行处理，如果有pending广播是不会继续执行的
                    if (mPendingBroadcast == r) { mPendingBroadcast = null; }
                    //重置当前这个超市的广播
                    finishReceiverLocked(r, r.resultCode, r.resultData, r.resultExtras, r.resultAbort, false);
                    //再发送消息到消息队列，申请再次调度广播
                    scheduleBroadcastsLocked();
                    //由于等待这个进程导致超时，给它一个anr
                    mHandler.post(new AppNotResponding(app, anrMessage));
                    */
                    broadcastTimeoutLocked(false); //处理一些超时的善后工作
                    forceReceive = true;
                    r.state = BroadcastRecord.IDLE;
                }
            }

            if (r.state != BroadcastRecord.IDLE) { //当前广播正在分发，不做处理
                return;
            }

          	//r.receivers == null：这个广播分发处理完了
          	//r.resultAbor：这个广播被拦截了，不分发给后续的接收者了，移除这个广播
            //forceReceive：广播超时了，抛弃这个广播，不处理了。
            if (r.receivers == null || r.nextReceiver >= numReceivers || r.resultAbort || forceReceive) {
                ...
                cancelBroadcastTimeoutLocked(); 
                mOrderedBroadcasts.remove(0); //移除掉这个广播
                r = null;
                looped = true;
                continue; //再走循环处理下一个广播
            }
        } while (r == null);

        int recIdx = r.nextReceiver++;
        r.receiverTime = SystemClock.uptimeMillis();
        if (recIdx == 0) {
            //设置分发时间戳
            r.dispatchTime = r.receiverTime;
            r.dispatchClockTime = System.currentTimeMillis();
            ...
        }
        if (!mPendingBroadcastTimeoutMessage) { //如果没有post超时消息，就执行post
            long timeoutTime = r.receiverTime + mTimeoutPeriod;
            setBroadcastTimeoutLocked(timeoutTime);
        }
        ...
        //取出接收者
        final Object nextReceiver = r.receivers.get(recIdx); 

        //给动态注册的接收者分发广播
        if (nextReceiver instanceof BroadcastFilter) { //判断是不是动态注册的接收者
            BroadcastFilter filter = (BroadcastFilter)nextReceiver;
            deliverToRegisteredReceiverLocked(r, filter, r.ordered, recIdx); //直接分发广播
            ...
            return; 
        }

        //走到这，就证明是静态注册的接收者
        ResolveInfo info = (ResolveInfo)nextReceiver;
        ...
        //获取当前要分发的接收者的进程
        ProcessRecord app = mService.getProcessRecordLocked(targetProcess, info.activityInfo.applicationInfo.uid, false);
        ...
        r.state = BroadcastRecord.APP_RECEIVE;
        ...

        if (app != null && app.thread != null && !app.killed) { //检查进程是否启动
            ...
            processCurBroadcastLocked(r, app, skipOomAdj); //分发广播给该进程
        }

        //如果进程还没有启动，执行AMS#startProcessLocked启动进程
        //== null：如果进程启动失败了，就放弃给这个接收者广播。
        if ((r.curApp = mService.startProcessLocked(...一大堆参数)) == null) {
            ...
            finishReceiverLocked(r, r.resultCode, r.resultData, r.resultExtras, r.resultAbort, false);
            scheduleBroadcastsLocked();
            r.state = BroadcastRecord.IDLE;
            return;
        }

        //进程不会立刻启动完的，这里设置下pending广播
        //等进程启动完成之后，再处理这个pending广播
        mPendingBroadcast = r;
        mPendingBroadcastRecvIndex = recIdx;
    }

```



##### BroadcastQueue#deliverToRegisteredReceiverLocked

```java
private void deliverToRegisteredReceiverLocked(BroadcastRecord r, BroadcastFilter filter, boolean ordered, int index) {
        ...
        try {
            ...
            performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver, new Intent(r.intent), r.resultCode, r.resultData,
                        r.resultExtras, r.ordered, r.initialSticky, r.userId);
            if (ordered) { //如果是有序广播，状态设置为CALL_DONE_RECEIVE
                r.state = BroadcastRecord.CALL_DONE_RECEIVE;
            }
        } catch (RemoteException e) { ... }
    }
```



##### BroadcastQueue#performReceiveLocked

```java
void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver, Intent intent, int resultCode, String data, Bundle extras,boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        if (app != null) { //app已经启动
            if (app.thread != null) {
                try { 
                    //IApplicationThread#scheduleRegisteredReceiver
                    //为什么不直接调用IIntentReceiver#performReceive呢？为了让广播在app端进行串行调用
                    app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode, data, extras, ordered, sticky, sendingUser, app.repProcState);
                } catch (RemoteException ex) {...}
            }
            ...
        } else { //app还没启动
            receiver.performReceive(intent, resultCode, data, extras, ordered, sticky, sendingUser);
        }
    }
```



##### app端动态的接收者接收广播逻辑

```java
//ActivityThread#scheduleRegisteredReceiver
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                int resultCode, String dataStr, Bundle extras, boolean ordered,
                boolean sticky, int sendingUser, int processState) throws RemoteException {
            updateProcessState(processState, false);
            receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                    sticky, sendingUser);
        }

//IIntentReceiver实现类是LoadedApk.ReceiverDispatcher.InnerReceiver
//InnerReceiver#performReceive 又调用了 ReceiverDispatcher#performReceive
public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            final Args args = new Args(intent, resultCode, data, extras, ordered, sticky, sendingUser);
            ...
            if (intent == null || !mActivityThread.post(args.getRunnable())) { //post Runnable
                ...
            }
        }

//最终在这里执行onReceive
final class Args extends BroadcastReceiver.PendingResult {
            ...

            public final Runnable getRunnable() {
                return () -> {
                    ...
                    try {
                        ClassLoader cl = mReceiver.getClass().getClassLoader();
                        intent.setExtrasClassLoader(cl);
                        intent.prepareToEnterProcess();
                        setExtrasClassLoader(cl);
                        receiver.setPendingResult(this);
                        receiver.onReceive(mContext, intent); //call onReceive
                    } catch (Exception e) { ... }

                    if (receiver.getPendingResult() != null) {
                        finish(); //AMS#finishReceiver
                    }
                };
            }
        }        
```



##### app端静态的接收者接收广播逻辑

```java
//AMS#processCurBroadcastLocked
private final void processCurBroadcastLocked(BroadcastRecord r, ProcessRecord app, boolean skipOomAdj) throws RemoteException {
        ...
        try {
            ...
            app.thread.scheduleReceiver(new Intent(r.intent), r.curReceiver,
                    mService.compatibilityInfoForPackageLocked(r.curReceiver.applicationInfo),
                    r.resultCode, r.resultData, r.resultExtras, r.ordered, r.userId,
                    app.repProcState);
            ...
        } finally { ... }
    }

//ActivityThread#scheduleReceiver
public final void scheduleReceiver(Intent intent, ActivityInfo info,
                CompatibilityInfo compatInfo, int resultCode, String data, Bundle extras,
                boolean sync, int sendingUser, int processState) {
            ...
            ReceiverData r = new ReceiverData(intent, resultCode, data, extras,
                    sync, false, mAppThread.asBinder(), sendingUser);
            ...
            sendMessage(H.RECEIVER, r);
        }

//sendMsg RECEIVER -> ActivityThread#handleReceiver
private void handleReceiver(ReceiverData data) {
        ...
        LoadedApk packageInfo = getPackageInfoNoCheck( data.info.applicationInfo, data.compatInfo);
        IActivityManager mgr = ActivityManager.getService();
        Application app;
        BroadcastReceiver receiver;
        ContextImpl context;
        try {
            app = packageInfo.makeApplication(false, mInstrumentation); //如果Application不存在，就创建并返回
            context = (ContextImpl) app.getBaseContext();
            ...
            java.lang.ClassLoader cl = context.getClassLoader();
            data.intent.setExtrasClassLoader(cl);
            data.intent.prepareToEnterProcess();
            data.setExtrasClassLoader(cl);
            ...
            //反射创建广播对象
            receiver = packageInfo.getAppFactory().instantiateReceiver(cl, data.info.name, data.intent);
        } catch (Exception e) { ... }

        try {
            ...
            receiver.setPendingResult(data);
          	//回调onReceive方法，这里Context是直属于Application
            receiver.onReceive(context.getReceiverRestrictedContext(), data.intent);
        } catch (Exception e) { ... } 

        if (receiver.getPendingResult() != null) {
            data.finish(); //AMS#finishReceiver
        }
    }
```



##### app端启动进程后通知AMS自己可以接收广播了

```java
//ActivityThread#main -> ActivityThread#attach -> AMS#attachApplication
//AMS#attachApplication -> AMS#sendPendingBroadcastsLocked -> BroadcastQueue#sendPendingBroadcastsLocked

//BroadcastQueue#sendPendingBroadcastsLocked
public boolean sendPendingBroadcastsLocked(ProcessRecord app) {
        final BroadcastRecord br = mPendingBroadcast;
        if (br != null && br.curApp.pid > 0 && br.curApp.pid == app.pid) {
            ...
            try {
                mPendingBroadcast = null;
                processCurBroadcastLocked(br, app, false); //执行分发广播，这里走也是静态的接收者接收广播逻辑
            } catch (Exception e) {
                ...
                finishReceiverLocked(br, br.resultCode, br.resultData, br.resultExtras, br.resultAbort, false);
                scheduleBroadcastsLocked();
                ...
            }
        }
        ...
    }
```



##### app端通知AMS自己已经收到了广播

```java
//ReceiverData#finish（ReceiverData继承自BroadcastReceiver.PendingResult）
//BroadcastReceiver.PendingResult#finish
public final void finish() {
          final IActivityManager mgr = ActivityManager.getService();
            ...
            sendFinished(mgr);
            ...
        }

//BroadcastReceiver.PendingResult#sendFinished
public void sendFinished(IActivityManager am) {
            synchronized (this) {
                ...
                try {
                    ...
                    if (mOrderedHint) { //如果是有序广播
                      	//mAbortBroadcast：该广播是否已经被拦截
                      	//调用了abortBroadcast()方法，mAbortBroadcast就是为true
                        am.finishReceiver(mToken, mResultCode, mResultData, mResultExtras, mAbortBroadcast, mFlags);
                    } else {
                        am.finishReceiver(mToken, 0, null, null, false, mFlags);
                    }
                } catch (RemoteException ex) { }
            }
        }

//AMS#finishReceiver
public void finishReceiver(IBinder who, int resultCode, String resultData, Bundle resultExtras, boolean resultAbort, int flags) {
        ...
        final long origId = Binder.clearCallingIdentity();
        try {
            boolean doNext = false;
            BroadcastRecord r;
            synchronized(this) {
                BroadcastQueue queue = (flags & Intent.FLAG_RECEIVER_FOREGROUND) != 0 ? mFgBroadcastQueue : mBgBroadcastQueue;
                r = queue.getMatchingOrderedReceiver(who);
                if (r != null) {
                    //finishReceiverLocked：重置广播的状态
                    //返回值：return state == BroadcastRecord.APP_RECEIVE || state == BroadcastRecord.CALL_DONE_RECEIVE;
                    //有序广播在分发到app进程前，会把状态设置为CALL_DONE_RECEIVE
                    doNext = r.queue.finishReceiverLocked(r, resultCode, resultData, resultExtras, resultAbort, true);
                }
                if (doNext) { //为true就执行分发给下一个接收者
                    r.queue.processNextBroadcastLocked(/*fromMsg=*/ false, /*skipOomAdj=*/ true);
                }
                ...
            }
        } ...
    }

//BroadcastQueue#finishReceiverLocked：主要逻辑是重置一些状态
public boolean finishReceiverLocked(BroadcastRecord r, int resultCode, String resultData, Bundle resultExtras, boolean resultAbort, boolean waitForServices) {
        final int state = r.state;
        final ActivityInfo receiver = r.curReceiver;
        r.state = BroadcastRecord.IDLE;
        r.receiver = null;
        r.intent.setComponent(null);
        if (r.curApp != null && r.curApp.curReceivers.contains(r)) {
            r.curApp.curReceivers.remove(r);
        }
        if (r.curFilter != null) {
            r.curFilter.receiverList.curBroadcast = null;
        }
        r.curFilter = null;
        r.curReceiver = null;
        r.curApp = null;
        mPendingBroadcast = null;

        r.resultCode = resultCode;
        r.resultData = resultData;
        r.resultExtras = resultExtras;
  			//将广播拦截标志位，赋值给
        if (resultAbort && (r.intent.getFlags()&Intent.FLAG_RECEIVER_NO_ABORT) == 0) {
            r.resultAbort = resultAbort;
        } else {
            r.resultAbort = false;
        }

        ...

        r.curComponent = null;

        return state == BroadcastRecord.APP_RECEIVE || state == BroadcastRecord.CALL_DONE_RECEIVE;
    }
```



## 发送广播

```java
ContextImpl#sendBroadcast -> ActivityManager#getService#broadcastIntent -> 
```



## 本地广播

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

```

