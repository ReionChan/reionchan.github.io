---
layout: post
title: 《Effective Java Second Edition》中文版 一 
categories: Book Java  
tags: Java effective  
excerpt: 读《Effective Java Second Edition》的笔记  
image: https://img3.doubanio.com/lpic/s3479802.jpg  
description: 读《Effective Java Second Edition》的笔记 
keywords: Java,  java, Effective, Second, Reion Chan, reionchan
licences: cc
--- 

### 创建和销毁对象

####  1 考虑静态工厂方法替换构造器  

相较于构造器，有以下优势：  

* 静态工厂方法有名称，不拘泥于类名，更易使用
	
* 不必在每次调用都创建新对象  
	可对实例进行重复利用，避免不必要的重复对象  
	
	*实例受控类：能够严格控制在何时哪些实例应该存在的类*   

* 可返回原返回类型的子类型对象，增加灵活性  
	适用于基于接口的框架，灵活性体现在：  
	* 返回对象可以是非公有的、且可以随着调用发生变化
		
	* 返回对象所属类在编写静态工厂方法时可以不必存在  
		例如：典型的 *服务提供者框架*  JDBC API  
		  	  
		由于此书出版较早，有些描述待更新，例如以下描述：
		> 接口不能有静态方法，因此按照惯例，接口 Type 的静态工厂方法被放在一个名为 Types 的不可实例化类中。  
		  
		我们知道，Java 8 中接口是可以有**静态**方法的，详见：[\[CJv1 p.218\]](https://reionchan.github.io/2017/02/01/corejava-v1-note-part1/#%E6%8E%A5%E5%8F%A3)
	
* 创建参数化类型实例时代码更简洁  
	由于此书出版较早，有些描述待更新，例如以下描述：  
	
	> 在调用参数化类的构造器时，即使类型参数很明显，也必须指明。  
  
	我们知道， Java 7 中是支持构造器 *类型推导*  的，故此优势可忽略。  
		
	**Tips：**  
		&emsp;&emsp;Java 7 之前类型推导支持静态方法的调用，不支持构造器、实例方法调用  

相较于构造器，有以下劣势：

* 静态工厂方法的类，如不含 `public`、`protected`的构造器就无法子类化

* 静态工厂方法虽能替代构造器产生对象，但不能很友好的体现在 API 文档中  
	但如能遵照命名规范，还是能弥补这一不足，静态工厂方法名可以是：  
	- valueOf 常用作类型转换方法
	- of 是 `valueOf` 的简化替代，常在枚举类型中使用
	- getInstance   
		一般含参数，返回实例的值与参数相关，无参认为返回唯一实例，即单例
	- newInstance  
		能确保返回的每个实例都与之前实例不同
	- get*Type*   
		与 `getInstance` 类似，但此方法不在返回的类中，后缀 *Type*  是返回类型
	- new*Type*   
		与 `newInstance` 类似，但此方法不在返回的类中，后缀 *Type*  是返回类型
  
####  2 考虑构建器代替多参数的构造器

*构建器*  是[Builder 模式](https://en.wikipedia.org/wiki/Builder_pattern#Java)的一种形式。  

&emsp;&emsp;客户端利用必要的参数调用构造器（或静态工厂）得到此构建器，在此构建器上调用类似 *setter* 方法来设置可选参数，最后调用此构建器无参的 `build` 方法来生成目标类，一般构建器为目标类的**静态成员类**。  

* 出现背景    

&emsp;&emsp;静态工厂、构造器都不能友好地扩展大量的可选参数，因其采用*重叠构造器模式* 会使得客户端代码调用时传参的混淆而导致运行时错误。  
  
&emsp;&emsp;而采用无参构造器配合 *JavaBeans 模式* 的不断 set 值得方式很容易使目标类处于状态不一致，需要额外的努力来实现线程安全。

* 优势
	
&emsp;&emsp;可以拥有多个可变的参数、可以对参数进行约束条件、提供出于安全考虑的保护性拷贝的切入点、且能在单个构建器中创建多个对象，这点可以取代 Java 传统的抽象工厂实现类 Class，因为 Class.newInstance 方法总是企图调用无参构造器，且放弃了编译时对没有无参构造器的检测，而恰恰在利用构建器时，可以避免出现此类问题。
	
**Tips：**  
&emsp;&emsp;如果设计的类很有可能将来会追加构建参数，那么最好一开始就使用构建器。

####  3 私有构造器及枚举类强化单例属性

* Java 5 之前实现单例方法
	
	- 构造器私有的公有静态类成员	
	
	- 构造器私有的公有静态工厂方法  
    
	以上两种方案都有反射及反序列化实例不唯一问题，解决办法：  
	
	1. 声明所有实例域为 `transient`
	
	2. 提供 `readResolve` 方法，详见[第77条](#)  
  
* Java 5 之后推荐使用包含单个元素的枚举类型  
  
	与之前方法相比，更它更简洁且一并解决序列化及反射攻击的问题。  
	
	**单元素枚举类型已成为实现单例的最佳方法。**

####  4 私有构造器强化不可实例化能力

&emsp;&emsp;一些只包含静态方法及静态类的类，即俗称的工具类，例如：java.util.Arrays  
工具类被设计出来是不希望被实例化的，为了避免被实例化，最佳实践为：  
  
```java
public class UtilClass {
	// Suppress default constructor for noninstantiability
	private UtilClass() { }
}
```
1. 将构造方法采用 `private` 修饰
2. 在方法增加一条不要实例化的注释

副作用：该类失去子类化的能力


####  5 避免不必要的对象创建

* 尽量使用字符串字面量而非创建一个新实例
* 不可变类同时提供静态工厂及构造器时，优先静态工厂方法
* 适配器或视图下，对给定的对象的特定视图或适配器不需要创建多个
* 避免不必要的对象创建绝非全部的类都缓存起来，除非重量级对象
* 出于安全性考虑或由于小对象创建回收的廉价性，可以进行保护性拷贝或快建快消

####  6 消除过期的对象引用

* 警惕**无意识的对象保持**，即程序逻辑上无用的对象还存在对其的引用
* 类是自己管理内存，就该考虑内存泄漏问题
* 缓存很容易被遗忘带来内存泄漏隐患
* 客户端注册一个监听器或回调，却没显式取消造成堆积也是内存泄漏隐患

解决办法：

* 及时将逻辑上无用的对象的引用置为 `null`
* 对可能被遗忘的容易堆积的对象采取 *[弱引用](https://www.ibm.com/developerworks/cn/java/j-lo-langref/)* 技术 


####  7 避免使用终结方法

* JLS 规范不保证终结方法及时执行，故不要依赖终结方法来更新重要的持久状态
* `System.gc` 和 `System.runFinalization` 两方法仅增加终结方法被执行的概率

两种终结方法的合法用途：

* 对象所有者忘记显式调用终止方法（如：I/O 类的 `close`）时，终结方法可充当安全网
* 对象的 *本地对等体*  [^1] 没有关键资源的前提下，终结方法可以清理 GC 无法清理的本地对象

[^1]: 本地对等体是一个本地对象，普通对象通过本地方法委托一个本地对象。

**Tips:**  
&emsp;&emsp;“终结方法链”并不会像“构造方法链”那样自动执行，除 `Object` 外含有终结方法的类，其子类覆盖了终结方法，那子类要手工调用超类的终结方法，否则父类终结方法不会被执行。  

一般做法是在子类中的 `finally` 语法块调用父类终结方法，形如：

```java
@Override 
protected void finalize() throws Throwable {
	try {
		... // Finalize subclass state
	} finally {
		super.finalize();
	}
}
```  

更稳妥的做法是采用匿名的单个实例作为 *终结方法守卫者*  ，子类可以省略调用超类调用：

```java
public class Foo {
	private final Object finalizerGuardian = new Object() {
		@Override 
		protected void finalize() throws Throwable {
			... // Finalize outer Foo object
		}
	};
}
```

**对于每一个带有终结方法的非 `final` 公有类，都应该考虑使用这种方法。**


### 所有对象通用的方法

#### 8 覆盖equals时的通用约定  

* 不应覆盖 `equals` 的情形

	1. 类的每个实例本质是是唯一的
	
	2. 不关心是否提供了“逻辑相等”的测试功能  
	例如：一些工具类，它的实例比较毫无意义
		
	3. 超类已经覆盖了 equals，从超类继承的行为对子类也合适
	
	4. 类是私有的或是包级私有，可以确定它的 equals 永远不会被调用  
	此种情形，最好提供一个抛异常的 equals 方法，及时发现意外调用
		  
		```java
		@Override  
		public boolean equals(Object o) {  
			throw new AssertionError(); // Method is never called  
		}
		```  
* 应该覆盖 `equals` 的约定   
**&emsp;&emsp;如果类具有自己特有的“逻辑相等”概念（不同于对象等同概念），而且超类还没有覆盖 equals 以实现期望的行为，这时我们就需要覆盖 equals 方法。这种类通常被称作“值类”。**  
&emsp;&emsp;当然，实例受控的值类就不需要覆盖 equals 方法，如：枚举类型。因为相同值的实例本身就应该是唯一的，此时逻辑相等与对象等同是一回事。    
  

* 覆盖 `equals` 必须满足如下约定 

	- 自反性	`x.equals(x)` 返回 `true`
	- 对称性	`x.equals(y)` 等同于 `y.equals(x)`
	- 传递性	`x.equals(y)`为真， 且 `y.equals(z)`为真，则：`x.equals(z)`也为真
	- 一致性	`x.equals(y)`为真，对象不变时，多次调用结果要一致
	- 非空性	`x != null`为真，则：`x.equals(null)`一定返回 `false`
	
	&emsp;&emsp; 一般保证**和当前类比较的对象属于当前类族（注：子类也属于父类的类族，即 `instanceof` 运算）**，就能保证对称性；而在子类增加值组件会破坏传递性；不依赖不可靠的资源就能确保一致性；非空性可以事先进行 `instanceof` 判断就可以保证（`null` 不属于任何类型，永远返回 `false`）。
	  
    	&emsp;&emsp;由于我们无法扩展一个可实例化的类并增加新的值组件，同时还确保传递性。如果放弃面向对象的多态，即：**和当前类比较的对象只能属于当前类型（注：不含子类型，`o.getClass() == getClass()` 为真）**还是能确保传递性，但此时已经违背 *里氏替换原则*  [^2]。  

	综上所述又引出一个原则：  
	> 一个可实例化类的子类是不允许增加值组件，换句话说，抽象类的子类或接口的实现类才能增加值组件。如果允许子类间、实现类间进行比较，那么 `equals` 里要用 `instanceof 抽象类/接口` 判断和比较，即将 `equals` 方法放在抽象类里共享使用，否则只能在各个具体子类或实现类里使用各自类型进行 `instanceof` 判断。（参考集合接口的 `equals` 处理）

[^2]: 里氏替换原则中说，任何基类可以出现的地方，子类一定可以出现。  

* 覆盖 `equals` 最佳实践  

	1. 使用 `==` 操作符检查参数是否为这个对象本身引用。（性能优化考虑）
	
	2. 使用 `instanceof` 操作检查参数是否为正确的类型。（正确类型指 `equals` 方法所在的类、或其实现/继承的接口/超类，取决与是否允许子类间/实现类间的比较）
	
	3. 把参数转换成正确类型
	
	4. 对该正确类型中每个关键域进行比较，查看是否都相同。（除 float、double 外的基本类型使用 `==` 比较，float、double 分别使用 `Float.compare`、`Double.compare`，数组类型使用 `Arrays.equals`比较，其他对象类型递归使用 `equals` 比较）
	
	5. 验证并测试是否满足对称性、传递性及一致性，不让 `equals` 方法过于智能，且需用 `@Override`标注覆盖方法，并同时覆盖 `hashCode` 方法

#### 9 覆盖equals时总要覆盖hashCode  

* JLS 有关 hashCode 的约定
	1. 对象的 equals 方法所用到的信息没被修改，多次调用 hashCode 必须返回相同结果，同应用的多次执行，每次返回的整数可以不同。后半句表明，同应用的一个对象可以在不同时间 hashCode 值不同，但不推荐这么做，详见：[hashcode-changes](https://stackoverflow.com/questions/28255640/java-equals-and-hashcode-changes)
	
	2.  两对象 equals 比较时相等，则两对象的 hashCode 必须返回相同结果
	
	3. equals 比较不相等两对象，不硬性要求它们的 hashCode 值不同，但考虑散列集性能，最好值不同

* hashCode 生成策略

	- 初始 result 为一个非零常数（为零会抹掉第一个域对最终 hashCode 值的影响），素数最佳
	
	- 对象中每个关键域 f （涉及 equals 比较的域），完成如下步骤：  
		1. 计算 f 的散列码 c  
			* 为 boolean 类型，`c = f ? 1 : 0`  
			
			* 为 byte、char、short 及 int 类型，`c = (int) f`  
			
			* 为 long 类型，`c = (int) (f ^ (f >>> 32))`
			
			* 为 float 类型，`c = Float.floatToIntBits(f)`  
			
			* 为 double 类型，`c' = Double.doubleToLongBits(f)`、  
			&emsp;&emsp;`c = (int) (c' ^ (c' >>> 32))`
			
			* 为引用类型，本类 equals 方法中：  
			&emsp;&emsp;若对 f 采用递归调用 equals，则 `c = f.hashCode()`  
			&emsp;&emsp;若对 f 采用更复杂比较，则为 f 生成一个*范式* ，然后对范式调用 hashCode  
			&emsp;&emsp;若 f == null，`c = 0`
			
			* 为数组类型，把数组每个元素当做新域进行上面的递归处理，1.5 以上版本可以使用 `Arrays.hashCode()` 
		
		2. 更新 result 值，使用公式：`result = 31 * result + c`
		
		3. 返回 result
		
		4. 编写单元测试验证是否达到 hashCode 约定

Tips  
 * HashSet、HashMap 的 keySet 都需要根据 hashCode 值来索引、比较对象。  
糟糕的 hashCode 会导致集合的性能大打折扣，详细参考： [hashmap-and-how-it-works](https://www.java-success.com/hashmap-and-how-it-works/)
* 用数字 “**31**” 去累乘在于其特性：`31 * i == i << 5 - i`，编译器能化乘为移位及减法操作 
* 不可变类如果计算散列码开销大，可以缓存散列码减小开销  
* 不要试图将部分关键域的耗时计算从求散列码的运算中移除，可能散列表慢到几乎不可用  
* 程序不能依赖 hashCode 方法的值来实现业务逻辑，这会限制 hashCode 算法的优化空间

#### 10 始终要覆盖toString
&emsp;&emsp;Object 对象默认提供了一个 toString 方法的实现，返回“类名@[散列码无符号十六进制]”的格式。但此格式并不能很好的描述对象，故推荐每个类都应覆盖默认的 toString方法，而一个好的 toString 方法应该满足如下约定：  
  
* 简洁易读且满足自描述  
&emsp;&emsp;应该将类的所有值得关注的信息纳入返回信息中，如果信息过多可以进行格式化处理，并最好提供对象到格式化的互转方法。但格式化的处理限制了将来更改的灵活性，一旦格式确定下来就要维持兼容性。而如果格式不固定也要为包含在返回值当中的信息做编程式的访问途径，否则使用者通过返回字符串来解析，变相的加大将来更改的格式的影响。

* 含详细描述的文档注释  
&emsp;&emsp;不管返回的字符串是否是确定的格式化信息还是非格式的字符串，都要在注释中加以说明。这样做能提示使用者是否能依赖返回格式，如格式不固定，要提醒使用者并做免责声明。

Tips：  
  
* Java 1.5 之后 [String.format](https://docs.oracle.com/javase/7/docs/api/java/lang/String.html#format(java.lang.String,%20java.lang.Object...)) 方法提供类似 C 语言的 sprint 的功能

#### 11 谨慎覆盖clone  
&emsp;&emsp; Cloneable 接口是一个典型的 [*mixin* ](https://www.zhihu.com/question/20778853) 接口，即可以将克隆行为混入到所有实现该接口的类中。实现了该接口的类都具有从 Object 对象继承下来默认受保护的 `clone()` 本地方法。至于为何如此别扭的设计，参考知乎 [*RednaxelaFX*  的回答](https://www.zhihu.com/question/52490586/answer/130786763)。 这种通过接口标识改变超类中受保护方法的行为的做法是一种极端的、不值得效仿的接口用法。  
  
&emsp;&emsp;之所以要定义成本地方法，是因为克隆属特殊操作交由 JVM 直接处理更为合适，然而在错综复杂的继承关系层级中要确保最终都能调用到本地方法是需要遵从如下约定：  
  
* 覆盖了非 `final` 类中的 clone 方法，则应该返回一个通过 `super.clone()` 方法得到的对象  
&emsp;&emsp;非 `final` 类存在潜在子类，如果不通过继承链一层层往上调用 `super.clone()` 就会出现断链而无法最终调用到 `Object.clone()` 的本地方法返回错误的克隆对象。 

* clone 方法是另一个构造器，你必须保证它不会伤害到原始的对象，并确保正确地创建克隆对象的约束条件  
&emsp;&emsp;此处的不伤害指对克隆对象的域的更改不能影响到原对象，基本类型及不可变对象默认不影响，尤其要注意引用类型及数组集合类型，必要时采取遍历进行深度克隆。而有些表达唯一性的基本类型也要进行处理，比如：对象的全局 ID值，每个对象理应不一致。  
&emsp;&emsp;克隆复杂对象时，也可以先调用 `super.clone()` 后，然后是该对象所有状态清空为空白状态，然后调用高层方法重新产生域状态，例如：得到克隆的 HashMap 对象后，将所有元素清空，然后遍历原对象，依次重新调用克隆对象的 put 方法，此种做法合理且优美，但比内部克隆方法高效。

* 为继承而设计的类应该模拟 `Object.clone()` 的行为：声明为 `protected`、抛出 CloneNotSupportedException 异常，并且不应该实现 Cloneable 接口  
&emsp;&emsp;不考虑继承的话，可以返回公有的、当前类型的、不抛克隆异常的方法，方便调用者使用

* clone 方法禁止给 `final` 域赋新值，除非原对象与克隆对象可以安全共享该域，否则只能去掉 `final` 修饰符

* 同构造器一样，clone 方法内不允许调用非 final 的方法，否则给覆盖了此方法的子类在克隆修正对象各个状态前，执行了子类当中的覆盖方法，导致克隆与原始对象不一致   

综上所述：  

1. 实现了 Cloneable 接口的类都应该用一个公有的方法覆盖 clone
    
2. 此方法先调用 `super.clone()`，然后修正任何需要修正的域，可变对象必要时进行深度拷贝  

3. 代表序列号、唯一 ID 值得域，或代表创建时间的域，不管是基本类型还是不可变都需修正  
  
Tips：  
&emsp;&emsp;有必要这么这么复杂来克隆一个对象么？  
  
&emsp;&emsp;答：如果你扩展了一个实现了克隆接口的类，那么你必须实现一个行为良好的克隆方法，别无他法。否则，最好提供某些其他途径来代替对象拷贝，或者干脆不提供这样的功能。其他途径包括提供一个 *拷贝构造器*  或 *拷贝工厂* ，形如：  
  
```java  
// 拷贝构造器
pubilc Foo(Foo f);  

// 拷贝工厂  
public static Foo newInstance(Foo f);  
```  
&emsp;&emsp;相比 clone 方法，它更灵活：  
  
* 无需遵守 clone 方法的那些约束，况且这些约束没能形成正式文档规范
  
* 不会与 final 域的正常使用发生冲突
  
* 不抛出不必要的受检异常
  
* 不需要类型转换
  
* 它们还可以追加参数不改变形式，起到将原对象转换成另一对象类型，达到 *转换器* 角色

#### 12 考虑实现Comparable接口  
&emsp;&emsp;虽然 `compareTo()` 方法并未在 Object 中声明，但它有很多与 equals 相似的特征，例如：需要遵照恰当的归约才能使一些基于该接口实现的一些集合类的排序、搜索得以顺利进行、比较时也应需满足： 自反性、对称性、传递性。值得注意，虽然并不强求，但比较的结果最好与 equals 保持一致，如果不一致需要在方法的文档注释中加以说明。禁止继承一个非 final 类，并在该类增加影响比较结果的关键域，因为这会破坏传递性。

&emsp;&emsp;简单来说，一个对象与另一个对象比较时，前者大于后者结果为正整数，前者等于后者为零，前者小于后者为负整数。在具体方法实现上，如果比较采用相减操作时，格外留意结果溢出问题。另外，如果想要比较两个没有实现 Comparable 接口，或者想用非标准形式的比较时，可以使用 显示的 comparator，或自定义实现一个 Comparator。    
