---
layout: post
title: String 类型对象内存使用分析
categories: Java Tools
excerpt: 利用 HSDB 分析字符串对象的内存分布
image: https://upload.wikimedia.org/wikipedia/en/3/30/Java_programming_language_logo.svg
description: 利用 HSDB 分析字符串对象的内存分布
keywords: String Runtime Constant Pool Memory intern HotSpot JVM
licences: cc
---

<br/>

<img src="https://upload.wikimedia.org/wikipedia/en/3/30/Java_programming_language_logo.svg" alt="Java Logo" width="300"/>

## 分析工具

### HSDB

#### 快速简介

&emsp;&emsp;HSDB（HotSpot Debugger）是一个用于分析 Java 应用程序性能问题的工具。它是针对 Java HotSpot 虚拟机的调试器，可以帮助开发人员诊断和调试应用程序中的性能瓶颈和问题。它提供了一系列功能，包括实时监控应用程序的性能指标、跟踪方法调用、分析内存使用情况、查看线程状态等。使用 Hotspot Debugger，开发人员可以深入了解应用程序在 JVM 层面的执行情况，从而更好地优化和调试 Java 应用程序的性能。

#### 使用说明

* JDK 8 及之前使用方法

  &emsp;&emsp;JDK 8 及之前版本并没有将 HSDB 打包成一个可执行文件，需要手动设置环境变量后以 Java 命令行形式启动。

  * 设置环境变量

    ```bash
    # 导出 JDK 1.8 主目录环境变量
    export JAVA_HOME=/path/to/jdk1.8
    # 导出 sa-jdi.jar 路径
    # Serviceability Agent Java virtual machine Debuger Interface
    # 服务性代理 Java 虚拟机调试接口
    export SA_JDI=$JAVA_HOME/lib/sa-jdi.jar
    ```

  * 启动 HSDB

    ```shell
    # 注意，可能需要管理员权限
    sudo java -cp $SA_JDK sun.jvm.hotspot.HSDB
    ```

    &emsp;&emsp;启动后将出现如下图形化界面：

    ![HSDB 界面](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/start.png)

  * 关联执行的 Java 应用进程

    &emsp;&emsp;使用 `jps` 或 `jcmd` 命令查看需要被调试的 Java 应用进程号 `pid`

    ```shell
    # 方式一：
    jps
    # 方式二：
    jcmd
    ```

    &emsp;&emsp;在 HSDB 工具菜单选择 【File】--> 【Attach to HotSpot process】

    ![关联进程1](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/attach.png)

    &emsp;&emsp;填入 `pid` 并点击 【Ok】

    ![关联进程2](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/attach2.png)

    &emsp;&emsp;稍等片刻，默认将显示 **Java Thread** 线程视图窗口

    ![attach3](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/attach3.png)

    &emsp;&emsp;选中 `main` 主线程，工具栏点击显示线程栈内存信息的图标即可显示该线程栈情况

    ![关联进程4](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/attach4.png)

    ![关联进程4](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/attach5.png)

* JDK 9 及之后使用方法

  &emsp;&emsp;JDK 9 及之后在 `JAVA_HOME/bin` 目录引入了可执行工具 `jhsdb` 取代之前的启动方式，该工具支持命令行方式运行 HSDB，也可以支持上面一样的的图像界面方式，简单执行命令获取帮助菜单：

  ```shell
  # 请将 JDK bin 放入 path 环境变量 
  jhsdb -h
      clhsdb       	command line debugger	# 命令行方式运行调试
      debugd       	debug server			
      hsdb         	ui debugger				# 图形化方式运行调试
      jstack --help	to get more information
      jmap   --help	to get more information
      jinfo  --help	to get more information
      jsnap  --help	to get more information
      
  # 执行图像化方式调试, 以管理员方式运行防止出现权限问题
  sudo jhsdb hsdb
  ```

  &emsp;&emsp;运行效果和之前类似，详细的使用教程参考：[JDK 9 Tools Reference - jhsdb](https://docs.oracle.com/javase/9/tools/jhsdb.htm#JSWOR-GUID-0345CAEB-71CE-4D71-97FE-AA53A4AB028E)

## 分析 String 内存分配

### 示例代码

```java
package demo;

public class Test {
    public static int a = 0x12345678;
    public static String b = "abcdef";

    public static void main(String[] args) {
        Test t = new Test();
        Class clazz = Test.class;
        String c = new String("abcdef");
        System.out.println(c == t.b);
        System.out.println(c.intern() == t.b);
    }
}
```

### 分析目标

&emsp;&emsp;使用 HSDB 工具调试分析示例代码执行 `main` 方法时各种 String 类型对象的内存分布情况，并画出内存示意图。

### 分析思路

* 利用 HSDB 的线程栈内存获得 `main` 线程执行 `main()` 方法时栈帧内存结构
* 根据栈上变量的引用地址查看相应类型的对象在堆上的数据，从中分析其中的 String 类型对象的内存所在区域
* 结合分析结果画出方法调用时的内存示意图

## 实际操作

#### 运行环境

* 本操作是基于 JDK 8 版本的 HSDB 工具
* 程序源码以 Java 8 版本编译及运行

#### JVM 内存区域

* 在菜单栏【Tools】中选择子菜单【Heap Parameters】获得 JVM 运行时内存区域划分信息：

  ![虚拟机运行时内存区域地址分别](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/8.png)

  格式整理后显示如下：

  ```shell
  Heap Parameters:
  # 采用 PS+PO 方式的 GC 策略来划分内存区域
  ParallelScavengeHeap 
  [ 
  	# 新生代
  	PSYoungGen 
  	[ 
  		#       起始地址              已使用到地址          结束地址
  		eden =  [0x0000000795580000, 0x00000007958c3800, 0x0000000797600000] , 
  		from =  [0x0000000797b00000, 0x0000000797b00000, 0x0000000798000000] , 
  		to   =  [0x0000000797600000, 0x0000000797600000, 0x0000000797b00000]  
  	] 
  	# 老年代
  	PSOldGen 
  	[  
  	    #       起始地址            已使用到地址          结束地址
  		[0x0000000740000000, 0x0000000740000000, 0x0000000745580000]  
  	]  
  ] 
  ```

#### VM 栈

```java
public static void main(String[] args) {
    Test t = new Test();
    Class clazz = Test.class;
    String c = new String("abcdef");
    System.out.println(c == t.b);
    System.out.println(c.intern() == t.b);
}
```

![主方法执行栈帧](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/1.png)

* 内存结构说明

  ```sh
  
  # 栈帧物理内存地址			# 存储值				# 备注说明
  #----------------------------------------------------------------------------
  #  高位 <-- 低位         小端存储，字节序已翻转
  #============================================================================
  0x00007000043fb9f0      0x00000007958660e0      PSYoungGen java/lang/String
  ```

  &emsp;&emsp;如上图所示，方法本地变量区域中的变量对应的引用分别为：

  * 变量 **`t`**：**`0x7958660d0`**, 在堆的 `eden` 区
  * 变量 **`clazz`**: **`0x795861e38`**，在堆的 `eden` 区
  * 变量 **`c`**: **`0x7958660e0`**，在堆的 `eden` 区

#### 变量 `t`

&emsp;&emsp;由变量 `t` 对象所在的内存地址为： **`0x7958660d0`**，采用 **Memory Viewer** 工具查询如下：

![var-t](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/2.png)

```java
package demo;

public class Test {
    public static int a = 0x12345678;
    public static String b = "abcdef";
    // 省略方法
}
```

&emsp;&emsp;结合 `demo.Test` 类代码结构看，该类只包含类静态变量而没有对象成员变量，所以**对象无数据**，图中绿色剪头所指示的只是用来填充占位用的 `0x00000000`, 使对象内存占用变为 `8` 的整数倍，原理自行查阅***对象对齐***。 下面着重看看该对象的内存信息：

```shell
0x00000007958660d0	0x0000000000000001 	--> markword
0x00000007958660d8	0x00000000f800c005 	--> 0xf800c005 Test.class 类元信息的压缩地址指针
```

&emsp;&emsp;可以看到 `Test` 类型的对象在内存占用 16 字节，前 8 个字节为 对象的 ***MarkWord***，后面 8 字节中的低位 4 个字节为 ***KlassWord***（64 位操作系统下采取 [*COOP*](https://wiki.openjdk.java.net/display/HotSpot/CompressedOops) 指针压缩省空间）[^1] ，它指向该对象类型的字节码信息存储区域，该区域在 JVM 规范中属方法区。由于本例采用 HotSpot 虚拟机 JDK 8 版本，采用***元空间***方式实现，被放在本地内存区域。

&emsp;&emsp;先通过对指针 **`0xf800c005`** 解压缩，看其所指向的真实物理地址：

```shell
# 通常寻址的步长为 1 字节，4 字节（32 bit）地址最多只能寻址 4 GB 内存空间
# JVM 将寻找步长扩展为 8 字节，并使对象以默认 8 字节方式对齐，则 4 字节长度地址最大可寻找 32 GB 内存空间
# 故：对压缩后的指针解压缩，只需将其扩大 8 倍（左移 3 位）
0xf800c005 << 3 = 0x7c0060028
```

> 采用左移 3 的算法前提是采用了 **Zeao Nerrow OOP Base** 技术[^2]，否则左移后还需要加上 Java 堆基地址。

&emsp;&emsp;解压缩后地址 **`0x7c0060028`** ，参考上面堆内存分配详情可知其已超出 JVM 堆内存区域范围，**间接证实 Java 8 中用来保存字节码信息的元空间已经在本地内存区域分配**。本案例重点不是分析字节码信息在内存中的真实存储情况，故此处不在跳转到该地址空间查看数据。

#### 变量 `clazz`

&emsp;&emsp;JVM 虚拟机类加载（Loading、Linking、Initalizing）行为会将类字节码文件加载到内存，然后经过验证、准备、解析后将类的**元信息**保存在**方法区**，同时将其包含的静态常量表收集汇总到**运行时常量池**，最终还会生成 **`Class<Test>`** 类型的实例对象用于在 JVM 中表示该类。上面解压缩后的内存地址 **`0x7c0060028`** 所存储的即为 **`Test`** 类元信息，通过 **Class Browser** 工具来看看该类元信息：

![Test meta data](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/7.png)

&emsp;&emsp;元数据中显示，两个静态变量 ` a` 和 `b` 在类对象的偏移量分别为：`104`、`108`。根据虚拟机规范描述，类静态变量发生在类加载的初始化阶段，而该阶段会在类首次被主动使用时完成。主动使用包括如下几种方式：

* 实例化类的对象
* 访问类或接口的静态变量
* 调用类或接口的静态方法
* 使用反射调用该类
* 初始化该类子类

&emsp;&emsp;本例中使用 `new Test()` 实例化对象触发静态变量的初始化，确保接下来查看下面类实例对象 `Class<Test> clazz = Test.class` 在内存地址 **`0x795861e38`** 的存储情况时，两个静态变量已完成赋值：

![Test.class](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/3.png)

&emsp;&emsp;可以看到类对象中也保留了指向**元空间**中**类元数据信息指针**：**`0x7c0060028`**，该指针是未被压缩的**本地内存**指针。 偏移量 `104`、`108` 对应的静态变量 `a` 和 `b` 的确已经分别赋值为：`0xf2b0cc11` 引用、`0x12345678` 整形值。将字符串变量 `b` 的压缩对象指针解压后的物理地址为：**`0x795866088`** , 接着查看堆中该对象：

![String object](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/4.png)

&emsp;&emsp;图中绿色显示的为该对象类元数据压缩指针，解压缩地址： Java 8 中 `String` 类是存在两个成员变量的：

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    // 字符串数组
    private final char value[];
    // 哈希值
    private int hash;
    // ...
}
```

&emsp;&emsp;图中蓝色部分的压缩指针 **`0xf2b0cc14`** 即为 `value[]` 字符数组对象的地址，解压缩的到物理地址为：**`0x7958660a0`**，同样是在 `eden` 堆空间，观察该地址对象：

![Char array object](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/5.png)

&emsp;&emsp;可以看到该对象是一个字符数组类型，其字节码元信息 `char[].class` 在压缩地址 `0xf8000041` 中保存，这里就不在解压缩观察。可以发现数组对象相比非数组对象在 **对象头** 中多出了 4 字节用来描述数组长度，本例字符数组长度值为 `0x00000006`，即字符数组长度为 6，后续数据中依次存储这 6 个字符，每个字符占用 2 字节（Unicode 编码）。

#### 变量 `c`

&emsp;&emsp;继续跟随代码执行 `String c = new String("abcdef")`，来观察变量 **`c`**: **`0x7958660e0`**，在堆的存储：

![String c object](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/6.png)

&emsp;&emsp;不难发现，字符串对象 `c` 内部的字符数组 `value[]` 指向的地址与静态字符串变量 `b = "abcdef"` 中的 `value[]` 成员属性地址相同。这是因为 JVM 在 **Test** **静态成员初始化时已在堆中创建字符串对象（地址：0x795866088）并将字面量** **`abcdef`** 为 key，字符串对象引用为 value 放入**字符串常量池**，其成员变量 `value[]` 指向堆内存中的字符数组对象（地址：0x7958660a0）。

&emsp;&emsp;而后实例化 `c` 变量时，调用 `new` 方法时所传递的字面量时，实际上传递的已然是字面量所引用的字符串对象（地址：0x795866088），在构造方法中将后者的 `value[]` 的引用（0x7958660a0）赋值给 c 字符串的相同成员变量 `value[]`。

####  分析结果

&emsp;&emsp;通过上面对变量的分析，已经获得所有相关字符串变量在内存的地址信息，结合 JVM 运行时内存堆地址范围的数据，可以定位各个变量在 JVM 内存区域位置，从而画出最终的变量内存分布图：

![Memory](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jhsdb/9.jpg)

&emsp;&emsp;值得注意**类静态变量**因为存储在**类对象**中，故随对象一起放在堆内存，而**类元信息**放在**本地内存**中的**元空间**。

&emsp;&emsp;**字符串字面量**所对应的字符串对象若在**字符串常量池**中未找到，**将在堆中创建该字符串字面量对应的字符串对象**，然后生成一个以字符串字面量为 Key，堆中对应的字符串对象引用为 Value 的哈希 Entry 放入到字符串常量池（StringTable）。

&emsp;&emsp;根据上面的分析结果，示例程序的如下几行打印结果的原因就很清晰：

```java
// false, 0x7958660e0 ≠ 0x795866088
System.out.println(c == t.b);
// true, 0x795866088 = 0x795866088
System.out.println(c.intern() == t.b);
```

&emsp;&emsp;执行 `c.intern()` 时，常量池以字面量 "abcdef" 为 key 找到了对应的字符串引用 `0x795866088` 并将此引用返回，而该引用与 `t.b` 静态字符串变量的引用一致。

## 推荐阅读

[^1]: [JVM Compressed OOPs](https://www.baeldung.com/jvm-compressed-oops)
[^2]: [CompressedOops - Zero based compressed oops](https://wiki.openjdk.org/display/HotSpot/CompressedOops)
