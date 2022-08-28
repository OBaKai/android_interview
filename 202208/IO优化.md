### 一、IO的一些知识

```java
页缓存（page cache）：文件读写并不是每次都进行磁盘IO，而是将对应的磁盘文件缓存到内存上，之后对该文件的操作实际上也是对内存的读写。而这个缓存就是页缓存。

脏页（dirty page）：被修改过但还没写入磁盘的页缓存称为脏页。
  
read过程：读取文件时，操作系统会先从缓存中查找对应文件。
	如果有对应缓存 -> 直接读取缓存内容。
	如果没有对应缓存 -> 产生缺页中断，将文件读取到缓存中，同时read过程也会被阻塞。

write过程：写入数据时，会先将数据写入缓存。
	如果有对应缓存 -> 直接写入缓存，标记为脏页。
	如果没有对应缓存 -> 产生缺页中断，将磁盘文件读取到缓存，再修改缓存内容，标记为脏页。

磁盘回写：
定时回写：由pdflush进程定时将脏页写入磁盘（周期为/proc/sys/vm/dirty_writeback_centisecs，单位是(1/100)厘秒）
fsync()回写：这时候系统会唤醒pdflush进程进行回写，直到所有的脏页都写到磁盘为止。
内存不足或脏页过多，write系统调用会同样会唤醒pdflush进程进行回写。
  
调用wirte发起写请求时，是先向page cache中写入修改的内容，然后将修改过的page cache标记为dirty。内核会周期性地将dirty的page cache写回到磁盘中，完成文件内容的落盘。
如果在dirty的page cache写回磁盘之前机器发生断电关机，这部分未写回磁盘的数据将会丢失。
```



#### IO的三种方式

##### 标志IO

```java
读：file（内核空间）-> page cache（内核空间）-> user buffer（用户空间）---> 访问
写：修改 ---> user buffer（用户空间）-> page cache（内核空间）-> file（内核空间）
每个 -> 都是一次拷贝，所以在没有页缓存的情况下读、写都需要两次拷贝
```



##### mmap

```java
如果可以直接在用户空间读写页缓存，那么就可以免去将页缓存的数据复制到用户空间缓冲区的过程。变成
读：file（内核空间）-> page cache（内核空间）<===> user buffer（用户空间）---> 访问
写：修改 ---> user buffer（用户空间）<===> page cache（内核空间）-> file（内核空间）
  
mmap：虚拟内存地址 --映射--> page cache -> file

内核并不会主动把mmap映射的页缓存同步到磁盘，而是需要用户主动触发。同步mmap映射的内存到磁盘有4个时机：
调用 msync 函数主动进行数据同步（主动）。
调用 munmap 函数对文件进行解除映射关系时（主动）。
进程退出时（被动）。
系统关机时（被动）。
```



##### 直接IO

```java
直接对磁盘进行访问，不经过page cache。

读：file（内核空间）-> user buffer（用户空间）---> 访问
写：修改 ---> user buffer（用户空间）-> file（内核空间） 
  
优点：降低了 CPU 的使用率以及内存的占用。
缺点：由于是同步的，并且磁盘操作耗时是比较久的，所以每次操作都需要等待很长时间。
  
Android 并没有提供直接I/O方法，需要自行在open()文件的时候需要指定O_DIRECT参数。
```



### 二、常见IO问题

#### 1、文件损坏

##### ① 应用程序维度

###### 使用不当

```java
由于大部分的IO都不是原子操作，这些操作都有可能导致数据被覆盖或者删除。
  
a、多线程写入同个文件
  解决方法：加锁
  
b、多进程写入同个文件
  解决方法：加进程锁，如：pthread_mutex、文件锁（fcntl）
  
c、操作一个已关闭的fd。
  解决方法：检查fd是否已经关闭
```

##### 进程异常退出

```java
在写入文件的过程中，进程突然挂了 或者 被杀，也可能导致数据写入失败。
解决方法：mmap。直接来个内存映射，数据直接映射到page cache。内核会周期性地将page cache写回到磁盘的。
```



##### ② 文件系统维度

```java
内核崩溃 或 系统突然断电 都有可能导致文件系统损坏。
不过文件系统也做了很多的保护措施，如：system分区只读不可写，增加异常检查和恢复机制，ext4的fsck、f2fs的fsck.f2fs 和 checkpoint机制等。

更多情况是因 断电 而导致的写入丢失。
即使数据去到了page cache，也无法幸免于难。
因为page cache写回磁盘的时候断电，这部分未写回磁盘的数据将会丢失。
  
解决方法：如果某些数据我们觉得非常重要，是完全不允许有丢失风险的，这个时候我们应该采用同步写机制。在写入后使用 sync、fsync、msync 等系统调用，内核都会立刻将相应的数据写回到磁盘。
```



#### 2、IO突然变慢

##### ① 内存不住

```java
当手机内存不足的时候，系统会回收 Page Cache 和 Buffer Cache 的内存，大部分的写操作会直接落盘，导致性能低下。
```

##### ② 写入放大

```java
文件系统中，闪存重复写入需要先进行擦除操作，但这个擦除操作的基本单元是 block 块，一个 page 页的写入操作将会引起整个块数据的迁移，这就是典型的写入放大现象。低端机或者使用比较久的设备，由于磁盘碎片多、剩余空间少，非常容易出现写入放大的现象。
具体来说，闪存读操作最快，在 20us 左右。写操作慢于读操作，在 200us 左右。而擦除操作非常耗时，在 1ms 左右的数量级。当出现写入放大时，因为涉及移动数据，这个时间会更长。
  

闪存和固态硬盘中一种不期望的现象，即实际写入的物理信息量是将要写入的逻辑数量的多倍。因为闪存在可重新写入数据前必须先擦除，执行这些操作的过程就产生了一次以上的用户数据和元数据的移动（或重新写入）。此倍增效应会增加请求写入的次数，这会缩短SSD的寿命，从而减小SSD可靠运行的时间。增加的写入也会消耗闪存的带宽，这个效应主要会降低SSD的随机写入性能。许多因素会影响SSD的写入放大；一些可以由用户来控制，而另一些则是数据写入和SSD使用的直接结果。
```



#### 3、IO泄漏

```java
open了忘记close
```



#### 4、缓冲区太小

```java
缓冲区设置太小，导致调用read/write过多
```



### 三、IO优化方案

##### 标准IO读写优化

```java
合理设置Buffer大小，避免频繁的read/write调用
```



##### 大文件读写优化

```java
对大文件使用 mmap 或者 NIO 方式: MappedByteBuffer就是 Java NIO 中的 mmap 封装，对于大文件的频繁读写会有比较大的优化。
```



##### 海量小文件读写优化

```java
对于文件系统来说，目录查找的性能是非常重要的。
比如社交app图片可能有几万张，如果我们每张图片都是一个单独的文件，那目录下就会有几万个小文件，你想想这对IO的性能会造成什么影响？

文件的读取需要先找到存储的位置，在文件系统上面我们使用 inode 来存储目录。读取一个文件的耗时可以拆分成下面两个部分。
  
  文件读取的时间 = 找到文件的 inode 的时间 + 根据 inode 读取文件数据的时间
  
如果我们需要频繁读写几万个小文件，查找 inode 的时间会变得非常可观。这个时间跟文件系统的实现有关。

大量的小文件合并为大文件后，我们还可以将能连续访问的小文件合并存储，将原本小文件间的随机访问变为了顺序访问，可以大大提高性能。同时合并存储能够有效减少小文件存储时所产生的磁盘碎片问题，提高磁盘的利用率。
  
业界中Google的GFS、淘宝开源的TFS、Facebook的Haystack 都是专门为海量小文件的存储和检索设计的文件系统。
淘宝开源的TFS地址：https://github.com/alibaba/tfs
```



##### Buffer复用

```java
我们可以利用Okio开源库，它内部的 ByteString 和 Buffer 通过重用等技巧，很大程度上减少 CPU 和内存的消耗。
```



##### 序列化与存储结构选择

```java
对象序列化选择：
1、Serializable - 持久化存储
源码分析：https://blog.csdn.net/yudan505/article/details/115498934
优点：使用简单；序列化信息丰富；支持版本管理；
缺点：使用大量反射性能不够好；序列化后的大小比Class文件本身还要大很多；
  
2、Parcelable - 跨进程通讯
源码分析：https://blog.csdn.net/yudan505/article/details/115680320
优点：相比Serializable性能更优；
缺点：使用麻烦；不支持版本管理；


3、Serial（Twitter开源的高性能序列化方案）https://github.com/twitter/Serial
Serial在序列化与反序列化耗时、落地文件大小都有很大的优势。Serial就像是把 Parcelable 和 Serializable 的优点集合在一起的方案。
① 由于没有使用反射，相比起传统的反射序列化方案更加高效，具体你可以参考上面的测试数据。
② 开发者对于序列化过程的控制较强，可定义哪些 Object、Field 需要被序列化。
③ 有很强的 debug 能力，可以调试序列化的过程。
④ 有很强的版本管理能力，可以通过版本号和 OptionalFieldException 做兼容。


存储结构选择：
XML（能别用就别用了）
JSON（应用最广）
Protocol Buffers（最棒，有空学下）
```



##### 存储方式的选择

```java
数据量少：
	SharedPreferences
  MMKV
  DataStore
  
跨进程：
  ContentProvider
 
数据量大、构成关系复杂：
  SQLite
```



##### SQLite优化（待学习）

```java
数据库选择：
SQLite - Android本身自带
Realm - YCombinator出品的多版本并发控制数据库 - 我们项目就是用这货
LevelDB - Google的开源key-value数据库


SQLite相关：
Android开发高手课 - 数据库SQLite的使用和优化：https://time.geekbang.org/column/article/77546

1、ORM（Object Relational Mapping）对象关系映射
Android中最常用的ORM框架有 开源greenDAO 和 Google官方的Room。ORM框架使用真的非常简单，能很大地提高我们开发效率。
但是这不能是我们可以不去学习数据库基础知识的理由，只有理解底层的一些机制，我们才能更加得心应手地解决疑难的问题。

2、并发处理数据库
① 多进程 - 文件锁
SQLite默认是支持多进程并发操作的，它通过文件锁来控制多进程的并发。
SQLite锁的粒度并没有非常细，它针对的是整个DB文件，内部有5个状态：
未加锁(UNLOCKED)
共享 (SHARED)
保留 (RESERVED)
未决 (PENDING)
排它 (EXCLUSIVE)：数据库连接在断开前都不会释放SQLite文件的锁，从而避免不必要的冲突
SQLite使用锁逐步上升机制，为了写数据库连接需要逐级地获得排它锁。
多进程可以同时获取 SHARED 锁来读取数据，但是只有一个进程可以获取 EXCLUSIVE 锁来写数据库。

② 多线程 - WAL模式
Write Ahead Logging(预写日志)，它是用于实现原子事务的一种机制。

 WAL日志锁实质是锁wal-index文件的区域，根据不同的锁类型，将wal-index文件的不同区域划定义成不同的锁，主要有读锁，写锁，检查点锁。
 WAL模式下，最新的数据位于日志文件中，无论是读事务还是写事务都需要持有WAL_READ_LOCK的读锁，因为它们都需要获取最新的事务点。因此，做检查点时，可以通过对WAL_READ_LOCK位置(124-127)上锁，来确定检查点需要等待还是停止推进。同时我们也可以看到，对于DB文件，读写事务都只需要对DB文件上读锁，对于WAL日志文件，WAL_READ_LOCK和WAL_WRITE_LOCK位于不同的位置，读写相互不影响，所以读写不互斥。 


3、查询优化
① 索引优化
  todo
② 页大小与缓存大小
  todo
③ 其他优化
	慎用“select*”，需要使用多少列，就选取多少列。
	正确地使用事务。
	预编译与参数绑定，缓存被编译后的 SQL 语句。对于 blob 或超大的 Text 列，可能会超出一个页的大小，导致出现超大页。建议将这些列单独拆表，或者放到表字段的后面。
	定期整理或者清理无用或可删除的数据，例如朋友圈数据库会删除比较久远的数据，如果用户访问到这部分数据，重新从网络拉取即可。
```



### 四、IO监控 - Matrix（IOCanary）

#### 1、基本原理

##### ① hook点介绍

1、通过native hook open、read、write、close系统调用，来记录以及分析IO问题
2、IO泄漏监控则通过hook dalvik.system.CloseGuard来实现

```java
//调用链：IOCanaryPlugin#start -> IOCanaryCore#start -> IOCanaryCore#initDetectorsAndHookers
//IOCanaryCore#initDetectorsAndHookers 分析
private void initDetectorsAndHookers(IOConfig ioConfig) {
        ...
        if (...) {
            //hook open、read、write、close
            IOCanaryJniBridge.install(ioConfig, this);
        }

        if (ioConfig.isDetectIOClosableLeak()) {
            //hook dalvik.system.CloseGuard
            mCloseGuardHooker = new CloseGuardHooker(this);
            mCloseGuardHooker.hook();
        }
    }

//IOCanaryJniBridge#install -> IOCanaryJniBridge#doHook -> io_cannary_jni.cc#doHook
Java_com_tencent_matrix_iocanary_core_IOCanaryJniBridge_doHook(JNIEnv *env, jclass type) {
            for (int i = 0; i < TARGET_MODULE_COUNT; ++i) {
                const char* so_name = TARGET_MODULES[i];
                __android_log_print(ANDROID_LOG_INFO, kTag, "try to hook function in %s.", so_name);

                void* soinfo = xhook_elf_open(so_name);
                if (!soinfo) {
                    __android_log_print(ANDROID_LOG_WARN, kTag, "Failure to open %s, try next.", so_name);
                    continue;
                }

                xhook_got_hook_symbol(soinfo, "open", (void*)ProxyOpen, (void**)&original_open);
                xhook_got_hook_symbol(soinfo, "open64", (void*)ProxyOpen64, (void**)&original_open64);

                bool is_libjavacore = (strstr(so_name, "libjavacore.so") != nullptr);
                if (is_libjavacore) {
                    if (xhook_got_hook_symbol(soinfo, "read", (void*)ProxyRead, (void**)&original_read) != 0) {
                        __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook read failed, try __read_chk");
                        if (xhook_got_hook_symbol(soinfo, "__read_chk", (void*)ProxyReadChk, (void**)&original_read_chk) != 0) {
                            __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook failed: __read_chk");
                            xhook_elf_close(soinfo);
                            return JNI_FALSE;
                        }
                    }

                    if (xhook_got_hook_symbol(soinfo, "write", (void*)ProxyWrite, (void**)&original_write) != 0) {
                        __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook write failed, try __write_chk");
                        if (xhook_got_hook_symbol(soinfo, "__write_chk", (void*)ProxyWriteChk, (void**)&original_write_chk) != 0) {
                            __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook failed: __write_chk");
                            xhook_elf_close(soinfo);
                            return JNI_FALSE;
                        }
                    }
                }

                xhook_got_hook_symbol(soinfo, "close", (void*)ProxyClose, (void**)&original_close);
                xhook_got_hook_symbol(soinfo,"android_fdsan_close_with_tag",(void *)Proxy_android_fdsan_close_with_tag,(void**)&original_android_fdsan_close_with_tag);

                xhook_elf_close(soinfo);
            }
            ...
        }

//CloseGuardHooker#hook -> CloseGuardHooker#tryHook
private boolean tryHook() {
        try {
            Class<?> closeGuardCls = Class.forName("dalvik.system.CloseGuard");
            Class<?> closeGuardReporterCls = Class.forName("dalvik.system.CloseGuard$Reporter");
            @SuppressLint("SoonBlockedPrivateApi") // FIXME
            Method methodGetReporter = closeGuardCls.getDeclaredMethod("getReporter");
            Method methodSetReporter = closeGuardCls.getDeclaredMethod("setReporter", closeGuardReporterCls);
            Method methodSetEnabled = closeGuardCls.getDeclaredMethod("setEnabled", boolean.class);

            sOriginalReporter = methodGetReporter.invoke(null);

            methodSetEnabled.invoke(null, true);

            // open matrix close guard also
            MatrixCloseGuard.setEnabled(true);

            ClassLoader classLoader = closeGuardReporterCls.getClassLoader();
            if (classLoader == null) {
                return false;
            }

            methodSetReporter.invoke(null, Proxy.newProxyInstance(classLoader,
                new Class<?>[]{closeGuardReporterCls},
                new IOCloseLeakDetector(issueListener, sOriginalReporter)));

            return true;
        } catch (Throwable e) {
            MatrixLog.e(TAG, "tryHook exp=%s", e);
        }

        return false;
    }
```



##### ② hook open、read、write、close

1、open：创建IOInfo对象然后加入到map容器，key为fd
2、read/write：拿到map中对应IOInfo对象，更新统计数据
3、close：取出map中对应IOInfo对象，加入到队列中并且唤醒轮询线程

```java
//调用链：hook open -> io_cannary_jni.cc#ProxyRead -> io_cannary_jni.cc#DoProxyOpenLogic -> io_cannary.cc#IOCanary::OnOpen -> io_info_collector.cc#IOInfoCollector::OnOpen
void IOInfoCollector::OnOpen(const char *pathname, int flags, mode_t mode
            , int open_ret, const JavaContext& java_context) {
        ...
  		  //创建IOInfo对象，存入info_map_这个map容器里边，key为fd
        std::shared_ptr<IOInfo> info = std::make_shared<IOInfo>(pathname, java_context);
        info_map_.insert(std::make_pair(open_ret, info));
    }

//调用链：hook read -> ..... io_info_collector.cc#IOInfoCollector::OnRead
void IOInfoCollector::OnRead(int fd, const void *buf, size_t size, ssize_t read_ret, long read_cost) {
        ...
        CountRWInfo(fd, FileOpType::kRead, size, read_cost);
    }

//调用链：hook write -> ..... io_info_collector.cc#IOInfoCollector::OnWrite
void IOInfoCollector::OnWrite(int fd, const void *buf, size_t size, ssize_t write_ret, long write_cost) {
        ...
        CountRWInfo(fd, FileOpType::kWrite, size, write_cost);
    }

//调用链：IOInfoCollector::CountRWInfo 对每一次的IO操作进行统计，记录IO耗时、操作次数、缓冲区大小等信息
void IOInfoCollector::CountRWInfo(int fd, const FileOpType &fileOpType, long op_size, long rw_cost) {
        ...
        const int64_t now = GetSysTimeMicros();
        info_map_[fd]->op_cnt_ ++;
        info_map_[fd]->op_size_ += op_size;
        info_map_[fd]->rw_cost_us_ += rw_cost;
        if (rw_cost > info_map_[fd]->max_once_rw_cost_time_μs_) {
            info_map_[fd]->max_once_rw_cost_time_μs_ = rw_cost;
        }
        if (info_map_[fd]->last_rw_time_μs_ > 0 && (now - info_map_[fd]->last_rw_time_μs_) < kContinualThreshold) {
            info_map_[fd]->current_continual_rw_time_μs_ += rw_cost;
        } else {
            info_map_[fd]->current_continual_rw_time_μs_ = rw_cost;
        }
        if (info_map_[fd]->current_continual_rw_time_μs_ > info_map_[fd]->max_continual_rw_cost_time_μs_) {
            info_map_[fd]->max_continual_rw_cost_time_μs_ = info_map_[fd]->current_continual_rw_time_μs_;
        }
        info_map_[fd]->last_rw_time_μs_ = now;
        if (info_map_[fd]->buffer_size_ < op_size) {
            info_map_[fd]->buffer_size_ = op_size;
        }
        if (info_map_[fd]->op_type_ == FileOpType::kInit) {
            info_map_[fd]->op_type_ = fileOpType;
        }
    }

//hook close -> ..... io_cannary.cc#IOCanary::OnOpen -> io_info_collector.cc#IOInfoCollector::OnClose
//IOInfoCollector::OnClose 统计总耗时、文件总大小，从map容器中取出IOInfo对象并且返回
std::shared_ptr<IOInfo> IOInfoCollector::OnClose(int fd, int close_ret) {
        ...

        info_map_[fd]->total_cost_μs_ = GetSysTimeMicros() - info_map_[fd]->start_time_μs_;
        info_map_[fd]->file_size_ = GetFileSize(info_map_[fd]->path_.c_str());
        std::shared_ptr<IOInfo> info = info_map_[fd];
        info_map_.erase(fd);
        return info;
    }

//IOCanary::OnOpen
void IOCanary::OnClose(int fd, int close_ret) {
        std::shared_ptr<IOInfo> info = collector_.OnClose(fd, close_ret);
        ...
        OfferFileIOInfo(info);
    }

//IOCanary::OfferFileIOInfo 将IOInfo对象加入队列，然后唤醒轮询线程
void IOCanary::OfferFileIOInfo(std::shared_ptr<IOInfo> file_io_info) {
        std::unique_lock<std::mutex> lock(queue_mutex_);
        queue_.push_back(file_io_info); //加入队列中
        queue_cv_.notify_one(); //唤醒轮询线程
        lock.unlock();
    }
```



##### ③ 分析与上报逻辑

1、轮询线程会一直轮询queue_队列，从里边取出IOInfo对象进行分析。如果队列没数据就进入等待
   在 close 函数会将IOInfo对象加入queue_队列，并且唤醒轮询线程的。
2、根据探测器们对IOInfo对象的分析，将分析结果回调给java层 IOCanaryJniBridge#onIssuePublish

```java
//IOCanary的构造函数里边会启动一个线程，执行 IOCanary::Detect 函数
IOCanary::IOCanary() {
        exit_ = false;
        std::thread detect_thread(&IOCanary::Detect, this);
        detect_thread.detach();
    }

//IOCanary::Detect while (true)轮询
void IOCanary::Detect() {
        std::vector<Issue> published_issues;
        std::shared_ptr<IOInfo> file_io_info;
        while (true) {
            published_issues.clear();
            int ret = TakeFileIOInfo(file_io_info); //取IOInfo对象
            if (ret != 0) { break; }
            //遍历探测器们，将IOInfo对象丢给他们分析，然后探测器会把分析结果加入到 vector<Issue>
            for (auto detector : detectors_) {
                detector->Detect(env_, *file_io_info, published_issues);
            }

            //上报分析结果到java层
            //JNI_OnLoad -> IOCanary::Get().SetIssuedCallback -> 赋值 issued_callback_ 回调
            //其实最终走到java层 IOCanaryJniBridge#onIssuePublish(ArrayList<IOIssue> issues)
            if (issued_callback_ && !published_issues.empty()) {
                issued_callback_(published_issues);
            }

            file_io_info = nullptr;
        }
    }

//IOCanary::TakeFileIOInfo 从queue_队列中取数据，如果队列为空就等待
int IOCanary::TakeFileIOInfo(std::shared_ptr<IOInfo> &file_io_info) {
        std::unique_lock<std::mutex> lock(queue_mutex_);
        while (queue_.empty()) {
            queue_cv_.wait(lock); //等待
            if (exit_) {
                return -1;
            }
        }

        file_io_info = queue_.front();
        queue_.pop_front();
        return 0;
    }
```



##### ④ 探测器

1、探测器是根据开发者配置来生成的，不同的探测器分析不同的IO问题。

2、探测器生成后会加入到 detectors_ 容器当中，在轮询线程会进行detectors_ 容器的遍历。

```java
//调用链：IOCanaryJniBridge#install -> IOCanaryJniBridge#enableDetector -> io_cannary_jni.cc#enableDetector -> io_cannary.cc#IOCanary::RegisterDetector

//IOCanary::RegisterDetector 根据type生成不同的探测器并且加入到detectors_容器
void IOCanary::RegisterDetector(DetectorType type) {
        switch (type) {
            case DetectorType::kDetectorMainThreadIO: //探测主线程是否存在IO操作
                detectors_.push_back(new FileIOMainThreadDetector());
                break;
            case DetectorType::kDetectorSmallBuffer: //探测缓冲区太小
                detectors_.push_back(new FileIOSmallBufferDetector());
                break;
            case DetectorType::kDetectorRepeatRead: //重复读同一文件
                detectors_.push_back(new FileIORepeatReadDetector());
                break;
            default:
                break;
        }
    }
```



#### 2、主线程是否存在IO操作（FileIOMainThreadDetector）

```java
//main_thread_detector.cc#FileIOMainThreadDetector::Detect
void FileIOMainThreadDetector::Detect(const IOCanaryEnv &env, const IOInfo &file_io_info, std::vector<Issue>& issues) {
        //判断是否在主线程
        if (GetMainThreadId() == file_io_info.java_context_.thread_id_) {
            int type = 0;
            //单次最大读写耗时 > 13*1000微妙 (1帧刷新周期16ms的80%)
            if (file_io_info.max_once_rw_cost_time_μs_ > IOCanaryEnv::kPossibleNegativeThreshold) {
                type = 1;
            }
            //最大持续读写耗时 > 开发者配置的监控阈值
            if(file_io_info.max_continual_rw_cost_time_μs_ > env.GetMainThreadThreshold()) {
                type |= 2;
            }

            if (type != 0) { //输出分析报告
                Issue issue(kType, file_io_info);
                issue.repeat_read_cnt_ = type;  //use repeat to record type
                PublishIssue(issue, issues);
            }
        }
    }
```



#### 3、缓冲区太小（FileIOSmallBufferDetector）

```java
//small_buffer_detector.cc#FileIOSmallBufferDetector::Detect
void FileIOSmallBufferDetector::Detect(const IOCanaryEnv &env, const IOInfo &file_io_info, std::vector<Issue>& issues) {
        //操作次数 > 20次
        //累计操作的大小 /  操作次数 < 开发者配置的监控阈值
        //最大持续读写耗时 >= 13*1000微妙 (1帧刷新周期16ms的80%)
        if (file_io_info.op_cnt_ > env.kSmallBufferOpTimesThreshold
            && (file_io_info.op_size_ / file_io_info.op_cnt_) < env.GetSmallBufferThreshold()
            && file_io_info.max_continual_rw_cost_time_μs_ >= env.kPossibleNegativeThreshold) {
            PublishIssue(Issue(kType, file_io_info), issues); //输出分析报告
        }
    }
```



#### 4、重复读同一文件（FileIORepeatReadDetector）

```java
//repeat_read_detector.cc#FileIORepeatReadDetector::Detect
void FileIORepeatReadDetector::Detect(const IOCanaryEnv &env,const IOInfo &file_io_info,std::vector<Issue>& issues) {
        const std::string& path = file_io_info.path_;
        if (observing_map_.find(path) == observing_map_.end()) {
            //最大持续读写耗时 < 13*1000微妙 (1帧刷新周期16ms的80%)，就放过它吧避免频繁上报
            if (file_io_info.max_continual_rw_cost_time_μs_ < env.kPossibleNegativeThreshold) {
                return;
            }
            //加入 observing_map_ 容器，key为文件路径，value为RepeatReadInfo对象的向量
            observing_map_.insert(std::make_pair(path, std::vector<RepeatReadInfo>()));
        }

        std::vector<RepeatReadInfo>& repeat_infos = observing_map_[path];
        ...

        //给本次的IOInfo对象，构建一个RepeatReadInfo对象
        RepeatReadInfo repeat_read_info(file_io_info.path_, file_io_info.java_context_.stack_, file_io_info.java_context_.thread_id_,
                                      file_io_info.op_size_, file_io_info.file_size_);
        ...

        bool found = false;
        int repeatCnt;
        //遍历向量，找出是否存在重复读
        for (auto& info : repeat_infos) {
            if (info == repeat_read_info) {
                found = true;
                info.IncRepeatReadCount();
                repeatCnt = info.GetRepeatReadCount(); //记录次数
                break;
            }
        }

        ...

        //如果重复读次数超过开发者配置的监控阈值，输出分析报告
        if (repeatCnt >= env.GetRepeatReadThreshold()) {
            Issue issue(kType, file_io_info);
            issue.repeat_read_cnt_ = repeatCnt;
            issue.stack = repeat_read_info.GetStack();
            PublishIssue(issue, issues);
        }
    }
```



#### 5、IO泄漏（hook CloseGuard）

#####  CloseGuard知识扩充

```java
StrictMode针对单个线程和虚拟机的所有对象都定义了检查策略，用来发现一些违规操作，譬如：主线程中的磁盘读/写、网络访问、未关闭cursor，这些操作都能够被StrictMode检查出来。 
怎么做到的呢？在做这些操作时，植入StrictMode的检查代码就可以了。有一部分植入代码是建立在BlockGuard和CloseGuard之上的，可以说，StrictMode是建立在BlockGuard和CloseGuard之上的机制。

CloseGurad提供了一种机制或者说是一个工具类，用来记录资源泄露的场景，比如使用完的资源(比如cursor/fd)没有正常关闭。
  
//Android已实现了资源泄漏监控的功能，它是基于工具类 CloseGuard 来实现的。
//以FileInputStream为例，在GC准备回收FileInputStream时，会调用guard.warnIfOpen来检测是否关闭了IO流：
public class FileInputStream extends InputStream {
    private final CloseGuard guard = CloseGuard.get(); // 1. 新建CloseGuard全局变量

    public FileInputStream(File file) throws FileNotFoundException {
        ...
        guard.open("close"); // 2. 设置CloseGuard标志
    }

    public void close() throws IOException {
        guard.close(); // 3. 清除CloseGuard标志
        ...
    }

    protected void finalize() throws IOException {
        if (guard != null) {
            guard.warnIfOpen(); // 4. 判断Close标志是否被清除
        }
        ...
    }
}

//CloseGuard 的部分源码如下：
//执行warnIfOpen时如果未关闭IO流，就调用reporter的report方法。利用反射把reporter换成自己的就行了
//很多代码都用了CloseGuard，因此，诸如文件资源没 close、Cursor 没有 close 等问题都能通过它来检测。
//CloseGuard#warnIfOpen
public void warnIfOpen() {
        if (closerNameOrAllocationInfo != null) {
            if (closerNameOrAllocationInfo instanceof Throwable) {
                reporter.report(MESSAGE, (Throwable) closerNameOrAllocationInfo);
            } else if (stackAndTrackingEnabled) {
                reporter.report(MESSAGE + " Callsite: " + closerNameOrAllocationInfo);
            } else {
                ...
            }
        }
    }
```



##### 代码分析

```java
//调用链：CloseGuardHooker#tryHook -> 代理CloseGuard$Reporter -> IOCloseLeakDetector#invoke（关注report方法调用）
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().equals("report")) {
            ...
            String stackKey = IOCanaryUtil.getThrowableStack(throwable);
            if (isPublished(stackKey)) {
                ...
            } else {
                Issue ioIssue = new Issue(SharePluginInfo.IssueType.ISSUE_IO_CLOSABLE_LEAK);
                ioIssue.setKey(stackKey);
                JSONObject content = new JSONObject();
                try {
                    content.put(SharePluginInfo.ISSUE_FILE_STACK, stackKey);
                } catch (JSONException e) { ... }
                ioIssue.setContent(content);
                publishIssue(ioIssue); //OnIssueDetectListener#onDetectIssue -> IOCanaryCore#onDetectIssue
                markPublished(stackKey);
            }


            return null;
        }
        return method.invoke(originalReporter, args);
    }
```



### 五、Key-Value存储框架分析

#### 1、SharedPreferences、MMKV、DataStore对比

```java
扔物线视频，推荐！！！：https://www.bilibili.com/video/BV1FU4y197dL?spm_id_from=333.999.0.0 

DataStore
解决SP的两个问题
  1、性能问题
	DataStore读跟写都是使用协程在子线程中完成的。
  2、回调问题
	场景：等数据写入之后再做某某事情
	使用SP的commit同步写入会卡主线程
	使用SP的apply异步写入你还需要设计回调机制
	DataStore使用协程线程切换是非常简单的
缺点：需要在kotlin、协程的支持下才能使用 


MMKV
优点：
  1、极高的同步写入磁盘速度
  2、高性能的高频写入小数据
  3、支持多进程
  4、进程崩溃不会影响写入
缺点：
  1、系统崩溃造成文件损坏的情况下会有可能导致丢数据。(SP、DataStore有内存缓存，不会出现这个问题)
  所以如果数据重要，需要自己手动备份数据。 
  2、写入大数据、初次加载文件（特别是文件数据特别大）会有可能造成卡顿的
```



#### 2、MMKV（mmap + 文件锁）学习

```java
文章推荐：https://cloud.tencent.com/developer/article/1354199

IPC选择：
	中心化框架：Binder（CS架构，缺点：启动慢（需要Server端的启动）、访问慢（各种鉴权、各种切binder线程））、其他框架（ socket、PIPE等就更慢了，需要两次拷贝）
	去中心化框架：既然用了mmap，它就支持进程通信了，进程间同步通过进程锁实现就好了。
进程锁选择：
	pthread_mutex：支持递归锁、锁升级降级。不支持进程退出自动释放锁，需要自行释放（可能会出现饿死）。
	文件锁（fcntl）：不支持进程退出自动释放锁。但是不支持递归锁、锁升级降级，需要自行实现。

实现细节（多进程的状态同步问题）：
文件头部保存有状态信息，各个进程也缓存自己的状态信息。如果发现缓存与文件头部的不一致，说明有进程写入了，则进程需要更新缓存。
写指针感知：
	保存到状态信息，进程在写入后都把写指针更新到状态信息里边。
	其他进程获得写锁之后，读取下状态信息的写指针是否与自己缓存的一致，即可感知到。
内存重整感知：
	使用一个单调递增的序列号，每次发生内存重整将序列号递增，并保存到状态信息中。
内存增长感知：
	无需额外保存，获取文件大小即可感知。

文件锁的完善：
支持递归锁：
	什么是递归锁：一个进程/线程已经拥有了锁，那么后续的加锁操作不会导致卡死，并且解锁也不会导致外层的锁被解掉。对于文件锁来说，前者是满足后者则不满足。因为文件锁是状态锁，没有计数器，无论加了多少次锁，一个解锁操作就全解掉。

	解决方法：增加锁计数

支持升级、降级锁：
	什么是锁升级：已经持有的共享锁（读锁），升级为互斥锁（写锁）。
	什么是锁降级：锁降级则是反过来。
	文件锁支持锁升级但是容易死锁：假如A、B进程都持有了读锁，现在都想升级到写锁，就会陷入相互等待的困境，发生死锁。
	另外由于文件锁不支持递归锁，也导致了锁降级无法进行，一降就降到没有锁。

	解决方法：
	加写锁时，如果当前已经持有读锁，那么先尝试加写锁，try_lock 失败说明其他进程持有了读锁，我们需要先将自己的读锁释放掉，再进行加写锁操作，以避免死锁的发生。
	
	解写锁时，假如之前曾经持有读锁，那么我们不能直接释放掉写锁，这样会导致读锁也解了。我们应该加一个读锁，将锁降级。
```



#### 3、SharedPreferences 引发的卡顿问题（严重甚至会发生ANR）

##### 问题1 SP首次启动，用线程读取文件内容就不会有事了吗？

在SP创建的时候，会开一个线程去解析sp文件并将sp内容加载到内存中。如果在解析的过程，主线程尝试访问SP就会被Block，直到解析完成。

```java
问题分析：SharedPreferencesImpl构造函数 -> startLoadFromDisk
	private void startLoadFromDisk() {
		//mLoaded设置false，sp文件就绪就会设置为true。并且mLock.notifyAll
        synchronized (mLock) { mLoaded = false; }
        new Thread("SharedPreferencesImpl-load") { ... }.start();
    }


   //SP的所有操作，都会先获取锁，然后再判断mLoaded
	synchronized (mLock) {
        awaitLoadedLocked();
        ...
    }

    private void awaitLoadedLocked() {
        if (!mLoaded) {
            BlockGuard.getThreadPolicy().onReadFromDisk();
        }
        while (!mLoaded) {
            try {
                mLock.wait(); //如果sp没就绪，那么这里就会进入Block
            } catch (InterruptedException unused) { }
        }
        ...
    }

解决方法：采用预加载的方式。
真正需要处理的是核心场景的SP一定不能太大，官方的声明还是有必要遵守一下.
```



##### 问题2 commit同步写入是阻塞的，apply异步写入就不会阻塞了吗？

```java
问题分析：
SP 调用 apply 方法，会创建一个等待锁放到 QueuedWork 中，并将真正数据持久化封装成一个任务放到异步队列中执行，任务执行结束会释放锁。
执行 QueuedWork.waitToFinish() 会等待所有的等待锁释放。
太多 pending 的 apply 行为没有写入到文件，主线程在执行到QueuedWork#waitToFinish的时候会有等待行为，会造成卡顿甚至出现ANR

apply的流程:
1、首先调用commitToMemory将数据改动同步到内存中，也就是SharedPreferencesImpl的mMap

2、将一个awaitCommit的Runnable加入到QueuedWork的finisher队列中
   这个awaitCommit的Runnable的逻辑是 mcr.writtenToDiskLatch.await(); 等待这次mMap的文件写入完成。

3、最后将执行mMap写入文件的逻辑封装在一个Runnable里边，并将其添加到QueuedWork的工作队列中（QueuedWork#queue）
   在文件写入完成之后，会执行writtenToDiskLatch.countDown() 唤醒等待并且移除掉finisher队列中的awaitCommit。

4、QueuedWork#queue 处了将Runnable加入到工作队列之外，还会进行sendMsg操作。让QueuedWork内部的HandleThread工作起来，
   执行工作队列中的所有Runnable。

QueuedWork：执行异步任务的工具类，内部的实现逻辑的就是创建一个HandlerThread作为工作线程，然后QueuedWorkHandler和这个HandlerThread进行管理，每当有任务添加进来就在这个异步线程中执行，这个异步线程的名字queued-work-looper

ActivityThread在处理组件生命周期中，分别有4处地方调用了 QueuedWork#waitToFinish
分别是：Activity#onPause、Activity#onStop、Service#onStartCommand、Service#onDestroy

waitToFinish：（这个方法是执行在主线程的）
1、如果工作队列还有Runnable，继续执行完它们
2、如果finisher队列有Runnable，执行完它们
   问题就出在这：
   apply的中写入操作也是在异步线程执行，不会导致主线程卡顿。
   但是如果异步任务执行时间过长，当ActvityThread执行到了QueuedWork#waitToFinish 
   就会进入 awaitCommit这个Runnable里边的 await 等待。
  
  
解决方法：hook掉finisher队列，让其poll方法返回null。
```



### 六、日志框架 - xlog

```java
源码路径（xlog的源码是混在mars里边的）：https://github.com/Tencent/mars
文章推荐：https://cloud.tencent.com/developer/article/1005575


1、日志写入：mmap 保证高性能又能保证高可靠性

  
2、日志压缩：短语式压缩(LZ77编码) -> ascci编码 -> huffman编码
短语式压缩(LZ77编码)：https://blog.csdn.net/qq_34244712/article/details/103678481
有两个滑动窗口，历史滑动窗口和前向缓存窗口，在前向缓存窗口中通过和历史滑动窗口中的内容进行匹配从而进行编码。

比如这句绕口令：吃葡萄不吐葡萄皮，不吃葡萄倒吐葡萄皮。中间是有两块重复的内容“吃葡萄”和“吐葡萄皮”这两块。第二个“吃葡萄”的长度是 3 和上个“吃葡萄”的距离是 10 ，所以可以用 (10,3) 的值对来表示，同样的道理“吐葡萄皮”可以替换为 (10,4 )

ascci编码：就是字符转0-255的整数

huffman编码：整数常用的压缩方案

压缩逻辑：收集到一定大小的日志作为一个压缩单位，进行压缩写入。
好处：某段日志损坏了也不会影响其他的日志段；防止大量日志压缩导致CPU曲线短时间内极速升高进而可能会导致程序卡顿。

  
3、日志安全：非对称加密
```

