---
layout: post
title: 关于 synchronized 锁优化
categories: Java
excerpt: HotSpot JVM 针对 synchronized 的隐式锁的优化，涉及偏向锁、轻量级锁、重量级锁。
image: https://en.wikipedia.org/wiki/File:Java_programming_language_logo.svg
description: HotSpot JVM 针对 synchronized 的隐式锁的优化，涉及偏向锁、轻量级锁（瘦锁）、重量级锁（胖锁）。
keywords: Java HotSpot JVM synchronized lock biasLock thinLock fatLock 偏向锁 轻量级锁 重量级锁 锁升级 锁降级
licences: cc
---

<br/>
> &emsp;&emsp;众所周知，让开发者简单轻松的编写保证线程安全的代码，一直是现代编程语言所最求的，Java 也不例外。Java 语言引入的 **synchronized** 关键字，无不彰显它在此方面的勃勃雄心。但理想丰满现实骨感，早期的 Java 版本里，对于此关键字的实现太过厚重，导致线程同步的性能远不如预期。Java HotSpot™ VM 经过多个版本的迭代，利用锁膨胀思想，尽量延迟使用重量级锁的手段来提升 **synchronized** 原语的性能。

## 对象存储

### 对象结构

* 对象
	
	|对象在内存中的结构 （64位 JVM）|
	|  :---: |
	| 对象头 object header |
	| 成员变量 object field  | 
	| 对齐/填充（可选） alignment/padding gap (optional)|
	
* 对象数组

	|对象数组在内存中的结构 （64位 JVM）|
	|  :---: |
	| 对象头 object header |
	| 数组元素字节序列 elements bytes  | 
	| 对齐/填充（可选） alignment/padding gap (optional)|

&emsp;&emsp;由于对象在内存以 **8 字节** 为最小单位，单个对象占用内存字节不是 8 的倍数时，需要增加留空字节凑成倍数，此时留空的字节被称为 **对象对齐（object alignment）**。单个对象内部的成员变量所占字节不满 4 的倍数时，也需增加留空字节凑成倍数，此时的留空字节被称为 **填充间隙（padding gap）**。

### 对象头结构

* 对象

	|*对象的 Object Hearder 结构 （64位 JVM）*|
	|  :---: |
	| Mark Word （64 位）|
	| Klass Word（压缩 32位，不压缩 64位）[^1] | 
	| 对齐/填充（可选） alignment/padding gap (optional)|

* 对象数组

	|*对象数组的 Object Hearder 结构 （64位 JVM）*|
	|  :---: |
	| Mark Word （64 位）|
	| Klass Word（压缩 32 位，不压缩 64 位）| 
	| 数组长度（32 位）|
	| 对齐/填充（可选） alignment/padding gap (optional)|
	
### Mark Word 结构

&emsp;&emsp;Mark Word，为运行时对象的标记字，字在 32 位系统中占用 32 bit，64 位操作系统占用 64 bit。其记录着对象运行时的数据，包括 *identity_hashcode*、*GC 分代年龄*、*锁状态* 等信息。以下引用自 JDK8 HotSpot 源码 [markOop.hpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/markOop.hpp) 中，关于 Mark Word 结构信息的描述：

```java
//  32 bits:
//  --------
//  hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//  size:32 ------------------------------------------>| (CMS free block)
//  PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
```

&emsp;&emsp;为了节省内存空间，64 位的 JVM 采取压缩普通对象指针 (COOPs，即 Compressed Ordinary Object Pointer) [^2] 技术，把对象指针由原来的 64 bit 压缩成 32 bit，进而省了一半的内存空间。

&emsp;&emsp;我们着重关注 64 位 JVM 的锁的几种状态，也就是通过上面的 `biased_lock`、`lock` 两个标志位的排列组合得到：

| biased_lock | lock | 状态 |
| :---: | :---: | :---: |
| 0 | 01 | 无锁 normal object |
| 1 | 01 | 偏向锁 biased object |
| 0 | 00 | 轻量级锁 lightweight/thin object |
| 0 | 10 | 重量级锁 heavyweight/fat object |
| 0 | 11 |  GC 标记 |

## 锁的种类

&emsp;&emsp;根据 Mark Word 中的锁状态，我们分别来介绍下。

### 重量级锁
&emsp;&emsp;利用系统级别的互斥量（mutex）实现同步临界区，由于系统级调用，开销大，故称其位重量级锁。

&emsp;&emsp;在下一节关于锁的状态改变图中，会发现一个 *重量级监视器指针*，由于它覆盖（官方称为 **Displaced**）了原本的 Mark Word，故它所指向的是复杂的数据结构 [*ObjectMonitor*](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/87ee5ee27509/src/share/vm/runtime/objectMonitor.hpp) 就包含有用来存储备份的 Mark Word 信息。除此之外，该数据结构还包含锁的等待列表等信息，详细可参考 [Monitor Object 设计模式](https://reionchan.github.io/2017/02/01/corejava-v1-note-part2/#monitor-object%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F)。

- 优点：持有锁时间长、竞争激烈时，其他等待的线程会让出 CPU 进入等待列表
- 缺点：系统级调用，开销大

### 自旋锁
&emsp;&emsp;利用循环的方式来实现线程等待（忙等），等待期间内不让出 CPU 执行时间、免去系统级线程切换开销。

- 优点：避免线程切换
- 缺点：等待期间占用 CPU 资源
 
### 轻量级锁
&emsp;&emsp;采用原子性的 [CAS](https://reionchan.github.io/2017/02/01/corejava-v1-note-part2/#%E5%8E%9F%E5%AD%90%E6%80%A7) 进行加解锁操作，加锁失败时自旋等待，成功则将 Mark Word 覆盖成指向线程栈中的 **Lock Record** 指针，此记录中同样含有原 Mark Word 记录的备份信息。CAS 调用不需系统级别调用，故称为轻量级锁。

- 优点：无需系统级调用，性能好于重量级锁
- 缺点：每次加解锁都需要一次 CAS 操作，采用自旋忙等，在竞争激烈、持有锁时间长会加重 CPU 负荷，会转到重量级锁。

### 偏向锁
&emsp;&emsp;给当前锁标记所属线程，使得所属线程进入同步临界区不用做任何特殊处理，只是简单的使用 CAS 操作将所属线程 ID 记录到 Mark Word 中，同一线程再次加解锁时无需 CAS 操作。

- 优点：只需初始化时进行一次 CAS 操作，之后加解锁都无需 CAS 操作
- 缺点：只能单线程无竞争时使用，一旦有其他线程就必须撤销转到轻量级。

## 锁转态转移

&emsp;&emsp;在给新建的对象分配内存时，其对象头信息会按照下图所示的进行分配，同时随着线程的竞争发送锁状态的转化：

<p align="center">
	<image src="https://wiki.openjdk.java.net/download/attachments/11829266/Synchronization.gif?version=4&modificationDate=1208918680000&api=v2"></image>
</p>

### 状态转步骤

&emsp;&emsp;具体锁转移的过程如下：

1. 如果偏向锁机制是启用，那么新建的对象被初始化成匿名的偏向锁。之所以是匿名，是因为此时并没有具体偏向的线程 ID。

	* 使用虚拟机参数 `-XX:-UseBiasedLocking` 可以关闭偏向锁，默认是启用的。
	* 由于虚拟机参数 `-XX:BiasedLockingStartupDelay` 默认 4 秒，偏向锁机制会延迟 4 秒生效，测试时可以将其设置为 0 秒，防止偏向锁设置不生效。
	* 调用对象默认的 `hashCode()`、 `System.identityHashCode(obj)` 方法会使偏向锁膨胀为轻量级锁 [^3]。原因为生成的 `identity_hashcode` 需要被记录到 Mark Word 中，而偏向锁结构没有设计存储备份 Mark Word 的指针，类似轻量级锁的 `lock record pointer`、重量级锁的 `heavyweight monitor pointer`。

2. 当有单个线程尝试获取该对象锁时，将把此线程的 ID 写入 Mark Word，完成加锁操作。
3. 当此单线程解锁时，一直不存在其他线程来竞争时，此时重偏向 (rebias) 匿名偏向锁，即无线程 ID 的偏向锁。
4. 一旦有其他线程时（**不管有无竞争**），就撤销偏向 (revoke bias) 转为轻量级锁。
	- 其他线程 CAS 操作将自己的线程号 ID 放入 Mark Word 会失败，此线程会执行撤销偏向，在原偏向锁持有线程到达安全点后暂停原线程，检查原线程是否还持有锁。如果已释放锁，将锁状态修改为普通无锁状态；如果未释放锁，拷贝 Mark Word 到原偏向锁线程的锁记录中，修改锁状态标志位为轻量级，把指向原偏向锁线程的锁记录的指针存入 Mark Word 中，唤醒原持有偏向锁线程。
	- 原持有偏向锁线程继续从安全点之后运行，解锁时判断对象头的锁记录指针是否指向当前线程锁记录、且锁记录的备份 Mark Word 与现有对象头里的 Mark Word 一致，如果都一致说明没有其他线程等待此锁，如果不一致说明有其他线程等待，释放锁之后需要换线挂起的那些线程。
5. 轻量级锁状态时，如果竞争激烈（等待线程多）、原线程持锁时间长（即其他线程自旋次数多、等待时间长）就会膨胀为重量级锁。
6. 轻量级、重量级锁解锁后转为普通无锁状态，即后三位为 `001`。
7. 当然，在重量级锁状态时，如果竞争转为不激烈时，锁会降级为轻量级状态。

### 状态转移验证源代码

#### 代码依赖

```xml
<dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jol</groupId>
            <artifactId>jol-cli</artifactId>
            <version>0.10</version>
        </dependency>
</dependencies>
```
#### 代码结构

```
project
└── src
│   └── test
│   │   └── java
│   │   │   └── org
│   │   │   │   └── reion
│   │   │   │   │   └── LockTest.java
│   └── main
│   │   └── resources
│   │   └── java
│   │   │   └── org
│   │   │   │   └── reion
│   │   │   │   │   └── Student.java
```

#### 代码清单

* Student.java

	```java
	package org.reion;
	
	/**
	 * 学生
	 */
	public class Student {
	    // 名
	    String firstName;
	    // 姓
	    String lastName;
	}
	```
* LockTest.java

	```java
	package org.reion;
	
	import org.junit.Before;
	import org.junit.Test;
	import java.util.Random;
	import org.openjdk.jol.info.ClassLayout;
	
	import static java.lang.System.out;
	
	public class OopTest {
	
	    public static final String LINE_SEPARATOR = "\n<<============= %s ==============>>";
	
	    Student stu1;
	
	    @Before
	    public void before() {
	        stu1 = new Student();
	    }
	    
	    // 具体测试方法在下面单个列出，此处省略
	}
	```
	
#### 偏向锁

* 测试方法

	```java
	
	   /**
	    * 该方法验证对象初始化后，由单个线程持有时的偏向锁形态。
	    */
	    @Test
	    public void testBiasedLock() throws InterruptedException {
	        // 确保代码运行时，已经启用偏向锁机制，有两种方式：
	        // ① 偏向锁机制默认延迟 4 秒启动，故休眠 5 秒
	        Thread.sleep(5000);
	        // stu1 之所以重新赋值，因为 before 里旧对象由于偏向锁延迟启动机制，生成不可偏向的对象
	        stu1 = new Student();
	        // ② 添加虚拟机参数，把默认延迟修改为 0 秒，虚拟机启动立马启用偏向锁机制
	        // -XX:BiasedLockingStartupDelay=0
	
	        //【未加锁，可偏向】
	        out.println(String.format(LINE_SEPARATOR, "未加锁，可偏向"));
	        out.println(ClassLayout.parseInstance(stu1).toPrintable());
	
	        //【偏向锁】
	        synchronized (stu1) {
	            out.println(String.format(LINE_SEPARATOR, "偏向锁"));
	            out.println(ClassLayout.parseInstance(stu1).toPrintable());
	        }
	
	        //【未加锁，可偏向】
	        out.println(String.format(LINE_SEPARATOR, "未加锁，可偏向"));
	        out.println(ClassLayout.parseInstance(stu1).toPrintable());
	    }
	```
* 输出结果

	```
	<<============= 未加锁，可偏向 ==============>>
	# WARNING: Unable to attach Serviceability Agent. You can try again with escalated privileges. Two options: a) use -Djol.tryWithSudo=true to try with sudo; b) echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
	      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= 偏向锁 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           05 a8 00 b9 (00000101 10101000 00000000 10111001) (-1191139323)
	      4     4                    (object header)                           e1 7f 00 00 (11100001 01111111 00000000 00000000) (32737)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= 未加锁，可偏向 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           05 a8 00 b9 (00000101 10101000 00000000 10111001) (-1191139323)
	      4     4                    (object header)                           e1 7f 00 00 (11100001 01111111 00000000 00000000) (32737)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	```

**注意：** 由于 JVM 为 [*小端法（Little Endian）*](https://reionchan.github.io/2015/11/25/about-unicode/#unicode-%E7%BC%96%E7%A0%81%E6%A0%BC%E5%BC%8F%E6%A3%80%E6%B5%8B) 故低字节在前，高字节在后。
	
	因此：
	    
	    打印结果中 object header 中 64 位 Mark Word 字节序列应该为 
	    00 00 7f e1 b9 00 a8 05

#### 轻量级锁

* 测试方法

	```java
	/**
     * 该方法验证调用默认 hashCode 或 System.identityHashCode 时，变为轻量级锁的形态
     */
    @Test
    public void testThinLock() {
        // 调用 System.identityHashCode(stu1) 使对象 stu1 禁用偏向锁
        // TIPS：
        //      让对象能够利用上偏向锁机制，应该重载对象的 hashCode 方法，
        //      避免 identityHashCode 方法调用，直接膨胀为轻量级锁
        out.println(String.format(LINE_SEPARATOR, "System.identityHashCode 方法调用"));
        out.println(Integer.toHexString(stu1.hashCode()));

        //【未加锁，不可偏向】
        out.println(String.format(LINE_SEPARATOR, "未加锁，不可偏向"));
        out.println(ClassLayout.parseInstance(stu1).toPrintable());

        //【轻量级锁】
        synchronized (stu1) {
            out.println(String.format(LINE_SEPARATOR, "轻量锁"));
            out.println(ClassLayout.parseInstance(stu1).toPrintable());
        }

        //【未加锁，不可偏向】
        out.println(String.format(LINE_SEPARATOR, "未加锁，不可偏向"));
        out.println(ClassLayout.parseInstance(stu1).toPrintable());
    }
	```
* 输出结果

	```
	<<============= System.identityHashCode 方法调用 ==============>>
	6aa8ceb6
	
	<<============= 未加锁，不可偏向 ==============>>
	# WARNING: Unable to attach Serviceability Agent. You can try again with escalated privileges. Two options: a) use -Djol.tryWithSudo=true to try with sudo; b) echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           01 b6 ce a8 (00000001 10110110 11001110 10101000) (-1462847999)
	      4     4                    (object header)                           6a 00 00 00 (01101010 00000000 00000000 00000000) (106)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= 轻量锁 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           88 18 94 01 (10001000 00011000 10010100 00000001) (26482824)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= 未加锁，不可偏向 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           01 b6 ce a8 (00000001 10110110 11001110 10101000) (-1462847999)
	      4     4                    (object header)                           6a 00 00 00 (01101010 00000000 00000000 00000000) (106)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	```

#### 偏向锁 -> 轻量级锁

* 测试方法

	```java
	/**
     * 该方法验证第二个线程出现时（此时第二线程已经在第一个线程解锁后才执行，故无竞争），会撤销偏向，变为轻量级锁的形态。
     */
    @Test
    public void testBias2ThinLock() throws InterruptedException {
        // 调用偏向锁方法，生成可偏向的 stu1 对象
        testBiasedLock();

        // 生成新线程，申请对象 stu1 锁
        out.println("当前线程：" + Thread.currentThread());
        Runnable runnable = () -> {
            out.println("\n进入线程：" + Thread.currentThread());
            out.println(String.format(LINE_SEPARATOR, "未请求锁，可偏向"));
            out.println(ClassLayout.parseInstance(stu1).toPrintable());

            //【偏向锁 -膨胀-> 轻量级锁】
            synchronized (stu1) {
                out.println(String.format(LINE_SEPARATOR, "已加锁，膨胀为轻量级锁"));
                out.println(ClassLayout.parseInstance(stu1).toPrintable());
            }

            //【未加锁，不可偏向】
            out.println(String.format(LINE_SEPARATOR, "未加锁，不可偏向"));
            out.println(ClassLayout.parseInstance(stu1).toPrintable());
            out.println("\n退出线程：" + Thread.currentThread());
        };

        Thread thread = new Thread(runnable);
        thread.start();
        thread.join();
    }
	```
* 输出结果

	```
	当前线程：Thread[main,5,main]

	进入线程：Thread[Thread-1,5,main]
	
	<<============= 未请求锁，可偏向 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           05 d0 00 be (00000101 11010000 00000000 10111110) (-1107243003)
	      4     4                    (object header)                           fb 7f 00 00 (11111011 01111111 00000000 00000000) (32763)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= 已加锁，膨胀为轻量级锁 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           08 c9 3e 06 (00001000 11001001 00111110 00000110) (104777992)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= 未加锁，不可偏向 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
	      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	```

#### 轻量级锁维持

* 测试方法

	```java
	/**
     * 该方法验证低竞争度时，轻量级锁能一直维持此状态，从而提高运行效率
     */
    @Test
    public void testLowContentionThinLock() throws InterruptedException {
        // 调用 testBias2ThinLock，使 stu1 经历 偏向锁 -> 轻量锁 过程
        // 最终使 stu1 成为 未加锁不可偏
        testBias2ThinLock();

        Runnable runnable = () -> {
            Thread current = Thread.currentThread();
            out.println("\n进入线程：" + current);

            //【轻量级锁】
            for (int i = 0; i<2; i++) {
                try {
                    // 400 毫秒内的随机休眠时间，模拟低对象锁竞争
                    Thread.sleep(new Random().nextInt(400));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (stu1) {
                    out.println(String.format(LINE_SEPARATOR, current + " ROUND-" + i) + "\n" + ClassLayout.parseInstance(stu1).toPrintable());
                }
            }
            out.println("\n退出线程：" + Thread.currentThread());
        };

        Thread t1 = new Thread(runnable);
        Thread t2 = new Thread(runnable);
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        // 结束竞争后对象状态
        out.println(String.format(LINE_SEPARATOR, Thread.currentThread()) + "\n" + ClassLayout.parseInstance(stu1).toPrintable());
    }
	```
	
* 输出结果

	```
	进入线程：Thread[Thread-2,5,main]
	
	进入线程：Thread[Thread-3,5,main]
	
	<<============= Thread[Thread-3,5,main] ROUND-0 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           f8 98 1e 11 (11111000 10011000 00011110 00010001) (287217912)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           83 02 01 f8 (10000011 00000010 00000001 11111000) (-134151549)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-3,5,main] ROUND-1 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           f8 98 1e 11 (11111000 10011000 00011110 00010001) (287217912)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           83 02 01 f8 (10000011 00000010 00000001 11111000) (-134151549)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	退出线程：Thread[Thread-3,5,main]
	
	<<============= Thread[Thread-2,5,main] ROUND-0 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           f8 68 0e 11 (11111000 01101000 00001110 00010001) (286157048)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           83 02 01 f8 (10000011 00000010 00000001 11111000) (-134151549)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-2,5,main] ROUND-1 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           f8 68 0e 11 (11111000 01101000 00001110 00010001) (286157048)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           83 02 01 f8 (10000011 00000010 00000001 11111000) (-134151549)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	退出线程：Thread[Thread-2,5,main]
	
	<<============= Thread[main,5,main] ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
	      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
	      8     4                    (object header)                           83 02 01 f8 (10000011 00000010 00000001 11111000) (-134151549)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

	```

#### 轻量级锁 -> 重量级锁 (高频型)

* 测试方法

	```java
	/**
     * 该方法验证竞争变激烈（请求锁频率快）后，锁由轻量级膨胀为重量级的过程。
     */
    @Test
    public void testHighContentionThin2FatLock() throws InterruptedException {
        // 调用 testBias2ThinLock，使 stu1 经历 偏向锁 -> 轻量锁 过程
        // 最终使 stu1 成为 未加锁不可偏
        testBias2ThinLock();

        Runnable runnable = () -> {
            Thread current = Thread.currentThread();
            out.println("\n进入线程：" + current);

            //【轻量级锁】
            for (int i = 0; i < 2; i++) {
                try {
                    // 5 毫秒内的随机休眠时间，模拟高对象锁竞争
                    Thread.sleep(new Random().nextInt(5));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (stu1) {
                    out.println(String.format(LINE_SEPARATOR, current + " ROUND-" + i) + "\n" + ClassLayout.parseInstance(stu1).toPrintable());
                }
            }
            out.println("\n退出线程：" + Thread.currentThread());
        };

        Thread t1 = new Thread(runnable);
        Thread t2 = new Thread(runnable);
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        // 结束竞争后对象状态
        out.println(String.format(LINE_SEPARATOR, Thread.currentThread()) + "\n" + ClassLayout.parseInstance(stu1).toPrintable());
    }
	```
	
* 输出结果

	```
	<<============= 未加锁，不可偏向 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
	      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	退出线程：Thread[Thread-1,5,main]
	
	进入线程：Thread[Thread-2,5,main]
	
	进入线程：Thread[Thread-3,5,main]
	
	<<============= Thread[Thread-2,5,main] ROUND-0 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           da d3 02 e4 (11011010 11010011 00000010 11100100) (-469576742)
	      4     4                    (object header)                           9a 7f 00 00 (10011010 01111111 00000000 00000000) (32666)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-3,5,main] ROUND-0 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           da d3 02 e4 (11011010 11010011 00000010 11100100) (-469576742)
	      4     4                    (object header)                           9a 7f 00 00 (10011010 01111111 00000000 00000000) (32666)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-2,5,main] ROUND-1 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           da d3 02 e4 (11011010 11010011 00000010 11100100) (-469576742)
	      4     4                    (object header)                           9a 7f 00 00 (10011010 01111111 00000000 00000000) (32666)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	退出线程：Thread[Thread-2,5,main]
	
	<<============= Thread[Thread-3,5,main] ROUND-1 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           da d3 02 e4 (11011010 11010011 00000010 11100100) (-469576742)
	      4     4                    (object header)                           9a 7f 00 00 (10011010 01111111 00000000 00000000) (32666)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	退出线程：Thread[Thread-3,5,main]
	
	<<============= Thread[main,5,main] ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           da d3 02 e4 (11011010 11010011 00000010 11100100) (-469576742)
	      4     4                    (object header)                           9a 7f 00 00 (10011010 01111111 00000000 00000000) (32666)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	```

#### 轻量级锁 -> 重量级锁 (长时型)

* 测试方法

	```java
	/**
     * 该方法验证持锁时间长（等待线程自旋过多），锁由轻量级膨胀为重量级的过程。
     */
    @Test
    public void testLongContentionThin2FatLock() throws InterruptedException {
        // 调用 testBias2ThinLock，使 stu1 经历 偏向锁 -> 轻量锁 过程
        // 最终使 stu1 成为 未加锁不可偏
        testBias2ThinLock();

        // 长期持有对象锁，在另一个线程开始竞争时，观察锁膨胀过程
        Runnable runnable = () -> {
            Thread current = Thread.currentThread();
            out.println("\n进入线程：" + current);

            //【轻量级锁】
            synchronized (stu1) {
                int counter = 0;
                while (counter < 5) {
                    counter++;
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    out.println(String.format(LINE_SEPARATOR, current + " 锁内") + "\n" + ClassLayout.parseInstance(stu1).toPrintable());
                }
            }

            out.println("\n退出线程：" + current);
        };

        // 睡眠 2.5 秒后，开始竞争锁
        Runnable runnable2 = () -> {
            Thread current = Thread.currentThread();
            out.println("\n进入线程：" + current);

            //【轻量级锁】
            try {
                out.println(String.format(LINE_SEPARATOR, current + "睡眠...") + "\n\n");
                Thread.sleep(2500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            // 2.5 秒后参与竞争
            out.println(String.format(LINE_SEPARATOR, current + "参与竞争...") + "\n\n");
            synchronized (stu1) {
                out.println(String.format(LINE_SEPARATOR, current) + "\n" + ClassLayout.parseInstance(stu1).toPrintable());
            }
        };

        Thread t1 = new Thread(runnable);
        Thread t2 = new Thread(runnable2);
        t1.start();
        t2.start();
        t1.join();
        t2.join();

    }
	```
	
* 输出结果

	```
	进入线程：Thread[Thread-2,5,main]

	进入线程：Thread[Thread-3,5,main]
	
	<<============= Thread[Thread-3,5,main]睡眠... ==============>>
	
	
	
	<<============= Thread[Thread-2,5,main] 锁内 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           f0 98 a9 0c (11110000 10011000 10101001 00001100) (212441328)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-2,5,main] 锁内 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           f0 98 a9 0c (11110000 10011000 10101001 00001100) (212441328)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-3,5,main]参与竞争... ==============>>
	
	
	
	<<============= Thread[Thread-2,5,main] 锁内 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           8a 3c 01 5d (10001010 00111100 00000001 01011101) (1560362122)
	      4     4                    (object header)                           d2 7f 00 00 (11010010 01111111 00000000 00000000) (32722)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-2,5,main] 锁内 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           8a 3c 01 5d (10001010 00111100 00000001 01011101) (1560362122)
	      4     4                    (object header)                           d2 7f 00 00 (11010010 01111111 00000000 00000000) (32722)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-2,5,main] 锁内 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           8a 3c 01 5d (10001010 00111100 00000001 01011101) (1560362122)
	      4     4                    (object header)                           d2 7f 00 00 (11010010 01111111 00000000 00000000) (32722)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	退出线程：Thread[Thread-2,5,main]
	
	<<============= Thread[Thread-3,5,main] ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           8a 3c 01 5d (10001010 00111100 00000001 01011101) (1560362122)
	      4     4                    (object header)                           d2 7f 00 00 (11010010 01111111 00000000 00000000) (32722)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	```

#### 轻量级锁 -> 重量级锁 -> 轻量级锁

* 测试方法

	```java
	/**
     * 该方法验证竞争由强转弱后，锁由轻量级膨胀为重量级后又降级为轻量级的过程。
     */
    @Test
    public void testHighContentionThin2FatThenDeflate2ThinLock() throws InterruptedException {
        // 调用 testBias2ThinLock，使 stu1 经历 偏向锁 -> 轻量锁 过程
        // 最终使 stu1 成为 未加锁不可偏
        testBias2ThinLock();

        Runnable runnable = () -> {
            Thread current = Thread.currentThread();
            out.println("\n进入线程：" + current);

            // 随机产生 10 之内的循环追加数，模拟双线程中，一个线程完成后，剩下的线程单独执行时锁降级
            int maxRound = new Random().nextInt(10);
            //【轻量级锁】
            for (int i = 0; i < 5 + maxRound; i++) {
                try {
                    // 5 毫秒内的随机休眠时间，模拟高对象锁竞争
                    Thread.sleep(new Random().nextInt(5));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (stu1) {
                    out.println(String.format(LINE_SEPARATOR, current + " ROUND-" + i) + "\n" + ClassLayout.parseInstance(stu1).toPrintable());
                }
                out.println(String.format(LINE_SEPARATOR, current + " ROUND-" + i) + "\n" + ClassLayout.parseInstance(stu1).toPrintable());
            }
            out.println("\n退出线程：" + Thread.currentThread());
        };

        Thread t1 = new Thread(runnable);
        Thread t2 = new Thread(runnable);
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        // 结束竞争后对象状态
        out.println(String.format(LINE_SEPARATOR, Thread.currentThread()) + "\n" + ClassLayout.parseInstance(stu1).toPrintable());
    }
	```
	
* 输出结果

	```
	进入线程：Thread[Thread-2,5,main]
	
	进入线程：Thread[Thread-3,5,main]
	
	<<============= Thread[Thread-2,5,main] ROUND-0 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           f0 a8 25 0f (11110000 10101000 00100101 00001111) (254126320)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-2,5,main] ROUND-0 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
	      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-3,5,main] ROUND-0 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           f0 d8 35 0f (11110000 11011000 00110101 00001111) (255187184)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	<<============= Thread[Thread-3,5,main] ROUND-1 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           5a d1 82 44 (01011010 11010001 10000010 01000100) (1149423962)
	      4     4                    (object header)                           dd 7f 00 00 (11011101 01111111 00000000 00000000) (32733)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-2,5,main] ROUND-1 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           5a d1 82 44 (01011010 11010001 10000010 01000100) (1149423962)
	      4     4                    (object header)                           dd 7f 00 00 (11011101 01111111 00000000 00000000) (32733)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	【【【【【【【【中间打印省略】】】】】】】】】	
	退出线程：Thread[Thread-3,5,main]
	
	<<============= Thread[Thread-2,5,main] ROUND-8 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           f0 a8 25 0f (11110000 10101000 00100101 00001111) (254126320)
	      4     4                    (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	
	<<============= Thread[Thread-2,5,main] ROUND-8 ==============>>
	org.reion.Student object internals:
	 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
	      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
	      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
	      8     4                    (object header)                           b4 02 01 f8 (10110100 00000010 00000001 11111000) (-134151500)
	     12     4   java.lang.String Student.firstName                         null
	     16     4   java.lang.String Student.lastName                          null
	     20     4                    (loss due to the next object alignment)
	Instance size: 24 bytes
	Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
	
	退出线程：Thread[Thread-2,5,main]
	```

## 参考资料

* [Synchronization](https://wiki.openjdk.java.net/display/HotSpot/Synchronization) Created by Christian Wimmer, last modified on Apr 29, 2008
* [Compressed Ordinary Object Pointer](https://wiki.openjdk.java.net/display/HotSpot/CompressedOops)
* [markOop.hpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/markOop.hpp)
* [What is in java object header](https://stackoverflow.com/questions/26357186/what-is-in-java-object-header)

## 脚注

[^1]: **压缩的类指针 (Compressed Class Pointer)**，使用虚拟机参数 `-XX:-UseCompressedClassPointers` 关闭压缩，默认开启。
[^2]: [**压缩的普通对象指针 (Compressed Ordinary Object Pointer)**](https://wiki.openjdk.java.net/display/HotSpot/CompressedOops)，使用虚拟机参数 `-XX:-UseCompressedOops` 关闭压缩，默认开启。
[^3]: [How does the default hashCode() work?](https://srvaroa.github.io/jvm/java/openjdk/biased-locking/2017/01/30/hashCode.html)