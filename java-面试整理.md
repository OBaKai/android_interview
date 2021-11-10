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

9. 开启线程的三种方式？

10. 线程和进程的区别？+2

11. run()和start()方法区别

12. 两个进程同时要求写或者读，能不能实现？如何防止进程的同步？

    

    

#### GC

1. gc泄漏在什么情况下出现？怎么解决？
2. 说说对垃圾回收器的理解。+3
3. 哪些情况下的对象会被垃圾回收机制处理掉？
4. 



#### 语法相关

1. final、finally、finalize()的区别。+3

2. 说说Java四种引用，以及用到的场景。+2

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

    

18. 

    

#### 其他

1. 动态代理传入的参数有哪些？非接口类能实现动态代理吗？ASM的原理是什么？
2. utf-8编码中的中文占几个字节；int型几个字节？
3. 静态代理和动态代理的区别，什么场景使用？