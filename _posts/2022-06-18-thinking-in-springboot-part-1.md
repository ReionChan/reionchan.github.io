---
layout: post
title: 《Spring Boot 编程思想（核心篇）》上篇
categories: Book Spring
tags: SpringBoot Spring Boot 
excerpt: 读该书 “第 1 部分 —— 总览 Spring Boot” 心得笔记 
image: https://www.image.com
description: SpringBoot 一些关键特性的原理及设计思想
keywords: Spring SpringBoot Thinking 编程思想 六大特性 总览 Spring Boot 理解规约大于配置
licences: cc
---
<br/>

<img src="https://spring.io/images/spring-logo-9146a4d3298760c2e7e49595184e1975.svg" alt="Spring Logo" width="800"/>

---
> &emsp;&emsp;Spring Boot 具有一套固化的视图，该视图用于构建生产级别的应用。
<div style="text-align: right;"> —— 摘自 <a href="https://spring.io/projects/spring-boot/">Spring 官网</a> &emsp;&emsp;</div>

## 初览 Spring Boot

### Spring Framework 时代

### Spring Boot 简介

> Spring Boot takes an opinionated view of building production reday applications.<br>
Spring Boot 具有一套固化的视图，该视图用于构建生产级别的应用

### Spring Boot 六大特性

- 创建独立的 Spring 应用
- 直接嵌入 Tomcat、Jetty 或 Undertow 等 Web 容器，无需部署 WAR 文件
- 提供固化的 **starter** 依赖，简化构建配置
- 当条件满足时自动装配 Spring 或第三方类库
- 提供运维（Production-Ready）特性，如指标信息、健康检查及外部配置
- 绝无代码生成，且无需 XML 配置

* 准备运行环境


## 理解独立的 Spring 应用

### 创建 Spring Boot 应用

### 运行 Spring Boot 应用

- 可执行 Jar 资源结构
- Fat Jar/War 执行模块 —— spring-boot-loader
- JarLauncher 实现原理

	* 注册自定义协议处理器 *JarFile.registerUrlProtocolHandler()*
	
		&emsp;&emsp;利用 *java.net.URLStreamHandler* **扩展机制**，即在 Java 系统属性 `java.protocal.handler.pkgs` 追加注册 Spring 自定义协议处理器所在的包名 *org.springframework.boot.loader*，从而使得 *URL.getURLStreamHandler(String)* 获得 Spring 的处理器 *org.springframework.boot.loader.jar.Handler* 进行 Fat Jar 的资源加载及运行。
		
		&emsp;&emsp;与 JarLauncher 类似，WarLauncher 对应处理器与前者相同，只是在包中的 `BOOT-INF` 变更为 `WEB-INF`, 同时为了兼顾外部 Web 容器的部署时依赖包冲突，新加入 `WEB-INF/lib-provided` 目录，把在 POM 文件中的标识 `<scope>provided</scope>` 的依赖包放入其中，该目录的包只在内嵌 Web 容器时生效，例如：Servlet API 包即可声明为 provided 放入该目录，当外部部署时的 Web 容器已经提供此包，从而防止引入冲突。
		

## 理解固化的 Maven 依赖
&emsp;&emsp;本小节设计 Maven 工具的相关说明，想了解 Maven 的基本使用，请参考：[Maven Basics](https://www.baeldung.com/tag/maven-basics/)

### spring-boot-starter-parent 与 spring-boot-dependencies 简介

- spring-boot-starter-parent

	&emsp;&emsp;该 starter 有别与一般的 Spring Boot Starter 项目，它是专门用来管理 Spring Boot 依赖、默认配置一些插件，如：maven-jar-plugin 等等。开发人员通过将其指定为父 POM 能快速构建 Spring Boot 应用程序。指定方式为在项目的 `pom.xml` 文件增加如下声明：
	
	```xml
	<parent>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-parent</artifactId>
	    <version>2.2.13</version>
	</parent>
	```
	
	&emsp;&emsp;它的打包方式为 [POM（Project Object Model）](https://www.baeldung.com/maven-super-simplest-effective-pom)，**它的父 POM 为 spring-boot-dependencies**。
	
- spring-boot-dependencies

	&emsp;&emsp;由上可知，它是 **spring-boot-starter-parent** 的父 POM，是完成**管理 Spring Boot 依赖**、**默认配置一些插件**的真正 POM，当 Spring Boot 项目 POM 有自定义的父 POM 时，无法通过指定 **spring-boot-starter-parent** 为父 POM 时，可以直接在项目 `pom.xml` 文件中引入这个依赖实现  Spring Boot 依赖管理及默认配置必要插件：
	
	```xml
	<dependencyManagement>
	    <dependencies>
	        <dependency>
	            <groupId>org.springframework.boot</groupId>
	            <artifactId>spring-boot-dependencies</artifactId>
	            <version>2.2.13.RELEASE</version>
	            <type>pom</type>
	            <scope>import</scope>
	        </dependency>
	    </dependencies>
	</dependencyManagement>
	```

### 理解 spring-boot-starter-parent 与 spring-boot-dependencies

- `spring-boot-parent` 具体内容，请查阅 [版本 2.2.13.RELEASE](https://search.maven.org/artifact/org.springframework.boot/spring-boot-starter-parent/2.2.13.RELEASE/pom)，可以看出其父 POM 声明：

	```xml
	  <!-- 注意：此处为父 POM 的声明 -->
	  <parent>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-dependencies</artifactId>
	    <version>2.2.13.RELEASE</version>
	    <relativePath>../spring-boot-dependencies</relativePath>
	  </parent>
	```
- `spring-boot-dependencies` 具体内容，请查阅 [版本 2.2.13.RELEASE](https://search.maven.org/artifact/org.springframework.boot/spring-boot-dependencies/2.2.13.RELEASE/pom)

## 理解嵌入式 Web 容器

Servlet 规范	|	Tomcat	|	Jetty	|	Undertow
:---:			|	:---:		|	:---:		|	:---: 
4.0			|	9.x 		|	10.x	| 	2.x
3.1			|	8.x 		|	9.x	| 	1.x
3.0			|	7.x 		|	8.x		| 	N/A
2.5			|	6.x 		|	6.x		| 	N/A

&emsp;&emsp;Spring Boot 1.0 版本应用支持的直接嵌入的 Web 容器有：Tomcat、Jetty 和 Undertow，Spring Boot 2.0 版本则开始支持嵌入式 Reactive Web 容器，默认实现为 Netty Web Server。不过由于 Spring WebFlux 基于 Reactor 框架实现，因此在 Spring Boot 中，Netty Web Server 属于 Reactor 与 Netty 的整合实现，由 `spring-boot-starter-webflux` 的依赖 `spring-boot-starter-reactor-netty` 间接引入。而前面表格提及的三个嵌入式 Servlet 容器也能作为 Reactive Web Server，并允许替换默认实现 Netty Web Server。这是因为支持 Servlet 3.1+ 规范的容器也开始支持 Reactive 异步非阻塞的特性。

### 嵌入式 Servlet Web 容器

&emsp;&emsp;Spring Boot 通过引入 `spring-boot-starter-web` 依赖来支持 Servlet Web 项目，默认的嵌入式 Servlet Web 容器为 Tomcat，可以通过 Maven 依赖排除方式移除默认容器，另外引入其他 Servlet Web 容器。

- Tomcat 作为嵌入式 Servlet Web 容器

	```xml
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	```
	
- Jetty 作为嵌入式 Servlet Web 容器

	```xml
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
		<exclusions>
			<!-- 剔除默认 Tomcat 依赖 -->
			<exclusion>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-starter-tomcat</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<!-- 引入 Jetty 依赖 -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-jetty</artifactId>
	</dependency>
	```
	
- **UndertowServletWebServer** 作为嵌入式 Servlet Web 容器

	```xml
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
		<exclusions>
			<!-- 剔除默认 Tomcat 依赖 -->
			<exclusion>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-starter-tomcat</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<!-- 引入 Undertow 依赖 -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-undertow</artifactId>
	</dependency>
	```

### 嵌入式 Reactive Web 容器

&emsp;&emsp;Spring Boot 2.0 开始支持嵌入式 Reactive Web 容器，但其处于被动激活状态，需要增加 `spring-boot-starter-webflux` 依赖，并且依赖中不能出现 `spring-boot-starter-web` 依赖。**一旦两者同时存在时， `spring-boot-starter-webflux` 会被忽略，导致项目还是使用嵌入式 Servlet Web 容器，而非 Reactive Web 容器。**这个机制是由 *SpringApplication* 实现中的 Web 应用类型（*WebApplicationType*）推断逻辑决定的。

&emsp;&emsp;本节开头提及：

> Spring Boot 2.0 版本则开始支持嵌入式 Reactive Web 容器，默认实现为 Netty Web Server。不过由于 Spring WebFlux 基于 Reactor 框架实现，因此在 Spring Boot 中，Netty Web Server 属于 Reactor 与 Netty 的整合实现，由 `spring-boot-starter-webflux` 的依赖 `spring-boot-starter-reactor-netty` 间接引入。

&emsp;&emsp;换言之，当 Reactive 型 Web 应用引入 `spring-boot-starter-webflux` 依赖时，默认会提供一个基于 Reactor 与 Netty 整合实现的 Netty Web Server，进一步查询 `spring-boot-autoconfigure` 中关于 Reactive Web Server 的自动装配配置类 [*ReactiveWebServerFactoryConfiguration*](https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/reactive/ReactiveWebServerFactoryConfiguration.java) 还能发现，**类资源路径中一旦发现其他嵌入式 Web 容器，此默认的 Netty Web Server 将被替换。**

```java
// 此处是默认 Netty 容器工厂类自动配置
@Configuration
@ConditionalOnMissingBean(ReactiveWebServerFactory.class)
@ConditionalOnClass({ HttpServer.class })
static class EmbeddedNetty {

	@Bean
	public NettyReactiveWebServerFactory nettyReactiveWebServerFactory() {
		return new NettyReactiveWebServerFactory();
	}

}

// 此处是默认 Tomcat 容器工厂类自动配置
@Configuration
@ConditionalOnMissingBean(ReactiveWebServerFactory.class)
@ConditionalOnClass({ org.apache.catalina.startup.Tomcat.class })
static class EmbeddedTomcat {

	@Bean
	public TomcatReactiveWebServerFactory tomcatReactiveWebServerFactory() {
		return new TomcatReactiveWebServerFactory();
	}

}

// Jetty 及 Undertow 的自动配置雷同，此处省略
```
&emsp;&emsp;可以看出不同容器的工厂是否装载都是互斥的，通过条件注解 `@ConditionalOnMissingBean(ReactiveWebServerFactory.class) `实现，那么问题来了，**当依赖中存在多个 Web 容器时，会优先采用哪个呢？**

&emsp;&emsp;其实这就要看这一组互斥的 *ReactiveWebServerFactory* Bean 哪个先注册，翻看 *ReactiveWebServerFactoryAutoConfiguration* 类的 `@Import` 注解代码，答案很明显优先级为：Tomcat > Jetty > Undertow > Netty

```java
@Import({ ReactiveWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		// 优先 Tomcat 的装载
		ReactiveWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ReactiveWebServerFactoryConfiguration.EmbeddedJetty.class,
		ReactiveWebServerFactoryConfiguration.EmbeddedUndertow.class,
		ReactiveWebServerFactoryConfiguration.EmbeddedNetty.class })
public class ReactiveWebServerFactoryAutoConfiguration {
}
```

- **默认** Netty Web Server 作为嵌入式 Reactive Web 容器

	```xml
	<!-- 引入 WebFlux 依赖，同时不要引入其他 Web 容器 -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-webflux</artifactId>
	</dependency>
	```

- **UndertowWebServer** 作为嵌入式 Reactive Web 容器

	```xml
	<!-- 引入 WebFlux 依赖 -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-webflux</artifactId>
	</dependency>
	<!-- 引入 Undertow 依赖 -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-undertow</artifactId>
	</dependency>
	```
	**注意区别：** 
	- `UndertowWebServer` 是 **Reactive Web 容器**
	- `UndertowServletWebServer` 是  **Servlet Web 容器**
	- 在引入 Undertow 时而不需要在 `spring-boot-starter-webflux` 中 exclusion `spring-boot-starter-reactor-netty` 有两方面原因：
		1.  上面提及的 `@Import` 自动装配优先级里，Netty 优先级最低，不干扰其他 Web 容器的引入
		2. [使用 WebClient 时需依赖 Netty 实现，即便已引入其他 Web 容器也可共存](https://docs.spring.io/spring-boot/docs/2.1.9.RELEASE/reference/html/howto-embedded-web-servers.html#howto-use-another-web-server)

- Jetty 作为嵌入式 Reactive Web 容器

	```xml
	<!-- 引入 WebFlux 依赖 -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-webflux</artifactId>
	</dependency>
	<!-- 引入 Jetty 依赖 -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-jetty</artifactId>
	</dependency>
	```
	
- Tomcat 作为嵌入式 Reactive Web 容器

	```xml
	<!-- 引入 WebFlux 依赖 -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-webflux</artifactId>
	</dependency>
	<!-- 引入 Tomcat 依赖 -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-tomcat</artifactId>
	</dependency>
	```

## 理解自动装配

### 理解 @SpringBootApplication 注解语义

&emsp;&emsp;@SpringBootApplication 等价于 @EnableAutoConfiguration、@SpringBootConfiguration 和 @ComponentScan 三个组合注解。

- @ComponentScan 负责激活对 @Component 组件的扫描，从 **Spring Boot 2.0 版本后并非使用默认值**，而是设置了 *excludeFilters* 属性值，包含 *TypeExcludeFilter*、*AutoConfigurationExcludeFilter* 类型的两个过滤注解 @Filter。前者用于 Bean 注册时判断某个 Bean 是否被排除，后者用于排除同时标注 @Configuration 和 @EnnableAutoConfiguration 的类。

- @SpringBootConfiguration 声明被标注类为配置类，它是从 **Spring Boot 1.4 开始引入**，它由元注解 @Configuration 注解（称作注解 *派生*）而 1.4 之前 @SpringBootApplication 一直由 @Configuration 直接注解。

- @EnableAutoConfiguration 负责激活 Spring Boot 自动装配机制。

### @SpringBootApplication 属性别名

&emsp;&emsp;@SpringBootApplication 注解属性方法中，很多被 @AliasFor 注解标注，它用于桥接其他注解的属性，例如：

```java
/* 
 * 属性 scanBasePackages 是桥接 ComponentScan 注解的 basePackages 属性
 * 称这种桥接为设置别名
 * 值得注意：如果此属性没被设置，默认扫描包路径为此注解被标注的类所在的包路径
 * /
@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
String[] scanBasePackages() default {};
```

### @SpringBootApplication 继承 @Configuration 的 CGLIB 提升特性

&emsp;&emsp;有别于 @Bean、@Componet 等注解，**被 @Configuration 注解的类是被 CGLIB 增强的对象，使得在增强类中被 @Bean 注解的属性及方法被引用或调用时，得到的对象时同一个实例**。官方称这种在 @Configuration 下声明的 @Bean 为 ***完全模式（Full）***，相反在非 @Configuration 标注的普通类下声明的 @Bean 为 ***轻量模式（Lite）*** 。了解此机制对某些场景 Bean 行为不符合预期时可以进行解释，例如：类中某个方法明明声明了事务 @Transaction，但在此类中直接调用时发生异常不回滚，多半是采用了轻量模式的方法调用，应该采用 CGLIB 增强的对象进行方法调用。

> 处理 @Configuration 注解的类的 CGLIB 提升，详见 [**Spring @Enable 模块驱动原理**](#Spring @Enable 模块驱动)

### 理解自动配置机制

&emsp;&emsp;Spring Boot 1.0 在 Spring Framework 4.0 基础上，添加了约定配置化导入 @Configuration 类的方式，使得自动装配 @Configuration 类得以实现。这种自动化加载的方式被称作 ***工厂加载机制（Factory Loading Mechanism）***，也可称为 Spring 形式的 *SPI*。该机制通过结合条件注解 @Conditional 达到更灵活的条件化装配。

&emsp;&emsp;具体实现自动装配的大致步骤：
1. 定义一个标注 @Configuration 的配置类 [*WebAutoConfiguration*](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/first-app-by-gui/src/main/java/thinking/in/spring/boot/autoconfigure/WebAutoConfiguration.java)
2. 在项目路径新建资源文件 [`src/main/resources/META-INF/spring.factories`](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/first-app-by-gui/src/main/resources/META-INF/spring.factories)
3. 文件以键值对形式的出现，键为自动装配激活注解 @EnableAutoConfiguration 的全限定名，值为配置类全限定名

   ```sh
   # 自动装配配置类设置
   org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
   thinking.in.spring.boot.autoconfigure.WebAutoConfiguration
   ```

4. 项目可扫描的路径类中使用 [@EnableAutoConfiguration](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/first-app-by-gui/src/main/java/thinking/in/spring/boot/firstappbygui/FirstAppByGuiApplication.java) 激活自动装配，即可实现自定义自动装配

   ```java
   // 激活自动装配
   @EnableAutoConfiguration
   public class FirstAppByGuiApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(FirstAppByGuiApplication.class, args);
   	}
   }
   ```

> [FirstAppByGuiApplication 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/tree/master/spring-boot-2.0-samples/first-app-by-gui)

## 理解 Production-Ready 特性

### 理解 Production-Ready 一般性定义

- [12-factor](https://12factor.net/zh_cn/)

	* 外部化配置
	* 健康检测
	* 日志记录
	* 应用监控

- 云原生 Cloud Native
	
	&emsp;&emsp;云原生是一种鼓励采取简单的最佳实践进行持续交付及价值驱动的应用开发形式。
	
	&emsp;&emsp; Spring Cloud 也视 *12-factor* 为其理论基础，由于 Spring Boot Actuator 的特性很好的契合 *12-factor* 应用的特征，故 Spring Cloud 组件几乎均扩展 Spring Boot Actuator 特性的原因。
	
	

### 理解 Spring Boot Actuator

- 使用场景：监视及管理运行中的应用
- 监管媒介：HTTP 或 JMX 端点（Endpoints）
- 端点类型：审计（Auditing）、健康（Health）、指标收集（Metrics Gathering）
- 基本特点：自动运用（Automatically Applied）

参考阅读：[What Is an Actuator?](https://www.baeldung.com/spring-boot-actuators#understanding-actuator)



### Spring Boot Actuator Endpoints

添加依赖：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
Spring Boot Actuator 主要包含如下端点：
- beans 		显示 Spring 应用上下文完整 Bean 列表（所有 ApplicationContext 层次）
- conditions 	显示应用所有配置类和自动装配类的条件评估结果（包含匹配及未匹配）
- env		暴露 Spring ConfigurableEnvironment 中的 PropertySource 属性
- health		显示应用健康信息
- info		显示任意引用信息

默认仅显示 health、info 两个 Web 端点，指定属性 `management.endpoints.web.exposure.include=*` 暴露所有 Web 端点。

访问 Web 端点方式：`http://IP:Port/actuator/endpointName`，其中 *endpointName* 替换成具体的端点



### 理解外部化配置

&emsp;&emsp;Spring Boot 通过资源文件（Properties）、YAML 文件、环境变量或者命令行参数形式实现外部化配置属性值，而这些属性值可以被以下几种方式被 Spring Boot 应用所引用获取：
- Bean 的 *@Value* 注解注入
- Spring *Enviroment* 读取
- *@ConfigurationProperties* 绑定到结构化对象

参考阅读：[外部配置优先级](https://docs.spring.io/spring-boot/docs/2.6.8/reference/html/features.html#features.external-config)



### 理解规约大于配置

&emsp;&emsp;Spring Boot 规约大于配置的字面理解：期间绝对无代码生成、无须 XML 配置。换言之 Spring Boot 尽管功能强大，但是能通过一些规约达到不生成中间代码、不需要 XML 配置的情况下达到开箱即用。Spring Boot 能完成这点，从技术角度看是借助 Spring Framework 基础设施的完善结果。具体 Spring Framework 从 **XML 文件驱动** 走向 **注解驱动** 的演进过程参考如下表：

Spring Framework 版本	|	特性	|	阶段	|	备注
:--:						| 	:--:		|	:--:	|	:--:
2.5						|	@Component<br/>&lt;context:component-scanbasePackage="..."&gt;		| XML 与 @Component 结合阶段	|	XML 指定扫描范围扫描 @Component 组件
3.0						|	@Bean<br/>@Import<br/>@Configuration<br/>	|	@Bean 代替 &lt;bean&gt;<br/>@Configuration 部分替代 XML<br/> &lt;context:component-scan&gt; 	| 	需硬编码指定 Bean 扫描范围、部分摆脱 XML
3.1						|	@ComponentScan<br/>*ImportSelector* 接口<br>@EnableXxx 模块激活注解	|	全面走向注解	|	能部分选择性加载配置
4.0					|	@Conditional<br>spring.factories 工厂加载机制	|	全面走向注解	|	灵活的 @Conditional 条件注解驱动的自动化装配



### 总结

技术	|	 说明
:-- 		|	:--
Spring Framework	| 	提供核心特性：Bean 工厂、 Ioc、注解驱动、条件装配等
Spring Boot			|	提供微服务核心特性：SpringApplication、自动装配、外部化配置、Spring Boot Actuator、嵌入式 Web 容器
Spring Cloud			|	提供分布式核心特性：分布式配置、服务注册与发现、路由、服务调用、负载均衡、熔断机制、分布式消息等

&emsp;&emsp;Spring Framework 为 Spring Boot 提供核心技术支持，使得 Spring Boot 具备完整的微服务各项功能。而 Spring Cloud 将 Spring Boot 当做分布式系统中微服务的中间件，并结合自身的分布式核心特性，最终给开发者带来完整且强大的分布式系统解决方案。



## 推荐阅读

[《Spring Boot 编程思想（核心篇）》中篇 —— 走向自动装配](https://reionchan.github.io/2022/06/19/thinking-in-springboot-part-2)

[《Spring Boot 编程思想（核心篇）》下篇 —— 理解 SpringApplication](https://reionchan.github.io/2022/06/20/thinking-in-springboot-part-3)
