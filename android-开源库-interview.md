#### JetPack

##### WorkManager

```java
WorkManager：负责管理后台任务（设计用意就是取代后台服务）
api23以下：使用AlarmManager + BroadcastReceiver实现
api23以上：使用JobScheduler实现

使用：https://blog.csdn.net/qq_36699930/article/details/109840041
Worker：任务的执行者，是一个抽象类，需要继承它实现要执行的任务。
WorkRequest：指定让哪个 Woker 执行任务，指定执行的环境，执行的顺序等。 要使用它的子类 OneTimeWorkRequest 或 PeriodicWorkRequest。
WorkManager：管理任务请求和任务队列，发起的 WorkRequest 会进入它的任务队列。
WorkStatus：包含有任务的状态和任务的信息，以 LiveData 的形式提供给观察者，更新相关UI。

特点：
强大的生命力：任务一旦加入工作队列。就算进程被杀、设备重启，任务依然会在满足条件的情况下得到执行。


原理总结：
1、初始化流程
在编译apk的时候在Manifest加入WorkManagerInitializer的ContentProvider，
在ContentProvider#onCreate里边执行WorkManager#initialize。
1.1、创建Room数据库保存配置信息（持久化保持）
1.2、创建GreedyScheduler调度器
1.3、检查并且执行发生意外的任务

2、执行流程
所有Work的操作都会更新到数据库，为了防止意外停止的时候可以重新读取数据库找到Work的操作进度重新执行。

2.1、无约束任务的执行：通过GreedyScheduler丢到线程池里边执行

2.2、带约束任务的实行：也是丢到线程池执行，但是带有约束条件
约束条件的实现：广播接受者 + 数据库轮询机

例如：work是受网络连接的约束
网络关闭 -> 网络关闭广播 -> 打断work -> 更新work进度到数据库
网络打开 -> 网络打开广播 -> 轮询机轮询数据库 -> 找到work -> 继续执行

广播接受者逻辑：（静态注册广播）
ConstrainProxy是一个广播接收者，ConstrainProxy有很多子类都对应着一种约束。
在编译apk的时候会自动在Manifest里边加入这些约束子类，用于接受对应的广播。

数据库轮询机：
某个约束子类收到广播后，就会启动一个SystemAlarmScheduler服务，这个服务就会去读数据库的找到对应要执行的work，丢到线程池里边执行。
```



##### LiveData

###### LiveData总结

```java
MediatorLiveData实现红点的统一管理：https://juejin.cn/post/6945419430176227359

LiveData：一个可被观察的数据持有者类
特点：
1、数据可以给观察者进行订阅；
2、能够感知组件（Fragment、Activity、Service）的生命周期；
3、只有在组件出于激活状态（STARTED、RESUMED）才会通知观察者有数据更新；

作用：
1、保证数据与界面的实时更新
LiveData采用了观察者模式设计，LiveData是被观察者，当数据发生变化时会通知观察者进行数据更新。
2、有效避免内存泄漏
LiveData能够感知组件的生命周期，它可以在组件处于激活状态下才通知观察者有数据更新；
当组件状态处于destoryed状态时，观察者对象会被remove；
比如Activity的生命周期，LiveData能确保仅在Activity处于活动状态下（生命周期处于onStart与onResume时）才会更新，也就是说当观察者处于活动状态，才会去通知数据更新，当生命周期处于onStop或者onPause时，不回调数据更新，直至生命周期为onResume时，立即回调。
3、Activity/Fragment销毁掉时不会引起崩溃
这是因为组件处于非激活状态时，在界面不会收到来自LiveData的数据变化通知。这样规避了很多因为页面销毁之后，修改UI导致的crash。
```



###### LiveData原理

```java
原理总结：LiveData即时观察者，也是被观察者
LiveData是观察者：能够观察UI组件的生命周期
Activity/Fragment 被观察者（订阅过程是通过Lifecycle机制实现的。）
LiveData是被观察者：能够被别人观察它里边的数据变化

都是在 LiveData#observe 方法完成订阅（双向绑定）


原理：
1、LiveData#observe：简单来说就是实现了双向绑定（实现对UI组件这个被观察者的订阅，以及实现数据观察者对LiveData这个被观察者的订阅）
LiveData#observe(LifecycleOwner owner（生命周期的被观察者）, Observer observer（数据的观察者）)
1.1、实现对UI组件这个被观察者的订阅
核心代码：
//LifecycleBoundObserver对象，实现了LifecycleObserver，所以可以通过Lifecycle#addObserver订阅UI组件的生命周期
//通过Lifecycle机制，收到UI组件的生命周期变化
LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
owner.getLifecycle().addObserver(wrapper);

1.2、实现数据观察者对LiveData这个被观察者的订阅
核心代码：
//LiveData里边维护了一个Observer容器（SafeIterableMap<Observer<? super T>, ObserverWrapper>）
//当数据变化的时候，就会通知到这个容器的所有观察者
LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);

2、LiveData#setValue：更新数据（就遍历LiveData里边维护了一个Observer容器，给执行每个Observer#onChanged）
2.1、细节1：激活状态的双重判断（确保UI是激活状态才更新数据）
      if (!observer.mActive) { return; } //判断Observer自己的激活标志位
      if (!observer.shouldBeActive()) { //判断UI组件这个被观察者是否处于激活状态
         observer.activeStateChanged(false);
         return;
      }
2.2、细节2、Version检查（防止重复回调）
      //mVersion在 LiveData#setValue 的时候 ++了
      //每次更新数据都会mVersion++。Observer里边也会维护自己的Version
      //如果Observer的Version已经跟mVersion一样了，也就是该Observer收到过数据了，不给它重复回调了
      if (observer.mLastVersion >= mVersion) { return; }
      observer.mLastVersion = mVersion;



```

###### LiveData粘性事件问题

```java
LiveData产生粘性事件的原因：
场景：ActivityA中LiveData#setValue(1)后，启动ActivityB后进行LiveData的订阅，ActivityB能够收到这个1的数据。
原因：Lifecycle机制
ActivityB启动后走完onStart生命周期，Lifecycle机制会给订阅它的观察者们更新生命周期状态。其中就包括了LiveData的这个观察者。
在onStart状态更新后LiveData就会去触发一次数据分发。
解决：通过反射更新一下observer的mLastVersion就行了
```



###### LiveData防递归设计

```java
LiveData防递归设计：
场景：
liveData.setValue(1); //① 更新数据1，分发逻辑是遍历容器给观察者a、b分发数据

//② 观察者a收到了数据1，就立马执行更新数据2
//③ 但是数据1的分发还没走完（还没走出遍历，也还没走完setValue），就再次触发更新数据（再次走setValue），形成递归
liveData.observe(this, (Observer<Integer>) integer -> {
      if (integer == 1){
         liveData.setValue(2);
      }
});
liveData.observe(this, (Observer<Integer>) integer -> { }); //观察者b

LiveData的解决：
void dispatchingValue(ObserverWrapper initiator) {
        //② 由于 mDispatchingValue 被数据1分发流程设置为true了，所以数据2分发流程进入if
        if (mDispatchingValue) {
            //③ 数据2分发流程把 mDispatchInvalidated 设置为true，然后return了
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do { //⑦ 进入数据2的分发流程
            mDispatchInvalidated = false;
            ...
            for (Iterator<...> iterator = mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                //① 数据1分发流程，给观察者a分发，但是观察者a这个老6，执行数据2更新
                ocnsiderNotify(iterator.next().getValue());
                //④ 由于数据2分发流程那边return了，这边走出来了。并且发现mDispatchInvalidated被数据2分发流程设置为ture了
                if (mDispatchInvalidated) {
                    //⑤ 直接跳出数据1分发流程的for循环，可怜的观察者b没机会收到数据1了
                    break;
                }
            }
        //⑥ mDispatchInvalidated被数据2分发流程设置为ture了，进入下一个do/while循环
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
}
```

###### postValue丢事件问题

```java
相关文章：https://zhuanlan.zhihu.com/p/461314535

问题：
postValue("aaa");
postValue("bbb");
实际只收到了bbb。期望应该先收到aaa再收到bbb

原因：postTask标志位作怪
1、执行postValue("aaa")
2、由于mPendingData是NOT_SET（Runnable执行会setValue，然后会让mPendingData会变成NOT_SET），所以postTask标志位为true
3、mPendingData = "aaa"
4、执行main thread postRunnable(mPostValueRunnable)，返回了。
5、执行postValue("bbb")
6、由于mPendingData不是NOT_SET（mPendingData现在是"aaa"，因为postRunnable是异步的，不是立即执行），所以postTask为false
7、mPendingData = "bbb"
8、postTask为false，直接返回了。不执行postRunnable。
  
...一段时间后，执行 mPostValueRunnable 这个Runnable
1、获取 mPendingData 值（值为"bbb"）
2、mPendingData设为NOT_SET
3、setValue((T) newValue)（走setValue逻辑分发事件）
  
最后导致只会收到最新的那个 postValue("bbb")
  
protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;
        }
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }

private final Runnable mPostValueRunnable = new Runnable() {
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
            setValue((T) newValue);
        }
    };

为什么LiveData这样设计？
我举得，LiveData强调的是Data，更关注数据，而不是关注过程。
关注数据那么就会稍微忽略过程，从 postValue 方法体现出来。
同一时间不管post多少次，只会为你保证能更新到最新的那个数据，中途的多次变化忽略。
这样设计的好处是，减轻主线程消息队列的消息处理压力。
```



##### ViewModel

```java
ViewModel：以注重生命周期的方式存储和管理界面相关的数据

作用：
1、保护数据不被销毁。
   屏幕旋转会造成Activity/Fragment发生重建，缓存的数据就会跟着丢失，ViewModel可以保证自身不被重建。
2、避免内存泄漏。
   Activity destroy时会调用ViewModel#onCleared()
3、用非常简单的方式，解决同一个Activity的不同Fragment的数据共享问题
   在不同Fragment中通过 ViewModelProvider(activity).get(XXViewModel.clss) 就能够获取到相同的ViewModel对象了

注意：ViewModel生命周期比Activity长，所以ViewModel中不能持有Activity引用
   
原理总结：
ViewModelStore对象都是间接存在ActivityClientRecord中的，而ActivityClientRecord是维护在ActivityThread的，只有Activity真正销毁ActivityThread才会移除对应的ActivityClientRecord。
在Activity配置变化时
	ViewModelStore会在Activity销毁前保存到ActivityClientRecord
	在Activity启动的时候又会从ActivityClientRecord恢复回来。

原理：
ViewModelProvider(activity).get(XXViewModel.clss)
1、创建ViewModelProvider对象的时候传入activity对象，这里传入的是ViewModelStoreOwner，因为activity实现了ViewModelStoreOwner接口
   public ViewModelProvider(@NonNull ViewModelStoreOwner owner)
2、通过 ViewModelStoreOwner#getViewModelStore 方法获取到 ViewModelStore对象，执行 ViewModelStore#get(XXViewModel.clss)
   ViewModelStore：ViewModel存储器（内部维护着一个 mViewModelStore容器，存储ViewModel的）
   ViewModelStore#get：从mViewModelStore获取ViewModel，如果获取不到就通过反射创建一个，并且加入到mViewModelStore容器里边。
       
3、ViewModelStore又是如何存取的呢？使得ViewModel的生命周期比Activity长。
  3.1、Activity实现了ViewModelStoreOwner接口，实现了接口唯一的方法 getViewModelStore
        //在返回ViewModelStore对象前，会有一个 ensureViewModelStore 操作
	    public ViewModelStore getViewModelStore() {
	    	...
	        ensureViewModelStore();
	        return mViewModelStore;
	    }
  3.2、ensureViewModelStore：先通过 getLastNonConfigurationInstance 获取 NonConfigurationInstances，再从NonConfigurationInstances里边获取 ViewModelStore对象，如果获取不到才创建ViewModelStore对象

  3.3、NonConfigurationInstances对象从哪里来？
   ① Activity重写了onRetainNonConfigurationInstance，这个方法有个Object的返回值。
   ② 如果Activity的ViewModelStore对象不为空，就会创建NonConfigurationInstances对象，然后将ViewModelStore对象赋值给NonConfigurationInstances对象
   ③ 然后将NonConfigurationInstances对象从 onRetainNonConfigurationInstance 方法中返回。
  3.4、onRetainNonConfigurationInstance 是在哪里调用的？
   ① 在 Activity#retainNonConfigurationInstances 里边调用的，该方法也是有个Object返回值。
   ② 同样包装了一下 onRetainNonConfigurationInstance返回的Object对象之后，然后返回。
  3.5、retainNonConfigurationInstances 又是在哪里调用的？
   ① 在 ActivityThread#performDestroyActivity 里边调用的。
   ② 将retainNonConfigurationInstances返回的Object对象，保存到了Activity对应的ActivityClientRecord里边（ActivityClientRecord#lastNonConfigurationInstances）
      ActivityClientRecord对象只有在Activity真正销毁的时候才会被销毁。
  3.6、那什么时候将NonConfigurationInstances对象恢复呢？
   ① 在 ActivityThread#performLaunchActivity 会从ActivityClientRecord中取出来
   ② 通过Activity#attach()将NonConfigurationInstances对象给Activity.mLastNonConfigurationInstances，进而取到ViewModelStore。

  3.4、ViewModelStore是在什么时候创建的？
   ViewModelStore创建都是通过 ensureViewModelStore 方法创建的
   ensureViewModelStore方法在 ViewModelStoreOwner#getViewModelStore 以及 Activity生命周期变化的时候被调用
   ① ViewModelStore要被使用的时候创建
   ② Activity生命周期变化的时候也会创建
```



##### Lifecycle

```java
Lifecycle：Lifecycle包含有关Activity与Fragment生命周期状态的信息，并允许其他对象观察此状态。
作用：
1、监听Activity与Fragment的生命周期变化，在变化时能及时通知其他组件；
2、可以有效的避免内存泄漏和解决android生命周期的常见难题；

场景：
1、Glide在Activity/Fragment处于前台的时候加载图片，不可见的状态下停止图片的加载。
   Glide在早期的做法添加一个隐形的fragment来感知生命周期变化，老麻烦了。
2、RxJava的Disposable能够在Activity/Fragment销毁是自动dispose。
3、自定义View想要感知Activity/Fragment的生命周期

原理总结：
Activity实现LifecycleOwner变成  被观察者
某个对象实现LifecycleObserver变成 观察者

通过Activity的Lifecycle，Lifecycle#addObserver(LifecycleObserver) 进行订阅
Activity的生命周期变化的时候（29-添加空Fragment、29+注册生命周期回调监听），通知订阅的LifecycleObserver

原理：以Activity为例
1、Activity实现了LifecycleOwner接口，接口只有一个方法getLifecycle，该方法需要返回Lifecycle对象
2、Activity实现了自己的Lifecycle对象并在getLifecycle方法中返回（Activity变成了被观察者）
3、我们想要某个对象接收到Activity的生命周期变化，就需要变成观察者。
   做法是该对象实现LifecycleObserver接口，然后利用Activity的getLifecycle.addObserver进行订阅
4、Activity的Lifecycle里边维护了一个 LifecycleObserver 容器，addObserver操作就是将LifecycleObserver添加到这个容器中
5、Activity#onCreate的时候会执行 ReportFragment#injectIfNeededIn
   ReportFragment#injectIfNeededIn：
      api29以下：添加一个空白的ReportFragment（类型Glide的做法）
      api29以上：Activity#registerActivityLifecycleCallbacks 直接注册回调来获取Activity的生命周期回调
6、当Activity的生命周期变化的时候，就会遍历容器通知到每个容器中的LifecycleObserver
```





#### Google支付

##### 说说Google支付流程

```java
  1、获取订单ID
	   客户端发起支付，请求服务端将商品ID传给服务端，服务端返回订单ID
	2、查询商品信息
	   客户端检查Google服务器是否连接，没连接就连接Google服务器，连接了就向Google服务器发起查询商品信息
	3、发起支付
	   查询成功后，通过查询到的商品信息以及订单ID来拉起Google支付，支付成功、失败、取消都会通过回调通知
	4、支付成功
	   将购买信息传给服务端，让服务端校验本次支付是否有效，有效则服务端可以发货了
	5、消费购买Token
	   服务端确认本次支付有效后，客户端还需要将购买信息的Token，传给Google服务器，让其消费本次购买
	6、Google服务器消费成功后，完成整个支付流程
```

##### 掉单（补单）如何处理

```java
  1、查询是否需要补单
		用户登录后，并且Google服务器连接成功了，主动发起补单查询
	2、购买信息状态为PURCHASED，则视为需要补单的
	    遍历返回的购买信息列表，查看状态。如果状态为PURCHASED，执行补单逻辑。
	3、将购买信息传给服务端，让服务端校验本次支付是否有效，有效则服务端可以发货了
	4、消费购买Token
	   服务端确认本次支付有效后，客户端还需要将购买信息的Token，传给Google服务器，让其消费本次购买
	5、Google服务器消费成功后，完成整个补单流程
```





#### RxJava

##### 说说RxJava的原理。

```java
Observable.create((ObservableOnSubscribe) emitter -> { 
        emitter.onNext();
        emitter.onComplete();
    }).subscribe(new Observer());

原理：
Observable#subscribe：
	1、将Observer对象传给新创建的一个Emitter对象中。
	2、Emitter对象实现了Observer的所有方法，并且在对应的方法中调用Observer对象对应的方法。
		例如：Emitter#onNext 调用了 Observer#onNext。
	3、而这个新创建的Emitter对象会传给一个ObservableOnSubscribe对象。
	4、这个ObservableOnSubscribe对象，就是在create操作符里边创建的。
	这样就形成一个上下游的持有链。
	当在create操作符中操作emitter，就相当于间接地在操作observer。
```



##### onComplete之后还能发射onNext、onError吗？

````java
不能了。
因为onComplete、onError处理完之后，就会调用dispose()做释放了。
而onNext、onError、onComplete等方法在执行前，都会做isDisposed的判断。

@Override
public void onComplete() {
    if (!isDisposed()) {
        try {
            observer.onComplete();
        } finally {
            dispose();
        }
    }
}

@Override
public void onNext(T t) {
    ...
    if (!isDisposed()) {
        observer.onNext(t);
    }
}

````



##### 说说map、flatMap、concatMap的区别

```java
map：根据原始数据类型 返回 另外一种数据类型
在Emitter#onNext的时候，使用了Function#apply实现类型变换
		@Override
        public void onNext(T t) {
            ...
            U v;
            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex); return;
            }
            downstream.onNext(v);
        }

concatMap和flatMap：
相同点：功能是一样的，将一个发射数据的Observable变换为多个Observable。
不同点：
concatMap是有序的（采用concat），flatMap是无序的（采用merge）。（concatMap输出的顺序与原序列保持一致。而flatMap则不一定有可能出现交错。）
merge：多个Observable交叉合并。
//如果ob1有延时，会到ob2发射了，不等ob1了。
Observable.merge(ob1, ob2).subscribe();

concat：多个Observable顺序合并。
//如果ob1有延时，会一直等到ob1发射完成，ob2才会发射。
Observable.concat(ob1, ob2).subscribe();

zip：多个Observable发射的数据合为一体
Observable.zip(ob1, ob2, new BiFunction<String, String, JSONObject>() {
            @Override
            public JSONObject apply(String response, String response2) throws Exception {
                return new JSONObject().put("one", response).put("two", response2);
            }
        }).subscribe();
```



##### Maybe、Observer、Single、Flowable、Completable几种观察者的区别，以及他们在什么场景用？

```java
Maybe：发数据只发一条。
没有onNext方法。
发送onSuccess就不会再发送其他了，发送onComplete那么相当于没发数据。
public interface MaybeObserver<@NonNull T> {  
    void onSubscribe(@NonNull Disposable d);
    void onSuccess(@NonNull T t);  
    void onError(@NonNull Throwable e);
    void onComplete();
}

Observer：能发送多条数据的。
有onNext方法，没有onSuccess方法。

Single：发数据只发一条。要么成功要么失败。
没有onNext方法，没有onComplete方法。
public interface SingleObserver<@NonNull T> {
    void onSubscribe(@NonNull Disposable d);
    void onSuccess(@NonNull T t);
    void onError(@NonNull Throwable e);
}

Completable：不能发数据，只会发成功或失败。
public interface CompletableObserver {
    void onSubscribe(@NonNull Disposable d);
    void onComplete();
    void onError(@NonNull Throwable e);
}

Flowable：支持背压策略，用于处理发送大量事件的场景。
MISSING：被观察者发送大量事件，当观察者处理不过来时，就放入缓存池。如果缓存池满了，就会抛出异常，并给出友好提示。
ERROR：  被观察者发送大量事件，当观察者处理不过来时，就放入缓存池。如果缓存池满了，就会抛出异常。
BUFFER： 被观察者发送大量事件，当观察者处理不过来时，就放入缓存池。如果缓存池满了，就会等待下游处理。
DROP：   被观察者发送大量事件，当观察者处理不过来时，就放入缓存池。如果缓存池满了，就会无法存入事件。
LATEST： 被观察者发送大量事件，当观察者处理不过来时，就放入缓存池。如果缓存池满了，就会丢弃旧的事件，缓存新的事件进来。
```



##### RxJava切换线程是怎么实现的?

```java
subscribeOn(Schedulers.io())：指定上游的Observable的线程
    1、将下游Observer，包装成另一个Observer（SubscribeOnObserver）。
    2、将一个Runnable丢给Scheduler，并触发Scheduler，Scheduler在线程池里边申请一个可用线程1，线程1运行走Runnable#run方法。
    3、run方法中执行会Observable#subscribe将包装Observer传给上游。
    4、届时上游Observable已经是处于线程1中了，Observable发射数据就是在子线程中发射的了。


observeOn(AndroidSchedulers.mainThread())：指定下游的Observer的线程
    1、将下游Observer，包装成另一个Observer（ObserveOnObserver），并调用Observable#subscribe传给上游。
    2、包装Observer里边缓存着Scheduler（HandlerScheduler）以及一个数据队列。
    3、当上游调用了其onNext，就会将数据加入到队列中，并且触发Scheduler。
    4、Scheduler内执行Handler#postRunnable，让包装Observer走run方法（包装的Observer实现了Runnable）
    5、run方法就会从队列中取数据，执行最下游Observer的onNext方法。
```



##### RxJava的subscribeOn多次调用哪个有效?

```java
第一次的subscribeOn。
但是调多次subscribeOn会切多次线程。每一次subscribeOn，都会包装成一个Observer，然后在切的线程中用上游的Obserable执行subscribe，将这个包装Observer往上传。
            Observable.create()
                .subscribeOn(Schedulers.io()) //为create()切线程
                .subscribeOn(Schedulers.io()) //为上一个subscribeOn切线程
                .subscribeOn(Schedulers.io()) //为上一个subscribeOn切线程
                .subscribe();
```



##### RxJava的observeOn多次调用哪个有效?

```java
RxJava的observeOn多次调用哪个有效?
最后一次的observeOn。
但是调多次observeOn会切多次线程。每一次subscribeOn，都会包装成一个Observer，然后在切的线程中用上游的Obserable执行subscribe，将这个包装Observer往上传。
  					Observable.create()
                .observeOn(Schedulers.io()) //为下一个observeOn切线程
                .observeOn(Schedulers.io()) //为下一个observeOn切线程
                .observeOn(Schedulers.io()) //为subscribe的Observe切线程
                .subscribe();
```



##### 说说RxBus的实现原理？

```java
Processor 既是观察者，也是被观察者。Processor 继承 FlowableProcessor支持背压。

AsyncProcessor
不论何时订阅，都只发射最后一个数据，如果因为异常而终止，不会释放任何数据，但是会传递一个异常通知。

BehaviorProcessor
发射订阅之前的一个数据和订阅之后的全部数据。如果订阅之前没有值，可以使用默认值。

PublishProcessor
从哪里订阅就从哪里发射数据。

ReplayProcessor
无论何时订阅，都发射所有的数据。

SerializedProcessor
其它 Processor 不要在多线程上发射数据，如果确实要在多线程上使用，用这个 Processor 封装，可以保证在一个时刻只在一个线程上执行。

//创建SerializedProcessor，保证线程安全
mBus = PublishProcessor.create().toSerialized();

发射事件：
    public void post(Object o) {
        new SerializedSubscriber<>(mBus).onNext(o);
    }

接收事件：通过ofType操作符，来过滤事件类型只接收自己关注的类型
    public <T> Flowable<T> toFlowable(Class<T> tClass) {
        return mBus.ofType(tClass); 
    }
```



##### 说说Rxjava异常处理以及如果完整捕获。

```java
1、RxJava自己支持的全局捕获异常
    RxJavaPlugins.setErrorHandler(new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {}
        });
但是捕获到的异常信息不全。由于调用栈太深了，有时候并没能给出这个Error在实际项目中的调用路径。

2、RxJavaExtensions（debug环境下可以用，但是release不要用（对每个Observable都提前保存堆栈是非常耗时的，获取堆栈是耗时的））
https://github.com/akarnokd/RxJavaExtensions
使用：
1、开启：RxJavaAssemblyTracking.enable();
2、发生异常时调用：RxJavaAssemblyException.find
RxJavaPlugins.setErrorHandler(new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                Throwable thw = RxJavaAssemblyException.find(throwable);
            }
        });

原理：
1、通过RxJava提供的插件方法，构建自己的Observable。可以在每个操作符中包裹上自己的Observable
        RxJavaPlugins.setOnObservableAssembly(new Function<Observable, Observable>() {
            @Override
            public Observable apply(Observable f) throws Exception {
                if (f instanceof Callable) {
                    ...
                    //将Observable包装起来，然后在返回自己的Observable
                    return new ObservableOnAssemblyCallable(f);
                }
                return new ObservableOnAssembly(f);
            }
        });

        public static void setOnObservableAssembly(Function<...> onObservableAssembly) {
            RxJavaPlugins.onObservableAssembly = onObservableAssembly;
        }

        //create操作符
        public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
            return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
        }

        //RxJavaPlugins#onAssembly
        public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
            Function<? super Observable, ? extends Observable> f = onObservableAssembly;
            //这里已经变成了RxJavaExtensions的Observable
            if (f != null) {
                //apply方法：f.apply(source);
                return apply(f, source);
            }
            return source;
        }

2、
由于操作符的Observable被自己的Observable包裹了。操作都是经过自己的Observable。
自己的Observable在构造函数的时候，将 error 信息报错下来，等到出错的时候，再将 error 信息，替换成保存下来的 error信息。

3、RxJavaAssemblyException.find
如果是RxJavaAssemblyException（自己保存下来的error类型），就直接返回。
```





#### OKHttp

1. okhttp 的拦截器设计的非常好，扩展性极强，那么这部分它是基于哪些设计模式？具体如何实现的呢？
2. okhttp 其内部多个请求的任务管理，返回结果的分发是如何设计的？

3. okhttp 内部的连接池是如何实现的？缓存多少连接，缓存多久，何时释放资源？



##### OKHttp 请求的整体流程是怎样的?

```java
1、通过建造者模式构建 OKHttpClient 与 Request
2、OKHttpClient 通过 newCall 发起一个新的请求
3、通过分发器维护请求队列与线程池，完成请求调配
4、通过五大默认拦截器完成请求重试，缓存处理，建立连接等一系列操作
5、得到网络请求结果
```



##### OKHttp 分发器是怎样工作的?

```java
分发器的主要作用是维护请求队列与线程池。
比如我们有100个异步请求，肯定不能把它们同时请求，而是应该把它们排队分个类，分为正在请求中的列表和正在等待的列表， 等请求完成后，即可从等待中的列表中取出等待的请求，从而完成所有的请求。

同步请求：同步请求不需要线程池，也不存在任何限制。所以分发器仅做一下记录。
异步请求：正在执行的任务未超过最大限制64，同时同一Host 的请求不超过5个，则会添加到正在执行队列，同时提交给线程池。
否则先加入等待队列。每个任务完成后，都会调用分发器的 finished 方法,这里面会取出等待队列中的任务继续执行
```



##### OKHttp 拦截器是如何工作的?

```java
责任链：执行链上有多个节点，每个节点都有机会（条件匹配）处理请求事务，如果某个节点处理完了就可以根据需求传递给下一个节点 或 直接返回。

应用拦截器：拿到的是原始请求，可以添加一些自定义 header、通用参数、参数加密、网关接入等等。
RetryAndFollowUpInterceptor：处理错误重试和重定向
BridgeInterceptor：应用层和网络层的桥接拦截器，主要工作是为请求添加cookie、添加固定的header，比如Host、Content-Length、Content-Type、User-Agent等等，然后保存响应结果的cookie，如果响应使用gzip压缩过，则还需要进行解压。
CacheInterceptor：缓存拦截器，如果命中缓存则不会发起网络请求。
ConnectInterceptor：连接拦截器，内部会维护一个连接池，负责连接复用、创建连接（三次握手等等）、释放连接以及创建连接上的socket流。
networkInterceptors：用户自定义拦截器，通常用于监控网络层的数据传输。（网络拦截器）
CallServerInterceptor：请求拦截器，在前置准备工作完成后，真正发起了网络请求。
```



##### 应用拦截器和网络拦截器有什么区别?

```java
应用拦截器在RetryAndFollowUpInterceptor和CacheInterceptor之前，一旦请求发生错误或者重定向都会执行多次。但是应用拦截器永远只会触发一次。

应用拦截器因为只会调用一次，通常用于统计客户端的网络请求发起情况；
网络拦截器一次调用代表了一定会发起一次网络通信，因此通常可用于统计网络链路上传输的数据。
```



##### OKHttp 如何复用 TCP 连接?

```java
ConnectInterceptor 的主要工作就是负责建立 TCP 连接，建立 TCP 连接需要经历三次握手四次挥手等操作。
每个 HTTP 请求都要新建一个 TCP 比较耗资源。在Http1.1已经支持 keep-alive（即多个 Http 请求复用一个 TCP 连接）。

OKHttp在ConnectInterceptor同样也有实现：具体实现在ExchangeFinder#findConnection
①看看是不是重定向，是则说明已经有连接了；
②通过address、host、port、代理去连接池匹配；
③通过理路由信息，IP再去连接池匹配；
④新建连接，TCP + TLS握手连接服务（阻塞）；
⑤使用新连接去连接池匹配（确保HTTP2.0的多路复用）；
⑥新连接存入连接池，返回连接。
```



##### OKHttp 空闲连接如何清除?

```java
1、在将连接加入连接池时就会启动定时任务
2、有空闲连接的话，如果最长的空闲时间大于5分钟 或 空闲数 大于5，就移除关闭这个最长空闲连接；如果 空闲数 不大于5 且 最长的空闲时间不大于5分钟，就返回到5分钟的剩余时间，然后等待这个时间再来清理。
3、没有空闲连接就等5分钟后再尝试清理。
```



##### OKHttp 有哪些优点?

```java
使用简单，在设计时使用了外观模式，将整个系统的复杂性给隐藏起来，将子系统接口通过一个客户端 OkHttpClient 统一暴露出来。
扩展性强，可以通过自定义应用拦截器与网络拦截器，完成用户各种自定义的需求
功能强大，支持 Spdy、Http1.X、Http2、以及 WebSocket 等多种协议
通过连接池复用底层 TCP(Socket)，减少请求延时
无缝的支持 GZIP 减少数据流量
支持数据缓存,减少重复的网络请求
支持请求失败自动重试主机的其他 ip，自动重定向
```



##### OKHttp 框架中用到了哪些设计模式?

```java
构建者模式：OkHttpClient 与 Request 的构建都用到了构建者模式
外观模式：OkHttp使用了外观模式,将整个系统的复杂性给隐藏起来，将子系统接口通过一个客户端 OkHttpClient 统一暴露出来。
责任链模式: OKHttp 的核心就是责任链模式，通过5个默认拦截器构成的责任链完成请求的配置
享元模式: 享元模式的核心即池中复用, OKHttp 复用 TCP 连接时用到了连接池，同时在异步请求中也用到了
```





#### Retrofit





#### Glide

##### 说说Glide的实现原理

```java
Glide.with(context).load(url).into(imageView);

with(context)：构建RequestManager
	//根据传入的上下文，进入对应的with()
	//with(Context context)
	//with(Activity activity)
	//with(FragmentActivity activity)
	//with(android.app.Fragment fragment)
	//with(Fragment fragment)
	public static RequestManager with(Context context) {
		//通过单例RequestManagerRetriever，构建一个RequestManager对象
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(context);
    }

1 根据传入的上下文，构建对应的RequestManager对象。
2 不同上下文的构建逻辑：
	Application上下文：该上下文生命周期app的生命周期，因此并不需要做什么处理，来防止内存泄漏的发生。（当不在主线程使用Glide的情况，一律按照Application上下文逻辑处理）
	非Application上下文：该上下文不管是Activity、FragmentActivity、v4包的Fragment、还是app包的Fragment，都是走同样的逻辑。那就是会向当前的Activity添加一个隐藏的Fragment，通过这个Fragment来监听当前页面的生命周期，从而防止内存泄漏的发生。


load(url)：构建RequestBuilder
RequestManager#asBitmap -> RequestBuilder<Bitmap>
RequestManager#asGif -> RequestBuilder<GifDrawable>
RequestManager#asDrawable -> RequestBuilder<Drawable>（没调用任何as，默认Drawable）
RequestManager#as(T) -> RequestBuilder<T>（例如：as(SVGADrawable.class)）


into(imageView)：
1 构建ViewTarge
根据 RequestManager#as传入的class，构建对应的ViewTarge。（ImageView最终存到ViewTarget中的view变量）
Bitmap.class -> BitmapImageViewTarget
Drawable.class -> DrawableImageViewTarget

2 构建Request
2.1 ImageView#getTag获取Request，如果不为空证明这个ImageView有正在执行的请求，将这旧请求销毁。
2.2 通过ViewTarge构建出一个SingleRequest（实现了Request接口）。调用ImageView#setTag，将Request存到ImageView中。

3 执行Request（SingleRequest）
3.1 Request#begin 执行请求，将State状态更改为RUNNING。
3.2 根据url、ImageView宽高等属性构建出key，先走loadFromCache方法（查找LRU的内存缓存），再走loadFromActiveResources（查找ActiveResources缓存）
	ActiveResources：
3.3 分别使用engineJobFactory和decodeJobFactory构建EngineJob和DecodeJob对象。
	EngineJob负责开启线程，以及切换回主线程。
	DecodeJob负责网络请求，将InputStream转换成Bitmap，对Bitmap进行缩放，以及磁盘的缓存与读取等。（网络请求使用HttpURLConnection）
3.4 回调ViewTarge#onResourceReady，将图片设置给ImageView
```



##### 说说Glide如何监控生命周期的 +3

```java
with(context)：构建RequestManager
  //根据传入的上下文，进入对应的with()
  //with(Context context)
  //with(Activity activity)
  //with(FragmentActivity activity)
  //with(android.app.Fragment fragment)
  //with(Fragment fragment)
  public static RequestManager with(Context context) {
    //通过单例RequestManagerRetriever，构建一个RequestManager对象
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(context);
    }

1 根据传入的上下文，构建对应的RequestManager对象。
2 不同上下文的构建逻辑：
  Application上下文：该上下文生命周期app的生命周期，因此并不需要做什么处理，来防止内存泄漏的发生。（当不在主线程使用Glide的情况，一律按照Application上下文逻辑处理）
  非Application上下文：该上下文不管是Activity、FragmentActivity、v4包的Fragment、还是app包的Fragment，都是走同样的逻辑。那就是会向当前的Activity添加一个空Fragment，通过这个Fragment来监听当前页面的生命周期，从而防止内存泄漏的发生。

通过空Fragment监听Activity的生命周期，在创建Fragment的时候会创建一个Lifecycle，Lifecycle是用来分发生命周期的。
每个RequestManager都实现了LifecycleListener接口，并且在RequestManager创建后都会注册到Lifecycle里边。
当Fragment生命周期发生变化，Lifecycle就会通知所有注册的RequestManager。

保证一个Activity添加一个Fragment的方法：
  1、通过tag获取获取空Fragment，如果为null再通过map获取空Fragment，如果为null证明该Activity没有添加空Fragment
  2、在add 空Fragment之后，将空Fragment添加到map里边，然后发送Handler消息再将这空Fragment从map中移除。
  （Fragment事务是靠Handler来处理的，并不是同步操作。map配合Handler的逻辑，就是防止在频繁调用Glide#with api出现添加多个空Fragment）

  SupportRequestManagerFragment getSupportRequestManagerFragment(final FragmentManager fm, Fragment parentHint) {
    //1、由FragmentManager通过tag获取fragment
    SupportRequestManagerFragment current = (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
      //2、从缓存集合中获取fragment的，map以fragmentManger为key，
      //以fragment为value进行存储
      current = pendingSupportRequestManagerFragments.get(fm);
      if (current == null) {
        //3、创建一个fragment实例
        current = new SupportRequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        //将创建的Fragment放入HashMap缓存
        pendingSupportRequestManagerFragments.put(fm, current);
        //提交事物，将fragment绑定到Activity       
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        //发消息，从map缓存中删除fragment        
        handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
  }

  public boolean handleMessage(Message message) {
   ...
    switch (message.what) {
      case ID_REMOVE_SUPPORT_FRAGMENT_MANAGER:
        FragmentManager supportFm = (FragmentManager) message.obj;
        key = supportFm;
        //从map缓存中删除fragment        
        removed = pendingSupportRequestManagerFragments.remove(supportFm);
        break;
  }
```



##### 说说Glide的缓存原理 +3

```java
内存缓存：LruCache + ActiveResources（弱引用缓存）
ActiveResources：一个弱引用的HashMap，用来缓存正在使用中的图片。
  实现原理：图片被加载之后 或者 图片从LruCache中取出这两种情况，就会将图片资源对象EngineResource put到ActiveResources。

          EngineResource里一个计数器来记录图片被引用的次数，调用acquire()方法会让变量加1，调用release()方法会让变量减1。
          计数器大于0证明EngineResource有人在用。在release()方法中检查到计数器小于0的时候，就会回调onResourceReleased，将EngineResource加入到LruCache。

          ActiveResources里边也会有一个线程一直在轮询，用来查看ReferenceQueue是否有元素，如果有则证明有EngineResource对象被回收了，也会回调onResourceReleased，将EngineResource加入到LruCache。

为什么内存缓存要设计 弱引用 + LruCache？
用弱引用缓存的资源都是当前活跃资源，使用频率比较高。这个时候如果从LruCache取资源，LinkHashmap查找资源的效率不是很高的。
所以他会设计一个弱引用来缓存当前活跃资源，来替Lrucache减压。


硬盘缓存：DiskLruCache
Glide 5大磁盘缓存策略：
DATA: 只缓存原始图片；
RESOURCE:只缓存转换过后的图片；
ALL:既缓存原始图片，也缓存转换过后的图片；对于远程图片，缓存 DATA和 RESOURCE；对于本地图片，只缓存 RESOURCE；
AUTOMATIC：默认策略。尝试对本地和远程图片使用最佳的策略。当下载网络图片时，使用DATA；对于本地图片，使用RESOURCE；
NONE：不缓存任何内容。

实现原理：在DecodeJob中，主要依靠三个解析生成器来实现硬盘缓存读取策略。
    解析生成器的三个实现类：
    ResourceCacheGenerator:管理变换之后的缓存数据；
    SourceGenerator:管理未经转换的原始缓存数据；
    SourceGenerator：直接从网络下载解析数据。

    依次通过ResourcesCacheGenerator、SourceGenerator、DataCacheGenerator来获取缓存数据。ResourcesCacheGenerator获取的是转换过的缓存数据；SourceGenerator获取的是未经转换的原始的缓存数据；DataCacheGenerator是通过网络获取图片数据再按照按照缓存策略的不同去缓存不同的图片到磁盘上。


小技巧：
图片存在七牛云上，而七牛云为了对图片资源进行保护，会在图片url地址的基础之上再加上一个token参数。
token并不是一成不变的，很有可能时时刻刻都在变化。而如果token变了，那么图片的url也跟着变了，缓存Key也就跟着变了。
导致同一张图片可能出现多缓存的情况，这样会浪费内存、浪费磁盘空间。
解决：Glide的Url是存在GlideUrl里边的，key通过getCacheKey方法获取。所以我们可以继承GlideUrl类，重写getCacheKey，去掉token即可。
     然后使用就直接 Glide.with(this).load(new MyGlideUrl(url)).into(imageView);


Glide缓存总结：弱引用 + LruCache+ DiskLruCache
读取数据的顺序是：弱引用 --> LruCache --> DiskLruCache --> 网络
写入缓存的顺序是：网络 --> DiskLruCache --> LruCache --> 弱引用

磁盘缓存就是通过DiskLruCache实现的，根据缓存策略的不同会获取到不同类型的缓存图片。它的逻辑是：先从转换后的缓存中取；没有的话再从原始的（没有转换过的）缓存中拿数据；再没有的话就从网络加载图片数据，获取到数据之后，再依次缓存到磁盘和弱引用。
```



##### 说说Glide的图片转换

```java
如果ImageView是wrap_content的那么Glide会怎么加载何图片？
例如加载500*200的图，实际上显示宽度占满屏幕，高度也根据长宽比拉大了。

ImageView默认的scaleType是FIT_CENTER。
Glide#into中，会根据ImageView的scaleType来设置相关配置。
      switch (view.getScaleType()) {
        case CENTER_CROP: 
          //new CenterCrop()
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          //new CenterInside()
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          //new FitCenter()
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          //new CenterInside()
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default: // Do nothing.
      }

CenterCrop、CenterInside、FitCenter类都是继承自BitmapTransformation，通过重写transform方法来实现各自的逻辑。
```



##### 如果设计一个图片加载框架

```java
1、Api设计：Api使用构造者模式；全局参数配置（缓存目录、缓存大小自定义、bitmap公参配置）。
2、图片加载：网络图片、本地文件、drawable资源。
3、防内存泄漏：生命周期监控。
4、图片转换：缩放到ImageView的大小；根据scaleType转换图片大小；自定义bitmap大小；自定义转换bitmap等。
5、缓存设计：三级缓存。
6、拓展性设计：支持插件拓展，比如网络加载模块可支持第三方网络框架的接管。
```



### EventBus

##### 说说EventBus原理

```java
Bus总线：Map<Event, List<Subscription>>
Subscription：包装着 订阅者类对象 以及 一个接收事件的方法

register(obj)：
1、根据订阅者类，反射找出所有对应的方法（带@Subscribe的方法）；
2、将这些方法根据事件类型，分类缓存到Bus总线。

unregister(obj)：
1、根据事件类型，从总线中查找到订阅方法列表；
2、遍历订阅方法列表，找到对应订阅者类的方法，将其移除。

post(event)：
1、根据事件类型，从总线中找出订阅该事件的集合；
2、遍历这个集合，通过反射将事件对象传给所有订阅方法；
3、发送事件的时候，会根据订阅方法在注解中配置的ThreadMode来切换到对应线程，然后再发射。

postSticky(event)：
1、先将事件类型和事件对象保存到粘性事件队列中；
2、然后走post(event)流程；
3、register(obj)的时候，如果注解中配置可接受粘性事件，则会去粘性事件队列查找使用有对应的事件，如果有则发送。
```



##### 说说EventBus3的索引加速

```java
索引加速：订阅过程中 通过APT（注解处理器）生成的索引类 取代 大量反射，从而提高订阅过程（register(obj)）的性能。

需要引入EventBusAnnotationProcessor：
apply plugin: 'com.neenbedankt.android-apt'
apt {
    arguments {
        eventBusIndex "com.llk.xxx.AAAIndex"
    }
}

apt 'org.greenrobot:eventbus-annotation-processor:3.0.1'


编译期，注解处理器会遍历所有的类找出我们订阅的方法，然后生成我们配置的索引类。
public class AAAIndex implements SubscriberInfoIndex {
private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;
    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();
        // 每有一个订阅者类，就调用一次putIndex往索引中添加相关的信息
        putIndex(new SimpleSubscriberInfo(MainActivity.class, true, new SubscriberMethodInfo[] {
        	// 类中每一个被Subscribe标识的方法都在这里添加进来
            new SubscriberMethodInfo("onEvent", MainActivity.DriverEvent.class, ThreadMode.POSTING, 0, false), 
        })); 
    }
    // 下面的代码就是EventBusAnnotationProcessor中写死的了
    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }

    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
}

EventBusBuilder.addIndex(new AAAIndex())：添加索引类
那么在register(obj)的时候，就会走索引方式查找订阅方法，而不是通过大量的反射。从而提高性能。
```

