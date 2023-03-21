---
layout: post
title: Servlet 容器 —— Tomcat 架构及原理
categories: Tomcat
excerpt: 详解 Tomcat 的架构及运行原理
image: https://tomcat.apache.org/res/images/tomcat.png
description: 详解 Tomcat 的架构及运行原理
keywords: Tomcat Servlet Web Server architecture
licences: cc
repo: tomcat
---

<br/>

<img src="https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/webapps/ROOT/tomcat.svg" alt="Tomcat Logo" width="300"/>

> Apache Tomcat 旨在成为荟聚世界众多优秀开发者合作完成的优秀项目。

## Tomcat 介绍

### Tomcat 是什么？

&emsp;&emsp;**Apache Tomcat** 软件是对 Servlet、JSP、EL、WebSocket 等 Jakarta EE 平台技术规范的开源实现。它遵照 Apache 许可（版本二）发行，为各种行业的重大 Web 应用程序提供支持。它是 Servlet 的容器，给 Servlet 提供一个符合 JakartaEE 规范的运行时环境。

&emsp;&emsp;**Servlet** 是基于 Java 技术的 Web 组件，它被容器管理进而与 Web 客户端以请求响应的方式进行交互，从而产生动态内容。**Tomcat 就是这种管理 Servlet 组件生命周期的容器**，它既可以是 Web 服务器的一部分也可以是独立的 Web 应用服务器，由此服务器提供基于请求响应式的网络服务，它接收并解析基于 MIME 的请求，形成基于 MIME 的响应发送给客户端。

&emsp;&emsp;本文基于 **Apache Tomcat 8.5** 版本源码来详细分析 Tomcat 总体架构及运行详情，同时关注 Spring Boot 将其整合成为内嵌 Servlet 容器时的相关配置调整。该版本实现了 **Servlet 3.1 规范 [JSR340](https://www.jcp.org/en/jsr/detail?id=340)**，故而后续 Spring Boot 也是选取基于此内嵌 Tomcat 版本的 **2.0.2.RELEASE**，文中所有涉及的源代码如无特殊说明均为各自对应版本。另外可以参阅 [Apache Tomcat Versions](https://tomcat.apache.org/whichversion.html) 来查看不同 Tomcat 版本对应的 Jakarta EE 规范版本。

### Servlet 规范简介

&emsp;&emsp;要讲解 Servlet 在具体的容器运行过程之前，先简要介绍 Servlet 规范的主要体系结构，为接下来了解此体系结构如何在 Tomcat 中得到具体实现。下面是 Servlet 规范中重要接口的关系图。

![Servlet 体系结构](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/servlet_3_1_class.png)

| 接口                | 描述                                                         |
| :------------------ | :----------------------------------------------------------- |
| **Servlet**         | Java Servlet API 核心抽象，定义 Servlet 生命周期方法。       |
| **ServletConfig**   | Servlet 初始化时，由 Servlet 容器传递给 Servlet 。该抽象包含此 Servlet 参数配置信息。 |
| **ServletRequest**  | 封装所有来自客户端请求相关信息的抽象。客户端如使用 HTTP 协议通讯，那么此信息将包含 HTTP 请求头及消息体。 |
| **ServletResponse** | 封装所有来自服务端响应相关信息的抽象。服务端如使用 HTTP 协议通讯，那么此信息将包含 HTTP 响应头及消息体。 |
| **ServletContext**  | 为 Servlet 提供获取当前的 Servlet 容器运行环境以及交互的抽象，即当前 Web 应用的视图。提供对 Servlet 组件的维护及容器相关信息的获取。 |

&emsp;&emsp;这五个抽象定义出 Servlet 在处理客户端与服务端交互的逻辑抽象。Servlet 初始化时，容器将包含此 Servlet 配置参数信息封装成 ServletConfig 传递给 Servlet 从而完成定制化的初始化。当客户端一次请求到来时，首先将客户端的请求信息封装成 ServletRequest 交给 Servlet。随后 Servlet 根据请求信息并结合 ServletContext 获取当前 Web 应用下请求所对应的资源进行业务逻辑处理，最后将响应结果封装成 ServletResponse 返回给客户端。在这次请求响应中，ServletContext 充当客户端服务器交互的一个场景视图，它能为 Servlet 提供运行时对容器的交互操作，例如：记录日志、获得请求相对应的资源、设置存储一些属性使得其他在此场景视图下其它 Servlet 能够访问这些属性。

## Tomcat 的 Servlet 规范实现

&emsp;&emsp;Tomcat 既然被称为 Servlet 的容器，那么它必然遵照 Servlet 规范为 Servlet 的全生命周期提供一个整体的运行环境，其中包含对 Servlet 的创建、初始化、运行、销毁等操作。此章节将介绍 Tomcat 单纯作为 Servlet 容器时，究竟是如何构建出 Servlet 的运行环境，以及 Servlet 在 Tomcat 容器中具体执行过程。

&emsp;&emsp;Tomcat 通过 Wrapper 容器对 Servlet 进行封装，再将 Wrapper 容器放入上层 Context 容器。Tomcat 容器层级模型如下图所示。

![Tomcat 容器组件模型](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_container_model.svg)

&emsp;&emsp;从上图可以看出，Tomcat 容器组件分为四个等级：Engine、Host、Context、Wrapper，而真正管理 Servlet 的容器组件为 Context。一个 Context 对应一个 Web 项目工程，其中每个工程的 Context 容器组件中包含若干个 Servlet 组件的包装组件 Wrapper。因此对 Tomcat 而言，多个 Context 容器组件的集合才算是真正意义上的 Servlet 容器（*Servlet Container*）。

### Servlet 容器启动

&emsp;&emsp;既然 Context 是真正管理 Servlet 的容器，那么跟踪 Context 容器的生命周期过程就可得知 Servlet 配置是如何被扫描解析并生成包装容器 Wrapper 后添加到 Context 容器中，以及最后 Servlet 的实例化及初始化的过程。下面是 Tomcat 有关 Context 容器的初始化及启动的时序图（Tomcat 容器全生命周期时序图见后续章节）。

![Context Starting Process](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_context_init_start_lifecyle.svg)

&emsp;&emsp;Engine 容器启动时，将会联动的启动其内部的子容器。而作为 Servlet 运行时容器 —— Context 容器也被联动的启动。Tomcat 容器都会继承 Lifecycle 接口，都具备一致的初始化及启动逻辑。而所有容器在初始化启动过程中会伴随着状态的变化，这些变化会通知给已注册到容器中的各种监听器（Listener）从而完成生命周期各个阶段的一些额外操作。Host 容器就注册了一个声明周期监听器 **HostConfig**，而 Context 容器也注册了一个生命周期监听器 **ContextConfig**，它在 Context 容器的不同生命周期阶段执行一些相应的逻辑。**值得注意的是**，Tomcat 应用部署目录 webapps 中的应用由于在 Host 启动时还未被添加到 Host 容器中不会再启动中联动启动，故这些应用会延迟到 Host 容器启动后状态为 STARTING 时由监听器 HostConfig 扫描解析部署后形成 Context 容器，随即在添加到 Host 容器时，激发容器添加事件执行 Context 启动操作。

* Context 容器初始化后

  &emsp;&emsp;图中 Context 容器在执行生命周期初始化方法 `init()` 后其状态变更为 **INITIALIZED**，会使的 ContextConfig 激发初始化操作，而初始化时会执行 `contextConfig()` 方法完成默认的上下文配置文件 `conf/context.xml` 以及 Context 自身的配置文件 `META-INF/context.xml` 的解析操作。

* Context 容器启动时

  &emsp;&emsp;图中 Context 容器在执行生命周期启动方法 `start()` 时其在执行过程中会在其子容器启动之前激发 **`CONFIGURE_START_EVENT`** 事件，使得 ContextConfig 执行 `configureStart()` 方法完成 Web 应用配置文件 `conf/web.xml`以及 Context 自身的配置文件 `WEB-INF/web.xml` 解析，将得到的信息放入 Context 容器之中，而其中的 Servlet 配置信息则额外封装为 Wrapper 子容器添加到 Context 容器，随后启动这些子容器完成。其中以下代码片段来源于 ContextConfig 类的 `configureContext(WebXml)` 方法，功能是将 Servlet 封装为 Wrapper：

  ```java
  private void configureContext(WebXml webxml) {
      // 省略代码 ...
      for (ServletDef servlet : webxml.getServlets().values()) {
          // 创建 Wrapper 包装类    
          Wrapper wrapper = context.createWrapper();
  
          if (servlet.getLoadOnStartup() != null) {
              wrapper.setLoadOnStartup(servlet.getLoadOnStartup().intValue());
          }
          if (servlet.getEnabled() != null) {
              wrapper.setEnabled(servlet.getEnabled().booleanValue());
          }
          wrapper.setName(servlet.getServletName());
          // 设置初始化参数 Map        
          Map<String,String> params = servlet.getParameterMap();
          for (Entry<String, String> entry : params.entrySet()) {
              wrapper.addInitParameter(entry.getKey(), entry.getValue());
          }
          // ...
          // 设置 Servlet 类全限定名
          wrapper.setServletClass(servlet.getServletClass());
          // ...
          // 将 Wrapper 放入 Context 父容器
          context.addChild(wrapper);
      }
      // 省略代码 ...
  }
  ```

### Servlet 实例化与初始化

&emsp;&emsp;在上面 Servlet 容器（即：Context 容器）启动过程中，如果在 Servlet 的配置中将 `load-on-startup` 参数配置为非负数则表明此 Servlet 在容器启动时进行实例化以及初始化。Context 容器在其启动过程中会执行 `loadOnStartup` 方法将符合条件的 Servlet 创建并初始化，下面来介绍具体的创建及初始化的过程。

```java
public class StandardContext extends ContainerBase implements Context, NotificationEmitter {
    public boolean loadOnStartup(Container children[]) {

        // 筛选出 loadOnStartup >= 0 的 warapper，按照值进行排序
        TreeMap<Integer, ArrayList<Wrapper>> map = new TreeMap<>();
        for (Container child : children) {
            Wrapper wrapper = (Wrapper) child;
            int loadOnStartup = wrapper.getLoadOnStartup();
            if (loadOnStartup < 0) {
                continue;
            }
            Integer key = Integer.valueOf(loadOnStartup);
            ArrayList<Wrapper> list = map.get(key);
            if (list == null) {
                list = new ArrayList<>();
                map.put(key, list);
            }
            list.add(wrapper);
        }

        // 按照值小优先级高的原则依次加载 servlet
        for (ArrayList<Wrapper> list : map.values()) {
            for (Wrapper wrapper : list) {
                try {
                    // Servlet 实例化初始化关键方法
                    wrapper.load();
                } catch (ServletException e) {
                    // ...
                }
            }
        }
        return true;
    }
}
```

&emsp;&emsp;下面来看 Wrapper 的 `load()` 方法：

```java
public class StandardWrapper extends ContainerBase implements ServletConfig, Wrapper, NotificationEmitter {
    @Override
    public synchronized void load() throws ServletException {
        // 实例化方法
        instance = loadServlet();
        if (!instanceInitialized) {
            // 初始化方法
            initServlet(instance);
        }
        // ...
    }
}
```

* 实例化

  &emsp;&emsp;具体实例化方法 `loadServlet()`：

  ```java
  public class StandardWrapper extends ContainerBase implements ServletConfig, Wrapper, NotificationEmitter {
      public synchronized Servlet loadServlet() throws ServletException {
  
          // ...
          Servlet servlet;
          try {
              // 由父容器中的 instanceManager 进行反射实例化
              InstanceManager instanceManager = ((StandardContext)getParent()).getInstanceManager();
              try {
                  servlet = (Servlet) instanceManager.newInstance(servletClass);
              } catch (ClassCastException e) {
                  // ...
              } catch (Throwable e) {
                  //...
              }
              // 初始化
              initServlet(servlet);
              // 触发容器事件 load
              fireContainerEvent("load", this);
          } finally {
              // ...
          }
          return servlet;
      }
  }
  ```

* 初始化

  &emsp;&emsp;具体初始化方法 `initServlet()`：

  ```java
  public class StandardWrapper extends ContainerBase implements ServletConfig, Wrapper, NotificationEmitter {
      /**
       * 本 wrapper 的门面，充当 Servlet 初始化时传入的 ServletConfig，使用了门面模式
       */
      protected final StandardWrapperFacade facade = new StandardWrapperFacade(this);
      
      private synchronized void initServlet(Servlet servlet)
              throws ServletException {
          // ...
          try {
              if( Globals.IS_SECURITY_ENABLED) {
                  // ...
              } else {
                  // 将 wrapper 的门面以 ServletConfig 传入 Servlet 初始化方法               
                  servlet.init(facade);
              }
              instanceInitialized = true;
          } catch (UnavailableException f) {
              // ...
          }
      }
  }
  ```

  &emsp;&emsp;注意初始化 Servlet 所传递的参数是 ServletConfig 的子类 **StandardWrapperFacade**，它也是当前包装类的门面类，用来对 Servlet 隐藏 StandardWrapper 的一些不必要的属性及方法。采用门面模式来隐藏一些不必要的信息在 Tomcat 中有很多，比如：在下一节介绍 Tomcat 对 Servlet 规范接口实现中 **ApplicationContextFacade**、**RequestFacade**、**ResponseFacade**。

### Servlet 规范实现

&emsp;&emsp;在 **Servlet 规范简介** 小节中介绍了规范中重要的五个接口以及它们的交互关系，从而勾勒出规范所规定的客户端及服务端交换信息的大致流程。下图是 Tomcat 对 Servlet 规范的五个接口的实现：

![Tomcat Servlet Implement](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/servlet_3_1_class_impl.png)

* ServletConfig、ServletContext 实现

  &emsp;&emsp;不难看出，为了对数据的封装四个接口都使用了门面模式。由上节可知，Servlet 初始化时所传递的 ServletConfig 实例类型是 **StandardWrapperFacade**，而该类实现 ServletConfig 接口方法 `getServletContext()` 返回的 ServletContext 实例类型是 **ApplicationContextFacade**，它们分别是对 **StandardWrapper**、**ApplicationContext** 两个类。对于 Servlet 实例来说，它所能使用的接口的实例都只能获得对应门面类暴露的信息，而无法获得原始类完整信息。

* ServletRequest、ServletResponse 实现

  &emsp;&emsp;Tomcat 没有直接实现 ServletRequest、ServletResponse 接口，而是实现其子接口 HttpServletRequest、HttpServletResponse，分别对应 **org.apache.catalina.connector.Request**、**org.apache.catalina.connector.Response**。之所以加上包名是为了区分 **org.apache.coyote.Request**、**org.apache.coyote.Response** 这两个类，它们分属 Tomcat 的 **Coyote**、**Catalina** [^1] 组件之中，Coyote 组件是 Tomcat 作为 Web 服务器的连接器组件，用来支持 HTTP 连接协议，而 Catalina 组件是 Tomcat 的 Servlet 容器组件。来自客户端的请求先被 Coyote 连接器接收，然后转化后传递给 Catalina 容器，最后封装为门面对象交给 Servlet 处理，具体请求的转换流转如下图所示：

  

  ![Reqest Transfer Model](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_request_transfer_model.svg)

  

  &emsp;&emsp;**CoyoteAdapter** 负责将 Coyote 请求转换为 Catalina 请求，而 **ApplicationFilterChain** 在调用 `doFilter` 方法时将 Catalina 请求封装成门面请求类交给过滤器及后续的 Servlet 处理。 之所以设计出三种不同的请求响应类型，是由上图中三个不同的环境特征决定的。

  1. Coyote 环境

     &emsp;&emsp;此环境是客户端请求的连接点，主要目的是增加单位时间的客户端请求吞吐量，所以对客户端的请求的封装要尽可能简单高效且具备可复用性。细看 coyote 包下的 **Request**、**Response** 类定义，不难发现其成员变量大多设计成 GC-Free （复用性）或容易被 JVM 回收（高效性）。

  2. Catalina 环境

     &emsp;&emsp;此环境是 Servlet 容器环境，而 Servlet 接口在 Catalina 环境下的实际实现类都继承至 **HttpServlet**，故 catalina 包下的 **Request**、**Response** 继承于 HTTP 协议的 **HttpServletRequest**、**HttpServletRespose**。此外该请求响应类型一直贯穿整个 Servlet 容器，故而包含了很多容器相关以及 Servlet 规范所涉及的变量。

  3. Servlet 环境

     &emsp;&emsp;此环境在 Servlet 内部使用，给到 Servlet 的  **Request**、**Response** 单纯能够满足 Servlet 规范规定的所有功能即可，故 Servlet 容器最终将 catalina 包下的  **Request**、**Response** 进行门面模式转换封装成  **RequestFacade**、**ResponseFacade** 传递给 Servlet。

* Servlet 请求映射

  &emsp;&emsp;到此 Tomcat Servlet 容器启动过程中如何实例化、初始化 Servlet 已经有大致认识，但请求在经过上述的转换后最终路由映射到某个 Servlet 的具体处理过程还不得而知，此小节将重点关注 Tomcat 容器对 Servlet 请求映射实现逻辑。
  
  &emsp;&emsp; Servlet 请求映射分两个阶段：**映射关系解析收集**、**映射关系匹配**。收集阶段发生在容器启动阶段，结合监听器模式在容器发生变化时变更已收集到的映射集合；匹配阶段发生在容器接收到客户端请求后，CoyoteAdapter 把请求交给 Servlet 容器处理之前。
  
  &emsp;&emsp;下面详细介绍这两个阶段的实现过程。
  
  1. 映射关系收集
  
     &emsp;&emsp;收集发生在容器启动阶段以及之后容器变更时，所以收集操作使用监听器模式再合适不过。实际也的确如此，该监听器为 **MapperListener**：
  
     ![MapperListener](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_mapper_listener.png)
  
     &emsp;&emsp;MapperListener 实现了容器监听器、生命周期监听器两个接口，分别用来监听容器的变动及容器生命周期变动事件来达到动态调整映射关系集合。与此同时，该监听器本身也实现了生命周期接口，它在 Servlet 容器（Engine、Host、Context、Wrapper）启动完成后也执行自身的启动方法，一方面将自身注册到所有 Servlet 容器中，另一方面对已有容器进行收集操作。
  
     ```java
     public class MapperListener extends LifecycleMBeanBase implements ContainerListener, LifecycleListener {
         // 映射器    
         private final Mapper mapper;
         
         @Override
         public void startInternal() throws LifecycleException {
     
             setState(LifecycleState.STARTING);
     
             Engine engine = service.getContainer();
             if (engine == null) {
                 return;
             }
     
             findDefaultHost();
             // 将映射监听器注册到容器以及所有子容器
             addListeners(engine);
     
             Container[] conHosts = engine.findChildren();
             for (Container conHost : conHosts) {
                 Host host = (Host) conHost;
                 if (!LifecycleState.NEW.equals(host.getState())) {
                     // 收集注册 Host, 同时会嵌套将子容器 Context、Wrapper 一并收集
                     registerHost(host);
                 }
             }
         }
         
         private void registerHost(Host host) {
             // 收集注册 Host
             mapper.addHost(host.getName(), aliases, host);
             // 收集注册子容器 Context
             for (Container container : host.findChildren()) {
                 if (container.getState().isAvailable()) {
                     registerContext((Context) container);
                 }
             }
         }
         
         private void registerContext(Context context) {
             // 单个 servlet 包含多个映射时，每个映射形成一个 WrapperMappingInfo
             List<WrapperMappingInfo> wrappers = new ArrayList<>();
     
             for (Container container : context.findChildren()) {
                 prepareWrapperMappingInfo(context, (Wrapper) container, wrappers);
             }
             // 将此 Context 添加到 mapper 映射器中
             mapper.addContextVersion(host.getName(), host, contextPath,
                     context.getWebappVersion(), context, welcomeFiles, resources,
                     wrappers);
         }
         
         private void registerWrapper(Wrapper wrapper) {
     
             Context context = (Context) wrapper.getParent();
             String contextPath = context.getPath();
     
             String version = context.getWebappVersion();
             String hostName = context.getParent().getName();
     
             List<WrapperMappingInfo> wrappers = new ArrayList<>();
             // 单个 servlet 包含多个映射时，每个映射形成一个 WrapperMappingInfo
             prepareWrapperMappingInfo(context, wrapper, wrappers);
             // 将此 Wrapper 添加到 mapper 映射器中
             mapper.addWrappers(hostName, contextPath, version, wrappers);
         }
     }
     ```
  
     &emsp;&emsp;在启动方法 `startInternal()` 中的 `addListeners(engine)` 方法会递归将所有子容器都添加此映射监听器，而之后的 `registerHost(host)` 操作会把参数 host 中的所有子容器一并注册到映射容器中。当然，该类也提供单独针对某个容器变动的添加操作，方法名称以 `register` 开头。
  
     &emsp;&emsp;从上面代码可以看出，所有注册的映射关系都被收集到类 **Mapper** 之中，它即是映射关系的保存集合，也是下一阶段处理映射关系匹配功能的实现类。此处先来看它内部存储映射关系的结构设计：
  
     ![Mapping Model](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_mapping_model.svg)
  
     - Host、Context、Wrapper 容器映射包装类
  
       &emsp;&emsp;**Host**、**Context**、**Wrapper** 属于 Tomcat 容器组件，具体介绍讲放到 Tomcat 架构设计章节进行讲解，目前只需知道 Host 对应不同的虚拟主机，Context 对应具体某个应用，Wrapper 对应该下的具体 Servlet。假如客户端发出请求：`http://foo.org/app/hello`，在 Tomcat 容器的映射体规则下，域名 `foo.org` 被认为是该名称对应的 Host 容器实例， `/app` 被认为是该名称对应的 Context 容器实例（Context Path）, `/hello` 则被认为是该名称对应的 Wrapper 容器实例，其对应 Servlet 关联的映射配置。
  
       ```sh
       # Web URL Pattern
       PROTOCOL://DOMAIN:PORT/CONTEXT/servletpath
       # 例子
       http://foo.org:8080/app/hello
       # 映射对应关系
       DOMAIN		foo.org		Host
       CONTEXT		app			Context
       servletpath	hello		Wrapper
       ```
  
       
  
       &emsp;&emsp;&emsp;**Host**、**Context**、**Wrapper** 分别有其相匹配的映射包装类 **MappedHost**、**ContextVersion**、**MappedWrapper**。Context 的映射类之所以不是 **MappedContext** 是因为同一个应用 Context 支持多版本共存，具体版本的 Context 由 **ContextVersion** 来封装，**MappedContext** 只用来汇总相同应用 Context 的不同版本，方便交由上层 **MappedHost** 映射封装类操作管理。
  
       
  
     - 映射包装类包含关系
  
       &emsp;&emsp;从上面类图可以看出，在一个 Service 服务下维护这一个映射关系容器 Mapper。该 **Mapper** 包含多个 **MappedHost** 映射（即不同名称的 host ），而单个 **MappedHost** 下包含一个 **ContextList** 实例，该实例包含多个 **MappedContext** 映射（即不同名称的 context），而单个 **MappedContext** 下包含多个 **ContextVersion** （即相同名称的 context 的不同版本），而每个 context 的具体版本里包含多种类型的 **MappedWrapper** 映射（即不同映射规则的 wrapper）。**MappedWrapper** 映射实际反映的是 **Wrapper** 所封装的 **Servlet** 的映射配置。根据 Servlet 规范对映射规则的描述，映射规则分为三种：精确匹配映射、前缀匹配映射、扩展匹配映射，分别对应 exactWrappers、wildcardWrappers、extensionWrappers。
  
       
  
  2. 映射关系匹配
  
     &emsp;&emsp;&emsp;至此，所有的映射关系就随着容器的启动被汇总到 Service 容器组件的 Mapper 类中。而当客户端发起请求时，服务器将该请求转换为 Catalina 容器请求，并根据请求路径进行映射关系匹配后交给响应的 Servlet 容器处理。映射匹配后的结果将被保存在 Catalina 容器请求 Request 的成员变量 **MappingData** 中，该类包含一个请求最终映射到 Wrapper 的三个要素：Host、Context、Wrapper。
  
     ```java
     public class MappingData {
         // 匹配的虚拟主机
         public Host host = null;
         // 匹配的上下文，即应用名称
         public Context context = null;
         // 请求路径 / 的个数
         public int contextSlashCount = 0;
         // 不同版本的上下文实例数组
         public Context[] contexts = null;
         // 匹配的 Wrapper
         public Wrapper wrapper = null;
         // 是否是 jsp 通配符，/* 映射的
         public boolean jspWildCard = false;
     
         public final MessageBytes contextPath = MessageBytes.newInstance();
         public final MessageBytes requestPath = MessageBytes.newInstance();
         public final MessageBytes wrapperPath = MessageBytes.newInstance();
         public final MessageBytes pathInfo = MessageBytes.newInstance();
         public final MessageBytes redirectPath = MessageBytes.newInstance();
         public ApplicationMappingMatch matchType = null;
     }
     ```
  
     &emsp;&emsp;客户端请求被服务端接收后在 **CoyoteAdapter** 的 `service(org.apache.coyote.Request req, org.apache.coyote.Response res)` 方法中将 **org.apache.coyote.Request** 类型的请求转换为 **org.apache.catalina.connector.Request** 类型的请求，并通过方法 `postParseRequest(req, request, res, response)` 对请求路径进行映射关系查找匹配，将结果保存于 **org.apache.catalina.connector.Request** 的 `mappingData` 属性中。具体匹配逻辑在 **Mapper** 的 `map` 两个重构方法之中：
  
     ```java
     public final class Mapper {
     
         // 指定 host、请求 uri、context 版本来匹配
         public void map(MessageBytes host, MessageBytes uri, String version,
                         MappingData mappingData) throws IOException {
     
             if (host.isNull()) {
                 String defaultHostName = this.defaultHostName;
                 if (defaultHostName == null) {
                     return;
                 }
                 host.getCharChunk().append(defaultHostName);
             }
             host.toChars();
             uri.toChars();
             // 具体匹配算法       
             internalMap(host.getCharChunk(), uri.getCharChunk(), version, mappingData);
         }
     
         // 指定 Context、请求 uri 来匹配
         public void map(Context context, MessageBytes uri,
                 MappingData mappingData) throws IOException {
     
             ContextVersion contextVersion =
                     contextObjectToContextVersionMap.get(context);
             uri.toChars();
             CharChunk uricc = uri.getCharChunk();
             uricc.setLimit(-1);
             // 具体匹配算法 
             internalMapWrapper(contextVersion, uricc, mappingData);
         }
     }
     ```
  
     &emsp;&emsp;匹配结果保存到 **Request** 的成员变量 `mappingData` 中，其包含该次请求映射的三个目标容器（Host、Context、Wrapper）的具体容器实例。 **Request** 转交给 Engine 容器后，触发父子容器的管道式的责任链调用（该责任链调用会在 Tomcat 架构设计章节详细说明）。从 Engine 到 Host 再到 Context 一直到 Wrapper 都通过一个链传递 Request，而传递中流转到 Engine 的具体哪个 Host、Context、Wrapper 实例，就是通过 `mappingData` 保存的三个目标容器实例。
  
     &emsp;&emsp;下面拿 Engine 流转到 Host 时，挑选出目标 Host 实例为例进行说明（Host 流转到 Context，Context 流转到 Wrapper 类似，故此处省略）：
  
     ```java
     final class StandardEngineValve extends ValveBase {
     
         @Override
         public final void invoke(Request request, Response response)
             throws IOException, ServletException {
     
             // 此处从 request 取出目标 host
             Host host = request.getHost();
             // 省略...
             // 调用目标 host 的责任链 
             host.getPipeline().getFirst().invoke(request, response);
         }
     }
     ```
  
     &emsp;&emsp;详细查看 `request.getHost()` 代码：
  
     ```java
     public class Request implements HttpServletRequest {
         // 映射结果 mappingData    
         protected final MappingData mappingData = new MappingData();
         
         // 从映射结果 mappingData 中获得目标 Host 实例  
         public Host getHost() {
             return mappingData.host;
         }
     
         // 从映射结果 mappingData 中获得目标 Context 实例
         public Context getContext() {
             return mappingData.context;
         }
     
         // 从映射结果 mappingData 中获得目标 Wrapper 实例
         public Wrapper getWrapper() {
             return mappingData.wrapper;
         }
         
         // 获取当前请求的门面类 RequestFacade     
         public HttpServletRequest getRequest() {
             if (facade == null) {
                 facade = new RequestFacade(this);
             }
             if (applicationRequest == null) {
                 applicationRequest = facade;
             }
             return applicationRequest;
         }    
     }
     ```
     
     &emsp;&emsp;流转到目标 Wrapper 后，执行 Wrapper 的管道中的 Valve 链后，在最后一个执行的 Valve 实现类 **StandardWrapperValve** 的 `invoke(request, response)` 方法中，构造过滤器链 **ApplicationFilterChain** 来处理 **request**，并且此 request 交由该过滤器链处理的请求对象已经被替换成 RequestFacade 门面类。在所有过滤器处理完毕后，最终将 requestFacade 交给 Wrapper 封装的 Servlet 实例的 `service(ServletRequest req, ServletResponse res)` 方法处理。 
     
     ```java
     final class StandardWrapperValve extends ValveBase {
         @Override
         public final void invoke(Request request, Response response)
             throws IOException, ServletException {
             // 省略代码...
             // 为当前请求构造过滤器链
             ApplicationFilterChain filterChain =
                     ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);
     
             Container container = this.container;
             try {
                 if ((servlet != null) && (filterChain != null)) {
                     if (context.getSwallowOutput()) {
                         try {
                             SystemLogHandler.startCapture();
                             if (request.isAsyncDispatching()) {
                                 request.getAsyncContextInternal().doInternalDispatch();
                             } else {
                                 // 请求交给过滤器链处理时，已经转换成 RequestFacade 类型
                                 // request.getRequest() 代码详情参考上一代码片段                         
                                 filterChain.doFilter(request.getRequest(),
                                         response.getResponse());
                             }
                         } finally {
                             // ...
                         }
                     } else {
                         //...
                     }
                 }
             } catch (ClientAbortException | CloseNowException e) {
                 // ...            
             } finally {
                 // ...
             }
         }
     }
     ```

&emsp;&emsp;本章节详细描述了 Servlet 在 Tomcat 容器中进行实例化与初始化、请求映射以及最终调用执行的过程，而这三个过程的详细处理过程就是 Tomcat 作为 Servlet 容器对 Servlet 规范的具体实现。当然本章节并没有涵盖 Tomcat 对全部 Servlet 规范的所有实现细节，只仅包含最主要的部分，而忽略的部分有：过滤器、请求转发、应用生命周期、部署描述器等等。

## Tomcat 架构设计

&emsp;&emsp;上一章节着重关注 Tomcat 作为 Servlet 容器时对 Servlet 规范实现的种种细节，而本章节将会在整体架构设计上对 Tomcat 进行剖析。

### 总体架构设计

&emsp;&emsp;Tomcat 设计了很多组件，将这些组件按照拼合、包含嵌套等方式组合就能成为各个功能模块，而 Tomcat 的架构就由这些功能模块组成。下图是 Tomcat 的总体架构设计结构图。

<a herf="https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_architectrue.svg" target="_blank">![Tomcat Architecture](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_architectrue.svg)</a>

&emsp;&emsp;从图中可以看出 Tomcat 包含两个核心功能模块：**Coyote 连接器**、**Catalina Servlet 容器**。这两个模块将会在下文详细讲解，此处只需了解 Coyote 连接器组件是用来接受客户端的连接，并将请求传递给 Catalina Servlet 容器组件，而后者是支持 Servlet 整个生命周期的完整环境，是它将来自 Coyote 连接器传递过来的请求最终路由给 Servlet 处理。多个 Coyote 连接器组件可以连接到一个 Catalina Servlet 容器组件，再结合一些其他的基础功能组件，例如：Jasper （JSP 引擎）、JMX 等组件就构成一个 Service 组件并可以对外提供服务，而整个 Tomcat 服务器 **Server 组件**能管理控制多个这样的 Service 组件。

### 重要组件介绍

&emsp;&emsp;在介绍 Coyote、Catalina 两个核心功能模块之前，先来看构成这两个核心功能模块的一些重要组件。按照架构设计可以大致分为三类：顶级组件、连接器组件、容器组件。

* 顶级组件

  - Server

    &emsp;&emsp;在谈及 Tomcat 这个服务器时，Server 组件就代表某台 Tomcat 服务器实例。它下面可以包含多个 Service 组件及一些顶级命名资源，它的生命周期方法的执行会使其包含的所有 Service 执行同名方法。它的启动与关闭即代表整个 Tomcat 服务器的启动与关闭。

  - Service

    &emsp;&emsp;Service 组件是运行于 Server 组件之内，它将**一个或多个** Connector 组件与**单个** Engine 组件相关联，从而使得 Engine 组件可以处理来自不同连接方式的请求。

    

* 连接器组件

  * Connector

    &emsp;&emsp;Connector 处理与客户端的通信，在 Tomcat 中可以支持不同的连接器。例如：当 Tomcat 作为单独的 服务器时，HTTP 连接器就用来处理基于 HTTP 协议的客户端请求；而当借助 AJP 连接器时，Tomcat 就可以基于 AJP 协议与 Apache HTTPD 服务器进行连接。

    

* 容器组件

  * Engine

    &emsp;&emsp;Engine 组件代表着某个 Service 组件下的请求处理流水线，由于 Service 组件可以包含多个 Connector，所以 Engine 需要接收并处理来自不同 Connector 的请求，并将响应结果返回给对应的 Connector。

  * Host

    &emsp;&emsp;Host 组件起到将 Tomcat 服务器与域名进行绑定的作用，在一个 Engine 组件下面支持多个 Host 组件，换言之单个 Engine 可以映射多个域名。例如：domain1.org、domain2.org

  * Context

    &emsp;&emsp;Context 组件代表着一个 Web 应用，一个 Host 组件下可以包含多个 Context 组件，每个 Context 组件有着唯一的路径标识。例如：domain.org**/app1**、domain.org**/app2** 就表示名为 **domain.org** 下的  app1、app2 两个 Web 应用。

  * Wrapper

    &emsp;&emsp;Wrapper 组件代表着一个 Servlet，一个 Context 组件下可以包含多个 Wrapper 组件。每个 Wrapper 负责管理其内部封装的 Servlet 的装载、初始化、执行及销毁整个生命周期。该组件也是容器组件的最底层组件，它为每个客户端请求生成一条过滤器链，请求经过此过滤器链的处理后最终交给其内部封装的 Servlet 实例。

### Coyote 连接器

&emsp;&emsp;先来看位于结构图左侧的 Coyote 连接器，它实现 Tomcat 服务器与客户端的连接通信。说道计算机的通信那肯定涉及通信协议，该模块抽象出协议处理接口 **ProtocolHandler**，它是 Coyote 连接器模块的首要协议接口。

&emsp;&emsp;首先，ProtocolHandler 定义了一个关联此协议处理器的适配器 **Adapter** 接口，该接口扮演的是桥梁的角色，Adapter 将 ProtocolHandler 接受到的连接请求传递给后方容器组件。目前唯一的实现类 **CoyoteAdapter** 将客户端不同的协议请求最终适配成 Servlet 请求后交给 Catalina Servlet 容器处理。

&emsp;&emsp;其次， ProtocolHandler 接口中定义了协议本身的生命周期方法，并提供获取其实现类中关联的线程池的方法定义。

&emsp;&emsp;最后，ProtocolHandler 接口的抽象实现类 AbstractProtocol 则对连接协议处理有了更具体的规范定义，它将协议的处理过程拆分成两个过程，一是用来处理客户端 Socket 连接的抽象类 **AbstractEndpoint**，二是用来真正处理连接具体协议的接口类 **Processor**。可以认为连接端点 AbstractEndpoint 用来接受客户端的 TCP/UDP 连接，将其封装为 Java Socket 连接对象，交由 **Processor** 进行具体的通信协议的解析处理，目前 Tomcat 支持 HTTP 1.1、HTTP 2、AJP 三种协议。Coyote 抽象协议层的组织结构如下图所示。

![Coyote Abstraction](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_coyote_abstraction.png)

### Catalina Servlet 容器

&emsp;&emsp;再来看结构图由此的 Catalina Servlet 容器，它是一个提供 Servlet 全生命周期管的 Servlet 容器。它由具备上下级关系的四层容器组成，由父容器到子容器的顺序分别为 Engine、Host、Context、Wrapper。在单个 Service 模块下，有且仅有一个 Servlet Engine 容器，而它能被多个 Coyote 连接器组件所关联。Engine 容器可以由多个 Host 子容器实例（对应不同的虚拟主机地址），单个 Host 又能包含多个 Context 子容器实例（对应不同的 Web 应用），单个 Context 下包含多个 Wrapper 子容器实例（对应不同的 Servlet）。

&emsp;&emsp;客户端请求经过连接器处理后，由 CoyoteAdapter 适配转换成 Servlet 请求提交给 Engine 容器，从 Engine 容器开始请求经过被称作 **Pipeline 管道式**的容器间的流转处理最终到达底层容器 Wrapper。所谓管道式的容器间流转，就好比现实生活中的自来水管与手龙头：当请求到达某个容器，它将沿着类似自来水管（Pipeline）的单一方向依次流到其中设置的每个水龙头（Valve）进行处理，而每个容器最后一个 Valve 具备将请求流转到其下级容器的 Pipeline 的功能，从而使得请求自上而下进入最底层容器 Wrapper。Wrapper 的最后一个 Valve 为 **StandardWrapperValve** 将生成 Servlet 的过滤器请求链 **FilterChain** 实例，请求经过这条责任链处理后最终由所映射的 Servlet 处理。值的注意的是，Servlet 请求在进入过滤器链时已经封装为 **RequestFacade** 门面类对象。

### 组件生命周期

&emsp;&emsp;Tomcat 中的组件基本都接受生命周期的管理，它们基本都遵照生命周期根接口 **Lifecycle** 定义的方法进行初始化、启动、停止及销毁。

![Lifecycle Interface](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_lifecycle.png)

&emsp;&emsp;从类图可以看出，生命周期方法主要包括：`init()` 初始化、`start()` 启动、`stop()` 停止、`destroy()` 销毁 四个方法步骤。在此四个方法流转过程中会伴随着组件的状态变化，故而提供了获取状态的方法 `getState()`，而状态的变化可以传递给通过监听器注册方法 `addLifecycleListener(LifecyleListener)` 所添加的监听器进行**生命周期事件**处理。在接口 Lifecycle 类注释中详细说明了生命周期方法流转过程中对应的状态变化：

```java
/**
 *            start()
 *  -----------------------------
 *  |                           |
 *  | init()                    |
 * NEW -»-- INITIALIZING        |
 * | |           |              |     ------------------«-----------------------
 * | |           |auto          |     |                                        |
 * | |          \|/    start() \|/   \|/     auto          auto         stop() |
 * | |      INITIALIZED --»-- STARTING_PREP --»- STARTING --»- STARTED --»---  |
 * | |         |                                                            |  |
 * | |destroy()|                                                            |  |
 * | --»-----«--    ------------------------«--------------------------------  ^
 * |     |          |                                                          |
 * |     |         \|/          auto                 auto              start() |
 * |     |     STOPPING_PREP ----»---- STOPPING ------»----- STOPPED -----»-----
 * |    \|/                               ^                     |  ^
 * |     |               stop()           |                     |  |
 * |     |       --------------------------                     |  |
 * |     |       |                                              |  |
 * |     |       |    destroy()                       destroy() |  |
 * |     |    FAILED ----»------ DESTROYING ---«-----------------  |
 * |     |                        ^     |                          |
 * |     |     destroy()          |     |auto                      |
 * |     --------»-----------------    \|/                         |
 * |                                 DESTROYED                     |
 * |                                                               |
 * |                            stop()                             |
 * ----»-----------------------------»------------------------------
 */
```

&emsp;&emsp;继续观察生命周期接口的继承关系：

![Lifecycle Hierachy](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_lifecycle_hierarchy.png)

&emsp;&emsp;可以看出接口 Lifecycle 包含两个抽象实现类 **LifecycleBase**、**LifecycleMBeanBase**，前者对 Lifecycle 的生命周期方法做了模板代码实现，**同时暴露出以** ***internal*** **结尾的受保护的生命周期方法供子类订制实现**。后者在前者基础上提供默认的定制化实现的初始化与销毁方法，来达到对 JMX 的组件注册与取消注册功能。**Container** 子接口则是保留原声明周期方法定义的同时引入容器的功能，它的子类不单单具备生命周期而且还是一个容器，可以支持添加、删除、查找子容器等功能，它的抽象子类实现 **ContainerBase** 提供了容器功能的模板方法实现，同时定义了一些抽象的方法给子类具体实现，该类是 Catalina Servlet 容器组件的共同父类。值得注意的是 **Container** 接口也引入了监听器注册的方法，只不是它所注册的监听器属于 **ContainerListener** 类型，有别于生命周期的监听器 **LifecycleListener**，前者关注容器的管理事件，后者关注生命周期状态变化事件。

## Tomcat 服务启动

&emsp;&emsp;前面已经了解 Tomcat 的组要核心模块，并且知道这些模块是由一些具备生命周期的组件所组成。本章着重来介绍这些模块组件是如何依次完成自身的启动过程。Tomcat 可以作为独立式服务器启动，同时也支持以内嵌式服务器的方式启动，例如：Tomcat 可以作为 Spring Boot 内嵌 Web 服务器启动，这会在将来的文章中详细介绍。本章主要是介绍 Tomcat 以独立式的服务器的启动。

&emsp;&emsp;作为独立式的服务器启动时，Tomcat 设计了一个 Bootstrap 启动引导器，它包含 Tomcat 主程序运行 `main` 入口方法，它将实例化一个可以提供启动、关闭的交互 Shell 程序类 Catalina，最终由后者执行启动过程。下面的时序图描述的是 Tomcat 启动过程中一些重要组件的启动次序，同时也能了解组件间的交互关系。

<a herf="https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_start_abstraction.svg" target="_blank">![Tomcat Start Abstraction](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_start_abstraction.svg)</a>

&emsp;&emsp;从上图可知，整个 Tomcat 的启动过程的分为装载（load）、启动（start）两个阶段，下面来具体介绍这两阶段中包含的一些重要操作。

### load 装载阶段

&emsp;&emsp;Bootstrap 的 `main` 方法执行时，会在初始化过程中利用反射创建 Catalina 实例对象 `catalinaDaemon`，之后在启动命令中先后执行 `load` 和 `start` 方法。在加载方法中，将反射调用 `catalinaDaemon` 实例的 `load` 方法。整个 Catalina 装载过程中最重要的操作是解析 Tomcat 的配置文件 `conf/server.xml` 生成上文中介绍的组件实例对象，为之后的启动做准备。下面是 Catalina 类的 `load` 方法（代码有省略）：

```java
public class Catalina {
    public void load() {
        // 省略...     
        // 创建配置文件解析器对象 digester
        Digester digester = createStartDigester();

        InputSource inputSource = null;
        InputStream inputStream = null;
        File file = null;
        try {
            try {
                // 加载 conf/server.xml 配置文件
                file = configFile();
                inputStream = new FileInputStream(file);
                inputSource = new InputSource(file.toURI().toURL().toString());
            } catch (Exception e) {
            }
            
            try {
                inputSource.setByteStream(inputStream);
                // 将 Catalina 实例放入 digester 对象栈栈顶
                digester.push(this);
                // 开始解析配置文件并生成组件对象
                digester.parse(inputSource);
            } catch (SAXParseException spe) {
                return;
            }
        } finally {
        }

        getServer().setCatalina(this);
        getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
        getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

        // 调用 Server 的初始化方法
        try {
            getServer().init();
        } catch (LifecycleException e) {
        }
    }
}
```

&emsp;&emsp;总体逻辑是创建配置文件解析器 Digester，然后加载配置文件，然后使用解析器对其进行解析生成组件对象，然后获得顶层组件 **Server** 实例，开始调用其生命周期初始化方法 `init` 从而联动整个容器的组件的初始化。Digester 继承至 SAX2 的事件处理器 **DefaultHandler2** ，将 XML 解析事件转化抽象为基于 Rule 规则的处理方式。

![Digester Abstraction](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_digester_abstraction.png)

&emsp;&emsp;Digester 持有一个规则集合 Rules 的实现类 RulesBase，由它管理众多的抽象 Rule 规则的具体实现类。Tomcat 针对配置文件设计出了很多规则类，使得解析过程中的一些重复的模版代码能够被很好的复用，例如：解析到某个元素时，需要创建指定的类实例的规则类 **ObjectCreateRule** 等等。下面是构建用来解析 `server.xml` 的解析器，可以看出里面添加了大量不同的规则处理类来生成 Tomcat 的组件，并且组织它们之间的关系。

```java
public class Catalina {
    protected Digester createStartDigester() {
        long t1=System.currentTimeMillis();
        // 省略...
        // 添加创建 Server 组件的规则
        digester.addObjectCreate("Server",
                                 "org.apache.catalina.core.StandardServer",
                                 "className");
        // 添加解析 Server 元素属性并填充 Server 组件属性的规则
        digester.addSetProperties("Server");
        // 添加调用指定方法的规则
        digester.addSetNext("Server",
                            "setServer",
                            "org.apache.catalina.Server");
        // 省略类似添加规则的方法...
        return digester;
    }
}
```

&emsp;&emsp;设置好这些解析规则之后，执行 `digester.parse(inputSource)` 对符合的元素执行对应规则的实现代码，即可一一实例化组件，并且组织它们的结构层次及依赖关系。这其中就包含关键组件 Server、Service、Mapper、Connector 等关键组件的实例化，解析完毕后，最终会将顶级组件 Server 实例由 **SetNextRule** 规则调用 Catalina 实例的 `setServer` 方法填充 Catalina 实例的 `server` 属性，随之调用 Server 实例的 `init()` 初始化方法，对一些关键组件开始生命周期初始化操作（可以看出容器组件只完成了 Engine 的初始化，子容器初始化被推迟到启动阶段），涉及组件的初始化完毕后，整个加载阶段也就执行完毕，随之转入下一小节的启动阶段。

### start 启动阶段

&emsp;&emsp;启动阶段包含两个重要启动流程： **Catalina Servlet 容器**和 **Coyote 连接器**的启动。容器的启动过程已经在 [***Servlet 容器启动***](https://reionchan.github.io/2023/03/19/servlet-container-tomcat/#servlet-%E5%AE%B9%E5%99%A8%E5%90%AF%E5%8A%A8) 小节有过详细的讲解，至于连接器的启动过程由于涉及不同协议的连接器实现不同，这里只针对将选取 **Http11NioProtocol** 这个以 HTTP1.1 协议的 NIO 实现为例来讲解对抽象层启动的具体实现过程。从图中可以看出，连接器模块的启动是在容器启动完成后，所有映射关系都已收集完毕后开始启动。

&emsp;&emsp;首先，连接器组件 **Connector** 由管理它的 **Service** 的启动方法联动启动，而它的启动方法中又联动它关联的 **ProtocolHandler** 组件的启动。

```java
public class Connector extends LifecycleMBeanBase  {
    protected void startInternal() throws LifecycleException {

        // ...
        setState(LifecycleState.STARTING);

        try {
            // ProtocolHandler 组件启动
            protocolHandler.start();
        } catch (Exception e) {
            throw new LifecycleException(
                    sm.getString("coyoteConnector.protocolHandlerStartFailed"), e);
        }
    }
}
```

&emsp;&emsp;其次，**ProtocolHandler** 组件抽象类 **AbstractProtocol** 启动方法中又联动启动与它关联的连接端点 **AbstractEndpoint**，而后者的实现类就是具体某个协议 Socket 连接的服务端的实际连接点。

```java
public abstract class AbstractProtocol<S> implements ProtocolHandler, MBeanRegistration {
    
    private final AbstractEndpoint<S,?> endpoint;
    
    @Override
    public void start() throws Exception {
        // 连接点组件启动
        endpoint.start();
        // ...
    }
}
```

&emsp;&emsp;最后，就是抽象类 AbstractEndpoint 的具体协议实现子类的启动流程了，**ProtocolHandler** 组件抽象类 **AbstractProtocol** 在 HTTP1.1 协议下默认采用的是 NIO 连接模型的 **Http11NioProtocol** 实现类，而它关联的连接点实现类为 **NioEndpoint** ，它启动时就将启动 Poller 、Acceptor 线程进行非阻塞式的接受客户端的 Socket 连接。

```java
public class NioEndpoint extends AbstractJsseEndpoint<NioChannel,SocketChannel> {
    public void startInternal() throws Exception {

        if (!running) {
            // 省略...
            // 创建工作线程池
            if (getExecutor() == null) {
                createExecutor();
            }

            initializeConnectionLatch();

            // 启动 poller 线程
            poller = new Poller();
            Thread pollerThread = new Thread(poller, getName() + "-Poller");
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();

            // 启动服务端 Socket 连接监听线程     
            startAcceptorThread();
        }
    }
}
```

&emsp;&emsp;上面是代码的视角观察启动过程，下面的时序图描述了连接器 **Connector** 启动 **Http11NioProtocol** 的详细过程：

<a herf="https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_http11_nio_protocol_process.svg" target="_blank">![Http11NioProtocol Init&Start Process](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_http11_nio_protocol_process.svg)</a>

&emsp;&emsp;可以看出， **NioEndpoint** 的端口绑定操作原本是在初始化阶段执行，而默认被延迟到启动阶段才开始端口绑定操作。值得注意的是 Tomcat 在 8.5 小版本优化后，将之前生成多个 **Poller**、**Acceptor** 线程修改为两者都是单线程，同时删除了 **NioSelectorPool**、**NioBlockingSelector**、**BlockPoller**。

## Tomcat 请求处理

&emsp;&emsp;上一章节介绍了 Tomcat 启动的总体过程，并以 HTTP1.1 协议下默认采用的 NIO 连接模型的 **Http11NioProtocol** 实现类来进一步说明服务端连接器的 Socket 连接启动过程。本章将讨论客户端发来请求后 **Poller**、**Acceptor** 线程是如何处理 Socket 连接，并将其解析为 Coyote 模块下的 Request 对象，然后在 Connector 的桥梁作用下进一步转换成 Servlet Request 对象，最终交给 Servlet 处理的过程。

### 请求处理抽象

&emsp;&emsp;俗话说：“打蛇打七寸”，Tomcat 设计上对客户端请求与服务器端处理进行了一层抽象，这层抽象已经定义好请求流转处理的模版路径规范。来自客户端的不同协议、不同的连接模式只是对这层抽象路径的完善，从而能够处理具体协议的请求。下面通过类图的形式展现处理流程的抽象：

<a herf="https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_process_connection_abstraction.svg" target="_blank">![Process Connection Abstraction](https://raw.githubusercontent.com/ReionChan/tomcat/8.5.x/uml/ref/tomcat8.5_process_connection_abstraction.svg)</a>

&emsp;&emsp;客户端请求从与服务端建立 Socket 连接，到最终交给 Servlet 容器处理大致分为七个步骤：

1. Acceptor 接受来自客户端的 Socket 连接，将连接封装成 SocketWrapperBase
2. AbstractEndpoint 创建 SocketProcessorBase 处理 SocketWrapperBase
3. SocketProcessorBase 将 SocketWrapperBase 转交给 AbstractProtocol.ConnectionHandler 处理
4. AbstractProtocol.ConnectionHandler 创建 Processor 接口实例对象
5. 由 Processor 接口实例对象解析 SocketWrapperBase，生成 Coyote 下的统一 Request 对象
6. 由 Processor 接口实例对象将 Request 对象转交给关联的 Adapter 对象处理
7. Adapter 对象将 Coyote 下的 Request 适配转换成 Catalina Servlet 容器的 Request 并交给关联连接器所在的 Engine 容器进行处理

&emsp;&emsp;不难发现，步骤 1 ~ 6 为总体架构设计图中的 Coyote 连接器模块的处理过程，而第 7 步就借由 Connector 桥梁、经由 CoyoteAdapter 将请求交给 Engine 容器的 Pipeline 进行责任链式的处理。

### 请求处理实现

&emsp;&emsp;现在以  HTTP1.1 协议下默认采用的 NIO 连接模型的 **Http11NioProtocol** 实现类处理客户端的 HTTP 请求来演示 Http11NioProtocol 对上面七个抽象步骤的具体实现。

1. Acceptor 接受来自客户端的 Socket 连接，将连接封装成 SocketWrapperBase

   ```java
   public class Acceptor<U> implements Runnable {
   
       private final AbstractEndpoint<?,U> endpoint;
   
       // Http11NioProtocol 关联的 AbstractEndpoint 实现类为 NioEndpoint
       public Acceptor(AbstractEndpoint<?,U> endpoint) {
           this.endpoint = endpoint;
       }
   
       @SuppressWarnings("deprecation")
       @Override
       public void run() {
   
           try {
               while (!stopCalled) {
                   // 省略
                   while (endpoint.isPaused() && !stopCalled) {
                      
                   try {
                       // 省略
                       try {
                           // 接受来自客户端的 Socket 连接
                           socket = endpoint.serverSocketAccept();
                       } catch (Exception ioe) {
                       }
   
                       // Configure the socket
                       if (!stopCalled && !endpoint.isPaused()) {
                           // setSocketOptions 将 socket 交给 NioEndpoint 处理
                           if (!endpoint.setSocketOptions(socket)) {
                               endpoint.closeSocket(socket);
                           }
                       } else {
                           endpoint.destroySocket(socket);
                       }
                   } catch (Throwable t) {
                   }
               }
           } finally {
               stopLatch.countDown();
           }
           state = AcceptorState.ENDED;
       }
   }
   ```

   ```java
   public class NioEndpoint extends AbstractJsseEndpoint<NioChannel,SocketChannel> {
       // NioEndpoint 关联的轮询器     
       private Poller poller = null;
       
       @Override
       protected boolean setSocketOptions(SocketChannel socket) {
           // Socket 封装成 SocketWrapperBase 的子类 NioSocketWrapper     
           NioSocketWrapper socketWrapper = null;
           try {
               NioChannel channel = null;
               if (nioChannels != null) {
                   channel = nioChannels.pop();
               }
               // 省略
               // 构造 NioSocketWrapper
               NioSocketWrapper newWrapper = new NioSocketWrapper(channel, this);
               channel.reset(socket, newWrapper);
               connections.put(socket, newWrapper);
               socketWrapper = newWrapper;
               socket.configureBlocking(false);
               socketProperties.setProperties(socket.socket());
   
               socketWrapper.setReadTimeout(getConnectionTimeout());
               socketWrapper.setWriteTimeout(getConnectionTimeout());
               socketWrapper.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
               // 将 NioSocketWrapper 注册给轮询器线程处理      
               poller.register(socketWrapper);
               return true;
           } catch (Throwable t) {
               ExceptionUtils.handleThrowable(t);
               try {
                   log.error(sm.getString("endpoint.socketOptionsError"), t);
               } catch (Throwable tt) {
                   ExceptionUtils.handleThrowable(tt);
               }
               if (socketWrapper == null) {
                   destroySocket(socket);
               }
           }
           // Tell to close the socket if needed
           return false;
       }
   }
   ```

   &emsp;&emsp;可以看出，Socket 到 SocketWrapperBase 封装是 Http11NioProtocol 这个具体协议绑定的 NioEndpoint 处理，封装后的为 SocketWrapperBase 抽象类的具体实现类 **NioSocketWrapper**，完成步骤 1 的具体实现。

   

2. AbstractEndpoint 创建 SocketProcessorBase 处理 SocketWrapperBase

   &emsp;&emsp;AbstractEndpoint 在 Http11NioProtocol 协议下对应的实现类 NioEndpoint，来看它创建 SocketProcessorBase 处理器的具体实现。

   ```java
   public class NioEndpoint extends AbstractJsseEndpoint<NioChannel,SocketChannel> {
       @Override
       protected SocketProcessorBase<NioChannel> createSocketProcessor(
               SocketWrapperBase<NioChannel> socketWrapper, SocketEvent event) {
           // 创建 SocketProcessorBase 类的具体子类 SocketProcessor 
           return new SocketProcessor(socketWrapper, event);
       }
   }
   ```

   

3. SocketProcessorBase 将 SocketWrapperBase 转交给 AbstractProtocol.ConnectionHandler 处理

   &emsp;&emsp;上一步知晓 SocketProcessorBase 在此处创建的实际子类为 SocketProcessor，它是 NioEndpoint 的内部类。下面看其将 SocketWrapperBase 的实际子类 NioSocketWrapper 交给 AbstractProtocol.ConnectionHandler 处理。

   ```java
       protected class SocketProcessor extends SocketProcessorBase<NioChannel> {
   
           @Override
           protected void doRun() {
               // Do not cache and re-use the value of socketWrapper.getSocket() in
               Poller poller = NioEndpoint.this.poller;
               if (poller == null) {
                   socketWrapper.close();
                   return;
               }
   
               try {
                   int handshake = -1;
                   
                   if (handshake == 0) {
                       SocketState state = SocketState.OPEN;
                       // 将 NioSocketWrapper 交给 AbstractProtocol.ConnectionHandler 处理
                       if (event == null) {
                           state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
                       } else {
                           state = getHandler().process(socketWrapper, event);
                       }
                       if (state == SocketState.CLOSED) {
                           poller.cancelledKey(getSelectionKey(), socketWrapper);
                       }
                   } else if (handshake == -1 ) {
                       getHandler().process(socketWrapper, SocketEvent.CONNECT_FAIL);
                       poller.cancelledKey(getSelectionKey(), socketWrapper);
                   } else if (handshake == SelectionKey.OP_READ){
                       socketWrapper.registerReadInterest();
                   } else if (handshake == SelectionKey.OP_WRITE){
                       socketWrapper.registerWriteInterest();
                   }
               } catch (CancelledKeyException cx) {
               } finally {
               }
           }
       }
   
   ```

   

4. AbstractProtocol.ConnectionHandler 创建 Processor 接口实例对象

   ```java
       protected static class ConnectionHandler<S> implements AbstractEndpoint.Handler<S> {
   
           private final AbstractProtocol<S> proto;
           // 此处关联的 AbstractProtocol 实现类 Http11NioProtocol
           public ConnectionHandler(AbstractProtocol<S> proto) {
               this.proto = proto;
           }
   
           @Override
           public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {
               
               S socket = wrapper.getSocket();
               Processor processor = (Processor) wrapper.takeCurrentProcessor();
   
               // 省略
               try {            
                   if (processor == null) {
                       // Http11NioProtocol 创建 Processor 接口具体实现类 Http11Processor
                       processor = getProtocol().createProcessor();
                       register(processor);
                   }
                   SocketState state = SocketState.CLOSED;
                   do {
                       // 委托 Processor 接口具体实现类处理 NioSocketWrapper                
                       state = processor.process(wrapper, status);
                   } while ( state == SocketState.UPGRADING);
                   
                   if (processor != null) {
                       wrapper.setCurrentProcessor(processor);
                   }
                   return state;
               } catch(java.net.SocketException e) {
               }
   
               release(processor);
               return SocketState.CLOSED;
           }
       }
   ```

   

5. 由 Processor 接口实例对象解析 SocketWrapperBase，生成 Coyote 下的统一 Request 对象

   &emsp;&emsp;Processor 接口具体实现类 Http11Processor 解析  NioSocketWrapper 实际发生在 Http11Processor 父类 AbstractProcessorLight 的 `process` 方法中。

   ```java
   public abstract class AbstractProcessorLight implements Processor {
       @Override
       public SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status)
               throws IOException {
   
           SocketState state = SocketState.CLOSED;
           Iterator<DispatchType> dispatches = null;
           do {
               if (dispatches != null) {
                   DispatchType nextDispatch = dispatches.next();
                   state = dispatch(nextDispatch.getSocketStatus());
               } else if (status == SocketEvent.DISCONNECT) {
               } else if (isAsync() || isUpgrade() || state == SocketState.ASYNC_END) {
                   state = dispatch(status);
                   state = checkForPipelinedData(state, socketWrapper);
               } else if (status == SocketEvent.OPEN_WRITE) {
                   state = SocketState.LONG;
               } else if (status == SocketEvent.OPEN_READ) {
                   // 交由抽象方法 service 给子类 Http11Processor 处理 HTTP 协议               
                   state = service(socketWrapper);
               } else if (status == SocketEvent.CONNECT_FAIL) {
                   logAccess(socketWrapper);
               } else {
                   state = SocketState.CLOSED;
               }
           } while (state == SocketState.ASYNC_END ||
                   dispatches != null && state != SocketState.CLOSED);
           return state;
       }
       
       // 抽象方法 service  
       protected abstract SocketState service(SocketWrapperBase<?> socketWrapper) throws IOException;    
   }
   ```

   &emsp;&emsp; Http11Processor 解析 NioSocketWrapper HTTP 请求，并生成 Coyote 的 Request：

   ```java
   public class Http11Processor extends AbstractProcessor {
       @Override
       public SocketState service(SocketWrapperBase<?> socketWrapper)
           throws IOException {
   
           // 省略
           setSocketWrapper(socketWrapper);
   
           while (!getErrorState().isError() && keepAlive && !isAsync() && upgradeToken == null &&
                   sendfileState == SendfileState.DONE && !endpoint.isPaused()) {
               try {
                   // 解析请求行 METHOD URI PROTOCOL                
                   if (!inputBuffer.parseRequestLine(keptAlive)) {
                   }
               } catch (IOException e) {
               }
   
               // Process the request in the adapter
               if (getErrorState().isIoAllowed()) {
                   try {
                       // 将构造好的 Request Response 交给 CoyoteAdapter 处理                    
                       getAdapter().service(request, response);
                   } catch (InterruptedIOException e) {
                   }
               }
           }
       }
   }
   ```

    

6. 由 Processor 接口实例对象将 Request 对象转交给关联的 Adapter 对象处理

   &emsp;&emsp;此步骤即为上一步最后代码片段中的这段代码 `getAdapter().service(request, response)`，其中 `getAdapter` 获取的 Adapter 接口实例为 CoyoteAdapter。

   

7. Adapter 对象将 Coyote 下的 Request 适配转换成 Catalina Servlet 容器的 Request 并交给关联连接器所在的 Engine 容器进行处理

   &emsp;&emsp;CoyoteAdapter 的 `service` 方法

   ```java
   public class CoyoteAdapter implements Adapter {
       @Override
       public void service(org.apache.coyote.Request req, org.apache.coyote.Response res)
               throws Exception {
           // 此处将 Coyote 的 Request 获取 Catalina 下的 Request
           Request request = (Request) req.getNote(ADAPTER_NOTES);
           Response response = (Response) res.getNote(ADAPTER_NOTES);
   
           if (request == null) {
               // 创建 Catalina 下的 Request
               request = connector.createRequest();
               request.setCoyoteRequest(req);
               response = connector.createResponse();
               response.setCoyoteResponse(res);
           }
   
           try {
               // 解析请求行，转换成 Catalina request 请求类型并设置相关属性，其中包含映射数据 MappingData
               postParseSuccess = postParseRequest(req, request, res, response);
               if (postParseSuccess) {
                   // 将请求委托给关联的连接器对应的容器 Engine 的 Pipeline 处理
                   connector.getService().getContainer().getPipeline().getFirst().invoke(
                           request, response);
               }
               
           } catch (IOException e) {
               // Ignore
           } finally {
           }
       }
   }
   ```

   &emsp;&emsp;到此，请求就递交给 Servlet 容器 Engine 处理，之后的处理过程就如上面 [***Catalina Servlet 容器***](https://reionchan.github.io/2023/03/19/servlet-container-tomcat/#catalina-servlet-%E5%AE%B9%E5%99%A8) 小节所描述的那样最终交给映射的 Servlet 处理。 

## 推荐阅读

## 参考资料

* [Apache Tomcat 8 Architecture](https://tomcat.apache.org/tomcat-8.5-doc/architecture/overview.html)

## 脚注

[^1]: 详情参阅： <a href='https://stackoverflow.com/questions/32985051/what-are-the-tomcat-component-what-is-catalina-and-coyote'>what-is-catalina-and-coyote</a> 。

