## Java

#### 说说你对oop的理解，结合项目说一下

```java
oop：面向对象编程
oop的特点是：封装、继承、多态
oop优点：让程序易维护、易复用、易扩展

结合项目来说的话，以mvp框架为例吧
在mvp框架中，在p层会与v层、m层建立双向连接，让v层与m层彻底断开联系。

p层的类我们一般会设计一个 baseP 这样的父类，然后各个业务模块的p层子类继承自这个 baseP，体现了oop继承特性。
baseP设计的时候，我们是需要持有v层接口的实现，为了方便扩展，一般我们定义泛型 T extends baseView 接口（v层base接口）。当p层的子类需要到某个具体v层接口的时候，在给这个泛型T传入实际接口类，体现了oop多态特效。
p层的这个类，我们会封装一些方法暴露出去，当v层需要某种能力的时候，就可以调用这个p层的对象的方法，让这个p层对象帮我们实现，体现了oop封装特性。
```



#### 重写和重载区别

```java
1.重写(Override)：父类与子类之间的多态表现
其实就是在子类中把父类本身有的方法重新写一遍
例如：a extends A
A有个方法 set(){ "AAA" }
a重写了A中的set方法，set(){ "aaa" }

2.重载(Overload)：一个类中多态表现
多个具有不同的参数个数或者类型的同名函数（返回值类型可随意，不能以返回类型作为重载函数的区分标准），调用方法时通过传递不同参数个数和参数类型来决定具体使用哪个方法的多态性。
例如：
AView(Context c)
AView(Context c，AttributeSet att)
AView(Context c，AttributeSet att, int style)

https://blog.csdn.net/qunqunstyle99/article/details/81007712
```





#### 并发编程

（并发的更多问题：[Java并发编程 I - 并发问题的源头](https://blog.csdn.net/yudan505/article/details/117841171)）

1. 为什么会出现并发问题？

2. volatile能解决并发中的什么问题？

3. ThreadLocal怎么保证线程唯一？

4. sleep与wait的区别。+2

5. synchronize修饰方法和修饰静态方法的区别。+2

6. 写出一个死锁的例子。

7. List的加锁要如何加。

8. 多线程加锁的几种方法。+2

9. 开启线程的三种方式？

10. 线程和进程的区别？+2

11. run()和start()方法区别

12. 两个进程同时要求写或者读，能不能实现？如何防止进程的同步？

    

#### JVM

```java
运行时数据区域：
java虚拟机在执行程序时会把它所管理的内存划分成若干个不同的数据区域。

运行时数据区域包括：
方法区（运行时常量池）
堆（java堆）
虚拟机栈
本地方法栈
程序计数器

线程共享：方法区、堆
1、方法区：存储类信息、常量、静态变量、即时编译期编译后的代码

2、堆：存储对象实例、数组
（堆大小设置，最大上限：-Xmx，初始内存大小分配：-Xms）

线程私有：程序计数器、虚拟机栈、本地方法栈

1、程序计数器：指向当前线程正在执行的字节码指令地址
为什么需要程序计数器？
线程切换的时候需要记录当前线程执行的位置，确保多线程情况下程序的政策执行。

2、虚拟机栈：存储当前线程运行方法所需要的数据、指令、返回地址。
虚拟机栈里边存储的是栈帧。
栈帧里边内容包括：局部变量表、操作数栈、返回地址、动态连接 。
局部变量表（LocalVariableTable）：第一个位置存储的是this（类对象本身），之后存储的是方法的传参。
操作数栈（Code）：一大堆指令，程序就是依靠一条条指令执行实现的。
返回地址：记录返回地址，方法正常执行完后告诉虚拟机需要返回到哪里。
	比如：a方法里边执行了b方法，b的栈帧的返回地址就会记录a方法的地址，当b方法执行完了就会回到a方法对应的地方。（如果执行中发生异常，是去到异常处理器表）
动态连接：用来实现多态的。在类加载的时候记录当前具体是哪个实例的方法。
	比如：B、C类继承自A，B、C分别实现了A中的aa方法，B、C实现的aa方法是有区别的。aa方法中的动态连接就是为了记录当前实例到底是B还是C。

注意：虚拟机栈是有大小的（大小设置：-Xss），如果递归很深可能会导致虚拟机栈溢出（StackOverflowError）。

3、本地方法栈：存储当前线程所使用的native方法的信息。
当线程里边有调用native方法的时候，并不会在虚拟机栈中创建栈帧，而是在本地方法栈中创建。



虚拟机中的对象
虚拟机遇到一条new指令时：
1）检查加载
先执行相应的类加载过程。
2）分配内存
	指针碰撞
		接下来虚拟机将为新生对象分配内存。为对象分配空间的任务等同于把一块确定大小的内存从Java堆中划分出来。
		如果Java堆中内存是绝对规整的，所有用过的内存都放在一边，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把那个指针向空闲空间那边挪动一段与对象大小相等的距离，这种分配方式称为“指针碰撞”。
	空闲列表
		如果Java堆中的内存并不是规整的，已使用的内存和空闲的内存相互交错，那就没有办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记录上哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录，这种分配方式称为“空闲列表”。
		选择哪种分配方式由Java堆是否规整决定，而Java堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定。
	并发安全
		除如何划分可用空间之外，还有另外一个需要考虑的问题是对象创建在虚拟机中是非常频繁的行为，即使是仅仅修改一个指针所指向的位置，在并发情况下也并不是线程安全的，可能出现正在给对象A分配内存，指针还没来得及修改，对象B又同时使用了原来的指针来分配内存的情况。
	CAS机制
		解决这个问题有两种方案，一种是对分配内存空间的动作进行同步处理——实际上虚拟机采用CAS配上失败重试的方式保证更新操作的原子性；
		分配缓冲
		另一种是把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块私有内存，也就是本地线程分配缓冲（Thread Local Allocation Buffer,TLAB），如果设置了虚拟机参数 -XX:UseTLAB，在线程初始化时，同时也会申请一块指定大小的内存，只给当前线程使用，这样每个线程都单独拥有一个Buffer，如果需要分配内存，就在自己的Buffer上分配，这样就不存在竞争的情况，可以大大提升分配效率，当Buffer容量不够的时候，再重新从Eden区域申请一块继续使用。
		TLAB的目的是在为新对象分配内存空间时，让每个Java应用线程能在使用自己专属的分配指针来分配空间，减少同步开销。
		TLAB只是让每个线程有私有的分配指针，但底下存对象的内存空间还是给所有线程访问的，只是其它线程无法在这个区域分配而已。当一个TLAB用满（分配指针top撞上分配极限end了），就新申请一个TLAB。
3）内存空间初始化
（注意不是构造方法）内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值(如int值为0，boolean值为false等等)。这一步操作保证了对象的实例字段在Java代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。
4）设置
接下来，虚拟机要对对象进行必要的设置，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。这些信息存放在对象的对象头之中。
5）对象初始化
在上面工作都完成之后，从虚拟机的视角来看，一个新的对象已经产生了，但从Java程序的视角来看，对象创建才刚刚开始，所有的字段都还为零值。所以，一般来说，执行new指令之后会接着把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全产生出来。


虚拟机优化：逃逸分析
几乎所有的对象都是在堆中分配的，但是也有例外。

逃逸分析：可以在栈上创建对象
基本思想：对于线程私有的对象，将它打散分配在栈上而不分配在堆上。好处是对象跟着方法调用自行销毁，不需要进行垃圾回收，可以提高性能。
逃逸分析的目的是判断对象的作用域是否会逃逸出方法体。注意，任何可以在多个线程之间共享的对象，一定都属于逃逸对象。
public void test(int x,inty ){
String x = “”;
User u = ….
….. 
}
User类型的对象u就没有逃逸出方法test。
public  User test(int x,inty ){
String x = “”;
User u = ….
….. 
return u;
}
User类型的对象u就逃逸出方法test。

如何启用栈上分配（+是开启，-是关闭）
-XX:+DoEscapeAnalysis：开启逃逸分析(默认打开)
-XX:+UseTLAB 开启本地线程分配缓冲（默认打开，只有开启了 本地线程分配缓冲才能使用栈上分配）

-XX:+EliminateAllocations：标量替换(默认打开，)

-XX:+PrintGC：开启GC日志


测试：a方法内创建一个对象，然后a方法循环调用一亿次。
关闭逃逸分析：会出现频繁gc，最终一亿次执行时长大概接近1秒（这一亿个对象都是在堆创建的，导致出现大量gc）
开启逃逸分析：一亿次执行时长大概7、8毫秒（栈上创建，只创建了一次对象，创建到了缓存中。重复执行a方法直接从缓存取了）

```





#### GC

##### 说说对垃圾回收器的理解。+3

```java
理解：自动将Java堆中不需要的对象进行回收
  
堆内存划分：（新生代:老年代 = 1:2）
新生代（PSYoungGen）
	Eden区:两个S区 = Eden:From:To = 8:1:1
老年代（ParOldGen）

堆内存分配策略：
1、对象优先在Eden分配，如果说Eden内存空间不足，就会发生Minor GC
2、大对象直接进入老年代。大对象：需要大量连续内存空间的对象，比如很长的字符串或大型数组。
3、长期存活的对象将进入老年代。默认15岁，-XX:MaxTenuringThreshold调整
	对象头会记录对象的年龄，当对象在S区中被复制一次，年龄就会+1岁。当达到最大年龄晋升到老年区。
4、动态对象年龄判定。
	虚拟机并不是永远地要求对象的年龄必须达到最大年龄才能晋升老年代，如果在S区中相同年龄所有对象大小的总和大于S区的一半，年龄大于或等于该年龄的对象就可以直接进入老年代。 
5、空间分配担保。
	新生代中有大量的对象存活S区不够用了，当出现大量对象在MinorGC后仍然存活的情况（最极端的情况就是内存回收后新生代中所有对象都存活），就需要老年代进行分配担保，把S区无法容纳的对象直接进入老年代。
  
GC如何判断对象的存活？- 可达性分析算法。

GC如果进行回收的？- 复制回收算法；标记-清除算法；标记-整理算法。
```



##### 堆内存是怎么分配的？

```java
堆内存的划分：（新生代:老年代 = 1:2）
新生代（PSYoungGen）
	Eden区:两个S区 = Eden:From:To = 8:1:1
老年代（ParOldGen）

堆内存分配策略：
1、对象优先在Eden分配，如果说Eden内存空间不足，就会发生Minor GC
2、大对象直接进入老年代。大对象：需要大量连续内存空间的对象，比如很长的字符串或大型数组。
3、长期存活的对象将进入老年代。默认15岁，-XX:MaxTenuringThreshold调整
	对象头会记录对象的年龄，当对象在S区中被复制一次，年龄就会+1岁。当达到最大年龄晋升到老年区。
4、动态对象年龄判定。
	虚拟机并不是永远地要求对象的年龄必须达到最大年龄才能晋升老年代，如果在S区中相同年龄所有对象大小的总和大于S区的一半，年龄大于或等于该年龄的对象就可以直接进入老年代。 
5、空间分配担保。
	新生代中有大量的对象存活S区不够用了，当出现大量对象在MinorGC后仍然存活的情况（最极端的情况就是内存回收后新生代中所有对象都存活），就需要老年代进行分配担保，把S区无法容纳的对象直接进入老年代。

GC策略：
新生代区内存不够，会触发Minor GC。
老年代区内存不够，会触发Full GC。
```



##### 新生代中3个区（Eden区 + 两个S区）为什么内存比例是8:1:1 ？

```java
1、在新生代区中绝大部分对象的生命都是很短暂的，也就是说并不需要按照1:1的比例来划分内存空间；
2、经长期研究测算出当内存使用超过98%以上时内存就应被minor gc回收一次。但是实际应用中如果真到98%才GC可能就来不及了，所以保险起见当内存使用到达90%的时候就gc，留10%的内存放存活的对象。
3、这预留下来的这10%的空间称为S区（有两个s区 s1 和 s0），S区是用来存储新生代GC后存活下来的对象，而GC算法使用的是复制回收算法。需要1:1的内存空间。也就是需要占用总内存的20%，留下了80%给Eden区。
4、每次GC范围是Eden区+一个S区。（比例是，eden:s1:s0=8:1:1=8:1:1）这里的eden区（80%）和其中的一个S区（10%） 合起来共占据90%，GC就是清理的他们，始终保持着其中一个S区是空留的，保证GC的时候复制存活的对象有个存储的地方。

8:1:1的优点：提高内存的利用率，只浪费了10%的内存。

如果存活的对象超过10%的内存怎么办？
空间担保：空间担保是老年代区，如果S区放不下了，会把多出来的部分放到老年代区。
```



##### GC如何判断对象是否需要回收？+2

```java
可达性分析算法：通过GC Roots的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可达到，GC会回收它。
```



##### 什么对象可以作为GC Roots？

```java
1、虚拟机栈（栈帧中的本地变量表）中引用的对象。
2、方法区中类静态属性引用的对象。
3、方法区中常量引用的对象。
4、本地方法栈中JNI（即一般说的Native方法）引用的对象。

可达例子1：
static Obj a = new Obj(); //静态对象，是GC Roots
main(){
	Obj b = a;
	Obj c = b;
	Obj d = c;
	Obj e = a;
}

a -> b -> c -> d
  -> e
(b、c、d、e是可达的，方法执行完后，GC也不会回收它们)

可达例子2：
main(){
	Obj a = new Obj(); //GC Roots
	Obj b = a;
	Obj c = b;
}
(方法执行完之前，a、b、c都是可达的)

不可达例子：
Obj a = new Obj(); //不是GC Roots
main(){
	Obj b = a;
	Obj c = b;
}

a -> b -> c
(b、c是不可达的，方法执行完后，GC会回收它们)
```



##### GC如果进行回收的？

```java
垃圾回收算法：
  
复制回收算法：
将可用内存按容量划分为大小相等的两块区域，每次只使用其中的一块，当这一块的内存快用完了就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况。

优点：实现简单，运行高效，不会出现内存碎片。
缺点：内存缩小了一半利用率低，需要内存复制。
使用场景：新生代区使用。

------------------------------------------------------------------

标记-清除算法：
分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。
它的主要不足空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

优点：内存利用率百分百，无需内存复制。
缺点：会出现内存碎片。

------------------------------------------------------------------

标记-整理算法：
先标记出所有需要回收的对象，在标记完成后，后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

优点：内存利用率百分百，不会出现内存碎片。
缺点：需要内存复制。
  

Stop The World现象：GC事件发生过程中，会产生应用程序的停顿。停顿产生时整个应用程序线程都会被暂停。
1、可达性分析算法中枚举GC Roots会导致所有Java执行线程停顿。
2、完成GC后会切回原来的线程，由于切线程的过程也是耗时的，如果频繁GC频繁地切线程，也会造成卡顿。

GC收集器和我们GC调优的目标就是尽可能的减少STW的时间和次数。

单线程中的GC收集器：
Serial（新生代，复制回收算法）
SerialOld（老年代，标记整理算法）

多线程中的GC收集器：
ParNew（新生代，复制回收算法）- 搭配CMS垃圾回收器的首选 
ParallelScavenge（新生代，复制回收算法）
ParallelOld（老年代，标记整理算法）
CMS（老年代，标记清除算法）- 并行与并发收集器 - 互联网后端目前主流的垃圾回收器
G1（新生代 + 老年代）- 并行与并发收集器 - jdk1.7加入

除了G1是结合体之外，其他的垃圾回收器都是搭配使用的，都是新生代一个，老年代一个。

CMS
收集器是一种以获取最短回收停顿时间为目标的收集器。目前很大一部分的Java应用集中在互联网站或者B/S系统的服务端上，这类应用尤其重视服务的响应速度，希望系统停顿时间最短，以给用户带来较好的体验。CMS收集器就非常符合这类应用的需求。

CMS收集器是基于“标记—清除”算法实现的，它的运作过程相对于前面几种收集器来说更复杂一些，整个过程分为4个步骤，包括：
1、初始标记-短暂，仅仅只是标记一下GC Roots能直接关联到的对象，速度很快。 - 会暂停业务线程
2、并发标记-和用户的应用程序同时进行 - 并发标记，不会暂停业务线程
3、重新标记-短暂，为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。- 会暂停业务线程
4、并发清除 由于整个过程中耗时最长的并发标记和并发清除过程收集器线程都可以与用户线程一起工作。

浮动垃圾：由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生，这一部分垃圾出现在标记过程之后，CMS无法在当次收集中处理掉它们，只好留待下一次GC时再清理掉。这一部分垃圾就称为“浮动垃圾”。
```



#### 语法相关

1. final、finally、finalize()的区别。+3

2. 说说Java四种引用，以及用到的场景。+2

   ```JAVA
   四种引用：
   强引用
   
   软引用（SoftReference）：系统将要发生OOM之前，这些对象就会被回收。
   用途：例如为了让图片列表更快地显示，可以把它们直接加载到内存中，软引用它们。当系统内存不够即将OOM的时候，让系统释放掉这些图片的内存，保证程序的正常运行。大不了图片看不了而已，不至于闪退。
   
   弱引用（WeakReference）：只能生存到下一次垃圾回收之前，GC发生时不管内存够不够都会被回收。
   注意：软引用和弱引用，可以用在内存资源紧张的情况下以及创建不是很重要的数据缓存。当系统内存不足的时候，缓存中的内容是可以被释放的。
   实际运用（WeakHashMap、ThreadLocal）
   
   虚引用：幽灵引用，最弱，被垃圾回收的时候收到一个通知
   ```

   

3. 弱引用与强引用的区别 ，怎么判断一个弱引用被回收了 。

4. StringBuffer、StringBuilder的区别。+2

5. ==和equals和hashCode的区别。+2

6. String a="a"与String a = new String("a")的区别。+2

7. int、char、long各占多少字节数

8. int与integer的区别

9. 谈谈对java多态的理解。+2

10. 什么是内部类？内部类的作用

11. 泛型中extends和super的区别。

12. 说一下泛型原理，并举例说明。

13. Serializable 和Parcelable 的区别。+2

14. 父类的静态方法能否被子类重写

15. 静态属性和静态方法是否可以被继承？是否可以被重写？以及原因？

16. 静态内部类的设计意图

17. 成员内部类、静态内部类、局部内部类和匿名内部类的理解，以及项目中的应用

18. 抽象类和接口区别

19. 抽象类的意义

20. 抽象类与接口的应用场景

21. 抽象类是否可以没有方法和属性？

22. 接口的意义

23. Java中String的了解

24. String为什么要设计成不可变的？

25. Object类的equal和hashCode方法重写，为什么？

    

    

    

#### 集合相关

1. Arraylist和Linklist的区别。+3

2. HashMap扩容条件中链表转红黑树的条件是什么？

3. HashMap扩容条件为什么要2的指数次幂，如果输入17会是多少容量？（跟hashcode有关，输入17得出结果是32）

4. CurrentHashMap 读写锁是如何实现的。（无hash冲突用CAS插入，有则用synchronize加锁插入。当链表长度大于8且数据长度>=64的时候会用红黑树代替链表）

5. List,Set,Map的区别

6. List和Map的实现方式以及存储方式

7. ConcurrentHashMap的实现原理

8. ArrayMap和HashMap的对比

9. HashTable实现原理

10. TreeMap具体实现

11. HashMap和HashTable的区别

12. HashMap与HashSet的区别

13. HashSet与HashMap怎么判断集合元素重复？

14. 二叉树的深度优先遍历和广度优先遍历的具体实现

15. 堆和树的区别

16. 什么是深拷贝和浅拷贝

17. 如何防止 Java 源码被反编译

    

#### 其他

1. 动态代理传入的参数有哪些？非接口类能实现动态代理吗？ASM的原理是什么？
2. utf-8编码中的中文占几个字节；int型几个字节？
3. 静态代理和动态代理的区别，什么场景使用？



#### Map相关

##### 说说HashMap的hash方法的作用？（hash方法原理分析）

```java
hash方法作用是：均匀散列

hash()方法解析：1、Object#hashCode；2、取模算法（hashcode ^ (h >>> 16)）
static final int hash(Object key) {
	int h;
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

1、Object#hashCode：返回固定长度的摘要
hash算法：任意长度的输入 转换成 固定长度的摘要输出。这种转换是一种压缩映射，所有不同的输入可能会散列成相同的摘要输出。
java hash的实现方式：基于内存地址生成；利用位移生成随机数；随机数；自增。
java hashcode存储：对象的hashCode()未重写时，会返回一个由随机数算法生成的值。因为对象的hashCode不可变所以需要存到对象头中。当再次调用该方法时，会直接返回对象头中的hashcode。如果重写了hashCode()，对象头的code会失效并且也不会把code存对象头了。

为什么要用到hashCode方法？
一个对象想要插入一个集合中，首先得知道这个集合是否存在该对象，那怎么去集合中查找呢？
常规方法是调用equals方法来逐个比较，但是如果集合数据量庞大，逐个比较效率会很低。
集合存储基本上离不开数值下标，如果能够直接通过下标拿到对象效率就非常高的，所有需要用到唯一数值，便有了将对象转化为数值的hashCode方法。

2、取模算法（hashcode ^ (hashcode >>> 16)）（为了均匀散列表的下标，降低极端情况的出现从而导致链表拉长）
为什么要右移位？为什么右移位16？为什么要异或？
右移位，右移位16：取int类型的一半，将二进制数对半切开。（提高运算性能）
异或：降低极端情况的出现。
  
HashMap如何根据hash值找到数组中的对象，get方法中有这样一段代码：first = tab[(n - 1) & hash]（tab为数组，n为数组长度）
假设：数组长度为16（n-1也就是15），并且不做取模算法，直接使用对象的hashCode来做下标。
对象A hashCode（1000010001110001000001111000000）
对象B hashCode（0111011100111000101000010100000）
15 & 对象A的hashCode  -->  0
15 & 对象B的hashCode  -->  0
为啥？？？？？因为A、B hashCode后边一大堆000000。这样的散列结果太让人失望了。很明显不是一个好的散列算法。
但是如果我们将hashCode值右移16位，也就是取int类型的一半，刚好将该二进制数对半切开。并且使用位异或这样的话，就能避免我们上面的情况的发生。
总的来说，使用位移16位和异或就是防止这种极端情况。
但是一些极端情况下还是有问题，比如：10000000000000000000000000和 1000000000100000000000000这两个数，如果数组长度是16，那么即使右移16位，在异或，hash 值还是会重复。
但是为了性能，对这种极端情况，JDK选择了性能。毕竟这是少数情况，为了这种情况去增加 hash 时间，性价比不高。
```



##### 为什么get()、put()要用&运算来算下标？公式：(n - 1) & hash

```java
n为数组长度，下标肯定是在n范围内的。怎么把能够把hash计算变成 0 - n-1 的范围之内呢？
除法、求余、取模。这些方法速度都不快，最快的还是位运算。
a % b == (b-1) & a ,当b是底数是2的真数时，等式成立（N = 2的n次幂，2是底数，n为对数，N为真数）。

例如：
4 % 4 = 0  --->  (4-1) & 4 = 0
5 % 4 = 1  --->  (4-1) & 5 = 1

非2的n次幂
4 % 3 = 1  --->  (3-1) & 4 = 0
5 % 3 = 2  --->  (3-1) & 5 = 0
6 % 3 = 0  --->  (3-1) & 4 = 2
7 % 3 = 1  --->  (3-1) & 5 = 3
8 % 3 = 2  --->  (3-1) & 5 = 0

因为2的n次幂 - 1，二进制全是1。例如 3 -> 11    7 -> 111
用全是1的二进制数，正好可以做掩码。 在&运算的时候，结果取决于hash值。由于hash值是均匀散列的，所以结果也是均匀散列。
```



##### 为什么HashMap容器大小最好设置成2的次幂？（为什么HashMap默认的容器大小是16）

```java
hash算法的目的是为了让hash值均匀的分布在数组中。如果不使用2的幂次方作为数组的长度，就会失去了hash均匀分布作用。造成不同的key值全部进入到相同的数组下标中形成链表，性能急剧下降。
我们一定要保证 &运算 中的二进制位全为1，才能最大限度的利用hash值。

HashMap容量大小设置多少最好？ 知道数据量的情况，最好给容器合理的大小。能避免动态扩容带来的性能损耗（动态扩容：除了搬运数据耗时之外，还可能导致成倍的内存浪费）。
容量大小最好是2的次幂 同时 还需要根据负载系数来权衡设置。
例如：需要存2个数据，HashMap默认的负载系数是0.75
2 / 0.75 ≈ 2.67 那么容量最好设置为4，虽然会造成内存浪费，但是可以避免动态扩容。
如果你预计大概会插入 12 条数据的话，那么初始容量为16简直是完美，一点不浪费，而且也不会扩容。
```



##### HashMap的链表、红黑树是如何转换的？

```java
put操作的时候，发生hash冲突后并且数组下标有元素的情况下。就会判断元素类型是 TreeNode 还是 Node。TreeNode代表为红黑树，Node代表链表。

链表的遍历插入逻辑中，如果链表长度大于8了。就会转换为红黑树（走treeifyBin方法转换）。
//TREEIFY_THRESHOLD = 8
if (binCount >= TREEIFY_THRESHOLD - 1) treeifyBin(tab, hash);

并不是走treeifyBin方法就一定会变成红黑树的，还会先判断数组长度有没有大于64，如果没有的话。优先动态扩容，重新离散元素。
为什么会优先扩容，而不是优先转红黑树呢。因为红黑树虽然查询效率高了，但是插入效率不高。
//MIN_TREEIFY_CAPACITY = 64
final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }

为什么不直接使用红黑树呢？
红黑树为了维护平衡，需要进行左旋，右旋操作。 而单链表不需要。
如果元素小于8个，查询成本高，新增成本低。
如果元素大于8个，查询成本低，新增成本高。
由于红黑树也会存在高成本的地方，所有Hashmap在转换红黑树前，会先做动态扩容判断，就是为了通过牺牲空间来换时间的。

红黑树（一个自平衡的二叉查找树）
二叉查找树：
性质1 左子树上所有结点均小于或等于它的根结点；
性质2 右子树上所有结点均大于或等于它的根结点；
性质3 左、右子树也分别为二叉排序树。

“查找” -> 二分查找思想 -> 查出结果所需的最大次数就是二叉树的高度 -> 时间复杂度O(logn)
但是有极端的情况，会让二叉查找树退化为链表。
依次插入根节点为9如下五个节点：7,6,5,4,3。依照二叉查找树的特性7,6,5,4,3一个比一个小，那么就会成一条直线，也就是成为了线性的查询，时间复杂度变成了O(n)。红黑树登场！！！

红黑树：
性质1 节点是红色或黑色。
性质2 根节点是黑色。
性质3 每个叶节点（NIL节点，空节点）是黑色的。
性质4 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点)
性质5 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点。

红黑树通过“变色”和“旋转”来维护红黑树的规则，变色就是让黑的变成红的，红的变成黑的，旋转又分为“左旋转”和“右旋转”。
```





##### HashMap为什么是线程不安全？会出现什么问题？

```java
1.7 扩容操作 -> 死循环、数据丢失
死循环：转移元素使用的是头插法。链表的顺序会翻转，这是形成死循环的关键点。
头插法：
[1]-a-b-null
-------------
[1]-null
[2]-b-a-null


线程1 
[1]-b-null（取出了a，但是还没来得及丢到[2]，就挂起了）
[2]-null
线程2 
[1]-null
[2]-b-a-null（线程2又扩容了，并且走完扩容逻辑了，然后挂起）
线程1
[1]-null
[2]-a-b-a（把之前取出来的a，b.next = a。 a-b-a形成了死循环）


1.8 put操作 -> 数据覆盖
扩容操作，转移元素使用的是尾插法。不会出现1.7的死循环。

线程A和线程B同时进行put操作，刚好这两条不同的数据hash值算出来的数组下标一样。并且该位置数据为null，所以这线程A、B都会往这下标插入数据。
假设一种情况，线程A进入后还未进行数据插入时挂起，而线程B正常执行正常插入数据，然后线程A恢复继续插入。问题出现：线程A会覆盖线程B的数据。

尾插法：
[1]-a-b-null
-------------
[1]-null
[2]-a-b-null
```



##### HashMap能不能用二维数组实现？

```java
能。二维数组就是相当于数组+数组。而Hashmap实现是数组+链表。这里可以把链表替换成数组。
数组访问速度是O(1)，链表是O(N)。二维数组在查询速度上完胜。

但是会有以下的问题：
1、浪费空间。大多数情况下hash表并不存在大量冲突，二维数组会浪费内存。
2、动态扩容麻烦。二维需要考虑到两个数组的扩容问题。
3、二次散列。第二维的那个数组要用到O(1)的速度就必须要给value值进行散列。散列又会出现冲突问题，valueA散列出2，valueB也散列出2，导致冲突。解决方法就只能用开放寻地发。
   如果不想二次散列，就只能跟老老实实用O(N)速度，一个个遍历了。
```



##### 说说LinkedHashMap（HashMap + 双向链表）特点：有序的

```java
在HashMap的基础上（继承自HashMap），多维护了一个双向链表，用来保证元素的有序性。

关键属性：
Entry<K,V> header; //头结点（双向链表的入口）
boolean accessOrder; //false插入顺序，true访问顺序，也就是访问次数.

元素类继承自HashMap.Entry<K,V>
并且扩展了before、after
	Entry<K, V> before;
	NEtry<K, V> after;
before、after是用于维护Entry插入的先后顺序的。正是before、after和head的存在，才形成了双向链表

1、重写了init方法，为header进行初始化。（在HashMap构造函数中调用了这个init方法）
header中hash值为-1，其他都为null，before、after指向自己，header不在数组table中的。head的目的就是为了记录第一个插入的元素。

2、并没有重写HashMap的put方法，而是只重写了put方法逻辑中调用的子方法addEntry()和createEntry()
数据插入逻辑也是使用HashMap的逻辑。
addEntry()和createEntry()目的是为了将插入后元素的before、after，与header绑定在一起，形成双链表。

3、重写了HashMap的get方法
通过HashMap的get方法拿到数据之后，会判断accessOrder标志位。
如果accessOrder为true，将该元素转移到双向链表的尾部（实现访问顺序）
public V getOrDefault(Object key, V defaultValue) {
       Node<K,V> e;
       if ((e = getNode(hash(key), key)) == null)
           return defaultValue;
       if (accessOrder)
           afterNodeAccess(e);
       return e.value;
   }

4、迭代器LinkedHashIterator，遍历的是双向链表。拿到head，从head开始遍历
```



##### 说说ConcurrentHashMap的原理

```java
JDK1.7 Segment（ReentrantLock）+ HashEntry
ConcurrentHashMap维护着一个Segment对象数组。
一个Segment就相当于一个HashMap。Segment里包含一个HashEntry数组，每个数组元素能链成一条链表。

采取锁分段技术，每一个Segment就好比一个自治区，读写操作高度自治，Segment之间互不影响。
case1 不同Segment的并发写入【可以并发执行】
case2 同一Segment的一写一读【可以并发执行】
case3 同一Segment的并发写入【会上锁，保证一个线程操作，其他线程等待】
由此可见，当中每个Segment各自持有一把锁（Segment继承自ReentrantLock）。在保证线程安全的同时降低了锁的粒度，让并发操作效率更高。

Get方法：
为输入的Key做Hash运算，得到hash值。
通过hash值，定位到对应的Segment对象。
再次通过hash值，定位到 Segment 当中数组的具体位置。

Put方法：
为输入的Key做Hash运算，得到hash值。
通过hash值，定位到对应的Segment对象。
获取可重入锁
再次通过hash值，定位到Segment当中数组的具体位置。
插入或覆盖HashEntry对象。
释放锁

读写时均需要二次散列。首先定位到Segment，之后定位到Segment内的具体数组下标。

Size方法：每个Segment都各自加锁，那么在调用size()的时候，怎么解决一致性的？
遍历所有的Segment。
把Segment的元素数量累加起来。
把Segment的修改次数累加起来。
判断所有Segment的总修改次数是否大于上一次的总修改次数。如果大于，说明统计过程中有修改重新统计尝试次数+1；如果不是，说明没有修改统计结束。
如果尝试次数超过阈值，则对每一个Segment加锁，再重新统计。
再次判断所有Segment的总修改次数是否大于上一次的总修改次数。由于已经加锁，次数一定和上次相等。
释放锁，统计结束。

乐观锁 + 悲观锁：乐观地认为Size过程中不会有修改。当尝试多次次数，才无奈转为悲观，锁住所有Segment保证强一致性。

================================================

JDK1.8 Synchronized + CAS + Node（锁的粒度更小）

//sizeCtl变量：chm内部数组的状态
//0：sizeCtl为0，代表数组未初始化
//正数：该值代表当前数组的阈值（跟hashmap的一样，capacity*loadFactor）
//-1：代表数组正在初始化
//负数且不是-1：代表数组正在扩容，该数为-(n+1)，表示当前有n个线程在共同完成扩容操作
private transient volatile int sizeCtl;

//baseCount +  CounterCell[]是用来统计size的
private transient volatile CounterCell[] counterCells;
private transient volatile long baseCount;

//存储K,V数据
transient volatile Node<K,V>[] table;

Get方法：
1、计算hash值定位到table下标位置，如果是首节点符合就返回
2、如果遇到扩容的时候，会调用标志正在扩容节点ForwardingNode的find方法，查找该节点，匹配就返回
3、以上都不符合的话，就往下遍历节点，匹配就返回，否则最后就返回null


Put方法：初始化操作并不是在构造函数实现的而是在put操作中实现。
进行自旋（死循环）：1-5步骤
1、如果table为就先调用initTable()来进行初始化（懒初始化）
	如果其他线程正在初始化（sizeCtl<0），那么调用Thread.yield()挂起当前线程，等其他线程执行完后再继续工作。
2、如果没有hash冲突就直接CAS插入（table被volatile修饰，可以保证可见性）
3、如果还在进行扩容操作就先进行扩容
	走helpTransfer()方法，没有加锁或者挂起线程操作。利用多个线程一起帮助进行扩容提高扩容效率，而不是只有检查到要扩容的线程进行扩容。
4、如果存在hash冲突，就加锁这个元素（synchronized）来保证线程安全。
	链表形式就直接遍历到尾端插入。
	红黑树就按照红黑树结构插入。
5、最后一个如果该链表的数量大于阈值8，就要先转换成黑红树的结构，break再一次进入循环;

6、如果添加成功就调用addCount()方法统计size，并且检查是否需要扩容。

Size方法：baseCount +  CounterCell[]里所有value值
在扩容和addCount()方法就已经把baseCount、CounterCell[]处理好了，Put方法里就有addCount()。


addCount()方法解析：更新baseCount、CounterCell[]
多个线程进行put，要进行size++操作。那么是不是用一个原子类就完事了？ 不，这样太慢了。需要一个线程累加完才到下一个。
baseCount + CounterCell[]，先尝试对baseCount进行CAS，成功就更新了baseCount值。
如果累加baseCount失败，那么直接在CounterCell[]，拿到当前线程的hashcode运算出下标，给下标做累加。
这样就能实现多线程累加了。

transfer()方法解析：利用多个线程一起进行扩容
ForwardingNode：一个特殊的Node节点，hash值为-1，其中存储nextTable的引用。
只有table发生扩容的时候，才会发挥作用，作为一个占位符放在table中表示当前节点为null或则已经被移动。

节点从table移动到nextTable，大体思想是遍历、复制的过程：

首先根据运算得到需要遍历的次数i，然后利用tabAt方法获得i位置的元素f，初始化一个forwardNode实例fwd。
如果f == null，则在table中的i位置放入fwd，这个过程是采用Unsafe.compareAndSwapObjectf方法实现的，很巧妙的实现了节点的并发移动。
如果f是链表的头节点，就构造一个反序链表，把他们分别放在nextTable的i和i+n的位置上，移动完成，采用Unsafe.putObjectVolatile方法给table原位置赋值fwd。
如果f是TreeBin节点，也做一个反序处理，并判断是否需要untreeify，把处理的结果分别放在nextTable的i和i+n的位置上，移动完成，同样采用Unsafe.putObjectVolatile方法给table原位置赋值fwd。
```





##### 说说Android的SparseArray

```java
SparseArray系列（SparseArray，SparseBooleanArray，SparseIntArray，SparseLongArray，LongSparseArray）
HashMap -> key - 泛型，value - 泛型
SparseArray -> key - int，value - Object
SparseArray在使用上受到了很大的约束嘛，这样约束的意义何在呢？

SparseArray中的优秀设计：
延迟删除机制（删除设置“标志位”，来延迟删除，实现数据位的复用）；
二分查找的返回值处理；
在空间不足时，利用gc函数一次性压缩空间，提高效率。
  
SparseArray应用场景：
1、数据量不大，最好在千级以内（数据量大的情况下性能并不明显，将降低至少50%）
2、key必须为int类型，这中情况下的HashMap可以用SparseArray代替

SparseArray原理分析：
属性：
private static final Object DELETED = new Object(); //删除标志位
private boolean mGarbage = false; //是否垃圾回收
private int[] mKeys; //升序数组
private Object[] mValues;

添加：
	public void append(int key, E value) {
        if (mSize != 0 && key <= mKeys[mSize - 1]) {
        	//当mSize不为0并且不大于mKeys数组中的最大值时,因为mKeys是一个升序数组，最大值即为mKeys[mSize-1]
	        //直接执行put方法，否则继续向下执行
            put(key, value);
            return;
        }

		//当垃圾回收标志mGarbage为true并且当前元素已经占满整个数组，执行gc进行空间压缩
        if (mGarbage && mSize >= mKeys.length) {
            gc();
        }

        //当数组为空，或者key值大于当前mKeys数组最大值的时候，在数组最后一个位置插入元素。
        mKeys = GrowingArrayUtils.append(mKeys, mSize, key);
        mValues = GrowingArrayUtils.append(mValues, mSize, value);
        mSize++;
    }

    public void put(int key, E value) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key); //二分查找
        if (i >= 0) { //查找到
            mValues[i] = value;
        } else { //没有查找到
            i = ~i; //取反，拿到要插入的下标。
            if (i < mSize && mValues[i] == DELETED) { //元素要添加的位置正好==DELETED，直接覆盖它的值即可。
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }
			//垃圾回收，但是空间压缩后，mValues数组和mKeys数组元素有变化，需要重新计算插入的位置
            if (mGarbage && mSize >= mKeys.length) {
                gc();

                //重新计算插入的位置
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }
			//在i位置插入元素
            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            mSize++;
        }
    }

    //ContainerHelpers#binarySearch
    static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            final int mid = (lo + hi) >>> 1;
            final int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid; // 找到了
            }
        }
        //没找到，直接取反然后返回。
        //取反就变成负数了，外部只需要判断大于0就能够知道找没找到了
        //然后外部还可以再取反拿回没命中的这个下标位置，这个下标位置就是将要插入数据的位置。
        return ~lo;
    }

    例子：
	假设我们有个数组 3 4 6 7 8。用二分查找来查找元素5
	初始：lo=0 hi=4
	第一次循环：mid=(lo+hi)/2=2 2位置对应6 6>5 查找失败，下一轮循环 lo=0 hi=1
	第二次循环：mid=(lo+hi)/2=0 0位置对应3 3<5 查找失败，下一轮循环 lo=2 hi=1
	lo>hi 循环终止
	最终 lo=2 即5需要插入的下标位置


    //GrowingArrayUtils#insert 把目标下标后面的元素后移，再插入元素。（如果容量不够会进行动态库容）
    public static int[] insert(int[] array, int currentSize, int index, int element) {
        if (currentSize + 1 <= array.length) {//如果数组长度能够容下直接在原数组上插入
        	//调用了Java 的native方法，把array 从index开始的数复制到index+1上，
        	//复制长度是currentSize - index
            System.arraycopy(array, index, array, index + 1, currentSize - index);
            //空出来的那个位置直接放入我们要存入的值，
            //也不是空出来，其实index上还是有数的，
            // 比如：{2,3,4,5,0,0}从index=1开始复制，复制长度为5，复制后的结果就是{2,3,3,4,5,0}了
            array[index] = element;
            return array;
        }

		//这就是扩容了，新建了一个数组，长度*2
        int[] newArray = new int[growSize(currentSize)]; 
        
        //新旧数组拷贝，先拷贝最佳位置之前的到新数组
        System.arraycopy(array, 0, newArray, 0, index);
        
        newArray[index] = element;//直接在新数组上赋值
        //然后拷贝旧数组最佳位置index起的所有数到新数组里面，
        //只是做了分段拷贝而已
        System.arraycopy(array, index, newArray, index + 1, array.length - index);
        
        return newArray;
    }


查找：二分查找，找到了返回value给你。没找到返回你自己传的notFoundValue。
	public E get(int key, E valueIfKeyNotFound) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i < 0 || mValues[i] == DELETED) {
            return valueIfKeyNotFound;
        } else {
            return (E) mValues[i];
        }
    }

删除：没有实际删除，size值也不改？？？？？
	public void delete(int key) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            if (mValues[i] != DELETED) { //直接替换成DELETED对象
                mValues[i] = DELETED;
                mGarbage = true; //开启回收标志位
            }
        }
    }

获取大小：先触发gc一次，再返回size值，没毛病
	public int size() {
        if (mGarbage) {
            gc();
        }

        return mSize;
    }


gc方法：快慢指针思想，将DELETED元素排挤出数组。
    private void gc() {
        int n = mSize;//压缩前数组的容量
        int o = 0;//压缩后数组的容量，初始为0
        int[] keys = mKeys;//保存新的key值的数组
        Object[] values = mValues;//保存新的value值的数组

        for (int i = 0; i < n; i++) {
            Object val = values[i];
            if (val != DELETED) {//如果该value值不为DELETED，也就是没有被打上“删除”的标签
                if (i != o) {//如果前面已经有元素打上“删除”的标签，那么 i 才会不等于 o
	                //将 i 位置的元素向前移动到 o 处,这样做最终会让所有的非DELETED元素连续紧挨在数组前面
                    keys[o] = keys[i];
                    values[o] = val;
                    values[i] = null;//释放空间
                }
                o++;
            }
        }
        mGarbage = false; //恢复垃圾回收标志位
        mSize = o; //更新size值，回收之后数组的大小
    }
```



##### 说说Android的ArrayMap<K, V>

```java
ArrayMap内部跟SparseArray一样也是使用两个数组进行数据存储。一个数组记录key的hash值，另外一个数组记录Value值。
它和SparseArray一样，也会对key使用二分法进行从小到大排序，在添加、删除、查找数据的时候都是先使用二分查找法得到相应的index，然后通过index来进行添加、查找、删除等操作，所以，应用场景和SparseArray的一样。
  
ArrayMap应用场景：
1、数据量不大，最好在千级以内（数据量大的情况下性能并不明显，将降低至少50%）
```



