---
layout: post
title: 《 Java 核心技术 卷一》笔记 上篇
categories: Book Java
tags: Core Java 核心技术 卷Ⅰ
excerpt: 读《 Java 核心技术 卷一》的笔记
image: https://images-cn.ssl-images-amazon.com/images/I/41OFsAz290L._SY498_BO1,204,203,200_.jpg
description: 读《 Java 核心技术 卷一》的笔记  
keywords: Java, Core Java, 核心技术, 卷 Ⅰ
licences: cc
---

## 第一章 Java程序设计概述

1. 1996年 Java第一次发布
2. [Java白皮书](http://www.oracle.com/technetwork/java/langenv-140151.html)
	* `简单性`、`面向对象`、`分布式`、`健壮性`、`安全性`
	* `体系结构中立`、`可移植性`、`解释型`、`高性能`、`多线程`、`动态性`
4. Java语言发展状况

	版本	| 年份	| 语言特性						| 类、接口数量
	--:	| :--	| :-----------					| :--:
	1.0	| 1996	| 语言本身						| 211
	1.1	| 1997	| 内部类							| 477
	1.2	| 1998	| strictfp修饰符					| 1524
	1.3	| 2000	| ——							| 1840
	1.4	| 2002	| 断言							| 2723
	5.0	| 2004	| 泛型类、“for each”、可变元参数、自动装拆箱、元数据、枚举、静态导入| 3279
	6	| 2006	| ——							| 3793
	7	| 2011	| 字符串switch、钻石操作符、二进制字面量、异常处理改进| 4024
	8	| 2014	| lambda表达式、包含默认方法的接口、流和日期时间库| 4240

## 第二章 Java程序设计环境

1. JDK 下载、环境变量设置、源文件及文档
2. 命令行使用
3. 集成开发环境IDE：NetBeans、Eclipse……
4. 运行图形化程序
5. 运行Applet程序：jar、appletviewer

## 第三章 Java的基本程序设计结构

### 数据类型

* 8种基本类型
	- 整形 byte、short、int、long

		字面量表示：
		
		类型			| 字面量表示			| 备注
		-----		| -----------		| ----
		long		| long l = 400L		| l或L
		int	十进制	| int dec = 8		|
		int 八进制	| int oct = 010		|
		int 十六进制	| int hex = 0xCAFE	|
		int 二进制	| int bi = 0B101	| b或B，0b0110_0100 **Java7及之后支持**
				
	- 浮点型 float、double
		
		类型		| 字面量表示			| 备注
		-----	| -----------		| ----
		float	| float f = 3.14F	| f或F，有效位8位
		double	| double d = 3.14D	| d或D，不加后缀默认double，有效位16位
		
		特殊三个浮点值：  

		`POSITIVE_INFINITY` 正无穷  （正数除0）

		`NEGATIVE_INFINITY` 负无穷  （负数除0）

		`NaN` 不是数字 （0/0 或 负数的平方根）  
		  
		**浮点数计算细节：**  
		> double w = x * y / z  
		  
		Intel处理器：  

		> 计算 “x * y”，结果扩展存入80位寄存器，再除 “z” 后截断为64位的结果  
			  
		好处：更精确的结果、避免指数溢出  
		坏处：可能与始终在64位上计算的结果不一样
		 
		Java处理演进：  

		最初规范：  
		> JVM将所有中间计算都进行截断，但遭到计算团体的反对。  
		为了平衡 “最优性能” 和 “理想结果”，引入 “striptfp” 关键字，  
		即 Strict Float Point  
    
		默认情况：  
		> 计算中间结果使用扩展精度，指数与尾数都扩展取决于硬件
		Intel芯片扩展指数，截断尾数（截断不损性能）
		    	  
		加修饰符：  
		> 采用严格64位运算，中间结果指数尾数都截断为64位
		这样，使多次计算结果相同
		
		说道浮点数的精度问题，知乎上有个有趣的问题：  

		[Java float浮点数精度问题](https://www.zhihu.com/question/40994059)
		
		问题描述大致如下：  
			Java中float类型不是说精度为小数点后7位么，但是如下打印如何解释			  
		```java
		
		//----------代 码 片 段----------
		float a=1.23456789123456789f;
		float b=2.23456789123456789f;
		float c=3.23456789123456789f;
		float d=4.23456789123456789f;
		float e=5.23456789123456789f;
		float f=6.23456789123456789f;
		float g=7.23456789123456789f;
		float h=8.23456789123456789f;
		float i=9.23456789123456789f;
		System.out.println(a);
		System.out.println(b);
		System.out.println(c);
		System.out.println(d);
		System.out.println(e);
		System.out.println(f);
		System.out.println(g);
		System.out.println(h);
		System.out.println(i);
		//----------代 码 片 段----------
		
		//----------打 印 结 果----------
		1.2345679
		2.2345679
		3.2345679
		4.234568
		5.234568
		6.234568
		7.234568
		8.234568
		9.234568
		```  
  
		要回答此问题，得了解如下的背景知识：

		1. 任意两个实数之间是有无穷多个实数，故有实数能把数轴填满一说  
		  而浮点数（双精度类似）则不同，两浮点数之间只包含有限个浮点数  
		  换句话说，某个浮点数是存在确定的前序和后序的浮点数的  

		2. 相邻浮点数之间存在距离，被称为最小精度单位（Unit of Least Precision）  
		  或最后位置单位（Unit in the Last Place），简称ULP  
		  
		3. Java对于任意浮点字面量，最终都舍入到所能表示的最靠近的浮点值  
		  遇到该值离左右两个能表示的浮点值距离相等时，采取[IEEE_754](https://en.wikipedia.org/wiki/IEEE_floating_point#Rounding_rules)舍入原则  

			> 在左右距离相同时采用偶数优先（Ties To Even，默认）即取其中的偶数（在二进制中以0结尾的）
		  
		4. Java获取浮点数的前序或后序浮点数、ULP的方法为Math类的静态方法：  
			
			```java
			public static float nextDown(float f) // 前序，即比所给值小
			public static float nextUp(float f)   // 后序，即比所给值大
			public static float ulp(float f)      // 获取ULP
			```  

		了解这些知识后，我们来举个“栗子”：
		
		```java
		float e = 1.1110999f;		
		System.out.println("1.1110999f 打印真实为：" + e);
		System.out.println("1.1110999f 的前序：" + Math.nextDown(e));
		System.out.println("1.1110999f 的后序：" + Math.nextUp(e));
		System.out.println("1.1110999f 的最小精度单位：" + Math.ulp(1.1110999f));
	
		// e会等于d吗？
		float d = 1.1111f;
		System.out.println(e == d);

		// 打印结果：
		1.1110999f 打印真实为：1.1111           //  ①
		1.1110999f 的前序：1.1110998           //  ②
		1.1110999f 的后序：1.1111001           //  ③
		1.1110999f 的最小精度单位：1.1920929E-7 //  ④
		true                                 //  ⑤
		```
		①⑤可以验证背景知识第3条，字面量的确舍入到最近所能表示的浮点数‘1.1111f’
		    至于它为何只是诡异的保留5位（还压根没达到7位或8位）看问题总结
		    
		①②③可以得知在[1.1110998, 1.1111001]范围内其实只存在三个浮点数，即：
           { 1.1110998、      1.1111、      1.1111001 }
           
        ④可以得知，‘1.1110999f’这个区段的ULP为：0.00000011920929
          将上面三个数字任意相邻两个数字相减，确实也能印证，只是精度稍差点  
		

		再回到问题，为啥输出有时保留7位或8位，甚至是像上面例子中的5位‘1.1111’  
		其实就是根据字面量（本例为：1.1110999f），选取两个可表示的浮点值A和B  
		使得A<字面量<B（本例为：A =1.1110998, B=1.1111）然后根据浮点数舍入规则  
		选取其中之一（本例选取B，B二进制尾数为0）至于结果到底保留几位  
		由最终被选择的可表示的浮点数决定，本例由于选择1.1111，故有效位就是5位

		参考资料：
		* [IEEE floating point](https://en.wikipedia.org/wiki/IEEE_floating_point)
		* [Unit in the last place](https://en.wikipedia.org/wiki/Unit_in_the_last_place)  

					
	- 字符型 char  

		占用一个字节，java使用UTF-16字符集，故需两个char  
		码点：一个编码表中某个字符对应的代码值  
			
	- 布尔型 boolean  
		true	真值  
		false	假值  
		**注意：整型与布尔型不可转换**

### 运算符
* 位运算符  
	- & and运算，`同真为真，其余为假`
	- \|  or运算，`同假为假，其余为真`
	- ^ xor运算，`相同为假，相异为真`
	- ~ not运算，`假为真，真为假`
	- << 左移运算，`右边用0补充`
	- \>> 右移运算，`高位用符号位值填充`
	- \>>> 无符号右移，`高位用0填充`

### 字符串
* String类为不可变类，即字符串一旦产生就不能更改
* 字符串不是单纯的char数组，更像是指向char序列的指针
* 字符串常量池
	- 编译时期的字面量、符号引用都纳入class文件中的常量池
	- String str = “Str” + "ing";   
		这种方式编译器自动优化为“String”并纳入常量池
	- 字符串引用之间 “+” 操作会生产新对象，而非纳入常量池(生成StringBuilder对象)
	- String.intern() 方法会判断常量池，没有就讲当前字符串加入常量池  
  		推荐不错的一篇关于Java内存的模型及字符串分配的博文：  
		[Java内存模型及字符串的内存操作](http://www.cnblogs.com/kkcheng/archive/2011/02/25/1964521.html)
  
### 控制流程
* 嵌套的两个语句块中不能有同名变量
* 语句块（复合语句）可以在原本只能放一条语句的地方放多条语句
* 加上 `-Xlint:fallthrough` 编译器参数，可检验switch语句漏加break语句
* Java7及之后case标签开始支持字符串字面量
* 返回一个`数组长度为0的数组` 与 返回`null` 不同

## 第四章 对象与类

### 隐式参数与显式参数  
* 对象的方法第一个参数称为隐式参数，该参数即当前对象this  
	第二个参数开始称为显式参数，即方法名后面括号中的变量

* 方法内联  
	编译器监视那些简洁、经常被调用、没有重载、可优化的方法，  
	将其包含到调用者内部的行为，称作方法内联

* 对象成员变量的封装  
	`需要返回可变对象的引用时，应该对它进行克隆`，否则破坏封装

* 访问权限  
	方法可以调用对象的所有`私有特性`、`隐式参数`

* final 实例域  
	被 final 修饰的实例域，`自动放弃编译器的默认初始化`，必须确保每个  
	构造方法都要手动初始化该域的值，且一旦初始化不可更改

	为什么设计成成员变量声明的同时可以进行赋值（显式初始化），这样有啥好处？

	> 方便多个构造方法初始化时的代码共享，不需要每个构造方法都初始化一次。当然只针对以下几种情况的成员： `final修饰的成员`、`不想被赋予编译器默认值的成员`、`不能确保无论最初调用的构造是哪个但总能被初始化的成员`  
	
### 静态成员与静态方法  
	
* 静态成员表示类的所有对象共享该成员，不属于任何对象，只属于类  

* 静态方法可以认为是没有this参数的方法，故而不允许访问本类的对象的实例成员及方法  

* Java程序设计语言采用按值调用，方法得到的是参数值的一个拷贝  
	- 基本类型就为基本类型值
	- 对象类型就为引用值
  
* `相同方法名`、`不同的参数`便产生了重载  

	注意：返回类型不是方法签名的一部分  
  
### 对象构造
* 无参构造器  
	  
	- 类没有构造器时，编译器默认提供一个无参构造器
	- 无参构造器将所有实例域设置为默认值
	- 提供有参构造器，但没有无参构造器，该类不允许调用无参构造器  
	  
	
	构造器的在编译之后，默认执行的第一个操作是  
	  
	```java
	0: aload_0
	1: invokespecial #1 // Method java/lang/Object."<init>":()V
	```  

   	传入当前对象this，调用**类的特殊初始化方法**，注意\<init\>方法的具体所属类  
	- 如果，this对象所属类extends具体其他类，那么初始化方法所属类是父类  
	- 否则，所属类为java.lang.Object
   		  
	这样做，确保首先初始化继承链的顶层Object类 

* 调用另一个构造器  
	构造器的首行，可以使用this关键字调用另外一个构造器  
	从而达到代码共享的目的
	
* 初始化代码块  

	无论使用哪个构造器，都会首先运行初始化代码块，后运行构造器主体部分  
	
	到此，初始化非静态域成员的顺序如下：
	1. 所有数据初始化为默认值
	2. 按类声明中的次序，依次执行所有初始化语句和初始化块
	3. 如果构造器使用了this()调用其他构造器，则先执行其他构造器
	4. 执行当前构造器的主体  
	  
	初始化静态类成员是在类加载的时候发生，步骤如下：
	1. 遇到的静态域初始化成默认值
	2. 依照代码顺序执行静态初始化语句或静态语句块  

	类内，方法外为何要设计支持初始化代码块，这样有啥好处？
	>  
	有时，初始化某个成员的值，是由多条语句才能完成时、
	并且想要使得多个构造方法共享此初始化的操作时，这样子更简洁（较构造器互调用）
		
	Java7 之前，可以不需要main方法而使用静态代码块的方式，运行一个类程序  
	Java7 及后续，java程序会首先检查是否包含main方法
	
* 对象析构与finalize方法  
	  
	finalize方法在垃圾回收器清理对象的时候被调用（[不保证立即调用，而是加入队列](https://mp.weixin.qq.com/mp/getmasssendmsg?__biz=MzIzNjI1ODc2OA==#wechat_webview_type=1&wechat_redirect)）  
	因此，不要依赖finalize回收短缺的资源，替代方法 `Runtime.addShutdownHook`
	
### 包  
  
* 只能用`*`导入一个包，不能嵌套  
	例如：`import java.*`、`import java.*.*`  

* 包定义不能与java开头，编译能通，但执行会出错（规范禁止使用）  
	JDK 1.2开始，类加载器做了此限制  

* 静态导入静态方法与成员，可以后续调用时不需要加类名前缀  

* 编译器编译源文件不检查包目录结构，运行时必须要与包目录结构一致  

* 包作用域
  
	修饰符		| 作用域
	---			| ---
	public		| 修饰的域或方法可以在其被定义的类、同包类、不同包类可见
	protected	| 修饰的域或方法可以在其被定义的类、同包类、不同包子类可见
	(default)	| 修饰的域或方法仅在其被定义的类、及同包下的类可见
	private		| 修饰的域或方法仅能在其被定义的类中可见
	
	    
	注意：  
	* protected 修饰的域或方法，同包的类中，可以用定义的类的引用直接访问  
	效果同default，但不同包的子类中，不能通过定义类的引用直接访问  
	只能通过 **子类引用** 或 **super关键字** 进行特殊调用
	
	* protected、private 虽然不能修饰外部类，但却**可以修饰内部类**  
	**作用域与修饰域和方法类似，无显式构造方法时，默认构造方法修饰符同类修饰符一致**   

	```java
		
	//---------- SuperClass 代 码 ---------- 
	package obj.protect;

	public class SuperClass {
		protected void getObj() {

		}
	}
	
	//---------- InSubClass 代 码 ---------- 
	package obj.protect;

	public class InSubClass extends SuperClass {
		{
			// ① 继承关系调用父类方法，前提子类没有重写父类方法
			this.getObj();
			
			// ② 通过父类的引用直接调用
			new SuperClass().getObj();
			
			// ③ super关键字调用父类方法，即使子类重写了父类方法
			super.getObj();
			
			// ④ 此种多态形式的调用效果与 ①一致
			SuperClass one = new InSubClass();
			one.getObj();
		}
	}
	
	//---------- OutSubClass 代 码 ----------
	package obj.other;

	import obj.protect.SuperClass;

	public class OutSubClass extends SuperClass {
		{
			// ① 继承关系调用父类方法，前提子类没有重写父类方法 
			this.getObj();
			
			// ② 通过父类的对象是无法访问的，编译错误
			new SuperClass().getObj(); // 不可编译
			
			// ③ super关键字调用父类方法，即使子类重写了父类方法
			super.getObj();
		}
	}
	```
	从字节码指令的角度来解读：  
	①字节码指令为  
	```java  
	aload_0 [this]
	invokevirtual obj.protect.InSubClass.getObj()  
	```
			
	②字节码指令为  
	```java  
	new obj.protect.SuperClass
	dup
	invokespecial obj.protect.SuperClass()
	invokevirtual obj.protect.SuperClass.getObj()  
	```	
	③字节码指令为  
	```java
		aload_0 [this]
		invokespecial obj.protect.SuperClass.getObj()  
	```   
	④字节码指令为  
	```java
		new obj.protect.InSubClass
		dup
		invokespecial obj.protect.InSubClass()
		astore_1 [one]
		aload_1 [one]
		invokevirtual obj.protect.SuperClass.getObj()  
	```
		
	①、②、④ 都使用 invokevirtual 调用getObj()方法，即动态绑定调用。  
	根据所传对象实际的类来调用方法，如果子类重写了父类方法，  
	调用的将是子类的方法，反之调用父类的。  
	像④的最后一行指令，虽然引用是父类但实际的类确实子类，  
	但由于子类没有重写， 故还是调用父类的。
			
	③ 使用 invokespecial 调用getObj()方法，即静态调用  
	根据所传对象的引用类型来调用方法，引用是父类就调用父类的方法
		
	谈到这里有必要理清容易忽视的细节：  
	> 子类能重写的，永远是子类能够访问到的父类的方法
	1. 父类的private方法，任何子类都不能重写
	2. 父类的default方法，同包下子类能重写，不同包子类不能
			
	④的调用明明可以优化为：  
  	  
	```java
	invokespecial obj.protect.SuperClass.getObj()  
	```
	  
	为何却采用invokevirtual调用？  

	虽然子类没有重写父类方法，但还是在运行动态绑定的情况下
	检测出由于子类没复写而调用父类的这一过程
	如果子类一旦复写父类方法，那么编译期间就已经确定是调用
	子类的方法，调用字节码即会被优化为：
	  
	```java  
	invokespecial obj.protect.InSubClass.getObj()  
	```

	说到这，Java的几种方法调用的字节码指令用法，请参考如下资料：  

	[JVM方法调用的那些事](http://www.jianshu.com/p/56a7c4b26b14)  
	[invokevirtual，invokespecial，invokestatic，invokein](http://haiyupeter.iteye.com/blog/385806)
		  
### 类路径  

* 一个源文件只能包含一个公有类，且文件名必须与公有类相同  
这样规定方便编译器查找包外引用的类，因为仅能引用其它包的公有类，  
并且公有类刚好与文件名相同，当然包内引用在其它类能定义的类还是需要
搜索当前包的所有源文件  

*  运行时库会自动加入类搜索路径中  
%JAVA_HOME%/jre/lib  
%JAVA_HOME%/jre/lib/ext
		
### 类的设计技巧
*  保证数据私有
*  保证数据初始化
*  不要过多使用基本类型
*  不是所有域都需要访问器和修改器
*  将职责过多的类进行分解
*  类名方法名要体现它的职责
*  优先使用不可变类 
  
## 第五章 继承

### 类、超类和子类
	
* 子类通过`super`关键字调用父类方法  
	super 与 this 不同，它不是对象引用，只是告知编译器调用父类方法的一个标示  

* 子类构造器没显示调用父类构造器时，默认调用父类的无参构造器  
	父类没有无参构造器则编译报告异常  

* 一个对象变量只是多种实际类型的现象叫做`多态`  
	运行时自动选择调用哪个方法的现象称为`动态绑定`  
	从某个类到其祖先的路径被称为该类的`继承链`
	
* final关键字修饰类或方法会阻止其继承  
	早期使用 final ，主动让方法不能被重写好让编译器对它进行优化内联  
	防止 CPU 使用分支转移影响指令预取策略  
	现在更为高级的虚拟机采用即时编译器（ JIT ）处理，它能准确的知道  
	类之间的继承关系，并知道方法是否真正地存在覆盖的方法，必要时会  
	进行内联处理，将来发生变化方法被重写了，还能取消内联  

### 所有类的超类 Object  
	  
* equals方法最佳实践
	1. 显式命名参数为otherObject，稍后将它转换成other变量
	2. 检测this与otherObject是否是同一引用
	3. 检测otherObject是否为null，是直接返回
	4. 如果相等概念由具体子类决定（子类间非跨代比较，保证对称性）  
		使用下面语句：  
		```java  
		getClass() != otherObject.getClass()  
		```  
	5. 如果相等概念由父类决定（子类间、跨代都有相同语义）  
		使用下面语句：  
		```java  
		otherObject instanceof ClassName
		```
	6. 将otherObject转换为相应类的类型变量  
		```java  
		ClassName object = (ClassName) otherObject  
		```  
	7. 现在开始对需要的域进行比较
	8. 如果子类重新定义了equals，就要包含`super.equals(other)`  
  
* hashCode方法
	  
	> 如果重新定义equals方法，就必须重新定义hashCode方法  
	  
	关于hashCode值的算法，参考[《Effective Java》第二版第9条](#)  
  
* toString方法  

	默认格式 `对象所属类目@散列码`，例如：`java.io.PrintStream@2f6684`  
	数组格式 `[X@散列码` X为JLS规定的类型字母，例如：`[I@1a46e30`
	
	数组默认toString不直观，可以改用静态方法：  
	```java  
  	Arrays.toString();
	```
	多维数组可以采用：  
	```java  
  	Arrays.deepToString();
	```

### 对象包装器与自动装箱  
	  
* 小心包装器类的`==`操作
	  
	```java
	Integer a = 100;
	Integer b = 100;
	if (a == b) ...
	```
	上面会执行到if语句里去吗？  
	答案是不确定的，需看具体给的值落在哪个范围  
	自动装箱为了提高效率，规定：  

	- boolean、byte、char ≤ 127
	- short、int介于 -128 ~ 127之间  
	  
	被包装到固定的对象中，即缓存了以上的对象用于共享  
	所以，如果上面的值在以上之间，就会执行 if 语句，否则不执行
	    
* 一个条件表达式中混用Integer和Long、Float或Double类型  
  	  
	**Integer会拆箱后提升为float或double** 
  
	注意：书中原文有这么一段表述：	  
  
	> 如果在一个条件表达式中混合使用 Integer 和 Double 类型，  
	Integer 值就会拆箱，提升为 double ，再装箱为 Double 。  
  	  
	其实作者所述**再装箱为 Double **想表达的意思是拆箱后提升为 double 值，  
	与原 double 计算后的结果**可能**会再次封装为 Double （注意这里是说可能）   
	  
	拿书中的例子来说：  

	```java
	Integer n = 1;
	Double x = 2.0;
	System.out.println(true ? n : x);	// Prints 1.0
	```
	打印的是 1.0 没错，但这是调用 println 的基本类型 double 的重载方法，  
	故而没有再封装为 Double 的操作，字节码可以看出这点：
	  
	```java
	invokevirtual #23   // Method java/io/PrintStream.println:(D)V  
	```
	相反，如果是执行如下语句：
	
	```java
	// 受后面x.toString的对象调用影响  
	System.out.println(true ? n+x : x.toString())
	```
	打印的是 3.0 ，而且是调用 println 的 Object 类型的重载方法，  
	受 `toString` 方法影响，结果 double 被再封装为 Double 对象
	
	```java
	dadd
	invokestatic  #21   // Method java/lang/Double.valueOf:(D)Ljava/lang/Double;
	invokevirtual #24   // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
	```
	另外，不光是 Double 类， Integer 混合 Long 、Float 类只要参与比它精度高的计算过程，  
	都会有拆箱晋升的过程，但是如果没有参与比它精度高的计算过程，就不会拆箱晋升  
	例如下面语句，就没有拆箱晋升的过程：  
	  
	```java  
	System.out.println(true ? n.toString() : x);
	```  

### 枚举类  
	  
由于枚举类型其实是定义了一个类型，枚举元素是该类的实例对象，  
且每个元素都是单例的，故而处于性能上考虑比较两个枚举应使用`==`操作  

### 反射
	  
* 能否分析类能力的程序称为反射  

* Class对象表示的是一个类型，而这个类型未必是一种类  
	int.class表示的是int类型，int不是一种类
  
* 虚拟机为每个类型管理单独一个Class对象，故可用`==`比较两个类对象  

* 除非拥有访问权限，否则Java安全机制只允许反射查看对象有哪些域，  
而不允许读它们的值，但可用 Field 、Method 、Constructor 的 `setAccessible(true)` 方法  
解除此限制

Tips：
  
```
实现方法的回调有几种办法：
1. 接口方式（注册监听器）
2. 反射机制，使用Method的invoke
3. lambda表达式
```
	
### 异常捕获
	  
已检查异常：竭尽全力仍不能完全避免的异常，此时就要提供异常处理器  
未检查异常：精心控制可以完全避免的异常，例如：空指针、除0操作
		
## 第六章 接口、lambda表达式与内部类  
  
### 接口  
	  
* 接口方法为public、域为public static final，JLS推荐无须显式标记   

* Java SE 8中，接口支持静态方法，之前的做法是将静态方法放在伴随类中。  
	例如：` Collection/Collections `、` Path/Paths `  
  
* Java SE 8 中，接口方法可以提供一个默认实现，用 default 修饰符  
	1. 可以消除伴随类（当然 API 中肯定无法移除）
	2. 利于接口演进，就接口添加新默认方法可保证源代码、二进制兼容性
	  
		`源代码兼容`：改变类后，依赖该类的其他类能重新编译  
		`二进制兼容`：改变类后，依赖该类的其他类使用旧的编译文件还能执行
	
	3. 解决方法冲突规则
		- 超类优先  
			某类继承超类实现某接口，超类及接口默认含有相同方法名及参数的方法  
			且不管接口是否包含默认方法，此时子类一定使用是超类的方法
		- 接口冲突
			冲突特指类不实现接口时，使用接口默认方法是否存不知道该调用哪个方法（二义性）  
			某类实现了两个接口，且两接口含有方法名参数一致的方法  
			1. 当这两个方法是抽象时，该类不会提示冲突，选择其中之一实现即可
			2. 当某个接口含默认方法，该类必须实现方法解决冲突（虽然只有一个默认方法）
			3. 当两接口都含有默认方法，该类同样必须实现方法解决冲突  

### 接口示例  
* 回调是一种常见的程序设计模式，可以指出某特定事件发生时采取的动作  
* 接口给回调提供了一个标准和模板（利用接口实现的回调）  
* 对象克隆  
1. Object类包含一个` protected `的 clone 方法  
2. 某类只能在内部调用 Object 的默认 clone 方法克隆它自己  
3. 某类包含其他类，要克隆其他类时，不能调用其他类默认继承 Object 的 clone 方法  
4. 只能让其他类实现 Cloneable 标记接口，并覆盖 Object 的 clone 方法，将修饰符扩大为public才行
5. 所有数组类型都有一个 public 的 clone 方法，包含所有数组元素的副本（引用也是副本）
		
### lambda 表达式  

* 语法  

	`(类型 名称 [,类型 名称]) -> { 语句; [语句;]}`  

	只有一个变量，变量定义的括号可以省略
	变量的类型可以推测，变量的类型可以省略  
	语句只有一个时，代码块花括号可以省略  
	表达式有返回值时，确保每个分支都要有返回值  

* 函数式接口  

	只有一个`抽象方法`的接口称函数式接口，需要这种接口对象时可以提供一个 λ 表达式  
	其实，接口中是可以抽象声明 Object 类的 public 方法，  
	但实际上每个接口的实例都有这些被声明为抽象的方法的Object类的默认实现
	  
	最好把λ表达式看做函数而非对象，λ表达式可以传递到函数式接口中  
	执行λ表达式体与传统接口的实体对象的调用相比，可能更加高效，  
	相比 invokeinterface ， λ 采用 invokedynamic 调用[*引导方法*](https://stackoverflow.com/questions/30733557/what-is-a-bootstrap-method)（ Bootstrape Method ）
	
	反编译后λ表达式的调用被编译成为一个invokedynamic的字节码操作  
	关于此字节码的介绍请查看：[ Invokedynamic ：Java 的秘密武器](http://www.infoq.com/cn/articles/Invokedynamic-Javas-secret-weapon)  

* 方法引用  
  
	有时候现成方法可以完成想要传递到其他代码的某个动作就可以使用方法引用  
	  
	方法引用有3种情况：
	1. object::instanceMethod 例：`System.out::println` 
	2. Class::staticMethod    例：`Math::pow`  
	3. Class::instanceMethod  例：`String::compareToIgnoreCase`
	
	第一种，等同于 `x -> Sytem.out.println(x)`  
	第二种，等同于 `(x,y) -> Math.pow(x,y)`  
	第三种，等同于 `(x,y) -> x.compareToIgnoreCase(y)`
	
	`this::instanceMethod` 与 `super::instanceMethod` 也是合法的  
	分别调用当前类的方法或当前类的父类的方法
	
	构造器的引用为：`Object::new`  
	Java无法构造泛型类型的T的数组，利用构造器引用有时能够保证类型不被丢失  
	例如：`new T[n]` 编译错误，所有之前返回不确定类型的方法声明只能为`Object[]`  
	新方法使用构造方法引用来解决此问题，例如 Stream 的方法：  
	  
	 ```java  
	<A> A[] toArray(IntFunction<A[]>) 
	```
		
* lambda变量作用域  
	
	lambda表达式有3个部分：代码块、参数、自由变量的值（不在代码块定义的变量）  
	这些自由变量被lambda表达式引用，被称为捕获  
	被捕获的变量即使外部方法调用栈执行完后撤栈后仍然能被lambda表达式使用  
	被捕获的变量被要求在lambda代码块中及外部代码中不能改变（effectively final）
	
	**`闭包`**：能够维持外部自由变量的代码块，即使外部自由变量已经不存在  

	注意：  
	lambda表达式中声明的变量名不能与局部变量同名  
	lambda表达式中的this关键字是指创建这个表达式的方法的this  

* 处理lambda表达式  
	  
	使用lambda表达式的重点是`延迟执行(deferred execution)`  
	想延迟的原因：
	- 在单独线程中运行代码
	- 多次运行代码
	- 在算法适当的位置运行代码
	- 发生某种情况时执行代码
	- 只在必要时才运行代码
	
	Java API 中默认提供了函数式接口，[详见链接](https://docs.oracle.com/javase/8/docs/api/java/util/function/package-summary.html)  
	大多数函数式接口都提供了非抽象方法来生产或合并函数  
	例如：  
	Predicate 的 and 、 or 等方法  
	使用` @FunctionalInterface `标注函数式接口是个好习惯  
  
* 许多接口包含很多方便的静态方法来创建接口实例  
	  
	例如：  
	Comparator 的静态 cpmparing 方法有一个`键提取器`函数，  
	接受方法引用或lambda表达式：  
  
	```java  
	comparing(Function<? super T,? extends U> keyExtractor)  
	```  

### 内部类  
	  
* 内部类能够带来以下三点便利：  
1. 可访问该类定义所在的作用域的数据，包括私有数据
2. 可以将自身隐藏使得同包中的其他类无法看见
3. 当需要定义毁掉函数且不想编写大量代码，使用匿名内部类很便捷  

* 编译器修改了所有内部类的构造器，添加了一个外围类的引用参数  

* 内部类中声明的所有静态域都必须是final，确保唯一  
(外部类有多个实例，它们各自都有一个内部类实例，不为final可能就不唯一)  
  
* `非静态内部`内不能有static方法，非要允许有静态方法只能改为`静态内部类`  
静态内部类的静态方法只能访问外围类的静态域和静态方法

* 内部类是一种编译器现象，与虚拟机无关，编译器将内部类编译为$分隔内外部类的类文件  

* 内部类访问外部类的私有域是通过在外部类`加入特殊的静态的访问方法`，类似：access$0  
	局部内部类不能有public、static或private修饰符，它们的作用域被限定在局部代码块中  

* 内部类也可访问外部类的局部变量，只不过局部变量必须为final、或事实上的final  
	Java 7及之前：局部变量必须由final修饰  
	Java 8及之后：局部变量不须强制加final，但在外部类或内部类中都不能更改其值
	
* 匿名内部类可被用于进行`双括号初始化`，由于是匿名子类，故equals的getClass会不等  
  
	```java
	Arrays.toString(new ArrayList<String>() { {
		add("One"; 
		add("Two");
	} } );
	```
	
	如果要对匿名子类调用 equals 方法，由于 getClass 获取到的是匿名子类的，  
	故并不相等，要在内部类获取外部类的类型信息可以使用如下技巧：  

	```java  
	inner.getClass().getEnclosingClass();
	```
  
* 出于隐藏一个类，且该类不需引用外部类对象，可以声明为静态内部类  
	声明在接口内的内部类，自动为static和public的类  
	声明在静态方法里的内部类，虽无法被static修饰但其也不包含外部类对象  

	* **外部类为为何不能被static修饰？**  
	  
		在[链接](https://stackoverflow.com/questions/3584113/why-are-you-not-able-to-declare-a-class-as-static-in-java)给的回答中，`necromancer`的回答及`Lumi`的补答蛮贴切：
	> Top level classes are static by default. Inner classes are non-static by default  You can change the default for inner classes by explicitly marking them static  Top level classes, by virtue of being top-level, cannot have non-static semantics because there can be no parent class to refer to. Therefore, there is no way to change the default for top-level classes.  
	Top Level classes are static because you don't need any instance of anything to refer to them. They can be referred to from anywhere. They're in static storage (as in C storage classes). The language designers could have allowed the static keyword on a top-level class to denote and enforce that it may only contain static members, and not be instantiated; that could, however, have confused people as static has a different meaning for nested classes. 
	  
		大概译文：
		
		> 顶层类（外部类）默认即为静态的类 ，内部类默认为非静态的。可以通过static关键字修饰内部类，从而更改为非默认的静态内部类。顶层类不能有非静态语义其优点在于可以不需要父类就能引文它，因此不能通过任何方式更改顶层类的默认语义。  
		顶层类它们之所以是静态的是因为它们不需要任何具体的实例去引用它们，它们可以在任何地方被引用。它们存储在静态区，Java语言的设计者们原本大可允许使用static关键字修饰顶层类，强制该类只能包含静态的成员及方法而不能被实例化，但是为何不最终不允许static修饰呢？是因为如果允许static修饰外部类会使人们对staic修饰内部类的含义产生误导，静态的内部类是可以同时包含静态的、非静态的成员及方法的。
  
### 代理  
	  
代理类可以在运行时创建全新的类，能够实现指定的接口，具有：
* 指定接口所需的全部方法
* Object类中的全部方法
  
创建代理使用Proxy类的newProxyInstance方法，具有三个参数：
* 类加载器
* Class对象数组，每个元素都是需要实现的接口
* 调用处理器  

	```java
	Proxy.newProxyInstance(loader, interfaces, handler);
	```  

使用代理可能出于如下原因，包括不限于：
* 路由对远程服务器的方法调用
* 程序运行期间，将用户接口事件与动作关联起来
* 为调试跟踪方法调用  

处理器类模板代码：
  
```java
class XxxHandler implements InvocationHandler {
	private Object target;
	
	public XxxHandler(Object t) {
		target = t;
	}
	
	public Object invoke(Object proxy, Method m, Object[] args) 
		throws Throwable {
		// 自定义操作
		// do something...
		return m.invoke(target, args);
	}
}
```

所有代理类都覆盖了 Object 类的 toString 、equals 、hashCode 方法  
Sun 虚拟机中的 Proxy 类将生成以` $Proxy `开头的类名  
使用同一个加载器、接口数组调用两次 newProxyInstance 得到同一个类两个对象，  
该类为：

```java
Class proxyClass = Proxy.getProxyClass(loader, interfaces);
```

代理类一定是 `public` 、`final` 的，其所实现的所有接口为 public 时，代理类不属于某个  
特定包，否则所有 `非 public 的接口必须属于同包`，并且`代理类也属于该包`
  
##  第七章 异常、断言与日志  

### 异常分类  
	  
异常都派生于`Throwable`类，下层分解为：`Error`、`Exception`  
*	Error：Java运行时系统的内部错误和资源耗尽错误，应用程序不抛此类型  

* Exception：下层分为`RuntimeException`、其他异常（例如：IOException）  
	- RuntimeException：程序错误导致的异常，例如：数组越界、null指针
	- 其他异常：程序本身没问题，由于像I/O错误问题导致的属其他异常  

* 非受查异常与受查异常  
	- 非受查异常包括：派生于Error、RuntimeExceptioin的错误或异常  
	- 受查异常：除非受查异常外的其他异常 （此类异常需要进行异常处理）  
* 应该抛异常情况：
	1. 调用一个抛出受检查异常的方法
	2. 程序运行过程中发现错误，并利用throw语句抛出一个受查异常
	3. 程序出现错误，如数组越界
	4. Java虚拟机和运行时库出现的内部错误
	  
	1、2两条是需要使用throws声明显示抛出的，或者捕获异常  
	3、4两条为非检查异常，要么不可控、要么应该避免，故不强制一定抛出  
	  
	注意：  
	- 子类覆盖超类方法，抛出异常不能比超类的异常更通用，只能更特定或不抛出
	- 没有throws说明符的方法不能抛出任何受查异常

### 捕获异常  
	  
捕获异常通常原则：应该捕获知道如何处理的异常，不知如何处理的继续传递  
例外：子类覆写超类方法，而该方法没抛出异常子类必须捕获所有受查异常  

Java SE 7 及之后，单个 catch 支持捕获多个异常类型，但不存在子类关系，语法：
  
```java
catch (Exception | Exception e) { ... }
```
此时，`e` 变量为隐含的 final 的变量只能被赋值为其中之一，这种语法更简洁更高效

* 再次抛出异常与异常链  
	- catch子句可以抛出异常，可以起到如下作用
		- 出于异常识别分类，改变异常类型
		- 受查异常转运行异常（用于不能抛出受查异常的情形）
		- 记录异常日志  
	  
	- catch子句抛出新异常时，可以调用 newE.initCause( oldE ) 使得原始异常不丢失  
	  
	注意：  
	Java SE 7之后，抛出比声明的异常更广的类型时，只要try块仅有声明的异常类型，  
	且在catch块没被改变，声明的异常是合法的  

	如果final块也可能有异常时，最佳实践代码如下：  

	```java
	try {
		try {
			// might throw exceptions
		} finally {
			// might throw exceptions
		}
	} catch(Exception e) {
		// show error message
	}  
	```

	这样可以解耦 try/catch 和 try/finally ，使职责清晰明了  
	try/finally 只负责确保异常后仍可执行 finally 的语句  
	try/catch 只负责记录出现的错误  

	嵌套try时，很可能会出现外层异常覆盖了里层异常，导致原始异常丢失，  
	在外层增加个里层异常是否为空的判断，如果不为空抛出里层异常否则抛出外层异常很有用，  
	例如：
  
	```java
	InputStream in = ...;
	Exception ex = null;
	try {
		try {
			// 可能出现异常代码
		} catch(Exception e) {
			ex = e;
			throw e;
		}
	} finally {
		try {
			in.close();
		} catch(Exception e) {
			if (ex == null) throw e; // 如果里层没有异常，才抛外层异常
			throw ex; // 否则抛出里层异常
		}
	}  
	```
	
	当然，Java SE 7有自动关闭资源的特性，会比这里的处理更为简洁

	```java
	try (InputStream in = ...) {	// 这里的in资源会自动关闭
		// 可能出现异常代码
	} catch(Exception e) {
		// 处理代码可能出现的异常
	} finally {
	}  
	```

	当in关闭出现异常时，会被自动捕获并**被抑制**，  
	异常将由e.addSuppressed方法追加到原代码异常的后面，  
	当然带资源的try/catch会在资源关闭之后执行

* 使用异常机制技巧 （早抛出，晚捕获）
	1. 异常不能代替简单的测试（不能将异常用于逻辑判断，性能太差）
	2. 不要过分细化异常（不要把每条可能出现异常的代码装在独立的try块里）
	3. 利用异常层次结构（寻找合适子类或自创的异常，逻辑错误不抛受查异常）
	4. 不要压制异常（吃掉异常不处理）重要的异常一定要处理
	5. 检测到错误及时处理，不要放过 `早抛出`
	6. 不要羞于传递异常（读一个文件时，出现错误传递异常好过捕获）`晚捕获`

### 断言  
	  
断言允许测试期间向代码中插入检查语句，代码发布时这些插入的检测语句将自动移除  

语法：
* assert 条件;
* assert 条件 : 表达式;
  
条件为 false 时，抛出断言错误的异常  
第2种形式表达式传入异常构造器，转换成一个消息字符串     
		
启用/禁用：

* 由类加载器加载的类
	- 启用：java -enableassertions 或 java -ea
	- 禁用：java -disableassertions 或 java -da

* 虚拟机直接加载的类
	- 启用：java -enablesystemassertions 或 java -esa
	- 禁用：java -disablesystemassertions 或 java -dsa  
	
	参数后都可以接单独的类或包明，表示启用或禁用某个类或指定包下的类
	- java -ea:ClassName
	- java -ea:org.package  

断言只用于测试阶段确定程序内部的错误位置，它不改变程序的逻辑运行  
不要将断言参与业务逻辑处理，实际上也参与不了，断言是致命的不可恢复的错误  
处理系统错误的三种机制：抛异常、使用断言及记录日志    
  
### 日志  
	  
日志可以很容易的定制记录级别、或禁止，也可使用不同处理器进行不同格式的输出，  
并且记录器、处理器都可过滤记录、格式化日志，可配置性很强  

* 基本日志
	- 简单使用可以仅使用一个全局日志记录器  
	`Logger.getGlobal().info("Need to be log...")`  
	- 取消所有日志  
	`Logger.getGlobal().setLevel(Level.OFF)`  

* 高级日志  
	企业级的日志中，应该自定义日志记录器而不要全部交给全局日志记录器  
	```java  
	private static final Logger myLog = Logger.getLogger("org.package");
	```  
	**注意：**   
	- 之所以使用static修饰，是为了**防止垃圾回收**
	- 传入的类似包名字符串能很好的使得子包默认继承父包的设置  
	例如：org包设置日志级别，org.package默认与org设置一致  

	日志有7个记录器级别：  
		`SEVERE`、`WARNING`、`INFO`、`CONFIG`、`FINE`、`FINER`、`FINEST`  
	  
	默认值记录前三个级别的日志，调用`myLog.setLevel(Level.FINE)`可修改默认值  
		`Level.ALL`、`Level.OFF`为开启、关闭所有日志的特殊常量  
	  
	记录日志有下面几种方法：
	- myLog.info(message)				方法名指定日志级别
	- myLog.log(Level.INFO, message)	参数指定日志级别  
	  
	默认将显示日志调用的`类名`、`方法名`，虚拟机优化执行时得不到确切调用信息，采用：  
	
	```java  
	void logp(Level l, String className, String method, String msg);  
	```  
	来获取类和方法的具体位置，另外下面这些方法可用来跟踪方法执行流向：
	- void entering(String cName, String mName)
	- void entering(String cName, String mName, Object p)
	- void entering(String cName, String mName, Object[] ps)  
	- void exiting(String cName, String mName)
	- void exiting(String cName, String mName, Object r)
		
	cName 类名，mName 方法名，p 单参数，ps 参数数组，r 返回值  
	这些调用产生FINER级别、字符串为ENTRY、RETURN开始的日志记录  
		
	记录异常可使用
	- void throwing(String cName, String mName, Throwable t)
	- void log(Level l, String message, Throwable t)

* 日志管理器配置  
	系统默认配置文件存在于`jre/lib/logging.properties`文件中  

	1. 想要另外指定配置文件，有以下几种设置方式：  
		- 虚拟机参数式：`-Djava.util.logging.config.file = configFile`
		- 代码调用式： `System.setProperty("java.util.logging.config.file", file) ` 

	2. 修改默认日志级别，可以在配置文件中加入  
		`.level=INFO`  
		或者指定自己的记录器的记录级别  
		`org.package.level=FINE`  

		程序运行时，可通过`jconsole`改变日志的记录级别  
		具体查看 [Using JConsole to Monitor Applications](http://www.oracle.com/technetwork/articles/java/jconsole-1564139.html)
	  
	3. 日志处理器  
		默认日志记录器发送到ConsoleHandler，由它输出到System.err流  
		并还将记录发送到父处理器，最终处理器为ConsoleHandler（名称为空，即“”）  
		  
		通过`myLog.addHandler(yourHandler)`可以安装自己的处理器  
		通过`myLog.setUseParentHandlers(false)`可以阻止处理器向父处理器传递  
		  
		常用的处理器：
		- SocketHandler 将记录发送到特定的主机和端口
		- FileHandler 将记录收集到文件当中，默认写到用户主目录 java*n*.log  
			其中 *n* 为唯一编号，例如：`java001.log` 或 `java002.log`  
			文件处理器的参数及用法，参考[API文档](https://docs.oracle.com/javase/7/docs/api/java/util/logging/FileHandler.html)			  
	  
	4. 日志记录的加工处理  
		- 过滤器  
			通过实现Filter接口，实现`boolean isLoggable(LogRecord record)`
		- 格式化  
			扩展Formatter类，覆写`String format(LogRecord record)`  
			如果记录有头及尾部时，覆写`String getHead(Handler h)`和  
			`String getTail(Handler h)`
  
* 调试技巧  
	- 打印调试信息  
		`System.out.println("debug...")` 或 `Logger.getGlobal().info("...")`  
	
	- 每个类包含独立main方法，方便单元测试  
	
	- JUnit单元测试套件，比前两种更实用  
	
	- 匿名代理实现方法调用跟踪
	
	- 打印异常堆栈轨迹  
		`throwable.printStackTrace()` 或 `Thread.dumpStack()`  
	
	- 错误信息用文件保存  
		`java MyProgram 1> errors.txt 2>&1`  
	
	- 非捕获异常堆栈信息单独处理
		  
		```java
		Thread.setDefaultUncaughtExceptionHandler(
			new Thread.UncaughtExceptionHandler(){
				public void uncaughtException(Thread t, Throwable e){
					// 将非捕获异常信息额外处理的代码
					...
				}
			}
		);
		```  
	- 虚拟机参数  
		`-verbose` 能提供虚拟机启动相关信息，排除类加载及路径信息  
		`-Xlint` 告知编译器对代码进行检查，例如：`-Xlint:fallthrough`(case穿透)  
		`-Xprof` 能剖析代码调用，将分析结果发送到标准输出，获取哪些方法由JIT编译  
		  
		注：`-X`选项非正式支持，可运行`java -X`打印非标准参数列表  
	
	- jconsole图形工具对应用进行监控和管理，例如：内存、线程使用、类加载等情况  
	
	- jmap工具获得堆的快照，jhat加载该快照浏览`localhost:70000`查看堆内容