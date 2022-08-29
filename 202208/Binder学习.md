## 自己一些的理解

```java
无Binder不Android
  
1、Binder机制的初始化
1.1 三大步骤
① open Binder驱动（获得BinderFd）
② mmap binderFd，映射内存缓冲区
③ 启动binder线程池，并且让Binder主线程初始化
   Binder主线程是不会退出的，一直在轮询读取Binder驱动写进来的命令。
   Binder主线程主要作用是提供给Binder实体用的，接收其他进程的IPC调用。

1.1、介绍ProcessState这个单例类
Zygote fork后在子进程执行，第一时间就是调用 ProcessState::self()->startThreadPool()
ProcessState是个单例类，在ProcessState构造函数里边走了Binder机制的初始化步骤。
（ProcessState类无处不在，系统服务进程也是使用它的）
并且ProcessState还封装了所有IPC的发送、接收逻辑。

引出问题：
用了 ProcessState::self() 进程就已经拥有了使用Binder的能力了，那是不是就可以进行跨进程通讯了呢？
当然不行，你要跟谁通讯你都还不知道呢。
  
2、何如与其他进程通信
2.1、Binder实体 与 Binder引用
Binder实体：服务端的Binder对象实体，在Binder驱动以binder_node结构保存在服务端所在进程的binder_proc结构中，通过handle值能够查找到。当服务端的Binder对象发布给客户端的时候，Binder驱动会将其转为Binder引用。
  
Binder引用：客户端持有服务端的Binder对象的引用，在Binder驱动中以binder_ref结构保存在客户端所在进程的binder_proc结构中。客户端就是依靠Binder引用与服务端通讯的。  
 
Binder引用 与 Binder实体 的体现方式
java    BinderProxy（transact方法）   Binder对象（实现onTransact）
native  BpBinder      BBinder
驱动     binder_ref    binder_node

2.2、实名Binder 与 匿名Binder
实名Binder：系统进程为了方便应用进程很方便地与其进行通讯，会将自己的Binder实体发布给 service_manager进程。service_manager进程得到了其Binder引用就缓存起来。应用进程通过 name 方式跟service_manager进程查询获取对应的Binder引用。
  
匿名Binder：没有公开自己的Binder引用，双方进程通过协商一致后，服务端进程才会发布Binder实体给客户端进程。
最常见的就是两个应用进程之间的通讯了，通过客户端进程bindSerivce，AMS再向另外一个进程的Service组件请求发布Binder实体，客户端才能拿到Binder引用。
  
2.3、介绍 service_manager进程
① Binde的电话簿 - service_manager进程
就像我们想打电话给一个朋友，我们只记得他的名字，但是不记得他的号码，那我们就去电话簿对着名字找他号码就行了。

② 怎么与service_manager进程通讯？
存朋友的电话号码，那肯定是先有电话簿才能存号码吧。
service_manager进程是第一个open binder驱动的进程，其Binder实体的handle值为0（知道handle值，就能够让Binder驱动找到对应的Binder实体的数据结构了）。
defaultServiceManager函数：传入handle值0，让其返回service_manager进程的Binder引用。
 
③ 存电话号码 - 注册服务（当然这里的服务指系统服务，并不是四大组件中的Service）
defaultServiceManager()->addService(String16("media.player"), new MediaPlayerService());
名称"media.player"，服务MediaPlayerService，存到service_manager进程

④ 查电话号码 - 获取服务
defaultServiceManager()->getService(String16("media.player"));
通过名称获取对应的服务的Binder引用，这样就能够与其进行通讯了
  
  
3、Binder的通讯过程是怎么样的？
  
3.1、客户端进程向服务端进程申请Binder引用
① 客户端申请Binder引用（bindService）
② 服务端发布Binder实体（onBind -> 返回Binder对象）
③ Binder驱动将 服务端的Binder对象 转 node结构保存起来，然后转发Binder引用给到AMS
   Binder引用里边，其实就只有Binder实体对应的handle值
④ AMS将Binder引用转发客户端
   Binder驱动也会为拿到Binder引用的进程添加ref结构
3.2、客户端通过Binder引用发起Binder调用
① BinderProxy#transact -> BpBinder#transact -> ProcessState#transact
   将handle + 命令(BC_TRANSACTION) + Parcel数据包封装成 binder_transaction_data 写入到 mOut
③ 然后调用 ProcessState#waitForResponse
  while(1) -> 进入轮询，重复以下操作
  1、talkWithDriver()：与驱动交互，ioctl传入驱动fd，BINDER_WRITE_READ命令，mOut中的数据
     写了之后，下次轮询就是想驱动读数据了（会有判断是要向驱动读还是写的）
  2、mIn 读命令并且处理
     BR_TRANSACTION_COMPLETE：Binder驱动收到了（oneway的话这里就返回了，非oneway就继续等待回复）
     BR_REPLY：有回复了，从 mIn 读取出一个 binder_transaction_data，然后返回
3.3、服务端的Binder实体接收Binder调用
  Binder主线程不断读取Binder驱动发来的命令，重复以下操作
  1、talkWithDriver()：与驱动交互。这个方法不但是写的，读也是在这里发生的（会有判断是要向驱动读还是写的）
  2、mIn 读命令并且处理
    BC_TRANSACTION：读取数据，然后 BBinder#onTransact -> Binder#onTransact
  3、等返回数据给客户端啊！
    将handle + 命令(BC_REPLY) + Parcel数据包封装成 binder_transaction_data 写入到 mOut
    在下一次轮询的时候，talkWithDriver()会将mOut数据返送给驱动
  4、驱动将命令翻译为BR_REPLY，然后丢给客户端
```



## Binder大数据传输

### Binder传输大小限制

```java
Binder是有传输限制的，因为Binder设计是为了在方便与进程间频繁交互的场景的，所以数据量不能太大。

每个进程向Binder申请的缓冲区就 1MB - 8KB（详情可查看ProcessState初始化）
并且这个缓冲区是所以Binder线程公用的。也就是说每次Binder传输数据最大就是 1MB - 8KB，一般情况下都是小于这个值的。
  
注意：
oneway方式，最大传输为 (1MB - 8KB)/2
非oneway的方式，最大传输才是 1MB - 8KB（同步方式优先级大于异步，为了充分满足同步调用的内存需要，所以异步方式需要进一步限制）
```



### 传输大小限制带来的问题

最常见的问题：Intent无法传大数据

```java
实验1：传一个 小于(1MB - 8KB)/2 的数据
//警告日志
E/ActivityTaskManager: Transaction too large, intent: Intent { cmp=com.melody.test/.SecondActivity (has extras) }, extras size: 520236, icicle size: 0

实验2：传一个 大于(1MB - 8KB)/2 的数据
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



### 解决Binder传输大小限制

#### 方法1、不用ProcessState来初始化Binder机制 - 只能扩大到4MB

```java
可行，Binder服务的初始化有两步，open打开Binder驱动，mmap在Binder驱动中申请内核空间内存，所以我们只要手写open，mmap就可以轻松突破这个限制。
  
但是问题来了：mmap 最终走到 binder_mmap
//binder_mmap函数里边有这样一个限制
//如果申请的size大于4MB了，会在驱动中被修改成4MB
if ((vma->vm_end - vma->vm_start) > SZ_4M)
   vma->vm_end = vma->vm_start + SZ_4M;

也就是说通过这种方式，Binder缓冲区也只能最多扩展4mb
```



#### 方法2、匿名共享内存 - ashmem机制（Binder支持传输fd）

##### 突破Intent无法传大数据的实验

```java
//报错：TransactionTooLargeException
//原因就是我们上边说的，Binder传输有大小限制
Intent intent = new Intent(context, MainActivity.class);
intent.putExtra("bt", BitmapFactory.decodeFile("一个大图"));
context.startActivity(intent);


//再看看下边的写法，是没有的报错。
//这里突破了Binder传输大小限制，因为使用了匿名共享内存传输大数据
Intent intent = new Intent(context, MainActivity.class);
Bundle bundle = new Bundle();
bundle.putBinder("bt", new XXBinder(){
	@Override
	public Bitmap getBitmap(){
		return BitmapFactory.decodeFile("一个大图");
	}
});
intent.putExtras(bundle);
context.startActivity(intent);


为什么会出现这样的情况呢？
原因是：Binder本身并没有禁用匿名共享内存的使用，但是Intent被禁止使用匿名共享内存。
       所以这里传输Binder对象，然后再通过Binder对象获取Bitmap的方式是没有问题的。
  
接下来会分析 Binder如何触发匿名共享内存的传输数据 以及 Intent怎么就禁用了Binder的匿名共享内存传输
```



##### Binder中如何触发匿名共享内存传输数据

```java
//下面以传输Bitmap为例
1、bundle.putBinder 只是将key-value 丢到其内部的 mMap 里边
  
2、在跨进程传输前，会将Intent内的Bundle数据装到Parcel，最终传输的Parcel。
startActivity (IApplicationThread caller, String callingPackage, Intent intent, ..){
	...
	Parcel data = Parcel.obtain();
	...
	intent.writeToParcel(data, 0);
	...
	mRemote.transact(START_ ACTIVITY_ TRANSACTION, data, reply, 0);
	...
}

3、分析Intent是如何写入到Parcel
调用链：Intent#writeToParcel -> Parcel#writeBundle -> Bundle#writeToParcel -> BaseBundle#writeToParcelInner -> Parcel#writeArrayMapInternal -> Parcel#writeValue -> Parcel#writeParcelable -> Bitmap#writeParcelable
  	//Intent#writeToParcel
    public void writeToParcel(Parcel out, int flags) {
        ...
        out.writeBundle(mExtras);
    }

	  //Parcel#writeBundle
    public final void writeBundle(@Nullable Bundle val) {
        ...
        val.writeToParcel(this, 0);
    }

    //Bundle#writeToParcel
		public void writeToParcel(Parcel parcel, int flags) {
        //FLAG_ALLOW_FDS：是否允许传输文件描述符
        //pushAllowFds：将Bunder里边 是否允许传输文件描述符的配置 赋值给Parcel
        final boolean oldAllowFds = parcel.pushAllowFds((mFlags & FLAG_ALLOW_FDS) != 0);
        ...
        //将mMap丢给Parcel，让它写入
        super.writeToParcelInner(parcel, flags);
    }

    //Parcel#writeArrayMapInternal
    void writeArrayMapInternal(@Nullable ArrayMap<String, Object> val) {
        ...
        for (int i=0; i<N; i++) {
            ...
            writeString(val.keyAt(i));
            writeValue(val.valueAt(i));
            ...
        }
    }
 
    //Parcel#writeValue
		public final void writeValue(@Nullable Object v) {
        if (v == null) {
            writeInt(VAL_NULL);
        } else if (v instanceof String) {
            writeInt(VAL_STRING);
            writeString((String) v);
        ...
        } else if (v instanceof Parcelable) { //Bitmap实现了Parcelable，所以走这里
            writeInt(VAL_PARCELABLE);
            writeParcelable((Parcelable) v, 0);
        } else if (v instanceof Short) {
            writeInt(VAL_SHORT);
            writeInt(((Short) v).intValue());
        } else if (v instanceof Long) {
            writeInt(VAL_LONG);
            writeLong((Long) v);
        ...
    }
    
    //Parcel#writeParcelable
    public final void writeParcelable(@Nullable Parcelable p, int parcelableFlags) {
        ...
        p.writeToParcel(this, parcelableFlags); 
    }
      
    //Bitmap#writeParcelable
    public void writeToParcel(Parcel p, int flags) {
        noteHardwareBitmapSlowCall();
        if (!nativeWriteToParcel(mNativePtr, mDensity, p)) {
            throw new RuntimeException("native writeToParcel failed");
        }
    }
      
      
4、Bitmap写入Parcel的分析（打开匿名共享开关 并且 数据量大于16KB，将会使用匿名共享内存传递数据）
调用链：Bitmap#writeParcelable -> Bitmap.cpp#Bitmap_writeToParcel -> Parcel.cpp#writeBlob

//Bitmap.cpp#Bitmap_writeToParcel
jboolean Bitmap_writeToParcel(JNIEnv* env, jobject, jlong bitmapHandle, jboolean isMutable, jint density, jobject parcel) {
    ...
    //通过Java层parcel对象，获取其native层的parcel对象
    //其实Java层的Parcel对象也只是个壳子，它的方法最终都是走jni将信息写到native层的parcel对象的。
    android::Parcel* p = android::parcelForJavaObject(env, parcel);

    //获取 nativeBitmap的内存地址 获取到nativeBitmap对象，然后获取其SkBitmap对象（SkBitmap对象能获取到Bitmap图像数据等信息）
    SkBitmap bitmap;
    auto bitmapWrapper = reinterpret_cast<BitmapWrapper*>(bitmapHandle);
    bitmapWrapper->getSkBitmap(&bitmap);

    ...
    android::status_t status;
    //获取Bitmap的匿名fd，如果有的话
    int fd = bitmapWrapper->bitmap().getAshmemFd();
    //Bitmap本身就有匿名fd 并且 图像不能更改（!isMutable）&& 该Parcel运行传递fd
    if (fd >= 0 && !isMutable && p->allowFds()) {
        ...
        //将fd写到Parcel里边
        status = p->writeDupImmutableBlobFileDescriptor(fd);
        ...
        return JNI_TRUE; //直接返回
    }
    ...
    
    //上边的条件不满住，就会走 Parcel#writeBlob 函数
    //将Bitmap的像素数据拷贝到一个blob对象中，然后以blob方式写入Parcel
    size_t size = bitmap.computeByteSize(); //Bitmap像素数据大小
    android::Parcel::WritableBlob blob;
    //writeBlob：除了初始化blob对象之外，还做了是否使用匿名共享内存的判断
    status = p->writeBlob(size, mutableCopy, &blob);
    if (status) { //
        doThrowRE(env, "Could not copy bitmap to parcel blob.");
        return JNI_FALSE;
    }
    ...
    //将Bitmap像素数据写入到Blob对象里边
    //如果允许使用匿名共享内存，这里的写入将会是写入到匿名fd里边
    const void* pSrc =  bitmap.getPixels();
    if (pSrc == NULL) {
        memset(blob.data(), 0, size);
    } else {
        memcpy(blob.data(), pSrc, size);
    }
    blob.release();
    return JNI_TRUE;
}
      
//Parcel.cpp#writeBlob
status_t Parcel::writeBlob(size_t len, bool mutableCopy, WritableBlob* outBlob){
    ...
    status_t status;
    //!mAllowFds：Parcel不允许携带fd
    //len <= BLOB_INPLACE_LIMIT：数据长度小于16KB
    if (!mAllowFds || len <= BLOB_INPLACE_LIMIT) { 
        ...
        void* ptr = writeInplace(len);
        ...
        outBlob->init(-1, ptr, len, false);
        return NO_ERROR; //这里直接返回，外部会将像素数据写入Blob对象
    }

    //使用匿名共享内存方式（使用ashmem机制）
    int fd = ashmem_create_region("Parcel Blob", len);
    int result = ashmem_set_prot_region(fd, PROT_READ | PROT_WRITE);
    if (result < 0) {
        status = result;
    } else {
        //给fd分配内存空间
        void* ptr = ::mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
        if (ptr == MAP_FAILED) {
            status = -errno;
        } else {
            ...
            if (result < 0) {
                status = result;
            } else {
                status = writeInt32(mutableCopy ? BLOB_ASHMEM_MUTABLE : BLOB_ASHMEM_IMMUTABLE);
                if (!status) {
                    //将fd保存到Parcel
                    status = writeFileDescriptor(fd, true);
                    if (!status) {
                        outBlob->init(fd, ptr, len, mutableCopy);
                        return NO_ERROR; //这里直接返回，外部会将像素数据写入Blob对象的匿名fd里边
                    }
                }
            }
        }
        ...
    }
    ...
}
```



##### Intent是怎么禁用Binder的fd的传输

```java
调用链：startActivity -> ... -> ActivityManager#startActivity -> Instrumentation#execStartActivityFromAppTask -> Intent#prepareToLeaveProcess -> Intent#setAllowFds -> Bundle#setAllowFds（最终Parcel也会拿到这个配置开关）
  
//ActivityManager#startActivity
public void startActivity(Context context, Intent intent, Bundle options) {
	ActivityThread thread = ActivityThread.currentActivityThread();
	thread.getInstrumentation().execStartActivityFromAppTask(context,
		thread.getApplicationThread(), mAppTaskImpl, intent, options);
}

//Instrumentation#execStartActivityFromAppTask
public ActivityResult execStartActivity(..., Intent intent, int requestCode, Bundle options) {
	...
	intent.migrateExtraStreamToClipData();
	intent.prepareToLeaveProcess(who);
	int result = ActivityManager.getService().startActivity(...);
	...
}

//Intent#prepareToLeaveProcess
public void prepareToLeaveProcess(Context context) {
	final boolean leavingPackage = (mComponent == null) || !Objects.equals(mComponent.getPackageName(), context.getPackageName());
	prepareToLeaveProcess(leavingPackage);
}

//Intent#prepareToLeaveProcess
public void prepareToLeaveProcess(boolean leavingPackage) {
	setAllowFds(false); //这里！！！
	...
}

//Intent#setAllowFds
public void setAllowFds(boolean allowFds) {
	if (mExtras != null) { //这里的mExtras就是Bundle对象
		mExtras.setAllowFds(allowFds);
	}
}

//Bundle#setAllowFds：allowFds为false
public boolean setAllowFds(boolean allowFds) {
	final boolean orig = (mFlags & FLAG_ALLOW_FDS) != 0;
	if (allowFds) {
		mFlags |= FLAG_ALLOW_FDS;
	} else {
    //禁用了FLAG_ALLOW_FDS，到后面Parcel打包数据的时候，会先配置这个开关
    //上文提到的 parcel.pushAllowFds((mFlags & FLAG_ALLOW_FDS) != 0);
		mFlags &= ~FLAG_ALLOW_FDS;
	}
	return orig;
}
```



##### 自己也可以使用匿名共享内存 - MemoryFile

```java
MemoryFile 就是官方提供给我们使用的匿名共享内存API
其底层也是使用ashmem机制。
```



## Binder 驱动

```java
Binder驱动是通信的核心。尽管名叫‘驱动’，实际上和硬件设备没有任何关系，只是实现方式和设备驱动程序是一样的：它工作于内核态，提供open()，mmap()，poll()，ioctl()等标准文件操作，以字符驱动设备中的misc设备注册在设备目录/dev下，用户通过/dev/binder访问该它。
  
驱动负责进程之间Binder通信的建立，Binder在进程之间的传递，Binder引用计数管理，数据包在进程之间的传递和交互等一系列底层支持。驱动和应用程序之间定义了一套接口协议，主要功能由ioctl()接口实现，不提供read()，write()接口，因为ioctl()灵活方便，且能够一次调用实现先写后读以满足同步交互，而不必分别调用write()和read()。
  
Binder驱动的代码位于linux目录的drivers/misc/binder.c中。
```



## Binder协议

Binder协议基本格式是（命令+数据），使用ioctl(fd, cmd, arg)函数实现交互。命令由参数cmd承载，数据由参数arg承载，随cmd不同而不同。下表列举了所有命令及其所对应的数据：

| 命令                   | 含义                                                         | arg                                                          |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BINDER_WRITE_READ      | 该命令向Binder写入或读取数据。参数分为两段：写部分和读部分。如果write_size不为0就先将write_buffer里的数据写入Binder；如果read_size不为0再从Binder中读取数据存入read_buffer中。write_consumed和read_consumed表示操作完成时Binder驱动实际写入或读出的数据个数。 | struct binder_write_read {signed long write_size;signed long write_consumed;unsigned long write_buffer;signed long read_size;signed long read_consumed;unsigned long read_buffer;}; |
| BINDER_SET_MAX_THREADS | 该命令告知Binder驱动接收方（通常是Server端）线程池中最大的线程数。由于Client是并发向Server端发送请求的，Server端必须开辟线程池为这些并发请求提供服务。告知驱动线程池的最大值是为了让驱动发现线程数达到该值时不要再命令接收端启动新的线程。 | int max_threads;                                             |
| BINDER_SET_CONTEXT_MGR | 将当前进程注册为SMgr。系统中同时只能存在一个SMgr。只要当前的SMgr没有调用close()关闭Binder驱动就不能有别的进程可以成为SMgr。 | --                                                           |
| BINDER_THREAD_EXIT     | 将当前进程注册为SMgr。系统中同时只能存在一个SMgr。只要当前的SMgr没有调用close()关闭Binder驱动就不能有别的进程可以成为SMgr。 | --                                                           |
| ...                    | ...                                                          | ...                                                          |



### BINDER_WRITE_READ 命令

这是Binder中最常用的命令，参数包括两部分数据：一部分是向Binder写入的数据，一部分是要从Binder读出的数据，驱动程序先处理写部分再处理读部分。
这样安排的好处是应用程序可以很灵活地处理命令的同步或异步。

例如：
若要发送异步命令可以只填入写部分而将read_size置成0；
若要只从Binder获得数据可以将写部分置空即write_size置成0；
若要发送请求并同步等待返回数据可以将两部分都置上。

#### 写操作

Binder写操作的数据时格式同样也是（命令+数据）。这时候命令和数据都存放在binder_write_read 结构write_buffer域指向的内存空间里，多条命令可以连续存放。数据紧接着存放在命令后面，格式根据命令不同而不同。下表列举了Binder写操作支持的命令：

| cmd                                                       | 含义                                                         | arg                                                          |
| :-------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BC_TRANSACTION <br/>BC_REPLY                              | BC_TRANSACTION用于Client向Server发送请求数据；BC_REPLY用于Server向Client发送回复（应答）数据。其后面紧接着一个binder_transaction_data结构体表明要写入的数据。 | struct binder_transaction_data                               |
| BC_FREE_BUFFER                                            | 释放一块映射的内存。Binder接收方通过mmap()映射一块较大的内存空间，Binder驱动基于这片内存采用最佳匹配算法实现接收数据缓存的动态分配和释放，满足并发请求对接收缓存区的需求。应用程序处理完这片数据后必须尽快使用该命令释放缓存区，否则会因为缓存区耗尽而无法接收新数据。 | 指向需要释放的缓存区的指针；该指针位于收到的Binder数据包中   |
| BC_INCREFS<br/>BC_ACQUIRE<br/>BC_RELEASE<br/>BC_DECREFS   | 这组命令增加或减少Binder的引用计数，用以实现强指针或弱指针的功能。 | 32位Binder引用号                                             |
| BC_INCREFS_DONE<br/>BC_ACQUIRE_DONE                       | 第一次增加Binder实体引用计数时，驱动向Binder实体所在的进程发送BR_INCREFS， BR_ACQUIRE消息；Binder实体所在的进程处理完毕回馈BC_INCREFS_DONE，BC_ACQUIRE_DONE | void *ptr：Binder实体在用户空间中的指<br/>void *cookie：与该实体相关的附加数据 |
| BC_REGISTER_LOOPER<br/>BC_ENTER_LOOPER<br/>BC_EXIT_LOOPER | 这组命令同BINDER_SET_MAX_THREADS一道实现Binder驱动对接收方线程池管理。BC_REGISTER_LOOPER通知驱动线程池中一个线程已经创建了；BC_ENTER_LOOPER通知驱动该线程已经进入主循环，可以接收数据；BC_EXIT_LOOPER通知驱动该线程退出主循环，不再接收数据。 |                                                              |
| BC_REQUEST_DEATH_NOTIFICATION                             | 获得Binder引用的进程通过该命令要求驱动在Binder实体销毁得到通知。虽说强指针可以确保只要有引用就不会销毁实体，但这毕竟是个跨进程的引用，谁也无法保证实体由于所在的Server关闭Binder驱动或异常退出而消失，引用者能做的是要求Server在此刻给出通知。 | uint32 *ptr; 需要得到死亡通知的Binder引用<br/>void **cookie: 与死亡通知相关的信息，驱动会在发出死亡通知时返回给发出请求的进程。 |
| BC_DEAD_BINDER_DONE                                       | 收到实体死亡通知书的进程在删除引用后用本命令告知驱动。       | void **cookie                                                |

最常用的是BC_TRANSACTION / BC_REPLY命令对，Binder请求和应答数据就是通过这对命令发送给接收方。这对命令所承载的数据包由结构体struct binder_transaction_data定义。

Binder交互有同步和异步之分，利用binder_transaction_data中flag域区分。如果flag域的TF_ONE_WAY位为1则为异步交互，即Client端发送完请求交互即结束， Server端不再返回BC_REPLY数据包；否则Server会返回BC_REPLY数据包，Client端必须等待接收完该数据包方才完成一次交互。

#### 读操作

从Binder里读出的数据格式和向Binder中写入的数据格式一样，采用（消息ID+数据）形式，并且多条消息可以连续存放。下表列举了从Binder读出的命令字及其相应的参数：

| cmd                                                     | 含义                                                         | arg                                                          |
| ------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BR_ERROR                                                | 发生内部错误（如内存分配失败）                               |                                                              |
| BR_OK<br/>BR_NOOP                                       | 操作完成                                                     |                                                              |
| BR_SPAWN_LOOPER                                         | 该消息用于接收方线程池管理。当驱动发现接收方所有线程都处于忙碌状态且线程池里的线程总数没有超过BINDER_SET_MAX_THREADS设置的最大线程数时，向接收方发送该命令要求创建更多线程以备接收数据。 |                                                              |
| BR_TRANSACTION<br/>BR_REPLY                             | 这两条消息分别对应发送方的BC_TRANSACTION和BC_REPLY，表示当前接收的数据是请求还是回复。 | binder_transaction_data                                      |
| BR_DEAD_REPLY                                           | 交互过程中如果发现对方进程或线程已经死亡则返回该消息         |                                                              |
| BR_TRANSACTION_COMPLETE                                 | 发送方通过BC_TRANSACTION或BC_REPLY发送完一个数据包后，都能收到该消息做为成功发送的反馈。这和BR_REPLY不一样，是驱动告知发送方已经发送成功，而不是Server端返回请求数据。所以不管同步还是异步交互接收方都能获得本消息。 |                                                              |
| BR_INCREFS<br/>BR_ACQUIRE<br/>BR_RELEASE<br/>BR_DECREFS | 这一组消息用于管理强/弱指针的引用计数。只有提供Binder实体的进程才能收到这组消息。 | oid *ptr：Binder实体在用户空间中的指针<br/>void *cookie：与该实体相关的附加数据 |
| BR_DEAD_BINDER<br/>BR_CLEAR_DEATH_NOTIFICATION_DONE     | 向获得Binder引用的进程发送Binder实体死亡通知书；收到死亡通知书的进程接下来会返回BC_DEAD_BINDER_DONE做确认。 | void **cookie：在使用BC_REQUEST_DEATH_NOTIFICATION注册死亡通知时的附加参数。 |
| BR_FAILED_REPLY                                         | 如果发送非法引用号则返回该消息                               |                                                              |

和写数据一样，其中最重要的消息是BR_TRANSACTION 或BR_REPLY，表明收到了一个格式为binder_transaction_data的请求数据包（BR_TRANSACTION）或返回数据包（BR_REPLY）。



#### struct binder_transaction_data ：收发数据包结构

```java
//该结构是Binder接收/发送数据包的标准格式，每个成员定义如下：

//对于发送数据包的一方，该成员指明发送目的地。由于目的是在远端，所以这里填入的是对Binder实体的引用，存target.handle中。如前述，Binder的引用在代码中也叫句柄（handle）。
//当数据包到达接收方时，驱动已将该成员修改成Binder实体，即指向Binder对象内存的指针，使用target.ptr来获得。该指针是接收方在将Binder实体传输给其它进程时提交给驱动的，驱动程序能够自动将发送方填入的引用转换成接收方Binder对象的指针，故接收方可以直接将其当做对象指针来使用（通常是将其reinterpret_cast成相应类）。
union {
size_t handle;
void *ptr;
} target;


//发送方忽略该成员；接收方收到数据包时，该成员存放的是创建Binder实体时由该接收方自定义的任意数值，做为与Binder指针相关的额外信息存放在驱动中。驱动基本上不关心该成员。
void *cookie;


//该成员存放收发双方约定的命令码，驱动完全不关心该成员的内容。通常是Server端定义的公共接口函数的编号。
unsigned int code;


//与交互相关的标志位，其中最重要的是TF_ONE_WAY位。如果该位置上表明这次交互是异步的，Server端不会返回任何数据。驱动利用该位来决定是否构建与返回有关的数据结构。另外一位TF_ACCEPT_FDS是出于安全考虑，如果发起请求的一方不希望在收到的回复中接收文件形式的Binder可以将该位置上。因为收到一个文件形式的Binder会自动为数据接收方打开一个文件，使用该位可以防止打开文件过多。
unsigned int flags;


//该成员存放发送方的进程ID和用户ID，由驱动负责填入，接收方可以读取该成员获知发送方的身份。
pid_t sender_pid;
uid_t sender_euid;


//该成员表示data.buffer指向的缓冲区存放的数据长度。发送数据时由发送方填入，表示即将发送的数据长度；在接收方用来告知接收到数据的长度。
size_t data_size;


//驱动一般情况下不关心data.buffer里存放什么数据，但如果有Binder在其中传输则需要将其相对data.buffer的偏移位置指出来让驱动知道。有可能存在多个Binder同时在数据中传递，所以须用数组表示所有偏移位置。本成员表示该数组的大小。
size_t offsets_size;


//data.bufer存放要发送或接收到的数据；data.offsets指向Binder偏移位置数组，该数组可以位于data.buffer中，也可以在另外的内存空间中，并无限制。buf[8]是为了无论保证32位还是64位平台，成员data的大小都是8个字节。
union {
  struct {
    const void *buffer;
    const void *offsets;
  } ptr;
	uint8_t buf[8];
} data;


//offsets_size 和 data.offsets 两个成员，这是Binder通信有别于其它IPC的地方。
如前述，Binder采用面向对象的设计思想，一个Binder实体可以发送给其它进程从而建立许多跨进程的引用；另外这些引用也可以在进程之间传递，就象java里将一个引用赋给另一个引用一样。为Binder在不同进程中建立引用必须有驱动的参与，由驱动在内核创建并注册相关的数据结构后接收方才能使用该引用。而且这些引用可以是强类型，需要驱动为其维护引用计数。然而这些跨进程传递的Binder混杂在应用程序发送的数据包里，数据格式由用户定义，如果不把它们一一标记出来告知驱动，驱动将无法从数据中将它们提取出来。于是就使用数组data.offsets存放用户数据中每个Binder相对data.buffer的偏移量，用offsets_size表示这个数组的大小。驱动在发送数据包时会根据data.offsets和offset_size将散落于data.buffer中的Binder找出来并一一为它们创建相关的数据结构。在数据包中传输的Binder是类型为struct flat_binder_object的结构体，详见后文。
```


## Binder机制的初始化步骤

### ① 打开Binder驱动，并获得Binder Fd

```java
fd = open("/dev/binder", O_RDWR|O_CLOEXEC);

介绍：
这一步，Binder驱动会在内部为我们创建一个binder_proc结构，并且加入binder_proc链表中
  
备注：
某进程open binder驱动之后（open实际上最终走了 binder_open）
binder驱动内部会创建一个进程对应的binder_proc结构（一个进程一个binder_proc结构）
并且将这个binder_proc加入到 binder_proc链表中。
  
每个binder_proc都有一个唯一handle值，handle值从0开始递增。（handle值非常重要贯穿整个Binder机制，进程A要与进程B进行Binder通讯，必须知道对方的handle值才行）
service_manager进程的handle值就是0，因为它是第一个open binder驱动的进程。
```

### ② mmap Binder Fd，映射内存缓冲区

```java
mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE|MAP_NORESERVE, mDriverFD, 0);

介绍：
mmap BinderFd，分配内存映射区
  
备注：
使用ProcessState单例类的进行初始化Binder机制的，mmap的缓冲区大小默认是 1mb - 8kb
service_manager进程，mmap的缓冲区大小是 128kb
  因为service_manager进程主要负责类似电话簿的功能，所有不需要太大的缓冲区
```

### ③ 启动Binder线程池，loop Binder主线程

```java
ProcessState::self()->startThreadPool();

介绍：
向Binder驱动注册轮询线程，然后让Binder主线程进入loop，不断地与Binder驱动进行交互。

备注：
使用ProcessState单例类的进行初始化Binder机制的，Binder线程池默认16个（主线程 + 15个普通Binder线程）
```



## Binder类型

### 实名Binder

```java
注册到 service_manager进程 的Binder，是实名Binder。
只需要通过名字就能从 service_manager进程 获取到对应的Binder对象。
  
实名Binder是为了提供给系统所有的进程使用的（包括应用进程）。
  
例如：
//通过"media.player"就能获取到Binder对象了
defaultServiceManager()->getService(String16("media.player"));

//因为该系统服务在进程启动之后，就注册Binder到 service_manager进程 了
//main_mediaserver.cpp#main
defaultServiceManager()->addService(String16("media.player"), new MediaPlayerService());
```



### 匿名Binder

```java
没有注册到 service_manager进程 的Binder，是匿名Binder。
  
匿名Binder并不想给所有人用，而是只想给跟它有关系或者认识它的人用。
最常见的就是我们使用AIDL进行进程间通讯。我们通讯双方都知道组件的Action，并且都拥有相同的AIDL文件。
```



## service_manager进程

### service_manager介绍

```java
Binder通信过程中的守护进程，同时自身也是一个Binder实体。
service_manager作用是查询和注册系统服务。
  
service_manager进程由init进程通过解析init.rc文件而创建的。
  
对应的可执行程序/system/bin/servicemanager（源文件是service_manager.c，进程名为/system/bin/servicemanager）
```



### service_manager启动

1、走Binder机制三大步骤，Binder主线程进入looper。

2、等待Client端（应用程序）、Server端（系统服务）的请求

```java
//service_manager#main
int main(int argc, char **argv) {
    struct binder_state *bs;
    //打开Binder驱动，并且调用mmap分配内存映射区
    bs = binder_open(128*1024);
    ...
    //成为上下文管理者，整个系统中只有一个这样的管理者（service_manager进程）。
    //就是调用 ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
    if (binder_become_context_manager(bs)) { return -1; }
    ...
    //向Binder驱动注册轮询线程，并进入loop循环。传入回调函数指针，用来处理请求
    binder_loop(bs, svcmgr_handler);
    return 0;
}

struct binder_state *binder_open(size_t mapsize){
    struct binder_state *bs;
    struct binder_version vers;
    bs = malloc(sizeof(*bs));
    ...
    //通过系统调用陷入内核，打开Binder设备驱动
    bs->fd = open("/dev/binder", O_RDWR);
    ...
    bs->mapsize = mapsize;
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    ...
    return bs;
}

void binder_loop(struct binder_state *bs, binder_handler func) {
    ...
    //将BC_ENTER_LOOPER命令发送给binder驱动，让service_manager进入循环
    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        ...
        //read_size > 0，所以这里是读请求
        //不断读取Binder驱动发过来的请求
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        ...
        //解析请求，然后让func这个回调函数处理请求
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        ...
    }
}
```



### service_manager Binser对象的获取 - defaultServiceManager函数

函数作用：返回BpServiceManager Binder对象， 用于跟service_manager进程通信（defaultServiceManager() 等价于 new BpServiceManager(new BpBinder(0))）

ProcessState::self()主要工作：

- 调用open()，打开/dev/binder驱动设备；
- 再利用mmap()，创建大小为`1M-8K`的内存地址空间；
- 设定当前进程最大的最大并发Binder线程个数为`16`。

BpServiceManager巧妙将通信层与业务层逻辑合为一体，

- 通过继承接口`IServiceManager`实现了接口中的业务逻辑函数；
- 通过成员变量`mRemote`= new BpBinder(0)进行Binder通信工作。
- BpBinder通过handler来指向所对应BBinder, 在整个Binder系统中`handle=0`代表ServiceManager所对应的BBinder。

```java
1、在各个系统进程中，都很经常看到这样一段代码。
  例如：根据name获取对应Binder对象、
	const String16 name("window");
	mWindowManager = defaultServiceManager()->getService(name);

2、defaultServiceManager() 来源于 #include <binder/IServiceManager.h>
  //该函数用于获取IServiceManager对象，IServiceManager对象是service_manager的Binder对象
  sp<IServiceManager> defaultServiceManager();
  
	IServiceManager提供了很多接口函数：
	  sp<IBinder> getService(const String16& name) const override;
    sp<IBinder> checkService(const String16& name) const override;
    status_t addService(const String16& name, const sp<IBinder>& service, bool allowIsolated, int dumpsysPriority) override;
		...

3、来看看 defaultServiceManager() 是如何实现的
//IServiceManager.cpp#defaultServiceManager
sp<IServiceManager> defaultServiceManager(){
    std::call_once(gSmOnce, []() {
        sp<IServiceManager> sm = nullptr;
        while (sm == nullptr) { //一直轮询，知道拿到为止
          	//获取 IServiceManager 对象
            //① getContextObject(nullptr)：获取BpBinder对象
            //② interface_cast：将BpBinder对象转为对应的Binder对象
            sm = interface_cast<IServiceManager>(ProcessState::self()->getContextObject(nullptr));
            //如果为空，证明service_manager进程还没启动，那我睡一觉再试试
            //因为service_manager进程一启动第一时间就是把Binder对象注册到Binder驱动
            if (sm == nullptr) {
                sleep(1);
            }
        }
        gDefaultServiceManager = sp<ServiceManagerShim>::make(sm);
    });

    return gDefaultServiceManager;
}
```

#### ① 获取BpBinder对象

```java
//ProcessState::self()->getContextObject(nullptr) ：
getContextObject：该函数返回一个BpBinder对象

下文有ProcessState的详细介绍，其中就有 getContextObject 函数的展开分析。
```

#### ② 转为BpServiceManager对象

```java
1、interface_cast<IServiceManager>(...) 分析：
//interface_cast<IServiceManager> 等价于 IServiceManager::asInterface

//IServiceManager::asInterface 等价于 new BpServiceManager(obj)
 android::sp<IServiceManager> IServiceManager::asInterface(const android::sp<android::IBinder>& obj)
{
       android::sp<IServiceManager> intr;
        if(obj != NULL) {
           intr = static_cast<IServiceManager *>(
               obj->queryLocalInterface(IServiceManager::descriptor).get());
           if (intr == NULL) {
               intr = new BpServiceManager(obj);
            }
        }
       return intr;
}

2、new BpServiceManager(obj) 分析：
① BpServiceManager(const sp<IBinder>& impl) : BpInterface<IServiceManager>(impl){ ... }

② inline BpInterface<INTERFACE>::BpInterface(const sp<IBinder>& remote) : BpRefBase(remote){ ... }

③ BpRefBase(remote) 分析：
//BpRefBase的mRemote指向BpBinder(0)对象，从而BpServiceManager能够利用Binder进行通过通信
BpRefBase::BpRefBase(const sp<IBinder>& o) : mRemote(o.get()), mRefs(NULL), mState(0){
   ...
    if (mRemote) {
        mRemote->incStrong(this);
        mRefs = mRemote->createWeak(this);
    }
}
```



#### service_manager注册服务（待深入学习）

media服务注册：

```java
//main_mediaserver.cpp#main
defaultServiceManager()->addService(String16("media.player"), new MediaPlayerService());
```

1. MediaPlayerService进程调用 ioctl() 向Binder驱动发送IPC数据，该过程可以理解成一个事务 binder_transaction 

   (记为 T1 )，执行当前操作的线程binder_thread(记为 thread1 )，则T1->from_parent=NULL，T1->from = thread1，thread1->transaction_stack=T1。其中IPC数据内容包含：

   - Binder协议为BC_TRANSACTION；
   - Handle等于0；
   - RPC代码为ADD_SERVICE；
   - RPC数据为”media.player”。

2. Binder驱动收到该Binder请求，生成`BR_TRANSACTION`命令，选择目标处理该请求的线程，即ServiceManager的binder线程(记为`thread2`)，则 T1->to_parent = NULL，T1->to_thread = `thread2`。并将整个binder_transaction数据(记为`T2`)插入到目标线程的todo队列；

3. Service Manager的线程`thread2`收到`T2`后，调用服务注册函数将服务”media.player”注册到服务目录中。当服务注册完成后，生成IPC应答数据(`BC_REPLY`)，T2->form_parent = T1，T2->from = thread2, thread2->transaction_stack = T2。

4. Binder驱动收到该Binder应答请求，生成`BR_REPLY`命令，T2->to_parent = T1，T2->to_thread = thread1, thread1->transaction_stack = T2。 在MediaPlayerService收到该命令后，知道服务注册完成便可以正常使用。



#### service_manager获取服务（待深入学习）

```java
defaultServiceManager()->getService(String16("media.player"));
```

请求服务(getService)过程，就是向service_manager进程查询指定服务，当执行binder_transaction()时，会区分请求服务所属进程情况。

1. 当请求服务的进程与服务属于不同进程，则为请求服务所在进程创建binder_ref对象，指向服务进程中的binder_node;

   - 最终readStrongBinder()，返回的是BpBinder对象；

2. 当请求服务的进程与服务属于同一进程，则不再创建新对象，只是引用计数加1，并且修改type为BINDER_TYPE_BINDER或BINDER_TYPE_WEAK_BINDER。

   - 最终readStrongBinder()，返回的是BBinder对象的真实子类；

     

## ProcessState类

### ProcessState的介绍

```java
在介绍service_manager的两大功能注册、获取服务之前，需要先了解ProcessState类。
ProcessState是单例类，一个进程只有一个。

除了service_manager进程之外，其他进程（应用进程、系统进程）都是用过ProcessState类开启Binder机制的。
  
在进程创建的时候，会第一时间实例化ProcessState，在ProcessState构造函数里边就会开启Binder机制。
如：
Zygote fork后在子进程执行 
	-> ZygoteConnection#handleChildProc 
	-> ZygoteInit#zygoteInit 
	-> ZygoteInit#nativeZygoteInit
	-> app_main.cpp#onZygoteInit

//app_main.cpp#onZygoteInit 分析
virtual void onZygoteInit()
{
    ...
		sp<ProcessState> proc = ProcessState::self(); //获取ProcessState对象
		proc->startThreadPool(); //启动Binder线程池
    ...
}
```

### ProcessState的初始化

初始化逻辑主要就完成Binder机制三大步骤的前两步，以及告诉Binder驱动自己的一些配置。

```java
ProcessState::ProcessState(const char *driver) //driver 值为"/dev/binder"
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver)) //mDriverFD：Binder驱动的fd，用于访问Binder驱动
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mWaitingForThreads(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
    , mCallRestriction(CallRestriction::NONE)
{
    if (mDriverFD >= 0) {
    	 //BINDER_VM_SIZ 值为((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)
       //分配内存映射区,大小为1M-8k
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        ...
    }
}

static int open_driver(const char *driver)
{
    int fd = open(driver, O_RDWR | O_CLOEXEC);
    if (fd >= 0) {
        int vers = 0;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        ...
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS; //DEFAULT_MAX_BINDER_THREADS 值为15
        //告诉Binder驱动，自己的最大可并发访问的线程数为16
        //这里的值虽然为15，但是还得加上轮询线程的，所以其实是16的
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        ...
        uint32_t enable = DEFAULT_ENABLE_ONEWAY_SPAM_DETECTION; //DEFAULT_ENABLE_ONEWAY_SPAM_DETECTION 值为1
        //告诉Binder驱动，自己是否允许oneway
        result = ioctl(fd, BINDER_ENABLE_ONEWAY_SPAM_DETECTION, &enable);
        ...
    } else { ... }
    return fd;
}
```



#### ProcessState对象的获取

```java
//导入头文件 #include <binder/ProcessState.h>
//由于是单例，直接调用 ProcessState::self() 便能获取

sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) { //不为空直接返回
        return gProcess;
    }

    gProcess = new ProcessState; //实例化ProcessState
    return gProcess;
}
```



#### getContextObject分析：获取BpBinder对象

```java
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    //获取 handle为0 的Binder对象
    //这里 0 handle值对应的Binder对象 就是我们的service_manager
    //因为第一个注册到Binder驱动的Binder对象，必然是service_manager
    return getStrongProxyForHandle(0);
}


sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);
    handle_entry* e = lookupHandleLocked(handle);   //查找handle对应的资源项

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                Parcel data;
                //通过ping操作测试binder是否准备就绪
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT) return NULL;
            }
            //当handle值所对应的IBinder不存在或弱引用无效时，则创建BpBinder对象
            //BpBinder构造函数逻辑：将handle相对应Binder的弱引用增加1（IPCThreadState::self()->incWeakHandle(handle);）
            b = new BpBinder(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}
```



#### Binder线程池

##### Binder线程池创建过程

1、只有第一个Binder主线程(也就是Binder_pid_1线程)是由应用程序主动创建，Binder线程池中普通线程都是由Binder驱动根据IPC通信需求创建

2、每个进程默认设置最大线程数15，是指 BC_REGISTER_LOOPER 线程（普通Binder线程）。不包括 BC_ENTER_LOOPER 线程（Binder主线程）。也就是说Binder线程默认最大为16个。

**Binder系统中可分为3类binder线程**

- Binder主线程：进程创建过程会调用startThreadPool()过程中再进入spawnPooledThread(true)，来创建Binder主线程。编号从1开始，也就是意味着binder主线程名为`binder_pid_1`，并且主线程是不会退出的。
- Binder普通线程：是由Binder Driver来根据是否有空闲的binder线程来决定是否创建binder线程，回调spawnPooledThread(false) ，该线程名格式为`binde_pid_x`。
- Binder其他线程：其他线程是指并没有调用spawnPooledThread方法，而是直接调用IPC.joinThreadPool()，将当前线程直接加入binder线程队列。例如： mediaserver和servicemanager的主线程都是binder线程，但system_server的主线程并非binder线程。

```java
调用链：app_main.cpp#onZygoteInit -> ProcessState#startThreadPool -> ProcessState#spawnPooledThread -> PoolThread#threadLoop -> IPCThreadState#joinThreadPool

//app_main.cpp#onZygoteInit 分析
virtual void onZygoteInit()
{
    ...
		sp<ProcessState> proc = ProcessState::self(); //获取ProcessState对象
		proc->startThreadPool(); //启动Binder线程池
    ...
}  
  
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock); //多线程同步
    if (!mThreadPoolStarted) { //通过 mThreadPoolStarted 来保证每个进程只允许启动一个binder线程池
        mThreadPoolStarted = true;
        //本次创建的是binder主线程(isMain=true)，其余binder线程池中的线程都是由Binder驱动来控制创建的。
        spawnPooledThread(true);
    }
}

void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        //生成Binder线程名，格式为 ("Binder:%d_%X", pid, s)，s是个从1开始递增的整数值。
        String8 name = makeBinderThreadName();
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}

class PoolThread : public Thread{
public: PoolThread(bool isMain) : mIsMain(isMain) { }
protected:
    virtual bool threadLoop() {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }
    const bool mIsMain;
};

void IPCThreadState::joinThreadPool(bool isMain){
    //创建Binder线程
  	//BC_ENTER_LOOPER：表示是Binder主线程，不会退出的线程
    //BC_REGISTER_LOOPER：表示是Binder驱动创建的线程
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    set_sched_policy(mMyThreadId, SP_FOREGROUND); //设置前台调度策略
    status_t result;
    do { //轮询处理指令
        ...
        result = getAndExecuteCommand(); //处理下一条指令

        if (result < NO_ERROR && result != TIMED_OUT
                && result != -ECONNREFUSED && result != -EBADF) {
            abort();
        }

        if(result == TIMED_OUT && !isMain) {
            break; //非主线程出现timeout则线程退出
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER);  //线程退出循环
    talkWithDriver(false); //false代表bwr数据的read_buffer为空
}


//下边的函数是Binder线程处理命令的逻辑了 ======================== start
status_t IPCThreadState::getAndExecuteCommand(){
    status_t result;
    int32_t cmd;

    result = talkWithDriver(); //与Binder进行交互
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        cmd = mIn.readInt32(); //读取命令
        ...
        result = executeCommand(cmd); //执行Binder命令
        ...
    }

    return result;
}

//mOut有数据，mIn还没有数据。doReceive默认值为true
status_t IPCThreadState::talkWithDriver(bool doReceive){
    binder_write_read bwr;
    ...
    // 当同时没有输入和输出数据则直接返回
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;
    ...
    do {
        //ioctl执行binder读写操作，经过syscall，进入Binder驱动。调用Binder_ioctl
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        ...
    } while (err == -EINTR);
    ...
    return err;
}

//执行Binder驱动发送过来的命令
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
    ... //一大堆case
    case BR_TRANSACTION_SEC_CTX:
    case BR_TRANSACTION:
        ...
        break;
    case BR_DEAD_BINDER:
        ...
        break;
		... //一大堆case
    case BR_SPAWN_LOOPER: //创建普通Binder线程
        mProcess->spawnPooledThread(false);
        break;
    }
		...
    return result;
}
//上边的函数是Binder线程处理命令的逻辑了 ======================== end
```

##### Binder线程优先级继承

```java
假如：
线程A通过非oneway的Binder调用到线程B，如果线程A的优先级大于线程B，这里就会有一个问题出现，线程A会因为线程B的优先级较低而block更多的时间。
显然这是不合理的，Binder设计理念就是让你感觉不到IPC的存在，如同在同一线程中调用一个方法那么容易。
  
Binder的优化方法：将线程A优先级传递给线程B
1、将线程A的优先级打包进传输数据包里边
2、唤醒线程B之后，保存线程B的优先级参数，并设置成线程A的优先级
3、IPC结束后，恢复线程B的优先级

详情&源码分析文章：Binder线程优先级继承 https://www.jianshu.com/p/1115616e7d2a
```





## Binder权限控制

```java
进程A与进程B通信。进程A通过IPC调用了进程B的 doSomething 方法。
那么进程B可以在 doSomething 方法里边可以通过
Binder.getCallingPid();
Binder.getCallingUid();
获取到进程A的UID和PID，可以对UID和PID进行权限比对，判断进程A是否有权限使用线程B的一些能力。

但是如果进程B的 doSomething 方法内，需要使用到进程B的某些组件 或者 进程B要给进程C发起IPC调用
那么UID和PID就需要是进程B的才对。所以所以UID和PID需要进行切换，从进程A的切换到进程B的。
因此就会有下边的组合用法：
//在system_server进程的各个线程中比较常见（普通的app应用很少出现）
//清除远程Binder调用端uid和pid信息，并保存到origId变量
final long origId = Binder.clearCallingIdentity();
...
//还原远程Binder调用端的uid和pid信息
Binder.restoreCallingIdentity(origId);
```



### clearCallingIdentity()

clearCallingIdentity()作用是清空远程调用端的uid和pid，用当前本地进程的uid和pid替代

```java
mCallingUid 保存Binder IPC通信的调用方进程的Uid；
  可通过 Binder.getCallingPid() 方法获取
mCallingPid 保存Binder IPC通信的调用方进程的Pid；
  可通过 Binder.getCallingUid() 方法获取
  
int64_t IPCThreadState::clearCallingIdentity(){
    //通过移位操作，将UID和PID的信息保存到token，其中高32位保存UID，低32位保存PID
    int64_t token = ((int64_t)mCallingUid<<32) | mCallingPid;
    clearCaller();
    return token;
}

void IPCThreadState::clearCaller(){
    mCallingPid = getpid(); //当前进程pid赋值给mCallingPid
    mCallingUid = getuid(); //当前进程uid赋值给mCallingUid
}
```



### restoreCallingIdentity()

restoreCallingIdentity()作用是恢复远程调用端的uid和pid信息

```java
void IPCThreadState::restoreCallingIdentity(int64_t token){
    mCallingUid = (int)(token>>32);
    mCallingPid = (int)token;
}
```



## Binder死亡代理

### 死亡代理使用

```java
//调用 linkToDeath 方法注册死亡代理
//当该Binder对应的进程死了，就会执行 IBinder.DeathRecipient() 回调
Binder对象.asBinder().linkToDeath(new IBinder.DeathRecipient() {
                @Override
                public void binderDied() {
                    //todo
                }
            }, 0);
```



### 死亡代理总结

```java
linkToDeath过程
requestDeathNotification过程向驱动传递的命令BC_REQUEST_DEATH_NOTIFICATION，参数有mHandle和BpBinder对象；
binder_thread_write()过程，同一个BpBinder可以注册多个死亡回调，但Kernel只允许注册一次死亡通知。
注册死亡回调的过程，实质就是向binder_ref结构体添加binder_ref_death指针， binder_ref_death的cookie记录BpBinder指针。

  
unlinkToDeath过程
unlinkToDeath只有当该BpBinder的所有mObituaries都被移除，才会向驱动层执行清除死亡通知的动作， 否则只是从native层移除某个recipient。
clearDeathNotification过程向驱动传递BC_CLEAR_DEATH_NOTIFICATION，参数有mHandle和BpBinder对象；
binder_thread_write()过程，将BINDER_WORK_CLEAR_DEATH_NOTIFICATION事务添加当前当前进程/线程的todo队列

  
触发死亡回调
服务实体进程：进程退出的时候会走close Binder驱动。
close -> binder_close -> binder_release
binder_release过程会执行binder_node_release()，loop该binder_node下所有的ref->death对象。 当存在，则将BINDER_WORK_DEAD_BINDER事务添加ref->proc->todo（即ref所在进程的todo队列)
引用所在进程：执行binder_thread_read()过程，向用户空间写入BR_DEAD_BINDER，并触发死亡回调。
发送死亡通知sendObituary
```



#### linkToDeath方法分析

```java
public class Binder implements IBinder {
    public void linkToDeath(DeathRecipient recipient, int flags) { }
    public boolean unlinkToDeath(DeathRecipient recipient, int flags) { return true; }
}

final class BinderProxy implements IBinder {
    public native void linkToDeath(DeathRecipient recipient, int flags) throws RemoteException;
    public native boolean unlinkToDeath(DeathRecipient recipient, int flags);
}

native层调用链：android_util_Binder.cpp#android_os_BinderProxy_linkToDeath
  							-> BpBinder.cpp#BpBinder::linkToDeath

status_t BpBinder::linkToDeath(const sp<DeathRecipient>& recipient, void* cookie, uint32_t flags){
    ...
    IPCThreadState* self = IPCThreadState::self();
    self->requestDeathNotification(mHandle, this); //写入死亡通知方法
    self->flushCommands(); //发送死亡通知
    ...
}

//往驱动写入BC_REQUEST_DEATH_NOTIFICATION命令，并写入handle值(BpBinder句柄)，proxy(代表当前BpBinder指针)
status_t IPCThreadState::requestDeathNotification(int32_t handle, BpBinder* proxy)
{
    mOut.writeInt32(BC_REQUEST_DEATH_NOTIFICATION);
    mOut.writeInt32((int32_t)handle);
    mOut.writePointer((uintptr_t)proxy);
    return NO_ERROR;
}

void IPCThreadState::flushCommands(){
    if (mProcess->mDriverFD <= 0) return;
    talkWithDriver(false);
    if (mOut.dataSize() > 0) {
        talkWithDriver(false);
    }
		...
}


1、通过Handle值获取Binder引用
2、创建并且初始化一个binder_death对象(包含死亡链表以及BpBinder)，并且把对应的binder引用设置上这个对象。接着初始化好death->work.entry这个散列链表的头。
  
//kernel/drivers/android/binder.c#binder_thread_write
//Binder驱动处理 BC_REQUEST_DEATH_NOTIFICATION 逻辑：
    switch (cmd) {
        case BC_REQUEST_DEATH_NOTIFICATION:{ //注册死亡通知
            uint32_t target;
            void __user *cookie;
            struct binder_ref *ref;
            struct binder_ref_death *death;

            get_user(target, (uint32_t __user *)ptr); //获取target
            ptr += sizeof(uint32_t);
            get_user(cookie, (void __user * __user *)ptr); //获取BpBinder
            ptr += sizeof(void *);

            ref = binder_get_ref(proc, target); //拿到目标服务的binder_ref

            if (cmd == BC_REQUEST_DEATH_NOTIFICATION) {
                //native Bp可注册多个，但Kernel只允许注册一个死亡通知
                if (ref->death) {
                    break; 
                }
                death = kzalloc(sizeof(*death), GFP_KERNEL);

                INIT_LIST_HEAD(&death->work.entry);
                death->cookie = cookie; //BpBinder指针
                ref->death = death;
                //当目标binder服务所在进程已死,则直接发送死亡通知。这是非常规情况
                if (ref->node->proc == NULL) { 
                    ref->death->work.type = BINDER_WORK_DEAD_BINDER;
                    //当前线程为binder线程,则直接添加到当前线程的todo队列. 
                    if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
                        list_add_tail(&ref->death->work.entry, &thread->todo);
                    } else {
                        list_add_tail(&ref->death->work.entry, &proc->todo);
                        wake_up_interruptible(&proc->wait);
                    }
                }
            } else {
                ...
            }
        } break;
      case ...;
```



#### 死亡代理特殊情况

```java
情况1：调用 linkToDeath 方法注册死亡代理的时候，Binder对应的进程就已经死了
Binder驱动在处理BC_REQUEST_DEATH_NOTIFICATION过程中，正好遇到Binder对应的进程已经死了的情况。
向当前Binder线程的todo队列增加BINDER_WORK_DEAD_BINDER事务，会直接发送死亡通知。

情况2：发送死亡通知的过程中，监听死亡代理的进程已经死了
这种情况可以直接无视了。（关心他人的生死的人先死了）
```



## Binder驱动提高效率的小细节

### Binder远程转本地 - 减少Binder驱动负担

```java
情况1：单进程中使用Binder
为了提高效率，在相同进程中使用Binder，是不走Binder驱动的。
一个Binder对象同一个进程中拿到的是Binder对象本身，另一个进程中拿到的是BinderProxy代理类，跨进程调用也就变成了本地方法调用，提升Binder通信效率。
  
情况2：多进程辗转传递Binder，回到Binder实体所在进程
进程A将 Binder_A 通过Binder方法传递给进程B，进程B拿到的是 BinderProxy_A
进程B又将 BinderProxy_A 通过Binder方法传递给进程C，进程C拿到的还是 BinderProxy_A
进程C又将 BinderProxy_A 通过Binder方法传递给进程A，进程A拿到的却是 Binder_A

  
记住一句话
一个IBinder对象(Binder或者BinderProxy)通过Binder方法传递的时候，Binder驱动就会校验远程转本地这个机制。如果发现这个IBinder对象的服务端（Binder）定义在本进程，就直接返回Binder对象，否则返回BinderProxy对象。
  
  
问题1：AIDL oneway的这个标识符是不是在Binder远程转本地的时候，是不是也就失去了意思？
的确是失去了意义，远程转本地，就是普通方法调用了
  
问题2：Binder服务端oneway方法sleep10秒，是否会导致client端sleep10秒？
server端和client端是不是同一个进程，同一个进程会sleep 10秒，否则不会
  
详情&分析文章：Binder远程转本地 https://www.jianshu.com/p/740f1ee32fd1
```



### Binder线程栈复用 - 减少线程消耗

```java
假设: 
第一次Binder通信：进程A UI线程 ——> 进程B Binde线程
第二次Binder通信：进程B Binder线程 ——> 进程A Binder线程
涉及到3个线程
  
实际上：
第二次Binder通信：进程B Binder线程 ——> 进程A UI线程

为什么进程A的UI线程正在等待ServiceB的返回处于休眠的状态，竟然有空闲去响应进程B发起的ServiceA的Binder调用呢？

这一切都是Binder驱动搞的鬼，Binder驱动发现反正进程A的UI线程为了等进程B的结果休眠中，既然进程B又要向进程A发起Binder调用，与其采用进程A的Binder线程响应，还不如直接用进程A休眠的UI线程响应，这样子进程A的线程使用就从2个减少为1个

  
总结：
这个机制除了这种两个进程互相Binder调用的情况，就算是3个进程，4个进程，5个进程，甚至n个进程产生嵌套的Binder调用，也一样可以发挥作用。
当进程D发起Binder调用到进程B的时候，进程D会向后遍历整个Binder调用关系。检查是否已经有进程B参与，如果已经进程B参与了，直接唤醒进程B中参与本次Binder嵌套调用中休眠的线程，响应进程D对进程B的Binder调用
  
详情&分析文章：Binder线程栈复用 https://www.jianshu.com/p/b1b749b4ea8b
```





## 定向标签 in/out/inout

### 定向标签的使用与效果

 in： 默认方式。客户端将参数传给服务端使用，这参数在服务端被改成什么样都不关客户端的事情了。

如同子弹，打出去就不管了。

注意：元数据类型，String ，IBinder，还有AIDL的接口都是默认in，而且也不能强制给其加上out或inout



out：客户端将参数传给服务端，服务端将其作为容器，丢弃其中所有属性值后再填充内容，然后还给客户端继续处理。

如同一个盘子，服务端装满食物后，由客户端使用。



inout：客户端将参数传给服务器，服务端可以使用参数的值，同时对这个参数进行修改，客户端会得到修改后的参数，如果是集合数组等，可修改其内部的子对象。

如同客户端传给服务端一本书，服务端可以查看书中内容，也可以做一些笔记，然后还给客户端。



```java
//OrderBean类
public class OrderBean implements Parcelable{
    private String id;
    private String name;
    private int amount;
    ... //省略get/set/Parcel序列化/Parcel反序列化等方法
}

//IPC类 IOrderManager
interface IOrderManager {
    List<OrderBean> getAll();
    void add(in OrderBean bean);
    void getNameList(out String[] names);
    void updateIdList(inout String[] ids);
    void getOrder(out OrderBean bean);
    void updateOrder(inout OrderBean bean);
}

//IPC 客户端：

//增加订单，使用in方式
OrderBean orderBean=new OrderBean();
Random random=new Random();
orderBean.setAmount(random.nextInt(800)+100);
orderBean.setId(random.nextInt(100000000)+100000+"");
orderBean.setName("玩具"+random.nextInt());
orderManager.add(orderBean);
Log.i("MainActivity","@@ 增加订单 order1 -> "+ orderManager.getAll().get(0));


//测试out方式，传递数组
String[] names={"老虎","狮子","大象","骆驼"};
orderManager.getNameList(names);
Log.i("MainActivity","@@ 刷新订单 names="+ Arrays.toString(names));

//测试inout方式，传递数组
String[] ids={"a1","b2","c3","b4"};
orderManager.updateIdList(ids);
Log.i("MainActivity","@@ 刷新订单 ids="+ Arrays.toString(ids));

//测试out方式，传递Parcelable对象
OrderBean order1=new OrderBean("123","降龙十八掌",3456);
orderManager.getOrder(order1);
Log.i("MainActivity","@@ 刷新订单 order1 -> "+order1.toString());

//测试inout方式，传递Parcelable对象
OrderBean order2=new OrderBean("456","独孤九剑",6789);
orderManager.updateOrder(order2);
Log.i("MainActivity","@@ 刷新订单 order2 -> "+order2.toString());


//IPC 服务端：
Binder binder = new IOrderManager.Stub(){
         @Override
        public List<OrderBean> getAll() throws RemoteException {
            return list;
        }

        @Override
        public void add(OrderBean bean) throws RemoteException {
            bean.setId(bean.getId()+"_ii");
            bean.setName(bean.getName()+"_nn");
            bean.setAmount(bean.getAmount()+50000);
            Log.i("OrderService","@@ add bean="+ bean.toString());
            list.add(bean);
        }

        @Override
        public void getNameList(String[] names) throws RemoteException {
            Log.i("OrderService","@@ getNameList names="+ Arrays.toString(names));
            names[0]="苹果";
            names[1]="香蕉";
            names[2]="榴莲";
        }

        @Override
        public void updateIdList(String[] ids) throws RemoteException {
            Log.i("OrderService","@@ setIdList names="+ Arrays.toString(ids));
            ids[0]=ids[0]+"_11";
            ids[1]=ids[1]+"_22";
            ids[2]=ids[2]+"_33";
        }

        @Override
        public void getOrder(OrderBean bean) throws RemoteException {
            Log.i("OrderService","@@ updatePrice "+bean.toString());
            bean.setId(bean.getId()+"-12345");
            bean.setName(bean.getName()+"-九阴真经");
            bean.setAmount(bean.getAmount()+10000);
        }

        @Override
        public void updateOrder(OrderBean bean) throws RemoteException {
            Log.i("OrderService","@@ updateOrder id="+bean.toString());
            bean.setId(bean.getId()+"-98765");
            bean.setName(bean.getName()+"-葵花宝典");
            bean.setAmount(bean.getAmount()+20000);
        }
};

//in方式
04-16 17:24:01.648 31198-31229/com.example.wang.ordermanager I/OrderService: @@ add bean=OrderBean{id:43788467_ii,name:玩具1186168941_nn,amount:50731}
04-16 17:24:01.649 31279-31279/com.example.wang.client I/MainActivity: @@ 增加订单 order1 -> OrderBean{id:43788467,name:玩具1186168941,amount:731}
//out方式，数组参数
04-16 17:24:05.602 31198-31211/com.example.wang.ordermanager I/OrderService: @@ getNameList names=[null, null, null, null]
04-16 17:24:05.603 31279-31279/com.example.wang.client I/MainActivity: @@ 刷新订单 names=[苹果, 香蕉, 榴莲, null]
//inout方式，数组参数
04-16 17:24:05.611 31198-31229/com.example.wang.ordermanager I/OrderService: @@ setIdList names=[a1, b2, c3, b4]
04-16 17:24:05.612 31279-31279/com.example.wang.client I/MainActivity: @@ 刷新订单 ids=[a1_11, b2_22, c3_33, b4]
//out方式，Parcelable参数
04-16 17:24:05.613 31198-31210/com.example.wang.ordermanager I/OrderService: @@ updatePrice OrderBean{id:null,name:null,amount:0}
04-16 17:24:05.614 31279-31279/com.example.wang.client I/MainActivity: @@ 刷新订单 order1 -> OrderBean{id:null-12345,name:null-九阴真经,amount:10000}
//inout方式，Parcelable参数
04-16 17:24:05.615 31198-31211/com.example.wang.ordermanager I/OrderService: @@ updateOrder id=OrderBean{id:456,name:独孤九剑,amount:6789}
04-16 17:24:05.616 31279-31279/com.example.wang.client I/MainActivity: @@ 刷新订单 order2 -> OrderBean{id:456-98765,name:独孤九剑-葵花宝典,amount:26789}

通过输出结果可以发现：
1. 使用in方式时，参数值单向传输，客户端将对象传给服务端后，依然使用自己的对象值，不受服务端的影响。
2. 使用out方式传递数组类型时，客户端传递给服务端的只有数组的长度，客户端得到的是服务端赋值后的新数组。
3. 使用inout方式传递数组类型时，客户端会将完整的数组传给服务端，客户端得到的是服务端修改后的数组。
4. 使用out方式传递Parcelable对象时，客户端传给服务端的是一个属性值全部为空的对象，得到的是服务端重新赋值后的对象。
5. 使用inout方式传递Parcelable对象时，客户端会将完整的对象传递给服务端，得到的是服务端修改后的对象。
```



### 定向标签的原理

```java
AIDL中的使用的定向标签，效果体现在AIDL编译后的Java抽象类上。所以分析Java抽象类就能够知道其原理。

//IPC接口
interface IAidlTest {
      AidlTest.TestParcelable parcelableIn(in AidlTest.TestParcelable p);
      AidlTest.TestParcelable parcelableOut(out AidlTest.TestParcelable p);
      AidlTest.TestParcelable parcelableInOut(inout AidlTest.TestParcelable p);
}
```



#### in原理

```java
//in：将传参传给服务端，不需要知道服务端对这个传参做了什么
@Override 
public android.os.AidlTest.TestParcelable parcelableIn(android.os.AidlTest.TestParcelable p) {
          android.os.Parcel _data = android.os.Parcel.obtain();
          android.os.Parcel _reply = android.os.Parcel.obtain();
          android.os.AidlTest.TestParcelable _result;
		      ...
          //in的效果1：将传参传给服务端（这里体现是：将 传参p 的数据写入_data）
          if ((p!=null)) {
            _data.writeInt(1);
            p.writeToParcel(_data, 0);
          }
          ...
          boolean _status = mRemote.transact(Stub.TRANSACTION_parcelableIn, _data, _reply, 0);
          ...
          //in的效果2：并不会将服务端对 传参p 的改动回传回来（可以与下边out例子做对比）
            
          //注意：这里的 _result 这个返回值可以不用关注的，这例子有个返回值确实会迷糊。懒得写例子了网上拷贝的例子哈哈
          if ((0!=_reply.readInt())) { 
            _result = android.os.AidlTest.TestParcelable.CREATOR.createFromParcel(_reply);
          }
          ...
          return _result;
}
```



#### out原理

```java
//out：传参的数据不会给到服务端，但是服务端对传参数据的改动我需要知道
@Override 
public android.os.AidlTest.TestParcelable parcelableOut(android.os.AidlTest.TestParcelable p) {
          android.os.Parcel _data = android.os.Parcel.obtain();
          android.os.Parcel _reply = android.os.Parcel.obtain();
          android.os.AidlTest.TestParcelable _result;
		      ...
           //out的效果1：不会将传参传给服务端（这里体现是：没有对 传参p 的数据写入_data）
          boolean _status = mRemote.transact(Stub.TRANSACTION_parcelableOut, _data, _reply, 0);
          ...
          if ((0!=_reply.readInt())) { 
            _result = android.os.AidlTest.TestParcelable.CREATOR.createFromParcel(_reply);
          }
          ...
          //out的效果2：会将服务端对TestParcelable数据的改动，反应到传参里边（这里体现是：从返回对象_reply中读取数据到传参p）
          if ((0!=_reply.readInt())) {
            p.readFromParcel(_reply);
          }
  			  ...
          return _result;
}
```



#### inout原理

```java
//inout: 既要将传参的数据传给服务端，又要知道服务端对传参的数据做了什么改动
@Override 
public android.os.AidlTest.TestParcelable parcelableInOut(android.os.AidlTest.TestParcelable p) {
          android.os.Parcel _data = android.os.Parcel.obtain();
          android.os.Parcel _reply = android.os.Parcel.obtain();
          android.os.AidlTest.TestParcelable _result;
		      ...
          //inout效果1：将传参传给服务端（这里体现是：将 传参p 的数据写入_data）
          if ((p!=null)) {
            _data.writeInt(1);
            p.writeToParcel(_data, 0);
          }
  				...
          boolean _status = mRemote.transact(Stub.TRANSACTION_parcelableInOut, _data, _reply, 0);
          ...
          if ((0!=_reply.readInt())) { 
            _result = android.os.AidlTest.TestParcelable.CREATOR.createFromParcel(_reply);
          }
          ...
          //inout效果2：会将服务端对TestParcelable数据的改动，反应到传参里边（这里体现是：从返回对象_reply中读取数据到传参p）
          if ((0!=_reply.readInt())) {
            p.readFromParcel(_reply);
          }
  			  ...
          return _result;
}
```



## RemoteCallbackList

下边的注册回调逻辑是开发中比较常见的。

多个子模块为了让主模块完成某件事情之后通知它们，就会向主模块注册一个回调通知，当主模块某件事情之后就会调用这些已经注册的回调通知。

```java
		private List<Callback> callbacks = new ArrayList<>();
    
    public void register(Callback cb){
        callbacks.add(cb);
    }
    
    public void unregister(Callback cb){
        callbacks.remove(cb);
    }
    
    public void callBack(){
        for (Callback cb : callbacks){
            ...
        }
    }
```

RemoteCallbackList的作用也是类似的，不过这里的子模块换成了客户端进程。在IPC调用中，服务端进程可以通过RemoteCallbackList来保存多个客户端进程注册上来的接口，然后在需要的时候调用这些接口回调给客户端进程。

如下边的例子：

```java
//IPC接口
interface ICallback {
    void onGetservice(String info);
}


//IPC客户端
xxxServiceBinder.register(new ICallback{...});


//IPC服务端
private final RemoteCallbackList<ICallback> mRemoteCallbacks = new RemoteCallbackList<ICallback>();

//服务端实现的IPC方法
@Override
public void register(ICallback callback) throws RemoteException {
     mRemoteCallbacks.register(callback);
}

//服务端做完某件事情，回调客户端进程
final int N = mRemoteCallbacks.beginBroadcast();
    for (int i = 0; i < N; i++) {
         try {
              mRemoteCallbacks.getBroadcastItem(i).onGetservice("fuck android");
         } catch (RemoteException e) { e.printStackTrace(); }
    }
mRemoteCallbacks.finishBroadcast();
```



### RemoteCallbackList#register 分析

```java
//注册逻辑比较简单。
//1、就是缓存客户端进程Callback Binder对象
//2、注册这个客户端进程Callback Binder对象的死亡代理，如果客户端进程死了，就移除这其Callback的缓存
ArrayMap<IBinder, Callback> mCallbacks  = new ArrayMap<IBinder, Callback>();

public boolean register(E callback) {
     return register(callback, null);
}

public boolean register(E callback, Object cookie) {
        synchronized (mCallbacks) {
            if (mKilled) {
                return false;
            }
            IBinder binder = callback.asBinder();
            try {
                Callback cb = new Callback(callback, cookie);
                binder.linkToDeath(cb, 0); //监听客户端是否进程是否终止
                mCallbacks.put(binder, cb);
                return true;
            } catch (RemoteException e) {
                return false;
            }
        }
    }

private final class Callback implements IBinder.DeathRecipient {
      final E mCallback;
      final Object mCookie;
        
      Callback(E callback, Object cookie) {
          mCallback = callback;
          mCookie = cookie;
      }
 
      public void binderDied() {
          synchronized (mCallbacks) {
              mCallbacks.remove(mCallback.asBinder());
          }
          //RemoteCallbackList#onCallbackDied 是个空实现
          //提供给开发者继承RemoteCallbackList的时候使用的方法。
          onCallbackDied(mCallback, mCookie);
      }
}
```



### RemoteCallbackList其他方法分析

```java
unregister：反注册方法。
  
//这三个方法需要组合使用
beginBroadcast：开始对保存的集合进行回调，返回目前回调集合的大小
getBroadcastItem：获取指定索引的callback
finishBroadcast：清除之前beginBroadcast的初始状态，当处理结束请调用这个方法
  
getRegisteredCallbackCount：返回当前要处理的callback数量
  beginBroadcast返回当前处理的数量，getRegisteredCallbackCount返回当前注册的数量，两者的返回结果可能不同
  
kill：清空缓存方法。
```



## 文章

[Android Bander设计与实现 - 设计篇](https://blog.csdn.net/universus/article/details/6211589)

[彻底理解Android Binder通信架构](http://gityuan.com/2016/09/04/binder-start-service/)

[进程的Binder线程池工作过程](http://gityuan.com/2016/10/29/binder-thread-pool/)

[Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)

[Android源码的Binder权限是如何控制？](https://www.zhihu.com/question/41003297/answer/89328987)

[Android 重学系列 Binder的总结](https://www.jianshu.com/p/62eee6cf03a3)

[Android进程间通信AIDL的in、out、inout三种定向tag的使用](https://blog.csdn.net/wangwofeng1987/article/details/79963892)

