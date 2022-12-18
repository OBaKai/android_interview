## native hook（PLT hook & inline hook）

```java
PLT hook
Linux在执行动态链接的ELF的时候，为了优化性能使用了一个叫延时绑定的策略。
这个策略是为了解决原本静态编译时要把各种系统API的具体实现代码都编译进当前ELF文件里导致文件巨大臃肿的问题。

当在动态链接的ELF程序里调用共享库的函数时
第一次调用时先去查找PLT表中相应的函数，而PLT表中再跳跃到GOT表中希望得到该函数的实际地址，但这时GOT表中指向的是PLT中那条跳跃指令下面的代码，最终会执行_dl_runtime_resolve()并执行目标函数。
第二次调用时也是PLT跳转到GOT表，但是GOT中对应函数已经在第一次_dl_runtime_resolve()中被修改为函数实际地址，因此第二次及以后的调用直接就去执行目标函数，不用再去执行_dl_runtime_resolve()了。
因此，PLT Hook通过直接修改GOT表，使得在调用该共享库的函数时跳转到的是用户自定义的Hook功能代码。

技术特点：
1、可以大量Hook那些系统API，但是难以精准Hook住某次函数调用。
2、只能Hook PLT表中的函数（对于一些so内部自定义的函数无法Hook到）
3、在hook目标函数时，如果需要回调原来的函数，那就在Hook后的功能函数中直接调用目标函数即可
	假设对目标函数malloc()的调用在1.so中，用户用PLT Hook技术开发的HookMalloc()功能函数在2.so中。（因为通常情况下目标函数与用户的自定义Hook功能函数不在一个ELF文件里）当1.so中调用malloc()时会去1.so的PLT表中查询，结果是执行流程进入了2.so中的HookMalloc()中。如果这时候HookMalloc中希望调用原目标函数malloc()，那就直接调用malloc()就好了。因为这里的malloc会去2.so中的PLT表中查询，不受1.so中那个被修改过的PLT表的影响。


inline hook
代码段中插入跳转指令，从而把程序执行流程引向用户需要的功能代码中

在想要Hook的目标代码处备份下面的几条指令，然后插入跳转指令，把程序流程转移到一个stub段上去。
在stub代码段上先把所有寄存器的状态保存好，并调用用户自定义的Hook功能函数，然后把所有寄存器的状态恢复并跳转到备份代码处。
在备份代码处把当初备份的那几条指令都执行一下，然后跳转到当初备份代码位置的下面接着执行程序。


技术特点：
1、完全不受函数是否在PLT表中的限制，直接在目标so中的任意代码位置都可进行Hook。
2、可以介入任意函数的操作。
3、对Hook功能函数的限制较小。
```





## art method hook

```java
https://xiaozhuanlan.com/topic/5276903841

art hook方法：
1、entrypoint replacement：替换ArtMethod#entrypoint指针地址
  将目标函数的entrypoint替换成我们hook函数的指针地址，从而达到hook目的
  缺点：当系统知道你要调用的方法的entrypoint是多少的时候，会直接将地址写死在汇编代码里，从而不会去拿被替换的entrypoint，导致Hook失效
  例如：调用系统函数 textView.setText，系统早就知道了 setText 的函数地址的情况下，那么就会在你生成oat文件的过程中给你调用 setText 函数的指令中直接写死它的函数地址。

  
2、dynamic callee-side rewriting：修改entrypoint所指向的代码
  由于entrypoint并不一定会被用到（存在系统帮你写死函数地址的可能），但是ArtMethod#enntrypoint所指向的那段代码是一定会用到的。我们可以试图修改enntrypoint所指向那段代码。
  
  简单总结：原函数entrypoint指向的指令段，把该指令段的前面几个字节修改为一段固定跳板代码，这个跳板代码作用就是跳转到hook函数。
  
  详情：但是有个基本问题：Hook函数和原函数的代码长度基本上是不一样的，而且为了实现AOP，Hook函数通常比原函数长很多。如果直接把Hook函数的代码段copy到原函数entrypoint所指向的代码段，很可能没地儿放。因此，通常的做法是写一段trampoline。也就是把原函数entrypoint所指向代码的开始几个字节修改为一小段固定的代码，这段代码的唯一作用就是跳转到新的位置开始执行，如果这个「新的位置」就是Hook函数，那么基本上就实现了Hook；这种跳板代码我们一般称之为trampoline/stub，比如Android源码中的 art_quick_invoke_stub/art_quick_resolution_trampoline等。

  
3、vtable replacement
  ART中调用一个virtual method的时候，会查相应Class类里面的一张表，如果修改这张表对应项的指向，就能达到Hook的目的
  论文：https://ceur-ws.org/Vol-1575/paper_10.pdf
  代码：https://github.com/tdr130/art-hooking-vtable
  缺点：不是每个方法都是virtual method，这种方式非常不可靠，只能作为一种补充手段使用
```



## art下class加载过程

```java
class加载是一个懒加载的过程。
ClassLoader的创建过程完成了对dex的加载，但是class并不会进行初始化，只有被使用的时候才会初始化。

class加载过程（Class.forName执行过程）：
Class.forName 
  -> java_lang_Class.cc#Class_classForName 
  	1、类的表示转换为smali格式，然后通过指定的 class_loader 去查找类，查找过程主要通过 class_linker 实现。
    2、如果 Class.forName 中指定initialize为true，在找到对应类后还会额外执行一步 EnsureInitialized 函数进行初始化
  -> class_linker.cc#ClassLinker::FindClass（class_loader寻找类）
    1、已加载的类会保存在 ClassTable 中，以 hash 表的方式存储，该表的键就是类对应的 hash。如果 ClassTable 存在该类，直接取出来然后返回
    2、 ClassTable 中不存在，则需要进行查找。
      2.1、双亲委托机制 一个个ClassLoader -> DexFile对象 -> 根据类名搜索对应类的 ClassDef 字段
      2.2、找到类在对应Dex文件中的 ClassDef 内容后，会通过 ClassLinker 完成该类的后续注册流程，包括:
         ① 对于当前 DexFile，如果是第一次遇到，会创建一个 DexCache 缓存，保存到 ClassLinker 的 dex_caches_ 哈希表中；
         ② 通过 ClassLinker::DefineClass 完成类的初始化；
         ③ 将对应 DexFile 添加到类加载器对应的 ClassTable 中；
           
ClassLinker::DefineClass 类初始化分析：
  
ClassLinker::DefineClass 
  -> ClassLinker::LoadClass（从指定 dex 文件中加载目标类的属性和方法等内容）
     注意：这里其实是在对应类添加到 ClassTable 之后才加载的，这是出于 ART 的内部优化考虑，另外一个原因是类的属性根只能通过 ClassTable 访问，因此需要在访问前先在 ClassTable 中占好位置。
    
    属性加载通过 LoadField 实现：主要作用是初始化 ArtField 并与目标类关联起来
    方法加载通过 LoadMethod 实现：主要是使用 dex 文件中对应方法的 CodeItem 对 ArtMethod 进行初始化，并与 klass 关联。但是对于方法而言，还好进行额外的一步，即 Class_Linker.cc::LinkCode()
    
    
Class_Linker.cc::LinkCode() 中 entry_point_from_compiled_code_ 就是在这里赋值的
由于虚拟机执行情况复杂，解释、static、JNI等执行情况，entry_point_from_compiled_code_ 并不直接指向AOT编译的机器码，而是指向一段Trampoline代码来间接执行
```



## art下method调用过程

```java
method调用过程:
1、Method#invoke -> java_lang_reflect_Method.cc#Method_invoke -> reflection.cc#InvokeMethod
   java Method对象（jobject 类型的 javaMethod）强转成 ArtMethod 指针
2、经过一系列调用，最终执行 ArtMethod::Invoke 函数

void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result, const char* shorty) {
    Runtime* runtime = Runtime::Current();
    if (UNLIKELY(!runtime->IsStarted() || (self->IsForceInterpreter() && !IsNative() && !IsProxyMethod() && IsInvokable()))) {
        art::interpreter::EnterInterpreterFromInvoke(...); //解释模式
    } else { //快速模式
        bool have_quick_code = GetEntryPointFromQuickCompiledCode() != nullptr;
        if (LIKELY(have_quick_code)) {
            if (!IsStatic()) {
                (*art_quick_invoke_stub)(this, args, args_size, self, result, shorty);
            } else {
                (*art_quick_invoke_static_stub)(this, args, args_size, self, result, shorty);
            }
        }
        ...
    }
    ...
}

3、快速模式下，经过一些列调用终止通过 entry_point_from_compiled_code_ 执行对应的方法指令
art_quick_invoke_stub
art_quick_invoke_static_stub
   -> quick_entrypoints_cc_arm.cc#quick_invoke_reg_setup
   -> quick_entrypoints_cc_arm.cc#art_quick_invoke_stub_internal
   -> quick_entrypoints_arm.S#art_quick_invoke_stub_internal

//quick_entrypoints_arm.S#art_quick_invoke_stub_internal
ENTRY art_quick_invoke_stub_internal
    SPILL_ALL_CALLEE_SAVE_GPRS             @ spill regs (9)
	  ...
    ldr    ip, [r0, #ART_METHOD_QUICK_CODE_OFFSET_32]  @ get pointer to the code
    blx    ip                              @ call the method
    ...
END art_quick_invoke_stub_internal

//art_method.def#ART_METHOD_QUICK_CODE_OFFSET_32
ASM_DEFINE(ART_METHOD_QUICK_CODE_OFFSET_32, art::ArtMethod::EntryPointFromQuickCompiledCodeOffset(art::PointerSize::k32).Int32Value())

//art_method.h#EntryPointFromQuickCompiledCodeOffset
static constexpr MemberOffset EntryPointFromQuickCompiledCodeOffset(PointerSize pointer_size) {
  //终止执行 ArtMethod#entry_point_from_quick_compiled_code_ 指针地址
  return MemberOffset(PtrSizedFieldsOffset(pointer_size) + OFFSETOF_MEMBER(
         PtrSizedFields, entry_point_from_quick_compiled_code_) / sizeof(void*)
              * static_cast<size_t>(pointer_size));
}

ART对于Java方法实现了两种执行模式
  3.1、解释模式：像 Dalvik 虚拟机一样解释执行字节码
  解释执行很稳定基本没啥变化，因为解释执行的核心思路从 Android2.x 版本到现在都是一致的，因为字节码的定义并没有太多改变
  3.2、快速模式：直接调用通过 OAT 编译后的本地代码
  通过java Method对象转为native ArtMethod对象，并通过 entry_point_from_compiled_code_ 开始执行方法对应的指令



Java方法的调用主要通过 ArtMethod::Invoke 去实现
ArtMethod 结构是什么时候创建的呢？
为什么 jmethod/jobject 可以转换为 ArtMethod 指针呢？
```





## dex脱壳分析

### 加壳技术分析

```java
第一级：落地加载
apk内dex加密，app运行后加壳框架解密dex到本地
第一级衍生：不落地加载
apk内dex加密，app运行后加壳框架解密dex，dex直接加载到内存（自定义ClassLoader）

第二级：指令抽取
apk内dex除了加密外，还对dex中的方法的方法体的指令进行指令抽取（把指令换成nop），在类加载或函数被使用的时候进行指令还原，这样成功脱壳得到的dex也无法看到方法内容。
缺点：
  1、使用大量的虚拟机内部结构，会出现兼容性问题；
  2、使用android虚拟机进行函数内容的执行，无法对抗自定义虚拟机；
  3、它跟虚拟机的JIT优化出现冲突，达不到最佳的性能表现。

第三级：Java2c
apk内dex中的大量方法转成native方法（让c层去实现），这样成功脱壳也只能看到一堆native方法
缺点：
  1、会导致运行效率的降低
  2、代码大量膨胀

第四级：指令转换/VMP
主要通过实现自定义Android虚拟机的解释器，由于自定义解释器无法对Android系统内的其他函数进行直接调用，所有必须使用java的jni接口进行调用。
这种实现技术主要有两种实现：
  1、dex文件内的函数被标记为native，内容被抽离并转换为一个符合jni要求的动态库。
  2、dex文件内的函数被标记为native，内容被抽离并转换为自定义的指令格式。并通过实现自定义接收器，进行执行代码。它主要通过虚拟机提供的jni接口和虚拟机进行交互。
缺点：
  在攻击者面前，攻击者可以直接将这个加固技术方案当做黑盒，通过实现自定义的jni接口对象进行内部调试分析，从而得到完整的原始dex文件。
  
第五级：虚拟机源码保护
通过利用虚拟机技术保护app中的所有代码，包括java、Kotlin、C/C++等多种代码，虚拟机技术主要是通过把核心代码编译成中间的二进制文件，随后生成独特的虚拟机源码，保护执行环境和只有在该环境下才能执行的运行程序。
通过基于llvm工具链实现ELF文件的vmp保护。通过虚拟机保护技术，让ele文件拥有独特的可变指令集，大大提高了指令跟踪，逆向分析的强度和难度。
  
  
保护建议：
1、代码混淆
2、签名校验
3、字符串常量加密
4、防动态调试：防ptrace跟踪、防xposed
5、防动态注入：防so注入
6、防hook：
7、防多开：检查私有目录路径
8、防抓包：代理、伪造证书
9、后台接口增加签名校验，响应数据加密处理
10、其他：防截屏录屏、防界面劫持、usb调试检测
```





```java
https://zhuanlan.zhihu.com/p/450006093
https://blog.csdn.net/qq_41374107/article/details/123806053
https://blog.niunaijun.top/index.php/archives/13.html


dex加载过程 - 脱壳（packer）
1、ClassLoader对象创建的时候，就会对传入的dexPath进行切割，获得每个dex的路径，对每个dex的路径进行加载
2、每个dex的路径都会创建一个对应的DexFile对象，DexFile对象创建的时候会通过 openDexFile 方法生成mCookie对象（Object mCookie）
3、DexFile#mCookie其实是一个java long数组
   下标0是native层 OatFile对象 的指针地址
   剩余的下标都是native层 DexFile对象 的指针地址
   		native层 DexFile对象：begin_属性：Dex文件在内存中的映射地址是位于，size_属性：Dex文件的大小
4、dex的创建逻辑是在native层，大概流程是这样的
   首先将 vdex 映射到内存中，然后将已经映射到内存中的 dex 或者在磁盘中的 dex 转换为 DexFile 结构体，最后再将 vdex 和 oat 文件关联起来。


也就是说，我们拿到DexFile#mCookie，就等于获取到了所有的DexFile，我们可以将其进行dump dex


一、获取mCookie流程：
1、PathClassloader -> BaseClassloader -> DexPathList
2、DexPathList里面通过 makeDexElements 获得Element[]
3、Element[]是通过 loadDexFile返回的dalvik.system.DexFile对象创建的
4、dalvik.system.DexFile最终通过openDexFileNative返回mCookie

反过来，即可得知获取mCookie的线路

1、先获取PathClassloader中的DexPathList对象
2、再获取DexPathList中的dexElements，也就是Element[]
3、再获取每个Element中的dexFile对象
4、再获取dexFile中的mCookie值
5、通过以上路线，即可获取到该Classloader中的所有mCookie

加壳的应用需要在最早的流程中将加密的Dex进行解码，并且运行起加壳框架自己的Classloader来加载这些解密的Dex，否则应用将无法运行。


二、dump dex流程：
DexFile：
  const uint8_t* const begin_;
  const size_t size_;

预测法 预测begin_与size_位置的
   begin_与size_都是保持相对位置的，我们不知道begin_的大小，但是size_我们是肯定能知道的，我们根据这个大小，来获取size_，然后通过 begin_与size_都是保持相对位置这个特性，我们减去一个uint8_t的大小，那就能获取到begin_的偏移量。


三、对抗指令抽取壳：
todo
```



## ClassLoader加载dex分析

### ClassLoader的作用

```java
android ClassLoader的类型：
1、BootClassLoader：系统类加载器，全局唯一，app类加载器的parent
2、PathClassLoader：app类加载器，每个app都拥有一个自己的PathClassLoader
3、DexClassLoader：自定义dex的类加载器， 负责从.jar 或者 .apk 中加载类
  
ClassLoader的双亲委托机制：
类加载器在加载类的时候，会先委托给父类载器加载，其父加载器再委托给它的父加载器（如果有的话），以此类推。
	如果父加载器可以完成加载类的任务的话，就成功返回。
	如果父加载器不能完成加载类的任务或者没有父加载器的话，自己去加载类。
双亲委托机制目的：
	1.安全。防止核心API库被篡改。
  即使用户自定义的类与java核心api中的类名相同，因为双亲委托机制首先会使用父类加载器加载，由于PathClassLoader是Android应用程序的类加载器，应用开发者自己写的代码由PathClassLoader加载，PathClassLoader的父加载器是BootClassLoader，核心API类的Class已经被BootClassLoader加载过了，所以用户自定义的类不会加载。比如开发者自定义了一个String类，如果没有双亲委托机制，就会篡改String类
  2.避免重复加载。当一个类被父类加载器加载过的时候，就没必要再用子类加载器加载一遍了。
  
Class.forName方法：也存在双亲委托机制
```



### app的ClassLoader从哪来（PathClassLoader的创建过程）

```java
相关文章：https://blog.csdn.net/hp910315/article/details/78522308

总结：
1、app进程启动后，在AMS通知app进行Application启动的流程中，在创建Application对象前（需要ClassLoader来查找app自定义的Application类）
2、创建LoadedApk对象：根据AMS传过来的ApplicationInfo、CompatibilityInfo对象创建LoadedApk对象
3、创建ContextImpl对象：根据LoadedApk对象创建ContextImpl对象，并且缓存LoadedApk对象
4、通过ContextImpl#getClassLoader获取ClassLoader对象，用ClassLoader来寻找app自定义的Application类
	 4.1、ContextImpl#getClassLoader是通过LoadedApk对象的getClassLoader方法返回ClassLoader对象的
   4.2、LoadedApk#getClassLoader会判断是否缓存有PathClassLoader，如何没有则会new一个PathClassLoader并返回

AMS通知app进行Application启动的流程：IApplicationThread#handleBindApplication -> msg BIND_APPLICATION -> ActivityThread#handleBindApplication
  
//ActivityThread#handleBindApplication(AppBindData data)
private void handleBindApplication(AppBindData data) {
    //创建LoadedApk对象，缓存到data.info
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    //创建ContextImpl
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
  
    //这里又创建一个LoadedApk对象
    LoadedApk pi = getPackageInfo(instrApp, data.compatInfo, appContext.getClassLoader(), false, true, false);
    //又创建ContextImpl
    ContextImpl instrContext = ContextImpl.createAppContext(this, pi);
    ClassLoader cl = instrContext.getClassLoader();
  
    //创建Application对象，然后走Application初始化
    mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    mInitialApplication = app;
    mInstrumentation.callApplicationOnCreate(app);
}

//ContextImpl#createAppContext
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    return new ContextImpl(null, mainThread, packageInfo, null, null, false, null, null);
}

//ContextImpl构造函数：缓存LoadedApk
private ContextImpl(...LoadedApk packageInfo...) {
    ...
    mPackageInfo = packageInfo;
    ...
}

//ContextImpl#getClassLoader：通过 mPackageInfo#getClassLoader 返回ClassLoader
@Override
public ClassLoader getClassLoader() {
	return mPackageInfo != null 
    ? mPackageInfo.getClassLoader() 
    : ClassLoader.getSystemClassLoader();
}

//LoadedApk#getClassLoader
public ClassLoader getClassLoader() {
    synchronized (this) {
        if (mClassLoader != null) { return mClassLoader; }
        mClassLoader = ApplicationLoaders.getDefault().getClassLoader(zip, lib, mBaseClassLoader);
        return mClassLoader;
    }
}

//ApplicationLoaders#getClassLoader
public ClassLoader getClassLoader(String zip, String libPath, ClassLoader parent){
    ClassLoader baseParent = ClassLoader.getSystemClassLoader().getParent();
    synchronized (mLoaders) {
        if (parent == null) { parent = baseParent; }
        if (parent == baseParent) {
            //ClassLoader会被缓存在mLoaders中，也就是说它也是只会创建一次
            ClassLoader loader = mLoaders.get(zip);
            if (loader != null) {
                return loader;
            }
            //如果缓存中没有，就会创建一个PathClassLoader来对应用进行加载
            PathClassLoader pathClassloader = new PathClassLoader(zip, libPath, parent);
            mLoaders.put(zip, pathClassloader);
            return pathClassloader;
        }
        return pathClassloader;
    }
}
```



### Dex加载过程

#### java层DexFile对象的创建过程

```java
总结：
1、ClassLoader在构造函数中，就会对dexPath进行切割，获得每个dex的路径
2、对每个dex进行加载，生成对应的DexFile对象（DexFile核心是mCookie变量）

//PathClassLoader构造函数：全部转交给父类实现了
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }

    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }

    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent, ClassLoader[] sharedLibraryLoaders) {
        super(dexPath, librarySearchPath, parent, sharedLibraryLoaders);
    }
}

//BaseDexClassLoader的构造函数：DexPathList对象的创建
public BaseDexClassLoader(String dexPath, String librarySearchPath, ClassLoader parent, ClassLoader[] sharedLibraryLoaders, boolean isTrusted) {
        super(parent);
        this.sharedLibraryLoaders = sharedLibraryLoaders == null ? null : Arrays.copyOf(sharedLibraryLoaders, sharedLibraryLoaders.length);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);
        ...
        reportClassLoaderChain();
}

//DexPathList的构造函数：构建dex path列表，走makeDexElements方法加载dex
DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory, boolean isTrusted) {
        ...
        //splitPaths：分割路径String，检查路径合法性，返回文件路径列表（List<File>）
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext, isTrusted);
        ...
}

//DexPathList#makeDexElements：构建Element数组，Element缓存着DexFile对象
//核心方法loadDexFile，构建DexFile
private static Element[] makeDexElements(List<File> files, File optimizedDirectory, List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
      Element[] elements = new Element[files.size()];
      int elementsPos = 0;
      for (File file : files) {
          if (file.isDirectory()) {
              elements[elementsPos++] = new Element(file);
          } else if (file.isFile()) {
              String name = file.getName();
              DexFile dex = null;
              if (name.endsWith(DEX_SUFFIX)) {  //判断是否.dex结尾
                  try {
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                      if (dex != null) {
                          elements[elementsPos++] = new Element(dex, null);
                      }
                  } catch (IOException suppressed) {
                      suppressedExceptions.add(suppressed);
                  }
              } else {
                  try {
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                  } catch (IOException suppressed) {
                      suppressedExceptions.add(suppressed);
                  }

                  if (dex == null) {
                      elements[elementsPos++] = new Element(file);
                  } else {
                      elements[elementsPos++] = new Element(dex, file);
                  }
              }
          }
          ...
      }
      ...
      return elements;
    }

//DexPathList#loadDexFile：optimizedDirectory参数为odex的目录，一般都是null的
private static DexFile loadDexFile(File file, File optimizedDirectory, ClassLoader loader, Element[] elements) throws IOException {
        if (optimizedDirectory == null) {
            return new DexFile(file, loader, elements);
        } else {
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0, loader, elements);
        }
}

//DexFile的构造函数：核心变量mCookie
DexFile(String fileName, ClassLoader loader, DexPathList.Element[] elements) throws IOException {
  	    //Object mCookie：其实际内容是long[]
        mCookie = openDexFile(fileName, null, 0, loader, elements);
        mInternalCookie = mCookie;
        mFileName = fileName;
}
```

#### DexFile#mCookie的生成

```c++
总结：
DexFile的mCookie其实是一个java long数组
下标0是native层 OatFile对象 的指针地址
剩余的下标都是native层 DexFile对象 的指针地址
  

调用链：
DexFile#openDexFile -> DexFile#openDexFileNative -> dalvik_system_DexFile.cc#DexFile_openDexFileNative

//dalvik_system_DexFile.cc#DexFile_openDexFileNative
static jobject DexFile_openDexFileNative(JNIEnv* env, jclass, jstring javaSourceName,
                                         jstring javaOutputName ATTRIBUTE_UNUSED,
                                         jint flags ATTRIBUTE_UNUSED,
                                         jobject class_loader,
                                         jobjectArray dex_elements) {
  ...
  const OatFile* oat_file = nullptr;
  //核心函数OpenDexFilesFromOat，返回了DexFile对象列表，并且还初始化了OatFile对象
  std::vector<std::unique_ptr<const DexFile>> dex_files =
      Runtime::Current()->GetOatFileManager().OpenDexFilesFromOat(sourceName.c_str(), class_loader, dex_elements, /*out*/ &oat_file, /*out*/ &error_msgs);
  //CreateCookieFromOatFileManagerResult：将OatFile、DexFile对象转化为地址存放到一个java long数组，并返回这个long数组
  return CreateCookieFromOatFileManagerResult(env, dex_files, oat_file, error_msgs);
}

//dalvik_system_DexFile.cc#CreateCookieFromOatFileManagerResult
static jobject CreateCookieFromOatFileManagerResult(JNIEnv* env,
    std::vector<std::unique_ptr<const DexFile>>& dex_files,
    const OatFile* oat_file,
    const std::vector<std::string>& error_msgs) 
{
  jlongArray array = ConvertDexFilesToJavaArray(env, oat_file, dex_files);
  ...
  return array;
}

//dalvik_system_DexFile.cc#ConvertDexFilesToJavaArray
//注意kOatFileIndex、kDexFileIndexStart在头文件dalvik_system_DexFile.h
//constexpr size_t kOatFileIndex = 0;
//constexpr size_t kDexFileIndexStart = 1;
static jlongArray ConvertDexFilesToJavaArray(JNIEnv* env,
                                             const OatFile* oat_file,
                                             std::vector<std::unique_ptr<const DexFile>>& vec) {
  //创建一个java long数组，长度为 1 + DexFile列表大小
  jlongArray long_array = env->NewLongArray(static_cast<jsize>(kDexFileIndexStart + vec.size()));
  ...
  //long数组，0下标存放oat文件对象（OatFile）的指针地址
  long_data[kOatFileIndex] = reinterpret_cast64<jlong>(oat_file);
  //long数组，剩余下标存放dex文件对象（DexFile）的指针地址
  for (size_t i = 0; i < vec.size(); ++i) {
    long_data[kDexFileIndexStart + i] = reinterpret_cast64<jlong>(vec[i].get());
  }
  ...
  return long_array;
}
```



#### native层DexFile对象的创建过程

```java
文章参考： https://zhuanlan.zhihu.com/p/450006093

//Runtime::Current()->GetOatFileManager().OpenDexFilesFromOat 分析

//runtime.h
OatFileManager& GetOatFileManager() const {
      return *oat_file_manager_;
}

//oat_file_manager.cc OatFileManager#OpenDexFilesFromOat
std::vector<std::unique_ptr<const DexFile>> OatFileManager::OpenDexFilesFromOat(...) {
    std::vector<std::unique_ptr<const DexFile>> dex_files = OpenDexFilesFromOat_Impl(...);
    ...
    return dex_files;
}

//oat_file_manager.cc OatFileManager#OpenDexFilesFromOat_Impl
首先将 vdex 映射到内存中，然后将已经映射到内存中的 dex 或者在磁盘中的 dex 转换为 DexFile 结构体，最后再将 vdex 和 oat 文件关联起来。
```



#### 有了oat文件还需要dex吗？

```java
参考文章：https://blog.csdn.net/m0_66264699/article/details/122946517

art解释模式：需要dex文件存在于内存当中才能解释成机器指令。

art快速模式：
dex文件提前翻译成了机器指令存放在oat文件中了（对于app，oat文件以.odex结尾）
  
那么是不是意味着快速模式下不需要dex文件了呢？
原因：dex文件还存放着类相关的信息，如class_def_item,method_id_item，对于类方法的执行还离不开这些信息。
  
事实上快速模式 内存中也是会有原始的dex文件存在的，具体的执行逻辑在Runtime的GetOatFileManager().OpenDexFilesFromOat() 函数中，在这个函数中如果走dex2oat流程的话就会执行oat_file_assistant.MakeUpToDate()调用dex2oat命令生成odex和vdex文件，并生成OatFile类的对象,这个类就代表着dex2oat以后oat文件在内存中的映射表示，oat文件其实就是可执行文件，只不过它有特殊的几个符号:oatdata,oatlastword,oatbss,oatbsslastword等，OatFile类中的成员变量oat_dex_files_storage_是个std::vector<const OatDexFile*>类型的变量，而OatDexFile类表示的是这个oat文件所对应的dex文件的信息(列表)，通过OatDexFile的dex_file_pointer_即可以找到对应的art::DexFile对象的地址。
在android8.0以后，oat文件存放着dex文件编译出来的可执行指令，而原始的dex内容其实存放在vdex文件中，这一点可以在后面的dump程序中看到。
```





## JVM & DVM & ART

### JVM

```java
JVM：
jvm是运行在系统中，我们写的代码运行在jvm中，CPU只认识机器码而jvm就是将我们写的代码翻译为机器码让CPU去执行。

jvm内存模型（java堆、java虚拟机栈、方法区、本地方法栈、程序计数器（唯一不会OutOfMemoryError的））
类的生命周期：
  1、加载：查找并加载Class文件
  2、链接：包括验证、准备、和解析
  	验证：确保被导入类型的正确性
		准备：为类的静态字段分配字段，并使用默认值初始化这个字段
		解析：虚拟机将常量池内的符号引用替换为直接引用
  3、初始化：将类变量初始化为正确的初始值
  4、使用
  5、卸载
```



### DVM

```java
Dalvik：Android5.0之前都是用它
dvm的工作就是在app启动后，我们执行到对应功能的时候，就将这部分功能对应的代码翻译为机器码，保存起来然后执行。
用到的时候才进行翻译，被称为JIT（just in time）
	优点：节省内存（用到哪部分翻译哪部分）
  缺点：app运行速度变慢了（用到才去翻译，会影响到程序的执行速度）
dvm还有一个缺点，垃圾回收机制比较差。
```

### ART

```java
ART：Android5.0之后都是用它
ART在app安装的时候，就提前将所有代码翻译为机器码保存下来，等到执行的时候直接取出来让CPU执行。
提前进行翻译，被称为AOT（ahead of time）
  优点：提高了app的运行速度
  缺点：安装app时间边长（增加了翻译机器码过程）；费内存（一次加载整个app所有翻译好的机器码）
```



### JVM与android虚拟机的区别

```java
jvm与android虚拟机的区别：
1、基于的架构不同（jvm是基于栈的虚拟机，android虚拟机是基于寄存器的虚拟机）
基于栈的虚拟机：指令集普遍短，执行同样的功能需要的指令多，需要频繁出栈入栈，所以效率低，但是可移植性更强。
每一个运行的线程，都有一个独立的栈，栈中记录了方法调用的历史，每一个方法的调用，都对应一个栈帧，并且将栈帧压入栈中，最顶部的栈帧为当前栈帧，既 当前执行的方法，基于栈帧 与 操作数栈进行所有操作。

基于寄存器的虚拟机：指令集可以指定操作数，所以可以很长，执行相同的功能需要的指令更少，效率更高，但是移植性差。
没有操作数栈，不过有很多虚拟寄存器，类似操作数栈，这些寄存器也存放在运行时栈中，本质上就是一个数组，与JVM相似，在Dalvik 虚拟机中每个线程都有自己的PC 和 调用栈，方法调用的活动记录以帧为单位保存在调用栈上。

2、执行字节码不同（jvm中编译成一个或多个.class并打包成.jar，而dvm是将所以.class打包成.dex）
jvm：
不同的.java文件之间，会有一些相同的地方，分别编译为.class文件也会有相同的地方，这样的重复就有点浪费空间。
加载A.class文件的时候，发现它依赖了B.class，就需要将B.class也加载到内存，而加载到内存是一个IO操作，这样就需要两次IO操作,效率低。

dvm：.dex文件可以看成是多个.class文件的集合，在.class文件合并为.dex文件的过程中，会把重复的内容删除只留一份,这样就节省了空间。
执行.dex文件的时，因为一个.dex文件包含了多个.class文件，所以当我们加载A.class文件时，如果需要B.class文件，就不需要再加载一次了，因为它俩本来就在同一个.dex文件中，这样就减少了IO次数，提高了效率。
```



## odex、vdex、oat文件

### oat文件

```java
ART虚拟机使用的是oat文件，oat文件是一种Android私有ELF文件格式，它不仅包含有从dex文件翻译而来的本地机器指令，还包含有原来的det文件内容。

在安装apk时，系统主要通过 PacketManager 对应用进行解包和安装。其中在处理dex文件时候，会通过 installd进程 调用对应的二进制程序对字节码进行优化，ART中使用的是 dex2oat 程序。
dex2oat 基于 LLVM，优化后生成的是对应平台的二进制代码，以 oat 格式保存，oat 文件实际上是以 ELF 格式进行存储的，并在其中 oatdata 段(section) 包含了原始的 dex 内容。
  
framework中一些jar包，会生成相应的oat尾缀的文件，如system@framework@boot-telephony-common.oat。
  
文章分析：https://zhuanlan.zhihu.com/p/450006093
```



### odex、vdex文件

```java
.dex 用途：       存储java字节码
.odex/.oat 用途： optimized dex，ELF格式
.vdex 用途：      verified dex，包含 raw dex +（quicken info)
.art 用途：       image文件，存储热点方法string, method, types等
  
查看odex、vdex文件的方法：
maple_dsds:/data/app/com.llk.jni-GAEoHIXcP-gIcHJ9fR22YQ==/oat/arm64 # ls
base.odex base.vdex 

 
在Android8.0之后，oat文件一分为二，原oat仍然是elf格式（.odex），但原始dex文件内容被保存到了.vdex中。
 
为什么Android8.0引入了vdex机制？目的是为了降低dex2oat时间
因为当系统ota后，用户自己安装的应用是不会发生任何变化的，但framework代码已经发生了变化，所以就需要重新对这些应用也做dex2oat，所以如果有vdex的话，就可以省去重新校验apk里dex文件合法性的过程，节省一部分时间。
  
vdex文件
主要目的：降低dex2oat执行耗时
1、当系统OTA后，对于安装在data分区下的app，因为它们的apk都没有任何变化，那么在首次开机时，对于这部分app如果有vdex文件存在的话，执行dexopt时就可以直接跳过verify流程，进入compile dex的流程，从而加速首次开机速度；
2、当app的jit profile信息变化时，background dexopt会在后台重新做dex2oat，因为有了vdex，这个时候也可以直接跳过
文章：https://wwm0609.github.io/2017/12/21/android-vdex/
  
odex文件
在Android O之后，odex是从vdex这个文件中 提取了部分模块生成的一个新的 可执行二进制码 文件 ， odex从vdex中提取后，vdex的大小就减少了。具体过程：
1.第一次开机就会生成在/system/app/<packagename>/oat/下
2.在系统运行过程中，虚拟机将其 从“/system/app”下 copy到 “/data/davilk-cache/”下
3.odex + vdex = apk的全部源码 （vdex并不是独立于odex的文件，odex + vdex才代表一个apk）
```

