

## elf文件学习

### 概要

```java
ELF：Executable and Linkable Format的缩写，其中的“Executable”和“Linkable”表明ELF文件有两种重要的特性。
Executable：可执行。ELF文件将参与程序的执行（Execution）工作。包括二进制程序的运行以及动态库.so文件的加载。Linkable：可链接。ELF文件是编译链接工作的重要参与者。
```





## dex文件学习

Dex文件结构分析：https://www.jianshu.com/p/463d1dd66b96

### 概要

```java
总结：
dex = header + 多个索引表 + data
header：校验信息 + 所有索引表的大小以及偏移 + data大小以及偏移
多个索引表：索引表里边都是对应的结构体，结构体里边的字段都是偏移，拿这些偏移去data区找才能找到数据。
data：存放实际数据的地方（字符串常量、类定义、方法代码等等）
```



![dex总览](/Users/jjleong/Public/my_github_repo/android_interview/art/assets/dex总览.png)



#### 为什么Android要用dex，而不用class？

##### ① dex更好的利用arm cpu中的寄存器

###### Jvm虚拟机栈 - 栈帧中的 操作数栈 & 局部变量表

![虚拟机栈结构](/Users/jjleong/Public/my_github_repo/android_interview/art/assets/虚拟机栈结构.png)

```java
1、操作数栈 & 局部变量表 从哪里来？
当一个方法要被执行时，JVM会为它在虚拟机栈里创建一个栈帧。在栈帧里边就会有一个操作数栈以及一个局部变量表。

2、操作数栈 & 局部变量表 是怎么定长的？
它们的长度都是在编译期间就确定了。

.class结构 
  -> 方法结构（method_info）
  -> 方法的Code属性 
  -> max_stack：最大栈深度（操作数栈的定长）
     max_locals：最大局部变量（局部变量表定长）


3、操作数栈 & 局部变量表 的作用是什么？
操作数栈：
一个栈结构的数组，只有出栈和入栈操作。
主要用于保存计算过程的中间结果，同时作为计算过程中变量临时的存储空间。
操作数栈执行过程案例：https://copyfuture.com/blogs-details/202206290915417901


局部变量表：
一个数组。
主要用于存储方法参数和定义在方法体内的局部变量，这些数据类型包括各类基本数据类型、对象引用，以及returnAdrress类型。

局部变量表最基本的存储单元是Slot(变量槽)。
32位以内的内型只占用一个slot(包括returnAddress类型)，64位类型(long和doble)占用两个slot。
①、byte、short、char在存储前都会转换为int，boolean也被转换为int，0表示false，非0表示true。
②、long和double 则占据两个slot。

slot的重复利用：
栈帧中的局部变量表中的槽位是可以重复利用的，如果一个局部变量过了其作用域，那么在其作用域之后申明的新的局部变量就很有可能会复用过期局部变量的槽位，从而达到节省资源的目的。
```



###### dex的指令码可以更好地利用寄存器

```java
Android系统主要针对移动设备，而移动设备的内存、存储空间相对PC平台而言较小。
并且主要使用ARM的CPU，这种CPU有一个显著特点，就是通用寄存器比较多。
在这种情况下，Class格式的文件在移动设备上不能扬长避短。
  
Class文件中指令码执行的时候需要不断存取 操作数栈 & 局部变量表。操作都是在内存里边的进行的。
而在移动设备上，由于ARM的CPU有很多通用寄存器，Dex中的指令码可以利用它们来存取参数。
显然，寄存器的存取速度比内存中的存取速度要快得多。
  
Dex指令码的条数和Class指令码差不多，都不超过255条，但是Dex文件中存储函数内容的insns数组却比Class文件中存储函数内容的code数组解析起来要有难度。
其中一个原因是Android虚拟机在执行指令码的时候不需要操作数栈，所有参数要么和Class指令码一样直接跟在指令码后面，要么就存储在寄存器中。对于参数位于寄存器中的指令，指令码就需要携带一些信息来表示该指令执行时需要操作哪些寄存器。
```



#### ② dex文件比class文件集（jar包）的冗余度更低

```java
Class文件通过索引方式能减少字符串等信息的冗余度，这个只能个解决单个Class文件的冗余度。在多个Class文件之间可能还是有重复字符串等信息。
  
而dex文件由于包含了多个Class文件的内容，所以可以进一步去除其中的重复信息。
```



#### ③ 减少文件I/O的次数

```java
如果一个Class文件依赖另外一个Class文件，则虚拟机在处理的时候需要读取另外一个Class文件的内容，这可能会导致CPU和存储设备进行更多的I/O操作。
  
而dex文件由于一个文件就包含了所有Class的信息，相对而言会减少I/O操作的次数。
```



### dex文件总览

| 数据名称   | 介绍                                                         |
| ---------- | ------------------------------------------------------------ |
| header     | dex文件头部，记录整个dex文件的相关属性                       |
| string_ids | 字符串数据索引，记录了每个字符串在数据区的偏移量             |
| type_ids   | 类似数据索引，记录了每个类型的字符串索引                     |
| proto_ids  | 原型数据索引，记录了方法声明的字符串，返回类型字符串，参数列表 |
| field_ids  | 字段数据索引，记录了所属类，类型以及方法名                   |
| method_ids | 类方法索引，记录方法所属类名，方法声明以及方法名等信息       |
| class_defs | 类定义数据索引，记录指定类各类信息，包括接口，超类，类数据偏移量 |
| data       | 数据区，保存了各个类的真是数据                               |
| link_data  | 连接数据区                                                   |



```java
struct DexFile { // dalvik/libdex/DexFile.h
    const DexHeader*    pHeader;
    const DexStringId*  pStringIds;
    const DexTypeId*    pTypeIds;
    const DexFieldId*   pFieldIds;
    const DexMethodId*  pMethodIds;
    const DexProtoId*   pProtoIds;
    const DexClassDef*  pClassDefs;
    const DexLink*      pLinkData;
}

跟上边的 总览表格 是不是一一对应上了。
  
？？？ 
为什么没有 data ???
data是确实存在的，但是不需要用结构体表示出来。因为data区域存放实际内容的，其实只需要知道data区域的大小以及偏移地址就好了。
data区域的大小以及偏移地址 放到了 header 里边了。
  
DexFile里边的那些什么 DexStringId、DexTypeId...都是存索引数组而已，真正数据都是放在data区的。
```



### DexHeader - Dex头结构体

```java
struct DexHeader {
    u1  magic[8]; //值必须是"dex\n035\0" 或 {0x64, 0x65, 0x78, 0x0a, 0x30, 0x33, 0x35, 0x00}
    u4  checksum; //文件内容的校验和，用于检查文件是否损坏
    u1  signature[kSHA1DigestLen]; //签名信息，用于检查文件是否被篡改
    u4  fileSize; //整个文件的长度，单位为字节
    u4  headerSize; //默认是0x70个字节
    u4  endianTag; //标识处理文件内容字节序。默认0x12345678（Little Endian）如果是0x78563412（Big Endian）
  
    u4  linkSize; //链接段的大小，如果为0就是静态链接
    u4  linkOff; //链接段的开始位置
    u4  mapOff; //map数据基址
  
    u4  stringIdsSize; //字符串id列表大小（DexStringId*的长度）
    u4  stringIdsOff; //字符串id列表基址
  
    u4  typeIdsSize; //类id列表大小（DexTypeId*的长度）
    u4  typeIdsOff; //类id列表基址
  
    u4  protoIdsSize; //原型id列表大小（DexProtoId*的长度）
    u4  protoIdsOff; //原型id列表基址
  
    u4  fieldIdsSize; //字段id列表大小（DexFieldId*的长度）
    u4  fieldIdsOff; //字段id列表基址
  
    u4  methodIdsSize; //方法id列表大小（DexMethodId*的长度）
    u4  methodIdsOff; //方法id列表基址
  
    u4  classDefsSize; //类定义列表大小（DexClassDef*的长度）
    u4  classDefsOff; //类定义列表基址
  
    u4  dataSize; //数据段的大小，必须4k对齐
    u4  dataOff; //数据段基址
};
```



### DexStringId - 字符串常量索引

```java
在 header 里边有很多 xxxIdsSize & xxxIdsOff，它们的作用是什么？
根据size + off确定某个结构体数组的位置以及长度
 
以 stringIdsSize & stringIdsOff 为例。
u4  stringIdsSize; //字符串id列表大小
u4  stringIdsOff; //字符串id列表基址

stringIdsSize + stringIdsOff 就能够代表 DexFile中 DexStringId* pStringIds;
也就是一个字符串id，对应的是一个DexStringId的偏移地址。

比如：
stringIdsSize = 0E 00 00 00（转十进制就是 14，有14个DexStringId）
stringIdsOff  = 76 10 00 00

那么 DexStringId* 在dex中的位置就是：7610h -（7610h + 14 * 4字节）
但是在这个范围里边并不是DexStringId结构体的内容，只是个索引地址。
每4个字节代表一个DexStringId索引，可以通过这个索引地址，找到真正的DexStringId内容。
  
假设：
1、
DexStringId*在dex中的位置：76 10 00 00 7E 01 00 00 ....

2、
第一个DexStringId偏移地址为：76 10 00 00
第二个DexStringId偏移地址为：7E 01 00 00
记住offset=76 10，但是找的时候需要反过来的 0176h

3、
0170h 01 00 00 00 06 00 06 3C 69 6E 69 74 3E 00 0B 48
0180h 65 6C 6C 6F 20 57 6F 72 6C 64 00 ..............

3.1、找第一个DexStringId
offset=76 10，那么我们应该找 0176h
找到 0170h 的地方。
？？？
不是 0176h 为什么是 0170h（0170h是行号，6是列号。）
0170h 〇 ① ② ③ ④ ⑤ 06 3C 69 6E 69 74 3E 00 0B 48
也就是说 06 3C 69 6E 69 74 3E 00 才是第一个DexStringId
？？？
为什么是 06 3C 69 6E 69 74 3E 00 后面的那些就不要了吗？

DexStringId结构：长度（06，不包含00这个结束符） + 内容（6个字符） + 结束符（00）
真正字符串内容是：3C 69 6E 69 74 3E --ascii--> <init>
  
3.2、找第二个DexStringId
offset=7E 01，那么我们应该找 017Eh
还是找到 0170h 的地方（0170h是行号，E是列号。）
0170h 〇 ① ② ③ ④ ⑤ ⑥ ⑦ ⑧ ⑨ A B C D 0B 48
0180h 65 6C 6C 6F 20 57 6F 72 6C 64 00 ..............

第二个DexStringId：0B 48 65 6C 6C 6F 20 57 6F 72 6C 64 00
长度：11
字符串内容：48 65 6C 6C 6F 20 57 6F 72 6C 64 --ascii--> Hello World
```



### DexFieldId - 字段索引

```java
struct DexFieldId{
	u2 classIdx; //类的类型（用来确认字段在哪个类里边），指向一个DexTypeId索引
  u2 typeIdx; //字段类型，指向一个DexTypeId索引
  u4 nameIdx; //字段名，指向一个DexStringId索引
}

例如：
int HelloWorld.a
java.lang.String HelloWorld.b
```



### DexMethodId - 方法索引

```java
struct DexMethodId{
	u2 classIdx; //类的类型（用来确认字段在哪个类里边），指向一个DexTypeId索引
	u2 protoIdx; //方法声明类型，指向一个DexProtoId索引
	u4 nameIdx;	//方法名，指向一个DexStringId索引
}

例如：
void java.lang.Object.<init>()
void java.io.PrintStream.println(java.lang.String)
```



### DexClassDef - 类定义结构体

```java
struct DexClassDef{ //类定义
	u4 classIdx; /*类的类型，指向DexTypeId列表的索引*/
	u4 accessFlags; /*访问标志*/
	u4 superclassIdx;	/*父类类型，指向DexTypeId列表的索引*/
	u4 interfacesOff;	/*接口，指向DexTypeList的偏移*/
	u4 sourceFileIdx;	/*源文件名，指向DexStringId列表的索引*/
	u4 annotationsOff; /*注解，指向 DexAnnotationsDirectoryItem 结构的偏移*/
	u4 classDataOff; /*类的数据（所有字段、方法等），指向 DexClassData 结构的偏移*/
	u4 staticValuesOff;	/*指向 DexEncodedArray 结构的偏移*/
}

struct DexClassData{ //存储类的数据部分
	DexClassDataHeader header; /*指定字段与方法的个数，声明的是下边那些结构体数组的大小 */
	DexField* staticFields; /*静态字段，DexField结构*/
	DexField* instanceFields; /*实例字段，DexField结构*/
	DexMethod* directMethods; /*直接方法，DexMethod结构*/
	DexMethod* virtualMethods; /*虚方法，DexMethod结构*/
}

struct DexClassDataHeader{
	u4 staticFieldsSize; /*静态字段个数*/
	u4 instanceFieldsSize; /*实例字段个数*/
	u4 directMethodsSize;	/*直接方法个数*/
	u4 virtualMethodsSize;  /*虚方法个数*/
}
 
struct DexField{
	u4 fieldIdx; /*指向DexFieldId的索引*/
	u4 accessFlags; /*访问标志*/
}
 
struct DexMethod{
	u4 methodIdx; /*指向DexMethodId的索引*/
	u4 accessFlags; /*访问标志*/
	u4 codeOff; /*指向DexCode结构的偏移*/
}
```



##### DexCode - 代码段结构体

```java
struct DexCode { //Android 7.1 art/runtime/dex_file.h:281
    u2  registersSize; /* 该方法用到的总寄存器总个数 */
    u2  insSize; /* 入参所占空间，单位是2字节 */
    /* registersSize 和 insSize 进一步说明
    registersSize 指的是虚拟寄存器的个数，并非物理寄存器
    insSize 方法入参个数，同时也是入参占虚拟寄存器的个数
    registersSize - insSize 就是方法内部创建变量的个数
    */
  
    u2  outsSize; /* 在该方法调用子方法时，所需参数占用的空间，单位是2字节 */
    
    u4  insnsSize; /* 指令数组的大小*/
    u2  insns[1]; /* 指令数组的起始地址（指令数组）*/
    //这里可以看出，Dex文件中指令码长度为2个字节，而Class文件指令码长度为1个字节（Class只能有255个指令）
  
    u2  triesSize; /* try/catch个数 */
   /* 还有这些跟try/catch相关的字段，没列出来
   try_item[triesSize] DexTry结构
   handlersSize try/catch中handler的个数
   catch_handler_item[handlersSize] DexCatchHandler结构
   */
  
    u4  debugInfoOff; /* 指向调式信息的偏移 */
  
    ... //还有一些的字段的
};
```





### DexTypeId - 类的类型索引

```java
struct DexTypeId {
	u4 descriptorIdx; //类的类型，指向一个DexStringId索引
}

DexTypeId#descriptorIdx -> 某DexStringId -> 字符串

那DexTypeId到底是什么？？？
Ljava/lang/Object;
Ljava/lang/String;
...
就是这些，类的类型
```



### DexProtoId - 方法声明索引

```java
struct DexProtoId {
	u4 shortyIdx; //方法声明的字符串，指向一个DexStringId索引
	u4 returntypeIdx; //方法的返回类型，指向一个DexTypeId索引
	u4 parameterOff; //方法的参数列表，指向一个DexTypeList位置偏移
}

struct DexTypeList {
	u4 size; //DexTypeItem的个数
	DexTypeItem list[1];
}

struct DexTypeItem {
	u2 typeIdx; //指向一个DexTypeId索引
}

那DexProtoId到底是什么？？？
void (java.lang.String)
void (java.lang.String[])
...
就是这些，方法声明
```




## class文件学习

class文件浅析：https://cloud.tencent.com/developer/article/1333538

### 概要

```java
javap -v xxx.class //可查看class信息

class文件的存储形式就是一个二进制字节流

总结：
class = 类信息 + 常量池 + 字段表 + 方法表 + 属性表
类信息、字段表、方法表、属性表里边的值大部分都是常量池的索引，指向常量池中的某个常量。
  
为什么要设计常量池？为了减少class文件的体积。
比如：类信息中某个值是"A"，字段表中某个字段的某个值也是"A"，那么这个"A"常量就重复了。如果类信息与字段值都指向"A"常量的索引，这样就能节省掉一个"A"常量的空间了。
  
//u1、u2、u4、u8分别表示1、2、4、8个字节的无符号数 
```



### 类 - ClassFile

```java
ClassFile { //class文件包含了虚拟机指令集、符号表、若干辅助信息
	u4 magic; //class的魔数值固定为0xCAFEBABE
	u2 minor_version; //class文件版本的小版本
	u2 major_version; //主版本号

	u2 constant_pool_count; //常量池计数，值等于常量池表中的成员个数加1
	cp_info constant_pool[constant_pool_count-1]; //常量池（每个常量由 cp_info 结构体表示）

	u2 access_flags; //标明该类的访问权限，比如public、private之类的信息。

	u2 this_class; //当前类类名（存储索引值，指向常量池的某个元素）
	u2 super_class; //父类类名（存储索引值，指向常量池的某个元素）

	u2 interfaces_count; //实现的接口数（存储索引值，指向常量池的某个元素）
	u2 interfaces[interfaces_count]; //接口表（存储索引值，指向常量池的某个元素）

	u2 fields_count; //成员变量的数量（包括static变量、非static变量，但不包括继承的）
	field_info fields[fields_count]; //成员变量表（每个变量由 field_info 结构体表示）

	u2 methods_count; //成员方法的数量（不包括继承而来的）
	method_info methods[methods_count]; //成员方法表（每个方法由 method_info 结构体表示）

	u2 attributes_count; //属性数量（哪些是属性？如：调试信息就记录了某句代码对应源文件哪一行、方法对应的字节码也属于属性信息、源文件中的注解）
	attribute_info attributes[attributes_count]; //属性表（每个属性由 attribute_info 结构体表示）
}
```



### 常量池 - cp_info

```java
cp_info{ //常量元素结构体
	u1 tag; //常量类型（不同的tag，info区域的大小以及含义都是不同的）
	u1 info[]; //常量数据
}
```



#### tag类型

| CONSTANT_Class              | 7    |
| :-------------------------- | :--- |
| CONSTANT_Fieldref           | 9    |
| CONSTANT_Methodref          | 10   |
| CONSTANT_InterfaceMethodref | 11   |
| CONSTANT_String             | 8    |
| CONSTANT_Integer            | 3    |
| CONSTANT_Float              | 4    |
| CONSTANT_Long               | 5    |
| CONSTANT_Double             | 6    |
| CONSTANT_NameAndType        | 12   |
| CONSTANT_Utf8               | 1    |
| CONSTANT_MethodHandle       | 15   |
| CONSTANT_MethodType         | 16   |
| CONSTANT_InvokeDynamic      | 18   |



#### info类型

```java
CONSTANT_Utf8_info { //字符串常量
    u1 tag; 
    u2 length; 
    u1 bytes[length]; 
}

CONSTANT_Integer_info { //int整型
    u1 tag;	
    u4 bytes;	
}

CONSTANT_Float_info { //单精度浮点型 float
    u1 tag;	
    u4 bytes;	
}

CONSTANT_Long_info { //long 长整型（分为4个高字节和4个低字节）
    u1 tag;
    u4 high_bytes;
    u4 low_bytes;
}

CONSTANT_Double_info { //双精度浮点型 double
    u1 tag;
    u4 high_bytes;
    u4 low_bytes;
}

CONSTANT_NameAndType_info { //名称与类型（方法/字段）
    u1 tag;
    u2 name_index;  //也是指向一个CONSTANT_Utf8_info
    u2 descriptor_index;  //也是指向一个CONSTANT_Utf8_info
}

CONSTANT_String_info { //String类型的常量对象（它表示String类型的数据，具体的字符串常量还需要指向CONSTANT_Utf8_info）
    u1 tag;
    u2 string_index; //也是指向一个CONSTANT_Utf8_info
}

CONSTANT_MethodType_info { //方法类型		
    u1 tag;
    u2 descriptor_index;  //也是指向一个CONSTANT_Utf8_info
}

CONSTANT_Class_info { //类或接口
    u1 tag;
    u2 name_index; //也是指向一个CONSTANT_Utf8_info
}

CONSTANT_Fieldref_info { //字段
    u1 tag;
    u2 class_index; //指向一个CONSTANT_Class_info
    u2 name_and_type_index; //指向一个CONSTANT_NameAndType_info
}

CONSTANT_Methodref_info { //方法
    u1 tag;
    u2 class_index; //指向一个CONSTANT_Class_info
    u2 name_and_type_index; //指向一个CONSTANT_NameAndType_info
}

CONSTANT_InterfaceMethodref_info { //接口方法
    u1 tag;
    u2 class_index; //指向一个CONSTANT_Class_info
    u2 name_and_type_index; //指向一个CONSTANT_NameAndType_info
}

CONSTANT_MethodHandle_info { //方法调用
    u1 tag;
    u1 reference_kind;
    u2 reference_index;
}

//invokedynamic 是为了更好的支持动态类型语言，Java7通过JSR292给JVM增加的一条新的字节码指令bootstrap_method_attr_index的值必须是对当前Class文件中引导方法表的bootstrap_methods[]数组的有效索引name_and_type_index指向CONSTANT_NameAndType 表示方法名和方法描述符
CONSTANT_InvokeDynamic_info { //用于表示invokedynamic指令
    u1 tag;
    u2 bootstrap_method_attr_index;
    u2 name_and_type_index;
}
```



### 字段 - field_info

```java
field_info {
    u2 access_flags; //访问权限
    u2 name_index; //字段名，一个CONSTANT_utf8_info的索引
    u2 descriptor_index; //字段描述符，一个CONSTANT_utf8_info的索引
    u2 attributes_count; //属性数量
    attribute_info attributes[attributes_count];	//属性表	
}
```



#### 字段的access_flags类型 	

| ACC_PUBLIC    | 0x0001 | 字段是否为public 可以包外访问    |
| :------------ | :----- | :------------------------------- |
| ACC_PRIVATE   | 0x0002 | 字段是否为private 只能本类访问   |
| ACC_PROTECTED | 0x0004 | 字段是否为protected 子类可以访问 |
| ACC_STATIC    | 0x0008 | 字段是否为static                 |
| ACC_FINAL     | 0x0010 | 字段是否为final                  |
| ACC_VOLATILE  | 0x0040 | 字段是否为volatile               |
| ACC_TRANSIENT | 0x0080 | 字段是否为transient              |
| ACC_SYNTHETIC | 0x1000 | 字段是否由编译器产生             |
| ACC_ENUM      | 0x4000 | 字段是否为enum                   |



### 方法 - method_info

```java
method_info {
    u2 access_flags; //访问权限
    u2 name_index; //方法名，一个CONSTANT_utf8_info的索引
    u2 descriptor_index; //方法描述符，一个CONSTANT_utf8_info的索引
    u2 attributes_count; //属性数量
    attribute_info attributes[attributes_count];	//属性表	
}
```



#### 方法的access_flags类型 	

| ACC_PUBLIC       | 0x0001 | 方法是否为public 包外访问                           |
| :--------------- | :----- | :-------------------------------------------------- |
| ACC_PRIVATE      | 0x0002 | 方法是否为private 当前类访问                        |
| ACC_PROTECTED    | 0x0004 | 方法是否为protected 子类访问                        |
| ACC_STATIC       | 0x0008 | 方法是否为static                                    |
| ACC_FINAL        | 0x0010 | 方法是否为final                                     |
| ACC_SYNCHRONIZED | 0x0020 | 方法是否为synchronized                              |
| ACC_BRIDGE       | 0x0040 | 方法是否为 编译器为了字节码兼容自动生成的bridge方法 |
| ACC_VARARGS      | 0x0080 | 方法是否为变长参数                                  |
| ACC_NATIVE       | 0x0100 | 方法是否为native 本地方法                           |
| ACC_ABSTRACT     | 0x0400 | 方法是否为abstract 无实现代码                       |
| ACC_STRICT       | 0x0800 | 方法是否为strictfp 使用FP-strict浮点模式            |
| ACC_SYNTHETIC    | 0x1000 | 方法是否为编译器自动产生而不是由源代码编译而来      |



### 属性 - attribute_info

```java
//跟常量池不用，属性是由其名称来区别的
//也就是靠 attribute_name_index 指向的字符串来区分的。

attribute_info { 
    u2 attribute_name_index; //属性的名字索引，一个CONSTANT_utf8_info的索引
    u4 attribute_length;
    u1 info[attribute_length];
}
```



#### 常见的属性

##### Code属性 - 存储方法内代码编译后的字节码

```java
Code_attribute { //Code属性只存在于方法中（没法方法体的是没有Code属性，比如接口、抽象方法、native方法）
    u2 attribute_name_index; //固定指向"Code"字符串常量
    u4 attribute_length; //属性长度
  
    /*
    jvm执行一个指令，该指令的操作数会存储在一个名叫“操作数栈”的地方，每一个操作数占用一个或两个（long、double类型的操作数）栈项。stack就是一块只能进行先入后出的内存。
    max_stack 表示这个函数在执行过程中，需要最深多少栈空间（也就是多少栈项）。
    max_locals 表示该函数包括最多几个局部变量。
    注意，max_stack和max_locals都和JVM如何执行一个函数有关。
    根据JVM官方规范，每一个函数执行的时候都会分配一个操作数栈和局部变量数组。
    所以Code_attribute需要包含这些内容，这样JVM在执行函数前就可以分配相应的空间。
    */
    u2 max_stack; //方法在执行过程中需要最深多少栈空间
    u2 max_locals; //方法最大的局部变量数
    
    u4 code_length; //字节码指令的长度
    u1 code[code_length]; //存储字节码指令
  
    u2 exception_table_length; //方法中包含的try/catch异常个数
    {
      u2 start_pc; //try/cath从哪条指令开始
      u2 end_pc; //try语句到哪条指令结束
      u2 handler_pc;	//catch语句的内容从哪条指令开始
      u2 catch_type; //catch捕获的Exception（CONSTANT_Class_info），如果取值为0则表示它是final语句块
    } exception_table[exception_table_length]; //方法中包含的try/catch表
  
    u2 attributes_count; //code其他属性数量
    attribute_info attributes[attributes_count];
}

//code其他属性
StackMapTable：JVM加载Class文件的时候，将利用该属性的内容对函数进行类型校验（Type Checking）
LineNumberTable：用于调试，比如指明哪条指令。对应于源码哪一行。
LocalVariableTable：用于调试，调试时可以用于计算本地变量的值。
LocalVariableTypeTable：功能和LocalVariableTable类似。
```



##### ConstantValue属性 - 通知虚拟机为静态变量赋值

```java
/*
只有被static关键字修饰的变量才可以使用这个属性  也就是只有类变量才可以使用 	
非static类型的变量也就是实例变量的赋值在构造方法中<init> 	
 类变量可以再<clinit>方法中  也可以使用ConstantValue  属性 	
目前编译器的做法是 如果同时使用final和static来修饰,也就是常量了
如果变量的数据类型是基本类型或者java.lang.String的话,就生成ConstantValue 	
如果没有final 或者并非基本类型或者字符串 选择在<clinit>中 	
*/
ConstantValue_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 constantvalue_index;
}
```



##### Exceptions属性 - 声明方法中可能会抛出的异常

```java
/*
不是Code中的 exception_table 注意区分
列举出方法中可能抛出的已检查的异常 		
也就是方法声明throws后面的内容 	
*/
Exceptions_attribute {	
    u2 attribute_name_index;
    u4 attribute_length;
    u2 number_of_exceptions;
    u2 exception_index_table[number_of_exceptions];
}
```



##### SourceFile属性

```java
/*
class文件的源文件名,属性是可选的可以关闭 	
但是一旦关闭,当抛出异常时 不会显示出错代码所归属的文件名称 	
*/
SourceFile_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 sourcefile_index;
}
```





#### 属性的划分

| 属性                                                         | 位置                                                         | 备注                                                         | 版本 |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :--- |
| SourceFile                                                   | ClassFile                                                    | 表示class文件的源文件名称类独有属性                          | 45.3 |
| InnerClasses                                                 | ClassFile                                                    | 内部类相关信息类独有属性                                     | 45.3 |
| EnclosingMethod                                              | ClassFile                                                    | class为局部类或者匿名类才具有类独有属性                      | 49.0 |
| SourceDebugExtension                                         | ClassFile                                                    | 可选/保存扩展调试信息/最多一个类独有属性                     | 49.0 |
| BootstrapMethods                                             | ClassFile                                                    | 与  invokedynamic指令 					 常量池中CONSTANT_InvokeDynamic_info 					 相关 					 类独有属性 | 51.0 |
| ConstantValue                                                | field_info                                                   | fina修饰的字段的常量值 字段独有属性                          | 45.3 |
| Code                                                         | method_info                                                  | java程序方法体中的代码经过javac编译器处理后最终变为字节码指令存储在Code属性内Code属性出现在方法表的属性集合中抽象类和接口不存在code属性包含了方法的java虚拟机指令及相关辅助信息方法独有属性 | 45.3 |
| Exceptions                                                   | method_info                                                  | 方法可能抛出的已检查异常列表方法独有属性                     | 45.3 |
| RuntimeVisibleParameterAnnotations, 					 RuntimeInvisibleParameterAnnotations | method_info                                                  | 形参上的运行时的注解信息类型分为可见和不可见两种类型方法独有属性 | 49.0 |
| AnnotationDefault                                            | method_info                                                  | method_info表示注解类型中的元素时记录这个元素的默认值方法独有属性 | 49.0 |
| MethodParameters                                             | method_info                                                  | 形参相关信息,比如参数名称 方法独有属性                       | 52.0 |
| Synthetic                                                    | classFilefield_infomethod_info                               | Synthetic 标志编译器生成类 字段 方法都可能由编译器生成所以三种都有此属性 | 45.3 |
| Deprecated                                                   | classFilefield_infomethod_info                               | 语义同@Deprecated显然可以标注在类/接口/字段/方法上所以三种都有此属性 | 45.3 |
| Signature                                                    | classFile 					 field_info 					 method_info | 泛型信息类接口 字段 方法 都有可能有类型参数所以三种都有此属性 | 49.0 |
| RuntimeVisibleAnnotations, 					 RuntimeInvisibleAnnotations | classFile 					 field_info 					 method_info | 类 方法 字段上 运行时注解的可见性分为可见不可见两种类型三种都有此属性 | 49.0 |
| LineNumberTable                                              | Code                                                         | 调试用信息用于调试器确定源文件中给定行号所表示的内容,对应于虚拟机中code[]数组中的哪一部分也就是行号与字节码指令的对应关系 | 45.3 |
| LocalVariableTable                                           | Code                                                         | 调试用信息调试器执行方法过程中可以用它来确定某个局部变量的值 | 45.3 |
| LocalVariableTypeTable                                       | Code                                                         | 调试用信息 					 调试器执行方法过程中可以用它来确定某个局部变量的值 | 49.0 |
| StackMapTable                                                | Code                                                         | 虚拟机类型检查验证使用信息                                   | 50.0 |
| RuntimeVisibleTypeAnnotations, 					 RuntimeInvisibleTypeAnnotations | classFile 					 field_info 					 method_info | 类/方法/字段声明所使用的类型上面的运行时注解可见性分为可见/不可见两种三种都有此属性 |      |



