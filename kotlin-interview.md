## Kotlin

#### 协程

1. 说说kotlin的协程。+3
2. 概括一下kotlin协程的上下文。
3. 协程是如何挂起的。



#### 语法相关

##### 高阶函数是什么？

##### ==与===的区别。

==：值的比较（等同于java的equal）

===：比较的是两个引用指向是否为同一内存空间（等同于java的==）

```java
//一个神奇的问题，为什么b中的比较会出现false，而a，c，d中的比较又正常
val a: Int = 100
val a1: Int? = a
val a2: Int? = a
Log.e("llk", "a=" + (a1 === a2)) //输出：a=true
val b: Int = 10000
val b1: Int? = b
val b2: Int? = b
Log.e("llk", "b=" + (b1 === b2)) //输出：b=false

val c: Int = 100
val c1: Int = a
val c2: Int = a
Log.e("llk", "c=" + (c1 === c2)) //输出：c=true
val d: Int = 10000
val d1: Int = b
val d2: Int = b
Log.e("llk", "d=" + (d1 === d2)) //输出：d=true
  
//c，d都是非空Number类型，是基本数据类型了。
//Kt中的非空Number类型对应到 JVM 平台是基本类型：int , double 等等；
//Kt中的可空Number类型对应到 JVM 平台是封装类型：Integer , Double等等；

//a，b都是可空Number类型，是封装类
//封装类的赋值是使用Integer#valueOf
//封装类Integer，有-128~127的数字缓存，在这个范围内是直接拿数字缓存的对象引用
//如果超出数字缓存范围，那么就是创建新的对象了。
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```





##### lateinit与lazy的区别。



#### 其他

1. kotlin与java有什么区别。+3
2. kotlin有什么优缺点？+2
3. kotlin的lambda与java的lambda有什么区别。
4. data class 与 Gson发生的解析异常。

