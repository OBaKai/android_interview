## start方式的启动流程

#### 总结

```java
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
```



#### 源码分析

##### startService -> Service#onCreate

```java
onCreate：
ContextImpl#startService -> ContextImpl#startServiceCommon -> AMS#startService -> ActiveServices#startServiceLocked -> ActiveServices#startServiceInnerLocked -> ActiveServices#bringUpServiceLocked -> ActiveServices#realStartServiceLocked ->（app）ActivityThread#scheduleCreateService（sendMsg CREATE_SERVICE）-> ActivityThread#handleCreateService -> onCreate

关键点：
ActiveServices类：AMS里边有一个对象mServices（ActiveServices），ActiveServices类是负责Service相关的逻辑，包括启动，停止，和绑定，以及重启生命周期的调用等。

ActiveServices#bringUpServiceLocked 解析：
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
        //ServiceRecord#app不为空（ServiceRecord#app是在Service启动后才赋值的），证明这个Service已经启动了
        if (r.app != null && r.app.thread != null) {
            //给Service走onStartCommand
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }
        ...
        ProcessRecord app;
        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (app != null && app.thread != null) { //判断app进程是否已经启动 同时 判断app进程是否已经就绪
                try {
                    ...
                    realStartServiceLocked(r, app, execInFg); //真正启动Service的方法
                    return null;
                } catch (TransactionTooLargeException e) { ... } catch (RemoteException e) { ... }
            }
        } else { ... }

        //如果app进程没启动
        if (app == null && !permissionsReviewRequired) {
            //启动app进程
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,  hostingType, r.name, false, isolated, false)) == null) {
                ...
            }
            ...
        }

        ...
        //将ServiceRecord加入到等待启动列表里边
        //当app进程启动完成，就会去启动这个等待列表的ServiceRecord
        //AMS#attachApplicationLocked -> ActiveServices#attachApplicationLocked -> 遍历mPendingServices，给每个item执行realStartServiceLocked
        if (!mPendingServices.contains(r)) { 
            mPendingServices.add(r);
        }

        ...

        return null;
    }
  
ActiveServices#realStartServiceLocked 解析：在进程启动好的前提下，启动Service。负责分发onCreate、onBind、onStartCommand等生命周期
private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app, boolean execInFg) throws RemoteException {
        ...
        try {
            ...
            //分发onCreate生命周期
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            ...
        } catch (DeadObjectException e) { ... } finally { ... }
        ...
        requestServiceBindingsLocked(r, execInFg); //分发onBind生命周期
        ...
        sendServiceArgsLocked(r, execInFg, true); //分发onStartCommand生命周期
        ...
    }

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
        	//将Service缓存到一个map里边，key就是ServiceRecord
          mServices.put(data.token, service);
	        ...
	        try {
	            ActivityManager.getService().serviceDoneExecuting(
	                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
	        } catch (RemoteException e) {}
	    } catch (Exception e) {}
	}
```



##### Service#onStartCommand

```java
ActiveServices#sendServiceArgsLocked -> (app) ActivityThread#scheduleServiceArgs（sendMsg SERVICE_ARGS）-> ActivityThread.H#handleMessage（SERVICE_ARGS）-> ActivityThread#handleServiceArgs -> onStartCommand
  
ActiveServices#startServiceLocked 解析：ServiceRecord#pendingStarts专门处理onStartCommand的
ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
            throws TransactionTooLargeException {
        ...

        r.lastActivity = SystemClock.uptimeMillis();
        r.startRequested = true;
        r.delayedStop = false;
        r.fgRequired = fgRequired;
        //pendingStarts作用是之后准备调用onStartCommand用的
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(), service, neededGrants, callingUid));

        ...

        ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
        return cmp;
    }

ActiveServices#sendServiceArgsLocked 解析：
private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        final int N = r.pendingStarts.size();
        if (N == 0) { //pendingStarts为空就不处理
            return;
        }
  			...
				while (r.pendingStarts.size() > 0) {
            ServiceRecord.StartItem si = r.pendingStarts.remove(0);
            ...
        }
        ...
        try {
            r.app.thread.scheduleServiceArgs(r, slice);
        } catch (TransactionTooLargeException e) { ... } 
        ...
    }

ActivityThread#handleServiceArgs 解析：
 private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token); //根据ServiceRecord从map中取出Service的缓存
        if (s != null) {
            try {
               ...
                int res;
                if (!data.taskRemoved) {
                    res = s.onStartCommand(data.args, data.flags, data.startId);
                } else {
                    s.onTaskRemoved(data.args);
                    res = Service.START_TASK_REMOVED_COMPLETE;
                }
                ...
            } catch (Exception e) { ... }
        }
    }
```





## bind方式的启动流程

#### 总结

```java
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



#### 源码分析

##### ServiceRecord结构

```java
ServiceRecord
  变量bindings：ArrayMap<Intent.FilterComparison, IntentBindRecord>
    IntentBindRecord
      变量apps：ArrayMap<ProcessRecord, AppBindRecord>
        AppBindRecord
          变量connections：ArraySet<ConnectionRecord>
  
ServiceRecord包含多个IntentBindRecord
  Service可以包含多个Intent，因为可以通过不用的Intent绑定到同一个Service
IntentBindRecord包含多个AppBindRecord
  同一个Intent可能来自一个进程，或者多个进程
AppBindRecord包含多个ConnectionRecord
  同一个进程中可能有多个地方绑定这个Service
```



##### bindService -> Service#onCreate

```java
//bindService中Service的启动流程 跟 startService的Service启动流程基本是一样的
ContextImpl#bindService -> ContextImpl#bindServiceCommon -> AMS#bindService -> ActiveServices#bindServiceLocked -> ActiveServices#bringUpServiceLocked（这里开始与startService流程一直了）-> ActiveServices#realStartServiceLocked ->（app）ActivityThread#scheduleCreateService（sendMsg CREATE_SERVICE）-> ActivityThread.H#handleMessage（CREATE_SERVICE）-> ActivityThread#handleCreateService -> onCreate
```



##### ServiceConnection对象的传递

```java
//1、ServiceConnection对象并非Binder对象，真正的Binder对象是IServiceConnection。IServiceConnection接口的实现类持有了ServiceConnection，当IServiceConnection被调用了就会去调用ServiceConnection对象。
//2、bindService的过程中，会将IServiceConnection对象传给AMS。
//3、IServiceConnection对象是为了让AMS通知绑定进程是否连接上了服务进程。

//注意：ServiceConnection 与 IServiceConnection并不是一对一的关系
//ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>>
//不同Context，用相同的ServiceConnection去bindService，会出现新的IServiceConnection
//相同Context，用不用的ServiceConnection去bindService，也会出现新的IServiceConnection

private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
            handler, UserHandle user) {
        IServiceConnection sd;
        ...
        if (mPackageInfo != null) {
          	//LoadedApk#getServiceDispatcher
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
        } 
        ...
        try {
            ...
            int res = ActivityManager.getService().bindService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, getOpPackageName(), user.getIdentifier());
            ...
        } catch (RemoteException e) { ... }
    }

public final IServiceConnection getServiceDispatcher(ServiceConnection c, ...) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
            if (map != null) {
                sd = map.get(c);
            }
            if (sd == null) {
                sd = new ServiceDispatcher(c, context, handler, flags);
                if (map == null) {
                    map = new ArrayMap<>();
                    mServices.put(context, map);
                }
                map.put(c, sd);
            }
            ...
            return sd.getIServiceConnection();
        }
    }

static final class ServiceDispatcher { //LoadedApk#ServiceDispatcher
        private final ServiceDispatcher.InnerConnection mIServiceConnection;
        private final ServiceConnection mConnection;
        ...

        private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }

            public void connected(ComponentName name, IBinder service, boolean dead) throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    sd.connected(name, service, dead);
                }
            }
        }

        private final ArrayMap<ComponentName, ServiceDispatcher.ConnectionInfo> mActiveConnections
            = new ArrayMap<ComponentName, ServiceDispatcher.ConnectionInfo>();

        ServiceDispatcher(ServiceConnection conn, ...) {
            mIServiceConnection = new InnerConnection(this);
            mConnection = conn;
            ...
        }

        public void connected(ComponentName name, IBinder service, boolean dead) {
            if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0, dead));
            } else {
                doConnected(name, service, dead);
            }
        }

  			//IBinder service：服务的IBinder对象，不为null证明与Service绑定成功，为null证明与Service断开
        public void doConnected(ComponentName name, IBinder service, boolean dead) {
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;
            synchronized (this) {
              	//mActiveConnections作用是就缓存IBinder对象的 
                old = mActiveConnections.get(name);
                if (old != null && old.binder == service) {
                  	//有缓存过IBinder对象并且旧的与新的一样，回调了onServiceConnected
                  	//这里防止多次调用onServiceConnected
                    return;
                }

                if (service != null) {//绑定成功，缓存IBinder对象
                    info = new ConnectionInfo();
                    info.binder = service;
                    info.deathMonitor = new DeathMonitor(name, service);
                    try {
                      	//监听IBinder对象的死期
                      	//onServiceDisconnected：解绑是不会回调的，只有服务挂了才会回调。
                        service.linkToDeath(info.deathMonitor, 0);
                        mActiveConnections.put(name, info);
                    } catch (RemoteException e) {
                        mActiveConnections.remove(name);
                        return;
                    }
                } else { //连接断开，移除IBinder对象的缓存
                    mActiveConnections.remove(name);
                }

                if (old != null) { //如果有旧的绑定，那么接触死亡监听，因为IBinder对象更新了
                    old.binder.unlinkToDeath(old.deathMonitor, 0);
                }
            }

            if (old != null) { //如果有旧的绑定，那么就通知断开，因为IBinder对象更新了
                mConnection.onServiceDisconnected(name);
            }
            ...
            if (service != null) { //回调绑定成功
                mConnection.onServiceConnected(name, service);
            }
          	...
        }
  
				public void death(ComponentName name, IBinder service) {
            if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 1, false));
            } else {
                doDeath(name, service);
            }
        }
  
        public void doDeath(ComponentName name, IBinder service) {
            synchronized (this) {
                ConnectionInfo old = mActiveConnections.get(name);
                if (old == null || old.binder != service) {
                    return;
                }
                mActiveConnections.remove(name);
                old.binder.unlinkToDeath(old.deathMonitor, 0);
            }
            mConnection.onServiceDisconnected(name);
        }

        private final class RunConnection implements Runnable {
            ...
            public void run() {
                if (mCommand == 0) {
                    doConnected(mName, mService, mDead);
                } else if (mCommand == 1) {
                    doDeath(mName, mService);
                }
            }
        }

        private final class DeathMonitor implements IBinder.DeathRecipient{
            ...
            public void binderDied() { //当IBinder对象死亡的时候，才会被AMS调用
              	//death里边回调onServiceDisconnected
                death(mName, mService);
            }
        }
    } 
```





##### Service#onBind

```java
onBind生命周期执行流程：
ActiveServices#realStartServiceLocked -> ActiveServices#requestServiceBindingsLocked（只有bindService才执行） -> ActiveServices#requestServiceBindingLocked -> （app）ActivityThread#scheduleBindService（sendMsg BIND_SERVICE） -> ActivityThread#handleBindService -> onBind或onRebind

关键点：
ActiveServices#bindServiceLocked 解析：
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, final IServiceConnection connection, int flags,
            String callingPackage, final int userId) throws TransactionTooLargeException {
        ...
        try {
            ...
            if ((flags&Context.BIND_AUTO_CREATE) != 0) { //如果Service支持绑定拉起
                ...
                //拉起Service
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false, permissionsReviewRequired) != null) {
                    return 0;
                }
            }
            ...
            //Service已经启动 并且 Service的Binder对象已经发布给AMS了
            if (s.app != null && b.intent.received) { 
                try {
                    //将Service的Binder对象传给绑定进程
                    c.conn.connected(s.name, b.intent.binder, false);
                } catch (Exception e) { ... }
                ...
                //判断是否要调用Service的onRebind
                //条件1：你是第一个用这个Intent绑定Service的人
                //条件2：Intent的doRebind标志是否开启
                if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                    requestServiceBindingLocked(s, b.intent, callerFg, true);
                }
            } else if (!b.intent.requested) { //AMS还没有向Service请求Binder对象，马上去请求Binder对象
                requestServiceBindingLocked(s, b.intent, callerFg, false);
            }
        }
        ...
    }

ActiveServices#realStartServiceLocked 解析：在进程启动好的前提下，启动Service。负责分发onCreate、onBind、onStartCommand等生命周期
private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app, boolean execInFg) throws RemoteException {
        ...
        try {
            ...
            //分发onCreate生命周期
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            ...
        } catch (DeadObjectException e) { ... } finally { ... }
        ...
        requestServiceBindingsLocked(r, execInFg); //AMS向Service请求Binder对象（分发onBind生命周期）
        ...
    }  

ActiveServices#requestServiceBindingsLocked 解析：AMS向Service请求Binder对象
private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)throws TransactionTooLargeException {
		//只有bindService的时候，r.bindings才有值
	    for (int i=r.bindings.size()-1; i>=0; i--) {
	        IntentBindRecord ibr = r.bindings.valueAt(i);
	        if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
	            break;
	        }
	    }
	}

ActiveServices#requestServiceBindingLocked 解析：
private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i, boolean execInFg, boolean rebind) throws TransactionTooLargeException {
	    ...
      //如果AMS没有请求过Binder对象 或者 请求过但是触发了reBind的话
      //i.apps.size() > 0：当前有app要绑定Service
	    if ((!i.requested || rebind) && i.apps.size() > 0) {
	        try {
	            ...
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

ActivityThread#handleBindService 解析：scheduleBindService -> handleBindService
	private void handleBindService(BindServiceData data) {
	    Service s = mServices.get(data.token); //获取Service对象
	    if (s != null) {
	        try {
	           	...
	            try {
	                if (!data.rebind) { //判断是否为重新绑定
	                    //走onBind生命周期，并且获取返回值IBinder对象
	                    IBinder binder = s.onBind(data.intent);
	                    //将Binder对象发布到AMS
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
```



##### Service中Binder对象的发布（回调ServiceConnection#onServiceConnected）

```java
在onBind生命周期回调之后，会调用AMS#publishService，将Binder对象发布到AMS。
AMS#publishService -> ActiveServices#publishServiceLocked -> （app）ConnectionRecord.InnerConnection（继承自IServiceConnection）#connected -> LoadedApk.ServiceDispatcher#connected（post runnable） -> LoadedApk.ServiceDispatcher#doConnected -> ServiceConnection#onServiceConnected

关键点：
AMS#publishServiceLocked 解析：
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    final long origId = Binder.clearCallingIdentity();
    try {
        if (r != null) {
            Intent.FilterComparison filter = new Intent.FilterComparison(intent);
            IntentBindRecord b = r.bindings.get(filter); //找到IntentBindRecord对象
            if (b != null && !b.received) {
                b.binder = service; //赋值binder对象
                b.requested = true; //请求binder对象，标志位打开
                b.received = true; //收到binder对象，标志位打开
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
                        } catch (Exception e) { ... }
                    }
                }
            }
        }
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



##### Service#onRebind

```java
onRebind的调用时机：在Service#onUnbind方法返回true的情况下，应用进程解绑了这个Service后，再次使用相同的Intent绑定Service，就会回调onRebind。（注意：走onRebind了，Service就不会走onBind）

//在Service执行onBind的生命周期的过程中
//ActivityServices#bindServiceLocked
int bindServiceLocked(...) throws TransactionTooLargeException {
        ...
        try {
            ...
            //Service已经启动 并且 Service的Binder对象已经发布给AMS了
            if (s.app != null && b.intent.received) { 
                try {
                    //将Service的Binder对象传给绑定进程
                    c.conn.connected(s.name, b.intent.binder, false);
                } catch (Exception e) { ... }
                ...
                //判断是否要调用Service的onRebind
                //条件1：你是第一个用这个Intent绑定Service的人
                //条件2：Intent的doRebind标志是否开启
                if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                  	//最后一个传参（boolean rebind）为true，那么Service就会回调onReBind
                    requestServiceBindingLocked(s, b.intent, callerFg, true);
                }
            } else if (!b.intent.requested) { //AMS还没有向Service请求Binder对象，马上去请求Binder对象
                requestServiceBindingLocked(s, b.intent, callerFg, false);
            }
        }
        ...
    }

//那这个doRebind标志位是什么时候设置为true的呢？

//来看看unbindService操作
//ActiveServices#unbindServiceLocked
boolean unbindServiceLocked(IServiceConnection connection) {
        IBinder binder = connection.asBinder();
        ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
        ...
        try {
            while (clist.size() > 0) {
                ConnectionRecord r = clist.get(0);
                removeConnectionLocked(r, null, null);
                if (clist.size() > 0 && clist.get(0) == r) {
                    clist.remove(0);
                }
                ...
            }
        }
    }

//ActiveServices#removeConnectionLocked
void removeConnectionLocked(ConnectionRecord c, ProcessRecord skipApp, ActivityRecord skipAct) {
  			AppBindRecord b = c.binding;
        ...
        if (!c.serviceDead) {
          	//s.app != null && s.app.thread != null：Service还存在
          	//b.intent.apps.size() == 0：绑定Service的这个Intent下边已经没有绑定的应用进程了
            if (s.app != null && s.app.thread != null && b.intent.apps.size() == 0 
                && b.intent.hasBound) {
                try {
                    ...
                    b.intent.doRebind = false;
                    //告诉Service所在进程，执行Service的onUnbind回调
                    s.app.thread.scheduleUnbindService(s, b.intent.intent.getIntent());
                } catch (Exception e) { ... }
            }
            ...
        }
    }

//ActivityThread#scheduleUnbindService -> ActivityThread#handleUnbindService
private void handleUnbindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                ...
                //onUnbind返回值默认是false，除非我们重写了这个方法手动返回true
                boolean doRebind = s.onUnbind(data.intent);
                try {
                    if (doRebind) {
                        ActivityManager.getService().unbindFinished(
                                data.token, data.intent, doRebind);
                    } else {
                        ActivityManager.getService().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
                } catch (RemoteException ex) { ... }
            } catch (Exception e) { ... }
        }
    }

//AMS#unbindFinished -> ActivityServices#unbindFinishedLocked
void unbindFinishedLocked(ServiceRecord r, Intent intent, boolean doRebind) {
        try {
            if (r != null) {
                Intent.FilterComparison filter = new Intent.FilterComparison(intent);
                IntentBindRecord b = r.bindings.get(filter);
                boolean inDestroying = mDestroyingServices.contains(r);
                if (b != null) {
                    //removeConnectionLocked方法：IntentBindRecord#apps为0，才会走Service#onUnbind
                    //这里IntentBindRecord#apps已经是0了，所有会走else
                    if (b.apps.size() > 0 && !inDestroying) {
                        ...
                        try {
                            requestServiceBindingLocked(r, b, inFg, true);
                        } catch (TransactionTooLargeException e) { }
                    } else {
                        b.doRebind = true; //这里设置doRebind为true
                    }
                }
                ...
            }
        }
    }

//如果下次还有应用进程使用这个Intent来绑定Service，Service就会走onRebind回调
```

