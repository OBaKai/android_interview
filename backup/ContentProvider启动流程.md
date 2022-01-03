##  ContentProvider启动流程

####  总结

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



####  源码分析

##### 调用者进程

```java
//使用
ContentResolver resolver = context.getContentResolver();
resolver.insert(...);
resolver.delete(...);
resolver.update(...);
resolver.query(...);

//ContentResolver：ContextImpl#getContentResolver -> ContextImpl#mContentResolver
private ContextImpl(...) {
    ...
		mContentResolver = new ApplicationContentResolver(this, mainThread);
}

//ContentResolver#insert -> IContentProvider#insert（根据url获取binder对象）
public final @Nullable Uri insert(@RequiresPermission.Write @NonNull Uri url, @Nullable ContentValues values) {
        ...
        IContentProvider provider = acquireProvider(url); //获取Provider的Binder对象
        ...
        try {
            ...
            Uri createdRow = provider.insert(mPackageName, url, values);
            ...
            return createdRow;
        } catch (RemoteException e) { ... }
    }
```



###### 向AMS请求ContentProviderHodler

```java
ApplicationContentResolver#acquireProvider -> ActivityThread#acquireProvider -> AMS#getContentProvider -> AMS#getContentProviderImpl -> 返回ContentProviderHodler

public final IContentProvider acquireProvider(Context c, String auth, int userId, boolean stable) {
        //查询应用端本地是否已经存在了这个ContentProvider记录
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) { //本地有ContentProvider记录就直接
            return provider;
        }
        ContentProviderHolder holder = null;
        try { //本地没有就向AMS请求，AMS返回的是一个Holder对象，并不是IContentProvider
            holder = ActivityManager.getService().getContentProvider(getApplicationThread(), auth, userId, stable);
        } catch (RemoteException ex) { ... }
        ...
        //根据AMS返回的Holder对象，在应用端本地安装一个ContentProvider记录
        holder = installProvider(c, holder, holder.info, true, holder.noReleaseNeeded, stable);
        return holder.provider;
    }

 
public final IContentProvider acquireExistingProvider(Context c, String auth, int userId, boolean stable) {
        synchronized (mProviderMap) {
            final ProviderKey key = new ProviderKey(auth, userId); //构建key对象
            final ProviderClientRecord pr = mProviderMap.get(key); //用key到mProviderMap里边找
            if (pr == null) { //找不到就证明本地没有
                return null;
            }

            IContentProvider provider = pr.mProvider;
            IBinder jBinder = provider.asBinder();
            if (!jBinder.isBinderAlive()) { //找到也需要判断下这个Binder对象是否已经死了
                handleUnstableProviderDiedLocked(jBinder, true);
                return null;
            }
            ...
            return provider;
        }
    }
```



###### 安装Provider

```java
private ContentProviderHolder installProvider(Context context,ContentProviderHolder holder, ProviderInfo info,boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;
        //ContentProvider进程运行在本地安装ContentProvider对象，这里需要在本地安装ContentProvider
        if (holder == null || holder.provider == null) { 
            Context c = null; //Provider进程的Context
            ApplicationInfo ai = info.applicationInfo;
            if (context.getPackageName().equals(ai.packageName)) {
                c = context;
            } else if (mInitialApplication != null && mInitialApplication.getPackageName().equals(ai.packageName)) {
                c = mInitialApplication;
            } else {
                try { //根据Provider进程包名创建Context
                    c = context.createPackageContext(ai.packageName,Context.CONTEXT_INCLUDE_CODE);
                } catch (PackageManager.NameNotFoundException e) { }
            }
            ...

            try {
                final java.lang.ClassLoader cl = c.getClassLoader();
                LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
                if (packageInfo == null) {
                    packageInfo = getSystemContext().mPackageInfo;
                }
                //instantiateProvider：(ContentProvider) cl.loadClass(className).getDeclaredConstructor().newInstance();
                //反射创建ContentProvider对象
                localProvider = packageInfo.getAppFactory().instantiateProvider(cl, info.name);
                provider = localProvider.getIContentProvider();
                if (provider == null) { return null; }
              	//attachInfo：给Provider赋予上下文 并且 调用ContentProvider#onCreate
                localProvider.attachInfo(c, info); 
            } catch (java.lang.Exception e) { ... }
        } 
        //AMS返回有Binder对象
        else {
            provider = holder.provider;
        }

        ...  //将ContentProvider加入到缓存里边

        return retHolder;
    }
```



##### AMS#ContentProviderHolder

###### AMS返回ContentProviderHolder的逻辑

```java
//ProviderInfo info：全局变量，记录着提供者在清单文件中配置的信息
//(info.multiprocess || info.processName.equals(app.processName))：要么支持多进程，要么跟调用者进程名称相同
//uid == app.info.uid：支持者的uid要跟调用者的uid相同
public boolean canRunHere(ProcessRecord app) {
    return (info.multiprocess || info.processName.equals(app.processName)) && uid == app.info.uid;
}

//getContentProviderImpl逻辑：
//如果ContentProviderRecord存在，就直接返回
//如果ContentProviderRecord不存在，创建一个·  
//ContentProvider能跑在调用者进程，直接返回，否则继续往下走
//ContentProvider所在进程没有启动，就先启动进程，然后等待发布Binder对象，完成的时候返回
//ContentProvider所在进程已经启动，请求发布Binder对象，完成的时候返回
private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
            String name, IBinder token, boolean stable, int userId) {
        ContentProviderRecord cpr;
        ContentProviderConnection conn = null;
        ProviderInfo cpi = null;

        synchronized(this) {
            ProcessRecord r = null;
            ...
            cpr = mProviderMap.getProviderByName(name, userId);
            ...

            boolean providerRunning = cpr != null && cpr.proc != null && !cpr.proc.killed;
            //处理提供者的进程存在的情况
            if (providerRunning) { 
                cpi = cpr.info;
                String msg;
                
                //判断提供者能否运行在调用者进程
                if (r != null && cpr.canRunHere(r)) {
                    ContentProviderHolder holder = cpr.newHolder(null);
                    holder.provider = null; //将holder的provider置空，这样调用者就会在本地自己创建一个provider实例来用。
                    return holder;
                }

                ...
            }

            //处理提供者进程不存在的情况 
            if (!providerRunning) {
                ...
                cpr = new ContentProviderRecord(this, cpi, ai, comp, singleton);

                //再次判断能否运行在调用者进程，如果可以直接return。连提供者进程都不需要启动了。 
                if (r != null && cpr.canRunHere(r)) { 
                    return cpr.newHolder(null);
                }

                //正在发布的支持者进程队列，还在等着它们给AMS发布提供者的Binder对象
                final int N = mLaunchingProviders.size();
                int i;
                for (i = 0; i < N; i++) {
                    if (mLaunchingProviders.get(i) == cpr) {
                        break;
                    }
                }

                if (i >= N) {
                    try {
                       ...
                        ProcessRecord proc = getProcessRecordLocked(cpi.processName, cpr.appInfo.uid, false);
                        if (proc != null && proc.thread != null && !proc.killed) { //如果提供者进程已经存在，那么就请求提供者的binder对象
                            if (!proc.pubProviders.containsKey(cpi.name)) {
                                proc.pubProviders.put(cpi.name, cpr);
                                try { //向提供者进程请求Binder对象
                                    proc.thread.scheduleInstallProvider(cpi);
                                } catch (RemoteException e) { }
                            }
                        } else { //提供者进程并未启动，启动进程
                            proc = startProcessLocked(cpi.processName, cpr.appInfo, false, 0, "content provider",
                                    new ComponentName(cpi.applicationInfo.packageName,cpi.name), false, false, false);
                        }
                        cpr.launchingApp = proc;
                        mLaunchingProviders.add(cpr); //由于还没获得Binder对象，就先加入这个队列
                    } finally { ... }
                ...
            }
        }

        synchronized (cpr) {
            //一直等待提供者进程发布Binder对象
            while (cpr.provider == null) {
                ...
                try {
                    if (conn != null) {
                        conn.waiting = true;
                    }
                    cpr.wait();
                } catch (InterruptedException ex) { ... } finally { ... }
            }
        }
        //返回给调用者
        return cpr != null ? cpr.newHolder(conn) : null;
    }
```





##### Provider进程发布Binder对象的两种方式

###### 1、提供者进程在启动时主动发布Provider的Binder对象

```java
//AMS#attachApplicationLocked
private final boolean attachApplicationLocked(IApplicationThread thread,int pid, int callingUid, long startSeq) {
        ...
        //从PMS那拿到提供者的注册信息
        List<ProviderInfo> providers = generateApplicationProvidersLocked(app);
        //兜底的超时消息，防止提供者进程发布的Binder对象超时。 
        if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
            Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
            msg.obj = app;
            mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
        }

        try {
            ...
            //告诉应用端需要安装这些Provider
            //在安装过程中，这些Provider就会把Binder对象发布到AMS
            thread.bindApplication(processName, appInfo, providers, ...);
            
        } catch (Exception e) { ... }
       ...
    }

//ActivityThread#handleBindApplication
private void handleBindApplication(AppBindData data) {
        ...
        Application app;
        try {
          	//makeApplication：创建Application实例 并且调用Application#attachBaseContext
            app = data.info.makeApplication(data.restrictedBackupMode, null);

            mInitialApplication = app;

            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                  	//安装Provider们
                    installContentProviders(app, data.providers);
                    ...
                }
            }
            ...
            try {
              	//调用Application#onCreate
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) { ... }
        } finally { ... }
        ...
    }

//ActivityThread#installContentProviders
private void installContentProviders(Context context, List<ProviderInfo> providers) {
        final ArrayList<ContentProviderHolder> results = new ArrayList<>();

        //遍历安装这些Provider
        for (ProviderInfo cpi : providers) {
            ContentProviderHolder cph = installProvider(context, null, cpi, false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph != null) {
                cph.noReleaseNeeded = true;
                results.add(cph);
            }
        }

        try { //然后将这些安装好的Provider，发布到AMS
            ActivityManager.getService().publishContentProviders(getApplicationThread(), results);
        } catch (RemoteException ex) { ... }
    }

//AMS#publishContentProviders
public final void publishContentProviders(IApplicationThread caller, List<ContentProviderHolder> providers) {
        synchronized (this) {
            ...
            for (int i = 0; i < N; i++) {
                ContentProviderHolder src = providers.get(i);
                ContentProviderRecord dst = r.pubProviders.get(src.info.name);
                ...
                if (dst != null) {
                    ...
                    if (wasInLaunchingProviders) { //移除兜底的超时消息
                        mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
                    }
                    synchronized (dst) {
                        dst.provider = src.provider;
                        dst.proc = r;
                        //唤醒一直在等待这个Provider的Binder对象的调用者进程们
                        //AMS在发现Binder对象没有发布的时候，会调用wait()等待Binder对象的发布
                        //那么调用者与AMS之间是用过binder线程连接的，所以wait()是会一直阻塞那条binder线程的。直到notifyAll()。
                        dst.notifyAll();
                    }
                    ...
                }
            }
        }
    }
```



###### 2、AMS向提供者进程申请Provider的Binder对象

```java
AMS（proc.thread.scheduleInstallProvider）-> ActivityThread#scheduleInstallProvider -> sendMsg(INSTALL_PROVIDER) -> ActivityThread#handleInstallProvider 
-> ActivityThread#installContentProviders -> AMS#publishContentProviders
```

