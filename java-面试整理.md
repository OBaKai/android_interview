## Java

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

   

#### GC

1. gc泄漏在什么情况下出现？怎么解决？
2. 说说对垃圾回收器的理解。+3



#### 语法相关

1. final、finally、finalize()的区别。+2

2. 说说Java四种引用，以及用到的场景。+2

3. 弱引用与强引用的区别 ，怎么判断一个弱引用被回收了 。

4. StringBuffer、StringBuilder的区别。

5. equals与==的区别。

6. String a="a"与String a = new String("a")的区别。+2

   

#### 集合相关

1. Arraylist和Linklist的区别。+3

2. HashMap扩容条件中链表转红黑树的条件是什么？

3. HashMap扩容条件为什么要2的指数次幂，如果输入17会是多少容量？（跟hashcode有关，输入17得出结果是32）

4. CurrentHashMap 读写锁是如何实现的。（无hash冲突用CAS插入，有则用synchronize加锁插入。当链表长度大于8且数据长度>=64的时候会用红黑树代替链表）

   

#### 其他

1. 动态代理传入的参数有哪些？非接口类能实现动态代理吗？ASM的原理是什么？