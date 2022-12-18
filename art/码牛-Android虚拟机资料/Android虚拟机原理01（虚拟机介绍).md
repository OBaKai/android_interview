# Android虚拟机原理01（虚拟机介绍）

> **码牛学院用代码码出自己牛逼的人生**

#### 1. Android虚拟机的生命周期

- 启动。启动一个App程序时，一个JVM实例就产生了，任何一个拥有

  > public static void main(String[] args)

  函数的class都可以作为JVM实例运行的起点。

- 运行。main()作为该程序初始线程的起点，任何其他线程均由该线程启动。

- 消亡。当程序中的所有非守护线程都终止时，JVM才退出；若安全管理器允许，程序也可以使用Runtime类或者System.exit()来退出。

　　一个运行中的Java虚拟机有着一个清晰的任务：执行Java程序。程序开始执行时他才运行，程序结束时他就停止。你在同一台机器上运行三个程序，就会有三个运行中的Java虚拟机。 Java虚拟机总是开始于一个main()方法，这个方法必须是公有、返回void、直接受一个字符串数组。在程序执行时，你必须给Java虚拟机指明这个包换main()方法的类名。main()方法是程序的起点，他被执行的线程初始化为程序的初始线程。程序中其他的线程都由他来启动。

　　Java中的线程分为两种：守护线程 （daemon）和普通线程（non-daemon）。守护线程是Java虚拟机自己使用的线程，比如负责垃圾收集的线程就是一个守护线程。当然，你也可以把自己的程序设置为守护线程。包含main()方法的初始线程不是守护线程。

　　只要Java虚拟机中还有普通的线程在执行，Java虚拟机就不会停止。如果有足够的权限，你可以调用exit()方法终止程序。

###  

#### 2.Android体系结构

> 　 类装载器（ClassLoader）（用来装载.class文件）
>
> 　  执行引擎（执行字节码，或者执行本地方法）
>
> 　 运行时数据区（方法区、堆、java栈、PC寄存器、本地方法栈）

 

#### 3. JVM运行时数据区

![img](https://img-blog.csdnimg.cn/20190429152539760.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpX3hpYW9fbWluZw==,size_16,color_FFFFFF,t_70)

##### 3.1 Java堆(Heap)

- 被所有线程共享的一块内存区域，在虚拟机启动时创建
- 用来**存储对象实例**
- 可以通过-Xmx和-Xms控制堆的大小
- OutOfMemoryError异常：当在堆中没有内存完成实例分配，且堆也无法再扩展时。

　　java堆是垃圾收集器管理的主要区域。java堆还可以细分为：新生代（New/Young）、旧生代/年老代（Old/Tenured）。持久代（Permanent）在**方法区**，不属于Heap。

![img](https://img-blog.csdnimg.cn/20190429152606825.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpX3hpYW9fbWluZw==,size_16,color_FFFFFF,t_70)

 

**新生代：**新建的对象都由新生代分配内存。常常又被划分为Eden区和Survivor区。Eden空间不足时会把存活的对象转移到Survivor。新生代的大小可由-Xmn控制，也可用-XX:SurvivorRatio控制Eden和Survivor的比例。

**旧生代：**存放经过多次垃圾回收仍然存活的对象。

持久代：存放静态文件，如今Java类、方法等。持久代在方法区，对垃圾回收没有显著影响。

##### 3.2 方法区

- 线程间共享
- 用于**存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据**
- OutOfMemoryError异常：当方法区无法满足内存的分配需求时
- 运行时常量池
  - 方法区的一部分
  - 用于存**放编译期生成的各种字面量与符号引用**，如String类型常量就存放在常量池
  - OutOfMemoryError异常：当常量池无法再申请到内存时

##### 3.3 java虚拟机栈（VM Stack）

- 线程私有，生命周期与线程相同
- 存储方法的局部变量表(**基本类型**、对象引用)、操作数栈、动态链接、方法出口等信息。
- java方法执行的内存模型，每个方法执行的同时都会创建一个栈帧，每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。
- StackOverflowError异常：当线程请求的栈深度大于虚拟机所允许的深度
- OutOfMemoryError异常：如果栈的扩展时无法申请到足够的内存

　　JVM栈是线程私有的，每个线程创建的同时都会创建JVM栈，JVM栈中存放的为当前线程中局部**基本类型**的变量、部分的返回结果以及Stack Frame。其他引用类型的对象在JVM栈上仅存放**变量名**和指向堆上对象实例的**首地址**。

##### 3.4 本地方法栈（Native Method Stack）

- 与虚拟机栈相似，主要为虚拟机使用到的Native方法服务，在HotSpot虚拟机中直接把本地方法栈与虚拟机栈二合一

##### 3.5 程序计数器（Program Counter Register）

- 当前线程所执行的字节码的行号指示器
- 当前线程私有
- 不会出现OutOfMemoryError情况

##### 3.6 直接内存（Direct Memory）

- 直接内存并不是虚拟机运行的一部分，也不是Java虚拟机规范中定义的内存区域，但是这部分内存也被频繁使用
- NIO可以使用Native函数库直接分配堆外内存，堆中的DirectByteBuffer对象作为这块内存的引用进行操作
- 大小不受Java堆大小的限制，受本机(服务器)内存限制
- OutOfMemoryError异常：系统内存不足时

　　**总结：**Java对象实例存放在堆中；常量存放在方法区的常量池；虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据放在方法区；以上区域是所有线程共享的。栈是线程私有的，存放该方法的局部变量表(基本类型、对象引用)、操作数栈、动态链接、方法出口等信息。

　　一个Java程序对应一个JVM，一个方法（线程）对应一个Java栈。



#### 4. Java代码的编译APK和执行过程

**Java代码的编译和执行包括了三个重要机制：**

1. Java源码编译机制（.java源代码文件 -> .class字节码文件-》dex文件->apk）
2. 类加载机制（ClassLoader）
3. 类执行机制（Android执行引擎）

 

 