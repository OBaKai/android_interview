# Android虚拟机原理03（dex文件反汇编原理详解)

> **码牛学院用代码码出自己牛逼的人生**

##### 1.1 dex文件反编译后代码

	 以一个非常简单的函数为例：
> frameworks/base/core/java/android/app/Activity.java:988
>

```
   public void onCreate(@Nullable Bundle savedInstanceState,
            @Nullable PersistableBundle persistentState) {
       onCreate(savedInstanceState);
   }
```

> 这个函数，仅仅是调用了同名的onCreate函数的实现。
> 编译成oat文件后（存放在boot.oat内），用oatdump出来，我找到对应的dex代码

```
  129: void android.app.Activity.onCreate(android.os.Bundle, android.os.PersistableBundle) (dex_method_idx=2085)
    DEX CODE:
      0x0000: 6e20 2408 1000            | invoke-virtual {v0, v1}, void android.app.Activity.onCreate(android.os.Bundle) // method@2084
      0x0003: 0e00                      | return-void
```



这个dex代码看起来也非常简单。
接下来，我们看看编译出的机器码，反汇编后是什么:

    QuickMethodFrameInfo
      frame_size_in_bytes: 48
      core_spill_mask: 0x000040e0 (r5, r6, r7, r14)
      fp_spill_mask: 0x00000000 
      vr_stack_locations:
        ins: v0[sp + #52] v1[sp + #56] v2[sp + #60]
        method*: v3[sp + #0]
        outs: v0[sp + #4] v1[sp + #8]
    CODE: (code_offset=0x0337a6d5 size_offset=0x0337a6d0 size=66)...
      0x0337a6d4: f5ad5c00  sub     r12, sp, #8192
      0x0337a6d8: f8dcc000  ldr.w   r12, [r12, #0]
        StackMap [native_pc=0x337a6dd] (dex_pc=0x0, native_pc_offset=0x8, dex_register_map_offset=0xffffffff, inline_info_offset=0xffffffff, register_mask=0x0, stack_mask=0b000000)
      0x0337a6dc: b5e0      push    {r5, r6, r7, lr}
      0x0337a6de: b088      sub     sp, sp, #32
      0x0337a6e0: 9000      str     r0, [sp, #0]
      0x0337a6e2: f8b9c000  ldrh.w  r12, [r9, #0]  ; state_and_flags
      0x0337a6e6: f1bc0f00  cmp.w   r12, #0
      0x0337a6ea: d10a      bne     +20 (0x0337a702)
      0x0337a6ec: 461d      mov     r5, r3
      0x0337a6ee: 460e      mov     r6, r1
      0x0337a6f0: 4617      mov     r7, r2
      0x0337a6f2: 6808      ldr     r0, [r1, #0]
      0x0337a6f4: f8d004c8  ldr.w   r0, [r0, #1224]
      0x0337a6f8: f8d0e020  ldr.w   lr, [r0, #32]
      0x0337a6fc: 47f0      blx     lr
        StackMap [native_pc=0x337a6ff] (dex_pc=0x0, native_pc_offset=0x2a, dex_register_map_offset=0x0, inline_info_offset=0xffffffff, register_mask=0xe0, stack_mask=0b000000)
          v0: in register (6)   [entry 0]
          v1: in register (7)   [entry 1]
          v2: in register (5)   [entry 2]
      0x0337a6fe: b008      add     sp, sp, #32
      0x0337a700: bde0      pop     {r5, r6, r7, pc}
      0x0337a702: 9103      str     r1, [sp, #12]
      0x0337a704: 9204      str     r2, [sp, #16]
      0x0337a706: 9305      str     r3, [sp, #20]
      0x0337a708: f8d9e2a8  ldr.w   lr, [r9, #680]  ; pTestSuspend
      0x0337a70c: 47f0      blx     lr
        StackMap [native_pc=0x337a70f] (dex_pc=0x0, native_pc_offset=0x3a, dex_register_map_offset=0x3, inline_info_offset=0xffffffff, register_mask=0x0, stack_mask=0b111000)
          v0: in stack (12) [entry 3]
          v1: in stack (16) [entry 4]
          v2: in stack (20) [entry 5]
      0x0337a70e: 9903      ldr     r1, [sp, #12]
      0x0337a710: 9a04      ldr     r2, [sp, #16]
      0x0337a712: 9b05      ldr     r3, [sp, #20]
      0x0337a714: e7ea      b       -44 (0x0337a6ec)

​	这个代码很长，你也许会奇怪，为什么简单的一句源码，会产生如此多的机器码呢？下面我为大家分段解析。
解析时，请大家注意对照我说的部分

##### 1.2 QuickMethodFrameInfo详解

​		首先，看到的”QuickMethodFrameInfo”部分，定义了相关代码的信息，包含：frame_size_in_bytes： 这个信息指出堆栈的大小。该例子中的值是48。下面的机器码中(0x0337a6dc-0x0337a6de)，我们可以看到push指令和sub指令正好占用了堆栈的48字节。
core_spill_mask： 需要保存的寄存器值。这个值是0x000040e0，对应的就是0x337a6dc的 push {r5, r6, r7, lr} (lr即r14)。
vr_stack_locations: 则定义了dex寄存器在堆栈上的位置。

Code部分
接下来，CODE部分，包含了所有相关的代码。CODE部分包括5部分：
> 1. 堆栈检测部分：0x0337a6d4-0x337a6d8，检测现在的堆栈是否溢出的，否则将会产生一个堆栈溢出的错误
> 2. 程序头部分：0x0337a6dc-0x0337a6f0, 主要是保护现场，保存参数，检查停止点等
> 3. 程序正文部分：0x0337a6f2-0x0337a6fc，dex字节码的实现部分
> 4. 程序结尾部分：0x0337a6fe-0x0337a700 负责恢复现场，然后返回
> 5. 执行检查点部分：0x0337a702-0x0337a714: 调用检查点，执行一些任务，比如GC等等。
> 这些部分，包含了一个程序大部分组成。
> 第一部分的堆栈检测比较简单，我们不详细说了，从第二部分开始。
>

**程序头部分**
这一部分的代码

      0x0337a6dc: b5e0      push    {r5, r6, r7, lr}
      0x0337a6de: b088      sub     sp, sp, #32
      0x0337a6e0: 9000      str     r0, [sp, #0]
      0x0337a6e2: f8b9c000  ldrh.w  r12, [r9, #0]  ; state_and_flags
      0x0337a6e6: f1bc0f00  cmp.w   r12, #0
      0x0337a6ea: d10a      bne     +20 (0x0337a702)
      0x0337a6ec: 461d      mov     r5, r3
      0x0337a6ee: 460e      mov     r6, r1
      0x0337a6f0: 4617      mov     r7, r2
     
    
	需要注意的是: 在ART里面，参数传递是使用r0~r3 (ARM架构），这一点与标准的C stdcall是一样的。不过，r0是被固定为参数ArtMethod 
	
	即目前被调用函数的指针，如果是非static函数，那么r1则是this指针，否则r1就是第一个参数。依次是java函数中声明的其他参数。
	如果参数超过了寄存器的数目，则用堆栈传递。 
​		这里看到，前两条语句是保存寄存器并分配堆栈。第3条语句(0x0337a6e0) 将r0(ArtMethod*)的值放在了栈顶。这是ART的规范要求，所有的java method，必须在堆栈上放入它的函数指针，这样在遍历栈的时候，ART就知道这个栈归属哪个函数。
​		第4～6句 (0x0337a6e2-0x0337a6ea) 是读取 Thread对象的state_and_flags成员，判断是否有检查点，如果有则转到检查点来执行。
r9参数总是放的当前线程的Thread对象指针。
​		第7句以后，则是保存参数到寄存器中，以便后面使用（实际上并没有使用，这说明ART的编译器还存在优化的空间）。 

调用部分
要调用一个函数，ART必须做的事情是：1. 得到这个函数的入口地址；2. 准备参数；3. 调用

      0x0337a6f2: 6808      ldr     r0, [r1, #0]
      0x0337a6f4: f8d004c8  ldr.w   r0, [r0, #1224]
      0x0337a6f8: f8d0e020  ldr.w   lr, [r0, #32]
      0x0337a6fc: 47f0      blx     lr
​		在这里，r1是this指针，第0x0337a6f2是取得this的class对象，放到r0，然后从r0中取得被调用的onCreate函数的ArtMethod 指针，再保存到r0中。第三步是从ArtMethod中取得入口地址（放在32偏移处），最后调用。
onCreate(Bundle) 函数需要3个参数：

> r0: 指向onCreate(Bundle)的ArtMethod ;
> r1: this指针
> r2: Bundle对象指针。

​		我们看到，ART巧妙的将ArtMethod的值取到R0后，就把它作为参数来使用。而r1,r2的值自进入函数后就没有被改变，所以就直接使用了。
​		最后一步调用blx跳转到函数入口。 blx指令是跳转到寄存器指定的地址，同时把下条指令的地址放入到lr寄存器，方便函数调用后返回。

**程序结束部分**
这部分很简单，就是恢复堆栈和寄存器。
因为这个函数没有返回值，所以这里省略了返回值的处理。正常情况下，返回值是放在r0,r1寄存器内的。

**检查点调用部分**
检查点是ART内一个非常重要的功能，ART依靠检查点来实现GC、异常处理等一系列关键功能。
ART为每个线程都分配了一个Thread对象，这个对象中有一个pTestSuspend入口，这个入口就是用来执行检查点函数调用的。

      0x0337a702: 9103      str     r1, [sp, #12]
      0x0337a704: 9204      str     r2, [sp, #16]
      0x0337a706: 9305      str     r3, [sp, #20]
      0x0337a708: f8d9e2a8  ldr.w   lr, [r9, #680]  ; pTestSuspend
      0x0337a70c: 47f0      blx     lr
      0x0337a70e: 9903      ldr     r1, [sp, #12]
      0x0337a710: 9a04      ldr     r2, [sp, #16]
      0x0337a712: 9b05      ldr     r3, [sp, #20]
      0x0337a714: e7ea      b       -44 (0x0337a6ec)

​		这段代码从0x0337a702-0x0337a712 是调用pTestSuspend入口的，调用入口不需要参数，所以前面它保存了r1, r2, r3三个参数，调用后又恢复了r1,r2,r3参数。至于要保存那些参数，根据实际情况定。
在最后一条语句，则是返回到调用现场，继续运行。这里它没有使用blx这样的调用，而是直接用b指令，目的应该是为了尽量减少寄存器的使用。

##### 1.3 了解dex各类指令以及异常的实现方法

​		接下来，我们了解下各类dex指令的实现。这些指令的实现都和class、object、method、field的实现结构密切相关，但是，在我们具体了解这些结构的具体内容之前，我们先了解下ART是怎样使用它们的，就更容易理解这些结构为什么被设计成那种样子。

上文提到了invoke的实现，这里我们就不再解释

> 1. dex指令的实现
> 2. const-string
> 3. boot内的字符串查找