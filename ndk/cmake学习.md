## cmake是什么

### makefile是什么

```java
Makefile简介：https://www.cnblogs.com/sddai/p/10329299.html

makefile作用：配置项目的编译规则
项目的makefile都写好了以后，只需要一个简单的make命令，就可以实现自动编译了。
  
make命令工具帮助我们实现我们想要做的事，而makefile就相当于是一个规则文件，make程序会按照makefile所指定的规则，去判断哪些文件需要先编译，哪些文件需要后编译，哪些文件需要重新编译。
  
 
假设我们编译单个源文件：
执行命令：g++ -o test test.cpp
完成编译。
  
我们用makefile文件完成编译，makefile内容为 test:test.cpp
执行命令：gmake -f makefile文件名
编译器为我们执行：g++ -o test test.cpp
完成编译。
  
这种单个源文件的情况，一般的项目会有成千上万的源文件，并且还会有各种额外配置。我们手写g++命令会炸的，所以makefile能给我们非常简单地完成编译配置。

  
makefile最基本的描述规则：
  TARGET：规则生成的目标文件，通常是需要生成的程序名（例如前面出现的程序名test）或者过程文件（类似.o文件）。
  PREREQUISITES：规则的依赖项，比如前面的Makefile文件中我们生成test程序所依赖的就是test.cpp。
  COMMAND：规则所需执行的命令行，通常是编译命令。这里需要注意的是每一行命令都需要以[TAB]字符开头。
  
再来看我们之前写过的Makefile文件，这个规则，用通俗的自然语言翻译过来就是：
	1、如果目标test文件不存在，根据规则创建它。
	2、目标test文件存在，并且test文件的依赖项中存在任何一个比目标文件更新（比如修改了一个函数，文件被更新了），根据规则重新生成它。
	3、目标test文件存在，并且它比所有的依赖项都更新，那么什么都不做。
```



### cmake简介

```java
CMake是一个跨平台的开源构建工具，它可以让我们通过一个与开发平台无关的CMakeLists.txt来定制整个编译流程，然后再根据目标用户的平台进一步生成所需的makefile和工程文件。
  
cmake命令将CMakeList.txt转化为make所需要的makefile。最后用make命令编译源代码生成可执行文件。
  
cmake语法介绍：https://juejin.cn/post/6844904067492216845
```



## CMakeLists常用命令

cmakelist学习文章：https://juejin.cn/post/6844904067492216845

### CMakeLists.txt例子

```cmake
# 指定cmake最低支持的版本。这个命令是可选的，如果要使用高版本特有的命令的话就需要加上这条命令，来指定CMake的最低支持版本。
cmake_minimum_required(VERSION 3.4.1)

# 添加一个库，根据native-lib.cpp源文件编译一个native-lib的动态库
add_library(
	native-lib
	SHARED
	native-lib.cpp)

# 查找系统库，这里查找的是系统日志库，并赋值给变量log-lib
find_library(
	log-lib
	log)

# 设置依赖的库（第一个参数必须为目标模块，顺序不能换）
target_link_libraries(
	native-lib
	${log-lib})
```



### add_library

#### 添加一个库

```cmake
add_library(
 <name> # 添加一个库文件
 [STATIC | SHARED] # 指定库的类型。STATIC：静态库，SHARED：动态库
 [EXCLUDE_FROM_ALL] # 表示该库不会被默认构建
 source1 source2 ... sourceN # 用来指定库的源文件
 )
```

#### 导入预编译库

```cmake
# add_library 配合 set_target_properties 使用

add_library(test SHARED IMPORTED)
set_target_properties(
	test # 指明目标库名
	PROPERTIES IMPORTED_LOCATION # 指明要设置的参数
	库路径/${ANDROID_ABI}/libtest.so # 导入库的路径
)
```



### set - 设置CMake变量

```cmake
# 设置可执行文件的输出路径（EXECUTABLE_OUTPUT_PATH是全局变量）
set(EXECUTABLE_OUTPUT_PATH [output_path])

# 设置库文件的输出路径（LIBRARY_OUTPUT_PATH是全局变量）
set(LIBRARY_OUTPUT_PATH [output_path])

# 设置C++编译参数（CMAKE_CXX_FLAGS是全局变量）
set(CMAKE_CXX_FLAGS "-Wall std=c++11")

# 设置源文件集合（SOURCE_FILES是本地变量即自定义变量）
set(SOURCE_FILES main.cpp test.cpp ...)
```



### include_directories - 设置头文件目录

```cmake
# 可以用相对或绝对路径，也可以用自定义的变量值（相当于g++选项中的-l参数）
include_directories(./include ${MY_INCLUDE})
```



### add_executable - 添加可执行文件

```cmake
add_executable(<name> ${SRC_LIST})
```



### target_link_libraries - 将若干库链接到目标库

```cmake
# 链接的顺序应当符合gcc链接顺序规则，被链接的库放在依赖它的库的后面，即下面的命令中，lib1依赖于lib2，lib2又依赖于lib3，则在上面命令中必须严格按照lib1 lib2 lib3的顺序排列，否则会报错。
target_link_libraries(<name> lib1 lib2 lib3)

# 如果出现互相依赖的静态库，CMake会允许依赖图中包含循环依赖，如：
add_library(A STATIC a.c)
add_library(B STATIC b.c)
target_link_libraries(A B)
target_link_libraries(B A)
add_executable(main main.c)
target_link_libraries(main A)
```



### add_definitions - 添加编译参数

```cmake
我们更改别人代码做实验时使用，既不对其源码进行破坏，又可以添加自己的功能。

具体的，在工程CMakeLists.txt 中，使用add_definitions()函数控制代码的开启和关闭：

//代码里：
#ifdef TEST_DEBUG
...
#else 
...
#endif


//CMakeLists.txt里：
option(TEST_DEBUG "option for debug" OFF)
if (TEST_DEBUG) 
	add_definitions(-DTEST_DEBUG)
endif(TEST_DEBUG)


//运行构建项目的时候可以添加参数控制宏的开启和关闭
cmake -DTEST_DEBUG=1 .. #打开
cmake -DTEST_DEBUG=0 .. #关闭
```



### set_directory_properties - 设置某个路径的一种属性

```cmake
set_directory_properties(PROPERTIES prop1 value1 prop2 value2)

prop1，prop2代表属性，取值为：
INCLUDE_DIRECTORIES
LINK_DIRECTORIES
INCLUDE_REGULAR_EXPRESSION
ADDITIONAL_MAKE_CLEAN_FILES
```



### set_property - 给定的作用域内设置一个命名的属性

```cmake
set_property(<GLOBAL | DIRECTORY [dir] | TARGET [target ...] | SOURCE [src1 ...] | TEST [test1 ...] | CACHE [entry1 ...]>
			[APPEND]
			PROPERTY <name> [value ...])
			
PROPERTY 参数是必须的。第一个参数决定了属性可以影响的作用域：

GLOBAL：全局作用域
DIRECTORY：默认当前路径，也可以用[dir]指定路径。
TARGET：目标作用域，可以是0个或多个已有目标。
SOURCE：源文件作用域，可以是0个或多个源文件。
TEST：测试作用域，可以是0个或多个已有的测试。
CACHE：必须指定0个或多个cache中已有的条目。
```





### file - 文件操作命令

```cmake
# 将message写入filename文件中，会覆盖文件原有内容
file(WRITE filename "message")
# 将message写入filename文件中，会追加在文件末尾
file(APPEND filename "message")
# 从filename文件中读取内容并存储到var变量中，如果指定了numBytes和offset，
# 则从offset处开始最多读numBytes个字节，另外如果指定了HEX参数，则内容会以十六进制形式存储在var变量中
file(READ filename var [LIMIT numbytes] [OFFSET offset] [HEX])
# 重命名文件
file(RENAME <oldname> <newname>)
# 删除文件，等于rm命令
file(REMOVE [file1 ...])
# 递归的执行删除文件命令，等于rm -r
file(REMOVE_RECURSE [file1...])
# 根据指定的url下载文件
# timeout超时时间；下载的状态会保存到status中；下载日志会被保存到log；sum指定所下载文件预期的MD5值，如果指定会自动进行比对，
# 如果不一致，则返回一个错误；SHOW_PROGRESS，进度信息会以状态信息的形式被打印出来
file(DOWNLOAD url file [TIMEOUT timeout] [STATUS status] [LOG log] [EXPECTED_MD5 sum] [SHOW_PROGRESS])
# 创建目录
file(MAKE_DIRECTORY [dir1 dir2 ...])
# 会把path转换为以unix的/开头的cmake风格路径，保存到result中
file(TO_CMAKE_PATH path result)
# 它会把cmake风格的路径转换为本地路径风格：windows下用"\"，而unix下用"/"
file(TO_NATIVE_PATH path result)
# 将会为所有匹配查询表达式的文件生成一个文件list，并将该list存储进变量variable里，如果一个表达式指定了RELATIVE，返回的结果
# 将会是相对于定路径的相对路径，查询表达式例子：*.cxx，*.vt?
# NOTE：按照官方文档的说法，不建议使用file的GLOB指令来收集工程的源文件
file(GLOB variable [RELATIVE path] [globbing expressions]...)

使用这种方式来指定源文件的话，如果后面项目需要增加源文件，比如项目中原本有2个源文件a.c和b.c，后面新增一个c.c，如果我们直接增加c.c，然后进行编译是会报错的，因为CMakeLists文件没有改动，所以cmake不会重新生成makefile文件，所以我们需要简单的改动一下CMakeLists文件，可以在CMakeLists文件中添加一个空格，然后再编译，它就会生成makefile文件。
```

