---
layout: post
title: Spring Boot 快速入门笔记
categories: Spring
tags: Spring Boot JavaBrains
excerpt: Spring Boot Quick Start 视频课程笔记
image: https://spring.io/img/homepage/icon-spring-boot.svg
description: JavaBrains 关于 Spring Boot 入门视频笔记
keywords: Spring, Boot, Quick Start, note
licences: cc
---

## 资源
* 课程主页：  
&emsp;&emsp;[*JavaBrains - Spring Boot Quick Start*](https://javabrains.thinkific.com/courses/springboot-quickstart)
* 课程视频：  
&emsp;&emsp;[*JavaBrains - Youtube*](https://www.youtube.com/playlist?list=PLqq-6Pq4lTTbx8p2oCgcAQGQyqN8XeA1x)  

## 简介  
  
&emsp;&emsp;本笔记是对 *[Java Brains](https://javabrains.io/)*  出品的关于 *Spring Boot 快速入门*  视频教程的概要笔记。通过该视频将会
学到如何使用 Spring Boot 创建完整的端到端的 Spring 应用，以及提供一个使用 Spring 技术构建、具备数据库支持的 REST API 的实践指南。

## 介绍 Spring Boot  
### Spring Boot 是什么
> &emsp;&emsp;Spring Boot makes it easy to **create** stand-alone, production-grade **Spring based Applications** that you can "just run".  <br>&emsp;&emsp;Spring Boot 可以轻松**创建**独立的、生产级的**基于 Spring 的应用程序**，且可以轻易“运行”它。
  
### Spring 痛点
* 框架庞大
* 步骤繁杂的设置、配置、构建及部署  

### Spring Boot 有何好处  
* 自决断性 (Opinionated)  
&emsp;&emsp;默认一套配置，且支持定制修改
* 约定优于配置 (Convention over configuration)  
&emsp;&emsp;默认约定适用大部分场景，减少配置  
* 独立性 (Stand alone)  
&emsp;&emsp;打包一切，使得Spring 的应用可独自运行
  
### Spring Boot 示例  
*  JDK 8 安装及配置  
	- 查看所有安装版本
	  
		```bash  
		Reion@MBPr ~$ /usr/libexec/java_home -V
		```  
	- 编辑 *.bash_profile* 文件，导出 *JAVA_HOME*  
  
		```bash  
		export JAVA_HOME=`/usr/libexec/java_home -v 1.8`
		```
* IDE  
&emsp;&emsp;[Spring STS](https://spring.io/tools "Spring Tool Suite")、IntelliJ IDEA
  
* Maven
	- [Maven 是什么](https://maven.apache.org/what-is-maven.html "介绍")  
	&emsp;&emsp;用来管理项目依赖关系的一个工具，自动从仓库下载依赖包

* 创建 Spring Boot 项目  
	- 打开 *STS*，【Package Explorer】窗体右键，选择【New】后选取【Maven Project】
	
	- 弹出【New Maven Project】窗口中勾选【Create a simple project(skip archetype selection)】，点击【Next】  
	
	- Group Id 输入`io.javabrains.springbootquickstart`、Artifact Id 输入`course-api`、Name 输入`Java Brains Course API`，点击【Finish】
	
	- 完成后的目录结构  
	
		```
		course-api
		└── pom.xml
		└── src
		│   └── main
		│   │   └── java
		│   │   └── resources
		│   └── test
		│       └── java
		│       └── resources
		└── target
		    └── classes
		    │   └── META-INF
		    │       └── MANIFEST.MF
		    │       └── maven
		    │           └── io.javabrains.springbootquickstart
		    │               └── course-api
		    │                   └── pom.properties
		    │                   └── pom.xml
		    └── test-classes
		```
	- 修改 *pom.xml* 文件，增加 Spring Boot 相关依赖
	
		```xml
		<!-- 告诉 Maven 依赖的 spring boot 版本信息  -->	
		<parent>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-parent</artifactId>
			<version>1.4.2.RELEASE</version>
		</parent>
		<!-- 具体 spring boot 依赖 -->
		<dependencies>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-starter-web</artifactId>
			</dependency>
		</dependencies>
		<!-- JDK 版本 -->
		<properties>
			<java.version>1.8</java.version>
		</properties>		
		```
	- 增加 Spring Boot 启动类 *CourseApiApp.java*  
	  
		```java  
		package io.javabrains.springbootstarter;

		import org.springframework.boot.SpringApplication;
		import org.springframework.boot.autoconfigure.SpringBootApplication;

		// 标注此为 Spring Boot 应用入口
		@SpringBootApplication
		public class CourseApiApp {
			public static void main(String[] args) {
				// 启动
				SpringApplication.run(CourseApiApp.class, args);
			}
		}  
		```  
		&emsp;&emsp;到此完成 Spring Boot 应用的环境搭建及配置，点击运行后竟然能够启动了，而且从后台日志可以看出包括 WebApplicationContext、Tomcat等都初始化启动完成了，果真是打包一切，开箱即运行！  

		```
		2018-04-03 12:52:02.569  INFO 3385 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8080 (http)
		2018-04-03 12:52:02.582  INFO 3385 --- [           main] i.j.springbootstarter.CourseApiApp       : Started CourseApiApp in 4.117 seconds (JVM running for 4.615)
		```  
  
	&emsp;&emsp;启动 Spring Boot，`SpringApplication.run(CourseApiApp.class, args)` 这段静态方法将会执行如下步骤：  
	- 设置默认配置
	- 启动 Spring 容器应用上下文
	- 进行类路径下的扫描（获取注解进行相关初始化）
	- 启动 Tomcat 服务器
	
* 增加 *REST 控制器* [^1] 
		
	*HelloController.java* 

	```java
	package io.javabrains.springbootstarter.hello;

	import org.springframework.web.bind.annotation.RequestMapping;
	import org.springframework.web.bind.annotation.RestController;

	// 标注此为一个控制器
	@RestController
	public class HelloController {
	
		 /*
	 	*  将 `/hello` URL 映射到 `sayHi` 方法
	 	*  
	 	*  Tips: @RequestMapping 默认映射所有 http 请求方法
	 	*  	例如：GET、POST、PUT 等等
	 	*/
		@RequestMapping("/hello")
		public String sayHi() {
			return "Hi";
		}
	}
	```  
	
	*Topic.java*  
	  
	```java
	package io.javabrains.springbootstarter.topic;
	
	public class Topic {
		
		private String id;
		private String name;
		private String description;
		
		public Topic() {}
		
		public Topic(String id, String name, String description) {
			this.id = id;
			this.name = name;
			this.description = description;
		}
	
	
		public String getId() {
			return id;
		}
		public void setId(String id) {
			this.id = id;
		}
		public String getName() {
			return name;
		}
		public void setName(String name) {
			this.name = name;
		}
		public String getDescription() {
			return description;
		}
		public void setDescription(String description) {
			this.description = description;
		}
	}		
	```
	 
	*TopicController.java*  
	  
	```java
	package io.javabrains.springbootstarter.topic;
	
	import java.util.Arrays;
	import java.util.List;
	
	import org.springframework.web.bind.annotation.RequestMapping;
	import org.springframework.web.bind.annotation.RestController;
	
	@RestController
	public class TopicController {
		
		// 返回一个列表对象的方法，对象以 Json 格式表示
		@RequestMapping("/topics")
		public List<Topic> getAllTopics() {
			return Arrays.asList(
					new Topic("spring", "Spring Framework", "Spring Framework Description"),
					new Topic("java", "Core java", "Core java Description"),
					new Topic("javascript", "Javascript", "Javascript Description"));
		}
	}		
	```
	&emsp;&emsp;启动应用，分别访问如下 URL：  
	
	```html
	http://localhost:8080/hello
	http://localhost:8080/topics
	```  

	&emsp;&emsp;得到相应结果：
	
	```
	Hi
	```
	
	```json
	[
	    {
	        "id": "spring",
	        "name": "Spring Framework",
	        "description": "Spring Framework Description"
	    },
	    {
	        "id": "java",
	        "name": "Core java",
	        "description": "Core java Description"
	    },
	    {
	        "id": "javascript",
	        "name": "Javascript",
	        "description": "Javascript Description"
	    }
	]	
	```	
		
### 实现原理  

&emsp;&emsp;从上面示例可知，只需简单的注解式的编写控制器就能实现一个客户端到服务端的请求调用，没有繁琐的配置、部署及启动操作。其实在幕后，这些繁琐操作都被 Spring Boot 隐藏起来，从而达到简化操作的目的。

&emsp;&emsp;那么 Spring Boot 如何做到这点呢？

* 材料清单  
&emsp;&emsp;Spring Boot 通过 *bom.xml* 提供应用所需的所有预设配置及依赖组合清单，这份被验证过的清单使得应用配置、部署及运行的所有操作对开发人员完全透明化。  

* 嵌入 Servlet 容器  
&emsp;&emsp;采用内嵌方式后，使得 Spring Boot 能够接管及配置 Servlet 容器。以前对容器的配置操作现在被整合提取到应用的相应配置中，使得以前分散的模块聚集成一个独立的应用，这对实现 *微服务架构*  [^2] 是一个良好基础。  
&emsp;&emsp;参考 [Spring Boot 文档关于嵌入式 Servlet 容器支持](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-embedded-container) 小节，为了实现从 **Servlet 容器管理 ServletContainerInitializer 进而引导 Spring Web 应用初始化** 转化为 **由 Spring 管理 ServletContextInitializer 引导并配置嵌入式 Servlet 容器** 的目的，应该注册实现 `org.springframework.boot.web.servlet.ServletContextInitializer` 接口的 bean，其 `onStartup` 方法提供对 `ServletContext` 的访问。  

## Spring MVC - 视图层  
### Spring MVC 原理
 
 > &emsp;&emsp;Spring Web MVC is the original web framework built on the Servlet API and included in the Spring Framework from the very beginning. The formal name "Spring Web MVC" comes from the name of its source module spring-webmvc but it is more commonly known as "Spring MVC". <br/> &emsp;&emsp;Spring Web MVC 是构建在 Servlet API 上的最初的 Web 框架，从一开始就包含在 Spring 框架中。正式名称“Spring Web MVC”来自其源模块 *spring-webmvc* 的名称， 但它通常被称为 “Spring MVC”。
 
 &emsp;&emsp;正如 [Spring 官方文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html) 所述，Spring MVC 是构建在 Java Serverlet API 之上，其核心包含一个 *DispatcherServlet* ，此 Servlet 包含一个根 *WebApplicationContext*  上下文，此根上下文持有与之相关的 *ServletContext* 引用，及此 *ServletContext* 上下文相关的 *Servlet* 信息。
 
 &emsp;&emsp;根上下文下又包括子上下文，根 *WebApplicationContext* 通常包含需要跨多个 *Servlet* 实例共享的基础架构 *bean*，例如：数据存储库和业务服务。这些 bean 被有效地继承，并且可以在特定的 *Servlet* 的子 *WebApplicationContext* 中重写（即重新声明），子上下文通常包含给定 *Servlet*  的本地 bean，例如：控制器、映射处理、视图解析器。
 
 <center><img src="https://docs.spring.io/spring/docs/current/spring-framework-reference/images/mvc-context-hierarchy.png" width="350" /><br /> 图一 上下文层级关系</center>
 
 &emsp;&emsp;由于 Spring Boot 使用 Spring 配置来引导自身及嵌入的 Servlet 容器，所以在 Servlet 容器遵循不同的初始化顺序。Filter、Servlet 是交由 Spring 配置声明，并注册到内嵌的 Servlet 容器中。  
  
### REST API  
&emsp;&emsp;REST 架构把通过 HTTP 协议预定义的请求方法来操作服务端的各种资源，下面以 Topic 为例，设计如下 REST API：  
  
请求方法 | URL | 方法说明
:-----------: | :----------- | :-----------
GET         | /topics      | 得到所有主题
GET         | /topics/id  | 得到某个 id 的主题
POST       | /topics      | 新增一个主题
PUT         | /topics/id  | 更新某个 id 的主题
DELETE    | /topics/id  | 删除某个 id 的主题  

* 代码结构：  
	  
	```	
	└── io
	    └── javabrains
	        └── springbootstarter
	            └── CourseApiApp.java
	            └── topic
	                └── Topic.java
	                └── TopicController.java
	                └── TopicService.java
	```
  
&emsp;&emsp;实现源代码：[course-api](https://github.com/ReionChan/PhotoRepo/tree/master/course-api)  
&emsp;&emsp;Http 请求模拟工具：[Postman](https://www.getpostman.com/apps)

## 生成 Spring Boot 项目   
### 生成方式 
* 使用 Spring Initializr
	- 打开链接 <https://start.spring.io/>
	- 按页面导引填入相关信息，选择依赖模块
	- 点击【Generate Project】生成一个压缩包
	- 下载并解压
	- 用 IDE 导入 Maven 工程方式即可开始开发

* 使用 Spring Boot CLI  
&emsp;&emsp;详细参见：[Installing the Spring Boot CLI](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#getting-started-installing-the-cli)

* 使用 IDE，这里以 STS 为例
	- 右键选择【New】创建工程
	- 选择【Spring Starter Project】
	- 在弹出的窗口填入必要信息，点击【Next】
	- 在依赖窗口选择必要的组件依赖，点击【Next】
	- 随后出现压缩包 url 窗口，点击【Finish】即可创建新工程

### 配置 Spring Boot
* 配置文件路径

	```  
	/app/src/main/resources/application.properties
	```
* 可配置的属性大全   

	&emsp;&emsp;[Common application properties](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#common-application-properties)
  
## Spring Data JPA - 数据层
### JPA 是什么
&emsp;&emsp;JPA ( Java Persistence API ) 是一个应用程序编程接口规范，该规范描述 Java 平台应用如何对关系型数据进行管理。其包含编程接口、Java 持久化查询语言（JPQL）及对象关系元数据三方面。

&emsp;&emsp;Spring Data JPA 是 Spring Data 体系的一部分，可以轻松实现基于 JPA 的存储库。该模块加强对基于 JPA 的数据访问层的支持。它使得构建基于 Spring 的数据访问技术应用程序更加简单。

### 添加 Spring Data JPA 支持   
 * 代码结构  
  
	```
	io
	└── javabrains
		└── CourseApiDataApplication.java
		└── springbootstarter
			└── course
			│	└── Course.java
			│	└── CourseController.java
			│	└── CourseRepository.java
			│	└── CourseService.java
			└── topic
				└── Topic.java
				└── TopicController.java
				└── TopicRepository.java
				└── TopicService.java	
	 ```  

&emsp;&emsp;一个 Topic 包含多个 Course，注意 CourseRepository 类中的关联查找，及所用到的 JPA 注解。

&emsp;&emsp;实现源代码：[course-api-data](https://github.com/ReionChan/PhotoRepo/tree/master/course-api-data) 
 	
## 部署及监控  
### 打包及运行 Spring Boot 应用
* 打包成可独立执行的 jar 文件  

	在终端 cd 到 Spring Boot 应用根目录，运行：

	```
	mvn clean install
	```
	将会把项目打包成可执行的 jar 文件：  
	```
	target
	└── course-api-data-0.0.1-SNAPSHOT.jar
	└── course-api-data-0.0.1-SNAPSHOT.jar.original
	```  
	执行如下命令即可部署及运行项目可执行 jar 文件：
	```
	java -jar target/course-api-data-0.0.1-SNAPSHOT.jar
	```
* 打包成可部署到外部 Web 容器的 war 文件

	将项目中 pom.xml 文件中的打包格式修改成 war：
	```xml
	<groupId>io.javabrains</groupId>
	<artifactId>course-api-data</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<!-- 此处修改为 war -->
	<packaging>war</packaging>
	```  
	在终端 cd 到 Spring Boot 应用根目录，运行：

	```
	mvn clean install
	```
	将会把项目打包成可外部部署的 war 文件：  
	```
	target
	└── course-api-data-0.0.1-SNAPSHOT.war
	└── course-api-data-0.0.1-SNAPSHOT.war.original
	```  
* jar vs war 和 jar.original vs war.original
	
	这两组压缩文件的目录结构区别，请查看：  
	* [course-api-data-0.0.1-SNAPSHOT.jar](https://github.com/ReionChan/PhotoRepo/blob/master/course-api-data/course-api-data-0.0.1-SNAPSHOT.jar.txt)
	* [course-api-data-0.0.1-SNAPSHOT.war](https://github.com/ReionChan/PhotoRepo/blob/master/course-api-data/course-api-data-0.0.1-SNAPSHOT.war.txt)
	* [course-api-data-0.0.1-SNAPSHOT.jar.original](https://github.com/ReionChan/PhotoRepo/blob/master/course-api-data/course-api-data-0.0.1-SNAPSHOT.jar.original.txt)
	* [course-api-data-0.0.1-SNAPSHOT.war.original](https://github.com/ReionChan/PhotoRepo/blob/master/course-api-data/course-api-data-0.0.1-SNAPSHOT.war.original.txt)
	  
&emsp;&emsp;**注意：**lib 目录中包含项目所依赖的 jar 包被故意省略了很多。

### Spring Boot Actuator
&emsp;&emsp;Actuator 能够帮助监控及管理所关联的应用程序，通过在 pom.xml 中添加 actuator 组件：  
  
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
&emsp;&emsp;增加此模块后，就可以在相应端点查看应用信息，例如：  

* /health - 查看当前应用健康信息
* /metrics - 显示当前应用的“指标”信息
* /trace - 显示跟踪信息（最后几个 http 请求）

&emsp;&emsp;例如：浏览器输入：
```
http://localhost:8080/health
```  
&emsp;&emsp;将得到如下信息：  
```json
{
    "status": "UP",
    "diskSpace": {
        "status": "UP",
        "total": 249769230336,
        "free": 74248032256,
        "threshold": 10485760
    },
    "db": {
        "status": "UP",
        "database": "Apache Derby",
        "hello": 1
    }
}
```


&emsp;&emsp;当然还有很多端点可供配置，有些没有配置默认不显示信息，例如：/info

&emsp;&emsp;详细资料参见：[spring-boot-actuators](http://www.baeldung.com/spring-boot-actuators)

## 脚注  


  [^1]: 控制器对请求 URL 和方法进行映射匹配，实现服务端的方法的调用。像这种基于 HTTP 预定义一组无状态操作来访问及操作 Web 资源的架构方式，称之为 RESTful ( *REpresentational State Transfer* ) 风格。  

[^2]: [微服务](https://en.wikipedia.org/wiki/Microservices)是面向服务架构的变体，它将应用程序构建为一系列松耦合的服务。