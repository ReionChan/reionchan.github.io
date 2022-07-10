---
layout: post
title: 《Spring Boot 编程思想（核心篇）》下篇
categories: Book Spring
tags: SpringBoot Spring Boot 
excerpt: 读该书 “第 3 部分 —— 理解 SpringApplication” 心得笔记 
image: https://www.image.com
description: SpringBoot 一些关键特性的原理及设计思想
keywords: Spring SpringBoot Thinking SpringApplication 初始化 监听器 配置 事件 配置源
licences: cc
---
<br/>

<img src="https://spring.io/images/spring-logo-9146a4d3298760c2e7e49595184e1975.svg" alt="Spring Logo" width="800"/>

---
> &emsp;&emsp;Spring Boot 具有一套固化的视图，该视图用于构建生产级别的应用。
<div style="text-align: right;"> —— 摘自 <a href="https://spring.io/projects/spring-boot/">Spring 官网</a> &emsp;&emsp;</div>

## SpringApplication 初始化阶段

### SpringApplication 构造阶段

* 理解 SpringAppliction 主配置类

  &emsp;&emsp;运行 SpringApplication 应用，是通过调用其静态方法：

  ```java
  public class SpringApplication {
  	public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
  		return run(new Class<?>[] { primarySource }, args);
  	}    
  }
  ```

  &emsp;&emsp;其中所指定的 *主配置类* `primarySource` 将被注册成为 BeanDefinition 后续被 *ConfigurationClassPostProcessor* 处理，检查其是否能成为候选的配置类，从而进行配置类的解析操作。一般默认将 SpringApplication 引导类上标注 *@SpringBootApplication* 或 *@EnableAutoConfiguration* 注解，然后将其作为 `run` 方法的 `primarySource` 主配置类参数进而激活 Spring Boot 自动化装配。当然，知道默认这样操作的本质后，完全可以指定另外的配置类来充当 `run` 方法的主配置参数。

  &emsp;&emsp;除使用 `run` 方法外，还可以通过下面方法追加主配置类：

  ```java
  public class SpringApplication {
      public void addPrimarySources(Collection<Class<?>> additionalPrimarySources) {
          this.primarySources.addAll(additionalPrimarySources);
      }
  }
  ```

  &emsp;&emsp;总之，主配置类一般都是作为包含 *@SpringBootApplication* 从而启动自动化装配、设置包扫描路径等行为的配置载体，只是顺带有了向容器注册 BeanDefinition 的能力。不过像容器中添加额外的资源，例如：类名、包名、XML 资源路径，SpringApplication 提供了额外的方法 `setSources(Set<String>)` 用来区别主配置与额外资源之间功能分工的不同。

  

* SpringApplication 的构造过程

  &emsp;&emsp;SpringApplication 的构造过程主要分下面几个过程：

  ```java
  public class SpringApplication {
      public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
          this.resourceLoader = resourceLoader;
          Assert.notNull(primarySources, "PrimarySources must not be null");
          this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
          // 推断 Web 应用类型
          this.webApplicationType = deduceWebApplicationType();
          // 设置 Spring 应用上下文初始化器
          setInitializers((Collection) getSpringFactoriesInstances(
                  ApplicationContextInitializer.class));
          // 设置 Spring 应用事件监听器
          setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
          // 推断应用主方法类
          this.mainApplicationClass = deduceMainApplicationClass();
      }
  }
  ```

  &emsp;&emsp;下面将针对这四个过程逐一讨论。

  

  * 推断 Web 应用类型

    ```java
    public class SpringApplication {
        private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
                "org.springframework.web.context.ConfigurableWebApplicationContext" };
        public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
                + "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";
        private static final String REACTIVE_WEB_ENVIRONMENT_CLASS = "org.springframework."
                + "web.reactive.DispatcherHandler";
        private static final String MVC_WEB_ENVIRONMENT_CLASS = "org.springframework."
                + "web.servlet.DispatcherServlet";
    
        private WebApplicationType deduceWebApplicationType() {
            // 判断是否是 Reactive Web 类型
            if (ClassUtils.isPresent(REACTIVE_WEB_ENVIRONMENT_CLASS, null)
                    && !ClassUtils.isPresent(MVC_WEB_ENVIRONMENT_CLASS, null)) {
                return WebApplicationType.REACTIVE;
            }
            // 判断是否为非 Web 应用类型
            for (String className : WEB_ENVIRONMENT_CLASSES) {
                if (!ClassUtils.isPresent(className, null)) {
                    return WebApplicationType.NONE;
                }
            }
            // 否则为 Servlet Web 类型
            return WebApplicationType.SERVLET;
        }
    }
    ```

    1. Reactive Web 类型标志：

       包含： *DispatcherHandler*

       不包含：*DispatcherServlet*

    2. 非 Web 类型标志：

       不包含：*Servlet*、*ConfigurableWebApplicationContext*

    3. Servlet Web 类型标志：

       非前两种类型时，即为此类型。

    

  * 加载 Spring 应用上下文初始化器

    &emsp;&emsp;利用 *Spring 工厂加载机制* 加载所有设置的上下文初始化器，并将其保存到初始化器列表属性中。在 *ConfigurableApplicationContext* 生命周期方法 `refresh()` 调用前对其进行一些初始化操作。

    ```java
    public class SpringApplication {
        // 工厂加载机制获取 ApplicationContextInitializer 实例
        private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
                                                              Class<?>[] parameterTypes, Object... args) {
            ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
            Set<String> names = new LinkedHashSet<>(
                    SpringFactoriesLoader.loadFactoryNames(type, classLoader));
            List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
                    classLoader, args, names);
            // ApplicationContextInitializer 实例可能实现 Ordered 接口或 @Order 注解的进行排序
            AnnotationAwareOrderComparator.sort(instances);
            return instances;
        }
        // 将其保存在 initializers 中，等待生命周期方法调用
        public void setInitializers(
                Collection<? extends ApplicationContextInitializer<?>> initializers) {
            this.initializers = new ArrayList<>();
            this.initializers.addAll(initializers);
        }
    }
    ```

    &emsp;&emsp;查看 *SpringApplication* 运行方法 `run` 可知，这些收集的初始化器将在生命周期方法 `prepareContext` 中被方法 `applyInitializers` 所执行，而 `prepareContext` 方法的确是在 *ConfigurableApplicationContext* 的 `refresh` 方法之前调用。除利用 Spring 工厂加载机制之外，还可以通过 *SpringApplication* 的 `addInitializers` 进行追加、环境参数 `context.initializer.classes` 进行添加（通过实现类 *DelegatingApplicationContextInitializer* 增强类得以实现）。

    &emsp;&emsp;应用默认设置的初始化器：

    ```properties
    # Application Context Initializers
    org.springframework.context.ApplicationContextInitializer=\
    org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
    org.springframework.boot.context.ContextIdApplicationContextInitializer,\
    org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
    org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
    ```
  
    
  
  * 加载 Spring 应用事件监听器
  
    &emsp;&emsp;加载事件监听器与上面加载初始化器类似，也是通过 Spring 工厂加载机制实现。
  
    ```properties
    # Application Listeners
    org.springframework.context.ApplicationListener=\
    org.springframework.boot.ClearCachesApplicationListener,\
    org.springframework.boot.builder.ParentContextCloserApplicationListener,\
    org.springframework.boot.context.FileEncodingApplicationListener,\
    org.springframework.boot.context.config.AnsiOutputApplicationListener,\
    org.springframework.boot.context.config.ConfigFileApplicationListener,\
    org.springframework.boot.context.config.DelegatingApplicationListener,\
    org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
    org.springframework.boot.context.logging.LoggingApplicationListener,\
    org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
    ```
  
    &emsp;&emsp;加载的监听器将被添加到 *SpringApplication* 的 `this.listeners` 属性中。 值得注意不管是监听器列表或是前面提及的上下文初始化器列表，它们的 `set` 方法都是覆盖更新，只有 `add` 方法才是追加操作。
  
    
  
  * 推断应用引导类
  
    &emsp;&emsp;使用运行时堆栈信息来获取包含 `main` 方法的启动引导类名。
  
    ```java
    public class SpringApplication {
        private Class<?> deduceMainApplicationClass() {
            try {
                StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
                for (StackTraceElement stackTraceElement : stackTrace) {
                    // 查找堆栈中方法名为 main 的类名称
                    if ("main".equals(stackTraceElement.getMethodName())) {
                        return Class.forName(stackTraceElement.getClassName());
                    }
                }
            }
    		...
        }
    }
    ```
  
    

### SpringApplication 配置阶段

&emsp;&emsp;配置阶段位于构造阶段和运行阶段之间，该阶段是可选的，其主要用于调整或补充构造阶段的状态，左右运行时行为。调整操作主要通过以 **set** 开头的方法，而补充操作主要通过以 **add** 开头的方法。由于可用来调整的方法众多，引入了 **构造器 *SpringApplicationBuilder* API** 链式调用简化调整。

&emsp;&emsp;配置阶段主要可分以下三个内容：

1. 调整 SpringApplication 设置

   &emsp;&emsp;除 `setInitializers`、`setListeners` 方法外，基本上 SpringApplication 的 **set** 开头的方法都对应有 SpringApplicationBuilder 中以**属性命名规则相同**的 Builder 方法。

    

2. 增加 SpringApplication 配置源

   &emsp;&emsp;为 SpringApplication 添加配置源在 **Spring Boot 1.x** 时代是通过下面放实现：

   ```java
   public class SpringApplication {
       // 配置源集合，注意为 Object 类型
       private final Set<Object> sources = new LinkedHashSet<Object>();
   
       // 构造方法调用 initialize
       public SpringApplication(Object... sources) {
           initialize(sources);
       }
   
       // this.sources 将参数传入全部接收
       private void initialize(Object[] sources) {
           if (sources != null && sources.length > 0) {
               this.sources.addAll(Arrays.asList(sources));
           }
           this.webEnvironment = deduceWebEnvironment();
           setInitializers((Collection) getSpringFactoriesInstances(
                   ApplicationContextInitializer.class));
           setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
           this.mainApplicationClass = deduceMainApplicationClass();
       }
   }
   ```

   &emsp;&emsp;注意看构造方法，接受的是 ***Object*** 类型的可变参数。此时的配置源没做过多的区分，可以指定 *Class*、*Resource*、字符串对象，字符串形式有限解析是否为 *Class* 或 *Resource* 资源路径，否则视作包名处理。

   &emsp;&emsp;而在 **Spring Boot 2.x** 时代，配置源做了一些区分。将具备引导实现自动化装配的配置类视作主配置类，剩下的其它配置统统以字符串形式配置放入字符串形式的集合中。

   ```java
   public class SpringApplication {
       // 主配置源，Class 类型集合
       private Set<Class<?>> primarySources;
       // 其他配置源，字符串类型集合
       private Set<String> sources = new LinkedHashSet<>();
   
       // 构造方法调整为 Class 类型可变参数
       public SpringApplication(Class<?>... primarySources) {
           this(null, primarySources);
       }
       // 设置其他配置源时，统一修改为接受字符串集合对象
       public void setSources(Set<String> sources) {
           Assert.notNull(sources, "Sources must not be null");
           this.sources = new LinkedHashSet<>(sources);
       }
   
       // 获取所有配置源时，将主配置源级其他配置源组合返回
       public Set<Object> getAllSources() {
           Set<Object> allSources = new LinkedHashSet<>();
           if (!CollectionUtils.isEmpty(this.primarySources)) {
               allSources.addAll(this.primarySources);
           }
           if (!CollectionUtils.isEmpty(this.sources)) {
               allSources.addAll(this.sources);
           }
           return Collections.unmodifiableSet(allSources);
       }
   }
   ```

   &emsp;&emsp;方法 `setSources(Set<String>)` 参数不在支持 *Class* 类型的传入，相反只能用 *Class* 的全限定名形式传入。

   

3. 调整 Spring Boot 外部化配置

   &emsp;&emsp;Spring Boot 支持外部配置文件 `application.properties` 来覆盖默认的属性配置 `defaultProperties`，而此属性配置的初始值是由方法 `setDefaultProperties` 设置，也即 `application.properties` 可以覆盖 `setDefaultProperties` 设置的默认属性。反过来，通过调用 `setDefaultProperties` 方法，指定属性参数 `spring.config.location` 或 `spring.config.additional-location` 又能改变对 `application.properties` 文件搜索路径。

   

## SpringApplication 运行阶段

&emsp;&emsp;SpringApplication 运行阶段围绕 `run(String...)` 方法展开，该过程将初始化阶段收集的状态转换为运行是的具体行为，进而启动 Spring 应用上下文，形成完整的 SpringApplication 生命周期，同时伴随着 Spring Boot 和 Spring 事件的触发。

```java
public class SpringApplication {
    // 运行方法
    public ConfigurableApplicationContext run(String... args) {
		...
        // ---------- 阶段一：准备 -----------
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        configureHeadlessProperty();
        SpringApplicationRunListeners listeners = getRunListeners(args);
        listeners.starting();
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
            configureIgnoreBeanInfo(environment);
            Banner printedBanner = printBanner(environment);
            context = createApplicationContext();
            exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                    new Class[] { ConfigurableApplicationContext.class }, context);
            prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            // ---------- 阶段二：启动 -----------
            refreshContext(context);
            // ---------- 阶段三：启动后 -----------
            afterRefresh(context, applicationArguments);
			...
            listeners.started(context);
            callRunners(context, applicationArguments);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, listeners);
            throw new IllegalStateException(ex);
        }

        try {
            listeners.running(context);
        }
		...
        return context;
    }
}
```

&emsp;&emsp;从方法注释，将运行阶段分为三个子阶段：分别为准备、启动、启动后，下面将从这三个阶段进行分析。

### SpringApplication 准备阶段

* 理解 *SpringApplicationRunListeners*

  &emsp;&emsp;应用启动首先是构造 ***SpringApplicationRunListeners*** 对象，由方法 `getRunListeners` 生成：

  ```java
  public class SpringApplication {
  
      private SpringApplicationRunListeners getRunListeners(String[] args) {
          Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
          // 构造 SpringApplicationRunListeners
          return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
                  SpringApplicationRunListener.class, types, this, args));
      }
  }
  ```

  &emsp;&emsp;*SpringApplicationRunListeners* 属于*组合模式*的实现，内部关联了 *SpringApplicationRunListener* 的集合：

  ```java
  class SpringApplicationRunListeners {
  
      private final List<SpringApplicationRunListener> listeners;
  
      SpringApplicationRunListeners(Log log, Collection<? extends SpringApplicationRunListener> listeners) {
          this.log = log;
          this.listeners = new ArrayList<>(listeners);
      }
  
      public void starting() {
          for (SpringApplicationRunListener listener : this.listeners) {
              listener.starting();
          }
      }
      ...
  }
  ```

  &emsp;&emsp;其主要作用是在应用运行的每个重要阶段，遍历关联的 *SpringApplicationRunListener* 集合，依次调用其中的监听器方法。

  

* 理解 *SpringApplicationRunListener*

  &emsp;&emsp;按照字面意思，该监听器接口是用来充当在 Spring Boot 运行时做一系列监听操作，其监听的方法被 *SpringApplicationRunListeners* 迭代地执行，例如：上面代码的 `starting()` 方法。

  ```java
  public interface SpringApplicationRunListener {
  
      // SpringApplication 刚执行 run 时被执行
      void starting();
  
      // ConfigurableEnvironment 准备完毕，应用上下文被创建之前，允许将其调整
      void environmentPrepared(ConfigurableEnvironment environment);
  
      // ConfigurableApplicationContext 准备完毕，但资源还未装载时，允许将其调整
      void contextPrepared(ConfigurableApplicationContext context);
  
      // ConfigurableApplicationContext 已装载 BeanDefinition 完毕，但还未启动 refresh 
      void contextLoaded(ConfigurableApplicationContext context);
  
      // ConfigurableApplicationContext 已 refresh 完毕，但 ApplicationRunners 未执行时
      void started(ConfigurableApplicationContext context);
  
      // SpringApplication run 方法执行完毕前，ApplicationRunners 执行完后
      void running(ConfigurableApplicationContext context);
  
      // SpringApplication run 方法执行异常时被执行
      void failed(ConfigurableApplicationContext context, Throwable exception);
  }
  ```

  &emsp;&emsp;从该接口方法定义顺序，依次代表着 SpringApplication 运行时先后经历的生命周期过程。该接口的实现类是在 *SpringApplicationRunListeners* 构建过程中，利用 *Spring 工厂加载机制* 从 `META-INF/spring.factories` 中加载并实例化：

  ```java
  public class SpringApplication {
  
      private SpringApplicationRunListeners getRunListeners(String[] args) {
          // SpringApplicationRunListener 构造器参数
          Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
          return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
                  SpringApplicationRunListener.class, types, this, args));
      }
  
      // Spring 工厂加载机制，获得类型为 SpringApplicationRunListener 实现类实例
      private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
                                                            Class<?>[] parameterTypes, Object... args) {
  		...
          Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
          // 利用传入的构造器参数实例化监听器
          List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
                  classLoader, args, names);
          AnnotationAwareOrderComparator.sort(instances);
          return instances;
      }
  }
  ```

  &emsp;&emsp;从其实例化方法传入的 `parameterTypes` 可以得知，其构造方法需要两个参数： *SpringApplication* 实例及命令行参数。查看 `spring-boot` 项目工厂配置文件：

  ```properties
  # Run Listeners
  org.springframework.boot.SpringApplicationRunListener=\
  org.springframework.boot.context.event.EventPublishingRunListener
  ```

  &emsp;&emsp;可以发现只存在 *EventPublishingRunListener* 一个实现类，其构造方法确实为上述的两个参数：

  ```java
  public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
  
      private final SpringApplication application;
      private final String[] args;
      private final SimpleApplicationEventMulticaster initialMulticaster;
  
      // 构造方法
      public EventPublishingRunListener(SpringApplication application, String[] args) {
          this.application = application;
          this.args = args;
          this.initialMulticaster = new SimpleApplicationEventMulticaster();
          // application.getListeners()
          for (ApplicationListener<?> listener : application.getListeners()) {
              this.initialMulticaster.addApplicationListener(listener);
          }
      }
  
      @Override
      public void starting() {
          this.initialMulticaster.multicastEvent(
                  new ApplicationStartingEvent(this.application, this.args));
      }
  }
  ```

  &emsp;&emsp;在该实现构造方法中，发现其将 *SpringApplication* 关联的所有监听器都添加到 ***SimpleApplicationEventMulticaster*** 类型的对象中，该类源于 Spring Framework，实现 ***ApplicationEventMulticaster*** 接口，用于发布 Spring 应用事件。因此，可以得知 ***EventPublishingRunListener*** 监听器实际上是用来充当 Spring Boot 运行期事件发布者的角色，不过它只负责 **在 Spring Boot 运行过程生命周期方法中创建相应 Spring Boot 事件**，具体 Spring Boot 事件的广播则委托给事件广播器 ***SimpleApplicationEventMulticaster***，如上代码中： `starting` 方法中，*EventPublishingRunListener* 创建了 *ApplicationStartingEvent* 事件，并通过 *SimpleApplicationEventMulticaster* 的 `multicastEvent` 方法进行广播。

* 理解 Spring Boot 事件

  &emsp;&emsp;根据上面了解到，Spring Boot 运行监听器接口 *SpringApplicationRunListener* 实现类 *EventPublishingRunListener* 负责监听运行期各个阶段生命周期方法的执行，并在这些方法中创建相应的 Spring Boot 事件。

  

  &emsp;&emsp;下面将监听方法对应的 Spring Boot 事件总结如下：

  | 监听方法            | Spring Boot 事件                    | Spring Boot 起始版本 |
  | ------------------- | ----------------------------------- | :------------------: |
  | starting            | ApplicationStartingEvent            |         1.5          |
  | environmentPrepared | ApplicationEnvironmentPreparedEvent |         1.0          |
  | contextPrepared     |                                     |         1.0          |
  | contextLoaded       | ApplicationPreparedEvent            |         1.0          |
  | started             | ApplicationStartedEvent             |         1.0          |
  | running             | ApplicationReadyEvent               |         1.3          |
  | failed              | ApplicationFailedEvent              |         1.0          |

  &emsp;&emsp;这些事件的发生顺序及时机与监听方法被执行一一对应，想要自定义 Spring Boot 事件监听器干预或调整某个阶段的行为，那么必定要熟悉事件发生时所对应监听方法执行的时机，可以翻看 [理解 SpringApplicationRunListener](#理解 SpringApplicationRunListener) 小节中接口方法的注释。

  &emsp;&emsp;查看这些 Spring Boot 事件代码，不难发现其都继承 ***SpringApplicationEvent*** 抽象类，该类即为 Spring Boot 事件的基类：

  ```java
  // Spring Boot 事件继承 Spring 事件类
  public abstract class SpringApplicationEvent extends ApplicationEvent {
  
      private final String[] args;
      // Spring Boot 事件源为 SpringApplication, 并记录运行参数信息
      public SpringApplicationEvent(SpringApplication application, String[] args) {
          super(application);
          this.args = args;
      }
      ...
  }
  ```

  &emsp;&emsp;至此，出现了两种类型的监听器及两种事件，有必要区分它们：

  

  - *SpringApplicationRunListener* **VS** *ApplicationListener*

    |                                    | 监听对象                           | 触发执行对象                                                 |
    | ---------------------------------- | ---------------------------------- | ------------------------------------------------------------ |
    | ***SpringApplicationRunListener*** | 监听 Spring Boot 各运行阶段方法    | SpringApplication `run` 方法的不同阶段触发执行               |
    | ***ApplicationListener***          | 监听 Spring 事件、Spring Boot 事件 | 由 *ApplicationEventMulticaster* 广播器接口实现类 *SimpleApplicationEventMulticaster* 触发执行 |

    

  - *SpringApplicationEvent* **VS** *ApplicationContextEvent*

    |                               | 相同点                  | 不同点                                                       |
    | ----------------------------- | ----------------------- | ------------------------------------------------------------ |
    | ***SpringApplicationEvent***  | 继承 *ApplicationEvent* | Spring Boot 事件；事件源类型为 *SpringApplication*；由 *EventPublishingRunListener* 创建；通过 *SimpleApplicationEventMulticaster* 广播 |
    | ***ApplicationContextEvent*** | 继承 *ApplicationEvent* | Spring 上下文事件；事件源类型为 *ApplicationContext*；由 *AbstractApplicationContext* 创建 |

    &emsp;&emsp;既然 **Spring Boot 事件** 和 **Spring 上下文事件** 都继承 *ApplicationEvent*，那么它们肯定最终都被 *ApplicationListener* 监听处理。**前面已知 Spring Boot 事件通过 *SimpleApplicationEventMulticaster* 广播给 *ApplicationListener*，那么 Spring 上下文事件由谁广播给 *ApplicationListener* 呢？**回答此问题前，先来了解 **Spring 事件/监听 机制**。

    

* 理解 Spring 事件/监听 机制

  &emsp;&emsp;Spring 事件/监听 遵照 Java 监听模式规则，Spring 事件抽象类 *ApplicationEvent* 扩展 Java 定义的事件抽象类 *EventObject*：

  ```java
  // 扩展至 EventObject
  public abstract class ApplicationEvent extends EventObject {
      // EventObject 构造时需指定外部事件源，用于记录跟踪事件来源
      public ApplicationEvent(Object source) {
          super(source);
          this.timestamp = System.currentTimeMillis();
      }
      ...
  }
  ```

  &emsp;&emsp;Spring 监听器接口 *ApplicationListener* 则继承 Java 定义的监听器接口 *EventListener*，该接口为标签接口：

  ```java
  // 标签接口，无任何抽象方法
  public interface EventListener {
  }
  ```

  &emsp;&emsp;传统 Java AWT/Swing 中一般采取在同一个监听器实现类中定义处理不同事件的方法，优点是相同事件源的不同事件能在一个监听器中集中处理，而缺点是开发者自定义监听器即使只关注部分事件处理方法，也必须实现全部事件处理方法。虽然可以通过适配器或 Java 8 的接口默认方法特性能够弥补这些缺点，可是 Spring 事件监听器使用了相反的设计理念，它的监听器中只包含单个事件处理方法，监听的是**扩展至 Spring 事件抽象基类 *ApplicationEvent* 的泛型类型**：

  ```java
  @FunctionalInterface
  public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
      // 定义事件的泛型类型，达到一个方法处理不同 Spring 事件类
      void onApplicationEvent(E event);
  }
  ```

  &emsp;&emsp;通过泛型参数可以在实现类中设定特定的监听事件，然而此方式缺点是无法再同一监听器监听不同事件。Spring Framework 3.0 的解决办法是引入 *SmartApplicationListener* 接口：

  ```java
  // 监听的事件泛型类型为事件抽象基类 ApplicationEvent
  public interface SmartApplicationListener extends ApplicationListener<ApplicationEvent>, Ordered {
      // 判断此监听器支持的事件类型
      boolean supportsEventType(Class<? extends ApplicationEvent> eventType);
      // 判断此监听器支持的事件源类型
      boolean supportsSourceType(@Nullable Class<?> sourceType);
  }
  ```

  1. Spring 事件发布

     &emsp;&emsp;在讨论 *EventPublishingRunListener* 时提到，Spring Boot 运行时事件交由 *SimpleApplicationEventMulticaster* 发布，而它来自 Spring Framework，是 *ApplicationEventMulticaster* 接口的实现类。该接口主要承担两种职责：**关联 *ApplicationListener***、**广播 *ApplicationEvent***。所以 Spring 事件的发布将交由该接口的实现类，下面是具体类关系图：

     

     ![Spring 事件/监听 类图](https://raw.githubusercontent.com/ReionChan/thinking-in-spring-boot-samples/9151d0c409e3bb41b13e3888078bb6ee4efa1241/spring-boot-2.0-samples/spring-application-sample/src/main/resources/uml/Spring-Event-Listener.svg)

     - 关联 *ApplicationListener*

       &emsp;&emsp;类图显示，接口 *ApplicationEventMulticaster* 与具体实现类 *SimpleApplicationEventMulticaster* 之间还存在一个抽象类 ***AbstractApplicationEventMulticaster***，由它来添加、删除及归类 Spring 事件监听器。每当有监听器通过它的 `addApplicationListener` 方法添加到广播器时，都会暂存在 `defaultRetriever` 变量中，当子类 *SimpleApplicationEventMulticaster* 需要获取某事件类型所有监听器列表时，会调用它的 `getApplicationListeners` 方法，该方法会从按事件类型归类的监听器缓存 Map 中查找，如果没命中会对暂存的监听器中遍历找寻某事件类型的所有监听器，并将这些监听器组合成 *ListenerRetriever* 对象放入以该事件、事件源类型组合形成的 *ListenerCacheKey* 对象为 key 的缓存 Map 中，达到按事件类型分类监听器且缓存方便下次索引，详细方法：

       ```java
       public abstract class AbstractApplicationEventMulticaster {
           ...
       
           // 子类 SimpleApplicationEventMulticaster 根据事件类型，调用此方法获得事件对应的监听器列表
           protected Collection<ApplicationListener<?>> getApplicationListeners(
                   ApplicationEvent event, ResolvableType eventType) {
       
               Object source = event.getSource();
               Class<?> sourceType = (source != null ? source.getClass() : null);
               // 以 eventType 事件类型、sourceType 事件源类型构造索引 key
               ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);
       
               // 缓存查找
               ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
               if (retriever != null) {
                   return retriever.getApplicationListeners();
               }
       
               if (this.beanClassLoader == null ||
                       (ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
                               (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
                   // 加锁，保证线程安全
                   synchronized (this.retrievalMutex) {
                       // double check
                       retriever = this.retrieverCache.get(cacheKey);
                       if (retriever != null) {
                           return retriever.getApplicationListeners();
                       }
                       // 缓存未命中，构建新 ListenerRetriever 从新遍历监听器列表，符合事件类型的监听器被此对象收集归类
                       retriever = new ListenerRetriever(true);
                       Collection<ApplicationListener<?>> listeners =
                               retrieveApplicationListeners(eventType, sourceType, retriever);
                       // 放入缓存对象
                       this.retrieverCache.put(cacheKey, retriever);
                       return listeners;
                   }
               }
               ...
           }
       
           // 遍历收集所有监听器，按搜索目标事件对应的监听器（从当前暂存的监听器列表、Bean 工厂中的监听器 Bean 中搜寻）
           private Collection<ApplicationListener<?>> retrieveApplicationListeners(
                   ResolvableType eventType, @Nullable Class<?> sourceType, @Nullable ListenerRetriever retriever) {
       
               LinkedList<ApplicationListener<?>> allListeners = new LinkedList<>();
               Set<ApplicationListener<?>> listeners;
               Set<String> listenerBeans;
               synchronized (this.retrievalMutex) {
                   // 将之前暂存的监听器放入列表
                   listeners = new LinkedHashSet<>(this.defaultRetriever.applicationListeners);
                   // 将之前暂存的监听器 Bean 放入列表
                   listenerBeans = new LinkedHashSet<>(this.defaultRetriever.applicationListenerBeans);
               }
               for (ApplicationListener<?> listener : listeners) {
                   // 寻找目标事件的监听器，放入归类对象 retriever 中
                   if (supportsEvent(listener, eventType, sourceType)) {
                       if (retriever != null) {
                           retriever.applicationListeners.add(listener);
                       }
                       allListeners.add(listener);
                   }
               }
               if (!listenerBeans.isEmpty()) {
                   // 从 BeanFactory 中搜索
                   BeanFactory beanFactory = getBeanFactory();
                   for (String listenerBeanName : listenerBeans) {
                       try {
                           Class<?> listenerType = beanFactory.getType(listenerBeanName);
                           if (listenerType == null || supportsEvent(listenerType, eventType)) {
                               ApplicationListener<?> listener =
                                       beanFactory.getBean(listenerBeanName, ApplicationListener.class);
                               // 寻找目标事件的监听器，放入归类对象 retriever 中
                               if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
                                   if (retriever != null) {
                                       retriever.applicationListenerBeans.add(listenerBeanName);
                                   }
                                   allListeners.add(listener);
                               }
                           }
                       }
                       ...
                   }
               }
               // 对监听器排序
               AnnotationAwareOrderComparator.sort(allListeners);
               return allListeners;
           }
       }
       ```

       &emsp;&emsp;可以看出，广播器 *EventPublishingRunListener* 缓存中包含已事件类型归类的多个 *ListenerRetriever*，而每个 *ListenerRetriever* 中又包含相同事件类型的多个监听器集合，所以广播器是借由 *ListenerRetriever* 对象关联多个监听器列表。那么接下来看广播器如何广播 *ApplicationEvent* 事件给对应的多个监听器。

       

     - 广播 *ApplicationEvent*

       &emsp;&emsp;前面提到是由子类 *SimpleApplicationEventMulticaster* 来广播事件，具体的广播方法为：

       ```java
       public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
       	...
           // 广播指定事件类型的事件
           @Override
           public void multicastEvent(ApplicationEvent event) {
               // 转调重载方法
               multicastEvent(event, resolveDefaultEventType(event));
           }
       
           @Override
           // 重载方法
           public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
               ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
               // 调用父类 getApplicationListeners 获取 event 类型的所有监听器列表
               for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
                   Executor executor = getTaskExecutor();
                   // 异步执行器不为空时，异步执行监听器方法
                   if (executor != null) {
                       executor.execute(() -> invokeListener(listener, event));
                   }
                   else {
                       // 同步调用监听器方法，将事件注入
                       invokeListener(listener, event);
                   }
               }
           }
       
           protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
       		...
       		else {
                   // 转调 doInvokeListener
                   doInvokeListener(listener, event);
               }
           }
       
           @SuppressWarnings({"unchecked", "rawtypes"})
           private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
               try {
                   // 执行监听器的 onApplicationEvent 方法，并将当前事件传入
                   listener.onApplicationEvent(event);
               }
       		...
           }
       }
       ```

       &emsp;&emsp;可以看到，它会调用父类 *AbstractApplicationEventMulticaster* 的 `getApplicationListeners` 方法获取指定事件的监听器列表，而该方法即为上文提及能触发归类、缓存监听器的方法。到此就完成了 Spring 事件/监听 机制从监听器注册到事件广播的整个流程描述，然而 Spring 上下文事件由谁创建并交给广播器 *SimpleApplicationEventMulticaster* 进行广播的呢？

       

     - *SimpleApplicationEventMulticaster* 与 ApplicationContext 的关系

       &emsp;&emsp;首先来看 Spring 上下文事件定义：

       ```java
       // Spring 应用上下文事件
       public abstract class ApplicationContextEvent extends ApplicationEvent {
           // Spring 应用上下文事件的事件源为 ApplicationContext
           public ApplicationContextEvent(ApplicationContext source) {
               super(source);
           }
           ...
       }
       ```

       &emsp;&emsp;可以看出，它为抽象类，所关联的事件源为 *ApplicationContext*。那么从继承树入手，查询它的所有子实现类：

       ```
       ApplicationContextEvent
           | -> ContextRefreshedEvent	上下文就绪事件
           | -> ContextStartedEvent	上下文启动事件
           | -> ContextStoppedEvent	上下文停止事件
           | -> ContextClosedEvent		上下文关闭事件
       ```

       &emsp;&emsp;跟踪 *ContextRefreshedEvent* 构造方法调用堆栈，发现是在 *AbstractApplicationContext* 的方法 `finishRefresh` 中创建，并且创建完立刻调用发布方法 `publishEvent` 进行事件发布，而该方法由接口 *ApplicationEventPublisher* 定义，并也在此类实现。

       

       &emsp;&emsp;接口 *ApplicationEventPublisher* 定义：

       ```java
       @FunctionalInterface
       public interface ApplicationEventPublisher {
           // 发布 ApplicationEvent 类型的事件
           default void publishEvent(ApplicationEvent event) {
               publishEvent((Object) event);
           }
           // 发布 Object 类型的事件
           void publishEvent(Object event);
       }
       ```

       &emsp;&emsp;*AbstractApplicationContext* 创建 *ContextRefreshedEvent* 事件，并调用自身实现接口 *ApplicationEventPublisher* 的发布方法：

       ```java
       public abstract class AbstractApplicationContext extends DefaultResourceLoader
               implements ConfigurableApplicationContext {
       
           // 关联广播器
           private ApplicationEventMulticaster applicationEventMulticaster;
           
           ...
           protected void finishRefresh() {
               ...
               // 创建上下文就绪事件，并发布
               publishEvent(new ContextRefreshedEvent(this));
               ...
           }
       
           @Override
           // 发布方法为 ApplicationEventPublisher 接口定义，在此实现
           public void publishEvent(ApplicationEvent event) {
               publishEvent(event, null);
           }
       
           // 发布方法为 ApplicationEventPublisher 接口定义，在此实现
           protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
       
               // 判断事件类型
               ApplicationEvent applicationEvent;
               if (event instanceof ApplicationEvent) {
                   applicationEvent = (ApplicationEvent) event;
               }
               else {
                   // 如果非 ApplicationEvent 类型，包装为 PayloadApplicationEvent 类型
                   applicationEvent = new PayloadApplicationEvent<>(this, event);
                   if (eventType == null) {
                       eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
                   }
               }
       
               // 广播器尚未初始化时，放入暂存事件列表，待广播器初始化后再进行广播
               if (this.earlyApplicationEvents != null) {
                   this.earlyApplicationEvents.add(applicationEvent);
               }
               else {
                   // 获取广播器立即广播
                   getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
               }
               ...
           }
           // 返回本类关联的广播器实例
           ApplicationEventMulticaster getApplicationEventMulticaster() throws IllegalStateException {
               return this.applicationEventMulticaster;
           }
       }
       ```

       &emsp;&emsp;从事件发布实现方法中可知，事件交由成员变量 `this.applicationEventMulticaster` 进行广播，而该成员变量初始化是在：

       ```java
       public abstract class AbstractApplicationContext extends DefaultResourceLoader
               implements ConfigurableApplicationContext {
           // 模版方法定义上下文生命周期方法
           public void refresh() throws BeansException, IllegalStateException {
               synchronized (this.startupShutdownMonitor) {
                   prepareRefresh();
                   ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
                   prepareBeanFactory(beanFactory);
       
                   try {
                       postProcessBeanFactory(beanFactory);
                       invokeBeanFactoryPostProcessors(beanFactory);
                       registerBeanPostProcessors(beanFactory);
                       initMessageSource();
                       // 初始化事件广播器
                       initApplicationEventMulticaster();
                       onRefresh();
                       // 将广播器未创建之前发生的事件，交由已生成的广播器进行广播
                       registerListeners();
       
                       finishBeanFactoryInitialization(beanFactory);
                       finishRefresh();
                   }
                   ...
               }
           }
       
           // 实例化广播器
           protected void initApplicationEventMulticaster() {
               ConfigurableListableBeanFactory beanFactory = getBeanFactory();
               // BeanFactory 中查询名称为 applicationEventMulticaster 的广播器 Bean
               if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
                   this.applicationEventMulticaster =
                           beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
               }
               else {
                   // 【重要】BeanFactory 中未找到时，把类型为 SimpleApplicationEventMulticaster 的广播器声明为该名称的 Bean
                   this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
                   beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
               }
           }
       
           // 收集监听器并添加到广播器中，并且将早期发生的事件通过广播器进行广播
           protected void registerListeners() {
               // 通过上下文注册的监听器，添加到广播器
               for (ApplicationListener<?> listener : getApplicationListeners()) {
                   getApplicationEventMulticaster().addApplicationListener(listener);
               }
               // BeanFactory 定义的监听器 Bean, 添加到广播器
               String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
               for (String listenerBeanName : listenerBeanNames) {
                   getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
               }
               // 广播器生成前发生的事件，推迟到此时进行广播
               Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
               // 已广播后，将暂存的已发生事件清空
               this.earlyApplicationEvents = null;
               if (earlyEventsToProcess != null) {
                   for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
                       getApplicationEventMulticaster().multicastEvent(earlyEvent);
                   }
               }
           }
       }
       ```

       &emsp;&emsp;由 `initApplicationEventMulticaster` 方法得出，成员变量 `this.applicationEventMulticaster` 被 BeanFactory 中名为 `applicationEventMulticaster` 类型为 *SimpleApplicationEventMulticaster* 的对象赋值。**换言之，*ApplicationContext* 与 *SimpleApplicationEventMulticaster* 的联系，是借由抽象实现类 *AbstractApplicationContext* 的成员变量来关联，应用上下文产生的上下文事件被 *ApplicationEventPublisher* 接口实现类 *AbstractApplicationContext* 的方法 `publishEvent()` 发布，而具体发布操作又是委托给该成员变量广播给相应的监听器。**

       

     &emsp;&emsp;此时再来回答 ***理解 Spring Boot 事件*** 小节最后提出的问题：

     > 已知 Spring Boot 事件通过 *SimpleApplicationEventMulticaster* 广播，那么 Spring 上下文事件由谁广播呢？
     >
     > &emsp;&emsp;答：Spring 上下文事件由 *AbstractApplicationContext* 创建，而该类实现事件发布接口 *ApplicationEventPublisher*，在其发布方法中，也是委托 *SimpleApplicationEventMulticaster* 进行事件广播。即 Spring Boot 事件、Spring 上下文事件最终都被相同类型的广播器进行广播。

     

     &emsp;&emsp;既然 Spring Boot 事件、Spring 上下文事件都由 ***SimpleApplicationEventMulticaster*** 进行广播，那么这两个广播器会是 BeanFactory 中的相同对象吗？暂且保留此疑问，先将 **理解 Spring 事件/监听 机制**  小节剩余部分描述完，到 **理解 Spring Boot 事件/监听 机制** 小节再做回答。

  

  2. Spring 内建事件

     &emsp;&emsp;在上小节讨论上下文与广播器关系时已经找出很多 Spring 内建的事件，下面在此汇总：

     | Spring 内建事件           | 说明                                                         |
     | ------------------------- | ------------------------------------------------------------ |
     | *ContextRefreshedEvent*   | 当 *ApplicationContext* *已初始化* 及 *更新后* 被触发，例如：在 *ConfigurableApplicationContext* 的 `refresh()` 执行完后。“已初始化” 指所有 beans 都已装载、post-processor beans 都已被检测激活、单例已被初始化，ApplicationContext 对象以可以使用。只要该上下文未被关闭，上下文可以被更新 `refresh` 多次，也即所谓的热更新支持。并不是所有上下文实现类都支持热更新，例如：*GenericApplicationContext* 就不支持。 |
     | *ContextStartedEvent*     | 当 *ApplicationContext* 被 *启动* 时触发，一般是调用 *AbstractApplicationContext* 的 `start()` 时触发。“启动” 指的是生命周期 beans 接收到特定的启动信号。典型使用此场景是被执行生命周期停止信号后发出重新启动时，或者一些组件在初始化时未被启用，直到收到此启动信号才开始启用激活。 |
     | *ContextStoppedEvent*     | 当 *ApplicationContext* 被 *停止* 时触发，一般是调用 *AbstractApplicationContext* 的 `stop()` 时触发。“停止” 指的是生命周期 beans 接收到特定的停止信息。停止的上下文可以通过上面所说的启动信号进行重新启动。 |
     | *ContextClosedEvent*      | 当 *ApplicationContext* 被 *关闭* 时触发，一般是调用 *ConfigurableApplicationContext* 的 `close()` 时触发。“关闭” 指的是所有的单例 beans 被销毁。被关闭的上下文也是其生命周期的终点，它不能再被重启或更新。 |
     | *PayloadApplicationEvent* | 这是一个 Spring 事件包装类，它将上下文发布的非 Spring 事件类型的对象封装成 Spring 事件，从而获得与 Spring 事件一样的事件广播流程处理。 |
     | *RequestHandledEvent*     | Web 应用特有的事件，在请求被处理完毕后触发，通知所有 beans Http 请求已完成。该事件仅在使用 Spring 的 *DispatcherServlet* 的 Web 应用中被运用。 |

     &emsp;&emsp;上面除最后两个事件外，前面四个均继承 *ApplicationContextEvent*，属于 Spring 上下文事件，且事件源都是 ApplicationContext，下面简要说明他们的运用场景或最佳实践：

     

     - *ContextRefreshedEvent*

       &emsp;&emsp;`refresh()` 方法执行最后被触发，此时 Spring 应用上下文中的 Bean 均已完成初始化，并能投入使用。故一般实现此类型事件的监听器来获得需要的 Bean 能避免所需 Bean 被提早初始化的潜在风险。

       

     - ContextStartedEvent 与 *ContextStoppedEvent*

       &emsp;&emsp;这两者分别在 *AbstractApplicationContext* 的 `start()`、`stop()` 方法中发布，这俩方法会触发 *LifeCycle* Bean 的相应方法被执行。值得注意的是，虽然 *AbstractApplicationContext* 自身也实现 *LifeCycle* 接口，但它却不是 LifeCycle Bean，所以它自身 `start()`、`stop()`操作触发的生命周期方法执行不会递归调用到自身。而且这两个方法在 Spring Boot 中没有出现，而是被运用到 Spring Cloud 的 “resume”、“pause” 两个 *Endpoint* 场景，而该场景是通过监听 *ContextRefreshedEvent* 来实现，间接证明 *ContextRefreshedEvent* 事件发生在 *ContextStartedEvent*、*ContextStoppedEvent* 两事件之前。

       

     - *ContextClosedEvent*

       &emsp;&emsp;它由 *ConfigurableApplicationContext* 的子类 *AbstractApplicationContext* 中 `close()` 实现方法中发布，此事件出现表面所有 LifeCycle Bean 已进入 `onClose()` 声明周期回调。一般该方法是由 JVM 关闭时回调所激发，但是如果主动调用上下文的 `close()` 时，那么相应为了防止循环调用，会在其中取消已注册的 JVM 的  ShutdownHook 线程。这同时表明，一旦执行此操作就无法重启该应用上下文了。

       

  3. 自定义 Spring 事件

     &emsp;&emsp;通过扩展抽象类 *ApplicationEvent* 就可以自定义 Spring 事件，而上文可知 ***AbstractApplicationContext*** 实现了接口 *ApplicationEventPublisher* 具备事件发布能力，而可配置的 *ConfigurableApplicationContext* 应用上下文接口的实现类都继承于改抽象类，故**可配置的应用上下文非抽象实现类**都可以发布自定义 Spring 事件。

      

  4. Spring 事件监听

     &emsp;&emsp;通过前面讨论实际已经了解，要对 Spring 事件进行监听，实现监听接口 *ApplicationListener* 是一种途径，属接口编程的方式。实际上，Spring Framework 4.2 开始引入 *@EventListener* 注解方式实现事件监听，属注解驱动编程的方式。

     

     - 监听 Spring 内建应用上下文事件

       &emsp;&emsp;从 ***SimpleApplicationEventMulticaster* 与 ApplicationContext 的关系** 小节可知，上下文关联的广播器初始化后会将初始化前通过上下文注册的监听器及 BeanFactory 中包含的所有监听器类型的 Bean 添加到该广播器中。那么先示范**通过上下文注册的方式添加一个监听上下文事件的 lambda 形式的监听器**：

       ```java
       public class ApplicationListenerOnSpringEventsBootstrap {
       
           public static void main(String[] args) {
               // 创建 ConfigurableApplicationContext 实例 GenericApplicationContext， 不支持多次 refresh
               ConfigurableApplicationContext context = new GenericApplicationContext();
       
               // 创建 ConfigurableApplicationContext 实例 ClassPathXmlApplicationContext，支持多次执行 refresh
               // ConfigurableApplicationContext context = new ClassPathXmlApplicationContext();
       
               System.out.println("创建 Spring 应用上下文 : " + context.getDisplayName());
               // 添加 ApplicationListener 非泛型实现
               context.addApplicationListener(event ->
                       System.out.println(event.getClass().getSimpleName())
               );
       
               // refresh() : 初始化应用上下文
               System.out.println("应用上下文准备初始化...");
               context.refresh(); // 发布 ContextRefreshedEvent
               System.out.println("应用上下文已初始化...");
       
               // stop() : 停止应用上下文
               System.out.println("应用上下文准备停止启动...");
               context.stop();    // 发布 ContextStoppedEvent
               System.out.println("应用上下文已停止启动...");
       
       
               // start(): 启动应用上下文
               System.out.println("应用上下文准备启动启动...");
               context.start();  // 发布 ContextStartedEvent
               System.out.println("应用上下文已启动启动...");
       
       
               // close() : 关闭应用上下文
               System.out.println("应用上下文准备关闭...");
               context.close();  // 发布 ContextClosedEvent
               System.out.println("应用上下文已关闭...");
       
               // refresh() : 再次初始化应用上下文
               // System.out.println("应用上下文再次准备初始化...");
               // context.refresh(); // 发布 ContextRefreshedEvent
               // System.out.println("应用上下文已再次初始化...");
           }
       }
       ```

       &emsp;&emsp;lambda 形式的监听器，它的 `onApplicationEvent(E event)` 方法参数为 *ApplicationEvent* 本身，即监听所有类型的 Spring 事件。值得注意的是，*GenericApplicationContext* 是不支持多次执行 `refresh()` 方法更新上下文的，而 *ClassPathXmlApplicationContext* 是可以支持，即使上下文已调用了 `close()` 方法之后也能更新成功。

       

       > [ApplicationListenerOnSpringEventsBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/context/event/ApplicationListenerOnSpringEventsBootstrap.java)

       

     - 监听自定义 Spring 事件的监听器

       &emsp;&emsp;上例演示了通过上下文的 `addApplicationListener` 方法添加监听器，本例将示范**通过注册监听器 Bean 的方式**来监听自定义 Spring 泛型事件。 同时观察在上下文关闭后再次发布自定义事件，来验证此时能否广播给监听器处理：

       ```java
       public class ApplicationListenerBeanOnCustomEventBootstrap {
       
           public static void main(String[] args) {
               // 创建 Spring 应用上下文 GenericApplicationContext
               GenericApplicationContext context = new GenericApplicationContext();
               // 注册 ApplicationListener<MyApplicationEvent> 实现 MyApplicationListener
               context.registerBean(MyApplicationListener.class); // registerBean 方法从 Spring 5 引入
               // 初始化上下文
               context.refresh();
               // 发布自定义事件 MyApplicationEvent
               context.publishEvent(new MyApplicationEvent("Hello World"));
               // 关闭上下文
               context.close();
               // 再次发布事件
               context.publishEvent(new MyApplicationEvent("Hello World Again"));
           }
       
           // 自定义事件
           public static class MyApplicationEvent extends ApplicationEvent {
               // 父类必须指定事件源
               public MyApplicationEvent(String source) {
                   super(source);
               }
           }
       
           // 定义指定监听事件为自定义事件的监听器，通过泛型参数设定监听类型
           public static class MyApplicationListener implements ApplicationListener<MyApplicationEvent> {
       
               @Override
               // 方法参数类型为类上泛型所设定的事件类型
               public void onApplicationEvent(MyApplicationEvent event) {
                   System.out.println(event.getClass().getSimpleName());
               }
           }
       }
       ```

       &emsp;&emsp;运行后发现，上下文关闭前自定义事件能被监听器处理，而关闭后发布的事件虽然没有报错，但监听器已无法收到事件广播。那么究竟关闭上下文时，Spring 做了什么操作切断了广播器与监听器之间的联系呢？下一小节揭开疑问。

       

       > [ApplicationListenerBeanOnCustomEventBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/context/event/ApplicationListenerBeanOnCustomEventBootstrap.java)

       

     - 广播器与监听器断开广播联系的原理

       &emsp;&emsp;Spring 上下文调用关闭时，入口方法为 `close()` 方法，跟踪此方法可知：

       ```java
       public abstract class AbstractApplicationContext extends DefaultResourceLoader
               implements ConfigurableApplicationContext {
           @Override
           public void close() {
               synchronized (this.startupShutdownMonitor) {
                   // 1. 执行关闭
                   doClose();
                   ...
               }
           }
       
           protected void doClose() {
               if (this.active.get() && this.closed.compareAndSet(false, true)) {
                   ...
                   // 2. 销毁所有当前上下文的 BeanFactory 缓存的单例 Bean
                   destroyBeans();
                   ...
               }
           }
       
           protected void destroyBeans() {
               // 3. 获得 BeanFactory 调用 destroySingletons
               getBeanFactory().destroySingletons();
           }
       }
       ```

       &emsp;&emsp;跟踪调用堆栈，发现 `destroySingletons()` 方法最终会执行到广播器的移除监听器和监听器 Bean 的方法：

       ```java
       class ApplicationListenerDetector implements DestructionAwareBeanPostProcessor, MergedBeanDefinitionPostProcessor {
           ...
           @Override
           public void postProcessBeforeDestruction(Object bean, String beanName) {
               if (bean instanceof ApplicationListener) {
                   try {
                       // 获得上下文的广播器，即 SimpleApplicationEventMulticaster
                       ApplicationEventMulticaster multicaster = this.applicationContext.getApplicationEventMulticaster();
                       // 删除监听器
                       multicaster.removeApplicationListener((ApplicationListener<?>) bean);
                       // 删除监听器 Bean
                       multicaster.removeApplicationListenerBean(beanName);
                   }
                   catch (IllegalStateException ex) {
                       ...
                   }
               }
           }
           ...
       }
       ```

       &emsp;&emsp;由于广播器移除了事件监听器，这就切断了广播器与监听器之间的联系，所以之后发布的事件在广播器中无法找到相应的监听器而不再被监听。至此，接口编程方式的监听器实现已经讲完，接下来将详细讲解注解驱动监听器实现。

       

     - *@EventListener* 实现 Spring 事件监听

       &emsp;&emsp;下面利用注解驱动方式实现监听器，同时为了验证 *@EventListener* 对被注解的方法的限制要求，特意将注解标注在抽象类的方法、含返回类型的方法、不同访问修饰符的方法上，具体代码如下：

       ```java
       public class AnnotatedEventListenerBootstrap {
       
           public static void main(String[] args) {
               // 创建 注解驱动 Spring 应用上下文
               AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
               // 注册 @EventListener 类 MyEventListener
               context.register(MyEventListener.class);
               // 初始化上下文
               context.refresh();
               // 关闭上下文
               context.close();
           }
       
           public static abstract class AbstractEventListener {
       
               // 1. 抽象类中方法也可以
               @EventListener(ContextRefreshedEvent.class)
               public void onContextRefreshedEvent(ContextRefreshedEvent event) {
                   System.out.println("AbstractEventListener : " + event.getClass().getSimpleName());
               }
           }
       
           public static class MyEventListener extends AbstractEventListener {
       
               // 2. 带返回值的 public 方法也可以
               @EventListener(ContextClosedEvent.class)
               public boolean onContextClosedEvent(ContextClosedEvent event) {
                   System.out.println("MyEventListener on public boolean Method: " + event.getClass().getSimpleName());
                   return true;
               }
       
               // 3. protected 方法也可以
               @EventListener(ContextClosedEvent.class)
               protected void onProtectedContextClosedEvent(ContextClosedEvent event) {
                   System.out.println("MyEventListener on protected Method: " + event.getClass().getSimpleName());
               }
       
               // 4. default 方法也可以
               @EventListener(ContextClosedEvent.class)
               void onDefaultContextClosedEvent(ContextClosedEvent event) {
                   System.out.println("MyEventListener on default Method: " + event.getClass().getSimpleName());
               }
       
               // 5. 甚至 private 方法也可以
               @EventListener(ContextClosedEvent.class)
               private void onPrivateContextClosedEvent(ContextClosedEvent event) {
                   System.out.println("MyEventListener on private Method: " + event.getClass().getSimpleName());
               }
           }
       }
       ```

       &emsp;&emsp;运行结果显示，*@EventListener* 标注在以上五种方法签名上都能正常工作。先不急下定论，方法签名还包括方法参数，目前都是指定监听一个事件，故参数都只设置成监听的那个事件，那么想要在一个监听方法中监听多个事件，是否能设置多个参数呢？下一小节接着实验。

       

       > [AnnotatedEventListenerBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/context/event/AnnotatedEventListenerBootstrap.java)

       

     - *@EventListener* 监听多个 Spring 事件

       &emsp;&emsp;下面设置无参、单参、两个参数形式的方法，观察结果：

       ```java
       public class AnnotatedEventListenerOnMultiEventsBootstrap {
       
           public static void main(String[] args) {
               // 创建 注解驱动 Spring 应用上下文
               AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
               // 注册 @EventListener 类 MyMultiEventsListener
               context.register(MyMultiEventsListener.class);
               // 初始化上下文
               context.refresh();
               // 关闭上下文
               context.close();
           }
       
           public static class MyMultiEventsListener {
       
               // 设置无参形式
               @EventListener({ContextRefreshedEvent.class, ContextClosedEvent.class})
               public void onEvent() {
                   System.out.println("onEvent");
               }
       
               // 设置单个参数，此参数类型为两个事件的父类
               @EventListener({ContextRefreshedEvent.class, ContextClosedEvent.class})
               public void onApplicationContextEvent(ApplicationContextEvent event) {
                   System.out.println("onApplicationContextEvent : " + event.getClass().getSimpleName());
               }
       
               // 设置两个参数，报错：Maximum one parameter is allowed for event listener method
               @EventListener({ContextRefreshedEvent.class, ContextClosedEvent.class})
               public void onEvents(ContextRefreshedEvent refreshedEvent, ContextClosedEvent contextClosedEvent) {
                   System.out.println("onEvents : " + refreshedEvent.getClass().getSimpleName()
                           + " , " + contextClosedEvent.getClass().getSimpleName());
               }
           }
       }
       ```

       &emsp;&emsp;运行发现无参、单参的方法可以支持，但两个参数的方法报错，提示最多仅支持一个参数。目前事件监听器注解对方法签名的要求为：访问修饰符无限制、允许返回值、参数支持不超过一个。下一节再叠加异步注解，观察是否都方法签名的要求是否收窄。

       

       > [AnnotatedEventListenerOnMultiEventsBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/context/event/AnnotatedEventListenerOnMultiEventsBootstrap.java)

       

     - *@EventListener* 异步监听

       &emsp;&emsp;结合访问修饰符、返回值的不同情况，开启异步支持：

       ```java
       public class AnnotatedAsyncEventListenerBootstrap {
       
           public static void main(String[] args) {
               // 创建 注解驱动 Spring 应用上下文
               AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
               // 注册 异步 @EventListener 类 MyAsyncEventListener
               context.register(MyAsyncEventListener.class);
               context.register(AnnotatedAsyncEventListenerBootstrap.class);
               println(" Spring 应用上下文正在初始化...");
               // 初始化上下文
               context.refresh();
               // 关闭上下文
               context.close();
           }
       
           @EnableAsync() // 需要激活异步，否则 @Async 无效
           public static class MyAsyncEventListener {
       
               @EventListener(ContextRefreshedEvent.class)
               @Async
               // 1. 原始类型返回值的方法
               // 运行期 AopInvocationExceptionNull 异常： return value from advice does not match primitive return type
               public boolean onPrimitiveTypeContextRefreshedEvent(ContextRefreshedEvent event) {
                   println(" MyAsyncEventListener on primitive boolean type: " + event.getClass().getSimpleName());
                   return true;
               }
       
               @EventListener(ContextRefreshedEvent.class)
               @Async
               // 2. 装箱后的返回值类型的方法
               public Boolean onContextRefreshedEvent(ContextRefreshedEvent event) {
                   println(" MyAsyncEventListener on boxing Boolean type: " + event.getClass().getSimpleName());
                   return true;
               }
       
               @EventListener(ContextRefreshedEvent.class)
               @Async
               // 3. protected 访问修饰符的方法
               protected void onProtectedContextRefreshedEvent(ContextRefreshedEvent event) {
                   println(" MyAsyncEventListener on protected Method: " + event.getClass().getSimpleName());
               }
       
               @EventListener(ContextRefreshedEvent.class)
               @Async
                   // 4. default 的方法
               void onDefaultContextRefreshedEvent(ContextRefreshedEvent event) {
                   println(" MyAsyncEventListener on default Method: " + event.getClass().getSimpleName());
               }
       
               @EventListener(ContextRefreshedEvent.class)
               @Async
               // 5. private 访问修饰符的方法
               // 编译报错：@Async 要求方法必须能被重写，private 方法不支持
               private void onPrivateContextRefreshedEvent(ContextRefreshedEvent event) {
                   println(" MyAsyncEventListener on private Method: " + event.getClass().getSimpleName());
               }
           }
       
           private static void println(String content) {
               // 当前线程名称
               String threadName = Thread.currentThread().getName();
               System.out.println("[ 线程 " + threadName + " ] : " + content);
           }
       }
       ```

       &emsp;&emsp;首先，第 5 种 `private` 修饰的方法没法通过编译，将其注释后再次运行。

       &emsp;&emsp;然后，发现只有第 1、2 两个方法有打印，虽然第 1 种原始类型返回类型方法体顺利执行，但抛出运行期 *AopInvocationException* 异常（Null 返回值不能匹配原始返回类型），打印结果显示这两个监听器的确是异步执行，日志表明：当容器中未存在名称为 `taskExecutor` 类型为 *TaskExecutor* 的 Bean 时，会生成 *SimpleAsyncTaskExecutor* 类型的默认执行器。将第 1 种原始类型的方法注释再次运行。

       &emsp;&emsp;最后，发现封装型有返回值、`protected`、`default` 型的方法也得可运行。

       

       > [AnnotatedAsyncEventListenerBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/context/event/AnnotatedAsyncEventListenerBootstrap.java)

       

     - *@EventListener* 监听排序

       &emsp;&emsp;注解形式的监听器方法支持排序，可以在其标注的方法增加 *@Order* 注解，另外接口编程形式的监听器除支持 *@Order* 注解外，还可以实现 *Ordered* 接口来指定顺序（值越小优先级越高）。至于排序是如何实现、不标注排序时默认是什么顺序将在 ***@EventListener* 实现监听的原理** 小节一并分析。

       

     - *@EventListener* 监听 Spring 泛型事件

       &emsp;&emsp;与接口编程一样，注解驱动型也是支持监听带泛型的事件，而且广播器甚至支持非 *ApplicationEvent* 事件的对象的发布。前文已经提过实现方式是把该对象封装成 *PayloadApplicationEvent* 事件，该对象称为 *PayloadApplicationEvent* 事件的 `payload`，而将上下文对象作为该事件的 `source` 事件源（小马哥在 P479 中称该对象为事件源其实与语义不符），再由广播器广播。准确的说广播器最终广播的还是 *ApplicationEvent* 事件。

       ```java
       public class GenericEventListenerBootstrap {
       
           public static void main(String[] args) {
               // 创建 注解驱动 Spring 应用上下文
               AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
               // 注册 UserEventListener，即实现 ApplicationListener ，也包含 @EventListener 方法
               context.register(UserEventListener.class);
               // 初始化上下文
               context.refresh();
               // 构造泛型事件
               GenericEvent<User> event = new GenericEvent(new User("小马哥"));
               // 发送泛型事件
               context.publishEvent(event);
               // 发送 User 对象作为事件源
               context.publishEvent(new User("mercyblitz"));
               // 关闭上下文
               context.close();
           }
       
           public static class UserEventListener implements ApplicationListener<GenericEvent<User>> {
       
               @EventListener
               // 注解形式监听非事件类型的对象
               public void onUser(User user) {
                   System.out.println("onUser : " + user);
               }
       
               @EventListener
               // 注解形式监听自定义的泛型事件
               public void onUserEvent(GenericEvent<User> event) {
                   System.out.println("onUserEvent : " + event.getSource());
               }
       
               @Override
               // 接口形式监听自定义的泛型事件
               public void onApplicationEvent(GenericEvent<User> event) {
                   System.out.println("onApplicationEvent : " + event.getSource());
               }
       
       /*
           @Override
           // 接口形式监听非事件类型的对象，借由 PayloadApplicationEvent
           // 【注意】需要将 UserEventListener 的 implements 修改为 ApplicationListener<PayloadApplicationEvent>
           public void onApplicationEvent(PayloadApplicationEvent event) {
               System.out.println("onApplicationEvent : " + event.getPayload());
           }
        */
           }
       
           public static class User {
       
               private final String name;
       
               public User(String name) {
                   this.name = name;
               }
       
               @Override
               public String toString() {
                   return "User{name='" + name + "\'}";
               }
           }
       
           // PayloadApplicationEvent 也实现 ResolvableTypeProvider 来达到提取泛型参数实际类型
           public static class GenericEvent<T>
                   extends ApplicationEvent implements ResolvableTypeProvider {
       
               public GenericEvent(T source) {
                   super(source);
               }
       
               // 实现 ResolvableTypeProvider 方法，返回包含泛型参数类型的事件类型
               @Override
               public ResolvableType getResolvableType() {
                   return ResolvableType.forClassWithGenerics(getClass(),
                           ResolvableType.forInstance(getSource()));
               }
       
               @Override
               public T getSource() {
                   return (T) super.getSource();
               }
           }
       }
       ```

       &emsp;&emsp;此处自定义的泛型事件 *GenericEvent* 有别于 *PayloadApplicationEvent*，它直接把传入的对象充当事件源。另外，监听泛型事件时需要实现 *ResolvableTypeProvider* 接口来暴露泛型元信息，从而使得监听器的方法参数能够进一步使用粒度为精确到泛型参数类型的匹配策略，原因来自事件发布时关联相关监听器的查找匹配逻辑：

       ```java
       public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
       
           @Override
           public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
               // eventType 未指定时，调用 resolveDefaultEventType(event) 解析 event 类型
               ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
               for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
                   Executor executor = getTaskExecutor();
                   if (executor != null) {
                       executor.execute(() -> invokeListener(listener, event));
                   }
                   else {
                       invokeListener(listener, event);
                   }
               }
           }
       
           // 转调用 ResolvableType 解析事件类型
           private ResolvableType resolveDefaultEventType(ApplicationEvent event) {
               return ResolvableType.forInstance(event);
           }
       }
       ```

       &emsp;&emsp;而 `ResolvableType.forInstance(Object)` 检查事件类型具体方法：

       ```java
       public class ResolvableType implements Serializable {
           
           public static ResolvableType forInstance(Object instance) {
               // instance 实现 ResolvableTypeProvider 时，会调用接口方法获得指定的类型
               // 通过此接口方法，可以更进一步提供包含泛型参数类型的 type，使得监听器匹配更精确
               if (instance instanceof ResolvableTypeProvider) {
                   ResolvableType type = ((ResolvableTypeProvider) instance).getResolvableType();
                   if (type != null) {
                       return type;
                   }
               }
               return ResolvableType.forClass(instance.getClass());
           }
       }
       ```

       &emsp;&emsp;可以看出，`GenericEvent<T>` 实现了 *ResolvableTypeProvider* 接口的方法，该方法返回含泛型参数 `T` 具体类型的事件类型 `GenericEvent<User>`，才使得接口实现的类型监听器一旦带有泛型参数时，泛型参数只能是 *User* 类型才能正常监听。如果取消此接口的实现，那么泛型参数即使指定的不是 *User* 类型也能正常监听，因为发布事件时事件对象类型中已丢失泛型参数类型，而导致泛型类型参数的过滤已失效。

       

     - *@EventListener* 与 *ApplicationListener* 实现监听区别

       &emsp;&emsp;到此，汇总一下接口形式与注解形式实现的监听器的区别：

       | 监听类型                  | 访问性     | 顺序控制          | 返回类型   | 参数数量 | 参数类型               | 泛型参数类型 |
       | ------------------------- | ---------- | ----------------- | ---------- | -------- | ---------------------- | ------------ |
       | *@EventListener* 同步方法 | 任意       | @Order            | 任意       | 0 或 1   | 事件类型或泛型参数类型 | 支持         |
       | *@EventListener* 异步方法 | 非 private | @Order            | 非原生类型 | 0 或 1   | 事件类型或泛型参数类型 | 支持         |
       | *ApplicationListener*     | public     | @Order 或 Ordered | void       | 1        | 事件类型               | 不支持       |

       &emsp;&emsp;***泛型参数类型***：指监听通过接口 *ApplicationEventPublisher* 的 `publishEvent(Object)` 方法直接发布的**泛型事件的泛型参数类型**。 

       

       &emsp;&emsp;同步方法访问性为 **任意** 是有一个前提条件：`private` 修饰时其所在类不能被 Spring AOP 动态代理，否则得加上 `static`，详细会在 ***@EventListener* 实现监听的原理** 部分进行说明。

       

       

       ```
       例如：
       	示例中定义了 GenericEvent<User> 泛型事件，它的泛型参数类型为 User。
       	
       	注解方式：支持泛型参数类型的监听方法
           		public void onUser(User user) √
           		
       	接口方式：不支持泛型参数类型的监听方法
       			onApplicationEvent(User user) ×
       ```

       

       &emsp;&emsp;默认情况下，注解形式的监听器的方法参数可以直接使用该对象类型也能实现监听，而接口方式的监听器由于监听的参数必须是事件类型，所以无法得到支持。但是，由上文可知，泛型事件其实是封装为 *PayloadApplicationEvent* 事件，而泛型事件对象被赋值给 payload 属性，所以接口形式 `onApplicationEvent` 可以通过监听 *PayloadApplicationEvent* 事件，然后通过 `payload` 的到非时间类型的对象。（将上面注释的监听器实现方法替换已有的代码，就能实现非时间类型对象的监听）

       

       > [GenericEventListenerBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/context/event/GenericEventListenerBootstrap.java)

       

     - *@EventListener* 实现监听的原理

       &emsp;&emsp;*@EventListener* 注解是在 Spring Framework 4.2 版本引入。像 [Spring @Enable 模块驱动](https://reionchan.github.io/2022/06/19/thinking-in-springboot-part-2/#spring-enable-%E6%A8%A1%E5%9D%97%E9%A9%B1%E5%8A%A8) 章节关于 **@Import 注解处理器的注册** 类似，Spring 引入新注解，势必要注册处理该注解的解析器。*@Import* 是由 `AnnotationConfigUtils.registerAnnotationConfigProcessors` 方法在容器中注册它的处理器 *ConfigurationClassPostProcessor*，依照此惯例，该方法同样包含注册处理 *@EventListener* 注解的处理器：

       ```java
       public class AnnotationConfigUtils {
           // @EventListener 注解的处理器 Bean 名称
           public static final String EVENT_LISTENER_PROCESSOR_BEAN_NAME =
                   "org.springframework.context.event.internalEventListenerProcessor";
           // EventListenerFactory Bean 名称
           public static final String EVENT_LISTENER_FACTORY_BEAN_NAME =
                   "org.springframework.context.event.internalEventListenerFactory";
       
           public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
                   BeanDefinitionRegistry registry, @Nullable Object source) {
               ...
               // 向容器注册 @EventListener 注解的处理器 EventListenerMethodProcessor
               if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
                   RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
                   def.setSource(source);
                   beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
               }
               // 向容器注册生产事件监听器的工厂类 DefaultEventListenerFactory
               if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
                   RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
                   def.setSource(source);
                   beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
               }
               ...
           }
       }
       ```

       &emsp;&emsp;推测 ***EventListenerMethodProcessor*** 是负责事件监听器注解的解析处理等操作，而 ***DefaultEventListenerFactory*** 应该是将注解获取的信息转化产生具体事件监听器，先来分析 ***EventListenerMethodProcessor***，它实现 ***SmartInitializingSingleton*** 接口，操作逻辑都在此接口的 `afterSingletonsInstantiated()` 方法执行。查看该接口描述可知，该方法的回调时机是在**所有非懒加载的单例 Bean** 实例化后。**它很适合本身就是非懒加载的单例 Bean 在容器中所有非懒加载单例 Bean 实例化之后做一些额外初始化的操作，而不会有误触发某些 Bean 过早初始化的风险**。

       

       &emsp;&emsp;下面从上下文生命周期的 `refresh()` 方法，来具体追踪 *SmartInitializingSingleton* 接口的回调时机：

       ```java
       public abstract class AbstractApplicationContext extends DefaultResourceLoader
               implements ConfigurableApplicationContext {
           @Override
           public void refresh() throws BeansException, IllegalStateException {
               ...
               try {
                   ...
                   // 1. 实例化所有非懒加载的单例 Bean
                   finishBeanFactoryInitialization(beanFactory);
                   ...
               }
               ...
           }
       
           protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
               ...
               // 2. 转调用 DefaultListableBeanFactory 中的预实例化单例 Bean 方法
               beanFactory.preInstantiateSingletons();
           }
       }
       ```

       &emsp;&emsp;步骤 2  `preInstantiateSingletons()` 方法由接口 *ConfigurableListableBeanFactory* 的子类 *DefaultListableBeanFactory* 中实现：

       ```java
       public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
               implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
       
           @Override
           public void preInstantiateSingletons() throws BeansException {
               // 容器中所有 Bean 名称
               List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
       
               // 3. 触发所有非懒加载的单例 Bean 实例化
               for (String beanName : beanNames) {
                   RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
                   // 此判断方法：非抽象、单例、非懒加载
                   if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
                       if (isFactoryBean(beanName)) {
                           Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                           if (bean instanceof FactoryBean) {
                               final FactoryBean<?> factory = (FactoryBean<?>) bean;
                               boolean isEagerInit;
                               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                                   isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                                   ((SmartFactoryBean<?>) factory)::isEagerInit,
                                           getAccessControlContext());
                               }
                               else {
                                   isEagerInit = (factory instanceof SmartFactoryBean &&
                                           ((SmartFactoryBean<?>) factory).isEagerInit());
                               }
                               if (isEagerInit) {
                                   getBean(beanName);
                               }
                           }
                       }
                       else {
                           getBean(beanName);
                       }
                   }
               }
       
               // 4. 非懒加载单例 Bean 全部初始化后完成后 
               for (String beanName : beanNames) {
                   Object singletonInstance = getSingleton(beanName);
                   if (singletonInstance instanceof SmartInitializingSingleton) {
                       final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
       				...
       				else {
                           // 5. 回调接口 SmartInitializingSingleton 的 afterSingletonsInstantiated 方法
                           smartSingleton.afterSingletonsInstantiated();
                       }
                   }
               }
           }
       }
       ```

       &emsp;&emsp;以 *SmartInitializingSingleton* 的实现类 ***EventListenerMethodProcessor*** 为例，它自身是注册到容器中的非懒加载的单例 Bean，包括它在内的所有非懒加载的单例 Bean 都实例化之后，再在这些单例 Bean 中筛选出类型为 *SmartInitializingSingleton* 的实现类 *EventListenerMethodProcessor*，执行 `afterSingletonsInstantiated()` 方法。此时能够确保所有在该方法执行 Bean 实例获取操作不会带来该 Bean 提前初始化的风险，例如：`ListableBeanFactory.getBeansOfType` 方法就属于潜在引起 Bean 提前初始化风险的方法。

       

       &emsp;&emsp;下面就来分析 *EventListenerMethodProcessor* 的 `afterSingletonsInstantiated()` 方法具体处理监听器注解的逻辑：

       ```java
       // 监听方法处理器实现 SmartInitializingSingleton 生命周期回调接口
       public class EventListenerMethodProcessor implements SmartInitializingSingleton, ApplicationContextAware {
       
           // 由生命周期回调到此方法
           @Override
           public void afterSingletonsInstantiated() {
               // 获得事件监听工厂类
               List<EventListenerFactory> factories = getEventListenerFactories();
               ConfigurableApplicationContext context = getApplicationContext();
               String[] beanNames = context.getBeanNamesForType(Object.class);
               // 遍历上下文的所有已注册的 Bean
               for (String beanName : beanNames) {
                   if (!ScopedProxyUtils.isScopedTarget(beanName)) {
                       ...
                       if (type != null) {
                           ...
                           try {
                               // 1. 依次处理 Bean
                               processBean(factories, beanName, type);
                           }
                           ...
                       }
                   }
               }
           }
       
           // 从上下文拿到前面注册的事件监听器工厂类 DefaultEventListenerFactory 的实例
           protected List<EventListenerFactory> getEventListenerFactories() {
               Map<String, EventListenerFactory> beans = getApplicationContext().getBeansOfType(EventListenerFactory.class);
               List<EventListenerFactory> factories = new ArrayList<>(beans.values());
               AnnotationAwareOrderComparator.sort(factories);
               return factories;
           }
       
           protected void processBean(
                   final List<EventListenerFactory> factories, final String beanName, final Class<?> targetType) {
       
               if (!this.nonAnnotatedClasses.contains(targetType)) {
                   Map<Method, EventListener> annotatedMethods = null;
                   try {
                       // 2. 获得标注有 @EventListener 的方法，构建 方法<->注解 的映射集合
                       annotatedMethods = MethodIntrospector.selectMethods(targetType,
                               (MethodIntrospector.MetadataLookup<EventListener>) method ->
                                       AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));
                   }
                   ...
       			else {
                       ConfigurableApplicationContext context = getApplicationContext();
                       // 遍历这些标注方法，利用工厂类生成监听器对象
                       for (Method method : annotatedMethods.keySet()) {
                           for (EventListenerFactory factory : factories) {
                               if (factory.supportsMethod(method)) {
                                   // 3. 在指定的 class 中找出与 method 匹配的方法
                                   Method methodToUse = AopUtils.selectInvocableMethod(method, context.getType(beanName));
                                   // 4. 利用工厂生成最终的监听器实例
                                   ApplicationListener<?> applicationListener =
                                           factory.createApplicationListener(beanName, targetType, methodToUse);
                                   if (applicationListener instanceof ApplicationListenerMethodAdapter) {
                                       // 5. 初始化 ApplicationListenerMethodAdapter 类型的监听器的上下文
                                       ((ApplicationListenerMethodAdapter) applicationListener).init(context, this.evaluator);
                                   }
                                   // 6. 将监听器实例注册到上下文
                                   context.addApplicationListener(applicationListener);
                                   break;
                               }
                           }
                       }
                       ...
                   }
               }
           }
       }
       ```

       &emsp;&emsp;&emsp;上面六个步骤完成从已注册到上下的所有 Bean 类型中搜寻被 *@EventListener* 标注的方法，然后通过与 *EventListenerMethodProcessor* 一同注册到上下文的监听器工厂类 ***DefaultEventListenerFactory*** 创建出具体监听器，并注册到上下文中。在 ***@EventListener* 与 *ApplicationListener* 实现监听区别** 小节中提到被 *@EventListener* 注解的方法的**访问修饰**、**返回值类型**在**同步**与**异步**之间存在差异，现在来具体分析差异存在的根本原因。

       

       &emsp;&emsp;以前面提及的 [AnnotatedAsyncEventListenerBootstrap](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/context/event/AnnotatedAsyncEventListenerBootstrap.java) 代码为例，它涵盖不同访问修饰符及返回值类型的 5 个监听器方法声明。首先将 *@EnableAsync*、*@Async* 注释掉，恢复成同步形式的监听器定义：

       ```java
       public class AnnotatedAsyncEventListenerBootstrap {
           ...
           //@EnableAsync()
           public static class MyAsyncEventListener {
       
               //@Async
               // 1. 原始类型返回值的方法
               @EventListener(ContextRefreshedEvent.class)
               public boolean onPrimitiveTypeContextRefreshedEvent(ContextRefreshedEvent event) {
                   println(" MyAsyncEventListener on primitive boolean type: " + event.getClass().getSimpleName());
                   return true;
               }
       
               //@Async
               // 2. 装箱后的返回值类型的方法
               @EventListener(ContextRefreshedEvent.class)
               public Boolean onContextRefreshedEvent(ContextRefreshedEvent event) {
                   println(" MyAsyncEventListener on boxing Boolean type: " + event.getClass().getSimpleName());
                   return true;
               }
       
               //@Async
               // 3. protected 访问修饰符的方法
               @EventListener(ContextRefreshedEvent.class)
               protected void onProtectedContextRefreshedEvent(ContextRefreshedEvent event) {
                   println(" MyAsyncEventListener on protected Method: " + event.getClass().getSimpleName());
               }
       
               //@Async
               // 4. default 的方法
               @EventListener(ContextRefreshedEvent.class)
               void onDefaultContextRefreshedEvent(ContextRefreshedEvent event) {
                   println(" MyAsyncEventListener on default Method: " + event.getClass().getSimpleName());
               }
       
               //@Async
               // 5. private 访问修饰符的方法
               @EventListener(ContextRefreshedEvent.class)
               private void onPrivateContextRefreshedEvent(ContextRefreshedEvent event) {
                   println(" MyAsyncEventListener on private Method: " + event.getClass().getSimpleName());
               }
           }
           ...
       }
       ```

       &emsp;在小马哥原书 **P477**、**P483** 两处表格中指出同步监听器方法的访问修饰符需为 `public`，并且进一步在 **P490** 给出原因是由于 **`AnnotatedElementUtils.findMergedAnnotation`** 方法只筛选出 `public`的监听器方法。但是在 ***@EventListener* 实现 Spring 事件监听** 小节实际代码执行结果显示访问修饰符不存在限制，从 `public` 到 `private` 的修饰符都可以。下面着重看类 *EventListenerMethodProcessor* 中以下两处涉及方法搜索及过滤的代码，来分析具体原因：

       ```java
       protected void processBean(final List<EventListenerFactory> factories,
           final String beanName, final Class<?> targetType) {
           if(!this.nonAnnotatedClasses.contains(targetType)) {
               Map<Method, EventListener> annotatedMethods=null;
               try {
                   // 第一处：获得标注有 @EventListener 的方法，构建 方法<->注解 的映射集合
                   annotatedMethods=MethodIntrospector.selectMethods(targetType,
                   (MethodIntrospector.MetadataLookup<EventListener>)method->
                   // 小马哥指出的此方法只是将 @EventListener 注解转换成一个合成注解的方法，并无修饰符过滤功能
                   AnnotatedElementUtils.findMergedAnnotation(method,EventListener.class));
               }
               ...
           else {
               ConfigurableApplicationContext context=getApplicationContext();
               for(Method method:annotatedMethods.keySet()) {
                   for(EventListenerFactory factory:factories) {
                       if(factory.supportsMethod(method)) {
                           // 第二处：根据找到的 method 到 bean 的实际类型中匹配真正使用的类
                           Method methodToUse=AopUtils.selectInvocableMethod(method,context.getType(beanName));
                           ...
                       }
                   }
               }
               ...
           }
       }
       ```

       &emsp;&emsp;先看第一处 `MethodIntrospector.selectMethods(targetType, metadataLookup)` 方法，它的主要作用是收集当前类 `targetType` 中标注了 *@EventListener* 的方法，并将其构建成方法到该注解的映射集合：

       ```java
       public abstract class MethodIntrospector {
       
           public static <T> Map<Method, T> selectMethods(Class<?> targetType, final MetadataLookup<T> metadataLookup) {
               ...
               for (Class<?> currentHandlerType : handlerTypes) {
       			...
                   ReflectionUtils.doWithMethods(currentHandlerType, method -> {
                       // 1. 从目标类 targetClass 中找到具体与 method（可能是接口方法） 匹配的方法
                       Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
                       // 外部指定查找具备 @EventListener 注解的方法
                       T result = metadataLookup.inspect(specificMethod);
                       if (result != null) {
                           // 2. 从给定的方法（可能是桥方法）找被桥接的原始方法，如果 specificMethod 非桥方法，返回本身
                           Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
                           if (bridgedMethod == specificMethod || metadataLookup.inspect(bridgedMethod) == null) {
                               // 非桥方法都会被添加到最终的 Map
                               methodMap.put(specificMethod, result);
                           }
                       }
                   }, ReflectionUtils.USER_DECLARED_METHODS);
               }
       
               return methodMap;
           }
       }
       ```

       &emsp;&emsp;`ClassUtils.getMostSpecificMethod(method, targetClass)` 方法代码：

       ```java
       public abstract class ClassUtils {
           // 从 targetClass 中找到与 method 匹配的方法
           public static Method getMostSpecificMethod(Method method, @Nullable Class<?> targetClass) {
               // 如果 targetClass 就是方法声明所在的类时，直接返回原方法，故此 if 都跳过不执行
               if (targetClass != null && targetClass != method.getDeclaringClass() && isOverridable(method, targetClass)) {
                   try {
                       if (Modifier.isPublic(method.getModifiers())) {
                           try {
                               return targetClass.getMethod(method.getName(), method.getParameterTypes());
                           }
                           catch (NoSuchMethodException ex) {
                               return method;
                           }
                       }
                       else {
                           Method specificMethod =
                                   ReflectionUtils.findMethod(targetClass, method.getName(), method.getParameterTypes());
                           return (specificMethod != null ? specificMethod : method);
                       }
                   }
                   ...
               }
               return method;
           }
       }
       ```

       &emsp;&emsp;由于 *@EventListener* 标注的 5 个方法都在具体的类上声明，不走上面的 `if` 语句块，且这些方法不存在泛型擦除而形成的[***桥方法***](https://reionchan.github.io/2017/02/01/corejava-v1-note-part2/#%E6%B3%9B%E5%9E%8B%E4%BB%A3%E7%A0%81%E5%92%8C%E8%99%9A%E6%8B%9F%E6%9C%BA)，故没有方法在此方法被剔除。接着看第二处方法 `AopUtils.selectInvocableMethod(method, targetType)` ：

       ```java
       public abstract class AopUtils {
           // 从给定的上下文 Bean 的真实类型 targetType （可能是 AOP Proxy 类型）中，找出与 method 匹配的方法
           public static Method selectInvocableMethod(Method method, @Nullable Class<?> targetType) {
               ...
               // 从 targetType 找出与 method 匹配的可调用方法，此步骤可能获得的是 APO Proxy 的代理方法，要么是 method 自身
               Method methodToUse = MethodIntrospector.selectInvocableMethod(method, targetType);
               // 重点：方法是 private 非静态 且是 SpringProxy 代理对象（JDK 动态代理 或 CGLIB代理），抛出异常
               if (Modifier.isPrivate(methodToUse.getModifiers()) && !Modifier.isStatic(methodToUse.getModifiers()) &&
                       SpringProxy.class.isAssignableFrom(targetType)) {
                   throw new IllegalStateException(String.format(
                           "Need to invoke method '%s' found on proxy for target class '%s' but cannot " +
                                   "be delegated to target bean. Switch its visibility to package or protected.",
                           method.getName(), method.getDeclaringClass().getSimpleName()));
               }
               return methodToUse;
           }
       }
       ```

       &emsp;&emsp;由于 *@EventListener* 标注的 5 个方法所在的类 *MyAsyncEventListener* 未被 Spring AOP 动态代理，故 `MethodIntrospector.selectInvocableMethod(method, targetType)` 返回的即是 `method` 自身，接着这 5 个方法中都虽然有一个是 `private`、非 `static` 的方法，好在它所在的类**非 Spring AOP 动态代理类**，故也不会报错。所谓 **Spring AOP 动态代理** 即运用了Spring 的动态代理技术对原始类进行了增强处理以实现面向切面编程逻辑，Spring 动态代理可能采用 JDK、CGLIB 方式, 不管何总方式其最后生成的代理类都会实现 ***SpringProxy*** 接口。特别注意， *@EventListener* 方法如果定义在 *@Configuration* 注解类中，**该方法不算属于动态代理方法**，因为配置类的增强代理没有实现 *SpringProxy* 接口，详细参考：**[Spring @Enable 模块驱动](https://reionchan.github.io/2022/06/19/thinking-in-springboot-part-2/#spring-enable-%E6%A8%A1%E5%9D%97%E9%A9%B1%E5%8A%A8)** 章节关于 ***@Configuration* 类 CGLIB 增强** 的描述。

       

       &emsp;&emsp;根据以上条件，**同步形式的注解监听器方法**除**访问修饰符为 `private`**、**非静态**、**方法所在类被 Spring AOP 动态代理** 这三个条件同时存在的情况外，都是被允许的，而返回值方面没有限制。

       

       &emsp;&emsp;要使监听器方法所在类被 Spring AOP 动态代理，可以有很多途径，通过在类上标注 *@EnableAsync* 开启异步支持，并且将某个方法标注 *@Async* 为一种方法，刚好接下来就要分析异步形式的注解监听器。首先来分析异步形式时对返回值类型的要求，先只将方法 1 加上异步注解：

       ```java
       @EventListener(ContextRefreshedEvent.class)
       @Async
       // 1. 原始类型返回值的方法
       // 运行期 AopInvocationException 异常： Null return value from advice does not match primitive return type
       public boolean onPrimitiveTypeContextRefreshedEvent(ContextRefreshedEvent event) {
           println(" MyAsyncEventListener on primitive boolean type: "+event.getClass().getSimpleName());
           return true;
       }
       ```

       &emsp;&emsp;执行后发现方法能正常运行，但是会有运行期异常 *AopInvocationException*, 定位代码行：

       ```java
       class CglibAopProxy implements AopProxy, Serializable {
           ...
           // 动态代理方法拦截器内部实现类
       	private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {
       		@Override
       		@Nullable
       		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
       			TargetSource targetSource = this.advised.getTargetSource();
       			try {
                       ...
       				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
       					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
       					retVal = methodProxy.invoke(target, argsToUse);
       				}
       				else {
       					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
       				}
                       // 代理方法调用完后，调用外部类的 processReturnType 处理返回值类型
       				retVal = processReturnType(proxy, target, method, retVal);
       				return retVal;
       			}
                   ...
       		}
               ...
       	}
           // 外部类方法
       	private static Object processReturnType(
       			Object proxy, @Nullable Object target, Method method, @Nullable Object returnValue) {
       		...
              	// 获取被代理的方法返回值类型
       		Class<?> returnType = method.getReturnType();
               // 如果返回值不为空，方法定义的返回值不为 void 时，返回值被要求不是基本类型
       		if (returnValue == null && returnType != Void.TYPE && returnType.isPrimitive()) {
       			throw new AopInvocationException(
       					"Null return value from advice does not match primitive return type for: " + method);
       		}
       		return returnValue;
       	}
           ...
       }
       ```

       &emsp;&emsp;从代码可知，在异步执行时 Spring AOP 的 CGLIB 实现中的方法拦截器在调用完代理方法后，如果方法返回值不为空时，要求**方法定义的返回值类型不能为基本类型**。将方法 1 注释，放开其余方法且在方法上恢复异步注解运行时，发现方法 5 编译器错误警告：***@Async* 要求方法必须能被覆盖**。这说明方法的异步执行依托 Spring AOP 动态代理机制，被代理的方法被要求可覆盖，显然 `private` 修饰符限制了这一点，如果忽略编译警告直接运行，此时会由于和同步方法类似，无法满足 `AopUtils.selectInvocableMethod(method, targetType)` 对方法的要求而异常。即便注释 *@Async* 恢复同步执行，也还是会由于其他方法存在 *@Async* 使得所在类为动态代理类型没能符合此方法的要求，此时把 `private` 修饰符改为其他修饰符即可，或者保持 `private` 不变，将方法增加 `static` 修饰即可正常运行，但可以发现静态方法由于无法满足动态代理对方法可覆盖的要求而不会被代理执行而退为同步执行。

       

       &emsp;&emsp;根据以上分析，**异步形式的注解监听器方法**访问修饰符**不能为 `private`**，或方法**不能为静态**，或者**返回值类型不能为基本类型**。

       

       &emsp;&emsp;此外，观察发现方法 2、3、4 对应的修饰符均可异步执行，异步执行器为 *SimpleAsyncTaskExecutor*，接下来分析异步处理的过程及原理。根据之前分析可知，通过与 *EventListenerMethodProcessor* 一同注册到上下文的监听器工厂类 ***DefaultEventListenerFactory*** 创建出具体监听器，随后注册到上下文中。首先就来分析此工厂创建监听器实例的具体过程：

       ```java
       public class DefaultEventListenerFactory implements EventListenerFactory, Ordered {
           ...
           public boolean supportsMethod(Method method) {
               return true;
           }
       
           // 创建监听器实例
           @Override
           public ApplicationListener<?> createApplicationListener(String beanName, Class<?> type, Method method) {
               // 创建的是一个监听器适配器实例 ApplicationListenerMethodAdapter
               return new ApplicationListenerMethodAdapter(beanName, type, method);
           }
       }
       ```
       
       &emsp;&emsp;*DefaultEventListenerFactory* 工厂创建的是 *ApplicationListenerMethodAdapter* 类型的监听器对象：
       
       ```java
       public class ApplicationListenerMethodAdapter implements GenericApplicationListener {
       
           // 构造方法
           public ApplicationListenerMethodAdapter(String beanName, Class<?> targetClass, Method method) {
               this.beanName = beanName;
               // 桥接方法的原始方法，此处为代理方法本身
               this.method = BridgeMethodResolver.findBridgedMethod(method);
               // 代理方法
               this.targetMethod = (!Proxy.isProxyClass(targetClass) ?
                       AopUtils.getMostSpecificMethod(method, targetClass) : this.method);
               this.methodKey = new AnnotatedElementKey(this.targetMethod, targetClass);
       
               // 处理方法上的注解信息，包含拦截条件、监听器排序
               EventListener ann = AnnotatedElementUtils.findMergedAnnotation(this.targetMethod, EventListener.class);
               this.declaredEventTypes = resolveDeclaredEventTypes(method, ann);
               this.condition = (ann != null ? ann.condition() : null);
               // 获得监听器排序值，从 @Order 获取，未找到默认 0
               this.order = resolveOrder(method);
           }
       
           // 事件处理方法，ApplicationListener 接口的实现
           @Override
           public void onApplicationEvent(ApplicationEvent event) {
               // 1. 具体处理方法
               processEvent(event);
           }
       
           public void processEvent(ApplicationEvent event) {
               Object[] args = resolveArguments(event);
               // 分析 @EventListener 中 condition 条件，绝对是否处理事件
               if (shouldHandle(event, args)) {
                   // 2. 反射执行方法
                   Object result = doInvoke(args);
                   if (result != null) {
                       // 返回值不为空时，会委托上下文继续发布返回值的事件
                       handleResult(result);
                   }
                   // 返回值为空时，什么都不做
                   ...
               }
           }
           
           @Nullable
           protected Object doInvoke(Object... args) {
               Object bean = getTargetBean();
               ReflectionUtils.makeAccessible(this.method);
               try {
                   // 3. 方法的反射调用，此处执行代理对象的方法
                   return this.method.invoke(bean, args);
               }
               ...
           }
       }
       ```
       
       &emsp;&emsp;在此监听器构造方法中，已经将监听注解的信息、排序值等设置完毕（此处解释 ***@EventListener* 监听排序** 小节有关监听器排序如何实现的问题），此监听器添加到上下文监听器列表时即可完成排序，而在收到广播器事件通知时分析条件决定是否处理该事件，如果符合处理条件随即利用反射调用代理类的方法处理事件。而反射调用的方法由于被 *@Async* 修饰，故其所在的类会被 Spring AOP 进行 CGLIB 代理处理，此时的 `bean` 对象是代理类型的实例。由此可以得知，实现监听器的异步执行是交由此代理类来处理，此异步代理机制由 *@EnableAsync* 驱动模块开启，并由 *@Async* 落实到具体要异步执行的方法上。根据 [Spring @Enable 模块驱动](https://reionchan.github.io/2022/06/19/thinking-in-springboot-part-2/#spring-enable-%E6%A8%A1%E5%9D%97%E9%A9%B1%E5%8A%A8) 关于 **@Enable 模块驱动原理** 小节的讲解，跟踪 *@Import* 的导入类：
       
       ```java
       public class AsyncConfigurationSelector extends AdviceModeImportSelector<EnableAsync> {
       
           private static final String ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME =
                   "org.springframework.scheduling.aspectj.AspectJAsyncConfiguration";
       
           @Override
           @Nullable
           public String[] selectImports(AdviceMode adviceMode) {
               switch (adviceMode) {
                   // 基于代理模式实现异步执行，此为默认模式
                   case PROXY:
                       return new String[] { ProxyAsyncConfiguration.class.getName() };
                   // 基于 AspectJ 技术实现异步执行，需引入 spring-aspects 依赖
                   case ASPECTJ:
                       return new String[] { ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME };
                   default:
                       return null;
               }
           }
       }
       ```
       
       &emsp;&emsp;默认采用代理模式实现，故此时配置类 *ProxyAsyncConfiguration* 被激活：
       
       ```java
       @Configuration
       @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
       public class ProxyAsyncConfiguration extends AbstractAsyncConfiguration {
           // Bean 名称 org.springframework.context.annotation.internalAsyncAnnotationProcessor
           @Bean(name = TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)
           @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
           public AsyncAnnotationBeanPostProcessor asyncAdvisor() {
               // 引入 Bean 后置处理器 AsyncAnnotationBeanPostProcessor
               AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
               Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");
               // 设定自定义异步执行注解类型，默认值为 annotation
               if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {
                   bpp.setAsyncAnnotationType(customAsyncAnnotation);
               }
               // 执行器不为空，设定异步执行器
               // 父类 AbstractAsyncConfiguration 关联的 AsyncConfigurer Bean 没有配置，此为空
               if (this.executor != null) {
                   bpp.setExecutor(this.executor);
               }
               // 异常处理器不为空，设定异常处理器
               // 父类 AbstractAsyncConfiguration 关联的 AsyncConfigurer Bean 没有配置，此为空
               if (this.exceptionHandler != null) {
                   bpp.setExceptionHandler(this.exceptionHandler);
               }
               // 设定使用基于类的代理模式（CGLIB），默认 false 即使用 Java 动态代理
               bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
               bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));
               return bpp;
           }
       }
       ```
       
       &emsp;&emsp;可以看到代理模式的异步执行配置类向容器注册类型为 *AsyncAnnotationBeanPostProcessor* 的 Bean 后置处理器：
       
       ```java
       public class AsyncAnnotationBeanPostProcessor extends AbstractBeanFactoryAwareAdvisingPostProcessor {
       
           // 父类实现了 BeanFactoryAware 接口，故包含此接口的回调注入 beanFactory 方法
           @Override
           public void setBeanFactory(BeanFactory beanFactory) {
               super.setBeanFactory(beanFactory);
               // 初始化父类 advisor 属性为 AsyncAnnotationAdvisor 实例
               AsyncAnnotationAdvisor advisor = new AsyncAnnotationAdvisor(this.executor, this.exceptionHandler);
               if (this.asyncAnnotationType != null) {
                   advisor.setAsyncAnnotationType(this.asyncAnnotationType);
               }
               advisor.setBeanFactory(beanFactory);
               this.advisor = advisor;
           }
       }
       ```
       
       &emsp;&emsp;此后置处理器通过 *BeanFactoryAware* 接口回调，初始化 `advisor` 属性为 ***AsyncAnnotationAdvisor*** 的实例。该类包含 ***Advice***、***Pointcut*** 两个重要的成员变量，分别定义通知（例如：各种拦截器）、切点（例如：拦截器的目标类及方法的验证器）。该后置处理器的父类 ***AbstractAdvisingBeanPostProcessor*** 实现 *BeanPostProcessor* 接口定义的 `postProcessAfterInitialization(bean, beanName)` 方法，该方法对 *@Async* 注解所在类的 Bean 对象进行代理，此处就不再展开而着重关注 *AsyncAnnotationAdvisor* 实例化：
       
       ```java
       public class AsyncAnnotationAdvisor extends AbstractPointcutAdvisor implements BeanFactoryAware {
           ...
           // 通知
           private Advice advice;
           // 切点
           private Pointcut pointcut;
       
           // 构造方法
           public AsyncAnnotationAdvisor(@Nullable Executor executor, @Nullable AsyncUncaughtExceptionHandler exceptionHandler) {
               Set<Class<? extends Annotation>> asyncAnnotationTypes = new LinkedHashSet<>(2);
               // 将 @Async 注解类型放入异步注解类型列表
               asyncAnnotationTypes.add(Async.class);
               // 1. 构建通知
               this.advice = buildAdvice(executor, this.exceptionHandler);
               // 2. 构建切点
               this.pointcut = buildPointcut(asyncAnnotationTypes);
           }
       
           // 1.1 创建 AnnotationAsyncExecutionInterceptor 类型的通知，注解执行拦截器
           protected Advice buildAdvice(@Nullable Executor executor, AsyncUncaughtExceptionHandler exceptionHandler) {
               return new AnnotationAsyncExecutionInterceptor(executor, exceptionHandler);
           }
       
           // 2.1 构建拦截器的目标切点，类及方法上是否包含 @Async 的组合匹配器
           protected Pointcut buildPointcut(Set<Class<? extends Annotation>> asyncAnnotationTypes) {
               ComposablePointcut result = null;
               for (Class<? extends Annotation> asyncAnnotationType : asyncAnnotationTypes) {
                   Pointcut cpc = new AnnotationMatchingPointcut(asyncAnnotationType, true);
                   Pointcut mpc = new AnnotationMatchingPointcut(null, asyncAnnotationType, true);
                   if (result == null) {
                       result = new ComposablePointcut(cpc);
                   }
                   else {
                       result.union(cpc);
                   }
                   result = result.union(mpc);
               }
               return (result != null ? result : Pointcut.TRUE);
           }
       }
       ```
       
        &emsp;&emsp;切点的构造很好理解，它最终形成的组合切点验证器用来校验类、方法上是否包含 *@Async* 注解，这个组合切点验证器最终影响 *CglibAopProxy* 类的 `getProxy(classLoader)` 方法生成的代理类中哪些方法需要被代理执行。而构造的 *AnnotationAsyncExecutionInterceptor* 是实现异步的关键，它继承 ***AsyncExecutionInterceptor*** 类。当代理方法被执行时，会调用 ***org.springframework.cglib.proxy.MethodInterceptor*** 接口的 `intercept` 方法，而此方法会将代理方法封装成 ***MethodInvocation*** 对象，调用此对象 `proceed()` 方法。而 `proceed()` 方法的执行最终被 ***AsyncExecutionInterceptor*** 拦截，将 ***MethodInvocation*** 对象充当其实现了 ***org.aopalliance.intercept.MethodInterceptor*** 接口的 **`invoke`** 方法的参数：
       
       ```java
       public class AsyncExecutionInterceptor extends AsyncExecutionAspectSupport implements MethodInterceptor, Ordered {
           ...
           // MethodInterceptor 接口实现方法，代理类中的被代理方法的执行封装成 MethodInvocation 对象
           @Override
           @Nullable
           public Object invoke(final MethodInvocation invocation) throws Throwable {
               Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
               Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
               final Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
               // 获取异步执行器, 委托父类 AsyncExecutionAspectSupport 获取
               AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
               ...
               // 构造异步执行任务
               Callable<Object> task = () -> {
                   try {
                       // 调用代理类中被代理方法，让其执行
                       Object result = invocation.proceed();
                       if (result instanceof Future) {
                           return ((Future<?>) result).get();
                       }
                   }
                   ...
                   return null;
               };
       
               // 委托父类 AsyncExecutionAspectSupport 将任务提交给异步执行器执行
               return doSubmit(task, executor, invocation.getMethod().getReturnType());
           }
           
           // 覆盖父类 AsyncExecutionAspectSupport 的方法
           protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
               Executor defaultExecutor = super.getDefaultExecutor(beanFactory);
               return (defaultExecutor != null ? defaultExecutor : new SimpleAsyncTaskExecutor());
       	}
           ...
       }
       ```
       
       &emsp;&emsp;该拦截器将代理类的执行过程封装成异步任务 `task`，提交给异步执行器进行异步处理，而异步执行器的设置及任务提交执行都在父类 *AsyncExecutionAspectSupport* 中实现：
       
       ```java
       public abstract class AsyncExecutionAspectSupport implements BeanFactoryAware {
           ...
           // 1. 根据拦截的方法匹配相应的执行器
           @Nullable
           protected AsyncTaskExecutor determineAsyncExecutor(Method method) {
               AsyncTaskExecutor executor = this.executors.get(method);
               if (executor == null) {
                   Executor targetExecutor;
                   // 子类 AnnotationAsyncExecutionInterceptor 实现方法，检查 @Async 注解的 value 值
                   String qualifier = getExecutorQualifier(method);
                   if (StringUtils.hasLength(qualifier)) {
                       // @Async 注解的 value 有值，从 beanFactory 寻找 @Qualifier 或 beanName 为指定的异步执行器
                       targetExecutor = findQualifiedExecutor(this.beanFactory, qualifier);
                   }
                   else {
                       targetExecutor = this.defaultExecutor;
                       if (targetExecutor == null) {
                           synchronized (this.executors) {
                               if (this.defaultExecutor == null) {
                                   // 当前默认执行器为空，执行子类 AsyncExecutionInterceptor 的覆盖方法
                                   // 翻查上一个代码片段子类方法，子类执行本类的 getDefaultExecutor 获得执行器
                                   // 当容器中没有时，子类会生成一个 SimpleAsyncTaskExecutor 类型的执行器
                                   this.defaultExecutor = getDefaultExecutor(this.beanFactory);
                               }
                               targetExecutor = this.defaultExecutor;
                           }
                       }
                   }
                   if (targetExecutor == null) {
                       return null;
                   }
                   executor = (targetExecutor instanceof AsyncListenableTaskExecutor ?
                           (AsyncListenableTaskExecutor) targetExecutor : new TaskExecutorAdapter(targetExecutor));
                   this.executors.put(method, executor);
               }
               return executor;
           }
       
           @Nullable
           protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
               if (beanFactory != null) {
                   try {
                       // 先从 beanFactory 中按类型匹配
                       return beanFactory.getBean(TaskExecutor.class);
                   }
                   catch (NoUniqueBeanDefinitionException ex) {
                       try {
                           // 类型未找到再按 bean 名称 taskExecutor 搜寻
                           return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
                       }
                       ...
                   }
                   ...
               }
               return null;
           }
       
           // 2. 任务提交执行器执行
           @Nullable
           protected Object doSubmit(Callable<Object> task, AsyncTaskExecutor executor, Class<?> returnType) {
               // 根据返回定义的返回类型不同进行不同的提交执行
               if (CompletableFuture.class.isAssignableFrom(returnType)) {
                   return CompletableFuture.supplyAsync(() -> {
                       try {
                           return task.call();
                       }
                       catch (Throwable ex) {
                           throw new CompletionException(ex);
                       }
                   }, executor);
               }
               else if (ListenableFuture.class.isAssignableFrom(returnType)) {
                   return ((AsyncListenableTaskExecutor) executor).submitListenable(task);
               }
               else if (Future.class.isAssignableFrom(returnType)) {
                   return executor.submit(task);
               }
               else {
                   // 未匹配到的返回值定义，采用默认提交并返回 null 执行结果
                   executor.submit(task);
                   return null;
               }
           }
       }
       ```
       
       &emsp;&emsp; *@Async* 异步处理的整个过程以及异步监听器方法为何默认执行器是 *SimpleAsyncTaskExecutor* 就全部阐述完毕。到此整个 *@EventListener* 注解实现监听的实现原理也已解释完毕，而下一小节将对 **Spring 事件/监听 机制** 做一个全面性总结。
       
       

  5. 总结

     1. Spring 事件

        &emsp;&emsp;Spring 事件 API 由 ApplicationEvent 表述，继承 `java.util.EventObject`。Spring Framework 内建五种事件，除 *RequestHandledEvent* 外，其余四种均继承抽象类 *ApplicationContextEvent*, 属于 Spring 应用上下文事件，事件源都为 *ApplicationContext*。

        &emsp;&emsp;Spring Framework 允许自定义 *ApplicationEvent* 事件类型，并且能够实现带泛型参数的自定义事件，但此时需要实现 *ResolvableTypeProvider* 接口暴露事件的泛型参数具体类型，使得监听器的匹配进一步精确到泛型参数类型。

        

     2. Spring 事件监听手段

        &emsp;&emsp;Spring 事件监听可以通过实现 *ApplicationListener* 接口、*@EventListener* 注解两种途径。两种途径均能监听一种或多种实践，且都支持泛型事件。实现层面来看，*@EventListener* 注解形式是通过 Bean 方法与 *ApplicationListener* 接口的适配，即 *ApplicationLinstenerMethodAdapter*。注解形式必须依赖 Spring 应用上下文进行注册，而接口形式既可以与 Spring 应用上下文关联，但也支持仅关联 *ApplicationEventMulticaster* 实现监听功能，Spring Boot 事件即是这种模式。

        

     3. Spring 事件广播器

        &emsp;&emsp;Spring 事件广播在 Spring Framework 中有两类 API 表达方式，一种是 *ApplicationEventPublisher* 的 `publishEvent` 方法，另一种是 *ApplicationEventMulticaster* 的 `multicastEvent`。前一种由 *AbstractApplicationContext* 具体实现，后一种由 *SimpleApplicationEventMulticaster* 具体实现。然而 *AbstractApplicationContext* 最终还是委托给 *SimpleApplicationEventMulticaster*，故后者是整个 Spring 事件/监听 机制的纽带。在 Spring Boot 中也是通过它进行 Spring Boot 事件的广播，那么它在 Spring Framework 及 Spring Boot 中存在何种联系呢？留到下一章节继续讨论。

  

* 理解 Spring Boot 事件/监听 机制

  

  &emsp;&emsp;前面提到 Spring Framework、Spring Boot 的监听事件最终都落到 *SimpleApplicationEventMulticaster* 类型的广播器进行广播，它们关联的是否是同一个对象呢？**Spring Boot 1.4 之前，两者关联的是同一个对象，而之后进行了隔离**。下文将以 Spring Boot 1.4 为分水岭，之前版本称作早期的 Spring Boot 事件/监听 机制，之后版本称作当前 Spring Boot 事件/监听 机制。对比两者差别，来分析出于什么动机使得 Spring Boot 在之后的版本做广播器进行事件隔离。

  

  1. 早期 Spring Boot 事件/监听 机制

     - Spring 生命周期方法 `refresh()` 关联广播器 Bean 对象

       &emsp;&emsp;从之前的 Spring 事件/监听 机制可知，Spring Framework 是在 `refresh()` 生命周期方法中调用 `initApplicationEventMulticaster()` 进行事件广播器的初始化，具体逻辑是检查当前上下文关联的 BeanFactory 中是否包含名称为 **`applicationEventMulticaster`** 的 Bean 对象，存在时将其关联到上下文中充当 *ApplicationEventPublisher* 事件发布时的广播器，否则创建一个类型为 *SimpleApplicationEventMulticaster* 的广播器，并将其设置为该名称的 Bean 对象注册到 BeanFactory 当中。

     

     - Spring Boot 运行监听器 `contextPrepared()` 事件方法注册广播器 Bean 对象

       &emsp;&emsp;在 [SpringApplication 准备阶段](#SpringApplication 准备阶段) 中可知 Spring Boot 运行期间的各种事件交由唯一的运行监听器实现类 *EventPublishingRunListener* 进行事件发布，它将事件广播委托给了关联的 *ApplicationEventMulticaster* 广播器，而改广播器是在运行监听器的构造方法中被赋值为 *SimpleApplicationEventMulticaster* 实现类对象，而该对象在随后的 `contextPrepared()` 上下文就绪的监听方法中被注册到上下文的 BeanFactory 中，且 Bean 名称刚好为 **`applicationEventMulticaster`**：

       ```java
       public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
       
           public EventPublishingRunListener(SpringApplication application, String[] args) {
               this.application = application;
               this.args = args;
               // 事件发布监听器初始化广播器实例
               this.multicaster = new SimpleApplicationEventMulticaster();
               for (ApplicationListener<?> listener : application.getListeners()) {
                   this.multicaster.addApplicationListener(listener);
               }
           }
       
           // Spring Boot 上下文就绪事件监听方法
           @Override
           public void contextPrepared(ConfigurableApplicationContext context) {
               // 注册广播器到 BeanFactory
               registerApplicationEventMulticaster(context);
           }
       
           private void registerApplicationEventMulticaster(
                   ConfigurableApplicationContext context) {
               // 向上下文关联的 BeanFactory 注册当前的广播器实例 SimpleApplicationEventMulticaster
               context.getBeanFactory().registerSingleton(
                       AbstractApplicationContext.APPLICATION_EVENT_MULTICASTER_BEAN_NAME,
                       this.multicaster);
               if (this.multicaster instanceof BeanFactoryAware) {
                   ((BeanFactoryAware) this.multicaster)
                           .setBeanFactory(context.getBeanFactory());
               }
           }
       }
       ```

       &emsp;&emsp;结合 Spring Boot 声明周期可知，Spring Boot 上下文就绪事件发生在上下文 `refresh()` 方法之前，所以当后者执行时上下文关联的 BeanFactory 中已包含由 Spring Boot 上下文就绪事件注册的广播器 Bean，由此说明 Spring Boot 1.4 之前，Spring 事件广播器与 Spring Boot 事件广播器为同一个 Bean 对象，其类型为 *SimpleApplicationEventMulticaster*。前文提到，其能关联多个 *ApplicationListener* 监听器，并可根据事件进行归类管理，既然 Spring Framework 与 Spring Boot 共用此广播器，势必 Spring 事件监听器与 Spring Boot 事件监听器将统一由其管理，这会不会出现什么问题呢？要回答此问题，先来看 Spring Boot 事件监听器及 Spring 事件监听器添加到此广播器的具体过程。

       

     - Spring 事件监听器添加至广播器

       &emsp;&emsp;Spring 事件监听器添加至广播器在前面讲解 **Spring 事件/监听 机制** 时概略提到，其在 `refresh()` 时调用 `registerListeners` 方法添加：

       ```java
       public abstract class AbstractApplicationContext extends DefaultResourceLoader
               implements ConfigurableApplicationContext {
       
           public void refresh() throws BeansException, IllegalStateException {
               synchronized (this.startupShutdownMonitor) {
                   ...
                   try {
                       ...
                       // 将广播器未创建之前发生的事件，交由已生成的广播器进行广播
                       registerListeners();
                       ...
                   }
                   ...
               }
           }
       
           protected void registerListeners() {
               // 1. 从当前上下文 this.applicationListeners 获取关联的监听器
               for (ApplicationListener<?> listener : getApplicationListeners()) {
                   // 注册到广播器中
                   getApplicationEventMulticaster().addApplicationListener(listener);
               }
               // 2. BeanFactory 定义的监听器 Bean，（类扫描获得、Bean 注册获得的监听器）
               String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
               for (String listenerBeanName : listenerBeanNames) {
                   // 注册到广播器中
                   getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
               }
               ...
           }
       
           // 从当前上下文 this.applicationListeners 获取关联的监听器
           public Collection<ApplicationListener<?>> getApplicationListeners() {
               // 返回 this.applicationListeners 属性
               return this.applicationListeners;
           }
       }
       ```

       &emsp;&emsp;特别注意事件监听器的主要来源有两处：

       

       1. **当前应用上下文中获取监听器集合**

          &emsp;&emsp;通过调用上下文对象的 `addApplicationListener(ApplicationListener<?>)` 所收集到的事件监听器 ***Set*** 集合。

       2. **当前应用上下文关联的 BeanFactory 中类型为 *ApplicationListener* 的 Bean**

          &emsp;&emsp;通过配置类扫描、Bean 定义注册、@EventListener 等途径生成的监听器类型的 Bean

       

     - Spring Boot 事件监听器添加至广播器

       &emsp;&emsp;Spring Boot 刚启动时会先构造 *EventPublishingRunListener* 事件发布器，进而初始化其事件广播器成员变量，而 Spring Boot 事件监听器也是在事件广播器初始化之后即刻被添加至该广播器中：

       ```java
       public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
           ...
           // 广播器成员变量
           private final ApplicationEventMulticaster multicaster;
           
           // 事件发布器构造方法
           public EventPublishingRunListener(SpringApplication application, String[] args) {
               this.application = application;
               this.args = args;
               // 初始化广播器为 SimpleApplicationEventMulticaster
               this.multicaster = new SimpleApplicationEventMulticaster();
               // 随即将 SpringApplication 关联的 Spring Boot 事件监听器添加到该广播器
               for (ApplicationListener<?> listener : application.getListeners()) {
                   this.multicaster.addApplicationListener(listener);
               }
           }
           ...
       }
       ```

       &emsp;&emsp;而 *SpringApplication* 关联的事件监听器来源有两处：

       

       1. 构造 *SpringApplication* 时采用 **Spring 工厂加载机制** 获得的监听器集合

          ```java
          public class SpringApplication {
              ...
              // 关联的监听器列表
              private List<ApplicationListener<?>> listeners;
          
              // 构造方法
              public SpringApplication(Object... sources) {
                  // 1. 调用初始化方法
                  initialize(sources);
              }
          
              // 执行初始化
              private void initialize(Object[] sources) {
                  ...
                  // 2. Spring 工厂加载机制获得配置的监听器列表，调用设置方法添加到关联的监听器列表
                  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
                  ...
              }
          
              private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
                  return getSpringFactoriesInstances(type, new Class<?>[] {});
              }
          
              // 3. 工厂加载 ApplicationListener 类型的监听器实例列表
              private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
                                                                              Class<?>[] parameterTypes, Object... args) {
                  ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
                  // 从类路径中的 MET-INF/spring.factories 文件中提取监听器类型
                  Set<String> names = new LinkedHashSet<String>(
                          SpringFactoriesLoader.loadFactoryNames(type, classLoader));
                  List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
                          classLoader, args, names);
                  AnnotationAwareOrderComparator.sort(instances);
                  return instances;
              }
          
              // 4. 添加到当前关联的监听器列表当中
              public void setListeners(Collection<? extends ApplicationListener<?>> listeners) {
                  this.listeners = new ArrayList<ApplicationListener<?>>();
                  this.listeners.addAll(listeners);
              }
          }
          ```

          

       2. 调用 *SpringApplication* 的 `addListeners(ApplicationListener<?>...)` 添加的监听器集合

          &emsp;&emsp;此调用同样也是把事件监听器添加到 *SpringApplication* 关联的监听器列表中 `List<ApplicationListener<?>> listeners`

       

     &emsp;&emsp;到此，广播器如何关联收集 Spring Framework、Spring Boot 两者运行时的监听器的过程、它们当中的监听器来源都已描述完毕。由于 Spring Boot 应用是先初始化 *SpringApplication* 之后，再生成 Spring Framework 的上下文对象，并通过上下文的 `refresh()` 方法驱动 Spring Framework 的运行，那么根据此运行顺序先后进行 Spring Boot、Spring Framework 监听器的收集、最终汇集添加到广播器的过程做一个顺序图：

     ![早期 Spring Boot 广播器关联监听器顺序图](https://raw.githubusercontent.com/ReionChan/thinking-in-spring-boot-samples/master/spring-boot-1.x-samples/spring-boot-1.3.x-project/src/main/resources/spring-boot-1.3-listeners-added-into-multicaster.svg)

     &emsp;&emsp;图中 ①、②、③ 是在 Spring Boot 中完成监听器的搜集及添加到广播器的流程，而 ④、⑤ 是在 Spring Framework 中完成监听器的搜集及添加到广播器的流程。跟踪图中 *SimpleApplicationEventMulticaster* 广播器实例蓝色生命线，很容易看出其生命周期的过程：

     1. 首先，由 Spring Boot 步骤 `2.2 new` 创建广播器
     2. 其次，由 Spring Boot 步骤 `4.4 调用 registerSingleton 将广播器注册到 BeanFactory` 中成为 Bean 对象
     3. 然后，在 Spring Framework 步骤 `6.4 从 BeanFactory 中获得广播器并赋值给上下文的 this.applicationEventMulticaster`
     4. 最后，可知广播器由 Spring Boot 初始化并注册到 BeanFactory 中，借由 BeanFactory 共享给 Spring Framework

     

     &emsp;&emsp;步骤 ① 中 3.3 步骤是将 *SpringApplication* 收集的监听器放入广播器，步骤 ③ 中 5.4 将 *SpringApplication* 收集的监听器依次添加到 *ApplicationContext* 中，步骤 ⑤ 中 7.3 步骤是将 *ApplicationContext* 收集的监听器放入广播器，而步骤 ①、⑤ 关联的广播器为同一个实例，这就势必导致步骤 ① 中已放入到广播器的监听器借由 *ApplicationContext* 被步骤 ⑤ 重复放入广播器，但由于广播器中用来存放监听器的是 *Set* 类型的集合，相同实例多次添加是会覆盖的，故一般情况不会包含相同的监听器实例，除非像示例 [DuplicatedEventListenerBootstrap](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-1.x-samples/spring-boot-1.3.x-project/src/main/java/thinking/in/spring/boot/samples/spring/application/event/DuplicatedEventListenerBootstrap.java) 中的 [ContextRefreshedEventListener](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-1.x-samples/spring-boot-1.3.x-project/src/main/java/thinking/in/spring/boot/samples/spring/application/event/listener/ContextRefreshedEventListener.java) 监听器一样，覆盖 `hashCode` 方法使得每次调用返回不一样的 hash 值。这将导致相同实例由于每次 hash 值不同而被重复添加到 Set 集合，最终导致一个事件被两次执行。

     

     > [DuplicatedEventListenerBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-1.x-samples/spring-boot-1.3.x-project/src/main/java/thinking/in/spring/boot/samples/spring/application/event/DuplicatedEventListenerBootstrap.java) 

     

     &emsp;&emsp;步骤 ③ Spring Boot 将 *SpringApplication* 收集的监听器依次添加到 *ApplicationContext* 中具体逻辑是在 *EventPublishingRunListener* `contextLoaded` 方法中完成：

     ```java
     public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
     
         private final SimpleApplicationEventMulticaster initialMulticaster;
     	
         ...
         // 上下文已加载完毕执行
         @Override
         public void contextLoaded(ConfigurableApplicationContext context) {
             // 循环 SpringApplication 关联的监听器列表
             for (ApplicationListener<?> listener : this.application.getListeners()) {
                 if (listener instanceof ApplicationContextAware) {
                     ((ApplicationContextAware) listener).setApplicationContext(context);
                 }
                 // 依次将监听器添加到上下文中，进行上下文关联
                 context.addApplicationListener(listener);
             }
             publishEvent(new ApplicationPreparedEvent(this.application, this.args, context));
         }
         ...
     }
     ```

     &emsp;&emsp;因为 Spring Boot 及 Spring Framework 共享相同广播器实例，粗略看步骤 ③ 中 Spring Boot 的监听器已经被广播器收集，没必要再通过上下文传递再在 Spring Framework 中重复收集。其实不然，细心观察 *SpringApplication* 的 `addListeners(ApplicationListener<?>...)` 方法注释：

     > Add ApplicationListeners to be applied to the SpringApplication **and registered with the ApplicationContext**.

     &emsp;&emsp;留意后半句，这些监听器将被 *ApplicationContext* 注册，言外之意之所以要将监听器全部传递给上下文，就是实现该方法语义必须要这么做，况且之后的版本中二者不再共享相同的广播器实例，就更有必要将一些在 Spring Boot 初始化配置的 Spring 事件监听器传递到应用上下文中监听非 Spring Boot 事件。另一方在上下文已加载完毕后，在此处统一处理监听器实现 *ApplicationContextAware* 的回调操作再合适不过。

     &emsp;&emsp;共享相同广播器实例，除上面提及的可能带来监听器重复注册风险外，另外的 “副作用” 是有些监听器原本只想监听 Spring 事件本身，但会额外监听到**部分 Spring Boot 事件**，其原因为二者的事件都实现相同的事件接口 *ApplicationEvent*。之所以说是部分 Spring Boot 事件，是因为大部分 Spring Boot 事件都在 Spring 应用上下文初始化之前已被发布。能被监听到的 Spring Boot 事件是那些在上下文调用 `refresh()` 之后发生的事件，例如：*ApplicationReadyEvent*、*ApplicationFailedEvent*。另外与 Spring 上下文事件都相同的父类 *ApplicationContextEvent* 类似，Spring Boot 事件也有相同的父类 *SpringApplicationEvent*, 它将 *SpringApplication* 当做事件源。

     

     &emsp;&emsp;示例 [MultipleSpringBootEventsListenerBootstrap](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-1.x-samples/spring-boot-1.3.x-project/src/main/java/thinking/in/spring/boot/samples/spring/application/event/MultipleSpringBootEventsListenerBootstrap.java) 中的 [MultipleSpringBootEventsListener](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-1.x-samples/spring-boot-1.3.x-project/src/main/java/thinking/in/spring/boot/samples/spring/application/event/listener/MultipleSpringBootEventsListener.java) 监听器通过继承 *SmartApplicationListener* 支持多事件的监听：

     ```java
     public class MultipleSpringBootEventsListener implements SmartApplicationListener {
     
         @Override
         public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
             // 支持事件的类型
             return ApplicationReadyEvent.class.equals(eventType) ||
                     ApplicationFailedEvent.class.equals(eventType);
         }
     
         @Override
         public boolean supportsSourceType(Class<?> sourceType) {
             // SpringApplicationEvent 均已 SpringApplication 作为配置源
             return SpringApplication.class.equals(sourceType);
         }
     
         @Override
         public void onApplicationEvent(ApplicationEvent event) {
             if (event instanceof ApplicationReadyEvent) {
                 // 当事件为 ApplicationReadyEvent 时，随机的抛出异常
                 if (new Random().nextBoolean()) {
                     throw new RuntimeException("ApplicationReadyEvent 事件监听异常!");
                 }
             }
             System.out.println("MultipleSpringBootEventsListener 监听到事件 : " + event.getClass().getSimpleName());
         }
         ...
     }
     ```

     &emsp;&emsp;这种 “副作用” 其实并不是设计者的初衷，Spring Boot 事件不应该影响到本不想监听它们的监听器。所以 Spring Boot 在 1.4 版本做出调整，不在让 *SimpleApplicationEventMulticaster* 对象复用到应用上下文中，具体调整详情参考下一小节。

     

     > [MultipleSpringBootEventsListenerBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-1.x-samples/spring-boot-1.3.x-project/src/main/java/thinking/in/spring/boot/samples/spring/application/event/MultipleSpringBootEventsListenerBootstrap.java)

     

  2. 当前 Spring Boot 事件/监听 机制

     &emsp;&emsp;由上面可知，Spring Boot 与 Spring Framework 之间是通过将广播器注册为上下文的 Bean 实现共享，要打断共享只需取消广播器的注册。之后 Spring Framework 在初始化广播器时，由于 BeanFactory 找不到此类型的 Bean 进而触发创建操作，具体流程如下：

     ![当前 Spring Boot 广播器关联监听器顺序图](https://raw.githubusercontent.com/ReionChan/thinking-in-spring-boot-samples/master/spring-boot-2.0-samples/spring-application-sample/src/main/resources/uml/spring-boot-1.4-listeners-added-into-multicaster.svg)

     &emsp;&emsp;步骤 ③ 初始化广播器为新建的 *SimpleApplicationEventMulticaster* 实例（橙色），虽然广播器类型相同，但已经与 Spring Boot 的广播器（蓝色）属不同的实例，需要加以区别。另外由于步骤 ② 保持不变，因此在 *SpringApplication* 添加的监听器还是能被新广播器添加，对新广播器而言，这些监听器为初次添加，而非之前的重复添加。

     

  3. Spring Boot 内建事件监听器

     &emsp;&emsp;Spring Boot 应用场景中，Spring 事件监听器、Spring Boot 事件监听器都配置在 `META-INF/spring.factories` 资源中，并以 `org.springframework.context.ApplicationListener` 作为属性名。所有内建事件监听器的配置文件分别在 spring-boot 和 spring-boot-autoconfigure 中，现提取总结如下表：

     | ApplicationListener 实现                     | 监听事件                                                     | 场景说明                                                     | 引入版本             |
     | -------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------- |
     | *ClearCachesApplicationListener*             | *ContextRefreshedEvent*                                      | 清理 *ReflectionUtils* 和 *ClassLoader* 缓存                 | 1.4                  |
     | *ParentContextCloserApplicationListener*     | *ParentContextAvailableEvent*                                | 当 *SpringApplication* 关联上下文初始化时关联父上下文        | 1.0                  |
     | *FileEncodingApplicationListener*            | *ApplicationEnvironmentPreparedEvent*                        | 检查系统 file.encoding 是否与 `spring.mandatory-file-encoding` 属性一致 | 1.0                  |
     | *AnsiOutputApplicationListener*              | *ApplicationEnvironmentPreparedEvent*                        | 环境配置绑定 *AnsiOutput.Enabled* 并配置 *AnsiOutput*        | 1.2                  |
     | *ConfigFileApplicationListener*              | *ApplicationEnvironmentPreparedEvent* 和 *ApplicationPreparedEvent* | 加载应用配置文件，默认：application.properties 或 application.yml | 1.0                  |
     | *DelegatingApplicationListener*              | *ApplicationEnvironmentPreparedEvent*                        | 环境配置 `context.listener.classes` 设置有监听器时，此监听器作为事件转发广播器 | 1.0                  |
     | *ClasspathLoggingApplicationListener*        | *ApplicationEnvironmentPreparedEvent* 和 *ApplicationFailedEvent* | 当 DEBUG 级别日志开启时，在左边两个事件发生时记录当前线程上下文的 Class Path | 1.0 (2.0 包名被重构) |
     | *LoggingApplicationListener*                 | *ApplicationStartingEvent* 或 *ApplicationEnvironmentPreparedEvent* 或 *ApplicationPreparedEvent* 或 *ContextClosedEvent* 或 *ApplicationFailedEvent* | 识别日志框架，并加载日志配置文件                             | 1.0 (2.0 包名被重构) |
     | *LiquibaseServiceLocatorApplicationListener* | *ApplicationStartingEvent*                                   | 根据 Spring Boot 版本替换 liquibase 的 *ServiceLocator*      | 1.0                  |
     | *BackgroundPreinitializer*                   | *ApplicationStartingEvent* 或 *ApplicationReadyEvent* 或 *ApplicationFailedEvent* | 以后台异步线程方式触发早期耗时的初始化操作                   | 1.3                  |

     

  4. 总结 Spring Boot 事件/监听 机制

     - Spring Boot 事件
     
       &emsp;&emsp;Spring Boot 事件继承 Spring 事件类型 *ApplicationEvent*, 且也是 *SpringApplicationEvent* 的子类，事件源为 *SpringApplication*。
     
       
     
     - Spring Boot 事件监听手段
     
       &emsp;&emsp;Spring Boot 1.4 版本之前应用上下文关联的事件监听器是可以监听到 Spring Boot 事件，在 1.4 之后由于广播器各自独立的原因，Spring Boot 事件只能由 *SpringApplication* 关联的事件监听器监听，手段有两种：1. 配置 `META-INF/spring.factories` 方式；2. 利用 *SpringApplication* 的 API `addListeners(ApplicationListener...)` 或 *SpringApplicationBuilder* 的 API `listeners(ApplicationListener...)`。
     
       
     
     - Spring Boot 事件广播器
     
       &emsp;&emsp;广播器的实际类型为 *SimpleApplicationEventMulticaster*，在 Spring Boot 1.4 之后该广播器不再注册到 BeanFactory 当中，与后续上下文初始化启动后的广播器类型相同但属于不同实例对象，即此版本后 Spring 事件不再由 Spring Boot 初始化生成的广播器广播。
     
     

* 装配 ApplicationArguments

  &emsp;&emsp;*ApplicationArguments* 是对 *SpringApplication* 启动的参数解析后生成的封装类，查看 *SimpleCommandLineArgsParser* 解析处理源码，可以反向了解启动参数语法格式：

  ```java
  class SimpleCommandLineArgsParser {
      public CommandLineArgs parse(String... args) {
          CommandLineArgs commandLineArgs = new CommandLineArgs();
          for (String arg : args) {
              // 类型一：以 -- 开通
              if (arg.startsWith("--")) {
                  String optionText = arg.substring(2, arg.length());
                  String optionName;
                  String optionValue = null;
                  // 是否具备 = 分隔符
                  if (optionText.contains("=")) {
                      // 以分隔符分割，提取 optionName optionValue
                      optionName = optionText.substring(0, optionText.indexOf('='));
                      optionValue = optionText.substring(optionText.indexOf('=')+1, optionText.length());
                  }
                  else {
                      // 否则全部充当 optionName
                      optionName = optionText;
                  }
                  if (optionName.isEmpty() || (optionValue != null && optionValue.isEmpty())) {
                      throw new IllegalArgumentException("Invalid argument syntax: " + arg);
                  }
                  commandLineArgs.addOptionArg(optionName, optionValue);
              }
              // 类型二：其他
              else {
                  commandLineArgs.addNonOptionArg(arg);
              }
          }
          return commandLineArgs;
      }
  }
  ```

  &emsp;&emsp;应用参数分为两类：

  - 可选参数 *Option Arguments*

    语法格式：`--optionName[=optionValue]`

    *optionName* 不允许包含空格

    *optionValue* 首尾不允许有空格，允许以空格、英文逗号为分隔符的字符串，以空格为分隔符时首尾需添加双引号

    

  - 非可选参数 *Non-option Arguments*

    不满足可选参数之外的就属于非可选参数，格式即不包含空格符的字符串

    

  &emsp;&emsp;解析完毕后，根据参数类型不同分别保存在 *CommandLineArgs* 的归类属性中：

  ```java
  class CommandLineArgs {
      // 可选参数，可以看出以键值对形式结构，其中值为字符串列表
      private final Map<String, List<String>> optionArgs = new HashMap<>();
      // 非可选参数，简单的字符串列表
      private final List<String> nonOptionArgs = new ArrayList<>();
  }
  ```

  &emsp;&emsp;该命令行参数对象最终以 *Source* 类型属性保存在 *ApplicationArguments* 的实现类 *DefaultApplicationArguments* 中：

  ```java
  public class DefaultApplicationArguments implements ApplicationArguments {
  
      private final Source source;
      private final String[] args;
  
      public DefaultApplicationArguments(String[] args) {
          // 命令行参数构造 DefaultApplicationArguments
          this.source = new DefaultApplicationArguments.Source(args);
          this.args = args;
      }
  }
  ```

  

* 准备 ConfigurableEnvironment

  略

* 创建 Spring 应用上下文

  - 根据 *WebApplicationType* 创建 Spring 应用上下文

    &emsp;&emsp;根据初始化阶段应用类型推断产生的 *WebApplicationType* 生成具体应用上下文对象，默认：

    

    `WebApplicationType.SERVLET` 对应 *AnnotationConfigServletWebServerApplicationContext*

    `WebApplicationType.REACTIVE` 对应 *AnnotationConfigReactiveWebServerApplicationContext*

    `WebApplicationType.NONE` 对应 AnnotationConfigApplicationContext

    

  - 指定 *ConfigurableApplicationContext* 类型创建 Spring 应用上下文

    &emsp;&emsp;也可在初始化阶段手动设置上下文类型，可通过两个 API 接口实现：

    1. *SpringApplication* 方法 `setApplicationContextClass(Class)`

    2. *SpringApplicationBuilder* 方法 `contextClass(Class)`

       

* Spring 应用上下文运行前准备

  1. Spring 应用上下文准备阶段
  
     - 设置 Spring 应用上下文 *ConfigurableEnvironment*
  
       &emsp;&emsp;原本 Spring 应用上下文初始化生成 *ConfigurableEnvironment* 是在生命周期方法 `refresh()` 中的字方法 `prepareRefresh()`中，之所以 Spring Boot 将其初始化提前至上面提及的 **准备 ConfigurableEnvironment** 步骤，是因为 Spring Boot 中很多配置属性源（*PropertySource*）的装载优先级很高，如果延迟到 Spring 生命周期中完成时依托的 *BeanFactoryPostProcessor* 在众多该类型的处理器中保持最高优先级比较困难。所以提前初始化后，通过此步骤的 `setEnvironment` 方法注入上下文中。
  
       
  
     - Spring 应用上下文后置处理
  
       &emsp;&emsp;Spring 应用上下文后置处理，是 *SpringApplication* 的一个允许扩展的方法，目前用来注册 *BeanNameGenerator* Bean 以及把当前应用的资源加载器（*ResourceLoader*）、类加载器（*ClassLoader*）设置到应用上下文。
  
     
  
     - 运用 Spring 应用上下文初始化器
  
       &emsp;&emsp;*SpringApplication* 构造阶段收集的初始化器列表 `initializers`，该列表为实现 *ApplicationContextInitializer* 接口的实例集合。 根据接口名称可知它是在上下文还未调用 `refresh()` 方法时，对该上下文做一些调整初始化等操作。此处是统一在此进行回调接口定义的初始化方法，这也算是另外一个允许程序扩展的，较上面使用子类覆盖形式的上下文后置处理方法而言，接口形式更灵活。可以通过 **Spring 工厂加载机制** 增加自定义的初始化器实现对上下文的调整操作。内建的初始化器包含：
  
       ```properties
       # Application Context Initializers
       org.springframework.context.ApplicationContextInitializer=\
       # 向上下文注册一个通用错误配置的警告器
       org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
       # 设置上下文 ID 的初始化器
       org.springframework.boot.context.ContextIdApplicationContextInitializer,\
       # 由 context.initializer.classes 属性设置的初始化器的代理初始化器
       org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
       # 设置内嵌 Web Server 监听端口的环境属性
       org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
       ```
  
       
  
     - 执行 *SpringApplicationRunListener* 上下文就绪方法回调
  
       &emsp;&emsp;此方法在此前讨论 Spring Boot 事件广播器时已有说明，在 1.4 版本之前广播器是通过此回调方法进行广播器注册到上下文的 BeanFactory 中达到与 Spring 上下文的广播器共享实例目的。在 1.4 之后此回调方法默认是空方法，没有任何操作，当然这也是留给程序进行扩展的一处时机点。
  
       
  
  2. Sping 应用上下文装载阶段
  
     - 注册 Spring Boot Bean
  
       &emsp;&emsp;此步骤将之前初始化生成的 `applicationArguments`、`printedBanner` 以单例 Bean 形式注册到上下文关联的 BeanFactory。
  
       
  
     - 合并 Spring 应用上下文配置源
  
       &emsp;&emsp;此步骤将之前初始化阶段收集的主配置源、其他配置合并成一个 *Set* 集合。
  
       
  
     - 加载 Spring 应用上下文配置源
  
       &emsp;&emsp;上一步生成的配置源集合交由 *BeanDefinitionLoader* 解析后收集其中的 *BeanDefinition* 到实现 *BeanDefinitionRegistry* 接口的 BeanFactory 中。
  
       
  
     - 执行 SpringApplicationRunListener 上下文加载完毕回调
  
       &emsp;&emsp;当所有上下文配置源加载完毕后，执行上下文加载完毕回调，此过程已经很熟悉，就是之前提及的将 *SpringApplication* 关联的事件监听器做上下文关联回调，以及把这些监听器设置到上下文当中，使其被添加到 Spring 中的事件广播器之中。
  
       

### Spring 应用上下文启动阶段

&emsp;&emsp;在准备阶段结束后，就进入应用上下文的启动阶段，Spring Boot 使用 `refreshContext` 方法封装此阶段的操作。该方法执行两个逻辑：第一是调用其关联的应用上下文实例的 `refresh()` 方法启动 Spring 生命周期，Spring Boot 的核心特性也随之启动：组件自动装配、嵌入式容器启动等。第二是上下文将注册 `shutdownHook` 线程，此线程利用 ***JVM shutdown hook*** 机制，在应用结束时实现优雅的 Spring Bean 销毁生命周期回调。

### Spring 应用上下文启动后阶段

&emsp;&emsp;启动完成后，就进入了上下文启动后阶段，此阶段包含如下过程。

- 启动后扩展方法

  &emsp;&emsp;此方法默认空实现，其留给开发人员自行扩展。值得注意的是 1.4 版本前，其方法包含执行 *Runner* 的调用操作，在之后此操作移除此方法，保持此扩展方法的简单性，否则子类覆盖该方法，都需要 `super` 调用父类该方法来激发 *Runner* 的执行。

  

- 发布 Spring Boot 启动完成事件

  &emsp;&emsp;此部分发布 *ApplicationStartedEvent* 启动完成事件，激发监听器操作。

  

- 执行 *CommandLineRunner* 和 *ApplicationRunner*

  &emsp;&emsp;最后执行一些 *CommandLineRunner*、*ApplicationRunner* 类型的方法，这两个接口都有 `run` 方法，不同的是前者参数类型是命令行的字符串数组，而后者是经过之前的所讲的命令行参数解析生成的 *ApplicationArguments*。同时，它们均支持接口 *Ordered* 或标注 *@Order* 来指定执行顺序。

  - *CommandLineRunner*、*ApplicationRunner* 使用场景

    &emsp;&emsp;这两个类都是在 Spring Boot 启动完成执行，其执行时机完全可以用 Spring Boot 监听器监听 *ApplicationStartedEvent* 事件来替代，但与后者相比扩展便利性稍有不同，各有取舍。监听器形式由于它并非注册到容器中的 Bean, 无法享受注解驱动和 Bean 生命周期回调接口的遍历（得通过关联的上下文进行获取），相反 Runner 类型的是容器 Bean 可以被支持。其次监听器形式它关联的事件源为 *SpringApplication* 实例，而 Runner 类型无法获取该实例。

    

## SpringApplication 结束阶段

&emsp;&emsp;以 Spring Boot 2.0 版本为例，在 *SpringApplication* 的 `run` 方法最后阶段，包含两个执行过程：

```java
public class SpringApplication {
    public ConfigurableApplicationContext run(String... args) {
        ...
        try {
            ...
        }
        catch (Throwable ex) {
            // 异常结束
            handleRunFailure(context, ex, exceptionReporters, listeners);
            throw new IllegalStateException(ex);
        }

        try {
            // 正常执行结束
            listeners.running(context);
        }
        catch (Throwable ex) {
            // 异常结束
            handleRunFailure(context, ex, exceptionReporters, null);
            throw new IllegalStateException(ex);
        }
        return context;
    }
}
```

* *SpringApplication* 正常结束

  &emsp;&emsp;正常结束时，Spring Boot 触发了 SpringApplicationRunListener 的 `running` 方法，从而触发事件广播器广播 *ApplicationReadyEvent* 事件。换言之，可以通过实现 *SpringApplicationRunListener* 监听器的 `running` 方法，或实现监听 *ApplicationReadyEvent* 事件的 *ApplicationListener* 监听器来处理运行后的逻辑处理。

  

* *SpringApplication* 异常结束

  &emsp;&emsp;异常结束时，分两种情况。首先是还未执行到正常结束的 `running` 方法之前的异常，和执行 `running` 方法时的异常。二者区别是处理方法是否传入当前的运行时监听器参数：

  ```java
  public class SpringApplication {
  
      private void handleRunFailure(ConfigurableApplicationContext context,
                                    Throwable exception,
                                    Collection<SpringBootExceptionReporter> exceptionReporters,
                                    SpringApplicationRunListeners listeners) {
          try {
              try {
                  // 处理退出码
                  handleExitCode(context, exception);
                  // 运行时监听器参数不为空时，触发运行监听器的 failed
                  if (listeners != null) {
                      listeners.failed(context, exception);
                  }
              }
              finally {
                  // 报告失败
                  reportFailure(exceptionReporters, exception);
                  if (context != null) {
                      // 关闭上下文
                      context.close();
                  }
              }
          }
          catch (Exception ex) {
              logger.warn("Unable to close ApplicationContext", ex);
          }
          // 抛出运行时异常
          ReflectionUtils.rethrowRuntimeException(exception);
      }
  }
  ```

  &emsp;&emsp;可以看出运行 `running` 方法前异常时，会触发运行监听器执行 `failed` 方法，从而给事件广播器广播 Spring Boot *ApplicationFailedEvent* 事件，而执行监听方法 `running` 时异常是不触发监听器的异常操作。而失败报告器是利用 ***Spring 工厂加载机制*** 所加载的类型为 *SpringBootExceptionReporter* 的集合，默认目前只有一个实现类：

  ```properties
  org.springframework.boot.SpringBootExceptionReporter=\
  org.springframework.boot.diagnostics.FailureAnalyzers
  ```

  &emsp;&emsp;观察该类关键方法：

  ```java
  final class FailureAnalyzers implements SpringBootExceptionReporter {
  
      ...
      private final List<FailureAnalyzer> analyzers;
  
      // 构造方法
      FailureAnalyzers(ConfigurableApplicationContext context, ClassLoader classLoader) {
          this.classLoader = (classLoader != null ? classLoader : context.getClassLoader());
          // 加载分析器
          this.analyzers = loadFailureAnalyzers(this.classLoader);
          // 分析器的属性初始化
          prepareFailureAnalyzers(this.analyzers, context);
      }
  
      private List<FailureAnalyzer> loadFailureAnalyzers(ClassLoader classLoader) {
          // 工厂加载机制加载类型为 FailureAnalyzer 的实例
          List<String> analyzerNames = SpringFactoriesLoader
                  .loadFactoryNames(FailureAnalyzer.class, classLoader);
          List<FailureAnalyzer> analyzers = new ArrayList<>();
          for (String analyzerName : analyzerNames) {
              try {
                  Constructor<?> constructor = ClassUtils.forName(analyzerName, classLoader)
                          .getDeclaredConstructor();
                  ReflectionUtils.makeAccessible(constructor);
                  // 实例化
                  analyzers.add((FailureAnalyzer) constructor.newInstance());
              }
              ...
          }
          AnnotationAwareOrderComparator.sort(analyzers);
          return analyzers;
      }
  
  
      @Override
      public boolean reportException(Throwable failure) {
          // 分析器对异常进程分析
          FailureAnalysis analysis = analyze(failure, this.analyzers);
          // 将分析器交给报告器
          return report(analysis, this.classLoader);
      }
  
      private FailureAnalysis analyze(Throwable failure, List<FailureAnalyzer> analyzers) {
          for (FailureAnalyzer analyzer : analyzers) {
              try {
                  FailureAnalysis analysis = analyzer.analyze(failure);
                  // 一旦有符合的分析时返回
                  if (analysis != null) {
                      return analysis;
                  }
              }
              ...
          }
          return null;
      }
  
      private boolean report(FailureAnalysis analysis, ClassLoader classLoader) {
          // 工厂加载机制找到报告器 FailureAnalysisReporter
          List<FailureAnalysisReporter> reporters = SpringFactoriesLoader
                  .loadFactories(FailureAnalysisReporter.class, classLoader);
          if (analysis == null || reporters.isEmpty()) {
              return false;
          }
          for (FailureAnalysisReporter reporter : reporters) {
              // 将分析依次交给报告器处理
              reporter.report(analysis);
          }
          return true;
      }
  
  }
  ```

  &emsp;&emsp;可以看到 Spring Boot 留了两处基于 Spring 工厂加载机制的扩展，实现针对某异常的自定义 *FailureAnalyzer* 和 *FailureAnalysisReporter*，一旦此处理完毕，就到了应用真正退出阶段。

## Spring Boot 应用退出

* 应用正常退出

  - *ExitCodeGenerator* Bean 生成退出码

    &emsp;&emsp;可以向容器注册一个 *ExitCodeGenerator* 类型的退出码生成器 Bean，它用来在 Spring Boot 应用退出时产生一个退出码。不过该 Bean 产生退出码的方法得通过显式调用 *SpringApplication* 的 `exit` 方法：

    ```java
    public class SpringApplication {
        // 静态退出方法
        public static int exit(ApplicationContext context,
                               ExitCodeGenerator... exitCodeGenerators) {
            int exitCode = 0;
            try {
                try {
                    ExitCodeGenerators generators = new ExitCodeGenerators();
                    // 从上下文中找出 ExitCodeGenerator 类型的 Bean
                    Collection<ExitCodeGenerator> beans = context
                            .getBeansOfType(ExitCodeGenerator.class).values();
                    generators.addAll(exitCodeGenerators);
                    generators.addAll(beans);
                    // 依次调用 ExitCodeGenerator 的 getExitCode 方法，正数时，取最大返回值；负数时，取最小返回值
                    exitCode = generators.getExitCode();
                    if (exitCode != 0) {
                        // 退出码不为 0 时，表示异常退出，激发 ExitCodeEvent 事件
                        context.publishEvent(new ExitCodeEvent(context, exitCode));
                    }
                }
                finally {
                    close(context);
                }
            }
            catch (Exception ex) {
                ex.printStackTrace();
                exitCode = (exitCode != 0 ? exitCode : 1);
            }
            return exitCode;
        }
    }
    ```

    &emsp;&emsp;要想使得最终该返回值在 JVM 退出时被系统进程获取，得将此退出码显式调用 `System.exit(int)` 才能最终实现。

    

  - ExitCodeGenerator Bean 退出码使用场景

    &emsp;&emsp;从上代码可知，退出码不为 0 时，即非正常退出时会发布 *ExitCodeEvent* 类型的事件。可以通过监听该事件进行一些退出时的业务拓展。

    

* 应用异常退出

  &emsp;&emsp;在上一章提及 *SpringApplication* 异常结束时，其中有一步骤是调用方法 `handleExitCode(context, exception)`，该方法最终会去检查当前异常是否是接口 *ExitCodeGenerator* 的实现类，如果是会调用其接口方法产生退出码。

  

  * ExitCodeGenerator 异常使用场景

    &emsp;&emsp;在 Spring Boot 运行事件过程中出现异常，都会执行异常是否实现 *ExitCodeGenerator* 接口，可以自定义实现该接口的异常类在应用逻辑异常时抛出，而后在 Spring Boot 异常结束 `handleExitCode(context, exception)` 调用链的 *SpringBootExceptionHandler* 类将异常码注册绑定到该类的 `exitCode` 属性，而该类实现 *java.lang.Thread.UncaughtExceptionHandler* 接口，JVM 退出时最终会调用 `dispatchUncaughtException`方法而执行此接口的实现方法 `uncaughtException` 使得实现类 *SpringBootExceptionHandler* 的 `exitCode` 最终暴露给 `System.exit(int)` ：

    ```java
    class SpringBootExceptionHandler implements Thread.UncaughtExceptionHandler {
        
        private int exitCode = 0;
    
        // 注册退出码
        public void registerExitCode(int exitCode) {
            this.exitCode = exitCode;
        }
    
        @Override
        // JVM 调用当前线程的 java.lang.Thread.dispatchUncaughtException 转调此方法
        public void uncaughtException(Thread thread, Throwable ex) {
            try {
                if (isPassedToParent(ex) && this.parent != null) {
                    this.parent.uncaughtException(thread, ex);
                }
            } finally {
                this.loggedExceptions.clear();
                // 最终将退出码交给 System.exit
                if (this.exitCode != 0) {
                    System.exit(this.exitCode);
                }
            }
        }
    }
    ```

    

  * ExitCodeExceptionMapper Bean 映射异常与退出码

    &emsp;&emsp;该接口定义了异常与退出码之间的映射关系，当此映射中没有找到相应异常对应的异常代码时，就好继续判断上一小点的 *ExitCodeGenerator* 是否有异常码，故此优先级相对较高。具体执行往容器中定义注册此类型的 Bean 即可生效。不过由于是往容器注册 Bean 的形式，故需依赖上下文对象是激活可用状态，想法上面的接口方式就无此限制。

    

  * 退出码用于异常结束

    &emsp;&emsp;此过程即上面讲解 **ExitCodeGenerator 异常使用场景** 时解释为何最终异常结束时可以将退出码最终关联到 `System.exit` 已做说明。在此说明 *SpringBootExceptionHandler* 是如何关联到当前线程：

    ```java
    public class SpringApplication {
        // 获得 ExceptionHandler
        SpringBootExceptionHandler getSpringBootExceptionHandler() {
            if (isMainThread(Thread.currentThread())) {
                return SpringBootExceptionHandler.forCurrentThread();
            }
            return null;
        }
        // 判断是否主线程
        private boolean isMainThread(Thread currentThread) {
            return ("main".equals(currentThread.getName())
                    || "restartedMain".equals(currentThread.getName()))
                    && "main".equals(currentThread.getThreadGroup().getName());
        }
    }
    ```

    &emsp;&emsp;而 `forCurrentThread` 方法是从 *ThreadLocal* 子类 *LoggedExceptionHandlerThreadLocal* 获得 *SpringBootExceptionHandler*：

    ```java
    private static class LoggedExceptionHandlerThreadLocal
            extends ThreadLocal<SpringBootExceptionHandler> {
        // 内部类
        private static class LoggedExceptionHandlerThreadLocal
                extends ThreadLocal<SpringBootExceptionHandler> {
    
            @Override
            protected SpringBootExceptionHandler initialValue() {
                // 实例化 handler
                SpringBootExceptionHandler handler = new SpringBootExceptionHandler(
                        Thread.currentThread().getUncaughtExceptionHandler());
                // 此处将 handler 关联当前线程
                Thread.currentThread().setUncaughtExceptionHandler(handler);
                return handler;
            }
        }
    }
    ```

    

## 勘误

* Page 71 Jetty 版本

  - Servlet 规范 Jetty 容器版本支持
    | Servlet 规范 | Tomcat | Jetty                                                        | Undertow |
    | ------------ | ------ | ------------------------------------------------------------ | -------- |
    | 4.0          | 9.x    | <font color='red'> ~~9.x~~ </font> <font color='green'>10.x</font> | 2.x      |
    | 3.1          | 8.x    | <font color='red'>~~8.x~~</font> <font color='green'>9.x</font> | 1.x      |
    | 3.0          | 7.x    | <font color='red'>~~7.x~~</font> <font color='green'>8.x</font> | N/A      |
  
  * **资料参考**
    * [Jetty Versions](https://www.eclipse.org/jetty/download.php)

* Page 153、154 属性

  - 第三个表格

    | Spring 注解 | 场景说明                                                     | 起始版本 |
    | ----------- | ------------------------------------------------------------ | -------- |
    | @Primary    | 替换 XML <font color='red'>~~元素~~</font> <font color='green'>属性</font>&lt;bean primary="true \| false"&gt; | 3.0      |
    | @Role       | 替换 XML <font color='red'>~~元素~~</font> <font color='green'>属性</font>&lt;bean role="..."&gt; | 3.1      |
  
  
  - 第六个表格
  
    | Java 注解      | 场景说明                                                     | 起始版本 |
    | -------------- | ------------------------------------------------------------ | -------- |
    | @PostConstruct | 替换 XML <font color='red'>~~元素~~</font> <font color='green'>属性</font>&lt;bean init-method="..."&gt; 或 InitializingBean | 2.5      |
    | @PreDestroy    | 替换 XML <font color='red'>~~元素~~</font> <font color='green'>属性</font>&lt;bean destroy-method="..."&gt; 或 DisposableBean | 2.5      |
  
* Page 159 排版

  * 正文第一段第一行：

    > 并使用 <context:<font color='red'>~~component- scan~~</font><font color='green'> component-scan</font> /> 元素扫

* Page 209 描述歧义

  &emsp;&emsp;AnnotationAttributes 扩展 LinkedHashMap 目的论述存在歧义

  

  * 正文第一段第三行：
  
    > 又要确保其顺序保持与 <font color='red'>~~属性方法声明~~</font> <font color='green'>Class#getDeclaredMethods 方法返回的数组顺序</font>一致。
  
    ***属性方法声明*** ：有人（包括我）误认为是 *注解类的属性方法* 在 **源代码里声明顺序**。
  
    
  
    除上面绿色修改建议外，另一完全使用中文描述候选建议方案：
  
    > 又要确保其顺序保持与 <font color='green'>运行时反射加载的属性方法数组顺序</font> 一致。
  
  * 修改原由
  
    &emsp;&emsp;当把 **属性方法声明顺序** 误认为是属性方法在源代码中声明的顺序时，实际代码验证这种理解是错的。而真正 **属性方法声明顺序** 是与注解的 Class 对象调用 `getDeclaredMethods()` 方法返回的 `Method[]`数组顺序一致。下面的源代码可以支持这一论点：
  
    ```java
    public abstract class AnnotationUtils {
    
        static AnnotationAttributes retrieveAnnotationAttributes(@Nullable Object annotatedElement, Annotation annotation,
                                                                 boolean classValuesAsString, boolean nestedAnnotationsAsMap) {
    
            Class<? extends Annotation> annotationType = annotation.annotationType();
            AnnotationAttributes attributes = new AnnotationAttributes(annotationType);
            // Class#getDeclaredMethods() 方法数组顺序借由 List<Method> 传递到此
            for (Method method : getAttributeMethods(annotationType)) {
                try {
                    Object attributeValue = method.invoke(annotation);
                    Object defaultValue = method.getDefaultValue();
                    if (defaultValue != null && ObjectUtils.nullSafeEquals(attributeValue, defaultValue)) {
                        attributeValue = new DefaultValueHolder(defaultValue);
                    }
                    // attributes 由于是 LinkedHashMap 的扩展，故此处 put 插入顺序得以和 Class#getDeclaredMethods() 保持一致
                    attributes.put(method.getName(),
                            adaptValue(annotatedElement, attributeValue, classValuesAsString, nestedAnnotationsAsMap));
                }
                catch (Throwable ex) {
                    if (ex instanceof InvocationTargetException) {
                        Throwable targetException = ((InvocationTargetException) ex).getTargetException();
                        rethrowAnnotationConfigurationException(targetException);
                    }
                    throw new IllegalStateException("Could not obtain annotation attribute value for " + method, ex);
                }
            }
    
            return attributes;
        }
    
        static List<Method> getAttributeMethods(Class<? extends Annotation> annotationType) {
            // 第一次执行跳过缓存
            List<Method> methods = attributeMethodsCache.get(annotationType);
            if (methods != null) {
                return methods;
            }
    
            methods = new ArrayList<>();
            // 此处收集的属性方法列表，顺序来源于 Class#getDeclaredMethods()
            for (Method method : annotationType.getDeclaredMethods()) {
                if (isAttributeMethod(method)) {
                    ReflectionUtils.makeAccessible(method);
                    methods.add(method);
                }
            }
    
            attributeMethodsCache.put(annotationType, methods);
            return methods;
        }
    }
    ```
  
    &emsp;&emsp;而之所以属性方法顺序不会与源代码中属性方法顺序一致，根源是 Class#getDeclaredMethods 方法返回的 `Method[]` 数组元素顺序的不确定性。
  
    随着每次 JVM 重启后初次调用此方法加载目标 class 文件时， `Method[]` 数组元素顺序都不一致 [^1]。
  
    
  
    &emsp;&emsp;以下引用该方法的 JavaDoc 注释：
  
    > The elements in the returned array are not sorted and are not in any particular order.
  
    ```java
    /**
    * <p> The elements in the returned array are not sorted and are not in any
    * particular order.
    *
    * @jls 8.2 Class Members
    * @jls 8.4 Method Declarations
    * @since JDK1.1
    */
    @CallerSensitive
    public Method[] getDeclaredMethods() throws SecurityException {
      ...
    }
    ```

* Page 294 词语颠倒

  * 正文最后一行：

    > 该引导类在启动 <font color='red'>~~秒后数~~</font> <font color='green'>数秒后</font>，抛出以下异常：


* Page 330 排版

  * 正文倒数第三行：

    > 建议尽可能地使用 @AutoConfigureBefore 或 <font color='red'>~~@AutoConfigureAftername()~~</font> <font color='green'>@AutoConfigureAfter 的 name()</font>属性方法，


* Page 464 多了泛型二字

  * 正文第一行：

    > 2）ApplicationListener 监听自定义 Spring <font color='red'>~~泛型~~</font> 事件


​	&emsp;&emsp;由文中本小节的代码示例中自定义事件的定义 `class MyApplicationEvent extends ApplicationEvent` 可以看出，未携带泛型参数，不属于泛型事件。

* Page 477 访问修饰符

  - 正文表格：

    | 方法类型 | 访问修饰符                                                   |
    | -------- | ------------------------------------------------------------ |
    | 同步     | <font color='red'>~~public~~</font> <font color='green'>任意</font> |
    | 异步     | <font color='red'>~~同上~~</font> <font color='green'>非 private</font> |

    原因参考：[*@EventListener* 实现监听的原理](#)

    


* Page 483 访问修饰符及 @ 符合

  - 正文表格：

    | 监听类型                  | 访问性                                                       | 顺序控制                                                     |
    | ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
    | *@EventListener* 同步方法 | <font color='red'>~~public~~</font> <font color='green'>任意</font> | @Order                                                       |
    | *@EventListener* 异步方法 | <font color='red'>~~public~~</font> <font color='green'>非 private</font> | <font color='red'>~~Order~~</font> <font color='green'>@Order</font> |
    | *ApplicationListener*     | public                                                       | <font color='red'>~~Order 或 Ordered~~</font> <font color='green'>@Order 或 Ordered</font> |

    原因参考：[*@EventListener* 实现监听的原理](#)

    


* Page 490 @EventListener 只支持 public 论述

  - 正文第一段论述 @EventListener 只支持 public 的原因：

    > 由于 AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class) 语句的作用，所以方法筛选的规则是仅判断 Bean 中所有 public 方法是否标注 @EventListenr

    

    &emsp;&emsp;实际上，此方法没有关于访问修饰符的筛选逻辑，仅将 @EventListener 注解转换成一个合成注解，该方法将会把标注此注解的所有修饰符的方法都提取出来。而且通过上面两个勘误得知，同步对访问修饰符无要求，而异步需要非私有、非静态方法。

    

    原因参考：[*@EventListener* 实现监听的原理](#)

    


* Page 538 丢失词语

  * 正文最后一行，需追加 Boot：

    > <font color='red'>Spring </font><font color='green'>Boot</font> <font color='red'>1.4 开始</font>，将它们重构至


* Page 558 方法参数

  ```java
  public ConfigurableApplicationContext run(String... args) {
      StopWatch stopWatch = new StopWatch();
      stopWatch.start();
      ConfigurableApplicationContext context = null;
      configureHeadlessProperty();
      SpringApplicationRunListeners listeners = getRunListeners(args);
      listeners.started();
      try {
          ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
          context = createAndRefreshContext(listeners, applicationArguments);
          afterRefresh(context, applicationArguments);
          listeners.finished(context, null);
          stopWatch.stop();
          if (this.logStartupInfo) {
              new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
          }
          return context;
      } catch (Throwable ex) {
          // 此处多了 analyzers 参数
          // handleRunFailure(context, listeners, analyzers, ex);
          
          // 应替换为下面的调用
          handleRunFailure(context, listeners, ex);
          throw new IllegalStateException(ex);
      }
  }
  ```

  

## 推荐阅读

[《Spring Boot 编程思想（核心篇）》上篇 —— 总览 Spring Boot](https://reionchan.github.io/2022/06/18/thinking-in-springboot-part-1)

[《Spring Boot 编程思想（核心篇）》中篇 —— 走向自动装配](https://reionchan.github.io/2022/06/19/thinking-in-springboot-part-2)



## 脚注
[^1]: IDE 开发环境重新执行 main 方法不会导致 JVM 重新加载未变化的 class 字节码