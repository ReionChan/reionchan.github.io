---
layout: post
title: Java 技术栈笔记 —— Java 上篇
categories: Java Book
excerpt: JavaGuide 笔记，涵盖 Java 程序员需要掌握的核心知识
image: https://upload.wikimedia.org/wikipedia/en/3/30/Java_programming_language_logo.svg
description: JavaGuide 笔记，涵盖 Java 程序员需要掌握的核心知识
keywords: Java Guide Collection Thread IO Concurrent Program
licences: cc
---

<br/>

<img src="https://upload.wikimedia.org/wikipedia/en/3/30/Java_programming_language_logo.svg" alt="Java Logo" width="=180"/>



## 序

&emsp;&emsp;本 Java 技术栈笔记系列文章，是读[《JavaGuide（Java学习&面试指南）》](https://javaguide.cn/home.html)这篇涵盖 Java 程序员需要掌握的核心知识的文章时的个人的摘录及补充笔记。未读此篇指南的读者请优先阅读，而本笔记仅针对个人情况做了筛选摘录与补充，留以自用，若同时也能您有所帮助深感荣幸。

&emsp;&emsp;在此感谢 **[JavaGuide](https://javaguide.cn/)** 及开源社区，对本文有何建议欢迎补充。

> [《JavaGuide（Java学习&面试指南）》](https://javaguide.cn/home.html)

## 基础

### 语法相关

* ***strictfp*** 严格遵照 **IEEE** 规范的浮点数表示，中间结果指数、尾数都截断为 **64** 位

* ***transient*** 所修饰的类成员变量将不参与序列化，自定义变量类型需实现 Serializable 接口才能被此修饰符修饰

* **`>>`** 带符号右移，高位如果负数填充 **1**，正数填充 **0**

* 移位超过最大位数（**int** 为 32 位，**long** 为 64 位），将先对移位执行最大位取模运算后，移动余数位

* 经分析后对象确认没有逃逸到方法外部时，该对象会通过标量替换，实现栈上分配对象

* 包装类型的缓存机制

  * **Byte**、**Short**、**Integer**、**Long** 这 4 种包装类创建了 ***[-128, 127]*** 的缓存区间值
  * **Character** 创建了 ***[0, 127]*** 的缓存区间值
  * **Boolean** 缓存 ***TRUE***、***FALSE*** 两个常量

* 超出 **long** 取值范围的整数表示

  * **`java.math.BigInteger`** 内部使用 `int[]` 数组来模拟非常大的整数
  * `bigInteger.xxxValue()` 对象方法将转换成 xxx 类型的整形，超出 xxx 范围会被丢弃
  * `bigInteger.xxxValueExact()` 对象方法转换成 xxx 类型的整形，超出抛出 ***ArithmeticException*** 异常

* 超出浮点数取值范围的小数表示，即定点数

  * **`java.math.BigDecimal`** 内部通过一个 BigInteger 和 一个 scale 来表示整数及小数位数

    ```java
    public class BigDecimal extends Number implements Comparable<BigDecimal> {
        private final BigInteger intVal;
        private final int scale;
    }
    ```

  * `bigDecimal.stripTrailingZeros()` 对象方法去掉末尾的 0

    ```java
    BigDecimal d1 = new BigDecimal("123.4500");
    BigDecimal d2 = d1.stripTrailingZeros();
    System.out.println(d1.scale()); // 4
    System.out.println(d2.scale()); // 2,因为去掉了00
    
    BigDecimal d3 = new BigDecimal("1234500");
    BigDecimal d4 = d1.stripTrailingZeros();
    System.out.println(d3.scale()); // 0
    System.out.println(d4.scale()); // -2, 表示这个数是整数，并且末尾有两个 0
    ```

  * 几种保留规则

    ```java
    public enum RoundingMode {
       // 2.5 -> 3 , 1.6 -> 2
       // -1.6 -> -2 , -2.5 -> -3
       UP(BigDecimal.ROUND_UP),
       // 2.5 -> 2 , 1.6 -> 1
       // -1.6 -> -1 , -2.5 -> -2
       DOWN(BigDecimal.ROUND_DOWN),
       // 2.5 -> 3 , 1.6 -> 2
       // -1.6 -> -1 , -2.5 -> -2
       CEILING(BigDecimal.ROUND_CEILING),
       // 2.5 -> 2 , 1.6 -> 1
       // -1.6 -> -2 , -2.5 -> -3
       FLOOR(BigDecimal.ROUND_FLOOR),
       // 2.5 -> 3 , 1.6 -> 2
       // -1.6 -> -2 , -2.5 -> -3
       HALF_UP(BigDecimal.ROUND_HALF_UP),
       // 2.5 -> 2 , 1.6 -> 2
       // -1.6 -> -2 , -2.5 -> -2
       // 银行家算法:
       // 舍去位的数值小于5时，直接舍去。
       // 舍去位的数值大于5时，进位后舍去。
       // 当舍去位的数值等于5时，
       //    若5后面还有其他非0数值，则进位后舍去
       //    若5后面是0时，则根据5前一位数的奇偶性来判断:
       //      奇数进位
       //      偶数舍去
       HALF_EVEN(BigDecimal.ROUND_HALF_EVEN),
       //......
    }
    ```

    * 加、减、乘时，精度不会丢失
    * 除时，存在无法除尽，需指定舍入模式

  * `bigDecimal.equals()` 比较时，需要值相等、`scale()` 也必须相等，故最好使用 `compareTo()` 来比较 

* 方法重写遵循 **“两同两小一大”** 原则

  * ***方法名***、***形参列表*** **相同**
  * ***子类返回类型***、***子类声明抛出异常类*** 需和父类 **更小** 或 **相等**
  * ***访问权限*** **更大** 或 **相等**

### **Object 相关**

* ***notify()***、***notifyAll()***、***wait()***、***wait(long timeout)***、***wait(long timeout, int nanos)***

  * 协作方法必须在对象监视器下被调用（简单理解为需在 **synchronized** 代码块执行）
  * 否则将抛出 ***IllegalMonitorStateException*** 异常

* 协作方法语义

  ```java
  class HelloWorld {
      public static void main(String[] args) throws Exception {
          // 锁对象
          Object obj = new Object();
          
          System.out.println("start...");
          
          synchronized(obj) {
              // 等待方法
              // ----------------------------------------------
              // 语义：
              //   当前已获得 obj 对象监视器的执行线程 main 放弃该监视器，
              //   让出 CPU 执行时间，进入等待状态，
              //   等待其它线程对 obj 对象的监视器的通知调用恢复执行。
              //
              //   无参方法：一直等待通知
              //   有参方法：等待给定时间后恢复执行
              obj.wait();
              //obj.wait(1000);
              //obj.wait(1000, 100);
              
              // 通知方法
              // ----------------------------------------------
              // 语义：
              //   当前已获得 obj 对象监视器的执行线程 main
              //   通知一个等待 obj 对象监视器的线程从等待状态恢复执行
              //   
              //   notify: 只通知其中一个线程
              //   notifyAll: 通知所有线程
              //obj.notify();
              //obj.notifyAll();
          }
          
          System.out.println("stop...");
      }
  }
  ```

* Object 的 ***hashCode()*** 与 ***equals()*** 方法

  * hashCode 能提高容器存放对象时的效率
  * hashCode 相同时（哈希碰撞），对象并不一定相同，故还需比较 equals 方法
  * hashCode 相同，并且 equals 相同，才认为两个对象相同
  * hashCode 不同等对象，最好 equals 也不相同，即 hashCode 与 equals 需要一致
  * 当一个类不会被使用在哈希容器中时，hashCode 与 equals 不必强制要求一致
  * 当一个类会被使用在哈希容器时，equals 相同的对象请务必确保 hashCode 一致，否则哈希容器将出现重复

### **String 相关**

* Java 9 之前，采用 `private final char value[]` 保存字符串，之后采用字节数组
* String 不可变保证：
  1. String 类被 **final** 关键字修饰，保证没有子类修改其属性的风险
  2. value[] 数组是 **private final** 修饰，确保初始化后不再被修改
* StringBuilder 与 +、+= 比较
  * +、+= 是 StringBuilder 的语法糖
  * 循环体内请使用 StringBuilder 方式，避免循环体内重复创建 StringBuilder 实例
  * Java 9 对 +、+= 语法糖做了优化，可以毕淼循环体中大量创建 StringBuilder 实例
* intern() 方法
  * 如果字符串常量池不存在字符串对象的引用，就在常量池创建一个指向该字符串对象的引用并返回
  * 如果字符串常量池存在字符串对象的引用，直接返回该引用
* 对于在编译时确定的字符串的 +、+= 操作，编译器将会对其优化成拼凑后的字符串并放入常量池

### 异常相关

* Checked 异常、Unchecked 异常
  * Unchecked 异常编译时，不处理异常编译不会报错
  * Checked 异常编译时，一定要对其进行处理（抛出、捕获）否则会报错
  * **RuntimeException** 及其子类属于运行时异常，为 Unchecked 异常
  * 除此外属于 Checked 异常，例如：IO 相关异常、SQL 异常
* try-catch-finally 
  * finally 块的语句无论 try 块是否发生异常，都会在 try 块 return 之前被执行
    * 若 JVM 退出、执行线程死亡、CPU 关闭，finally 块也可能不会被执行
  * finally 块最好不使用 return 语句，否则会忽略 try 块中的 return
* try-with-resources
  * 资源需实现 **`java.lang.AutoCloseable`** 或者 **`java.io.Closeable`**
  * 关闭资源操作在任何 catch、finally 块之前被执行
* 注意事项
  * 异常不要定义成静态变量，会导致异常栈信息错乱
  * 抛出的异常信息要有意义、抛出尽量具体的异常（继承关系树尽量最底层）
  * 打印日志后的异常不建议再次抛出

### **泛型相关**

* 泛型类型变量限定 **`<T extends BoundingType>`**

  * BoundingType 可以由 **&** 分隔多个限定类型，类型中有且仅有一个类且放在第一位
  * 接口无此数量限制

* 子类覆盖父类泛型方法时的 ***桥方法***

  ```java
    class Pair<T> {
        public void setSecond(T second) { ... }
        public T getSecond() { ... }
    }
  	
    class DateInterval extends Pair<LocalDate> {
        public void setSecond(LocalDate second) { ... }
        pubilc LocalDate getSecond() { ... }
    }
  ```

  其本质是编译器生成合成方法，由该合成的桥方法桥接调用子类的方法：

  ```java
  // 此为编译器添加的隐藏桥方法
  public void setSecond(Object second) {
      setSecond((LocalDate) second); // 强制转换调用恰当的方法
  }
  // 此为编译器添加的隐藏桥方法
  public Object getSecond() { // 这个方法更诡异，同类中有同名同参数方法
      return getSecond();
  }
  ```

  还可以发现：

  * 在虚拟机规范层级，方法重载是会追加上方法返回值类型来做区分
  * 在 Java 语言规范层级，方法重载只考虑方法名、方法参数列表来做区分

* 约束与局限性

  * Java 语言规范层级，不允许创建泛型数组（例如：`new T[2]` 语句将报错）

  * 虚拟机规范层次，允许虚拟机生成泛型数组

    ```java
    // 此处会引发编译器警告
    // 此处 ts 可变参数实际由虚拟机产生 T 运行时确定类型的数组实例
    @SuppressWarnings("unchecked")
    public static <T> void addAll(Collection<T> coll, T... ts)  
        // Java SE 7之后还可以采用
        @SafeVarargs
        public static <T> void addAll(Collection<T> coll, T... ts) 
    
        // 传统反射方法，获取运行时具体类型，反射生成泛型实例进行泛型方法参数赋值调用    
        public static <T> Pair<T> makePair(Class<T> cl) {
        try {
            return new Pair<>(cl.newInstance(), cl.newInstance());
        } catch(Exception ex) {
            return null;
        }
    }	
    Pair<String> p = Pair.makePair(String.class);
    
    // JDK 8 之后采取函数式方法进行泛型方法参数赋值调用
    public static <T> Pair<T> makePair(Supplier<T> constr) {
        return new Pair<>(constr.get(), constr.get());
    }
    Pair<String> p = Pair.makePair(String::new);
    ```

  * 不能构造泛型数组，采取下面方式变通：

    ```java
    // 反射方式生成泛型数组
    public static <T extends Comparable> T[] minmax(T... a) {
        Object obj =   
            Array.newInstance(a.getClass().getComponentType(), 2); 
        return (T[]) obj;
    }
    String[] strs = Pair.minmax("Tom", "Dick");
    
    // JDK 8 方式
    public static <T> T[] minmax(IntFunction<T[]> constr) {
        return constr.apply(2);
    }
    Pair<String> p = Pair.makePair(String::new);
    ```

  * 类型变量、类型参数

    ```java
    类型变量（Type Parameter）：FooClass<T> 这里的 T 就是类型变量
    类型参数（Type Argument）：FooClass<String> 这里的 String 就是类型参数
    ```

  * 静态域、方法中不能引入非静态泛型类型变量

    ```java
    public class Singleton<T> { // 此处声明非静态的类型变量
        private static T singleInstance; // error 不能引用非静态类型变量
    
        public static T getSingleInstance() { // error
            return singleInstance;
        }
    
        // 注意区别如下静态类型变量, 该泛型变量 U 声明在静态方法中，故为静态类型变量
        public static <U> U methodName(U obj) { // 此处声明的U为静态类型变量
            return (U) obj;
        }
    
        // 类型参数的定义可以出现在 类名后的<> 或者 方法修饰符后返回参数之前的<>
    }
    ```

### 序列化相关

* ***serialVersionUID*** 属于版本控制作用，推荐手动指定，未指定时将由编译器动态生成

  ```java
  private static final long serialVersionUID = 1L;
  ```

  虽然静态成员变量不会被序列化，但这个序列化版本号做了特殊处理，反序列时用来校验一致性。

*  ***transient*** 关键字修饰的成员变量也被排除在序列化之外

  * 该关键字只能修饰变量，不能修饰类、方法
  * 反序列化时，被此修饰的成员变量将会赋默认值

* 第三方序列化

  * Kryo 高性能的 Java 序列化、反序列工具
  * Protobuf、ProtoStuff 支持多种语言的跨平台序列化、反序列化工具
  * Hessian 轻量级、自定义描述的二进制 RPC 协议

### 反射相关

* 获得 Class 对象四种方式

  ```java
  // 方式一
  Class alunbarClass = TargetObject.class;
  // 方式二
  Class alunbarClass1 = Class.forName("cn.javaguide.TargetObject");
  // 方式三
  TargetObject o = new TargetObject();
  Class alunbarClass2 = o.getClass();
  // 方式四，类加载器获取 Class 对象不会进行初始化，即：静态代码块、静态对象不会得到执行
  ClassLoader.getSystemClassLoader().loadClass("cn.javaguide.TargetObject");
  // 触发 Class 初始化的方法：
  // 1. 主动访问静态成员、方法
  // 2. 创建该 Class 类型的对象实例
  // 3. 反射访问这个 Class
  ```

### **代理相关**

* 代理模式与装饰模式

  * 代理模式强调访问控制，一般被代理的类由代理类隐藏，对客户端透明
  * 装饰模式强调增强功能，一般被装饰的类由客户端提供，可由客户端选择不同装饰类，达到不同功能

* 静态代理

* 动态代理

  * JDK 动态代理

    * `java.lang.reflect.Proxy` 生成动态代理三要素：**ClassLoader**、**Class[]**、**InvocationHandler**

      * ClassLoader 负责加载所需接口
      * Class[] 指明需要生成动态代理的接口类型数组
      * InvocationHandler 对代理方法的调用被分发到此处理器的 `invoke` 方法

    * 示例

      ```java
      /**
       * JDK 动态代理
       */
      private static void jdkDynamicProxy() {
      
          /**
           * 三要素
           * --------------------------------------
           * ClassLoader          负责加载所需代理接口的类加载器
           * Class[]              代理接口类型数组
           * InvocationHandler    方法调用处理器，对代理方法的调用都会被定向到此处理器的 invoke 方法
           */
          ClassLoader loader = DynamicProxyBootstrap.class.getClassLoader();
          Class[] interfaces = of(Human.class);
          InvocationHandler handler = new InvocationHandler() {
      
              // 被代理类，也可以使用构造参数由外部指定具体被代理对象
              private Human human = new Student("张三");
      
              /**
               * @param proxy     代理对象
               *                      1. 提供反射获取代理对象的一些字节码信息
               *                      2. 作为 invoke 方法返回值时，可以对代理对象进行链式调用，例如：proxy.foo().bar()
               * @param method    当前被调用的方法
               * @param args      当前被调用的方法参数
               */
              @Override
              public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                  System.out.println("---方法执行前---");
      
                  // 此处反射调用，被调用的对象传入被代理的对象，如果传递代理对象 proxy 将会导致循环调用
                  // 语义：代理对象【额外操作】后，交给被代理对象执行【具体业务】
                  Object result = method.invoke(human, args);
      
                  System.out.println("---方法执行后---");
                  return result;
              }
          };
      
          Human human = (Human) Proxy.newProxyInstance(loader, interfaces, handler);
          human.eat();
      }
      ```

  * CGLIB 动态代理

    * CGLIB 原始 API

      * CGLIB 重要类：**Enhancer**、**InvocationHandler**、**MethodInterceptor**

        * Enhancer 字节码增强器
        * InvocationHandler 与 `java.lang.reflect.InvocationHandler` 调用处理器类似
        * MethodInterceptor 采用方法拦截方式进行代理

        > [【深度思考】聊聊CGLIB动态代理原理](https://blog.csdn.net/zwwhnly/article/details/130286258)

      * 示例
      
        ```java
        /**
         * CGLIB 动态代理
         *
         * <pre>
         * {@code
         * 注意：
         *      此处用的 CGLIB API 为 spring-core 内置重新打包的版本
         *      Spring 单纯修改了包名，和原始版本并无差别
         *      如果想获取原始版本，请依赖如下 GAV
         *      ------------------------------------------------
         *      <!-- https://mvnrepository.com/artifact/cglib/cglib -->
         *      <dependency>
         *          <groupId>cglib</groupId>
         *          <artifactId>cglib</artifactId>
         *          <version>3.3.0</version>
         *      </dependency>
         * }
         * </pre>
         */
        private static void cglibDynamicProxy() {
            /**
             * Enhancer             增强器
             * InvocationHandler    调用处理器，对代理方法的调用都会被定向到此处理器的 invoke 方法 （与 JDK 动态代理类似）
             * MethodInterceptor    方法拦截器，
             */
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(Student.class);
            enhancer.setInterfaces(of(Flyable.class));
        
            // 第一种调用处理器 InvocationHandler
            class MyInvocationHandler implements org.springframework.cglib.proxy.InvocationHandler {
        
        
                // 被代理类，与 JDK 动态代理类似，需要依赖一个具体的被代理类实例，此处使用构造参数由外部指定具体被代理对象
                private Student student;
        
                public MyInvocationHandler(Student student) {
                    this.student = student;
                }
        
                /**
                 *  InvocationHandler 类型的方法调用处理器，与 JDK 动态代理使用方式一致
                 *
                 *  @param proxy    代理对象
                 *                      1. 提供反射获取代理对象的一些字节码信息
                 *                      2. 作为 invoke 方法返回值时，可以对代理对象进行链式调用，例如：proxy.foo().bar()
                 * @param method    当前被调用的方法
                 * @param objects   当前被调用的方法参数
                 */
                @Override
                public Object invoke(Object proxy, Method method, Object[] objects) throws Throwable {
                    System.out.println("\n---方法执行前---");
        
                    // 此处反射调用，被调用的对象传入被代理的对象，如果传递代理对象 proxy 将会导致循环调用
                    // 语义：代理对象【额外操作】后，交给被代理对象执行【具体业务】
                    Object result = method.invoke(student, objects);
        
                    System.out.println("---方法执行后---");
                    return result;
                }
            };
        
            // 第二种方法调用拦截器 MethodInterceptor
            class MyMethodInterceptor implements MethodInterceptor {
        
                // 被代理类，MethodInterceptor 方式的实现其实不必强依赖一个被代理实例
                // 当需要对父类原始方法的调用时，intercept 方法暴露了类似 super() 的方法引用 methodProxy
                // 此处只是单纯为了实现与第一种方式达到一致效果，而保存了一个被代理实例
                private Student student;
        
                public MyMethodInterceptor(Student student) {
                    this.student = student;
                }
        
                /**
                 *
                 * @param proxy         代理对象，被代理类的子类实例
                 * @param method        当前被调用的方法
                 * @param objects       当前被调用的方法参数
                 * @param methodProxy   当前被调用的方法对应的子类覆盖方法
                 */
                @Override
                public Object intercept(Object proxy, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                    System.out.println("\n---方法执行前---");
        
                    Object result = null;
        
                    // 检查方法来源于 Flyable 接口
                    if (method.getDeclaringClass().isAssignableFrom(Flyable.class)) {
                        // 不依赖代理对象的调用，注意第一个参数传的是代理对象 proxy
                        // 优点：
                        //      不依赖代理对象，可以引入任意其他接口，并在此处分发给具备接口行为能力的其他类处理
                        // Plane 具备飞行能力，将飞行行为交给飞机处理
                        Flyable c919 = new Plane("C919", student);
                        result = methodProxy.invoke(c919, objects);
                    } else if (method.getDeclaringClass().isAssignableFrom(Student.class)) {
                        // 依赖代理对象的调用，注意第一个参数传的是被代理对象 student
                        // 此种 调用模拟 【第一种 InvocationHandler 方式】、【JDK 动态代理方式】类似
                        // 语义：将方法调用直接委托给传入的 张三 执行
                        result = method.invoke(student, objects);
                        // 不依赖代理对象的调用，注意第一个参数传的是代理对象 proxy
                        // 语义：通过代理类 proxy 直接调用父类 Student 的方法，故不会造成嵌套死循环调用
                        result = methodProxy.invokeSuper(proxy, objects);
                    }
        
                    System.out.println("---方法执行后---");
                    return result;
                }
            };
        
            Student student = new Student("张三");
            // 添加两个回调
            enhancer.setCallbacks(of(new MyInvocationHandler(student), new MyMethodInterceptor(student)));
            // 设置每种回调具体类型
            enhancer.setCallbackTypes(of(MyInvocationHandler.class, MyMethodInterceptor.class));
        
            // 定义回调过滤器，通过此过滤器来指定哪个方法使用哪个回调进行处理
            CallbackFilter filter = new CallbackFilter() {
        
                // 回调方法索引，要保持和 setCallbacks 数组中的 Callback 顺序、索引最大长度一致
                // 索引 0，指定回调方法 MyInvocationHandler，交给被代理类执行
                public static final int TARGET = 0;
                // 索引 1，指定回调方法 MyMethodInterceptor，交给代理类执行
                public static final int PROXY = 1;
        
                /**
                 * @param method    当前被调用的方法
                 */
                @Override
                public int accept(Method method) {
                    String methodName = method.getName();
        
                    // eat、equals、hashCode 使用构造方法中传入的【被代理类】执行，也就是张三
                    if (methodName.equals("eat")
                            || methodName.equals("equals")
                            || methodName.equals("hashCode")) {
                        return TARGET;
                    } else {
                        // study 以及其他方法 使用【代理类】执行，也就是李四
                        return PROXY;
                    }
                }
            };
        
            // 设置回调过滤器，可以通过此过滤器来指定哪个方法使用哪个回调进行处理
            enhancer.setCallbackFilter(filter);
            Student studentProxy = (Student) enhancer.create(of(String.class), of("李四"));
            studentProxy.eat();
            studentProxy.study();
            // 动态代理使学生类具备飞行的行为，很有趣！
            // Spring AOP 中把这种运行时为某个类对象增加额外功能的操作称为引入 Introduction。
            //  下文讲解 Spring AOP 时将会使用 Spring AOP API 形式实现目前 CGLIB API 的引入操作。
            // （织入作用的是连接点方法，引入作用的是类）
            ((Flyable) studentProxy).fly();
            System.out.println(studentProxy.getName());
        }
        ```

  * Spring AOP

    * Spring AOP vs AspectJ AOP
  
      * AspectJ 则基于静态编译技术实现 AOP，功能强大但相较复杂
      * Spring AOP 同时运用了 JDK 动态代理、CGLIB 动态代理

    * Spring AOP 重要类：**ProxyFactory**

      * ProxyFactory 代理工厂类，可以配置动态代理实现方式并生成代理对象

      * 示例
  
        ```java
        /**
         * Spring AOP
         *
         * <pre>
         * {@code
         *      Spring 基于 JDK 动态代理、CGLIB 动态代理两种技术来实现 AOP 框架，
         *      Spring AOP 被用来实现 Spring 框架的一些特性，为用户提供声明式的企业级服务、及 AOP 能力。
         *      例如：Spring 声明式事务管理、Spring IoC 容器对中间件的整合能力。
         *
         *      Spring AOP 属于基于动态代理技术实现的 AOP 框架，功能简单但有局限性。
         *      AspectJ 则是基于静态编译技术实现的 AOP 框架，功能强大但有一定的复杂性。
         *
         * AOP 术语
         *
         *  Join Point （连接点）：
         *      能够被切入点表达式匹配的关注点。
         *      Spring AOP 支持 Method Execution（方法执行）的连接点。
         *
         *      AspectJ 则更强大，它还支持：
         *          Method Call（方法调用）、Constructor Call (构造方法调用)、
         *          Constructor Execution (构造方法执行)、
         *          Field Get（字段获取）、Field Set（字段设置）、
         *          Static Initializer Execution（静态初始化执行）等一系列的连接点。
         *
         *  Pointcut （切入点）：
         *      选取符合匹配条件的连接点的规则断言，一般使用正则表达式来描述此断言。
         *      凡是与此切入点匹配的连接点，都会执行该切入点关联的通知。
         *
         *  Advice （通知）：
         *          在切入点所执行的具体操作逻辑方法。
         *
         *  Target （目标对象）：
         *          符合切入点匹配表达式的连接点所属的对象，该目标对象的连接点将被织入通知。
         *          Spring AOP 目标对象是运行时的类的对象实例
         *          AspectJ 目标对象是类的字节码文件、二进制 Jar 文件等
         *
         *  Weaving （织入）：
         *      将目标对象符合切入点表达式的连接点上追加通知的过程。
         *
         *      Spring AOP 织入是将运行时对象符合切入点的连接点增加通知，生成动态代理对象的过程。
         *      AspectJ 织入是将目标字节码符合切入点的连接点上增加通知，编译生成最终字节码的过程。
         *
         * Introduction（引入）：
         *      运行时为某个类对象增加额外功能的操作称为引入 Introduction。
         *      引入作用范围是类，织入作用范围是方法。
         *
         *  Aspect （切面）：
         *      汇聚许多切入点及切入点关联的多个通知，从而完成特定功能的域对象，被称作切面。
         *
         * }
         * </pre>
         *
         * @see <a href="https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-pointcuts-examples">Spring AOP 切点表达式</>
         */
        private static void springAOP() {
            // 注册当前引导类作为 Configuration Class
            ConfigurableApplicationContext context = new AnnotationConfigApplicationContext(DynamicProxyBootstrap.class);
            // 注意：
            //   Student 实现了 Human 接口，默认使用 JDK 动态代理生成的代理对象类型只能为 Human，强制转换将会报 ClassCastException 异常
            // 解决办法：
            //   将 @EnableAspectJAutoProxy 注解 proxyTargetClass 设置为 true，强制使用 CGLIB 动态代理
            Student student = (Student) context.getBean("student");
            student.giveASpeech();
        
            System.out.println("\n-----Spring AOP 实现引入，让 Student 具备飞行能力-------\n");
        
            /*
             * 此处本可以直接使用 new 对象，不用容器中的 student、plane 单例 Bean
             * 但为了验证经过 CGLIB 代理过后的对象是否还能继续被 CGLIB 动态代理，故注释。
             * 结果：
             *      经过 CGLIB 提升的实例，是可以继续被 CGLIB、JDK 动态代理
             */
            //Student stu = new Student("王五");
            //Plane plane = new Plane("B737", stu);
        
            // 使用容器中的 CGLIB 代理的 Plane
            Plane plane = (Plane) context.getBean("plane");
            // 使用容器中的 CGLIB 代理的 Student
            ProxyFactory proxyFactory = new ProxyFactory(student);
        
            // 指定 CGLIB 动态代理，否则会由于 Student 有接口走 JDK 动态代理，使得代理对象无法转换成 Student 类型
            proxyFactory.setProxyTargetClass(true);
            // 动态引人 Flyable 飞行能力
            proxyFactory.addInterface(Flyable.class);
            // 添加通知，此处使用代理引入拦截器通知类型
            proxyFactory.addAdvice(new DelegatingIntroductionInterceptor(plane));
            Human studentProxy = (Human) proxyFactory.getProxy();
            // 由于被引入的对象 plane 是容器中的 Advised 被通知对象，故此处执 fly 方法会执行所有切入点关联的通知
            ((Flyable) studentProxy).fly();
        
            context.close();
        }
        ```

### **Unsafe 类**

* 内存分配

  * 典型使用案例：**DirectByteBuffer**

    ```java
    DirectByteBuffer(int cap) { // package-private
    
        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);
    
        long base = 0;
        try {
            // 分配内存并返回基地址
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        // 内存初始化
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        // 跟踪 DirectByteBuffer 对象的垃圾回收，以实现堆外内存释放
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
    ```

  * `allocateMemory(long bytes)` 直接申请 **堆外内存**

* 内存屏障

  * Java 8 引入 3 个内存屏障函数，隔离操作系统对内存屏障的实现差异

    * `loadFence()` 读屏障，屏障前后的 load 操作不允许越过屏障
    * `storeFence()` 写屏障，屏障前后的 store 操作不允许越过屏障
    * `fullFence()` 全屏障，屏障前后的 load、store 都不被允许越过屏障

  * 典型使用案例：Java 8 引入的 **StampedLock** 乐观读锁

    ```java
    public boolean validate(long stamp) {
       U.loadFence();
       return (stamp & SBITS) == (state & SBITS);
    }
    ```

* 对象操作

  * 对象属性操作
    * `unsafe.objectFieldOffset(Field field)` 获得对象属性在对象的偏移量
    * `unsafe.getInt(Object, offset)` 获得对象某个属性的值
    * `unsafe.putInt(Object, offset, value)` 设置对象某个属性的值为 value
  * 对象实例化
    * `unsafe.allocateInstance(Class clazz)` 获得指定类型的类的实例，绕过对象初始化、私有构造器

* 数组操作

  * `arrayBaseOffset(Class<?> arrayClass)` 返回数组第一个元素的偏移地址
  * `arrayIndexScale(Class<?> arrayClass)` 返回数组第一个元素占用的大小

* CAS 操作

  * `compareAndSwapObject(Object o, long offset,  Object expected, Object update)`
  * `compareAndSwapInt(Object o, long offset, int expected,int update)`
  * `compareAndSwapLong(Object o, long offset, long expected, long update)`

* 线程调度

  * `park`、`unpark`、`monitorEnter`、`monitorExit`、`tryMonitorEnter`方法进行线程调度

    ```java
    //取消阻塞线程
    public native void unpark(Object thread);
    //阻塞线程
    public native void park(boolean isAbsolute, long time);
    //获得对象锁（可重入锁）
    @Deprecated
    public native void monitorEnter(Object o);
    //释放对象锁
    @Deprecated
    public native void monitorExit(Object o);
    //尝试获取对象锁
    @Deprecated
    public native boolean tryMonitorEnter(Object o);
    ```

* Class 操作

  * 静态变量操作

    ```java
    //获取静态属性的偏移量
    public native long staticFieldOffset(Field f);
    //获取静态属性的对象指针
    public native Object staticFieldBase(Field f);
    //判断类是否需要初始化（用于获取类的静态属性前进行检测）
    public native boolean shouldBeInitialized(Class<?> c);
    ```

  * 类加载操作

    * 定义一个 Class

      ```java
      public native Class<?> defineClass(String name, byte[] b, int off, int len, ClassLoader loader,ProtectionDomain protectionDomain);
      ```

    * 定义一个匿名类

      ```java
      public native Class<?> defineAnonymousClass(Class<?> hostClass, byte[] data, Object[] cpPatches);
      ```

      用于 Lambda 表达式定义相应的函数式接口匿名类，JDK 15 指出该方法可能被废弃

* 系统信息

  * 获得当前操作系统相关信息

    ```java
    //返回系统指针的大小。返回值为4（32位系统）或 8（64位系统）。
    public native int addressSize();
    //内存页的大小，此值为2的幂次方。
    public native int pageSize();
    ```

### SPI 相关

* SPI 与 API 关系：SPI 定义的接口一般在调用方，API 定义的接口一般在实现方。
* Java SPI 
  * **`java.util.ServiceLoader`**
  * 约定规则：`META-INF\services` 文件夹下放入 SPI 接口全限定名的文件，里面包含实现类的全限定名

### Java 语法糖

* Java 7 `switch` 语句支持 `String`
* Java 5 泛型、枚举
* 自动装箱、拆箱
* 可变长参数
* 内部类
* 条件编译
* 断言
* 数值字面量
* for-each
* try-with-resource
* Lambda 表达式

## 集合

* 集合框架

  ![集合框架总览图](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jtsn/java-collection-hierarchy.png)

  [引用图源-JavaGuide](https://oss.javaguide.cn/github/javaguide/java/collection/java-collection-hierarchy.png)

* 四种容器的区别

  * `List` 存储有序、可重复
    * `ArrayList` 底层 `Object[]` 数组
    * `Vector` 底层 `Object[]` 数组
    * `LinkedList`
      * JDK 1.6 双向循环链表
      * JDK 1.7 双向链表
  * `Set` 不可重复
    * `HashSet` 底层 `HashMap` 实现，无法记录插入序
    * `LinkedHashSet` 底层 `LinkedHashMap` 实现，可以记录插入顺序
    * `TreeSet` 底层***红黑树***，根据元素大小排序
  * `Queue` 按排队规则确定先后，存储有序、可重复
    * `PriorityQueue` 底层 `Object[]` 数组实现小顶堆
    * `DelayQueue` 底层 `PriorityQueue` 延迟队列
    * `ArrayDeque` 底层可扩容动态双向数组
  * `Map` 键值对存储、键无序不可重复、值无序可重复、一个键最多映射一个值
    * `HashMap`
      * JDK 1.8 前，底层***数组+链表***实现，数组为主体，链表解决哈希冲突（拉链法）
      * JDK 1.8 后，链表长度 `默认 ≥8` 且数组长度 `≥64` 时，链表转红黑树，否则扩容
    * `LinkedHashMap`
      * 由 `HashMap` 实现，总体底层与 `HashMap` 一致
      * 增加***双向链表***，记录键值对的插入顺序
    * `Hashtable` 底层***数组+链表***实现，线程安全
    * `TreeMap` 底层***红黑树***（自平衡的有序二叉树） 

* 为什么使用集合

  较比数组，支持**不确定大小**、**不同类型**的元素、具备多样化的操作、支持泛型、内建算法。

* 集合判空：使用 `isEmpty()` 而不是 `size()==0`, 后者很多 JUC 包下的集合复杂度不为 O(1)

* `Collectors.toMap()` 时 value 为 `null` 时会抛 NPE 异常

* 不要在 foreach 循环里进行元素的新增删除，删除请使用迭代器的删除方法，并发操作对迭代器加锁

* Java 8 支持 `Collection.removeIf()` 来删除满足特定条件的元素，避免手写迭代遍历

* JU 包下的集合类都是 fail-fast，JUC 包下的都是 fail-safe

* 利用 `Set` 元素唯一特性去重，比使用 `List` 的 `contains()` 进行遍历去重更高效

* 集合转数组必须使用 `toArray(T[] array)` 传入类型完全一致、长度为 0 的空数组

* 数组转集合 `Arrays.asList()` 不能使用修改集合的相关方法，会抛出不支持操作异常

  * 该方法为泛型方法，传递的数组是对象数组，基本类型数组需要变成装箱类数组使用
  * 否则转换后的列表元素是基本类型数组对象，而非基本类型对象
  * 使用 `new ArrayList<>(Arrays.asList(...))` 或 `Arrays.stream(array).collect(Collectors.toList())` 可以支持全功能的列表
  * Java 9 也可采用 `List.of(array)` 获得全功能的列表
  * 当然 Guava、Commons Collections 第三方工具类也可以

### List

* `ArrayList` vs 数组

  * `ArrayList` 动态扩缩容、支持泛型、只存对象（基本类型需包装）、提供便利的操作方法、无需指定长度
  * 数组 可存基本数据类型、需指定数组大小、不可改变长度

* `ArrayList` vs `Vector`

  底层都是 `Object[]` 存储，前者线程不安全，后者线程安全

* `Vector` vs `Stack`

  两者都使用 *synchronized* 来保证线程安全，后者继承 `Vector` 且保证 FIFO。两者都不再推荐使用。

* `ArrayList` 支持添加 `null` 元素，但不推荐

* `ArrayList` 操作复杂度

  * 插入：头插 **O(n)**， 尾插 **O(1)**（扩容时变成 **O(n)**），指定位置插 **O(n)**
  * 删除：头删 **O(n)**，尾删 **O(1)**，指定位置删除 **O(n)**

* `LinkedList` 操作复杂度

  * 头插/删、尾插/删：**O(1)**
  * 指定位置插/删：**O(n)**

* `ArrayList` vs `LinkedList`

  * `ArrayList` 支持随机访问（利于二分查找）、内存浪费在结尾预留容量空间（**推荐**）
  * `LinkedList` 头尾插删 **O(1)**、内存浪费在每个元素多余的指针

* `ArrayList` 扩容分析

  * JDK 6 调用无参构造方法时，直接创建长度为 10 的对象数组

  * JDK 6 之后无参构造方法时，先预置长度为 0 的占位数组，真正添加元素在分配容量

  * `grow(int)` 为扩容核心方法，每次扩容原由容量的 **1.5** 倍

    ```java
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    ```

    默认最大容量：

    ```java
    /**
     * 要分配的最大数组大小
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    ```

    执行 `hugeCapacity()` 最大扩容后容量为：`Integer.MAX_VALUE`

  * `Arrays.copyOf()` 与 `System.arraycopy()` 区别

    前者内部使用后者，前者内部创建新目标数组，后者可以自定义指定目标数组（方便数组本身的拷贝操作）

  * `ensureCapacity(int minCapacity)` 单次新增较大元素时，**此方法一步到位扩容，减少中间多次的扩容**

* `CopyOnWriteArrayList`

  * JDK 5 引入的并发安全的 `List`, 属于 JUC 包内，底层使用 `ReentrantLock` 在修改时加锁
  * 该链表没有扩容方法，只在写时进行增加 1 个长度，故 `size()` 方法返回当前数组长度即可 
  * 核心采用**写时复制**策略
    * 优点：写时不干阻塞读操作，读性能提升
    * 缺点：写时拷贝，内存占用大、写开销大、数据一致性问题
  * 适用场景：读多写少、链表不大

### Set

* 无序性不等于随机性，它指数据存储不按照数组索引顺序添加，而是根据数据哈希值决定
* 不可重复性是指按 `equals()` 判断时返回 `false`，需同时重写 `equals()` 及 `hashCode()` 方法
* `HashSet`、`LinkedHashSet`、`TreeSet` 三者区别
  * 三者都保证元素唯一、且非线程安全
  * `HashSet` 底层是哈希表
  * `LinkedHashSet` 底层是链表和哈希表，满足 FIFO
  * `TreeSet` 底层红黑树，元素有序，支持自然排序与定制排序

### Queue

* `Queue` vs `Deque`
  * `Queue` 单端队列，一端插入另一端删除，遵循 FIFO
  * `Deque` 双端队列，双端均可插入或删除
  * 容量问题抛异常操作：`add(E)`、`remove()`、`element(E)`
  * 容量问题会阻塞操作：`put(E)`、`take()`、`offer(E, timeout, unit)`、`poll(e, timeout, unit)`
  * 容量问题返回特殊值操作：`offer(E)`、`poll()`、`peek()`
  * `Deque` 多了队头队尾操作方法（xxxFirst、xxxLash）及模拟栈操作的 `push()`、`pop()`
* `ArrayDeque` vs `LinkedList`
  * `ArrayDeque` 不存储 `null`、可变长数组+双指针实现、JDK 6 引入、插入 **O(n)** 存在扩容开销
  * `LinkedList` 支持存储 `null`、链表实现、JDK 1.2 引入、无需扩容但每次插入需申请新堆空间
  * 性能上，ArrayDeque 总体大于 LinkedList，另前者很轻松模拟栈操作
* `PriorityQueue`
  * 完全二叉堆数据结构，底层使用可变长数组存储数据
  * 左子节点下标 = 父节点下标 * 2 + 1
  * 右子节点下标 = 父节点下标 * 2 + 2
  * 父节点下标 = (子节点下标 -1) / 2
  * 小顶堆，即：任意非叶子节点权值都不大于其左右子节点的权值
  * 通过**堆元素上浮下沉**实现 **O(logn)** 时间复杂度的插入与删除堆顶元素
  * 非线程安全、不支持存储 `null` 及不可比较对象，默认小顶堆，支持定制 `Comparator` 实现不同优先级
  * 典型运用：堆排序、求第 K 大的数、带权图的遍历
* `BlockingQueue`
  * 当队列没有满足操作要求时会阻塞调用线程，此种队列成为阻塞队列
  * 常见阻塞队列：
    * `ArrayBlockingQueue` 
      * 数组实现的有界阻塞队列
      * 创建时指定容量大小
      * 支持公平非公平方式的锁
        * 底层采用 `ReentrantLock` 在空、非空两个条件锁下等待或唤醒
    * `LinkedBlockingQueue` 
      * 单向链表实现的可选有界阻塞队列
      * 创建时可指定容量大小，不指定 `Integer.MAX_VALUE`
      * 仅支持非公平锁访问
    * `PriorityBlockingQueue`
      * 优先级排序的无界阻塞队列
      * 元素不允许 `null` 且必须实现 `Comparable` 接口或指定 `Comparator`
    * `SynchronousQueue`
      * 至多只有一个元素的同步队列
      * 插入时需等元素被删除，删除时需等元素被添加
    * `DelayQueue`
      * 延迟队列，元素只有到指定的延时时间，才能够从队列中取出
* `ArrayBlockingQueue` vs `LinkedBlockingQueue`
  * 底层实现：前者数组实现，后者链表实现
  * 有界性：前者有界必须指定大小，后者可以有界指定大小，也可以不指定大小，即 `Integer.MAX_VALUE`
  * 锁是否分离：
    * 前者生产消费使用同一个锁
    * 后者生产使用 `putLock`，消费使用 `takeLock`,防止消费生产竞争
  * 内存占用：前者提前分配数组内存，后者动态分配链表节点
* `ConcurrentLinkedQueue`
  * 无界、线程安全的基于链表、CAS 原子操作的非阻塞队列

### Map

* `HashMap` vs `Hashtable`

  * 前者非线程安全，后者线程安全
  * 前者效率高一点
  * 前者支持一个 `null` 键，多个 `null` 值，后者不支持 `null` 键与值，抛 NPE
  * 前者初始化大小 16，2 倍扩容，指定大小将修改为最接近的 2 的幂
  * 后者初始化大小 11, 2n+1 方式扩容，指定大小容量时就为指定的大小
  * 前者 JDK 8 后哈希冲突，链表长度 `默认 ≥8` 且数组长度 `≥64` 时，链表转红黑树，否则扩容

* `HashMap` vs `HashSet`

  `HashSet` 是值为常量共享对象的 `HashMap`, 只用键来存储对象。

  `HashSet` 添加元素是直接调用 `HashMap` 的 `put()` 方法，不会先判断是否存在

  而是判断调用后返回值是否为空，从而表示插入位置是否原本有值

* `HashMap` vs `TreeMap`

  * 后者具备集合内元素搜索能力、对键升序排序（可定制排序）

* `HashMap` 底层实现

  * 扰动函数 `hash()` JDK 8 较之前版本扰动变少（1次 vs 4次）
  * JDK 8 之后，链表长度 `默认 ≥8` 且数组长度 `≥64` 时，链表转红黑树，否则优先数组扩容
  * JDK 8 之后，红黑树节点数 `≤6` 将降为链表
  * `HashMap` 最大桶长度（**区别于容量**）为 `MAXIMUM_CAPACITY = 1 << 30`
  * 扩容条件为元素数量超过当前容量的 75%， `DEFAULT_LOAD_FACTOR = 0.75f` （疏密度控制）
  * JDK 8 之后的扩容 `resize()` **算法无需重新计算哈希**，只需与原容量取余是否为 0
    * hash & oldCap == 0，保持原由索引位置不动
    * hash & oldCap != 0，新索引位 = 旧索引位 + oldCap
    * 参考：[Discuss HashMap resizing in JDK 1.8](https://www.itqit.com/programming/250.html)
  * 红黑树解决二叉树极限情况会退化成线性结构
  * 长度为 2 的幂次方，因为数组下标是优化的取模运算：`(n-1) & hash`，要求 n 为 2 的幂次方才成立

* `HashMap` 线程不安全

  * JDK 7 及之前在多线程下扩容可能出现死循环
  * JDK 8 下由尾插取代头插避免多线程并发扩容出现死循环
  * 但多线程下还是会存在数据覆盖，故推荐多线程环境用 **ConcurrentHashMap**

* `HashMap` [七种遍历方式](https://javaguide.cn/java/collection/java-collection-questions-02.html#hashmap-%E5%B8%B8%E8%A7%81%E7%9A%84%E9%81%8D%E5%8E%86%E6%96%B9%E5%BC%8F)

  * Iterator EntrySet
  * Iterator KeySet
  * For EntrySet
  * For KeySet
  * Lambda
  * Stream API
  * Parallel Stream API

  结论：

  * 不存在阻塞时，`Iterator | For EntrySet > Stream > Lambda > KeySet > Parrallel Stream`
  * 存在阻塞时，Parrallel Stream 性能最高

* `ConcurrentHashMap` vs `Hashtable`

  * 都是线程安全

  * 底层数据结构，前者跟随 `HashMap` 在 JDK 8 追加红黑树的转换，后者没有

  * 前者 JDK 8 之后，丢弃了 `Segment` 概念，直接用 `Node` 数组+链表/红黑树实现

    JDK 8 之前，采用 Segment 数组 + HashEntry 数组 + 链表 结构，锁加在 Segment 上

    默认 Segment 大小 16，即默认支持 16 个线程并发写，最大 `1 << 16` 即 65536 个并发

    属于 `ReentrantLock` 可重入锁的实现类

    JDK 8 之后，采用 Node 数组 + 链表/红黑树 结构，锁加在 Node 上

    并发控制使用 `synchronized` 与 CAS 来操作，锁粒度更加细（每个哈希桶，并发度高）

  * 后者则采用方法层级的 `synchronized` (同一个类上的锁，锁粒度大，并发度低)

* `ConcurrentHashMap` 为何不允许 `null` 的键与值

  * 避免二义性，若键为空，无法确认该键是存在于 Map 中，还是根本没有这个键
  * 多线程环境下无法通过 `containsKey(key)` 来判断键是否存在，就无法解决二义性问题

* `ConcurrentHashMap` 不能保证复合操作的原子性，需结合 CAS 来实现原子性

  * `putIfAbsent`、`compute`、`computeIfAbsent` 、`computeIfPresent`、`merge`等原子方法
  * `putIfAbsent` 当且仅当键不存在时，才将键值放入 Map
  * 使用 CAS 乐观锁替代加锁同步，使并发度更高

* `LinkedHashMap`

  * 继承 `HashMap` 后，维护了一条双向链表

  * 支持按插入顺序迭代（默认），按元素访问顺序排序

  * 遍历效率与元素个数成正比，比与容量成正比的 `HashMap` 迭代效率高

  * `accessOrder=false` 按插入顺序迭代，否则按访问顺序遍历，最近访问排在链表末尾

  * 可被用来实现 LRU 缓存，重写 `removeEldestEntry()` 方法， 头部删除最老未访问的元素 

    ```java
    /**
     * 判断size超过容量时返回true，告知LinkedHashMap移除最老的缓存项(即链表的第一个元素)
     */
    protected boolean removeEldestEntry(Map.Entry < K, V > eldest) {
        return size() > capacity;
    }
    ```

    

### Collections 工具类

* 排序操作

  ```java
  void reverse(List list)//反转
  void shuffle(List list)//随机排序
  void sort(List list)//按自然排序的升序排序
  void sort(List list, Comparator c)//定制排序，由Comparator控制排序逻辑
  void swap(List list, int i , int j)//交换两个索引位置的元素
  void rotate(List list, int distance)//旋转。当distance为正数时，将list后distance个元素整体移到前面。当distance为负数时，将 list的前distance个元素整体移到后面
  ```

* 查找替换操作

  ```java
  int binarySearch(List list, Object key)//对List进行二分查找，返回索引，注意List必须是有序的
  int max(Collection coll)//根据元素的自然顺序，返回最大的元素。 类比int min(Collection coll)
  int max(Collection coll, Comparator c)//根据定制排序，返回最大元素，排序规则由Comparatator类控制。类比int min(Collection coll, Comparator c)
  void fill(List list, Object obj)//用指定的元素代替指定list中的所有元素
  int frequency(Collection c, Object o)//统计元素出现次数
  int indexOfSubList(List list, List target)//统计target在list中第一次出现的索引，找不到则返回-1，类比int lastIndexOfSubList(List source, list target)
  boolean replaceAll(List list, Object oldVal, Object newVal)//用新元素替换旧元素
  ```

* 同步控制

  * `synchronizedXxx()` 方法，将指定集合封装成线程同步集合，保证多线程访问安全
  * 最好使用 JUC 包下的并发集合替代这些封装的同步集合

## 并发编程

### **线程概念**

* 进程与线程
  * 进程是程序被操作系统分配资源及执行的基本单位
  * 线程是进程运行时最小执行单位，多个线程能够共享进程中的各项资源
  * Window 和 Linux 等主流操作系统中，Java 线程采用一对一模型（1 个用户线程对应 1 个内核线程）
* 并发与并行

  * 并发：两个及以上作业在同一时间段内被执行
  * 并行：两个及以上作业在同一时刻被执行
* 多线程意义

  * 线程是轻量级的进程，线程切换及调度成本比进程小，充分利用多核 CPU 并行执行减少切换开销
  * 单核时代
    * 多线程提高硬件资源（CPU、IO）的利用率，例如：IO阻塞时，切换另一个线程使用 CPU
    * 单核 CPU 运行计算密集型的任务，多线程并不能提高执行效率，反倒由于线程切换降低效率
    * 单核 CPU 运行 IO 密集型的任务，多线程可以提高任务的吞吐量
  * 多核时代：多线程提高进程利用多核 CPU 的能力，例如：多个线程并行计算加速单个任务的完成时间
* 多线程带来的问题

  * 线程安全性问题（资源可见性、共享竞争、一致性）
  * 内存泄漏、死锁
* 如何理解线程安全

  * 多线程环境下对同一个资源的访问是否能够保证正确性及一致性的问题，被叫做线程安全问题

### Thread 相关

* 线程创建

  * 就 Java 语言而言，创建线程只有一种，即：`new Thread()` 也包含实例化继承该类的子类
  * 线程是执行可运行的任务的容器，而可运行的任务被称为线程体，即：`Runable`
  * 创建线程体的方式那就五花八门：`Runable` 本身、`Callable`、Lambda 表达式等等

* 线程生命周期及状态

  * **NEW**：`new Thread()`
  * **RUNNABLE**：`start()`
  * **BLOCKED**：`获取锁失败`
  * **WAITING**：`waite()`、`join()`、`LockSupport.park()`
  * **TIME_WAITING**：`waite(long)`、`join(long)`、`sleep(long)`、`LockSupport.parkNanos(long)`
  * **TERMINATED**：执行完成

  ![线程状态转化图](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jtsn/640.png)
  
  [引用图源-JavaGuide](https://oss.javaguide.cn/github/javaguide/java/concurrent/640.png)
* 线程上下文切换

  * 线程执行过程包含一些运行条件和状态（程序计数器、栈信息）被称作上下文
  * 当线程从占用 CPU 转态退出时，就会发生上下文切换，切换发生在如下时机
    * 主动让出 CPU：`sleep()`、`wait()`、`yeild()`等
    * CPU 时间片用完
    * 调用产生系统中断的方法，如：阻塞类型的方法、访问 IO 设备
    * 被终止或结束运行

* 死锁

  * 线程由于等待某个资源的释放而被无限期的阻塞，被称为死锁
  * 产生死锁必要条件：
  * 互斥条件：资源任意时刻只由单个线程占有
  * 请求与保持：对已获得资源保持不放，请求另外资源
  * 不被剥夺：对已获得资源未使用完之前不被其他线程强行剥夺
  * 循环等待：若干线程形成头尾相接的等待资源循环
  * 预防与避免
    * 破坏产生死锁的必要条件即可
    * 破坏请求与保持：一次性请求所有资源
    * 破坏不被剥夺：对已获得资源，请求另外资源不得时，释放已获得资源
    * 破坏循环等待：按顺序申请及释放资源

* `sleep` vs `wait`

  * sleep 方法没释放锁，通常用于暂停当前线程执行，指定时间后自动恢复，属于 **Thread** 类静态本地方法
  * sleep 不涉及对象隐式锁，只单纯让当前线程暂停执行，故不需要获得对象锁
  * wait 方法释放锁，用于线程间交互通信，无参方法不能自动苏醒，需其他线程唤醒
  * wait 属 **Object** 类的本地方法，它释放当前线程占有所属对象上的隐式锁，故需在已占有锁时被调用


### **ThreadLocal 相关**

* ThreadLocal 变量一般都是 `private static` 修饰，为什么？[线程局部变量语义](https://reionchan.github.io/2017/02/01/corejava-v1-note-part2/#%E7%BA%BF%E7%A8%8B%E5%B1%80%E9%83%A8%E5%8F%98%E9%87%8F)

* ThreadLocal 类型的变量，其真实是存储在 Thread 对象实例中的 `ThreadLocal.ThreadLocalMap` 的 Map 中

* 其中 key 为当前 ThreadLocal 实例，value 为 ThreadLocal 中所设置的值

* 此处注意，key 为当前 ThreadLocal 的 **弱引用**， value 是强引用

* 强引用、软引用、弱引用、虚引用

  * 强引用：最普遍的引用类型，只要强引用存在，垃圾回收器不会回收这个对象，即时内存不足。

  * 软引用：描述有用但非必须的对象，内存不足时，垃圾回收器会回收软引用对象。

  * 弱引用：比软引用生命周期短，只能生存到下一次垃圾收集器发生之前，无论内存空间是否足够。

  * 虚引用：对象生存周期完全忽视是否被虚引用所引用，也无法通过该引用来访问对象的任何属性或函数

    仅用作跟踪引用对象被垃圾回收器回收的活动。


* ThreadLocal 内存泄漏问题如何导致

  * 由于 `ThreadLocalMap` 中的 Entry 中的 key 采取弱引用方式引用 `ThreadLocal` 类型的对象

    当该对象没有被强引用的情况下，垃圾回收时该 key 所引用的 `ThreadLocal` 对象将被清理，

    从而导致 key 为 `null` 的 Entry，该 Entry 的 value 却是强引用的，但是该 value 由于缺少 key

    而无法被访问，如果该线程一直不结束，就会造成内存泄漏。

    强引用链一直存在：thread 引用 -> threadLocalMap -> Entry[] -> (null, value)

  * 解决办法：
    * 本身方法的 `get()`、`set()` 方法中是会处理 key 为空的清理
    * 使用完线程本地变量后，调用其 `remove()` 方法，并将 threadLocal 变量设置为 `null`

* ThreadLocal 当设置的值为引用类型时，请确保该引用的对象不被其他线程共享或对象是线程安全的

  * 要么设置的对象是在当前线程 `new` 出的对象，不会被其他线程访问
  * 要么采取对象深拷贝技术，保证当前线程引用的对象是原始对象的拷贝，例如：利用序列化反序列化

* InheritableThreadLocal

  * ThreadLocal 变量无法交给子线程共享，需要由其子类 InheritableThreadLocal 来实现

  * InheritableThreadLocal 类型变量被存在 Thread 的 `inheritableThreadLocals` 变量中

    该变量同样是 `ThreadLocalMap` 类型，当当前线程创建子线程 `new Thread()` 时，

    会将本线程中 `inheritableThreadLocals` 保存的 **Entry 拷贝**给子线程 `inheritableThreadLocals` 变量

    这里仅是 Entry 拷贝，里面的 key 和 value 还是直接拿父线程的引用，而非对应对象的拷贝

* ThreadLocal 典型应用
  * 日志系统中的 `traceId` 在跨线程、跨服务间的流转传递

### **线程池**

* 管理一系列线程资源的池，有任务处理时直接从池中获取线程处理，完成后线程不销毁回到池中等待任务

* 优点

  * 降低资源消耗
  * 提高响应速度
  * 提高线程的可管理性

* 线程池创建

  * `ThreadPoolExecutor` 推荐，高度定制线程池参数

    ```java
    public ThreadPoolExecutor(
        //线程池的核心线程数量
        int corePoolSize,
        //线程池的最大线程数
        int maximumPoolSize,
        //当线程数大于核心线程数时，多余的空闲线程存活的最长时间
        long keepAliveTime,
        //时间单位
        TimeUnit unit,
        //任务队列，用来储存等待执行任务的队列
        BlockingQueue<Runnable> workQueue,
        //线程工厂，用来创建线程，一般默认即可
        ThreadFactory threadFactory,
        //拒绝策略，当提交的任务过多而不能及时处理时，我们可以定制策略来处理任务
        RejectedExecutionHandler handler
    )
    ```

    * 参数

      * **`corePoolSize`**: 任务队列未达到队列容量时，可以同时运行的线程最大数量
      * **`maximumPoolSize`**：任务队列达到队列容量时，可同时运行的最大线程数量
      * **`workQueue`**：新任务请求线程时若达到核心数量时，被放入此队列等待
      * **`keepAliveTime`**：池中大于核心线程时，空闲的多余线程等待该时间后才被销毁
      * **`unit`**：多余空闲线程存活时间单位
      * **`threadFactory`**：线程执行器创建线程时的线程工厂
      * **`handler`**：任务饱和时的执行策略

    * 线程池饱和策略

      当前线程池运行线程达到 `maximumPoolSize` 且等待队列已放满时，对新任务的执行策略

      * **`AbortPolicy`**：抛 `RejectedExecutionException` 来拒绝新任务
      * **`CallerRunsPolicy`**：由当前提交任务的线程自身执行任务
      * **`DiscardPolicy`**：不处理新任务，直接丢弃
      * **`DiscardOldestPolicy`**：新任务替换最早未处理的任务请求

    * 线程池常用阻塞队列

      * 无界队列

        * `LinkedBlockingQueue` 线程池线程最大时，理论上任务队列永不会满

      * 同步队列

        * `SynchronousQueue` 无容量队列，有空闲线程给线程处理，无线程创建线程处理

      * 延迟阻塞队列

        * `DelayedWorkQueue` 延迟线程池使用的队列，越临近当前时间的任务先出列

          队列满扩容原来容量的 1/2，容量最大 `Integer.MAX_VALUE`，理论延迟线程

          只能创建核心线程数的线程，因为本队列理论无限大，不激发核心线程外的创建

    * 线程池命名

      * 利用 Guava 的 `ThreadFactoryBuilder`

        ```java
        ThreadFactory threadFactory = new ThreadFactoryBuilder()
         	.setNameFormat(threadNamePrefix + "-%d")
            .setDaemon(true).build();
        ```

      * 实现自己的 `ThreadFactory`

        ```java
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName(name + " [#" + threadNum.incrementAndGet() + "]");
            return t;
        }
        ```

    * 线程池大小设定

      * 公式

        `最佳线程数 = CPU 核心数 * (1 + 线程等待时间 / 线程计算时间)`

        计算密集型，线程等待时间与线程计算时间比值趋近 0，所以推荐与 CPU 核心数相同的大小

        I/O密集型，线程等待时间与线程计算时间比值可能很大，避免过多创建线程，比值默认 1，

        所以推荐 CPU 核心数 2 倍的线程数

      * CPU 密集型任务：N + 1

        N 为 CPU 核心数，多出的一个线程为了防止线程缺页中断或其他原因 CPU 空闲时

        该线程可以利用 CPU 的空闲时间。一般：利用 CPU 计算能力的任务被视作 CPU 密集型任务

      * I/O 密集型任务：2N

        大部分时间处理 I/O 交互，而 I/O 时间段内不会占用 CPU，此时可以交给其他线程使用

        一般：涉及网络读取、文件读取的任务被视作 I/O 密集型任务

      * 动态线程池

        * [《Java 线程池实现原理及其在美团业务中的实践》](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)
        * 利用可变容量的队列来实现动态修改线程池
        * Hippo4j
        * Dynamic TP

  * `Executors` 创建多种预置类型的 ThreadPoolExecutor

    * 预置类型

      * **`FixedThreadPool`**：固定线程数量的线程池，无可用任务放入无界阻塞队列
      * **`SingleThreadExecutor`** ：单线程的线程池，FIFO，无可以放入无界阻塞队列
      * **`CachedThreadPool`**：线程数变化的线程池，初始 0，无可用创建，60 秒空闲线程销毁
      * **`ScheduledThreadPool`**：接受指定延迟时间或定期执行任务的线程池

    * 使用预置的弊端

      * `FixedThreadPool`、`SingleThreadExecutor` 使用无界队列，可能大量堆积 OOM

      * `CachedThreadPool` 线程数无限制，同步队列，任务数多且执行慢，堆积大量线程 OOM

      * `ScheduledThreadPool`、`SingleThreadScheduledExecutor` 使用无界延迟阻塞队列

        可能堆积大量请求在队列，导致 OOM


### Future 相关

* 采取异步思想，用在一些需要执行耗时任务的场景，避免程序一直原地等待任务完成。

  ```java
  // V 代表了Future执行的任务返回值的类型
  public interface Future<V> {
      // 取消任务执行
      // 成功取消返回 true，否则返回 false
      boolean cancel(boolean mayInterruptIfRunning);
      // 判断任务是否被取消
      boolean isCancelled();
      // 判断任务是否已经执行完成
      boolean isDone();
      // 获取任务执行结果
      V get() throws InterruptedException, ExecutionException;
      // 指定时间内没有返回计算结果就抛出 TimeOutException 异常
      V get(long timeout, TimeUnit unit) 
          throws InterruptedException, ExecutionException, TimeoutExceptio
  }
  ```

* CompletableFuture
  * 弥补 `Future` 不足，增加异步任务编排、支持函数式编程 

### **JMM（Java 内存模型）**

* JMM 定义多线程环境下共享变量在线程间的可见性

* CPU 缓存模型
  * 解决 CPU 处理速度与内存处理速度不对等的问题
  * 现代 CPU 通常分三级缓存：L1、L2、L3，一般 L1、L2 由单核独享，L3 多核共享
  * 多个核心加载共享变量到各自核心独享缓存中操作时，会一起缓存不一致问题
  * 现代 CPU 解决缓存不一致协议很多，例如：MESI 协议、总线锁
  * 操作系统通过 **内存模型** 定义一系列规范来解决内存缓存不一致等问题
  * 不同操作系统都有其特定的内存模型，来解决各种硬件资源虚拟化到内存后的缓存不一致问题

* 指令重排
  * 为了提升执行速度/性能，系统执行指令时不完全按照代码顺序的现象，被称作指令重排

  * 指令重排发生时机：编译器优化时指令重排、CPU 多条指令重叠并行执行时的指令重排

  * Java 源代码可能发生指令重排的时期
    * 编译器优化重排（静态编译器优化、即时编译器优化 JIT）
    * CPU 指令并行重排
    * 内存系统重排

  * **指令重排可以保证串行语义一致，但没有义务保证多线程间语义一致，故也会导致一致性问题**

  * 编译器指令重排可以通过禁止特定编译器的指令重排优化来解决

  * CPU、内存发生的指令重排，可以通过设置 **内存屏障** 来保证指令执行的有序性

    > 参考阅读：
    >
    > * [缓存一致性](https://zhuanlan.zhihu.com/p/123926004)
    >
    > * [内存屏障](https://zhuanlan.zhihu.com/p/125549632?utm_source=ZHShareTargetIDMore&utm_medium=social&utm_oi=1224314960559403008)

  * JMM 定义的内存屏障类别

    * **`LoadLoad`**： 确保屏障前的读数据动作发生在屏障后的读及之后所有读之前

    * **`StoreStore`**：确保屏障前写的数据在屏障后写及之后所有写可见

    * **`LoadStore`**: 确保屏障前的读数据动作发生在屏障后写及之后所有写之前

    * **`StoreLoad`**：确保屏障前的写的数据在屏障后的读及之后所有读可见

      * 此为全能型屏障，大多处理器支持，但此屏障开销昂贵，一般需处理器将

        其对应的写缓冲里的缓存行刷新到主存中

* 为什么需要 JMM

  * JMM 为了隔离不同操作系统特定内存模型，定义了跨平台统一的虚拟机级别的内存模型规范 [JSR-133](https://www.cs.umd.edu/~pugh/java/memoryModel/CommunityReview.pdf)
  * JMM 规范线程、主存关系、字节码到 CPU 指令转化时遵循的并发原则，简化并发编程，增强移植性
  * JMM 的 *happens-before* 原则保证不同平台下的虚拟机对程序翻译执行语义的一致性
    * 语义一致性包括：指令执行顺序一致性保证、程序执行后结果一致、缓存一致性

* Java 内存区域和 JMM 有何区别
  * Java 内存区域定义了 JVM 在运行时如何对内存进行分区存储不同的运行时数据
  * JMM 与并发编程相关，它抽象了线程与主存之间关系来解决线程间共享变量的可见性等一些列问题

* JMM 所抽象的线程及主存关系

  * JMM 所定义的抽象

    * 主内存抽象：所有线程创建的实例对象都放在主内存中，包括：成员变量、局部变量、类、常量
    * 线程本地内存抽象：每个线程私有的本地内存，该内存存储对主内存中共享变量的本地副本
    * 线程本地内存中的副本对其他线程不可见，需将副本值刷新到主存才具备对其他线程的可见
    * 线程本地内存其实是逻辑概念，线程在 CPU 某个核心运行时，该逻辑区可理解为：
      * 运行此线程的 CPU 核心的 L1、L2 缓存或写缓冲区等

  * JMM 规定，线程可以把主内存共享变量的副本保存在 **本地内存** （本线程独享区，例如：L1、L2）

    对该共享变量的读写只在此本地内存的副本中进行，这将导致其他线程对该共享变量的副本不一致

  * JMM 规定 ***happens-before*** 原则来描述两个操作之间的内存可见性，它平衡程序员和编译器、处理器

    之间的冲突。程序员追求强内存模型来简化并发编程的复杂性，编译器、处理器追求弱内存模型，

    在更少的约束下尽可能的去优化性能。JMM 的该原则保证不会发生改变程序执行结果的指令重排。

    *happens-before* 具体描述：

    * 某操作 *happens-before* 另一操作，那前操作的执行结果将对第二个操作可见，且前操作执行顺序排在第二个操作之前。
    * 两操作存在 *happens-before* 关系，只要重排序后执行结果与按规则执行一致，也将被允许

    具体规则：

    * 程序顺序规则：线程内，按照代码顺序，书写在前的操作 *happens-before* 书写在后的操作
    * 解锁规则：解锁 *happens-before* 加锁
    * `volatile` 变量规则：对该变量写操作 *happens-before* 于后面对该变量的读操作
    * 传递规则：A *happens-before* B，B *happens-before* C，那么 A *happens-before* C
    * 线程启动规则：**Thread** 对象的 `start()` 方法 *happens-before* 于此线程的每一个操作

* `volatile` 关键字

  * 指示 JVM 其修饰的变量是共享且不稳定，***每次使用变量都要到主存中进行读取***

    > 此处说每次使用变量都要到主存中读取，只是描述最终达到的效果是对其更改所有线程都可感知
    >
    > 但现代 CPU 对实现该效果采用了 **MESI + 写缓存 + 失效队列** 等缓存行同步技术
    >
    > 可以做到不同 CPU 核心的缓存行之间通信获取变量，并非一定要到主存加载数据
    >
    > 详细参见：[缓存一致性](https://zhuanlan.zhihu.com/p/123926004)

  * 保证线程间对该变量修改的可见性，而且遵循 *happens-before* 原则，防止指令重排

  * `synchronized` 关键字既保证可见性也保证原子性

  * 实际应用案例（双重校验锁实现单例）

    ```java
    public class Singleton {
    
        private volatile static Singleton uniqueInstance;
    
        private Singleton() {
        }
    
        public  static Singleton getUniqueInstance() {
           //先判断对象是否已经实例过，没有实例化过才进入加锁代码
            if (uniqueInstance == null) {
                //类对象加锁
                synchronized (Singleton.class) {
                    if (uniqueInstance == null) {
                        uniqueInstance = new Singleton();
                    }
                }
            }
            return uniqueInstance;
        }
    }
    ```

    * 双重检查：
      * 第一次检查：性能优化考虑，如果不为空免去 `synchronized` 加锁操作
      * 第二次检查：获得锁后再次检查，防止并发环境下前一个已获得锁的线程已生成实例
    * volatile 修饰语义：
      * 保证改实例在线程间的可见性
      * 防止 `uniqueInstance = new Singleton()` 发生指令重排（先赋值后初始化）
      * 若发生先赋值后初始化，那么有机率其他线程或得到半初始化的单例对象

* 乐观锁 vs 悲观锁

  * 悲观锁：总是假设最坏情况，每次获取资源操作都会上锁
    * `synchronized`、`ReentrantLock` 都属于此类锁
    * 适用写多、竞争激烈，避免频繁失败与重试
  * 乐观锁：总是假设最好情况，每次获取资源无需加锁与等待，只在提交修改验证资源变化
    * 使用 **CAS** 实现的 `java.util.concurrent.atomic` 包下都是乐观锁实现
    * 适用读多，竞争较少，避免频繁加锁
    * 乐观锁可以使用 CAS、版本号机制实现
  * CAS
    * CAS 底层依赖一条 CPU 的原子指令
    * 涉及三个操作数：V 要更新的变量，E 预期值，N 拟写入的新值
    * CAS 常见问题
      * ABA 问题，采取增加**版本号**、**时间戳**来解决
      * 自旋操作来重试，长时间不成功 CPU 开销会很大
      * 只能保证单个共享变量的原子操作，可以使用原子引用类 `AtomicReference` 整合多个变量

  * 原子类

    * 基本类型，使用原子方式更新基本类型

      * `AtomicInteger`：整型原子类
      * `AtomicLong`：长整型原子类
      * `AtomicBoolean`：布尔型原子类

    * 数组类型，使用原子方式更新数组里的某个元素

      * `AtomicIntegerArray`：整型数组原子类
      * `AtomicLongArray`：长整型数组原子类
      * `AtomicReferenceArray`：引用类型数组原子类

    * 引用类型

      * `AtomicReference`：引用类型原子类

      * `AtomicMarkableReference`：原子更新带有标记的引用类型。

        该类将 boolean 标记与引用关联起来

      * `AtomicStampedReference`：原子更新带有版本号的引用类型

        该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。

    * 对象的属性修改类型

      * `AtomicIntegerFieldUpdater`:原子更新整型字段的更新器
      * `AtomicLongFieldUpdater`：原子更新长整型字段的更新器
      * `AtomicReferenceFieldUpdater`：原子更新引用类型里的字段

### **`synchronized` 关键字**

* 早期版本直接使用操作系统底层的 *Mutex Lock* 实现，属重量级锁，线程挂起或唤醒需要系统级调用
* Java 6 之后增加锁膨胀等优化，效率提升许多
  * 锁的四种状态：无锁、偏向锁、轻量级锁、重量级锁
* 其引入的偏向锁机制并没有带来多少性能提升
  * JDK 15 中默认关闭，开启参数 `-XX:+UseBiasedLock`
  * JDK 18 中已经废弃偏向锁，无法参数开启
* 构造方法无法使用该关键字修饰，原本构造方法已是线程安全
* 底层原理
  * 同步代码块，使用字节码指令 `monitorenter`、`monitorexit` 两条指令来实现
  * 修饰方法时，采用 `ACC_SYNCHRONIZED` 修饰符来标识方法为同步方法
  * 更底层采用 C++ 来实现，每个对象内置一个 **ObjectMonitor** 对象
  * `wait`、`notify` 等方法的调用也和此监视器对象相关，故需在同步代码块中才能被调用
* `synchronized` vs `volatile`
  * `volatile` 线程同步轻量级实现，只能用在变量上控制变量在多线程下的可见性，不保证原子性
  * `synchronized` 可以修饰方法及代码块，即保证可见性也保证原子性，解决多线程之间资源的同步

### ReentrantLock

* 实现了 `Lock` 接口的可重入独占式的锁，与 `synchronized` 关键字类似，但更灵活强大
* 增加轮询、超时、中断、公平锁、非公平锁等功能
* 公平锁：锁释放后，先申请的锁先得到锁，性能较差，为保证时间上绝对顺序，上下文切换频繁
* 非公平锁：锁释放后，等待的所有线程采取抢占式得到锁，随机或优先级，可能导致线程无法获的锁
* 底层基于 AQS 来实现
* ReentrantLock vs  `synchronized`
  * 都是可重入，前者依赖 API 实现锁，后者依赖 JVM 实现
  * 前者提供更高级的功能，如：获取锁等待可中断、公平锁、多条件选择性通知

### ReentrantReadWriteLock

* 实现 `ReadWriteLock` 接口的可重入读写锁，保证多线程读效率，也保证写操作线程安全
* 包含两把锁，`WriteLock` 独占写锁，`ReadLock` 共享读锁
* 底层也是 AQS 实现，同样支持公平锁与非公平锁
* 获得读锁情况下，是不能再获得写锁，获得写锁时，可以继续获得读锁
  * 读锁升级写锁会引起线程的争夺，而写锁降级为读锁不会
  * 若允许读锁升级写锁，那么两个同时获得读锁的线程都升级，但都释放自身读锁，造成死锁

### StampedLock

* JDK 8  引入的性能更好的读写锁，不可重入且不支持条件变量
* 它再获取锁时返回 `long` 型数据戳，用于锁释放参数，返回 0 表示获取锁失败
* 当前线程持有锁后再次获取锁时，会返回新的数据戳，故此锁不可重入
* 性能好原因
  * 乐观读机制比一般读写锁性能高，乐观读允许一个写线程获取写锁，故不会导致写线程阻塞
  * 读多写少时减少写线程等待时间，提高吞吐量
* 提供三种模式的读写控制模式
  * 写锁：独占锁，且不可重入
  * 读锁：共享锁，且不可重入
  * 乐观锁：允许多个线程获取乐观读锁，同时允许一个写线程获取写锁
* 底层原理
  * 它不直接实现 `Lock` 或 `ReadWriteLock` 接口，而是基于 [**CLH**](https://mp.weixin.qq.com/s/jEx-4XhNGOFdCo4Nou5tqg) 锁实现（AQS 同样是该原理）
  * CLH 锁是自旋锁的改良，属于隐式的链表队列，通过该队列进行线程的管理
  * 通过同步状态值 `state` 来标识锁的状态与类型

### **CLH 锁**

* 自旋锁

  * 不适用竞争激烈场景
  * 激烈场景可能带来锁饥饿问题、性能问题（锁状态中心化，CPU高速缓存频繁同步）

* CLH 锁是对自旋锁的改进优化

  * 将线程组织成队列，保证现请求线程先获得锁，避免锁饥饿问题

  * 锁状态去中心化，每个线程在不同的状态变量中自旋，某线程释放锁时

    只能使它后续线程高速缓存失效，缩小了影响范围，减少 CPU 开销

* CLH 锁实现机制

  * CLH 锁数据结构类似一个链表队列，所有请求获取锁的线程会排在链表队列中

  * 之所以是类似链表的队列，是因为其阶段没有使用前序后续引用属性

  * 每个节点只在局部变量保持对其上一个节点的引用

  * CLH 队列中的节点有两个属性：所代表的线程引用、是否持有锁的状态变量

  * 在队列中的线程会自旋访问排在它前面节点的状态

  * 当一个节点释放锁，只有其后的一个节点能得到锁

  * CLH 锁本身有一个队尾指针 Tail 原子变量

  * 当一个线程要获取锁时，先对 Tail 变量进行 getAndSet 原子操作

    设置自身为队尾，返回上一个队尾节点

  * 操作成功时，将自身封装成 CLH 节点加入队尾，并轮询上一个队尾节点的状态变量

  * 当上一个节点释放锁后，它将得到这个锁

* 优缺点

  * 优点
    * 性能优异，获取释放锁开销小，锁状态去中心化，降低自旋竞争开销，锁释放无需 CAS
    * 公平锁方式，防止锁饥饿
  * 缺点
    * 还是有自旋操作，锁持有时间长还是会带来 CPU 开销
    * 锁功能单一，不能支持复杂功能
  * AQS 对缺点的改进
    * AQS 采用阻塞线程替换自旋操作
    * 扩展每个节点状态、显式维护前驱及后续节点、出队列显式置 `null` 辅助 GC 优化
      * 节点状态包含
        * SIGNAL 正常等待
        * PROPAGATE 应将 releaseShared 传播到其他节点
        * CONDITION 该节点位于条件队列，不能用于同步队列节点
        * CANCELLED 超时、中断或其它原因，该节点被取消
      * 显式维护前驱后续节点，可以支持各种中断取消
        * 由于节点线程是阻塞，改为释放锁的节点唤醒下一个节点解除阻塞
        * 锁释放时，由于后驱节点并非原子性插入，不可靠，如果释放时后续节点不可用
        * 将利用队尾指针 Tail 从尾部逆序遍历到当前节点正确的后驱节点
        * 逆序遍历将自动跳过取消、中断等异常节点

  > [Java AQS 核心数据结构-CLH 锁](https://mp.weixin.qq.com/s/jEx-4XhNGOFdCo4Nou5tqg)

### **AQS (基于 Java 8 版本)**

* AQS 抽象队列同步器，`AbstractQueuedSynchronizer` 抽象类，主要用来构建锁和同步器

* AQS 提供构建锁和同步器的通用功能实现，基于 AQS 实现的有 `ReentrantLock`、`Semaphore`等等

* AQS 提供了锁和同步的通用实现，留下一些模版方法让子类进行定制：

  ```java
  //该线程是否正在独占资源。只有用到condition才需要去实现它。
  isHeldExclusively();
  //独占方式。尝试获取资源，成功则返回true，失败则返回false。    
  tryAcquire(int);
  //独占方式。尝试释放资源，成功则返回true，失败则返回false。
  tryRelease(int);
  
  //共享方式。尝试获取资源。
  // 负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
  tryAcquireShared(int);
  //共享方式。尝试释放资源，成功则返回true，失败则返回false。
  tryReleaseShared(int);
  ```

* 原理 [动画](https://www.bilibili.com/video/BV1z44y1X7BJ/?spm_id_from=333.788&vd_source=73d39b7559440790e99c2747efb22744)

  * 请求资源空闲，将当前请求资源的线程设置为工作线程，标识资源未锁定状态
  * 请求资源被占用，将线程阻塞放入优化过的 CLH 队列等待唤醒（双向链表，尾插法）
  * 线程阻塞前，会确保其前驱节点的状态为 SIGNAL，否则可能无线程通知此线程唤醒
  * 当占用资源的线程释放资源时，会从队头处开始唤醒后继节点（头部出队）
  * 队列遵循 FIFO，只有排在队头的节点有尝试获取锁的能力，其他节点都阻塞
  * AQS 重要的成员变量
    * head 头节点 **Node**
    * tail 尾节点 **Node**
    * state 锁资源状态
    * 持有资源的线程
  * AQS 将等待现场抽象为一个节点 **Node** 来实现锁的分配管理，节点内包含：
    * 线程引用 thread
    * 当前节点在队列的状态 waitStatus [状态说明](https://www.bilibili.com/video/BV1Fd4y1b7Qp/?share_source=copy_web&vd_source=1a66d16291c5f39e7d3214ae571faee7)
      * CANCELLED = 1（超时、中断等原因，取消等待锁）
      * SIGNAL = -1 (正常等待锁，有责任唤醒后继节点，后续节点已阻塞)
      * CONDITION = -2（等待条件锁，它不在同步队列，在某个条件的等待队列）
      * PROPAGATE = -3（用在共享模式）
    * 前驱节点 prev
    * 后继节点 next
  * AQS 使用 `Condition` 时，除会生成同步队列外，还会产生对应条件的条件队列（单向链表）
    * 条件等待队列也遵循 FIFO，即：尾插头出
    * 在条件等待队列的节点，waitStatus = -2
    * 当条件等待队列中的节点被通知时，将会移动到同步队列，并将 waitStatus = 0
    * 移回到同步队列时，它的前节点为 CANCELLED 或修改前节点为 SIGNAL 失败，将立即唤醒

* 基于 AQS 实现的常见锁和同步队列

  * **Semaphore** 共享锁，允许一定数量的线程获得锁，支持公平、非公平模式

  * **CountDownLatch** 共享锁，一次性的线程同步器

    * 初始化一定数量的 `count` 值，每次线程对 `countDown()` 方法调用时该值减 1

    * 调用该类的 `await()` 方法的线程，将阻塞等待，直到 `count` 值变为 0 才被唤醒

    * 运用场景：

      **单个或多个线程** 等待`await()`单个或多个步骤执行完 `countDown()`，同时唤醒开始做某事。

      * 阻塞 N 个线程（即：N 个线程调用 `await()` 方法）
      * 等待发生 M 个阶段执行后（即：`count` 初始化为 M，调用 M 次 `countDown()`）
      * M 个阶段执行后，N 个线程同时解除阻塞

  * **CyclicBarrier** 基于 **ReentrantLock** 的等待条件实现，可重复使用的线程同步器

    * 指定待同步线程个数 `parties`，以及计数器 `count`
    * 每当有线程 `await()` 时，计数器 `count` 减 1，并将当前线程放入条件等待队列
    * 当计数器 `count` 为 0 时，将等待队列的 parties 个线程全部唤醒
    * 执行 `nextGeneration()` 时，又可以开启下一轮上面的线程同步步骤

### **虚拟线程**

&emsp;&emsp;参考 **新特性 Java 20 章节**

## IO

### 字节流

* `InputStream`
  * Java 9 新增
    * `readAllBytes()` 读取所有字节，返回字节数组
    * `readNBytes(byte[] b, int off, int len)` 阻塞直到读取 `len` 个字节
    * `transferTo(OutputStream out)` 将所有字节从输入流传递到输出流
  * `FileInputStream` 从指定文件中读取字节或字节数组
  * `DataInputStream` 读取指定类型数据的包装流
  * `ObjectInputStream` 读取 Java 对象（反序列化）的包装流
  * `BufferedInputStream` 字节流包装流，实现字节缓冲
* `OutputStream`
  * `flush()` 将缓冲的字节强制写出，确保剩余缓冲字节顺利移交操作系统
  * `FileOutputSteam` 最常用的将字节流输出到文件
  * `DataOutStream` 将指定类型数据输出到输出流
  * `ObjectOutStream` 将 Java 对象（序列化）输出到字节流
  * `BufferedOutStream` 具备字节缓冲功能字节输出流包装类

### 字符流

* `Reader`

  * `InputStreamReader` 字节流转换成字符流的桥梁

    ```java
    public InputStreamReader(InputStream in) {
        super(in);
        sd = StreamDecoder.forInputStreamReader(in, this,
        Charset.defaultCharset()); // ## check lock object
    }
    ```

    字节流转成字符流，默认的字符集采用运行环境的默认字符集，也可使用重载的构造方法指定其他字符集。默认字符集获取规则：

    * 先获取系统变量 `file.encoding` 的编码集，获取到使用该编码集
    * 否则使用 `UTF_8` 字符集
    * 读取后的字符在 JVM 内部是由 [**UTF-16 编码**](https://reionchan.github.io/2015/11/25/about-unicode/#unicode-%E7%BC%96%E7%A0%81-%E4%B8%80%E8%88%AC%E5%8D%B3-utf-16-) 表示

  * `FileReader` 为 `InputStreamReader` 的子类，可指定文件读取字符流

* `Writer`

  * `OutputStreamWriter` 字符流转字节流的输出流

    将字符流输出到字节流，**注意此处也会发生字符编码转换**，默认使用当前系统环境的字符集

  * `FileWriter` 为 `OutputStreamWriter` 的子类，可以指定字符流最终保存的文件

* `RandomAccessFile` 随机访问流，支持随意跳转到文件的任意为止进行读写

  支持下面四种方式访问文件：

  * `r` : 只读模式。

  * `rw`: 读写模式

  * `rws`: 相对于 `rw`，`rws` 同步更新对“文件的内容”或“元数据”的修改到外部存储设备。

  * `rwd` : 相对于 `rw`，`rwd` 同步更新对“文件的内容”的修改到外部存储设备

### **I/O 模型**

* 同步阻塞 BIO

  ![同步阻塞](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jtsn/6a9e704af49b4380bb686f0c96d33b81~tplv-k3u1fbpfcp-watermark.png)

  [引用图源-JavaGuide](https://oss.javaguide.cn/p3-juejin/6a9e704af49b4380bb686f0c96d33b81~tplv-k3u1fbpfcp-watermark.png)

* 同步非阻塞 NIO

  ![同步非阻塞](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jtsn/bb174e22dbe04bb79fe3fc126aed0c61~tplv-k3u1fbpfcp-watermark.png)

  [引用图源-JavaGuide](https://oss.javaguide.cn/p3-juejin/bb174e22dbe04bb79fe3fc126aed0c61~tplv-k3u1fbpfcp-watermark.png)

* 多路复用 NIO select/poll/epoll

  ![多路复用](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jtsn/88ff862764024c3b8567367df11df6ab~tplv-k3u1fbpfcp-watermark.png)

  [引用图源-JavaGuide](https://oss.javaguide.cn/github/javaguide/java/io/88ff862764024c3b8567367df11df6ab~tplv-k3u1fbpfcp-watermark.png)

  * select/poll vs epoll

    * select/poll

      * select 采用数组结构保存 fd（*I/O 文件描述符*） 集合, 故有最大连接数限制

        每次 `select` 调用时，都会拷贝 fd 集合到内核态，在内核态开辟空间

        内核需遍历 fd 集合 **O(n)** 复杂度

      * poll 采用链表结构保存 fd，理论无上限，但具体受限操作系统限制

        每次 `poll` 调用时，也会拷贝 fd 集合到内核态，在内核态开辟空间
        
        内核需要遍历 fd 集合 **O(n)** 复杂度 

    * epoll

      * epoll 采用哈希表保存 fd，且改用事件通知回调方式将就绪的 fd 放到

        用户空间的 rdlist 里面，免去用户态频繁拷贝 fd 集合，内核态不断轮询
        
        * epoll_create
        
          * 该系统调用返回一个 epfd 指向内核的空间（红黑树和就绪链表）
        
            * 红黑树用于存储之后 epoll_ctl 传递的 fd **O(logn)**
        
            * 就绪链表用于存储就绪的事件，epoll_wait 仅观察此链表有无数据
        
              有数据返回，无数据 sleep，timeout 后无数据也返回
        
              由于只返回就绪的 fd 集合，免去遍历，数量小，内核仅拷贝少量 fd 到用户空间
        
        * epoll_ctl
        
          * 该系统调用将 fd 放入 epoll_create 创建的红黑树中
        
          * 向内核注册中断处理函数，当该 fd 中断时内核执行此函数将 fd 放入之前的就绪链表
        
            例如：
        
            ​	网卡 fd 有数据发生中断，网卡数据被拷贝到内核缓存区后，
        
            ​	内核就会把网卡 fd 插入到就绪链表（此步骤是内核响应中断后所做的延伸回调操作）
        
        * epoll_wait
        
          * 由于中断处理会将已准备就绪的 fd 放入就绪链表，故 epoll_wait 免去遍历红黑树
      
      ![多路复用区别对比](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jtsn/577140-20221104170510883-1267943094.png)
      
      [引用图源-cnblogs](https://img2022.cnblogs.com/blog/577140/202211/577140-20221104170510883-1267943094.png)

  * Reactor 模式

    > [IO 精讲-Java NIO 实现演进优化](https://www.mashibing.com/study?courseNo=340&sectionNo=23541&courseVersionId=1256 )
    >
    > [Java NIO 浅析](https://tech.meituan.com/2016/11/04/nio.html)
    >
    > [NIO 网络编程](https://www.mashibing.com/live/1758?r=793&courseVersionId=2363)

    * 单线程 Reactor
    * 多线程 Reactor
    * 主从多线程 Reactor

* 信号驱动

  ![信号驱动](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jtsn/577140-20221104170510676-1445375786.png)

  [引用图源-cnblogs](https://img2022.cnblogs.com/blog/577140/202211/577140-20221104170510676-1445375786.png)

* 异步

  ![异步](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/jtsn/3077e72a1af049559e81d18205b56fd7~tplv-k3u1fbpfcp-watermark.png)
  
  [引用图源-JavaGuide](https://oss.javaguide.cn/github/javaguide/java/io/3077e72a1af049559e81d18205b56fd7~tplv-k3u1fbpfcp-watermark.png)
  
  

### **内核零拷贝**

> [一文彻底弄懂零拷贝原理](https://juejin.cn/post/6995519558475841550)
>
> [底层：I/O 传输](https://zhuanlan.zhihu.com/p/497991516)
>
> [普通IO数据拷贝次数的问题探讨](https://www.cnblogs.com/shanml/p/16726953.html)
>
> [【操作系统】HeapByteBuffer和DirectByteBuffer的区别](https://blog.csdn.net/u022812849/article/details/136020787)
>
> [JAVA IO专题一：java InputStream和OutputStream读取文件并通过socket发送，到底涉及几次拷贝](https://www.jianshu.com/p/156f02ef0680)
>
> [JAVA IO专题二：java NIO读取文件并通过socket发送，最少拷贝了几次？堆外内存和所谓的零拷贝到底是什么关系](https://www.jianshu.com/p/f539c29920a1)

* 内核零拷贝是一种 I/O 操作优化技术，可以快速高效的将数据从文件系统移动到网络接口，而不需将其从内核空间复制到用户空间。其在 FTP、HTTP 等协议中显著的提升性能，但不是所有操作系统都支持这一特性，目前只在使用 NIO 和 Epoll 传输时才能使用该特性

* 传统 I/O 操作

  * read 系统调用，两次上下文切换（调用入内核，返回出内核）将内核数据拷贝到用户态冲区（CPU 拷贝）

    此外：内核数据来源于磁盘拷贝到内核，属于 [**DMA**](https://zhuanlan.zhihu.com/p/138573828) 拷贝（外设与内核缓存区直接拷贝，发生在内核态） 

  * write 系统调用，两次上下文切换（调用入内核，返回出内核）将用户缓冲区数据拷贝到内核（CPU 拷贝）

    此外：内核数据将写入外设（如：网卡），也属 DMA 拷贝

  因此，传统 I/O 操作包含：

  * 4 次上下文切换

  * 2 次 CPU 拷贝（用户态与内核态之间的拷贝）

    * **堆内内存**：JVM 申请的堆，Java 普通对象分配及发生 GC 处理的目标内存
    * **堆外内存**：JVM 进程申请的受 GC 管理的内存，也被称作直接内存

    > 注意：
    >
    > 1. **内核零拷贝只关注内核拷贝故忽略用户态中堆内外的拷贝，即数据真正到达 JVM 堆还需要进行一次 JVM 堆外到堆内的拷贝，下面篇幅雷同请留意！**
    > 2. **Java 零拷贝指的是使用直接内存（申请堆外内存）当做读写缓冲区，避免 JVM 堆内堆外的拷贝（发生在用户空间）。**

  * 2 次 DMA 拷贝

* mmap+write

  * 利用内存映射 mmap 将用户空间的缓冲区映射到内核缓冲区，达到减少用户态的 CPU 拷贝

  * mmap 系统调用时，两次上下文切换，但此时由于用户缓冲区和内核缓冲区共享

    DMA 拷贝磁盘数据到内核缓冲区的数据就被认为已在用户缓冲区

  * write 系统调用时，两次上下文切换，此时用户缓冲区数据拷贝到网卡对应的内核缓冲区

    现在仅发生在内核态上（从磁盘内核缓冲区拷贝到网卡内核缓冲区，一次内核态 CPU 拷贝）

  因此，mmap+write 操作包含：

  * 4 次上下文切换
  * 1 次 CPU 拷贝（内核态内的拷贝）
  * 2 次 DMA 拷贝

* sendfile

  * sendfile 将 mmap 和 write 两次系统调用替换为单次的 sendfile 单次系统调用，故减少了两次上下文切换
  * sendfile 将文件数据放入内核缓冲区后，直接将其拷贝到网卡缓冲区而无需用户态的 write 调用

  因此，sendfile 操作包含：

  * 2 次上下文切换（sendfile 系统调用）
  * 1 次 CPU 拷贝（内核态内的拷贝）
  * 2 次 DMA 拷贝

* sendfile+SG-DMA

  * Linux 2.4 内核优化，提供了跨设备内核缓冲区 DMA 优化，不同设备内核缓冲区之间拷贝数据采取

    传输文件描述符形式让 DMA 原设备内核缓冲区直接拷贝数据到新设备，而无需通过 CPU 拷贝，

    这种技术被称为 scatter/gather 技术，需要支持此功能的 DMA，即 SG-DMA

  因此，sendfile+SG-DMA 操作包含：

  * 2 次上下文切换（sendfile 系统调用）
  * 0 次 CPU 拷贝（被 SG-DMA 拷贝取代）
  * 2 次 DMA 拷贝（1 次正常 DMA 拷贝，1 次 SG-DMA 拷贝）

* splice 

  * splice 调用与 sendfile 类似，不同的是在不同设备内核缓冲区之间拷贝数据不是通过硬件支持的 SG-DMA

    而是采取在两个设备内核缓冲区之间建立文件传输管道，同样能够减少 CPU 拷贝，却无需硬件支持

  因此，splice 操作包含：

  * 2 次上下文切换（splice 系统调用）
  * 0 次 CPU 拷贝（采用两个内核缓冲区建立管道技术取代）
  * 2 次 DMA 拷贝（2 次均为正常的 DMA 拷贝，目标设备通过管道从另一个设备的内核空间获取数据）

### **Java NIO 零拷贝**

* 利用 `ByteBuffer.allocateDirect(int size)` 直接内存

  申请堆外内存当做缓冲区，减少 JVM 堆内与堆外内存的拷贝

* 利用 `FileChannel#transferTo` 或 `FileChannel#transferFrom` 

  这两个方法利用操作系统支持的 `sendfile` 系统调用在支持的两个通道（例如：磁盘文件与网卡之间）

  传递数据而省去 CPU 参与的拷贝

### **磁盘 I/O**

* [磁盘I/O那些事](https://tech.meituan.com/2017/05/19/about-desk-io.html)

## 推荐阅读

[Java 技术栈笔记 —— Java 下篇](https://reionchan.github.io/2024/03/18/java-tech-stack-note-p02)

[Java 技术栈笔记 —— 计算机基础](https://reionchan.github.io/2024/03/24/java-tech-stack-note-p03)