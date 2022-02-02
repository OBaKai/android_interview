#### RxJava

#### OKHttp

#### Retrofit

1. 说说你对retrofit的了解。

   

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



##### 说说Glide如果监控生命周期的 +3

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
```





#### Databinding

1. 说说databinding的原理。

   

#### 插件化

1. 启动Activity的hook方式。taskAffity。

   

#### 其他

1. 组件化中module和app之间的区别。module通信是如何实现的。
