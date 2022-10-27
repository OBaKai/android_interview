## 图片文件大小与图片内存大小

### 图片文件大小≠图片内存大小

```java
图片格式：JPEG、PNG、WEBP等。
它们本质上都是一种压缩格式，压缩的目的是为了降低存储和传输的成本。
也就是说图片加载到内存的过程会进行解压，所有文件大小≠内存大小。

JPEG：有损压缩格式，压缩比大，压缩后的体积比较小，但其高压缩率是通过去除冗余的图像数据进行的，因此解压后无法还原出完整的原始图像数据。
PNG：无损压缩格式，不会损失图片质量，解压后能还原出完整的原始图像数据，但也因此压缩比小，压缩后的体积仍然很大。
WEBP：有损压缩格式，图像的体积将比JPEG格式图像减小40%。
```



### Android中图片（Bitmap）

#### 内存大小计算

```java
Bitmap大小 = 分辨率 x 色深 x 像素密度

分辨率：像素的总数量，对应的是Bitmap的高度和宽度
色深：每个像素的大小（一个像素占用的内存大小）
像素密度：图片资源所在的密度目录（文件、网络加载的图片密度默认是1）
```

#### 分辨率

```java
分辨率：图像细节的精细程度，图像的分辨率越高，所包含的像素就越多，图像也就越清晰。
每一个方向上的像素数量来表示分辨率，也即「水平像素数×垂直像素数」，比如320×240，640×480，1280×1024等。
```

#### 色深

```java
色彩深度(Color Depth)：每个像素表示的颜色的丰富程度，色深越大表示的颜色越丰富。
假设色深的数值为n，代表每个像素会采用n个二进制位来存储颜色信息，即2^n，表示的是每个像素能显示2^n种颜色。
  
  
1bit：只能显示黑与白两个中的一个。因为在色深为1的情况下，每个像素只能存储2^1=2种颜色。
8bit：可存储2^8=256种的颜色。
24bit：可存储2^24种的颜色。每个像素的颜色由红（Red）、绿（Green）、蓝（Blue）3个颜色通道合成，每个颜色通道用8bit来表示，其取值范围是：二进制：00000000~11111111、十进制：0~255、十六进制：00~FF
32bit：在24位的基础上，增加多8个位的透明通道。
  

Android Bitmap中，可以在Bitmap.Config配置色深：
ARGB_8888：32bit，也就是一个像素占用4byte
ARGB_4444：16bit，也就是一个像素占用2byte
  RGB_565：16bit，也就是一个像素占用2byte
```

#### 像素密度

```java
像素密度：屏幕单位面积内的像素数，称为dpi。（分辨率是指像素的总数，像素密度指单位面积内的像素数，别搞混了。）
只有我们加载Android工程res目录下的资源，才需要注意像素密度。从外部文件、网络等地方加载的是不需要注意的。

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



## 图片内存优化

### 降低色深

```java
Android Bitmap中，可以在Bitmap.Config配置色深：
ARGB_8888：32bit，也就是一个像素占用4byte
ARGB_4444：16bit，也就是一个像素占用2byte
  RGB_565：16bit，也就是一个像素占用2byte

比如：
ARGB_8888 -> RGB_565 内存直接降一半好不好

Glide加载资源默认就是 RGB_565
```



### 降低分辨率（降低图片尺寸 - 下采样）

```java
针对图片尺寸的修改其实就是一个图像重新采样的过程，放大图像称为上采样（upsamping），缩小图像称为下采样（downsampling）。

Android中图片重采样提供了两种方法，一种叫做邻近采样（Nearest Neighbour Resampling），另一种叫做双线性采样（Bilinear Resampling）。
```



#### 邻近采样

```java
使用：
BitmapFactory.Options options = new BitmapFactory.Options();
//或者 inDensity 搭配 inTargetDensity 使用，算法和 inSampleSize 一样
options.inSampleSize = 2;
Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/test.png");
Bitmap compress = BitmapFactory.decodeFile("/sdcard/test.png", options);

介绍：
邻近采样的方式是比较粗暴的，直接选择其中的一个像素作为生成像素，另一个像素直接抛弃。
比如inSampleSize值设为2，采样率就是为 1/2，所以是两个像素生成一个像素。

造成的问题：
如果图片是由绿红相间的像素点组成的，最终生成的图片变成了纯绿色，也就是红色像素被抛弃
```



#### 双线性采样

```java
使用：
//例子1：bitmap.getWidth()/2, bitmap.getHeight()/2
Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/test.png");
Bitmap compress = Bitmap.createScaledBitmap(bitmap, bitmap.getWidth()/2, bitmap.getHeight()/2, true);

//例子2 或者直接使用 matrix 进行缩放
Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/test.png");
Matrix matrix = new Matrix();
matrix.setScale(0.5f, 0.5f);
bm = Bitmap.createBitmap(bitmap, 0, 0, bit.getWidth(), bit.getHeight(), matrix, true);

介绍：
双线性采样使用的是双线性內插值算法，这个算法不像邻近点插值算法一样，直接粗暴的选择一个像素，而是参考了源像素相应位置周围 2x2 个点的值，根据相对位置取对应的权重，经过计算之后得到目标图像。

双线性内插值算法在图像的缩放处理中具有抗锯齿功能, 是最简单和常见的图像缩放算法，当对相邻 2x2 个像素点采用双线性內插值算法时，所得表面在邻域处是吻合的，但斜率不吻合，并且双线性内插值算法的平滑作用可能使得图像的细节产生退化，这种现象在上采样时尤其明显。
  
邻近采样 & 双线性采样 对比：
邻近采样的方式是最快的，因为它直接选择其中一个像素作为生成像素，但是生成的图片可能会相对比较失真，产生比较明显的锯齿。
```



#### 双立方／双三次采样

```java
双立方／双三次采样使用的是双立方／双三次插值算法。邻近点插值算法的目标像素值由源图上单个像素决定，双线性內插值算法由源像素某点周围 2x2 个像素点按一定权重获得，而双立方／双三次插值算法更进一步参考了源像素某点周围 4x4 个像素。
  
双立方／双三次插值算法在平时的软件中是很常用的一种图片处理算法，但是这个算法有一个缺点就是计算量会相对比较大，是前三种算法中计算量最大的，软件 photoshop 中的图片缩放功能使用的就是这个算法。
  
这个算法在 Android 中并没有原生支持，如果需要使用，可以通过手动编写算法或者引用第三方算法库，幸运的是这个算法在 ffmpeg 中已经给到了支持，具体的实现在 libswscale/swscale.c 文件中：FFmpeg Scaler Documentation。
```



#### Lanczos 采样

```java
Lanczos 采样使用的 Lanczos 算法也可以用来作为图片的缩放，Lanczos 算法和双三次插值算法都是使用卷积核来通过输入像素计算输出像素，只不过在算法表现上稍有不同。

Lanczos 算法在 ffmpeg 的 libswscale/swscale.c 中也有实现
```



#### 总结

```java
四种采样，前两者是Android支持的，后两者需要使用ffmpeg。
Lanczos 采样的效果是最好的，但是计算量也是最大的。
在 Android 中，前两种采样方法根据实际情况去选择即可，如果对时间要求不高，倾向于使用双线性采样去缩放图片。
```



## 图片文件压缩

```java
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
```



## 如何加载大图

```java
以加载超长图为例：https://www.jianshu.com/p/705e562bc7a5
关键点：
  一、区域加载（防止内存浪费）- BitmapRegionDecoder
  二、Bitmap复用（防止快速滑动导致内存抖动）
    配置时：
    mOptions.inMutable = true; //开启复用内存
    绘制时：
		mOptions.inBitmap = mBitmap; //设置复用目标
    mBitmap = mDecode.decodeRegion(mRect, mOptions);

1、图片区域加载 - BitmapRegionDecoder（区域解码器）
① 获取Bitmap大小 以及 View大小
   创建一个以View大小为区域的Rect（mRect）
② 计算缩放因子（mScale）
   有可能图片比View大，或者比View小。所以需要计算缩放因子，让图片的宽填满View宽。
③ 区域加载
   设置图片时：
   mDecode = BitmapRegionDecoder.newInstance(inputStream, false);
   绘制Bitmap时：
   protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (null == mDecode) {
            return;
        }
        mOptions.inBitmap = mBitmap; //Bitmap复用
        matrix.setScale(mScale, mScale); //加入缩放因子
        mBitmap = mDecode.decodeRegion(mRect, mOptions);
        canvas.drawBitmap(mBitmap, matrix, null);
    }
  
2、上下滑更新加载区域 - GestureDetector（手势识别）
    在滑动时：更新加载区域
    public boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {
        mRect.offset((int)distanceX,(int)distanceY);
        //... 一些边界判断
        invalidate(); //申请重绘
        return false;
    }
 
3、如果需要缩放功能 - ScaleGestureDetector（缩放手势识别）
  更新缩放因子 + 更新加载区域
```



## Bitmap是如何释放内存的

### ① 主动释放

```java
//主动回收：recycle()
8.0之前：在Java层直接置空像素数据（byte[]）
8.0之后：调用native方法，在native层直接释放像素数据
recycle()只会给我们清除掉像素数据，但是Bitmap对象（Java层 以及 Native层）还是交由GC帮忙回收。
官方建议：平常开发中无需调用recycle()来释放内存，GC会帮我们释放的。
```

### ② 系统帮我们释放：7.0前（BitmapFinalizer）、7.0后（NativeAllocationRegistry）

#### 7.0前（BitmapFinalizer）

```java
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
```



#### 7.0后（NativeAllocationRegistry）

```java
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
