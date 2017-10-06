---
layout: post
title: Java 9 新特性 - Compact Strings
categories: Java
tags: Java Java9 Compact String
excerpt: 浅谈 Java 9 中的字符串压缩技术 【译文】
image: https://www.ibm.com/developerworks/community/blogs/ibmandgoogle/resource/BLOGS_UPLOADED_IMAGES/java-logo-2.png
description: Java 9 中的 Compact Strings 
keywords: Java,  java, Java9, java9, Compact String, compact string, Reion Chan, reionchan
--- 

[原文链接](http://www.baeldung.com/java-9-compact-string) 作者：baeldung 译者：Reion Chan

## 概述
  
字符串在 Java 的  `String` 类内部由一个包含该字符串中所有字符的 `char[]`  来表示，  
其中的每个字符 `char` 又是由 2 个字节组成，因为 **Java 内部使用 UTF-16**。  
举例来说，如果一个字符串含有英文字符，那么这些英文字符的前 8 比特都将为 0，  
因为一个ASCII字符都能被单个字节来表示。 
 
当然有许多字符需要 16 比特，但从统计角度来说只需 8 比特的情况占大多数，例如：LATIN-1 ，  
因此这能成为一种改善内存占用及性能的一个机会。更重要的是：由于 JVM 存储字符串的方式导致 JVM 堆空间通常很大一部分都被字符串所占据。  
  
大多数情况下，**字符串实例常占用比它实际需要的内存多一倍的空间**。  
  
在此篇文章中，我们将讨论 JDK 6 引入的可选的 *Compressed String* 功能及 JDK 9 最新引入的 *Compact String* ，它们两者的设计目的都是优化字符串在 JVM 中的内存占用。  

## Compressed String - Java 6  
 
 *JDK 6 update 21* 版本中引入了一个新的虚拟机参数选项：  

```java  
-XX:+UseCompressedStrings
```  
  
当此选项启用时，字符串将以 `byte[]` 的形式存储，代替原来的 `char[]`，因此可以节省一些内存。然而，此功能最终在 JDK 7 中被移除，主要原因在于它将带来一些无法预料的性能问题。  
  
## Compact String - Java 9  

Java 9 重新采纳字符串压缩这一概念。  
  
这意味着**无论何时我们创建一个所有字符都能用一个字节的 LATIN-1 编码来描述的字符串，都将在内部使用字节数组的形式存储**，且每个字符都只占用一个字节。  

另一方面，如果字符串中任一字符需要多于 8 比特位来表示时，该字符串的所有字符都统统使用两个字节的 UTF-16 编码来描述。因此基本上能如果可能，都将使用单字节来表示一个字符。  

现在的问题是：所有的字符串操作如何执行？ 怎样才能区分字符串是由 LATIN-1 还是 UTF-16 来编码？  
  
为了处理这些问题，字符串的内部实现进行了一些调整。引入了一个 `final` 修饰的成员变量 `coder`, 由它来保存当前字符串的编码信息。  

### Java 9 中字符串的实现  

之前，字符串都是以字符数组来存储：  

```java
private final char[] value;  
```  
 
 现在，它将使用字节数组来存储了：  
  
```java
private final byte[] value;  
``` 

成员变量 `coder` 的声明：

```java
private final byte coder;  
``` 
 
 `coder` 可以被赋值为如下的常量：
 
 ```java
static final byte LATIN1 = 0;
static final byte UTF16 = 1;
```

现在，大多数的字符串操作都将检查 `coder` 变量，从而采取特定的实现：  
  
```java  
public int indexOf(int ch, int fromIndex) {
    return isLatin1() 
      ? StringLatin1.indexOf(value, ch, fromIndex) 
      : StringUTF16.indexOf(value, ch, fromIndex);
}  
 
private boolean isLatin1() {
    return COMPACT_STRINGS && coder == LATIN1;
} 
```  

所有 JVM 需要的信息都准备就绪，虚拟机参数 `CompactString` 默认是被启用的，如果想要关闭它，可以使用如下启动参数：  
  
```java  
+XX:-CompactStrings
```

### `coder` 的作用

Java 9 中字符串长度计算是按如下方式实现的：  
  
```java  
public int length() {
    return value.length >> coder;
}
```  

如果字符串只包含 LATIN-1编码的字符，`coder` 变量的值将为 0，因此字符串的长度就为字节数组的长度一致，否则 `coder` 变量的值为1，采用 UTF-16 编码，那么字符串的长度为字节数组长度的一半。  
  
**注意，所有因为采用 Compact String 技术而做出的改变，都单单存在于 String 类内部实现，因此对使用 String 类的开发者而言此种改变是完全透明的。**

## Compact Strings VS Compressed Strings

JDK 6 的 *Compressed Strings* ，主要的问题在于 `String` 类的构造方法只接受 `char[ ]` 类型的参数，因此许多字符串的操作方法都依赖于字符数组的表现形式，而非字节数组。介于此，很多的方法都需将压缩后的字节数组解压缩为字符数组，无形中影响了性能。  

然而，JDK 9 的 *Compact Strings* ，维护额外的 `coder` 字段也会新增一些开销，为了缓和 `coder` 及 bytes 解压为 chars（针对 UTF-16编码的情形） 所带来的开销，一些方法都被[内置化](https://en.wikipedia.org/wiki/Intrinsic_function)，由 JIT 编译器产生的汇编代码也将被优化改进。  
  
这些改变也会带来一些反常规的结果。LATIN-1 编码的字符串的 `indexOf(String)` 方法能够调用内置化方法，而 `indexOf(char)` 方法却不能。 UTF-16 编码的字符串的这两种方法都能够调用内置化方法。这个问题只出现在 LATIN-1 编码的字符串中，会在将来的版本中被修复。  

因此， *Compact Strings*  在性能方面要优于 *Compressed Strings* 。
许多的 Java 堆转存工具都可以用来分析采用 *Compact Strings* 技术后究竟能节省多少内存。当然结果严重依赖具体的应用，总体而言改善还是相当可观的。  
  
### 性能差异

让我们通过一个非常简单的例子来观察开启与关闭 *Compact Strings* 时的性能差异：  
  
```java  
long startTime = System.currentTimeMillis();
  
List strings = IntStream.rangeClosed(1, 10_000_000)
  .mapToObj(Integer::toString) 
  .collect(toList());
  
long totalTime = System.currentTimeMillis() - startTime;
System.out.println(
  "Generated " + strings.size() + " strings in " + totalTime + " ms.");
 
startTime = System.currentTimeMillis();
  
String appended = (String) strings.stream()
  .limit(100_000)
  .reduce("", (l, r) -> l.toString() + r.toString());
  
totalTime = System.currentTimeMillis() - startTime;
System.out.println("Created string of length " + appended.length() 
  + " in " + totalTime + " ms.");
```  
  
这里，我们创建了 1 千万个字符串，并且以简单的方式合并这些字符串。  
运行这段代码后（默认开启了 *Compact Strings*），我们得到如下的输出：  

```
Generated 10000000 strings in 854 ms.
Created string of length 488895 in 5130 ms.
```
 
 类似的，如果我们通过 `-XX:-CompactStrings` 参数禁用 *Compact Strings* 得出如下输出：  

```
Generated 10000000 strings in 936 ms.
Created string of length 488895 in 9727 ms.
``` 

显然，此项测试非常浅陋，并没有很高的代表性。它只能反映出，在某些场景下采用这种新技术确实能改善程序的性能。
 
## 总结  
  
在此教程中，我们看到了 JDK 9 在 JVM 层面上，通过更有效的字符串存储方式来达到优化性能及内存占用的尝试。像以往一样，所有的代码可以在[Github](https://github.com/eugenp/tutorials/tree/master/core-java-9)获取到。  
