## Android

#### Context

1. 说说你对Context的了解。

2. Activity、Context、Application三者有什么不同。+2

3. Intent的作用。

   

#### Activity

1. bundle的数据结构，如何存储。
2. 说说bundle机制。+2
3. 为什么bundle无法传大图片。
4. Bunder传输对象为什么需要序列化？Serialzable与Parcelable的区别。
5. 说说Activity的启动模式，分别在什么场景使用？+3
6. Activity横竖屏切换的生命周期。+3
7. 说说Activity的启动流程。
9. activity.startActivity与context.startActivity的区别。
10. 如何保存Activity的状态？
11. onSaveInstanceState()与onRestoreIntanceState()作用是什么。
11. 为什么finish只会10s才执行onDestory？



#### Service

1. startService、bindService的区别？生命周期一样吗?

2. 启动service的两种方式以及其适用场景。

3. Service是否在 main thread执行，service里能否执行耗时操作？为什么？

4. Service与Activity 是在同一个线程吗？为什么？

   

#### BroadcastReceiver

1. 说说Broadcast的注册方式与区别。+3

   

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

   

#### C++相关

1. 函数指针如何写？

2. c++中构造函数的调用顺序，析构函数是否需要virtual。

3. . 与 -> 的区别。哪个性能更高。

4. :: 与 : 的区别。

   

#### 其他

1. 视频编解码是怎么做的。
