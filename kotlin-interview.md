## 协程

```java
协程的使用 ！！！

协程作用域：
1、runBlocking：顶层函数。会阻塞当前线程来等待，不适合用于业务开发比较适合写测试用例
2、GlobalScope：全局作用域api。生命周期是生命周期是进程级别的，可以一直运行到app停止。
    子协程运行在自己的调度器上不会继承上下文与父协程没有联系，因此所有开启的协程都需要分别手动来管理
3、CoroutineScope：上下文作用域api。intermal修饰未对外暴露，根据指定的上下文创建协程作用域。
    子类：
    MainScope：通常在Activity中使用。在onDestory要记得手动销毁掉。
    ViewModelScope：只能在ViewModel中使用。绑定viewModel的生命周期
    LifecycleScope：只能在Activity、Fragment中使用。并且会和Activity、Fragment生命周期进行绑定。

协程api：
launch：创建协程。返回Job（可通过Job对协程进行一些操作）
async：创建带返回值的协程。返回Deferred（继承自Job，Deferred#await()可等待数据返回）
withContext：不创建新的协程，指定协程上运行代码块。（主要用于切换到到指定的线程，并在闭包内的执行结束后自动切回去继续执行）

========================================================================================

协程是什么 ！！！


协程是什么？安卓平台上协程是Kotlin对线程封装的一套api（一个线程框架）
协程好在哪？好在方便。
最方便在哪？它能够在同一个代码块里边进行多次的线程切换

特点：‘非阻塞式挂起’ 就是用看起来同步的方式写出异步代码
这个特点消除了并发任务之间协作的难度，让我们可以轻松的写出复杂的并发代码。

‘挂起’的本质是什么？挂起协程。
换句话说，就是让这个协程从正在执行它的线程上脱离。

例如：
suspend fun doSomething() : String{
    return withContext(Dispatchers.IO){
        ... 
    }
}

//主线程逻辑：
coroutineScope.launch(Dispatchers.Main) {
   val result = doSomething() 
   tv.setText(result)
}
xxx()
...


分析：
主线程方面：
① launch(Dispatchers.Main) 在主线程开启一个协程
② 执行到 doSomething() 发现是 suspend 函数，挂起协程
③ 跳出闭包，继续执行下面的 xxx() ...

协程方面：
① 执行 doSomething() 函数
② withContext(Dispatchers.IO) 根据调度器指示，启动一个io线程执行闭包里边的耗时逻辑，执行耗时逻辑完成后返回
③ doSomething() 返回后，自动帮我们切回来（最爽的！！！ io -> main）
④ 执行 tv.setText(result) 这个ui操作


========================================================================================

挂起 与 suspend关键字 ！！！

suspend关键字：suspend有暂停的意思。我们在协程中应该理解为：当线程执行到协程的 suspend 函数的时候，暂时不继续执行协程代码了。
然后兵分两路，一方面是线程（不执行协程的代码了，继续往下执行），另一方面是协程（根据调度器指示启动对应的线程执行协程代码）

suspend的使用：关键字只能声明在函数上，并且suspend函数只能在协程里或者另一个suspend函数里被调用。
自己写一个挂起函数，仅仅只加上 suspend 关键字是不行的，还需要函数内部直接或间接地调用到 Kotlin 协程框架自带的 suspend 函数才行。

为什么suspend关键字并没有实际去操作挂起，但Kotlin却把它提供出来？ suspend起到一个提醒的作用。
函数的创建者对函数的使用者的提醒：我是一个耗时函数，我被我的创建者用挂起的方式放在后台运行，所以请在协程里调用我。

因为它本来就不是用来操作挂起的。挂起的操作 —— 也就是切线程，依赖的是挂起函数里面的实际代码，而不是这个关键字。

========================================================================================

‘非阻塞式挂起’ 到底是什么 ！！！

正常的代码：
val result = doSomething() //如果doSomething是耗时的函数，那么这里会卡当前线程
tv.setText(result) //等待上一行代码的执行
xxx() //等待上一行代码的执行

使用了协程后：‘非阻塞式挂起’
coroutineScope.launch(Dispatchers.Main) {
   val result = doSomething() //这里先切走，丢到一个现场里边处理，处理完后切回来
   tv.setText(result) //等待上一行代码的执行
} //发现doSomething是suspend函数，那么我直接跳出来了，继续往下执行，不卡当前线程
xxx()


‘非阻塞式挂起’ 只是个写法。
使用协程能够实现上下两行连续代码悄悄地把线程切走再切回来，不会卡当前线程。
不用协程的话，上下两行连续代码只能是单线程的，那就会卡当前线程。

```



## 语法相关

###  lateinit & lazy 区别

```java
val name: String = "test" 
//反编译后
@NotNull private String name = "test";


lateinit var name: String
//反编译后
public String name; //1、加了lateinit后，没有了@NotNull注解

//2、然后在所有使用 name属性的地方都会有如下判断：
String var10000 = this.name;
if (var10000 == null) {
		Intrinsics.throwUninitializedPropertyAccessException("name");
}

lateinit：非空类型可以使用 lateinit 关键字达到延迟初始化
原理：在使用该属性前，都会增加空判断，如果为空就抛异常。
  
class InitTest{
  val name by lazy { "test" }
}
//反编译后
public final class InitTest {
    //属性变成了一个 Lazy对象
    @NotNull private final Lazy name$delegate;
  
    @NotNull public final String getName() {
       Lazy var1 = this.name$delegate;
       return (String)var1.getValue(); //第一次getValue的时候，才对name进行赋值
    }
 
    public InitTest() {
       //构造函数中，Lazy对象被赋值了
       this.name$delegate = LazyKt.lazy((Function0)null.INSTANCE);
    }
 }

// LazyKt#lazy
//由于lazy模式是LazyThreadSafetyMode.SYNCHRONIZED模式，所以实现类为SynchronizedLazyImpl
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)
//逻辑类似Java双重检查单例，在getValue()函数的时候，进行赋值。而且是保证线程安全的
private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
     private var initializer: (() -> T)? = initializer
     @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    
     private val lock = lock ?: this

     override val value: T
         get() {
             val _v1 = _value
             if (_v1 !== UNINITIALIZED_VALUE) {
                 @Suppress("UNCHECKED_CAST")
                 return _v1 as T
             }

             return synchronized(lock) {
                 val _v2 = _value
                 if (_v2 !== UNINITIALIZED_VALUE) {
                     @Suppress("UNCHECKED_CAST") (_v2 as T)
                 } else {
                     val typedValue = initializer!!()
                     _value = typedValue
                     initializer = null
                     typedValue
                 }
             }
         }

     override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

     override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."
       
     private fun writeReplace(): Any = InitializedLazyImpl(value)
 }
lazy：懒加载，在第一次使用的时候才进行赋值。而且修饰的只是能是 val 常量
原理：将目标属性，包装成为一个Lazy对象，在第一次使用该属性的时候，才执行 Lazy#getValue方法进行属性赋值。而且根据不用lazy mode会有不同的赋值方式，比如默认的线程安全mode，通过synchronized + volatile来保证赋值的可靠性。
```



### kotlin中with、run、apply、let函数的区别？一般用于什么场景？

```kotlin
//let：记住it，最后一行为返回值
//一个参数，lambda
//对象可判空。需it访问对象的方法或者属性。最后一行为返回值
val result = obj?.let {
   it.xxx()
}

//also：记住it，返回值为对象本身
//一个参数，lambda
//对象可判空。需it访问对象的方法或者属性。返回值是对象本身。
val result = obj?.also{
	it.xxx()
}

//with：记住2个参数
//2个参数，第一个为处理的对象，第二个为lambda
//对象无法判空。可直接访问对象的方法或属性，或用this调用。最后一行为返回值。
val result = with(obj) {
	xxx()
}

//run：记住let+with
//一个参数，lambda
//对象可判空。可直接访问对象的方法或属性，或用this调用。最后一行为返回值。
val result = obj?.run{
  xxx()
}

//apply：记住几乎与run一模一样，唯一区别：返回值
//一个参数，lambda
//对象可判空。可直接访问对象的方法或属性，或用this调用。返回值是对象本身。
val result = obj?.apply{
	xxx()
}

使用场景：
 let：适用于处理不为null的操作场景
 with：适用于调用同一个类的多个方法时，可以省去类名重复，直接调用类的方法即可
 run：适用于let,with函数任何场景
 apply：适用于初始化对象或更改对象属性
 also：适用于不改变对象，不过想增加点记录或者打印信息
```





### 说说高阶函数？ 

#### 什么是高阶函数 - 传参或返回值是函数对象的函数

```java
fun add(num1: Int, num2: Int): Int {
    return num1 + num2
}

函数对象：
这个add的函数，它的函数类型可抽象为 (Int, Int) -> Int
  
//声明a变量，变量类型为 (Int, Int) -> Int
//将add函数赋值给a变量，a变量就是函数对象
val a : (Int, Int) -> Int = ::add //::add这种写法是一种函数引用方式的写法
//或者如下赋值
val a: (Int, Int) -> Int = {num1: Int, num2: Int -> num1 + num2}


高阶函数：
fun higherFunction(func: (Int, Int) -> Int) {
}

fun higherFunction(): (Int, Int) -> Int {
}

```

#### 高阶函数的原理是什么？- 匿名内部类

```java
高阶函数转成Java的样子是什么样子的呢？
//高阶函数例子：
fun main() {
    var i = 0
    foo {
        i++
    }
}

fun foo(block: () -> Unit) {
    block()
}

//转java后：主要代码，省略了一些没用的代码
//Function0是一个接口，可以看到高阶函数foo的函数类型参数，变成了Function0，而main()函数当中的高阶函数调用，也变成了“匿名内部类”的调用方式。
//所以高阶函数最终还是以匿名内部类的形式在运行
public final class HigherFunctionKt {
   public static final void main() {
      foo((Function0)(new Function0() {
         public Object invoke() {
            this.invoke();
            return Unit.INSTANCE;
         }

         public final void invoke() {
            int var10001 = i.element++;
            int var1 = i.element;
         }
      }));
   }

   public static final void foo(@NotNull Function0 block) {
      block.invoke();
   }
}
```



#### 难道 高阶函数只是为了简化“匿名内部类”的写法吗？- inline（内联函数）

```java
高阶函数不只是简化“匿名内部类”那么简单，高阶函数的性能是远远高于匿名内部类。
只需要我们在函数的前面加上一个 inline 关键字就可以了。
  
我们可以利用 JMH 来进行测试，结果表明，是否使用 inline，它们之间的效率几乎相差 30 倍。

inline原理：Kotlin编译器会将内联函数中的代码在编译的时候自动替换到调用它的地方，这样也就不存在运行时的开销了。
```



### 顶级函数是什么？

```java
为了更好地重复性非常高的代码。
 
//声明顶级函数
package hello.aa.cc.dd.KotlinLearn.chapter_1.hello

fun sayMessage(message:String){
    println("hello ${message}")
}

//使用顶级函数
import hello.aa.cc.dd.KotlinLearn.chapter_1.hello.sayMessage
class A{
 fun a() {
    sayMessage("hello world");
 } 
}
```



### 扩展函数是什么？

```java
扩展函数：允许我们在不改变已有类的情况下，为类添加新的函数，该函数为静态函数
  
使用：
class DogKt{
    fun run() = "狗会跑"
    fun cry() = "狗会汪汪"
}
//扩展函数
fun DogKt.order() = "扩展功能-》狗听从指令"

fun main(args: Array<String>) {
    var ex = DogKt()
    println(ex.run())
    println(ex.cry())
    println(ex.order)
}
 
扩展函数的解析为静态：//执行 javap -c DogKt.class
发现在 DogKt类里边并没有找到 order 函数。
扩展函数虽然会扩展一个类的功能，但是这新功能并不属于这个类，只是存在一个引用的关系。
  
  
扩展函数的作用域：
首先我们得清楚两个名词得概念，1：扩展接收者，2：分发接收者
扩展接收者：扩展函数所要扩展得那个类的实例；
分发接收者：扩展函数定义在哪个类里面，那个类的实例就叫做分发接收者。

class A{ //分发接收者
  fun aa() {}
  fun run() {}
  
  //作用域1：扩展函数都可以调用分发接收者和扩展接收者中的函数
  fun B.call(){ //扩展函数
    aa() //A类的函数
    bb() //B类的函数
  }
  
  //作用域2：当扩展接收者和分发接收者中存在相同的函数时，扩展函数调用了该函数，扩展接收者的优先级最高。
  fun B.call2(){ //扩展函数
    run() //A类、B类都有run函数，它会走哪一个呢？走的是B类的run函数
  }
}

class B{ //扩展接收者
    fun bb() {}
    fun run() {}
}

```



### 说说中缀函数？ - infix

```java
中缀表达式：操作符在中间，比较符合人的阅读习惯，就像a+b一样一目了然。
  
kotlin的中缀函数是标有infix关键字修饰的函数，常见的如to，until，step，into等。to在这里就是创建一个二元元组pair，下面是用了中缀表达式和不用的写法对比。
  
fun main(){
    val a = "a" to 2
    val b = Pair<String,Int>("a",2)
    val c = "a".to(2)

    //输出结果都是（a，2）
    println(a)
    println(b)
    println(c)
}

自己写一个中缀表达式
必要条件是
1.只能有一个参数
2.必须是成员函数或拓展函数
3.参数不能是可变参数或默认参数
  
class Money{
    var cost = 10
    infix fun add(amount:Int){
        this.cost = cost + amount
    }
}

fun main(){
    val money = Money()
    money add 5
    println(money.cost) //输出15
}
```







### ==与===的区别。

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



### Array、List、MutableList有什么区别？

```java
Array：（相当于java的数组）
List：有序接口，只能读取，不能更改元素。（kotlin自己重写的EmptyList，EmptyList中没有提供add方法remove方法等修改元素操作的方法）
MutableList：有序接口，可以读写与更改、删除、增加元素。（相当于java的ArrayList）
```



### 注解 @JvmOverloads 的作用？

```java
在有默认参数值的方法中使用@JvmOverloads注解，则Kotlin就会暴露多个重载方法。

@JvmOverloads fun f(a: String, b: Int=0, c:String="abc"){}

// 相当于Java三个方法 不加这个注解就只能当作第三个方法这唯一一种方法
void f(String a)
void f(String a, int b)
// 加不加注解，都会生成这个方法
void f(String a, int b, String c)
```



### Unit的应用以及和Java中void的区别？

```java
在java中，必须指定返回类型，即void不能省略，但是在kotlin中，如果返回为unit，可以省略。
java中void为一个关键字，但是在kotlin中unit是一个类。
```



### Kotlin中 的 Any 与Java中的 Object 有何异同

```java
相同：都是顶级父类
不同：成员方法不同，Any只声明了toString()、hashCode()和equals()作为成员方法。

为什么Kotlin还要设计一个Any？
当我们需要和 Java 互操作的时候，Kotlin 把 Java 方法参数和返回类型中用到的 Object 类型看作 Any，这个 Any 的设计是 Kotlin 兼容 Java 时的一种权衡设计。

所有 Java 引用类型在 Kotlin 中都表现为平台类型。当在 Kotlin 中处理平台类型的值的时候，它既可以被当做可空类型来处理，也可以被当做非空类型来操作。

试想下，如果所有来自 Java 的值都被看成非空，那么就容易写出比较危险的代码。反之，如果 Java 值都强制当做可空，则会导致大量的 null 检查。综合考量，平台类型是一种折中的设计方案。
```



### Kotlin中的构造方法？有哪些注意事项？

```java
1、kotlin中构造函数分为主构造和次级构造两类
2、使用关键词constructor标记次级构造函数，部分情况可省略
3、init关键词用于初始化代码块，注意与构造函数的执行顺序，类成员的初始化顺序
4、继承，扩展时候的构造函数调用逻辑
5、特殊的类如data class、object/componain object、sealed class等构造函数情况与继承问题
6、构造函数中的形参声明情况

主/次 构造函数：
kotlin中任何class（包括object/data class/sealed class）都有一个默认的无参构造函数
如果显式的声明了构造函数，默认的无参构造函数就失效了。
主构造函数写在class声明处，可以有访问权限修饰符private,public等，且可以省略constructor关键字。
若显式的在class内声明了次级构造函数，就需要委托调用主构造函数。
若在class内显式的声明处所有构造函数（也就是没有了所谓的默认主构造），这时候可以不用依次调用主构造函数。例如继承View实现自定义控件时，三四个构造函数同时显示声明。
  
init初始化代码块：
kotlin中若存在主构造函数，其不能有代码块执行，init起到类似作用，在类初始化时侯执行相关的代码块。
init代码块优先于次级构造函数中的代码块执行。
即使在类的继承体系中，各自的init也是优先于构造函数执行。
在主构造函数中，形参加有var/val，那么就变成了成员属性的声明。这些属性声明是早于init代码块的。
```



3. 

   

## 其他

### kotlin与java它们的相同点与区别

```java
相同点：
① 和java一样都是编译成class文件，然后被虚拟机加载。
② 可互操作（kt中用java，java中用kt）

区别：
① kt更加简洁
  比如：data类，自动重写了get，set，equals，hashCode，toString等方法
  比如：单例类，可以直接使用object实现java饿汉单例模式
  比如：高阶函数等等
② 代码更安全 - 空安全
 比如：可以通过?进行判空，不为空的才能继续往下走。
 比如：类默认不可继承，方法默认不可重写(相当于java中final)，如果需要继承或者重写都需要加open关键字，字段建议使用val，不可变。
```



### Kotlin与Java混合开发时需要注意哪些问题？

```java
kotlin调用java的时候，如果java返回值可能为null 那就必须加上@nullable@nullable否则kotlin无法识别，也就不会强制你做非空处理，一旦java返回了null 那么必定会出现null指针异常，加上@nullable注解@nullable之后kotlin就能识别到java方法可能会返回null，编译器就能会知道，并且强制你做非null处理，这也就是kotlin的空安全。
```





1. kotlin有什么优缺点？+2
2. kotlin的lambda与java的lambda有什么区别。
3. data class 与 Gson发生的解析异常。

