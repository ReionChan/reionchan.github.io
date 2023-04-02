---
layout: post
title: 《Spring Boot 编程思想（核心篇）》中篇
categories: Book Spring
tags: SpringBoot Spring Boot 
excerpt: 读该书 “第 2 部分 —— 走向自动装配” 心得笔记 
image: https://www.image.com
description: SpringBoot 一些关键特性的原理及设计思想
keywords: Spring SpringBoot Thinking 注解驱动 原理 条件装配 Spring Boot Web 自动装配 Enable 模块驱动 
licences: cc
repo: thinking-in-spring-boot-samples
---
<br/>

<img src="https://upload.wikimedia.org/wikipedia/commons/4/44/Spring_Framework_Logo_2018.svg" alt="Spring Logo" width="800"/>

---
> &emsp;&emsp;Spring Boot 具有一套固化的视图，该视图用于构建生产级别的应用。
<div style="text-align: right;"> —— 摘自 <a href="https://spring.io/projects/spring-boot/">Spring 官网</a> &emsp;&emsp;</div>

## 走向注解驱动编程

&emsp;&emsp;*IoC*（Inversion of Control）：控制反转

&emsp;&emsp;*DI*（Dependency Inject）: 依赖注入

> 所谓控制反转，即通过容器将代码需要的资源通过 *DI* 手段注入代码，由容器控制控制依赖关系。相较传统的直接在代码中直接控制依赖关系来说，是一个反向操作，所以称之为控制的反转。

### 注解驱动发展史

| 时代     | 版本                 | 描述                                                         |
| :------- | :------------------- | :----------------------------------------------------------- |
| 启蒙时代 | Spring Framework 1.x | 1.2.0 开始支持注解，但 XML 配置方式还是唯一选择。            |
| 过渡时代 | Spring Framework 2.x | 2.5 引入 @Autowired @Qualifier @Component<br> Spring MVC 模式注解，开始面向注解驱动编程。<br>由于缺乏注解驱动的应用上下文，故尚未完全替换 XML 配置驱动。 |
| 黄金时代 | Spring Framework 3.x | 全面制裁泛型、变量参数及模式注解派生层次。<br> 新增 AnnotationConfigApplicationContext 注解驱动上下文注册 @Configuration 配置类<br>加以 @ComponentScan @Bean @DependsOn @Lazy @Primary @Import 等注解，摆脱 XML 配置驱动。<br>Environment、PropertySources 接口抽象出的统一配置 API 为日后的外部化配置奠定根基。<br>借由 @Enable 模块驱动特性，更进一步拥抱注解驱动编程。<br>@Profile 是的应用上下文具备条件化的 Bean 定义能力。注解井喷式增长可谓黄金时代！ |
| 完善时代 | Spring Framework 4.x | @Conditional 完善 @Profile 不能自定义条件判断的不足。<br>支持 Java 8 新特性，新增 @EventListener 使事件监听器新增注解方式实现。<br>@AliasFor 解除派生注解必须存在相同方法签名的束缚<br>@Lookup 查找 Bean 生成方法，进一步完善 Bean 依赖 |
| 当下时代 | Spring Framework 5.x | 引入 @Indexed 减少扫描解析耗时、引入 JSR-305 适配注解，如： @NonNull @Nullable |



### Spring 核心注解场景分类

#### 模式注解

| Spring 注解    | 场景说明           | 起始版本 |
| -------------- | ------------------ | -------- |
| @Repository    | 数据仓储模式注解   | 2.0      |
| @Component     | 通用组件模式注解   | 2.5      |
| @Service       | 服务模式注解       | 2.5      |
| @Controller    | Web 控制器模式注解 | 2.5      |
| @Configuration | 配置类模式注解     | 3.0      |

#### 装配注解

| Spring 注解     | 场景说明                                    | 起始版本 |
| --------------- | ------------------------------------------- | -------- |
| @ImportResource | 替换 XML 元素 &lt;import&gt;                | 2.5      |
| @Import         | 限定 @Autowired 依赖注入范围                | 3.0      |
| @ComponentScan  | 扫描指定 package 下标注 Spring 模式注解的类 | 3.1      |

#### 依赖注入注解

| Spring 注解 | 场景说明                                                    | 起始版本 |
| ----------- | ----------------------------------------------------------- | -------- |
| @Autowired  | Bean 依赖注入，支持多种依赖查找方法（**先按类型再按名称**） | 2.5      |
| @Qualifier  | 细粒度的 @Autowired 依赖查找                                | 2.5      |

| Java 注解 | 场景说明                            | 起始版本 |
| ----------- | ----------------------------------- | -------- |
| @Resource | Bean 依赖注入，**仅支持名称依赖查找方式** | 2.5      |

#### Bean 定义注解

| Spring 注解 | 场景说明                                             | 起始版本 |
| ----------- | ---------------------------------------------------- | -------- |
| @Bean       | 替换 XML 元素 &lt;bean&gt;                           | 3.0      |
| @DependsOn  | 替换 XML 属性 &lt;bean depends-on="..."&gt;          | 3.0      |
| @Lazy       | 替换 XML 属性 &lt;bean lazy-init="true \| false"&gt; | 3.0      |
| @Primary    | 替换 XML 属性 &lt;bean primary="true \| false"&gt;   | 3.0      |
| @Role       | 替换 XML 属性 &lt;bean role="..."&gt;                | 3.1      |
| @Lookup     | 替换 XML 属性 &lt;bean lookup-method="..."&gt;       | 4.1      |

#### 条件装配注解

| Spring 注解  | 场景说明       | 起始版本 |
| ------------ | -------------- | -------- |
| @Profile     | 配置化条件装配 | 3.1      |
| @Conditional | 编程条件装配   | 4.0      |

#### 配置属性注解

| Spring 注解      | 场景说明                         | 起始版本 |
| ---------------- | -------------------------------- | -------- |
| @PropertySource  | 配置属性抽象 PropertySource 注解 | 3.1      |
| @PropertySources | @PropertySource 集合注解         | 4.0      |

#### 生命周期回调注解

| Java 注解 | 场景说明 | 起始版本 |
| ----------- | -------- | -------- |
| @PostConstruct | 替换 XML 属性 &lt;bean init-method="..."&gt; 或 InitializingBean | 2.5 |
| @PreDestroy | 替换 XML 属性 &lt;bean destroy-method="..."&gt; 或 DisposableBean | 2.5 |

#### 注解属性注解

| Spring 注解 | 场景说明                     | 起始版本 |
| ----------- | ---------------------------- | -------- |
| @AliasFor   | 别名注解属性，实现复用的目的 | 4.2      |

#### 性能注解

| Spring 注解 | 场景说明                                                     | 起始版本 |
| ----------- | ------------------------------------------------------------ | -------- |
| @Indexed    | 引入 `spring-context-indexer` 依赖后，编译时将 @Indexed 和 @Component 及派生注解标注的类<br>放入 `META-INF\spring.components` 文件中，启动时直接读取文件中组件类<br>从而替代扫 Spring 模式注解描操作，提升 Spring 启动效率 | 5.0      |

[示例-AnnotationIndexedConfiguration](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/config/AnnotationIndexedConfiguration.java)

### Spring 注解编程模型

#### 元注解

&emsp;&emsp;所谓元注解 (Meta-Annotations) 是指一个**能声明在其他注解上的注解**。元注解并非仅限定在 Spring 使用场景中，实际上 Java 标准中就有很多元注解，例如：@Inherited、@Repeatable、@Documented 等等

#### Spring 模式注解

&emsp;&emsp;所谓 Spring 模式注解 (Stereotype Annotations) 是指**定义 Spring 应用中组件角色的注解**。例如：@Repository 作为仓储标记注解，具备管理和存储某种领域对象的角色含义。

&emsp;&emsp;@Component 模式注解用来**定义能被 Spring 容器托管的通用组件角色**。换言之，任何被其标注的组件均是 Bean 扫描的候选对象。

- 理解 @Component 派生性及原理

  **派生性**

  &emsp;&emsp;所谓派生性指被**元注解**注解修饰的注解**“继承”元注解的角色含义**。

  &emsp;&emsp;拿注解 @Repository 元注解不同版本的变更为例：

  ```java
  // Spring Framework 2.0 中 @Repository 注解定义
  
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Inherited
  @Documented
  public @interface Repository {
  }
  ```

  ```java
  // Spring Framework 2.5 中 @Repository 注解定义
  
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Component
  public @interface Repository {
  }
  ```

  &emsp;&emsp;不难看出 2.0 版本时，@Repository 仅仅只用来标识被标注的类为仓储类型。在 2.5 版本时，增加 @Component 元注解修饰，表明它同时具备受 Spring 容器管理的属性，拥有被扫描成候选组件对象的能力。	

  

  **派生性原理**

  &emsp;&emsp;既然 Spring Framework 2.5 开始，@Repository 具备被扫描为候选组件能力，那么可以从组件扫描的实现入手来探究它是如何获得派生至 @Component 的这项能力。由于 2.5 版本包扫描还是依托 XML 配置文件中的 `<context:component-scan>`  元素来扫描 @Component 组件，而从 Spring Framework 2.0 开始，XML 文档引入了 [*Extensible XML authoring*](https://docs.spring.io/spring-framework/docs/3.0.x/spring-framework-reference/html/extensible-xml.html) 机制来实现 XML 元素与 Bean 定义及解析器之间的扩展，那么就从元素  `<context:component-scan>` 的解析开始入手。本例采取 Spring Framework 2.5.6.SEC03 版本的 [spring-context](https://mvnrepository.com/artifact/org.springframework/spring-context/2.5.6.SEC03) 来进行源码分析。

  1. 按照 [*Extensible XML authoring*](https://docs.spring.io/spring-framework/docs/3.0.x/spring-framework-reference/html/extensible-xml.html) 机制，查找命名空间`http://www.springframework.org/schema/context`  对应的处理类

     ```xml
     <beans xmlns="http://www.springframework.org/schema/beans"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:context="http://www.springframework.org/schema/context"
            xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
         http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
         <!-- 激活注解驱动特性 -->
         <context:annotation-config />
         <!-- 找寻被@Component或者其派生 Annotation 标记的类（Class），将它们注册为 Spring Bean -->
         <context:component-scan base-package="thinking.in.spring.boot.samples.spring25" />
     </beans>
     ```

     由 `xsi:schemaLocation="http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">` 可知：

     
  
     **命名空间**：`http://www.springframework.org/schema/context`

     **XSD 文件**：`http://www.springframework.org/schema/context/spring-context.xsd`

     
  
     以 **XSD 文件** 为键在 `/META-INF/spring.schemas` 中找到映射值即为 xsd 文件本地保存的 classpath 路径：

     ```properties
     http\://www.springframework.org/schema/context/spring-context.xsd=org/springframework/context/config/spring-context-2.5.xsd
     ```
  
     以 **命名空间** 在 `/META-INF/spring.handlers` 中找到映射值即为 context 元素处理类 *ContextNamespaceHandler* ：
  
     ```properties
     http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
     ```
  2. 找到属性 `component-scan` 解析器为 *ComponentScanBeanDefinitionParser* ：
  
     ```java
     // context 元素处理器
     public class ContextNamespaceHandler extends NamespaceHandlerSupport {
         public void init() {
           // component-scan 属性解析器类
           this.registerJava5DependentParser("component-scan", "org.springframework.context.annotation.ComponentScanBeanDefinitionParser");
         }
     }
     ```
  3. 重点看解析方法中用来参与扫描的类 *ClassPathBeanDefinitionScanner* ：
  
     ```java
     // component-scan 属性解析器类
     public class ComponentScanBeanDefinitionParser implements BeanDefinitionParser {
     
         private static final String BASE_PACKAGE_ATTRIBUTE = "base-package";
     
         // 解析方法
         public BeanDefinition parse(Element element, ParserContext parserContext) {
             String[] basePackages = StringUtils.commaDelimitedListToStringArray(element.getAttribute(BASE_PACKAGE_ATTRIBUTE));
     
             // 真正参与扫描的类 ClassPathBeanDefinitionScanner ①
             ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
             // 扫描具体方法 ④
             Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
             registerComponents(parserContext.getReaderContext(), beanDefinitions, element);
             
             return null;
         }
     
         protected ClassPathBeanDefinitionScanner configureScanner(ParserContext parserContext, Element element) {
             XmlReaderContext readerContext = parserContext.getReaderContext();
             // 采取默认过滤器
             boolean useDefaultFilters = true;
             // 创建 ClassPathBeanDefinitionScanner ②
             ClassPathBeanDefinitionScanner scanner = createScanner(readerContext, useDefaultFilters);
             scanner.setResourceLoader(readerContext.getResourceLoader());
             // ...
             return scanner;
         }
     
         protected ClassPathBeanDefinitionScanner createScanner(XmlReaderContext readerContext, boolean useDefaultFilters) {
             // 构造 ClassPathBeanDefinitionScanner ③
             return new ClassPathBeanDefinitionScanner(readerContext.getRegistry(), useDefaultFilters);
         }
     }
     ```
  
  4. 详细看 *ClassPathBeanDefinitionScanner* 中 构造方法 ③、扫描具体方法 ④ 详情：
  
     ```java
     public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {
     
         public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
             // 调用父类生成默认扫描过滤器 ①
             super(useDefaultFilters);
             // ...
         }
     
         // 扫描处理
         protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
             Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
             for (int i = 0; i < basePackages.length; i++) {
                 // 调用父类搜索可选组件方法 ②
                 Set<BeanDefinition> candidates = findCandidateComponents(basePackages[i]);
             }
             return beanDefinitions;
         }
     }
     ```
  
  5. 父类 *ClassPathScanningCandidateComponentProvider* 的 构造方法 ①，扫描处理方法 ②  ：
  
     ```java
     public class ClassPathScanningCandidateComponentProvider implements ResourceLoaderAware {
     
     	protected static final String DEFAULT_RESOURCE_PATTERN = "**/*.class";
     	
       	// 构造方法 ①
       	public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters) {
     		if (useDefaultFilters) {
     			// 默认构造过滤器，见具体方法 【CODE_1】
     			registerDefaultFilters();
     		}
     	}
       
     	// 【CODE_1】 默认过滤器中默认搜索目标 Component.class 注解类
     	protected void registerDefaultFilters() {
     		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
     	}
       
     	// 扫描处理方法 ② 
     	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
     		Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
     		...
     			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
     					resolveBasePackage(basePackage) + "/" + this.resourcePattern;
     			
     			Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
           
     			for (int i = 0; i < resources.length; i++) {
     				Resource resource = resources[i];
     				if (resource.isReadable()) {
     					MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
     					// 【条件一】从元注解读取器中筛选是否是可选组件
     					if (isCandidateComponent(metadataReader)) {
     						ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
     						sbd.setResource(resource);
     						sbd.setSource(resource);
     						// 【条件二】从 Bean 定义类中进一步判断
     						if (isCandidateComponent(sbd)) {
     							// 都符合时, 才加入可选组件集合
     							candidates.add(sbd);
     						}
     					}
     				}
           		}
     		...
     		return candidates;
     	}
       
     	// 【条件一】从元注解读取器中筛选是否是可选组件
     	protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
     		for (TypeFilter tf : this.excludeFilters) {
     			if (tf.match(metadataReader, this.metadataReaderFactory)) {
     				return false;
     			}
     		}
         	// includeFilters 中包含 new AnnotationTypeFilter(Component.class)
     		for (TypeFilter tf : this.includeFilters) {
     			// AnnotationTypeFilter 类的 match 方法
     			if (tf.match(metadataReader, this.metadataReaderFactory)) {
     				return true;
     			}
     		}
     		return false;
     	}
       
     	// 【条件二】验证 Bean 定义中的候选对象是否是非接口、非抽象的独立对象（顶级类或内部静态类）
     	protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
     		return (beanDefinition.getMetadata().isConcrete() && beanDefinition.getMetadata().isIndependent());
     	}
     }
     ```
  
  6. 观察上面代码，可以发现 `new AnnotationTypeFilter(Component.class).match()` 过滤器决定扫描对象是否被候选，而 `match()` 方法由父类默认实现，转调模板方法 `matchSelf(MetadataReader metadataReader)`, 该方法被子类 *AnnotationTypeFilter* 重写，具体方法逻辑：
  
     ```java
     public class AnnotationTypeFilter extends AbstractTypeHierarchyTraversingFilter {
         // 搜索目标注解
         private final Class<? extends Annotation> annotationType;
         // 表示是否考虑元注解
         private final boolean considerMetaAnnotations;
     
         // 由前面代码片段 【CODE_1】调用 new AnnotationTypeFilter(Component.class)
         // 搜索目标注解：Component
         // 是否考虑元注解：true
         public AnnotationTypeFilter(Class<? extends Annotation> annotationType) {
             this(annotationType, true);
         }
     
         // 覆盖父类实现，进行最终判断
         @Override
         protected boolean matchSelf(MetadataReader metadataReader) {
             // 影响最终判断逻辑的根源在于 MetadataReader 获取的 AnnotationMetadata 对象
             AnnotationMetadata metadata = metadataReader.getAnnotationMetadata();
             // 以下条件满足任意一个即可
             // 1. metadata 中所有注解中是否包含 @Component 注解
             // 2. metadata 中所有注解的元注解中是否包含 @Component 注解
             return metadata.hasAnnotation(this.annotationType.getName()) ||
                     (this.considerMetaAnnotations && metadata.hasMetaAnnotation(this.annotationType.getName()));
         }
     }
     ```
     
      
     
  7. 重点分析 `AnnotationMetadata metadata = metadataReader.getAnnotationMetadata()` 注解元数据获取，由于接口 `MetadataReader` 在 2.5 版本只存在唯一实现 *SimpleMetadataReader* ，它的 `getAnnotationMetadata()` 方法如下所示 ：
  
     ```java
     class SimpleMetadataReader implements MetadataReader {
         // 交给 AnnotationMetadataReadingVisitor 来生成 AnnotationMetadata 信息
         public AnnotationMetadata getAnnotationMetadata() {
             AnnotationMetadataReadingVisitor visitor = new AnnotationMetadataReadingVisitor(this.classLoader);
             this.classReader.accept(visitor, true);
             return visitor;
         }
     }
     ```
  
     
  
  8. 查看 *AnnotationMetadataReadingVisitor* 代码：
  
     ```java
     class AnnotationMetadataReadingVisitor extends ClassMetadataReadingVisitor implements AnnotationMetadata {
     
         // 类上的直接注解信息
         private final Map<String, Map<String, Object>> attributesMap = new LinkedHashMap<String, Map<String, Object>>();
         // 类上的直接注解的元注解信息
         private final Map<String, Set<String>> metaAnnotationMap = new LinkedHashMap<String, Set<String>>();
     
         // ASM 方式回调读取当前扫描到的 class 上的注解信息方法
         public AnnotationVisitor visitAnnotation(final String desc, boolean visible) {
             final String className = Type.getType(desc).getClassName();
             final Map<String, Object> attributes = new LinkedHashMap<String, Object>();
             return new EmptyVisitor() {
                 public void visit(String name, Object value) {
                     // Explicitly defined annotation attribute value.
                     attributes.put(name, value);
                 }
     
                 public void visitEnd() {
                     try {
                         // 加载被注解类上的某个注解的 Class 对象，例如：Component.class
                         Class annotationClass = classLoader.loadClass(className);
                         // 获取注解中所有默认属性方法.
                         Method[] annotationAttributes = annotationClass.getMethods();
                         for (int i = 0; i < annotationAttributes.length; i++) {
                             Method annotationAttribute = annotationAttributes[i];
                             String attributeName = annotationAttribute.getName();
                             Object defaultValue = annotationAttribute.getDefaultValue();
                             if (defaultValue != null && !attributes.containsKey(attributeName)) {
                                 // 收集注解的属性方法及默认值
                                 attributes.put(attributeName, defaultValue);
                             }
                         }
                         // 获取注解上的所有元注解，例如：注解在 Component.class 上的所有注解 ①
                         Annotation[] metaAnnotations = annotationClass.getAnnotations();
                         // 所有元注解类型集合
                         Set<String> metaAnnotationTypeNames = new HashSet<String>();
                         for (Annotation metaAnnotation : metaAnnotations) {
                             metaAnnotationTypeNames.add(metaAnnotation.annotationType().getName());
                         }
                         // 收集当前注解类名为 key 所有元注解类型为 value 的 Map
                         metaAnnotationMap.put(className, metaAnnotationTypeNames);
                     } catch (ClassNotFoundException ex) {
                     }
                     // 收集当前扫描到的 class 上的注解类名为 key 注解属性方法集合为 value 的 Map ②
                     attributesMap.put(className, attributes);
                 }
             };
         }
     
         // 判断类上的注解中，是否包含指定注解类型 
         public boolean hasAnnotation(String annotationType) {
             return this.attributesMap.containsKey(annotationType);
         }
     
         // 判断类上的注解的元注解中，是否包含指定注解类型
         public boolean hasMetaAnnotation(String metaAnnotationType) {
             Collection<Set<String>> allMetaTypes = this.metaAnnotationMap.values();
             for (Set<String> metaTypes : allMetaTypes) {
                 if (metaTypes.contains(metaAnnotationType)) {
                     return true;
                 }
             }
             return false;
         }
     }
     ```
  
     
  
     代码 ①  `annotationClass.getAnnotations()` 表明收集被扫描类的 **注解的上一层元注解**
  
     代码 ② `attributesMap.put(className, attributes)` 表明收集被扫描类的 **直接注解** 
  
     
  
     经过抽丝剥茧，发现最终判断逻辑为只要符合下面两句其中一个即为可选组件：
  
     - `this.attributesMap.containsKey("org.springframework.stereotype.Component")`
  
       被标注类 *UserBean* 是否被 *@Component* **直接注解**，例如：
  
       ```java
       // UserBean 被 @Component 直接注解，符合条件
       @Component
       public class UserBean {
       }
       ```
  
       
  
     - `metaTypes.contains("org.springframework.stereotype.Component")`
  
       被标注类 *UserBean* 是否被 *@Component* 间接注解（**注解的上一层元注解**），例如：
  
       ```java
       // UserBean 被 @Repository 直接注解
       // 而 @Repository 被元注解 @Component 修饰
       // 故 UserBean 符合被 @Component 间接注解，符合条件
       @Repository
       public class UserBean {
       }
       
       // @Repository 确实被 @Component 修饰
       @Target({ElementType.TYPE})
       @Retention(RetentionPolicy.RUNTIME)
       @Documented
       @Component
       public @interface Repository {
       }
       ```
  
  &emsp;&emsp;综上所述，可以发现如下结论：
  
  1. **Spring Framework 2.5 仅能够实现单层次派生**
  
     例如：某个类要称为可扫描的候选组件，那该类要么**被 @Component 修饰**，要么**被 @Component 派生的注解修饰**。
  
  2. 利用 *ClassPathBeanDefinitionScanner* 和 *AnnotationTypeFilter* 能实现**不派生 @Component 的注解**扫描注册（**编程式**）
  
  3. 利用 `<context:component-scan>` 子元素 `<context:include-filter>` 也能实现不派生 @Component 的注解扫描注册（**配置式**）：
  
     ```xml
     <!-- 
         注意 @Component 具备两项语义：
     		1. 可扫描成候选组件 
     		2. value 指定 Bean 名称
     	@StringRepository 不派生 @Component 但要达到相同功能就要将这两个语义修复
     		1. <context:include-filter> 使其拥有可扫描成候选组件语义
     		2. name-generator 属性使其 value 拥有指定 Bean 名称语义
     -->
     <context:component-scan base-package="thinking.in.spring.boot.samples.spring25"
                             name-generator="thinking.in.spring.boot.samples.spring25.annotation.CustomerAnnotationBeanNameGenerator">
         <!-- 此元素让 @StringRepository 具备可扫描性 -->
         <context:include-filter type="annotation"
                                 expression="thinking.in.spring.boot.samples.spring25.annotation.StringRepository"/>
     </context:component-scan>
     ```
     
     
  
  > [DerivedComponentAnnotationBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/tree/master/spring-framework-samples/spring-framework-2.5.6-sample)
  
  
  
- 多层次 @Component 派生性及原理

  

  &emsp;&emsp;通过上一小节可知，Spring Framework 2.5 仅支持单层次派生，那 3.0 版本会如何呢？我们修改 [DerivedComponentAnnotationBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/tree/master/spring-framework-samples/spring-framework-2.5.6-sample) 的 [pom.xml](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-2.5.6-sample/pom.xml) 中 Spring 的版本，并同时将 @NameRepository 从直接派生 @Component 修改为直接派生 @Repository，而 @Repository 是直接派生的 @Component，所以来验证 3.0 版本是否支持两层次的派生。

  

  修改 [pom.xml](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-2.5.6-sample/pom.xml) 使得依赖的 Spring Framework 为 3.0

  ```xml
  <properties>
      <!-- <spring.version>2.5.6.SEC03</spring.version> -->
      <!-- 升级 Spring Framework 到 3.0.0.RELEASE  -->
      <spring.version>3.0.0.RELEASE</spring.version>
  </properties>
  ```

  调整 [@NameRepository](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-2.5.6-sample/src/main/java/thinking/in/spring/boot/samples/spring25/annotation/StringRepository.java) 派生层级

  ```java
  //@Component // 测试多层次 @Component派生、或自定义可扫描注解时，请将当前注释
  @Repository // 测试多层次 @Component派生，请将当前反注释，并且将 spring-context 升级到 3.0.0.RELEASE
  public @interface StringRepository {
      String value() default "";
  }
  ```

  &emsp;&emsp;调整后项目可以正常运行，说明 Spring Framework 3.0 已经支持**双层派生**，通过上节可以知道能否支持几层，关键看 *AnnotationMetadataReadingVisitor* 对目标类上的注解的解析到底下探到几层，下面是 3.0 版本 *AnnotationMetadataReadingVisitor* 与 2.5 版本调整部分的代码片段：

  ```java
  // 变更为 final 类
  final class AnnotationMetadataReadingVisitor extends ClassMetadataReadingVisitor implements AnnotationMetadata {
  
      // 新增直接注解集合
      private final Set<String> annotationSet = new LinkedHashSet<String>();
  
      private final Map<String, Set<String>> metaAnnotationMap = new LinkedHashMap<String, Set<String>>();
  
      private final Map<String, Map<String, Object>> attributeMap = new LinkedHashMap<String, Map<String, Object>>();
  
      // 注解访问方式委托给 AnnotationAttributesReadingVisitor 类的 visitEnd() 方法
      // 2.5 版本则是委托给 EmptyVisitor 类的 visitEnd() 方法
      @Override
      public AnnotationVisitor visitAnnotation(final String desc, boolean visible) {
          String className = Type.getType(desc).getClassName();
          this.annotationSet.add(className);
          return new AnnotationAttributesReadingVisitor(className, this.attributeMap, this.metaAnnotationMap, this.classLoader);
      }
  
      // 2.5 是 this.attributesMap.containsKey(annotationType) 判断
      public boolean hasAnnotation(String annotationType) {
          return this.annotationSet.contains(annotationType);
      }
  
      // 【影响派生层次的判断方法不变】证明主要改动是 this.metaAnnotationMap.values() 值的变化
      public boolean hasMetaAnnotation(String metaAnnotationType) {
          Collection<Set<String>> allMetaTypes = this.metaAnnotationMap.values();
          for (Set<String> metaTypes : allMetaTypes) {
              if (metaTypes.contains(metaAnnotationType)) {
                  return true;
              }
          }
          return false;
      }
  }
  ```

  &emsp;&emsp;对比发现，判断是否包含某个元注解的方法 `boolean hasMetaAnnotation(String metaAnnotationType)` 没有变动，跨两级派生判断是否包含 @Component 却返回为 `true`，只能是 `this.metaAnnotationMap.values()` 里确实出现了 @Component，而这个 Map 的值构建是在 *AnnotationAttributesReadingVisitor* 类的 `visitEnd()` 方法：

  ```java
  final class AnnotationAttributesReadingVisitor implements AnnotationVisitor {
  
      private final String annotationType;
  
      private final Map<String, Map<String, Object>> attributesMap;
  
      private final Map<String, Set<String>> metaAnnotationMap;
  
      private final Map<String, Object> localAttributes = new LinkedHashMap<String, Object>();
  
      public void visitEnd() {
          this.attributesMap.put(this.annotationType, this.localAttributes);
          try {
              // 当前需解析上层元注解的注解 class，本例为 StringRepository.class
              Class<?> annotationClass = this.classLoader.loadClass(this.annotationType);
              // Check declared default values of attributes in the annotation type.
              Method[] annotationAttributes = annotationClass.getMethods();
              for (Method annotationAttribute : annotationAttributes) {
                  String attributeName = annotationAttribute.getName();
                  Object defaultValue = annotationAttribute.getDefaultValue();
                  if (defaultValue != null && !this.localAttributes.containsKey(attributeName)) {
                      this.localAttributes.put(attributeName, defaultValue);
                  }
              }
              // 该注解上的元注解名称集合，本例为 StringRepository 上的元注解集合
              Set<String> metaAnnotationTypeNames = new LinkedHashSet<String>();
              // 与 2.5 版本单层 for 循环相比，3.0 采取双层 for 循环
              // 很明显，下探层数为两级, 本例为：
              //		1. 第一层 for 解析 StringRepository 的元注解，得到 Repository
              // 		2. 第二层 for 解析 Repository 的元注解，得到 Component
              for (Annotation metaAnnotation : annotationClass.getAnnotations()) {
                  metaAnnotationTypeNames.add(metaAnnotation.annotationType().getName());
                  if (!this.attributesMap.containsKey(metaAnnotation.annotationType().getName())) {
                      this.attributesMap.put(metaAnnotation.annotationType().getName(),
                              AnnotationUtils.getAnnotationAttributes(metaAnnotation, true));
                  }
                  for (Annotation metaMetaAnnotation : metaAnnotation.annotationType().getAnnotations()) {
                      metaAnnotationTypeNames.add(metaMetaAnnotation.annotationType().getName());
                  }
              }
              if (this.metaAnnotationMap != null) {
                  this.metaAnnotationMap.put(this.annotationType, metaAnnotationTypeNames);
              }
          } catch (ClassNotFoundException ex) {
              // Class not found - can't determine meta-annotations.
          }
      }
  }
  ```
  
  &emsp;&emsp;由此证明，Spring Framework 3.0 版本支持两层次的派生, 并且由于采取双层 for 循环解析的原因，也仅仅只支持两层派生。 通过项目 [HierarchicalDerivedComponentAnnotationBootstrap](https://github.com/ReionChan/thinking-in-spring-boot-samples/tree/master/spring-framework-samples/spring-framework-3.0.x-sample) 修改 3.x 的不同版本，进一步发现：

  

  > Spring Framework 3.x 都只支持两层次派生，从 4.0 版本后支持多层次（两级以上）派生。 

  

  通过查看 4.0 版本源代码，发现解析元注解已从双层 for 循环变更为**递归实现**，从而实现多层次派生。

  ```java
  final class AnnotationAttributesReadingVisitor extends RecursiveAnnotationAttributesVisitor {
  
      private final String annotationType;
  
      private final MultiValueMap<String, AnnotationAttributes> attributesMap;
  
      private final Map<String, Set<String>> metaAnnotationMap;
  
  
      @Override
      public void doVisitEnd(Class<?> annotationClass) {
          super.doVisitEnd(annotationClass);
          List<AnnotationAttributes> attributes = this.attributesMap.get(this.annotationType);
          if (attributes == null) {
              this.attributesMap.add(this.annotationType, this.attributes);
          } else {
              attributes.add(0, this.attributes);
          }
          Set<String> metaAnnotationTypeNames = new LinkedHashSet<String>();
          for (Annotation metaAnnotation : annotationClass.getAnnotations()) {
              // 将每个元注解递归解析上层元注解
              recursivelyCollectMetaAnnotations(metaAnnotationTypeNames, metaAnnotation);
          }
          if (this.metaAnnotationMap != null) {
              this.metaAnnotationMap.put(annotationClass.getName(), metaAnnotationTypeNames);
          }
      }
  
      private void recursivelyCollectMetaAnnotations(Set<String> visited, Annotation annotation) {
          if (visited.add(annotation.annotationType().getName())) {
              // Only do further scanning for public annotations; we'd run into IllegalAccessExceptions
              // otherwise, and don't want to mess with accessibility in a SecurityManager environment.
              if (Modifier.isPublic(annotation.annotationType().getModifiers())) {
                  this.attributesMap.add(annotation.annotationType().getName(),
                          AnnotationUtils.getAnnotationAttributes(annotation, true, true));
                  for (Annotation metaMetaAnnotation : annotation.annotationType().getAnnotations()) {
                      // 此处递归收集元注解
                      recursivelyCollectMetaAnnotations(visited, metaMetaAnnotation);
                  }
              }
          }
      }
  }
  ```
  
  

- Spring 组合注解

  

  &emsp;&emsp;*组合注解* (Composed Annotations) 是指某个注解 “元标注” 一个或多个其他注解，目的在于**将关联的注解行为组合成单个自定义注解**。

  

  &emsp;&emsp;Spring 组合注解中的元注解允许是 **Spring 模式注解**与其他 **Spring 功能注解** 的任意组合。Spring 通过 ***AnnotationMetadata*** 接口对外暴露组合注解中包含的元注解信息，经过之前对 *@Component* 派生性原理分析可知，该接口的实现类为 ***AnnotationMetadataReadingVisitor*** 并且在 Spring Framework 4.0 开始关联 ***AnnotationAttributesReadingVisitor*** 类得以实现递归查找多层次元注解信息，并将其保存在 *AnnotationMetadataReadingVisitor* 的 `metaAnnotationMap` 属性字段中。

  

  围绕 ***AnnotationMetadata*** 接口相关类图，如下所示：

  

  <img src="https://raw.githubusercontent.com/ReionChan/thinking-in-spring-boot-samples/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/resources/annotation-metadata.svg" alt="annotation-metadata"/>

  

  &emsp;&emsp;由类图可知，要获取指定 **class 全限定名** 的 **AnnotationMetadata 注解元信息**，可以由 *CachingMetadataReaderFactory* 获取  *SimpleMeatdataReader* 元数据读取器，从而获得 *AnnotationMetadata* 接口实例，继而可以获取该 class 的 **类元数据信息（ClassMetadata）**、**注解元信息（AnnotationMetadata）** 等多种信息。

  

  > ***AnnotationMetadata API***  提供整套完善的访问类元信息的能力，为 Spring **走向注解驱动编程奠定良好的基础**。

  

  项目 [spring-framework-5.0.x-sample](https://github.com/ReionChan/thinking-in-spring-boot-samples/tree/master/spring-framework-samples/spring-framework-5.0.x-sample) 中的 [TransactionalServiceAnnotationMetadataBootstrap](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/bootstrap/TransactionalServiceAnnotationMetadataBootstrap.java) 示例演示了获取该类的注解信息：

  ```java
  @TransactionalService
  public class TransactionalServiceAnnotationMetadataBootstrap {
  
      public static void main(String[] args) throws IOException {
          // @TransactionalService 标注在当前类 TransactionalServiceAnnotationMetadataBootstrap
          String className = TransactionalServiceAnnotationMetadataBootstrap.class.getName();
          // 构建 MetadataReaderFactory 实例
          MetadataReaderFactory metadataReaderFactory = new CachingMetadataReaderFactory();
          // 读取 @TransactionService MetadataReader 信息
          MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
          // 读取 @TransactionService AnnotationMetadata 信息
          AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
  
          annotationMetadata.getAnnotationTypes().forEach(annotationType -> {
  
              Set<String> metaAnnotationTypes = annotationMetadata.getMetaAnnotationTypes(annotationType);
  
              metaAnnotationTypes.forEach(metaAnnotationType -> {
                  System.out.printf("注解 @%s 元标注 @%s\n", annotationType, metaAnnotationType);
              });
  
          });
      }
  }
  ```

  

  接口 *AnnotationMetadata* 有基于 **Java 反射** 和 **ASM** 两种技术实现。实现扫描候选 Bean 定义信息功能时，采取的是 **ASM** 方式。

  

  **ASM** 优点在于：

  * 应用启动阶段减少不必要的 *类装载 ClassLoader* 开销
  * 方便做字节码增强，为框架实现类动态功能增强做铺垫

    

  > [TransactionalServiceAnnotationMetadataBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/bootstrap/TransactionalServiceAnnotationMetadataBootstrap.java)

  

- Spring 注解属性别名和覆盖（Meta-Annotations）

  

  &emsp;&emsp;&emsp;上一小节中，在扫描候选 Bean 定义时，借助 *AnnotationMetadata* 的 `hasAnnotation(String annotationType)` 、`hasMetaAnnotation(String metaAnnotationName)` 方法能轻在某个类的多层级注解中判断是否包含指定注解。而那时使用的 *AnnotationMetadata* 接口的实现类是 ***AnnotationMetadataReadingVisitor***，它基于 ASM 技术。

  

  &emsp;&emsp;本小节将通过对比 **纯 Java 反射 API** 、*AnnotationMetadata* 的另一个实现类 ***StandardAnnotationMetadata（ 基于Java反射 ）*** 两种方式获取类的注解属性信息，来进一步验证 *AnnotationMetadata* 抽象带来的便利性。另外将理解 Spring 注解属性的抽象类 ***AnnotationAttributes*** 以及注解属性的 ***覆盖机制*** 和 ***别名机制***。

  

  - 理解 Spring 注解元信息抽象 ***AnnotationMetadata***

    获取类 *TransactionalService* 注解所有属性信息（纯 Java 反射 API 版本）

    ```java
    @TransactionalService(name = "test")
    public class TransactionalServiceAnnotationReflectionBootstrap {
    
        public static void main(String[] args) {
            // Class 实现了 AnnotatedElement 接口
            AnnotatedElement annotatedElement = TransactionalServiceAnnotationReflectionBootstrap.class;
            // 从 AnnotatedElement 获取 TransactionalService
            TransactionalService transactionalService = annotatedElement.getAnnotation(TransactionalService.class);
            // 获取 transactionalService 的所有的元注解
            Set<Annotation> metaAnnotations = getAllMetaAnnotations(transactionalService);
            // 输出结果
            metaAnnotations.forEach(TransactionalServiceAnnotationReflectionBootstrap::printAnnotationAttribute);
        }
    
        private static Set<Annotation> getAllMetaAnnotations(Annotation annotation) {
            Annotation[] metaAnnotations = annotation.annotationType().getAnnotations();
            if (ObjectUtils.isEmpty(metaAnnotations)) { // 没有找到，返回空集合
                return Collections.emptySet();
            }
            // 获取所有非 Java 标准元注解结合
            Set<Annotation> metaAnnotationsSet = Stream.of(metaAnnotations)
                    // 排除 Java 标准注解，如 @Target，@Documented 等，它们因相互依赖，将导致递归不断
                    // 通过 java.lang.annotation 包名排除
                    .filter(metaAnnotation -> !Target.class.getPackage().equals(metaAnnotation.annotationType().getPackage()))
                    .collect(Collectors.toSet());
    
            // 递归查找元注解的元注解集合
            Set<Annotation> metaMetaAnnotationsSet = metaAnnotationsSet.stream()
                    .map(TransactionalServiceAnnotationReflectionBootstrap::getAllMetaAnnotations)
                    .collect(HashSet::new, Set::addAll, Set::addAll);
            // 添加递归结果
            metaMetaAnnotationsSet.add(annotation);
            return metaMetaAnnotationsSet;
        }
    
    
        private static void printAnnotationAttribute(Annotation annotation) {
            Class<?> annotationType = annotation.annotationType();
            // 完全 Java 反射实现（ReflectionUtils 为 Spring 反射工具类）
            ReflectionUtils.doWithMethods(annotationType,
                    method -> System.out.printf("@%s.%s() = %s\n", annotationType.getSimpleName(),
                            method.getName(), ReflectionUtils.invokeMethod(method, annotation)) // 执行 Method 反射调用
                    , method -> !method.getDeclaringClass().equals(Annotation.class));// 选择非 Annotation 方法
        }
    }
    ```

    获取类 *TransactionalService* 注解所有属性信息（StandardAnnotationMetadata 版本）

    ```java
    @TransactionalService
    public class TransactionalServiceStandardAnnotationMetadataBootstrap {
    
        public static void main(String[] args) throws IOException {
    
            // 读取 @TransactionService AnnotationMetadata 信息
            // AnnotationMetadata Java 反射实现版本
            AnnotationMetadata annotationMetadata = new StandardAnnotationMetadata(TransactionalServiceStandardAnnotationMetadataBootstrap.class);
    
            // AnnotationMetadata ASM 实现版本
            // MetadataReader reader = new SimpleMetadataReaderFactory().getMetadataReader(TransactionalServiceStandardAnnotationMetadataBootstrap.class.getName());
            // AnnotationMetadata annotationMetadata = reader.getAnnotationMetadata();
    
            // 获取所有的元注解类型（全类名）集合
            Set<String> metaAnnotationTypes = annotationMetadata.getAnnotationTypes()
                    .stream() // TO Stream
                    .map(annotationMetadata::getMetaAnnotationTypes) // 读取单注解的元注解类型集合
                    .collect(LinkedHashSet::new, Set::addAll, Set::addAll); // 合并元注解类型（全类名）集合
    
            metaAnnotationTypes.forEach(metaAnnotation -> { // 读取所有元注解类型
                // 读取元注解属性信息
                Map<String, Object> annotationAttributes = annotationMetadata.getAnnotationAttributes(metaAnnotation);
                if (!CollectionUtils.isEmpty(annotationAttributes)) {
                    annotationAttributes.forEach((name, value) ->
                            System.out.printf("注解 @%s 属性 %s = %s\n", ClassUtils.getShortName(metaAnnotation), name, value));
                }
            });
        }
    }
    ```

    &emsp;&emsp;上面的例子中，也可以将 *反射版本*  ***StandardAnnotationMetadata*** 替换成 ASM 版本 ***AnnotationMetadataReadingVisitor***， ASM 版本更适合不装载 class 文件的场景，例如：扫描指定包路径下的 Spring 模式注解。两者性能上存在明显差异，ASM 版本更加优异，具体可以执行 [AnnotationMetadataPerformanceBootstrap](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/bootstrap/AnnotationMetadataPerformanceBootstrap.java) 类进行性能比较。

    

    &emsp;&emsp;而只用 **纯 Java 反射 API** 获取每个嵌套元注解的属性信息时，需要开发者自己编写递归查询。与之相对，***StandardAnnotationMetadata*** **`getAnnotationTypes()`** 方法却能直接获取所有嵌套元注解的类型集合，再通过 **`getAnnotationAttributes(String annotationName)`** 方法获取每个注解的属性信息。

    

    > ***AnnotationMetadata* 将复杂的递归搜集元注解的过程 “扁平化”，降低开发成本**。

    

  - 理解 Spring 注解属性抽象 ***AnnotationAttributes***

    

    &emsp;&emsp;观察 *StandardAnnotationMetadata* 类的 **`getAnnotationAttributes(String annotationName)`** 方法的返回类型为 **AnnotationAttributes**, 它由 **AnnotatedElementUtils.getMergedAnnotationAttributes** 静态方法生成，并且是一个继承于 ***LinkedHashMap<String, Object>*** 的**有序 Map**，因为它既要使用 **Key-Value 的数据结构**来保存属性方法的 **名称** 和 **值**，还要确保 **属性方法的次序与 运行时反射加载的属性方法数组顺序 [^1] 一致**。

    

    &emsp;&emsp;根据 [*AnnotationAttributesBootstrap*](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/bootstrap/AnnotationAttributesBootstrap.java) 验证，还能发现 *AnnotatedElementUtils.getMergedAnnotationAttributes(AnnotatedElement element， String annotationName)* 会在 element 的元注解中，搜索第一个类型名为 annotationName 注解及该 annotationName 上的元注解的属性合并添加到 **AnnotationAttributes** 之中。在属性合并中难免会发生 annotationName 注解属性名与其元注解上的属性名有相同的情况，而 *AnnotationAttributes*

    为 *Map* 的缘故，势必会出现相同 key 不同 value 的覆盖问题，就必须遵照一定的规则，也就是下一节要讲的属性覆盖规则。

    

  - 理解 Spring 注解属性覆盖（Overrides）

    

    &emsp;&emsp;*AnnotationAttributes* 采用注解就近覆盖的规则，也就是说较低层注解能够覆盖其元注解的同名属性。这种由于元注解层次高低关系衍生出的 “Spring 注解属性覆盖” 的规则，称其为 **隐式覆盖**。

    

    &emsp;&emsp;以项目 [**spring-framework-5.0.x-sample**](https://github.com/ReionChan/thinking-in-spring-boot-samples/tree/master/spring-framework-samples/spring-framework-5.0.x-sample) 中所定义的注解 *@TransactionalService* 为例，它的元注解层次关系为：

    ```java
    @Component                          高
        |-  @Service                    中
            |-  @TransactionalService   低
    ```

    

    - 隐式覆盖（Implicit Overrides）

      即较低层注解覆盖其上元注解的同名属性的覆盖规则。那上面例子举例：

      *@TransactionalService* 的 `value`覆盖 *@Service* 元注解的 `value`

      

    - 显式覆盖（Explicit Overrides）

      &emsp;&emsp;即使用元注解 ***@AliasFor*** 显式指定当前注解的某个属性显示覆盖其上层元注解的某个属性，拿 *@TransactionalService* 举例：

      ```java
      @Service(value = "transactionalService")
      public @interface TransactionalService {
          // 显式指定将 name 属性显示覆盖其上元注解 @Service 的 value 属性
          @AliasFor(attribute = "value", annotation = Service.class)
          String name() default "";
      }
      ```

      &emsp;&emsp;显示覆盖包含下面含义：

      - 被显示覆盖的注解一定是当前注解的元注解

        &emsp;&emsp;本例即：被显示覆盖的注解 *@Service* 一定是当前注解 *@TransactionalService* 的元注解

      - 指定覆盖的属性方法名可不与被覆盖的元注解属性方法名形同（隐式覆盖一定是同名属性）

        &emsp;&emsp;本例即：*@TransactionalService* 的 `name` 显示覆盖 *@Service* 的 `value`

      - **属性覆盖** 发生在注解及其元注解之间

        &emsp;&emsp;注意区别下一节的 **属性别名**，它只发生在相同注解内部

        

    - 传递的显式覆盖（Transitive Explicit Overrides）

      &emsp;&emsp;如果 *@One* 注解的属性 `A` 显示覆盖 @Two 注解的属性 `B`，而 *@Two* 注解的属性 `B` 又显式覆盖 *@Three* 注解的属性 `C`，那么 *@One* 注解的属性 `A` 传递显示覆盖 *@Three* 注解的属性 `C`。

      ```
      IF
          @One#A --Overrides--> @Two#B
          @Two#B --Overrides--> @Three#C
      THEN
          @One#A --Overrides--> @Three#C
      ```

      

  > [TransactionalServiceBeanBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-5.0.x-sample/src/main/java/thinking/in/spring/boot/samples/spring5/bootstrap/TransactionalServiceBeanBootstrap.java)

  

  - 理解 Spring 注解属性别名（Aliases）

    &emsp;&emsp;当 ***@AliasFor*** 用在同一注解属性方法之间，这些相互被注解的属性方法之间，互称为别名。

    - 显式别名

      &emsp;&emsp;**相同注解中属性方法之间相互 “@AliasFor”** 并且 **默认值必须相等**，例如：

      ```java
      @Target({ElementType.TYPE})
      @Retention(RetentionPolicy.RUNTIME)
      @Documented
      @Transactional
      @Service(value = "transactionalService")
      public @interface TransactionalService {
      
          @AliasFor(attribute = "value")
          String name() default "txManager";
      
          @AliasFor("name")
          String value() default "txManager";
      }
      ```

      &emsp;&emsp;本例中，`name()` 和 `value()` 即是显式别名，尤其注意它们的默认值必须相同 `default "txManager"`。

      

      &emsp;&emsp;细心留意，不难发现 *@TransactionalService* 的 `value()` 隐式覆盖 *@Transactional* 的 `value()`，而后者又显式别名 `transactionManager()`。当执行事务时，Spring 会优先在 Bean 容器中找名称为 "txManager" 的 *PlatformTransactionManager* 事务管理器。这是因为 *@TransactionalService* 的 `value()` 隐式别名覆盖了  *@Transactional* 的 `value()`，使得 *@Transactional* 的 `value()` 被设置为 "txManager"，即寻找事务管理器的**限定符**，符合该限定符的事务管理器会被优先采纳。

      

      扩展阅读：依赖的自动依赖推断

      

    - 隐式别名覆盖

      &emsp;&emsp;继续上面例子，*@TransactionalService* 中 `name()` 和 `value()` 显式别名，其中的 `value()` 又隐式覆盖其元注解 *@Service* 的 `value()`，那么可以称 *@TransactionalSerice* 的 `name()` 隐式别名覆盖 *@Service* 的 `value()`。

      ```
      IF
          @TransactionalService#name --Alias--> @TransactionalService#value
          @TransactionalService#value --Implicit Overrides--> @Service#value
      THEN
          @TransactionalService#name -- Implicit Alias Overrides--> @Service#value
      ```

      &emsp;&emsp;注意：这里之所以称为隐式别名覆盖，而不直接叫隐式别名，是遵照别名只发生在相同注解之中，只有覆盖才发生在不同注解之间。

      

    - 传递隐式别名覆盖

      &emsp;&emsp;隐式别名覆盖更进一步，考虑 *@Service* 的 `value()` 又隐式覆盖 *@Component* 的 `value()`，那么可以称  *@TransactionalSerice* 的 `name()` 传递隐式别名覆盖  *@Component* 的 `value()`。

      ```
      IF
          @TransactionalService#name --Alias--> @TransactionalService#value
          @TransactionalService#value --Implicit Overrides--> @Service#value
          @Service#value --Implicit Overrides--> @Component#value
      THEN
          @TransactionalService#name -- Transitive Implicit Alias Overrides--> @Component#value
      ```

  

## Spring 注解驱动设计模式

### Spring @Enable 模块驱动

&emsp;&emsp;Spring Framework 3.1 开始支持 **@Enable 模块驱动**。所谓模块是指具备相同领域功能组件集合，组合所形成的一个独立的单元。

#### 理解 @Enable 模块驱动

&emsp;&emsp;引入 “@Enable 模块驱动” 意义在于能够**简化装配步骤，实现了按需装配，屏蔽组件装配的细节**。

&emsp;&emsp;弊端是该模式必须手动添加 @Enable 激活注解，且实现成本相对较高。



常见的 @Enable 激活注解及对应功能模块如下表所示：

| 框架实现         | @Enable 激活注解               | 激活模块             |
| ---------------- | ------------------------------ | -------------------- |
| Spring Framework | @EnableWebMvc                  | Web MVC 模块         |
|                  | @EnableTransactionManagement   | 事务管理模块         |
|                  | @EnableCaching                 | Caching 模块         |
|                  | @EnableMBeanExport             | JMX  模块            |
|                  | @EnableAsync                   | 异步处理模块         |
|                  | @EnableWebFlux                 | Web Flux 模块        |
|                  | @EnableAspectJAutoProxy        | AspectJ 代理模块     |
| Spring Boot      | @EnableAutoConfiguration       | 自动装配模块         |
|                  | @EnableManagementContext       | Actuator 管理模块    |
|                  | @EnableConfigurationProperties | 配置属性绑定模块     |
|                  | @EnableOAuth2Sso               | OAuth 2 单点登录模块 |
| Spring Cloud     | @EnableEurekaServer            | Eureka 服务器模块    |
|                  | @EnableConfigServer            | 配置服务器模块       |
|                  | @EnableFeignClients            | Feign 客户端模块     |
|                  | @EnableZuulProxy               | 服务网格 Zuul 模块   |
|                  | @EnableCircuitBreaker          | 服务熔断模块         |



#### 自定义 @Enable 模块驱动

&emsp;&emsp;自定义 @Enable 模块驱动，需借助 Spring Framework 3.0 开始引入的注解 ***@Import***，注意区分导入配置资源文件的注解 *@ImportResource*。

*@Import* 在 Spring Framework 3.0 后续版本中，语义有所增加：

| Spring Framework 版本 | 语义                                                         |
| --------------------- | ------------------------------------------------------------ |
| 3.0                   | 导入一个或多个由 **@Configuration** 标注的配置类             |
| 3.1                   | 导入一个或多个由 @Configuration 标注的配置类<br/>导入一个或多个包含由 @Bean 标注声明方法的类<br/>导入 *ImportSelector* 或 *ImportBeanDefinitionRegistrar* 的实现类 |

&emsp;&emsp;将借助 @Configuration 和 @Bean 实现自定义模块驱动归类为 **注解驱动**

&emsp;&emsp;将借助 *ImportSelector* 或 *ImportBeanDefinitionRegistrar* 接口实现自定义模块驱动归类为 **接口编程**

1. 注解驱动实现自定义 @Enable 模块驱动

   - 定义配置类

     ```java
     @Configuration
     public class HelloWorldConfiguration {
     
         @Bean
         public String helloWorld() { // 创建名为"helloWorld" String 类型的Bean
             return "Hello,World";
         }
     }
     ```

   - 定义激活注解

     ```java
     @Target(ElementType.TYPE)
     @Retention(RetentionPolicy.RUNTIME)
     @Documented
     @Import(HelloWorldConfiguration.class) // 导入 HelloWorldConfiguration
     public @interface EnableHelloWorld {
     }
     ```

   - 将激活注解标注到能被 Spring 容器运行时注册的 Spring 模式组件 Bean 上（**原理小节对该句话有详细解释**） 

     ```java
     @EnableHelloWorld
     @Configuration
     public class EnableHelloWorldBootstrap {
     
         public static void main(String[] args) {
             // 构建 Annotation 配置驱动 Spring 上下文
             AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
             // 注册 当前引导类（被 @Configuration 标注） 到 Spring 上下文
             context.register(EnableHelloWorldBootstrap.class);
             // 启动上下文
             context.refresh();
             // 获取名称为 "helloWorld" Bean 对象
             String helloWorld = context.getBean("helloWorld", String.class);
             // 输出用户名称："Hello,World"
             System.out.printf("helloWorld = %s \n", helloWorld);
             // 关闭上下文
             context.close();
         }
     }
     ```

   > [EnableHelloWorldBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-3.2.x-sample/src/main/java/thinking/in/spring/boot/samples/spring3/bootstrap/EnableHelloWorldBootstrap.java)

2. 接口编程实现自定义 @Enable 模块驱动

   

   * 编写业务可选配置组件、业务类

     接口 [*Server*](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-3.2.x-sample/src/main/java/thinking/in/spring/boot/samples/spring3/server/Server.java) 及两个实现类 [*FtpServer*](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-3.2.x-sample/src/main/java/thinking/in/spring/boot/samples/spring3/server/FtpServer.java)、[*HttpServer*](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-3.2.x-sample/src/main/java/thinking/in/spring/boot/samples/spring3/server/HttpServer.java)，这里仅展示 HttpServer：

     ```java
     @Component // 根据 ImportSelector 的契约，请确保是实现为 Spring 组件
     public class HttpServer implements Server {
     
         @Override
         public void start() {
             System.out.println("HTTP 服务器启动中...");
         }
     
         @Override
         public void stop() {
             System.out.println("HTTP 服务器关闭中...");
         }
     }
     ```

     注：

     &emsp;&emsp;小马哥特别强调**遵照 *ImportSelector* 的契约** 此类中加上 *@Component* 注解（其实观察下方代码的判断方法，还可以加 @Configuration、@Bean注解），确保该类是 Spring 候选**配置型组件**。

     

     &emsp;&emsp;在了解后文原理小节可以得知，此处所指的 *ImportSelector* 契约，即**被它所引入的类是具备潜在配置能力的**，引入类能够被封装成为 ***ConfigurationClass*** 进行后续的解析它里面所包含的配置信息（例如：*@Bean*、*@PropertySource*、 *@ComponentScan* 等等）。小马哥在本例中类 HttpServer 没有任何配置信息，下面解析代码将不会解析到任何信息，单纯空跑。**尤其注意此契约是强制性要求，不满足契约将会抛出异常**。（契约的代码详见下文：*ImportSelector* 与 *ImportBeanDefinitionRegistrar* 区别）

     &emsp;&emsp;此处为 *ConfigurationClassParser* 解析配置型组件中包含配置信息的代码：

     ```java
     protected AnnotationMetadata doProcessConfigurationClass(ConfigurationClass configClass, AnnotationMetadata metadata) throws IOException {
         ...
     
         // 收集 @PropertySource 注解定义的配置信息
         AnnotationAttributes propertySource = MetadataUtils.attributesFor(metadata,
             org.springframework.context.annotation.PropertySource.class);
         if (propertySource != null) {
             ...
         }
     
         // 收集 @ComponentScan 注解定义的配置信息
         AnnotationAttributes componentScan = MetadataUtils.attributesFor(metadata, ComponentScan.class);
         if (componentScan != null) {
             ...
         }
     
         // 收集 @Import 注解定义的配置信息
         Set<Object> imports = new LinkedHashSet<Object>();
         Set<String> visited = new LinkedHashSet<String>();
         collectImports(metadata, imports, visited);
         if (!imports.isEmpty()) {
             processImport(configClass, metadata, imports, true);
         }
     
         // 收集 @ImportResource 注解定义的配置信息
         if (metadata.isAnnotated(ImportResource.class.getName())) {
             ...
         }
     
         // 收集 @Bean 注解定义的配置信息
         Set<MethodMetadata> beanMethods = metadata.getAnnotatedMethods(Bean.class.getName());
         for (MethodMetadata methodMetadata : beanMethods) {
             ...
         }
     
         // 递归处理父类中的配置信息
         if (metadata.hasSuperClass()) {
             String superclass = metadata.getSuperClassName();
             ...
         }
     
         return null;
     }
     ```
     
     
   
   * 定义激活注解
   
     ```java
     @Target(ElementType.TYPE)
     @Retention(RetentionPolicy.RUNTIME)
     @Documented
     @Import(ServerImportSelector.class) // 导入 ServerImportSelector
     //@Import(ServerImportBeanDefinitionRegistrar.class) // 替换 ServerImportSelector
     public @interface EnableServer {
     
         /**
          * 设置服务器类型
          * @return non-null
          */
         Server.Type type();
     }
     ```
   
     
   
   * 编写 *ImportSelector* 或 *ImportBeanDefinitionRegistrar* 实现类，达到选择性装配可选配置组件
   
     - *ImportSelector* 实现
   
       ```java
       public class ServerImportSelector implements ImportSelector {
       
           @Override
           public String[] selectImports(AnnotationMetadata importingClassMetadata) {
               // 读取 EnableServer 中的所有的属性方法，本例中仅有 type() 属性方法
               // 其中 key 为 属性方法的名称，value 为属性方法返回对象
               Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(EnableServer.class.getName());
               // 获取名为"type" 的属相方法，并且强制转化成 Server.Type 类型
               Server.Type type = (Server.Type) annotationAttributes.get("type");
               // 导入的类名称数组
               String[] importClassNames = new String[0];
               switch (type) {
                   case HTTP: // 当设置 HTTP 服务器类型时，返回 HttpServer 组件
                       importClassNames = new String[]{HttpServer.class.getName()};
                       break;
                   case FTP: //  当设置 FTP  服务器类型时，返回 FtpServer  组件
                       importClassNames = new String[]{FtpServer.class.getName()};
                       break;
               }
               return importClassNames;
           }
       }
       ```
   
       
   
     - *ImportBeanDefinitionRegistrar* 实现
   
       ```java
       public class ServerImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
       
           @Override
           public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
               // 复用 {@link ServerImportSelector} 实现，避免重复劳动
               ImportSelector importSelector = new ServerImportSelector();
               // 筛选 Class 名称集合
               String[] selectedClassNames = importSelector.selectImports(importingClassMetadata);
               // 创建 Bean 定义
               Stream.of(selectedClassNames)
                       .map(BeanDefinitionBuilder::genericBeanDefinition) // 转化为 BeanDefinitionBuilder 对象
                       .map(BeanDefinitionBuilder::getBeanDefinition)     // 转化为 BeanDefinition
                       .forEach(beanDefinition ->
                               // 注册 BeanDefinition 到 BeanDefinitionRegistry
                               BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry)
                       );
           }
       }
       ```
   
       *ImportSelector* 与 *ImportBeanDefinitionRegistrar* 区别
   
       - *ImportSelector* 仅暴露被 *@EnableServer* 注解标注的配置类的元注解信息
   
         语义：
   
          1. 根据给定元注解信息**条件**，**返回相应所需加载的配置型组件** 的 class 数组
   
             ***ImportSelector* 契约**：引入类必须符合 Full 模式 （@Configuration 注解） 或 Lite 模式 （@Component 派生注解等注解，从Spring Framework 3.1+ 后增加其他注解 ）
     
          2. 引入的配置型组件由 **ConfigurationClassBeanDefinitionReader** 生成并注册 BeanDefinition
     
             - 不满足契约，将抛出 ***InvalidConfigurationImportProblem*** 异常：
     
               ```java
               private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass){
                   if(configClass.isImported()){
                       // ImportSelector 引入的配置型组件类 交给此方法生成并注册 BeanDefinition
                       registerBeanDefinitionForImportedConfigurationClass(configClass);
                   }
                   for(BeanMethod beanMethod:configClass.getBeanMethods()){
                       loadBeanDefinitionsForBeanMethod(beanMethod);
                   }
                   loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
               }
               
               private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass){
                   AnnotationMetadata metadata=configClass.getMetadata();
                   BeanDefinition configBeanDef=new AnnotatedGenericBeanDefinition(metadata);
                       
                   // 【ImportSelector 契约】不满足 Full 或 Lite 模式将抛出异常
                   if(ConfigurationClassUtils.checkConfigurationClassCandidate(configBeanDef,this.metadataReaderFactory)){
                       String configBeanName=this.importBeanNameGenerator.generateBeanName(configBeanDef,this.registry);
                       // 在此进行引入的配置型组件类 BeanDefinition 的注册
                       this.registry.registerBeanDefinition(configBeanName,configBeanDef);
                       configClass.setBeanName(configBeanName);
                       if(logger.isDebugEnabled()){
                           logger.debug(String.format("Registered bean definition for imported @Configuration class %s",configBeanName));
                       }
                   }
                   else {
                       this.problemReporter.error(new InvalidConfigurationImportProblem(metadata.getClassName(),configClass.getResource(),metadata));
                   }
               }
               ```
     
               
     
             - 是否满足契约条件判断
             
               ```java
               public static boolean checkConfigurationClassCandidate(BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {
                       AnnotationMetadata metadata = null;
                   ...
                   if (metadata != null) {
                       // FULL 模式：metadata.isAnnotated(Configuration.class.getName())
                       if (isFullConfigurationCandidate(metadata)) {
                           beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
                           return true;
                       }
                       //  LITE 模式：(Spring Framework 3.1+ 之后添加了许多注解，本代码为 3.1 版本)
                       //  !metadata.isInterface() && (metadata.isAnnotated(Component.class.getName()) || metadata.hasAnnotatedMethods(Bean.class.getName())));
                       else if (isLiteConfigurationCandidate(metadata)) {
                           beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
                           return true;
                       }
                   }
                   
                   return false;
               }
               ```
               
               
     
       - *ImportBeanDefinitionRegistrar* 额外提供了 BeanDefinitionRegistry，可以使开发人员额外注册一些 Bean 定义
     
         语义：根据给定元注解信息**条件**，利用暴露的 **BeanDefinitionRegistry 手动选取并注册** 所需的 ***BeanDefinition***
         
         
     
     - 将激活注解标注到能被 Spring 容器运行时注册的 Spring 模式组件 Bean 上（**原理小节对该句话有详细解释**） 
     
       ```java
       @Configuration
       @EnableServer(type = Server.Type.HTTP) // 设置 HTTP 服务器
       public class EnableServerBootstrap {
       
           public static void main(String[] args) {
               // 构建 Annotation 配置驱动 Spring 上下文
               AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
               // 注册 当前引导类（被 @Configuration 标注） 到 Spring 上下文
               context.register(EnableServerBootstrap.class);
               // 启动上下文
               context.refresh();
               // 获取 Server Bean 对象，实际为 HttpServer
               Server server = context.getBean(Server.class);
               // 启动服务器
               server.start();
               // 关闭服务器
               server.stop();
               // 关闭上下文
               context.close();
           }
       }
       ```
   
   
   
   &emsp;&emsp;相比较 **注解驱动** 实现，**编程方式** 实现的模块驱动弹性更大，但也更为复杂，使用成本更高。
   
   > [EnableServerBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-3.2.x-sample/src/main/java/thinking/in/spring/boot/samples/spring3/bootstrap/EnableHelloWorldBootstrap.java)
   
   

#### @Enable 模块驱动原理

&emsp;&emsp;&emsp;从自定义 *@Enable* 了解到，自定义的激活注解 *@EnableHelloWorld*、*@EnableServer* 要需**手动标注**在**能被 Spring 容器运行时注册的 Spring 模式组件 Bean** （即：被 ***@Component*** 或 **派生至 *@Component*** 的注解标注的 Bean）之上。之所以要强调 **Spring 容器运行时注册** 是想说明激活注解标注的类具备如下特点：

1. 该类被 Spring 容器运行时注册成为 Bean

   原因：不能在容器运行时注册为 Bean 就不可能识别其上标注的**自定义激活注解**，更没机会**分析其上的 *@Import* 元注解**

2. 该类被 ***@Component*** 或 **派生至 *@Component*** 的注解标注（Spring Framework 3.0 的限制，3.1 之后逐渐增加处理注解） 

   原因：只有被 Spring 模式注解标注，才会被后续提到的 ***ConfigurationClassPostProcessor*** 处理，进而**解析自定义激活注解上的 *@Import***

   版本差异：
   
   ​	**Spring Framework 3.1** 版本时，该类含有 *@Bean* 注解的方法也能得到支持
   
   ​	**Spring Framework 4.0** 版本时，该类被 *@Import* 注解（包含间接注解）也能得到支持
   
   ​	**Spring Framework 5.0** 版本后，继续添加支持 *@ComponentScan*、 *@ImportResource* 元注解
   
   ​	之所以存在差异，根本原因在于可被 ***ConfigurationClassPostProcessor*** 处理的判断方法 ***ConfigurationClassUtils.isLiteConfigurationCandidate(AnnotationMetadata)*** 在不同版本之间存在差异，该方法在 Spring Framework 3.1 版本引入，4.0 及 5.0 不断完善添加了多种注解。
   
   
   
   > &emsp;&emsp;Spring Framework 4.0 开始支持被 *@Import* 注解（含被间接注解）也能被 ***ConfigurationClassPostProcessor*** 处理，而自定义 @Enable 本身就带有 *@Import*，也就是说 4.0 开始**只要被标注自定义 *@Enable* 的类被注册即可实现自定义装配而无需该类是 @Component 标注的模式注解。**
   >
   > &emsp;&emsp;这点很重要，该特性使得 Spring boot 的 *@EnableAutoConfiguration* 无需依赖 *@SpringBootConfiguration* 或 *@Configuration* 标注的类，详细参考 [理解 Spring Boot 自动装配对 @EnableAutoConfiguration 理解小节阐述](https://reionchan.github.io/2022/06/19/thinking-in-springboot-part-2/#%E7%90%86%E8%A7%A3-spring-boot-%E8%87%AA%E5%8A%A8%E8%A3%85%E9%85%8D)。
   

&emsp;&emsp;由此可见，驱动模块实现核心是解析 ***@Import*** 注解，来装载其所指定的导入类，并将这些导入类定义为 Spring Bean。这些导入类可以是 *@Configuration* 标注的配置类，也可以是接口 *ImportSelector*、*ImportBeanDefinitionRegistrar* 的实现类，用来实现相对灵活的条件装配。**Spring Framework 3.0** 首先支持 *@Import* 装载 *@Configuration* 配置类，**Spring Framework 3.1** 才支持对接口 *ImportSelector*、*ImportBeanDefinitionRegistrar* 的实现类的装载。

&emsp;&emsp;下面将从***@Import* 注解处理器的注册**、***@Import* 的解析**、 ***@Import* 引入的 *@Configuration* 类装载**、***@Import* 引入的*ImportSelector*、*ImportBeanDefinitionRegistrar* 实现类装载** 、**装载的配置类注册转化为 *BeanDefinition***、**@Configuration 类 CGLIB 增强** 六个过程了解 @Enable 模块驱动实现原理。

- *@Import* 注解处理器的注册

  &emsp;&emsp;*@Import* 注解是自定义 @EnableXxx 激活注解的元注解。要搜索定位到 *@Import*，就必须将 *@EnableXxx* 标注在能被 Spring 容器运行时加载，具备 @Component 派生属性的类上。如上例中 *@EnableServer* 标注在被模式注解 *@Configuration* 修饰的 *EnableHelloWorldBootstrap* 类上，而该类使用显式注册为 Spring 容器中的模式组件。

  ```java
  /*
   * 此处替换为 @Component 或其它派生至 @Component 的注解
   * 原因：ConfigurationClassPostProcessor 只会处理 @Component 或 派生至 @Component 所标注的类的元注解信息。
   *      由它处理的注解包括：@PropertySource @ComponentScan @Import @ImportResource @Bean
   *
   * 由此，可以引出 Spring 模式注解标注的 Bean 与 未被 Spring 模式注解标注的 Bean 差异问题。
   *      至少，未被模式注解标注的 Bean 它上面的 @PropertySource @ComponentScan @Import @ImportResource @Bean 不会被识别处理
   */
  @Configuration
  @EnableServer(type = Server.Type.HTTP) // 设置 HTTP 服务器
  public class EnableServerBootstrap {
  
      public static void main(String[] args) {
          // 构建 Annotation 配置驱动 Spring 上下文
          AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
          // 注册 当前引导类（被 @Configuration 标注） 到 Spring 上下文
          // 显示注册到 Spring 容器
          context.register(EnableServerBootstrap.class);
          ...
      }
  }
  ```

  &emsp;&emsp;目前为止，Spring 上下文中就有了 *EnableServerBootstrap* Bean 的定义信息，其上的 *@EnableServer* 注解的元注解 *@Import* 将会交由接口 *BeanFactoryPostProcessor* 的实现类 ***ConfigurationClassPostProcessor*** 处理，那这个处理器是如何被注册到 Spring 上下文的 Bean 容器从而发挥其作用的呢？根据运行环境的不同，有以下注册途径：

  

  1. XML 配置驱动时代，由元素 `<context:annotation-config />` 或元素 `<context:component-scan>` 的**解析器**激活注册

     &emsp;&emsp;由 [Spring 注解编程模型关于**派生性及原理**小节](https://reionchan.github.io/2022/06/19/thinking-in-springboot-part-2/#spring-%E6%B3%A8%E8%A7%A3%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B) 中搜索元素`<context:component-scan>` 的 **解析器** 可知，两元素的解析器分别为：*AnnotationConfigBeanDefinitionParser*、 *ComponentScanBeanDefinitionParser*

     ```java
     public class ContextNamespaceHandler extends NamespaceHandlerSupport {
     
     	public void init() {
     		...
     		registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
     		registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
     		...
     	}
     }
     ```

     &emsp;&emsp;*AnnotationConfigBeanDefinitionParser* 激活 ConfigurationClassPostProcessor 注册方法：

     ```java
     public class AnnotationConfigBeanDefinitionParser implements BeanDefinitionParser {
     
     	public BeanDefinition parse(Element element, ParserContext parserContext) {
     		...
     		// 委托 AnnotationConfigUtils.registerAnnotationConfigProcessors 方法注册注解配置处理器
     		Set<BeanDefinitionHolder> processorDefinitions =
     				AnnotationConfigUtils.registerAnnotationConfigProcessors(parserContext.getRegistry(), source);
     		...
     	}
     }
     ```

     &emsp;&emsp;*ComponentScanBeanDefinitionParser* 激活 ConfigurationClassPostProcessor 注册方法：

     ```java
     public class ComponentScanBeanDefinitionParser implements BeanDefinitionParser {
     
         protected void registerComponents(
                 XmlReaderContext readerContext, Set<BeanDefinitionHolder> beanDefinitions, Element element) {
             ...
             boolean annotationConfig = true;
             if (element.hasAttribute(ANNOTATION_CONFIG_ATTRIBUTE)) {
                 // 属性 annotation-config 是否是 true
                 annotationConfig = Boolean.valueOf(element.getAttribute(ANNOTATION_CONFIG_ATTRIBUTE));
             }
             if (annotationConfig) {
                 // 为 true 注册注解配置处理器
                 Set<BeanDefinitionHolder> processorDefinitions =
                         AnnotationConfigUtils.registerAnnotationConfigProcessors(readerContext.getRegistry(), source);
                 ...
             }
             ...
         }
     }
     ```
     
     &emsp;&emsp;查看 `registerAnnotationConfigProcessors` 方法，发现 *ConfigurationClassPostProcessor* 处理器只是其中被注册的处理器之一：
     
     ```java
     public class AnnotationConfigUtils {
     	// XML 配置驱动时代调用的注册方法
     	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
     			BeanDefinitionRegistry registry, Object source) {
     		// ConfigurationClassPostProcessor 就在其中
     		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
     			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
     			def.setSource(source);
     			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
     		}
     		// 注册处理 @Autowire 的处理器
     		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
     			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
     			def.setSource(source);
     			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
     		}
     		// 省略很多处理器注册，包括：处理 @Required、 JSR-250、JPA 等注解的处理器
     		return beanDefs;
     	}
       
       // 注解驱动时代调用的注册方法，当然也只是对上面注册方法的重用
       public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
     		registerAnnotationConfigProcessors(registry, null);
     	}
     }
     ```
     
     
     
  2. 注解驱动时代，由 ***AnnotationConfigApplicationContext*** 上下文构造方法激活注册
  
     &emsp;&emsp;*AnnotationConfigApplicationContext* 构造方法中初始化两种 Bean 定义读取器时，委托它们俩由激活注册：
  
     ```java
     public class AnnotationConfigApplicationContext extends GenericApplicationContext {
     	// 两种 Bean 定义读取器，分别基于 ASM 和 Java 反射技术实现
     	private final AnnotatedBeanDefinitionReader reader;
     	private final ClassPathBeanDefinitionScanner scanner;
       
     	// 构造方法
     	public AnnotationConfigApplicationContext() {
     		this.reader = new AnnotatedBeanDefinitionReader(this);
     		this.scanner = new ClassPathBeanDefinitionScanner(this);
     	}
     }
     ```
  
     &emsp;&emsp;*AnnotatedBeanDefinitionReader* 激活 ConfigurationClassPostProcessor 注册方法：
  
     ```java
     public class AnnotatedBeanDefinitionReader {
         ...
         public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
             ...
             // 委托 AnnotationConfigUtils 注册，方法参考上面 XML 配置驱动涉及该类的源代码
             AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
         }
         ...
     }
     ```
  
     &emsp;&emsp;*ClassPathBeanDefinitionScanner* 激活 ConfigurationClassPostProcessor 注册方法：
  
     ```java
     public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {
         ...
         public int scan(String... basePackages) {
             ...
             doScan(basePackages);
             if (this.includeAnnotationConfig) {
                 // 扫描完毕后，进行注册，方法参考上面 XML 配置驱动涉及该类的源代码
                 AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
             }
             return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
         }
     }
     ```
  
  
  
  &emsp;&emsp;至此，已经明确知道 *@Import* 的处理器 *ConfigurationClassPostProcessor* 在何时加载注册到 Spring 应用上下文，接下来将详细讲述该处理器对 @Import 的解析过程。
  
  
  
- *@Import* 注解的解析

  

  &emsp;&emsp;*ConfigurationClassPostProcessor* 类在 **Spring Framework 3.0** 中直接实现 ***BeanFactoryPostProcessor*** 接口，而到 **Spring Framework 3.0.1** 时改为直接实现 ***BeanDefinitionRegistryPostProcessor*** 接口，而后者继承于 ***BeanFactoryPostProcessor***。所以 *ConfigurationClassPostProcessor* 实现逻辑从原本实现 *BeanFactoryPostProcessor* 接口的方法中 **抽取了大部分处理逻辑** 放到实现 *BeanDefinitionRegistryPostProcessor* 接口的方法中。从而将原本一步完成的动作分作两阶段完成（阅读下边源码的两阶段注释）。 下面所涉及的源码都将基于变动之后的版本 **Spring Framework 3.2**。

  

  &emsp;&emsp;和所有 *BeanFactoryPostProcessor* 一样，***ConfigurationClassPostProcessor*** 也经由抽象类 *AbstractApplicationContext* 的 `refresh()` 方法中的模板方法 **`invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory)`** 而被最终调用：

  ```java
  // 【注意】此代码为 Spring Framework 3.2 源码
  public abstract class AbstractApplicationContext extends DefaultResourceLoader
          implements ConfigurableApplicationContext, DisposableBean {
  
      ...
      // 模板方法默认实现
      protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
          // Invoke BeanDefinitionRegistryPostProcessors first, if any.
          Set<String> processedBeans = new HashSet<String>();
          if (beanFactory instanceof BeanDefinitionRegistry) {
              BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
              List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
              List<BeanDefinitionRegistryPostProcessor> registryPostProcessors =
                      new LinkedList<BeanDefinitionRegistryPostProcessor>();
              for (BeanFactoryPostProcessor postProcessor : getBeanFactoryPostProcessors()) {
                  // 此处的 BeanDefinitionRegistryPostProcessor 来源于 AbstractApplicationContext#beanFactoryPostProcessors
                  // 不是 ConfigurationClassPostProcessor 注册所存的位置
                  if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                      BeanDefinitionRegistryPostProcessor registryPostProcessor =
                              (BeanDefinitionRegistryPostProcessor) postProcessor;
                      registryPostProcessor.postProcessBeanDefinitionRegistry(registry);
                      registryPostProcessors.add(registryPostProcessor);
                  } else {
                      regularPostProcessors.add(postProcessor);
                  }
              }
              // 此处的 BeanDefinitionRegistryPostProcessor 来源于 DefaultListableBeanFactory#beanDefinitionMap
              // 正是 ConfigurationClassPostProcessor 注册所存的位置，并生成其单例对象返回
              Map<String, BeanDefinitionRegistryPostProcessor> beanMap =
                      beanFactory.getBeansOfType(BeanDefinitionRegistryPostProcessor.class, true, false);
              List<BeanDefinitionRegistryPostProcessor> registryPostProcessorBeans =
                      new ArrayList<BeanDefinitionRegistryPostProcessor>(beanMap.values());
              OrderComparator.sort(registryPostProcessorBeans);
              for (BeanDefinitionRegistryPostProcessor postProcessor : registryPostProcessorBeans) {
                  // 【阶段一】此处调用 ConfigurationClassPostProcessor 实现方法 postProcessBeanDefinitionRegistry
                  postProcessor.postProcessBeanDefinitionRegistry(registry);
              }
              invokeBeanFactoryPostProcessors(registryPostProcessors, beanFactory);
              // 再调用 invokeBeanFactoryPostProcessors 方法，跳转到【阶段二】调用 
              invokeBeanFactoryPostProcessors(registryPostProcessorBeans, beanFactory);
              invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
              processedBeans.addAll(beanMap.keySet());
          }
          ...
      }
  
      private void invokeBeanFactoryPostProcessors(
              Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
          for (BeanFactoryPostProcessor postProcessor : postProcessors) {
              // 【阶段二】此处调用 ConfigurationClassPostProcessor 实现方法 postProcessBeanFactory
              postProcessor.postProcessBeanFactory(beanFactory);
          }
      }
  }
  ```
  
  &emsp;&emsp;emsp;先看【阶段一】即 *ConfigurationClassPostProcessor* 实现接口 *BeanDefinitionRegistryPostProcessor* 的方法：
  
  ```java
  public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
          Ordered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
  
      // 实现 BeanDefinitionRegistryPostProcessor 的方法
      public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
          Set<BeanDefinitionHolder> configCandidates = new LinkedHashSet<BeanDefinitionHolder>();
          for (String beanName : registry.getBeanDefinitionNames()) {
              BeanDefinition beanDef = registry.getBeanDefinition(beanName);
              // 过滤出希望被 ConfigurationClassPostProcessor 处理的 Bean 定义
              //【注意】checkConfigurationClassCandidate 方法
              //  1. 它能解释为何 @EnableXxx 注解一定要标注在 Spring 模式注解注解的类上，能被称作具备配置能力的类需含如下特征
              //      1.1 明确被 @Configuration 注解, 归为 Full 模式，后续被 CGLIB 提升
              //      1.2	非接口，并且要么被 @Component 注解（4.0后支持多层递归派生）要么方法中有被 @Bean 标注，归为 Lite 模式，不会被提升
              //      1.3 Lite 模式条件在 4.0 中添加了 @Import, 5.0 中更是追加了 @ComponentScan 及 @ ImportResource
              //  2. 它能解释为何 @Configuration 注解的类会被 CGLIB 提升
              if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
                  configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
              }
          }
    		...
  
          // 开始解析过滤出的 Bean 定义（具备配置能力的 Bean 定义）
          ConfigurationClassParser parser = new ConfigurationClassParser(
                  this.metadataReaderFactory, this.problemReporter, this.environment,
                  this.resourceLoader, this.componentScanBeanNameGenerator, registry);
          for (BeanDefinitionHolder holder : configCandidates) {
              BeanDefinition bd = holder.getBeanDefinition();
              try {
                  if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                      // 已装载 class 的采用 Java 反射解析
                      parser.parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
                  } else {
                      // 未装载的采用 ASM 方式解析
                      parser.parse(bd.getBeanClassName(), holder.getBeanName());
                  }
              } catch (IOException ex) {
                  ...
              }
          }
          ...
      }
  }
  ```
  
  &emsp;&emsp;要关注对 *@Import* 的处理，那么观察解析器 *ConfigurationClassParser* 的解析方法 **`parse`** ：
  
  ```java
  class ConfigurationClassParser {
      // ASM 解析，将目标类封装为 ConfigurationClass
        public void parse(String className, String beanName) throws IOException {
            MetadataReader reader = this.metadataReaderFactory.getMetadataReader(className);
            processConfigurationClass(new ConfigurationClass(reader, beanName));
        }
    
        // Java 反射解析，将目标类封装为 ConfigurationClass
        public void parse(Class<?> clazz, String beanName) throws IOException {
            processConfigurationClass(new ConfigurationClass(clazz, beanName));
        }
    
        // 统一解析封装类 ConfigurationClass 的方法
        protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
            AnnotationMetadata metadata = configClass.getMetadata();
    		...
    
            do {
                // 如果当前的配置类有父类，会递归调用解析所有继承层级的类
                metadata = doProcessConfigurationClass(configClass, metadata);
            }
            while (metadata != null);
            // 将解析完成后的 configClass 放入集合，方便【阶段二】做 CGLIB 提升 
            this.configurationClasses.add(configClass);
        }
    
        // 真正解析的方法
        protected AnnotationMetadata doProcessConfigurationClass(ConfigurationClass configClass, AnnotationMetadata metadata) throws IOException {
    
            // 实际代码有省略，该处还会解析处理如下注解 @PropertySource @ComponentScan @ImportResource  @Bean methods
            ...
    
            // 解析 @Import 注解的处理方法
            Set<Object> imports = new LinkedHashSet<Object>();
            Set<String> visited = new LinkedHashSet<String>();
            // 解析 @Import 收集由它导入的类
            collectImports(metadata, imports, visited);
            if (!imports.isEmpty()) {
                // 都收集的导入类进行装载
                processImport(configClass, metadata, imports, true);
            }
    
            // 如果当前类有父类，返回父类元信息方便递归解析
            if (metadata.hasSuperClass()) {
                String superclass = metadata.getSuperClassName();
                if (!superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
                    this.knownSuperclasses.put(superclass, configClass);
                    // superclass found, return its annotation metadata and recurse
                    if (metadata instanceof StandardAnnotationMetadata) {
                        Class<?> clazz = ((StandardAnnotationMetadata) metadata).getIntrospectedClass();
                        return new StandardAnnotationMetadata(clazz.getSuperclass(), true);
                    } else {
                        MetadataReader reader = this.metadataReaderFactory.getMetadataReader(superclass);
                        return reader.getAnnotationMetadata();
                    }
                }
            }
            return null;
        }
    }
  ```
  
  &emsp;最终定位到对注解 *@Import* 处理的方法，其中 **`collectImport`** 方法为解析 @Import 并收集由其导入的类：
  
  ```java
  private void collectImports(AnnotationMetadata metadata, Set<Object> imports, Set<String> visited) throws IOException {
        String className = metadata.getClassName();
        if (visited.add(className)) {
            // 反射形式解析
            if (metadata instanceof StandardAnnotationMetadata) {
                StandardAnnotationMetadata stdMetadata = (StandardAnnotationMetadata) metadata;
                // 拿到当前类的所有注解
                for (Annotation ann : stdMetadata.getIntrospectedClass().getAnnotations()) {
                    if (!ann.annotationType().getName().startsWith("java") && !(ann instanceof Import)) {
                        // 递归嵌套收集 @Import 标注的类元信息，因为 @Import 可能引入被 @Import 标注的类
                        collectImports(new StandardAnnotationMetadata(ann.annotationType()), imports, visited);
                    }
                }
                // 获得 @Import 上的属性方法，拿到 value 值（引入类的集合）
                Map<String, Object> attributes = stdMetadata.getAnnotationAttributes(Import.class.getName(), false);
                if (attributes != null) {
                    Class<?>[] value = (Class<?>[]) attributes.get("value");
                    if (!ObjectUtils.isEmpty(value)) {
                        for (Class<?> importedClass : value) {
                            imports.remove(importedClass.getName());
                            // 收集引入类到集合
                            imports.add(importedClass);
                        }
                    }
                }
            } else {
                // ASM 形式解析，逻辑雷同，故省略
            }
        }
    }
  ```
  
  &emsp;&emsp;&emsp;解析完毕后得到了引入类的集合，进而对引入类进行装载，由 **`processImport`** 方法进行处理。按引入类的类型划分为**被 *@Configuration* 标注的类的装载**、**实现 *ImportSelector*、*ImportBeanDefinitionRegistrar* 接口的类的装载**，将在下面两小节介绍。 

 

- *@Import* 引入的 *@Configuration* 类装载

  &emsp;&emsp;**被 @Configuration 标注的类的装载**处理就是上小节出现过的方法 **`processConfigurationClass(ConfigurationClass configClass)`** ，证据在 **`processImport`** 方法的此处：

  ```java
  private void processImport(ConfigurationClass configClass, AnnotationMetadata metadata,
          Collection<?> classesToImport, boolean checkForCircularImports) throws IOException {
      ...
      else {
          ...
          for (Object candidate : classesToImport) {
              Object candidateToCheck = (candidate instanceof Class ? (Class) candidate :
                  this.metadataReaderFactory.getMetadataReader((String) candidate));
              ...
              else {
                  this.importStack.registerImport(metadata,
                      (candidate instanceof Class ? ((Class) candidate).getName() : (String) candidate));
                  // 此处就是处理被 @Configuration 注解的类、含 @Bean 的方法的类、@Component 修饰或普通类的装载
                  // 其实就是封装为 ConfigurationClass 后转调上一节的方法
                  processConfigurationClass(candidateToCheck instanceof Class ?
                      new ConfigurationClass((Class) candidateToCheck, true) :
                      new ConfigurationClass((MetadataReader) candidateToCheck, true));
              }
          }
      }
      ...
  }
  ```
  
  
  
- *@Import* 引入的 *ImportSelector*、*ImportBeanDefinitionRegistrar* 实现类装载

  &emsp;&emsp;同样在 **`processImport`** 方法中，可以看到针对实现两接口的类的处理逻辑，此处理逻辑是 Spring Framework 3.1 后添加：

  ```java
  private void processImport(ConfigurationClass configClass, AnnotationMetadata metadata,
          Collection<?> classesToImport, boolean checkForCircularImports) throws IOException {
      ...
      for (Object candidate : classesToImport) {
          Object candidateToCheck = (candidate instanceof Class ? (Class) candidate :
              this.metadataReaderFactory.getMetadataReader((String) candidate));
          
          // 判断是否为 ImportSelector 接口实现类
          if (checkAssignability(ImportSelector.class, candidateToCheck)) {
              Class<?> candidateClass = (candidate instanceof Class ? (Class) candidate :
                  this.resourceLoader.getClassLoader().loadClass((String) candidate));
              ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
              
              // 调用接口 ImportSelector#selectImports 方法，并将返回的引入类递归调用，继续处理引入
              processImport(configClass, metadata, Arrays.asList(selector.selectImports(metadata)), false);
          }
          // 判断是否为 ImportBeanDefinitionRegistrar 接口实现类
          else if (checkAssignability(ImportBeanDefinitionRegistrar.class, candidateToCheck)) {
              Class<?> candidateClass = (candidate instanceof Class ? (Class) candidate :
                  this.resourceLoader.getClassLoader().loadClass((String) candidate));
              ImportBeanDefinitionRegistrar registrar =
                  BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
              
              invokeAwareMethods(registrar);
              
              // 调用 ImportBeanDefinitionRegistrar#registerBeanDefinitions 交由它进行需要的 Bean 定义装载
              registrar.registerBeanDefinitions(metadata, this.registry);
          }
          else {
              // 走 @Configuration 装载逻辑，参考上一小节：@Import 引入的 @Configuration 类装载
          }
          ...
      }
  }
  ```

  

- 装载的配置类注册转化为 *BeanDefinition*

  &emsp;&emsp;该处理还是处于【阶段一】即 *ConfigurationClassPostProcessor* 实现接口 *BeanDefinitionRegistryPostProcessor* 的方法

  ```java
  public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
      Set<BeanDefinitionHolder> configCandidates = new LinkedHashSet<BeanDefinitionHolder>();
      
      // 开始解析过滤出的 Bean 定义（具备配置能力的 Bean 定义）
      ConfigurationClassParser parser = new ConfigurationClassParser(
          this.metadataReaderFactory, this.problemReporter, this.environment,
          this.resourceLoader, this.componentScanBeanNameGenerator, registry);
  
          // 解析处理上面有详细说明，故省略...
  
          // 实例化 ConfigurationClassBeanDefinitionReader
      if (this.reader == null) {
          this.reader = new ConfigurationClassBeanDefinitionReader(registry, this.sourceExtractor, this.problemReporter, 
              this.metadataReaderFactory, this.resourceLoader, this.environment, this.importBeanNameGenerator);
      }
      // 获得 parser 得到的配置类集合
      this.reader.loadBeanDefinitions(parser.getConfigurationClasses());
  }
  ```
  
  
  
- @Configuration 类 CGLIB 增强

  &emsp;&emsp;该处理处于【阶段二】即 *ConfigurationClassPostProcessor* 实现接口 BeanFactoryPostProcessor 的方法

  ```java
  // BeanFactoryPostProcessor 接口实现方法【阶段二】
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
      ...
      // 增强处理
      enhanceConfigurationClasses(beanFactory);
  }
  
  // 增强处理方法
  public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
      Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<String, AbstractBeanDefinition>();
      
      // 拿到所有 Bean 定义，筛选为 Full 模式的 Bean 定义，进行增强处理	
      for (String beanName : beanFactory.getBeanDefinitionNames()) {
          BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
          if (ConfigurationClassUtils.isFullConfigurationClass(beanDef)) {
              ...
              // 符合条件，放入待处理 Map
              configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
          }
      }
      // 配置类增强器
      ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer(beanFactory);
      for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
          AbstractBeanDefinition beanDef = entry.getValue();
          try {
              Class<?> configClass = beanDef.resolveBeanClass(this.beanClassLoader);
              
              // 进行增强处理
              Class<?> enhancedClass = enhancer.enhance(configClass);
              if (configClass != enhancedClass) {
                  beanDef.setBeanClass(enhancedClass);
              }
          }
          ...
      }
  }
  ```
  
  &emsp;&emsp;Spring 对 *@Configuration* 增强所采用的增强器为 ***ConfigurationClassEnhancer***：
  
  ```java
  class ConfigurationClassEnhancer {
      ...
      // 创建增强器
      private Enhancer newEnhancer(Class<?> configSuperClass, @Nullable ClassLoader classLoader) {
          Enhancer enhancer = new Enhancer();
          enhancer.setSuperclass(configSuperClass);
          // 注意：生成的代理类只实现了 EnhancedConfiguration 接口, 而 Spring AOP 利用 CGLIB 时，代理类会实现 SpringProxy 接口
          enhancer.setInterfaces(new Class<?>[] {EnhancedConfiguration.class});
          enhancer.setUseFactory(false);
          enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
          enhancer.setStrategy(new BeanFactoryAwareGeneratorStrategy(classLoader));
          enhancer.setCallbackFilter(CALLBACK_FILTER);
          enhancer.setCallbackTypes(CALLBACK_FILTER.getCallbackTypes());
          return enhancer;
      }
      ...
  }
  ```
  
  &emsp;&emsp;Spring 对 *@Configuration* 注解类的增强虽然也利用了 CGLIB 技术，但它与 Spring AOP 区别在于，它没有实现标识接口 ***SpringProxy***。虽然二者最终的代理类的名称都会出现 **EnchancerBySpringCGLIB**，但通过 *ConfigurationClassEnhancer* 增强的类，在判断是否是 Spring AOP 代理类时，返回结果为 `false`，例如：调用方法 `AopUtils.isAopProxy(Object)`。

&emsp;&emsp;&emsp;综上所述， **@Enable 模块驱动实现原理** 的**主要过程**已经全部描述完毕，其主要由 ***ConfigurationClassPostProcessor*** 负责筛选 @Component 类、@Configuration 类、@Bean 方法类的 **Bean 定义（*BeanDefinition*）**，然后通过 ***ConfigurationClassParser*** 从候选的 Bean 定义中解析出 ***ConfigurationClass*** 集合，随后被 ***ConfigurationClassBeanDefinitionReader*** 转化并注册为 *BeanDefinition*，以上过程由【阶段一】*ConfigurationClassPostProcessor* 实现接口 BeanDefinitionRegistryPostProcessor 的方法完成。接下来把所有符合 Full 模式的 BeanDefinition **进行 CGLIB 增强**，此过程由【阶段二】*ConfigurationClassPostProcessor* 实现接口 BeanFactoryPostProcessor 的方法完成。



### Spring Web 自动装配

&emsp;&emsp;Spring Framework 3.1 里程碑的意义不仅提供 “模块驱动” 的能力，还具备 “Web 自动装配” 的能力。有别于 Spring Boot 的 “自动装配”，Spring Framework 支持的自动装配仅限于 Web 应用场景，而且需要依赖 Servlet 3.0+ 容器。

#### 理解 Web 自动装配

&emsp;&emsp;Servlet 3.0 引入 ***ServletContainerInitializer*** 接口来支持编程的方式替换传统的 `web.xml` 文件来初始化 Servlet 上下文。Spring 借助此特性引入 **WebApplicationInitializer** 接口进行 Spring Web 自动装配，从而替换之前在 `web.xml` 中配置 ***ContextLoaderListener*** 、***DispatcherServlet*** 的方式。



&emsp;&emsp;XML 的方式添加 *DispatcherServlet*：

```xml
<web-app>
 <!-- 利用 ContextLoaderListener 对 Root ApplicationContext 初始化-->
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
 
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/dispatcher-servlet-context.xml</param-value>
  </context-param>
 	
  <!-- 向 ServletContext 添加 DispatcherServlet 从而获得 Child ApplicationContext -->
  <servlet>
    <servlet-name>dispatcher-servlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value></param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>
 
  <servlet-mapping>
    <servlet-name>dispatcher-servlet</servlet-name>
    <url-pattern>/*</url-pattern>
  </servlet-mapping>
 
</web-app>
```

&emsp;&emsp;自定义 WebApplicationInitializer 接口实现类的方式添加 *DispatcherServlet*：

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {
  @Override
  public void onStartup(ServeletContext container) {
    XmlWebApplicationContext appContext = new XmlWebApplicationContext();
		appContext.setConfigLocation("/WEB-INF/dispatcher-servlet-context.xml");
    
    ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
    registration.setLoadOnStartup(1);
    registration.addMapping("/*");
  }
}
```

&emsp;&emsp;WebApplicationInitializer 的实现类能够被 Servlet 3.0 容器 ***ServletContainerInitializer*** 接口的 **SPI** 实现类 ***SpringServletContainerInitializer*** 自动检测并调用，从而自动化装配整个 Spring Web。为了减轻开发人员的开发成本，Spring Framework 还提供了两个简化实现方案：

- ***AbstractDispatcherServletInitializer*** （Spring XML 配置驱动）
- ***AbstractAnnotationConfigDispatcherServletInitializer*** （Spring Java 代码配置驱动）

参考资料：[ContextLoaderListener vs DispatcherServlet](https://howtodoinjava.com/spring-mvc/contextloaderlistener-vs-dispatcherservlet/)

#### 自定义 Web 自动装配

&emsp;&emsp;下面通过实现 ***AbstractAnnotationConfigDispatcherServletInitializer*** 抽象类来自定义实现 Spring Web MVC 自动装配：

```java
public class SpringWebMvcServletInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[0];
    }

    @Override
    // DispatcherServlet 配置Bean
    protected Class<?>[] getServletConfigClasses() {
        return of(SpringWebMvcConfiguration.class);
    }

    @Override
    // DispatcherServlet URL Pattern 映射
    protected String[] getServletMappings() {
        return of("/*");
    }

    // 便利 API ，减少 new T[] 代码
    private static <T> T[] of(T... values) {
        return values;
    }
}
```

&emsp;&emsp;对比直接实现接口 *WebApplicationInitializer*，继承抽象类能使得自定义 Web 自动装配的开发成本进一步降低，仅需覆盖一小部分方法，大部分模板式的代码都被抽象类的默认方法完成。具体 Web 自动装配是如何通过 Servelet 容器一步步引导至 *WebApplicationInitializer* 进而装配整个 Spring Web MVC，将在下面的原理小节讲述。

> [HelloWorldController 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/tree/master/spring-framework-samples/spring-webmvc-3.2.x-sample)



#### Web 自动装配原理

&emsp;&emsp;Spring Framework 并不具备 “Web 自动装配” 原生能力，而是借助 **Servlet 3.0** 技术的两大特性：***ServletContext* 配置方法** 及 **运行时插拔**。

- ServletContext 配置方法

  &emsp;&emsp;Servlet 3.0 开始引入 *ServletContext* 配置方法达到以编程的方式动态地装配 *Servlet*、*Filter*、*Listener*，替换之前必须使用部署描述文件 `web.xml` 进行配置，增加了运行时配置的弹性。配置方法在 *ServletContext* 中以 ***add*** 开头的方法。

  

- 运行时插拔

  &emsp;&emsp;上面的配置方法所属的类 *ServletContext* 实例会被容器两处回调暴露给开发者，分别为 ***ServletContextListener* 的 `contextInitialized()` 方法** 以及 ***ServletContainerInitializer* 的 `onStartup()` 方法**。

  &emsp;&emsp;*ServletContextListener* 主要用作监听 *ServletContext* 的生命周期事件，包含 “初始化” 和 “销毁” 两个事件，其中的 `contextInitialized()` 就是 ServletContext 初始化时被触发执行的方法，它的参数为 ServletContextEvent 类型，包含 ServletContext 实例对象。通过实现 ServletContextListener 监听器接口的该初始化方法，达到执行配置方法的目的。

  &emsp;&emsp;*ServletContainerInitializer* 被容器启动时回调 `onStartup(Set<Class>, ServletContext)`，第一个参数可以由注解 *@HandlesType* 指定关心的类型，所指定的类型的子类（含抽象类）将成为该 Set 集合。容器启动时通过 ***SPI*** 技术查找 *ServletContainerInitializer* 服务接口的实现类并进行调用，而 Spring Framework 对该接口的实现类为 ***SpringServletContainerInitializer***，参见：

  

  &emsp;&emsp;spring-web Jar 包中 `META-INF/services/javax.servlet.ServletContainerInitializer` 里的实现类信息

  ```properties
  org.springframework.web.SpringServletContainerInitializer
  ```

  &emsp;&emsp;查看此类的代码：

  ```java
  // 过滤出 WebApplicationInitializer 接口的实现类
  @HandlesTypes(WebApplicationInitializer.class)
  public class SpringServletContainerInitializer implements ServletContainerInitializer {
      // 被 Servlet 容器调用的方法，第一个参数为所有的 WebApplicationInitializer 实现类集合
      public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
              throws ServletException {
  
          List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();
  
          if (webAppInitializerClasses != null) {
              for (Class<?> waiClass : webAppInitializerClasses) {
                  // 进一步筛选出 非接口、非抽象的 WebApplicationInitializer 子类
                  if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
                          WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
                      ...
                      initializers.add((WebApplicationInitializer) waiClass.newInstance());
                      ...
                  }
              }
          }
  		...
          // 排序
          AnnotationAwareOrderComparator.sort(initializers);
          servletContext.log("Spring WebApplicationInitializers detected on classpath: " + initializers);
          // 根据顺序依次调用实现类的 onStartup 方法
          for (WebApplicationInitializer initializer : initializers) {
              initializer.onStartup(servletContext);
          }
      }
  }
  ```
  

&emsp;&emsp;Spring Framework 3.1 时没有提供 *WebApplicationInitializer* 接口的任何实现，完全交给开发人员实现，而 Spring Framework 3.2 提供了三种抽象实现类以减少此接口的实现成本，三者继承关系为：

```
  AbstractContextLoaderInitializer
      |- AbstractDispatcherServletInitializer
          |- AbstractAnnotationConfigDispatcherServletInitializer
```

&emsp;&emsp;三个抽象类的使用场景为：

| 抽象类                                                 | 使用场景                                                     |
  | ------------------------------------------------------ | ------------------------------------------------------------ |
  | *AbstractContextLoaderInitializer*                     | 构建 Web Root 应用上下文，替代 `web.xml` 注册 *ContextLoaderListener* |
  | *AbstractDispatcherServletInitializer*                 | 替代 `web.xml` 注册 *DispatcherServlet*，必要时创建 Web Root 应用上下文 |
  | *AbstractAnnotationConfigDispatcherServletInitializer* | 具备注解配置驱动能力的 *AbstractDispatcherServletInitializer* |

&emsp;&emsp;接下来依次分析它们的装配原理，并分析它们继承扩展后都在哪些方面做了优化调整来达到简化开发成本。

- AbstractContextLoaderInitializer 装配原理


&emsp;&emsp;传统 Servlet 应用场景， Spring Web MVC 的 **Root** ***WebApplicationContext*** 由 ***ContextLoaderListener*** 装载：

```xml
  <listener>
      <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
```

&emsp;&emsp;此监听器是标准的 *ServletContextListener* 实现，当 Web 应用启动时 Servlet 容器会调用它的 `contextInitialized(ServletContextEvent)` 方法，该方法在 ***ContextLoaderListener*** 实现类里，会调用初始化 **Root *WebApplicationContext***。

&emsp;&emsp;当 Web 应用运行在 Servlet 3.0+ 环境时，可以简单通过继承 `spring-web` 中的抽象 *AbstractContextLoaderInitializer* 类，由它的默认方法向 *ServletContext* 添加 *ContextLoaderListener* 监听器来实现初始化 **Root *WebApplicationContext*** 的过程。所以此抽象类是帮助简化初始化 Root *WebApplicationContext*，如果开发人员想使用 Spring Web MVC，那么得完全手动完成核心前端控制器 *DispatcherServlet* 的注册，否则只能使用常规的 Servlet 进行 Web 开发。

```java
public abstract class AbstractContextLoaderInitializer implements WebApplicationInitializer {
    // 在此接口的实现方法上调用注册 ContextLoaderListener 监听器
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        registerContextLoaderListener(servletContext);
    }

    // 在默认实现中，先初始化 Root WebApplicationContext, 后注册监听器
    protected void registerContextLoaderListener(ServletContext servletContext) {
        WebApplicationContext rootAppContext = createRootApplicationContext();
        if (rootAppContext != null) {
            servletContext.addListener(new ContextLoaderListener(rootAppContext));
        }
        ...
    }

    // 初始化根应用上下文的具体实现交由开发人员处理
    protected abstract WebApplicationContext createRootApplicationContext();
}
```

- AbstractDispatcherServletInitializer 装配原理


&emsp;&emsp;为了使用 Spring Web MVC 框架，Spring Framework 在 `spring-webmvc` 给我们提供了 *AbstractContextLoaderInitializer* 的抽象子类 ***AbstractDispatcherServletInitializer*** 默认注册核心前端控制器 ***DispatcherServlet*** ：

```java
public abstract class AbstractDispatcherServletInitializer extends AbstractContextLoaderInitializer {
    
    public static final String DEFAULT_SERVLET_NAME = "dispatcher";

    // 覆盖父类方法
    public void onStartup(ServletContext servletContext) throws ServletException {
        // 先调用父类方法，初始化 Root WebApplicationContext
        super.onStartup(servletContext);
        // 然后注册 Spring Web MVC 核心控制器
        this.registerDispatcherServlet(servletContext);
    }

    // 此模板方法帮助开发人员注册 DispatcherServlet
    // 并提供模板方法方便开发人员扩展诸如：创建子 WebApplicationContext、监听器注册 等关键过程
    protected void registerDispatcherServlet(ServletContext servletContext) {
        String servletName = this.getServletName();
        // 创建子 WebApplicationContext 上下文
        WebApplicationContext servletAppContext = this.createServletApplicationContext();
        DispatcherServlet dispatcherServlet = new DispatcherServlet(servletAppContext);
        Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
        registration.setLoadOnStartup(1);
        registration.addMapping(this.getServletMappings());
        registration.setAsyncSupported(this.isAsyncSupported());
        Filter[] filters = this.getServletFilters();
        if (!ObjectUtils.isEmpty(filters)) {
              ...
        }
        this.customizeRegistration(registration);
    }
    ...
}
```

- AbstractAnnotationConfigDispatcherServletInitializer 装配原理


&emsp;&emsp;在针对上面抽象类 *AbstractDispatcherServletInitializer* 的实现中，开发人员还需完整实现创建 **Root *WebApplicationContext*** 的方法 **`createRootApplicationContext()`** 及创建子上下文 ***DispatcherServlet WebApplicationContext*** 的方法 **`createServletApplicationContext()`**，对 Spring Web MVC 上下文层次理解不深的开发人员，实现成本还是比较高，故在 Spring Framework 3.2  提供了基于注解驱动的抽象类 ***AbstractAnnotationConfigDispatcherServletInitializer***，进一步简化开发：

```java
public abstract class AbstractAnnotationConfigDispatcherServletInitializer extends AbstractDispatcherServletInitializer {
    public AbstractAnnotationConfigDispatcherServletInitializer() {
    }

    // Root WebApplicationContext 的创建已经实现，仅仅留下根上下文的配置类需要开发人员覆盖指定
    protected WebApplicationContext createRootApplicationContext() {
        Class<?>[] configClasses = this.getRootConfigClasses();
        if (!ObjectUtils.isEmpty(configClasses)) {
            // 创建基于注解驱动的上下文
            AnnotationConfigWebApplicationContext rootAppContext = new AnnotationConfigWebApplicationContext();
            rootAppContext.register(configClasses);
            return rootAppContext;
        } else {
            return null;
        }
    }

    // Servlet WebApplicationContext 的创建也已实现，仅仅留下上下文的配置需开发人员覆盖指定
    protected WebApplicationContext createServletApplicationContext() {
        // 创建基于注解驱动的上下文
        AnnotationConfigWebApplicationContext servletAppContext = new AnnotationConfigWebApplicationContext();
        Class<?>[] configClasses = this.getServletConfigClasses();
        if (!ObjectUtils.isEmpty(configClasses)) {
            servletAppContext.register(configClasses);
        }

        return servletAppContext;
    }

    // 根应用上下文配置类交由开发人员指定
    protected abstract Class<?>[] getRootConfigClasses();

    // Servlet 子应用上下文配置类交由开发人员指定
    protected abstract Class<?>[] getServletConfigClasses();
}
```

&emsp;&emsp;可以看到父子上下文默认都已创建为 *AnnotationConfigWebApplicationContext*，开发人员仅仅只需关注为两上下文指定配置类的操作。从全局视角看，***SpringServletContainerInitializer*** 通过实现 **Servlet 3.0 SPI** 接口 ***ServletContainerInitializer***，与 ***@HandlesTypes*** 配合过滤出 ***WebApplicationInitializer*** 具体类的集合，随后顺序迭代执行该集合的元素，进而利用 **Servlet 3.0 在类 *ServletContext* 中添加的配置方法 API** 实现 Web 自动装配的目的。在 Spring Framework 3.2 后更是利用 ***AbstractAnnotationConfigDispatcherServletInitializer*** 极大的简化了注解驱动开发的成本。

### Spring 条件装配

&emsp;&emsp;同一应用不同环境中所依赖的资源或表现的行为可能存在差异，大致实现手段分为两大类：编译时差异化、运行时配置化。Spring Framework 选择后者，利用设置环境变量或 Java 系统属性。

#### 理解配置条件装配

&emsp;&emsp;根据场景不同采用不同静态配置方式（XML 文件或注解）的装配方式，称为 “配置条件装配”。Spring Framework 允许设置两种类型配置：**有效配置（Active Profile）**、**默认配置（Default Profile）**。当有效配置不存在时，采用默认配置。同时，Spring 支持两种设置配置的方式：***ConfigurableEnvironment* API 编码配置**、**Java 系统属性配置**。

| 设置类型             | ConfigurableEnvironment API 配置 | Java 系统属性配置       |
| -------------------- | -------------------------------- | ----------------------- |
| 设置 Active Profile  | setActiveProfiles(String...)     | spring.profiles.active  |
| 添加 Active Profile  | addActiveProfiles(String)        |                         |
| 设置 Default Profile | setDefaultProfiles(String...)    | spring.profiles.default |

&emsp;&emsp;`spring.profiles.active` 对应的常量变量名为：`AbstractEnvironment.ACTIVE_PROFILES_PROPERTY_NAME`

&emsp;&emsp;`spring.profiles.default` 对应的常量变量名为：`AbstractEnvironment.DEFAULT_PROFILES_PROPERTY_NAME`

#### 自定义配置条件装配

1. 在所需设置配置条件的类上增加 *@Profile* 注解

   ```java
   @Service
   // 此处指定 Java8 的配置标签
   @Profile("Java8")
   public class LambdaCalculatingService implements CalculatingService {
   
       @Override
       public Integer sum(Integer... values) {
           int sum =  Stream.of(values).reduce(0, Integer::sum);
           System.out.printf("[Java 8 Lambda实现] %s 累加结果 : %d\n", Arrays.asList(values), sum);
           return sum;
       }
   }
   ```

2. 在环境变量或 Java 系统属性中指定有效配置

   ```java
   @Configuration
   @ComponentScan(basePackageClasses = CalculatingService.class)
   public class CalculatingServiceBootstrap {
       static {
           // 通过 Java 系统属性设置 Spring Profile
           // 以下语句等效于 ConfigurableEnvironment.setActiveProfiles("Java8")
           System.setProperty(AbstractEnvironment.ACTIVE_PROFILES_PROPERTY_NAME, "Java8");
           // 以下语句等效于 ConfigurableEnvironment.setDefaultProfiles("Java7")
           System.setProperty(AbstractEnvironment.DEFAULT_PROFILES_PROPERTY_NAME, "Java7");
       }
   }
   ```


> [CalculatingServiceBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-3.2.x-sample/src/main/java/thinking/in/spring/boot/samples/spring3/bootstrap/CalculatingServiceBootstrap.java)



#### 配置条件装配原理

&emsp;&emsp;按照 Bean 注册方式来划分，一个类被注册成为容器 Bean 通过**注解驱动方式**、**编程方式（上下文直接注册）**、**XML 配置方式**。Spring 对这几种方式都增加了对 *@Profile* 或 xml 元素属性 *profile* 的条件判断处理，从而支持配置条件装配。

1. *@Profile* 条件装配原理

   &emsp;&emsp;注解驱动方式、编程方式这两种方式其实都是对 ***@Profile*** 注解解析后比较环境或 Java 系统属性中是否包含指定配置。这其中包含**注解扫描处理 *ClassPathScanningCandidateComponentProvider***、**配置型组件解析器 *ConfigurationClassParser***、**上下文直接注册方法 *AnnotatedBeanDefinitionReader*** 三个类对 ***@Profile*** 注解的解析并判断的处理逻辑：

   ```java
   // 获取类中 Profile 注解的信息
   AnnotationAttributes profile = MetadataUtils.attributesFor(metadata, Profile.class);
   // 根据当前环境配置检测是否包含注解中指定的配置
   return this.environment.acceptsProfiles(profile.getStringArray("value"));
   ```

2. &lt;beans profile="..."&gt; 条件装配原理

   &emsp;&emsp;XML 配置驱动方式相比较注解方式而言，只是解析方式改为由 ***DefaultBeanDefinitionDocumentReader*** 类解析 XML 中的 profile 属性配置，除此之外与上面的判断并无不同：

   ```java
   public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {
       
       public static final String PROFILE_ATTRIBUTE = "profile";
   
       protected void doRegisterBeanDefinitions(Element root) {
           // 从 XML 元素中解析 profile 元素配置
           String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
           if (StringUtils.hasText(profileSpec)) {
               String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                       profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
               // 根据当前环境配置检测是否包含注解中指定的配置
               if (!getEnvironment().acceptsProfiles(specifiedProfiles)) {
                   return;
               }
           }
           ...
       }
   }
   ```

- *@Conditional* 条件装配

  &emsp;&emsp;***@Conditional*** 条件装配是 **Spring Framework 4.0** 引入的新特性，它与 “配置条件装配” 一样，都是选择性加载匹配的 Bean，但 *@Conditional* 条件装配弹性更大。因为 “配置条件装配” 偏向于**静态激活和配置**，而 *@Conditional* 条件装配更关注**运行时动态选择**。

  ```java
  /**
   * 从 4.0 开始引入
   *
   * @see Condition
   * @since 4.0
   */
  @Target({ElementType.TYPE, ElementType.METHOD})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  public @interface Conditional {
      // 条件判断逻辑在 Condition 接口实现类中
      // 允许指定多个条件实现类，当且仅当所有条件都满足才匹配
      Class<? extends Condition>[] value();
  }
  ```

  &emsp;&emsp;*Condition* 接口：

  ```java
  public interface Condition {
      /*
       * 具体条件判断方法
       * context
       *	包含 BeanDefinitionRegistry ConfigurableListableBeanFactory Envrionment ResourceLoader ClassLoader 信息
       * metadata
       *	包含当前注解所在的元注解信息
       */
      boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
  }
  ```

  1. 自定义 *@Conditioinal* 条件装配

     &emsp;&emsp;下面通过派生 @Conditional 注解来自定义条件装配，具体步骤如下：

     - 编写派生注解

       ```java
       @Target({ElementType.METHOD}) // 定义此注解只标注在方法上
       @Retention(RetentionPolicy.RUNTIME)
       @Documented
       @Conditional(OnSystemPropertyCondition.class) // 指定条件判断类为 OnSystemPropertyCondition
       public @interface ConditionalOnSystemProperty {
       
           /**
            * @return System 属性名称
            */
           String name();
       
           /**
            * @return System 属性值
            */
           String value();
       }
       ```

     - 实现条件匹配判断实现类

       ```java
       public class OnSystemPropertyCondition implements Condition {
       
           @Override
           public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
               // 获取 ConditionalOnSystemProperty 所有的属性方法值
               MultiValueMap<String, Object> attributes = metadata.getAllAnnotationAttributes(ConditionalOnSystemProperty.class.getName());
               // 获取 ConditionalOnSystemProperty#name() 方法值（单值）
               String propertyName = (String) attributes.getFirst("name");
               // 获取 ConditionalOnSystemProperty#value() 方法值（单值）
               String propertyValue = (String) attributes.getFirst("value");
               // 获取 系统属性值
               String systemPropertyValue = System.getProperty(propertyName);
               // 比较 系统属性值 与 ConditionalOnSystemProperty#value() 方法值 是否相等
               if (Objects.equals(systemPropertyValue, propertyValue)) {
                   System.out.printf("系统属性[名称 : %s] 找到匹配值 : %s\n",propertyName,propertyValue);
                   return true;
               }
               return false;
           }
       }
       ```

     - Bean 配置添加自定义条件注解

       ```java
       @Configuration
       public class ConditionalMessageConfiguration {
       
           @ConditionalOnSystemProperty(name = "language", value = "Chinese")
           @Bean("message") // Bean 名称 "message" 的中文消息
           public String chineseMessage() {
               return "你好，世界";
           }
       
           @ConditionalOnSystemProperty(name = "language", value = "English")
           @Bean("message") // Bean 名称 "message" 的英文消息
           public String englishMessage() {
               return "Hello,World";
           }
       }
       ```

     - 测试运行

       ```java
       public class ConditionalOnSystemPropertyBootstrap {
       
           public static void main(String[] args) {
               // 设置 System Property  language = Chinese
               System.setProperty("language", "Chinese");
               // 构建 Annotation 配置驱动 Spring 上下文
               AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
               // 注册 配置Bean ConditionalMessageConfiguration 到 Spring 上下文
               context.register(ConditionalMessageConfiguration.class);
               // 启动上下文
               context.refresh();
               // 获取名称为 "message" Bean 对象
               String message = context.getBean("message", String.class);
               // 输出 message 内容
               System.out.printf("\"message\" Bean 对象 : %s\n", message);
               
               context.close();
           }
       }
       ```

     > [ConditionalOnSystemPropertyBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-4.3.x-sample/src/main/java/thinking/in/spring/boot/samples/spring4/bootstrap/ConditionalOnSystemPropertyBootstrap.java)

  2. *@Conditional* 条件装配原理

     &emsp;&emsp;*@Conditional* 条件装配注解所标注的类是否进行装载被一个通用的**条件鉴别器 *ConditionEvaluator*** 来进行评估。像上一节讲的 *@Profile* 及 XML 配置中的 *profile* 属性也在 **Spring Framework 4.0** 引入 *@Conditional* 后进行了重构，*@Profile* 注解增加 *@Conditional* 元注解，并指明条件判断逻辑类为 *ProfileCondition*。重构后使得 **“配置条件” 装载** 和 *@Conditioanl* 条件装载都统一交给 *ConditionEvaluator* 进行处理：

     ```java
     class ConditionEvaluator {
     
         public boolean shouldSkip(AnnotatedTypeMetadata metadata, ConfigurationPhase phase) {
             if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
                 return false;
             }
     
             if (phase == null) {
                 if (metadata instanceof AnnotationMetadata &&
                         ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
                     // 配置类解析阶段
                     return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
                 }
                 // Bean 注册阶段
                 return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
             }
     
             List<Condition> conditions = new ArrayList<Condition>();
             for (String[] conditionClasses : getConditionClasses(metadata)) {
                 for (String conditionClass : conditionClasses) {
                     Condition condition = getCondition(conditionClass, this.context.getClassLoader());
                     conditions.add(condition);
                 }
             }
     
             AnnotationAwareOrderComparator.sort(conditions);
             // 遍历所有 @Conditional 中设置的 Condition 接口实现类，依次进行判断
             for (Condition condition : conditions) {
                 ConfigurationPhase requiredPhase = null;
                 if (condition instanceof ConfigurationCondition) {
                     requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
                 }
                 // 当条件在符合的阶段才会进行条件判断
                 if (requiredPhase == null || requiredPhase == phase) {
                     // 一旦有条件不符即跳过装载
                     if (!condition.matches(this.context, metadata)) {
                         return true;
                     }
                 }
             }
     
             return false;
         }
     }
     ```


&emsp;&emsp;至此 Spring Framework 已经走向注解驱动编程，但在组件自动化装配上还不尽如人意，例如：@Enable 模块驱动还需显示注解到配置类上且需 *@Import* 注解配合、Web 自动装配还需依赖 Servlet 3.0+ 容器支持。还无法真正做到 Spring 应用自我驱动，可能由于这些种种限制才会促成 Spring Boot 的诞生。因为正是为了解决 Spring Framework 中这些限制才会出现自动装配和嵌入式 Web 容器技术在 Spring 生态中得以应用，而由此产生的新产品就是 Spring Boot。下一小节将详细阐述 Spring Boot 是如何实现真正意义上的自动装配。

## Spring Boot 自动装配

&emsp;&emsp;Spring Framework 时代，借助 *@Import*、*@ComponentScan* 的能力来实现 *@Component*、*@Configuration* 组件的装配。由于应用依赖 JAR 存在变化的可能导致组件所在的包路径的不确定，如果要实现应用所有组件自动装配，使用 *@ComponentScan* 扫描应用 *默认包（Default Package，即 Java 中未声明包路径）* 路径或许会是一种可行的办法，但是事实真是如此吗？

### 理解 @ComponentScan 默认包

&emsp;&emsp;如何使得 *@ComponentScan* 扫描默认包呢，有人可能会这样设置：

```java
// 使用空字符串来表示默认包路径
@ComponentScan(basePackages = "")
```

&emsp;&emsp;实际上，使用空字符串 `""` 时，**Spring Framework 会忽略空串设置，而采用被 *@ComponentScan* 标注的类做在的包路径**：

```java
class ComponentScanAnnotationParser {
	...

    public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
		...
        Set<String> basePackages = new LinkedHashSet<>();
        String[] basePackagesArray = componentScan.getStringArray("basePackages");
        for (String pkg : basePackagesArray) {
            // pkg = "" 时，StringUtils.tokenizeToStringArray 会进行忽略
            String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
                    ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
            Collections.addAll(basePackages, tokenized);
        }
        for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
            basePackages.add(ClassUtils.getPackageName(clazz));
        }
        // 扫描包集合还是为空，添加 @ComponentScan 注解所在类的包路径为扫描路径
        if (basePackages.isEmpty()) {
            basePackages.add(ClassUtils.getPackageName(declaringClass));
        }
		...
    }
}
```

&emsp;&emsp;忽略空串的关键处理类：

```java
public abstract class StringUtils {
    // ComponentScanAnnotationParser 调用此方法获取包路径数组
    public static String[] tokenizeToStringArray(@Nullable String str, String delimiters) {
        // 委托此方法处理
        return tokenizeToStringArray(str, delimiters, true, true);
    }

    // 真正处理方法
    public static String[] tokenizeToStringArray(
            @Nullable String str, String delimiters, boolean trimTokens, boolean ignoreEmptyTokens) {

        if (str == null) {
            return new String[0];
        }

        StringTokenizer st = new StringTokenizer(str, delimiters);
        List<String> tokens = new ArrayList<>();
        while (st.hasMoreTokens()) {
            // token = ""
            String token = st.nextToken();
            if (trimTokens) {
                token = token.trim();
            }
            // token = "" 不满足条件，不会添加到列表中
            if (!ignoreEmptyTokens || token.length() > 0) {
                tokens.add(token);
            }
        }
        // 导致返回空列表
        return toStringArray(tokens);
    }
}
```

&emsp;&emsp;所以，要让 `basePackages = ""` 成为默认包路径，只需将 *@ComponentScan* 所标注的类放入默认包路径，即：

```java
// 该类没有 package 包声明语句
@ComponentScan(basePackages = "")
// 另一设置默认包路径方式
//@ComponentScan(basePackageClasses = DefaultPackageBootstrap.class)
public class DefaultPackageBootstrap {
}
```

&emsp;&emsp;当 *DefaultPackageBootstrap* 在默认包路径时，上面两种设置方式都能使得 *@ComponentScan* 扫描默认包下所有组件。但运行时会发现装配异常，所以进行全部包的扫描的手段来实现自动装配在 Spring 中是不可行的。在自动装配时，**它需要一种综合性技术手段来重新深度整合 Spring 注解编程模型、@Enable 模块驱动及条件装配等 Spring 原生特性，这种技术就是 “Spring Boot 自动装配”。**

>[DefaultPackageBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-framework-samples/spring-framework-4.3.x-sample/src/main/java/DefaultPackageBootstrap.java)

### 理解 Spring Boot 自动装配

&emsp;&emsp;Spring Boot 应用默认使用 *@SpringBootApplication* 注解就能实现诸多特性，而自动装配就是其中特性之一。不难发现该注解是一个组合注解，其中包含的元注解 *@EnableAutoConfiguration* 就是用来激活自动装配。

* 理解 *@EnableAutoConfiguration*

  - *@EnableAutoConfiguration* 也属于 *@Enable* 模块装配的实现

    ```java
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @AutoConfigurationPackage
    @Import(AutoConfigurationImportSelector.class)
    public @interface EnableAutoConfiguration {
      ...
    }
    ```

  - *@EnableAutoConfiguration* 标注在能被应用上下文注册的类时即可激活自动装配（Spring Framework 4.0+）

    &emsp;&emsp;根据 *@Enable* 模块驱动章节可知，Spring Framework 4.0 开始支持对含 *@Import* 注解的类进行配置解析，所以 *@EnableAutoConfiguration* 不依赖 *@SpringBootConfiguration* 或 *@Configuration* 的标注。

    

* 优雅地替换自动装配

  &emsp;&emsp;自动装配的组件非侵占性，开发者可以自定义组件覆盖那些被自动装配的组件。由此说明自动装配的组件优先级最低，具体如何实现参考原理小节部分。

  

* 失效自动装配

  &emsp;&emsp;除替换自动装配组件外，还可以指定失效某个自动装配。Spring Boot 提供两种失效手段：

  - 代码配置方式
    - 配置类型安全的属性方法（指定排除的类）：`@EnableAutoConfiguration.exclude()`
    - 配置排除类名的属性方法（指定排除的类全限定名）：`@EnableAutoConfiguration.excludeName()`
  - 外部化配置方式
    - 配置属性：`spring.autoconfigure.exclude`

  &emsp;&emsp;具体这两种方式如何运作实现，参考原理小节部分。

  

### Spring Boot 自动装配原理

&emsp;&emsp;既然 Spring Boot 自动装配也遵照 *@Enable* 模块驱动的实现逻辑，那么将重点关注其 *@Import* 导入的类 ***AutoConfigurationImportSelector***：

```java
// 实现 DeferredImportSelector 接口，而此接口继承 ImportSelector
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
        BeanFactoryAware, EnvironmentAware, Ordered {
	...

    // ImportSelector 接口方法实现
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        // 1. 加载所有自动装配元信息 META-INF/spring-autoconfigure-metadata.properties
        AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
                .loadMetadata(this.beanClassLoader);
        // 2. 获取当前标注类的所有注解方法属性信息
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        // 3. 获取所有候选的自动装配配置类列表 META-INF/spring.factories 中 EnableAutoConfiguration.class 的属性值列表
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
                attributes);
        // 4. 去除重复
        configurations = removeDuplicates(configurations);
        // 5. 获得需要排除的自动装配配置类集合
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        // 6. 失效被排除的自动装配类
        configurations.removeAll(exclusions);
        // 7. 根据过滤条件进一步排除不需要自动装配类
        configurations = filter(configurations, autoConfigurationMetadata);
        // 8. 触发自动装配导入事件
        fireAutoConfigurationImportEvents(configurations, exclusions);
        // 9. 返回最后需要自动装配的类全限定名称数组
        return StringUtils.toStringArray(configurations);
    }
    
    ...
}
```

&emsp;&emsp;观察 `selectImports` 方法，**步骤 3** 可以了解候选装配组件的获取来源，**步骤 5、6** 可以解答上一节的失效自动装配。但没能找到为何能优雅的使用自定义组件替换自动装配组件、不同自动装配模块间的依赖先后次序等问题。要解答这些疑问，还得分析配置类处理器 ***ConfigurationClassPostProcessor*** 导入 *ImportSelector* 的子接口 ***DeferredImportSelector*** 的具体过程。而 ***DeferredImportSelector* 是 Spring Framework 4.0 才开始引入**，这也间接证明 Spring Boot 初始版本至少要依赖 Spring Framework 4.0，因为从该版本起才能借助 *DeferredImportSelector* 实现 Spring Boot 的自动装配特性，而本原理分析**将基于 Spring Framework 5.0 版本**。

* *ConfigurationClassPostProcessor* 识别含 *@EnableAutoConfiguration* 的配置类

  ```java
  public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
          PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
  
      // 从 Bean 定义注册器中筛选出候补的配置类
      public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
          List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
          // 1. 拿到已注册的 Bean 定义名称数组
          String[] candidateNames = registry.getBeanDefinitionNames();
          for (String beanName : candidateNames) {
              BeanDefinition beanDef = registry.getBeanDefinition(beanName);
              if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
                      ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
                  ...
              }
              // 2. 判断当前 Bean 定义是否是配置类（符合 Full 或 Lite 模式，Spring Framework 5.0 已经包含甄别 @Import）
              //      @EnableAutoConfiguration 包含 @Import 所以被选中
              else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
                  configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
              }
          }
          ...
  
          // 3. 构建配置类解析器
          ConfigurationClassParser parser = new ConfigurationClassParser(
                  this.metadataReaderFactory, this.problemReporter, this.environment,
                  this.resourceLoader, this.componentScanBeanNameGenerator, registry);
  
          Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
          Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
          do {
              // 4. 解析器对 @EnableAutoConfiguration 所标注的类进行解析操作
              parser.parse(candidates);
              parser.validate();
              ...
          }
          while (!candidates.isEmpty());
          
          ...
      }
  }
  ```

  

* *ConfigurationClassParser* 解析含 *@EnableAutoConfiguration* 的配置类，处理其中的 *DeferredImportSelector* 导入类

  &emsp;&emsp;解析收集 *DeferredImportSelector* 的封装类 ***DeferredImportSelectorHolder*** 列表：

  ```java
  class ConfigurationClassParser {
      // 【方法 1】
      public void parse(Set<BeanDefinitionHolder> configCandidates) {
          // 先声明一个列表，用来收集解析过程中的 DeferredImportSelector 实现类
          this.deferredImportSelectors = new LinkedList<>();
          // 遍历候选配置类，进行解析
          for (BeanDefinitionHolder holder : configCandidates) {
              BeanDefinition bd = holder.getBeanDefinition();
              ...
              // 此处的解析逻辑最终都转到【方法 2】 
              if (bd instanceof AnnotatedBeanDefinition) {
                  parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
              } else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                  parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
              } else {
                  parse(bd.getBeanClassName(), holder.getBeanName());
              }
              ...
          }
          // 处理收集到的 DeferredImportSelector 实现类，参考【方法 4】
          processDeferredImportSelectors();
      }
  
      // 【方法 2】
      protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass) {
          // 解析其他注解的方法在此省略
          ...
          // 由于 DeferredImportSelector 实现类被 @Import 引入，着重观察对该注解的解析操作
          // 【注意】ImportSelector 契约就是被 @Import 引入，脱落它的 ImportSelector 不会被生命周期回调
          processImports(configClass, sourceClass, getImports(sourceClass), true);
  		...
      }
  
      // 【方法 3】
      private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
                                  Collection<SourceClass> importCandidates, boolean checkForCircularImports) {
  		...
          // 配置类上所有导入类集合
          for (SourceClass candidate : importCandidates) {
              // 获取导入类型为 ImportSelector 的类
              if (candidate.isAssignable(ImportSelector.class)) {
                  Class<?> candidateClass = candidate.loadClass();
                  ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                  ParserStrategyUtils.invokeAwareMethods(
                          selector, this.environment, this.resourceLoader, this.registry);
                  // 进一步如果是 DeferredImportSelector 类型，封装后放入 deferredImportSelectors 集合，延迟到【方法 4】再处理
                  if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
                      this.deferredImportSelectors.add(
                              new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                  } else {
                      String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                      Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                      // 非 DeferredImportSelector 类型，直接进行导入处理（说明 DeferredImportSelector 更低的优先级，为自定义组件的覆盖做基础）
                      processImports(configClass, currentSourceClass, importSourceClasses, false);
                  }
              }
  			...
          }
  
      }
  ```
  
  &emsp;&emsp;将收集到的 ***DeferredImportSelectorHolder*** 进行分组：
  
```java
  class ConfigurationClassParser {
    // 【方法 1】
      public void parse(Set<BeanDefinitionHolder> configCandidates) {
          // 1. 先声明一个列表，用来收集解析过程中的 DeferredImportSelector 实现类
          this.deferredImportSelectors = new LinkedList<>();
          ...
          // 处理收集到的 DeferredImportSelector 实现类，参考【方法 4】
          processDeferredImportSelectors();
      }
  
      // 【方法 4】
      private void processDeferredImportSelectors() {
          List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
          // 不同 DeferredImportSelector 间进行排序
          deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
          // 根据 group 对不同的 DeferredImportSelector 进行分组封装为 DeferredImportSelectorGrouping 组
          // 相同 group 中的 DeferredImportSelectorGrouping 对象，允许包含多个 DeferredImportSelectorHolder
          Map<Object, DeferredImportSelectorGrouping> groupings = new LinkedHashMap<>();
          // 配置类元数据 -> 配置类 的索引 Map
          Map<AnnotationMetadata, ConfigurationClass> configurationClasses = new HashMap<>();
          for (DeferredImportSelectorHolder deferredImport : deferredImports) {
              // 按 DeferredImportSelector 定义的 group 类别，对不同的 DeferredImportSelector 进行分组封装 
              Class<? extends Group> group = deferredImport.getImportSelector().getImportGroup();
              // 分组 Map 中不存在当前 group 时，创建分组，并拿到分组对象，存在则直接拿到分组对象
              DeferredImportSelectorGrouping grouping = groupings.computeIfAbsent(
                      (group == null ? deferredImport : group),
                      (key) -> new DeferredImportSelectorGrouping(createGroup(group)));
              // 将当前 deferredImport 追加到分组对象中
              grouping.add(deferredImport);
              // 收集 配置类元数据 -> 配置类 的索引
              configurationClasses.put(deferredImport.getConfigurationClass().getMetadata(),
                      deferredImport.getConfigurationClass());
          }
          // 一次对每个 DeferredImportSelectorGrouping 进行操作
          for (DeferredImportSelectorGrouping grouping : groupings.values()) {
              // 委托 DeferredImportSelectorGrouping 获取 DeferredImportSelector 的导入对象列表，遍历进行导入
              // *******【方法 5】grouping.getImports() 是筛选、排序等业务逻辑的关键 ******
              grouping.getImports().forEach((entry) -> {
                  ConfigurationClass configurationClass = configurationClasses.get(
                          entry.getMetadata());
                  try {
                      // 依次进行导入注册 Bean 定义
                      processImports(configurationClass, asSourceClass(configurationClass),
                              asSourceClasses(entry.getImportClassName()), false);
                  }
  				...
              });
          }
      }
  
      // 内部类
      private static class DeferredImportSelectorGrouping {
  
          private final DeferredImportSelector.Group group;
          private final List<DeferredImportSelectorHolder> deferredImports = new ArrayList<>();
  
          // 分组的组对象
          DeferredImportSelectorGrouping(Group group) {
              this.group = group;
          }
  
          // 将解析的到的 DeferredImportSelectorHolder 放入分组中的列表
          public void add(DeferredImportSelectorHolder deferredImport) {
              this.deferredImports.add(deferredImport);
          }
  
          // 【方法 5】
          public Iterable<Group.Entry> getImports() {
              // 遍历该分组中的所有 DeferredImportSelectorHolder
              for (DeferredImportSelectorHolder deferredImport : this.deferredImports) {
                  // 委托组对象 group 处理 DeferredImportSelectorHolder
                  this.group.process(deferredImport.getConfigurationClass().getMetadata(),
                          deferredImport.getImportSelector());
              }
              // 将 group 处理完毕的组实体返回给【方法 4】进行最后导入
              return this.group.selectImports();
          }
      }
  }
```

  &emsp;&emsp;由【方法 5】可知，解析收集的 ***DeferredImportSelectorHolder*** 交由 ***DeferredImportSelector.Group*** 接口实现类进行处理，而自动装配导入类 *AutoConfigurationImportSelector* 是 *DeferredImportSelector* 实现类，前者覆盖了接口默认方法 `getImportGroup()` 返回 *DeferredImportSelector.Group* 接口实现类为 ***AutoConfigurationGroup***：

  ```java
public class AutoConfigurationImportSelector
          implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
          BeanFactoryAware, EnvironmentAware, Ordered {
      ...
      @Override
      public Class<? extends Group> getImportGroup() {
          // 指定 DeferredImportSelector.Group 的实现类为 AutoConfigurationGroup
          return AutoConfigurationGroup.class;
      }
      ...
  }
  ```

  &emsp;&emsp;那么下一小节将着重分析 ***AutoConfigurationGroup*** 对 *AutoConfigurationImportSelector* 引入的候选自动装配类的**装载**、**筛选**、**排序**等操作。

  

* *AutoConfigurationGroup* 排序及装载候选组件

  &emsp;&emsp;由上节可知，接口 *DeferredImportSelector* 的实现类 ***AutoConfigurationImportSelector*** 导入的配置类将交由 ***AutoConfigurationGroup*** 进行处理，该类实现了接口 *DeferredImportSelector.Group* 的两个方法：`process(AnnotationMetadata, DeferredImportSelector)`、`selectImports()`，前者对指定的 *DeferredImportSelector* 所导入的类进行处理，然后交由后者组装成可顺序迭代的 ***DeferredImportSelector.Group.Entry*** 列表进行最终的导入处理。

  &emsp;&emsp;先来看 `process` 方法：

  ```java
  private static class AutoConfigurationGroup implements DeferredImportSelector.Group,
          BeanClassLoaderAware, BeanFactoryAware, ResourceLoaderAware {
      // 导入配置类名称为 key, @EnableAutoConfiguration 所在配置类的元注解为 value 的索引缓存对象    
      private final Map<String, AnnotationMetadata> entries = new LinkedHashMap<>();
  
      @Override
      public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
          // 执行 ImportSelector 接口的 selectImports 方法拿到导入配置类数组
          String[] imports = deferredImportSelector.selectImports(annotationMetadata);
          for (String importClassName : imports) {
              // 建立导入配置类名称及元注解信息的索引
              this.entries.put(importClassName, annotationMetadata);
          }
      }
  }
  ```

  &emsp;&emsp;可以看到 *AutoConfigurationGroup* 首先通过 *DeferredImportSelector* 来获取所有由它引入的候选自动配置类，该方法在小节开头已经展示过，不过在本原理小节将会更深入理解其中重要的步骤的具体实现：

  ```java
  // 实现 DeferredImportSelector 接口，而此接口继承 ImportSelector
  public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
          BeanFactoryAware, EnvironmentAware, Ordered {
  	...
      // ImportSelector 接口方法实现
      @Override
      public String[] selectImports(AnnotationMetadata annotationMetadata) {
          if (!isEnabled(annotationMetadata)) {
              return NO_IMPORTS;
          }
          // 1. 加载所有自动装配元信息 META-INF/spring-autoconfigure-metadata.properties
          AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
                  .loadMetadata(this.beanClassLoader);
          // 获取当前标注类的所有注解方法属性信息
          AnnotationAttributes attributes = getAttributes(annotationMetadata);
          // 2. 获取所有候选的自动装配配置类列表 META-INF/spring.factories 中 EnableAutoConfiguration.class 的属性值列表
          List<String> configurations = getCandidateConfigurations(annotationMetadata,
                  attributes);
          // 去除重复
          configurations = removeDuplicates(configurations);
          // 3. 获得需要排除的自动装配配置类集合
          Set<String> exclusions = getExclusions(annotationMetadata, attributes);
          checkExcludedClasses(configurations, exclusions);
          // 失效被排除的自动装配类
          configurations.removeAll(exclusions);
          // 4. 根据过滤条件进一步排除不需要自动装配类
          configurations = filter(configurations, autoConfigurationMetadata);
          // 5. 触发自动装配导入事件
          fireAutoConfigurationImportEvents(configurations, exclusions);
          // 返回最后需要自动装配的类全限定名称数组
          return StringUtils.toStringArray(configurations);
      }
    ...
  }
  ```

  &emsp;&emsp;将按照代码注释中的编号顺序，来分析最终得到需要自动装配的配置类数组的过程：

  

  1. 读取自动装配元数据

     &emsp;&emsp;此步骤读取的自动装配元数据对象 *AutoConfigurationMetadata* 将为后面过滤、排序提供元数据信息。Spring Boot 并没有采取 **ASM** 技术去获取所有自动装配类的元注解信息，而是将元注解信息提取出来汇总成 `spring-autoconfigure-metadata.properties` 文件。这样做一方面性能更优，另一方面使得第三方自定义扩展变得灵活，只需将自定义自动装配的元数据放入所在 JAR 包路径的相同名称的文件中即可。

     ```java
     final class AutoConfigurationMetadataLoader {
     
         protected static final String PATH = "META-INF/spring-autoconfigure-metadata.properties";
     
         // 1. 从 classpath 路径中加载 META-INF/spring-autoconfigure-metadata.properties，封装为 AutoConfigurationMetadata
         public static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader) {
             return loadMetadata(classLoader, PATH);
         }
     
         static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader, String path) {
             ...
             Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(path)
                     : ClassLoader.getSystemResources(path));
             Properties properties = new Properties();
             while (urls.hasMoreElements()) {
                 // 2. 将 classpath 路径中多个 spring-autoconfigure-metadata.properties 整合为单个 properties
                 properties.putAll(PropertiesLoaderUtils.loadProperties(new UrlResource(urls.nextElement())));
             }
             // 3. 将整合的 properties 封装为 AutoConfigurationMetadata
             return loadMetadata(properties);
         }
     
         // 4. 目前封装的 AutoConfigurationMetadata 实现类为 PropertiesAutoConfigurationMetadata
         static AutoConfigurationMetadata loadMetadata(Properties properties) {
             return new PropertiesAutoConfigurationMetadata(properties);
         }
     
         // 5. 该类提供对 properties 里的数据进行读取并转换类型操作
         private static class PropertiesAutoConfigurationMetadata implements AutoConfigurationMetadata {
     
             private final Properties properties;
     
             PropertiesAutoConfigurationMetadata(Properties properties) {
                 this.properties = properties;
             }
     		...
             // 为后续获取某个自动装配类的 Order 排序值
             @Override
             public Integer getInteger(String className, String key) {
                 return getInteger(className, key, null);
             }
             // 1. 为后续获取某个自动装配类排序需要用到的相对配置类集合, 例如：
             //		getSet("A", "AutoConfigureBefore")，表示 A 必须在哪些配置类之前装载 
             //		getSet("A", "AutoConfigureAfter")，表示 A 必须在哪些配置类之后装载
             // 2. 为后续自动装配类是否要被排除的条件
             //		getSet("A", "ConditionalOnClass"), 表示 A 必须在哪些类存在的情况下才自动装配
             @Override
             public Set<String> getSet(String className, String key) {
                 return getSet(className, key, null);
             }
             ...
         }
     }
     ```

     &emsp;&emsp;这个属性文件内容是包含很多以**自动装配候选类全限定名**后追加**元注解属性方法名**的键值对，如下所示：

     ```properties
     ...
     org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration.AutoConfigureAfter=org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration
     org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveDataAutoConfiguration.Configuration=
     org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveDataAutoConfiguration.ConditionalOnClass=com.datastax.driver.core.Cluster,org.springframework.data.cassandra.core.ReactiveCassandraTemplate,reactor.core.publisher.Flux
     org.springframework.boot.autoconfigure.data.solr.SolrRepositoriesAutoConfiguration.ConditionalOnClass=org.apache.solr.client.solrj.SolrClient,org.springframework.data.solr.repository.SolrRepository
     ...
     ```

     

  2. 读取候选装配组件

     &emsp;&emsp;Spring Boot 所有候选自动装配配置类都以 **工厂加载机制（Factory Loading Mechanism）**放入到 `META-INF/spring.factories` 文件中，这些配置类都配置在键名为 ***org.springframework.boot.autoconfigure.EnableAutoConfiguration*** 下，详情可参考 [理解自动配置机制](https://reionchan.github.io/2022/06/18/thinking-in-springboot-part-1/#%E7%90%86%E8%A7%A3%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE%E6%9C%BA%E5%88%B6)。该文件内容类似：

     ```properties
     org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
     org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
     org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
     org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
     ...
     ```

     

  3. 排除自动装配组件

     &emsp;&emsp;在前面理解小节得知，失效某些自动装配有两种方式：编码方式及配置方式。具体实现方式如下代码所示：

     ```java
     public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
             BeanFactoryAware, EnvironmentAware, Ordered {
     
         // 1. 从元注解中提取 @EnableAutoConfiguration 注解的属性方法信息  
         protected AnnotationAttributes getAttributes(AnnotationMetadata metadata) {
             // 注意：这一步很关键，元注解中可能有很多注解，这里只获取名称为 @EnableAutoConfiguration 的注解的属性方法
             //      防止有别的注解也有同名的 exclude 或 excludeName 属性
             String name = getAnnotationClass().getName();
             AnnotationAttributes attributes = AnnotationAttributes
                     .fromMap(metadata.getAnnotationAttributes(name, true));
             return attributes;
         }
         // 2. 从元注解中只提取注解 @EnableAutoConfiguration 的属性方法信息
         protected Class<?> getAnnotationClass() {
             return EnableAutoConfiguration.class;
         }
     
         // 3. 获得排除自动装配类集合，参数 attributes 即为步骤 1 返回的 @EnableAutoConfiguration 属性方法信息
         protected Set<String> getExclusions(AnnotationMetadata metadata,
                                             AnnotationAttributes attributes) {
             Set<String> excluded = new LinkedHashSet<>();
             // 获取 exclude 属性方法中需排除的配置类，已转成类的全限定名字符串列表
             excluded.addAll(asList(attributes, "exclude"));
             // 获取 exclude 属性方法中需排除的配置类全限定名字符串列表
             excluded.addAll(Arrays.asList(attributes.getStringArray("excludeName")));
             // 获取环境变量中排除类全限定名列表
             excluded.addAll(getExcludeAutoConfigurationsProperty());
             // 将三者列表汇总返回
             return excluded;
         }
     
         // 4. 读取外部化配置
         private List<String> getExcludeAutoConfigurationsProperty() {
             if (getEnvironment() instanceof ConfigurableEnvironment) {
                 Binder binder = Binder.get(getEnvironment());
                 // PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE = "spring.autoconfigure.exclude"
                 // 从外部环境变量中获取该参数设置的排除自动装配配置类全限定名列表
                 return binder.bind(PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE, String[].class)
                         .map(Arrays::asList).orElse(Collections.emptyList());
             }
             String[] excludes = getEnvironment()
                     .getProperty(PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE, String[].class);
             return (excludes != null ? Arrays.asList(excludes) : Collections.emptyList());
         }
     }
     ```

     &emsp;&emsp;在获得排除的候选自动配置类列表后，执行 **`configurations.removeAll(exclusions)`** 方法从自动装载候选列表中剔除这些配置类。接下来将从自动装配元数据的条件，进行条件化装配从而进一步从候选列表中过滤一些配置类。

     

  4. 过滤自动装配组件

     &emsp;&emsp;经过前面手动指定剔除后，将由 `filter(configurations, autoConfigurationMetadata)` 方法进行自动条件化过滤：

     ```java
     public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
             BeanFactoryAware, EnvironmentAware, Ordered {
         // 过滤方法，需借助第一步加载的 autoConfigurationMetadata 元数据
         private List<String> filter(List<String> configurations,
                                     AutoConfigurationMetadata autoConfigurationMetadata) {
             String[] candidates = StringUtils.toStringArray(configurations);
             boolean[] skip = new boolean[candidates.length];
             boolean skipped = false;
             // 1. 获得自动配置导入过滤器列表，依次执行过滤方法
             for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
                 invokeAwareMethods(filter);
                 // 过滤器的 match 方法，返回每个候选类是否符合加载要求的 布尔数组
                 boolean[] match = filter.match(candidates, autoConfigurationMetadata);
                 for (int i = 0; i < match.length; i++) {
                     if (!match[i]) {
                         skip[i] = true;
                         skipped = true;
                     }
                 }
             }
             if (!skipped) {
                 return configurations;
             }
             List<String> result = new ArrayList<>(candidates.length);
             for (int i = 0; i < candidates.length; i++) {
                 if (!skip[i]) {
                     // 当且仅当不需跳过的的才加入过滤后的列表
                     result.add(candidates[i]);
                 }
             }
             // 返回过滤后的候选配置类
             return new ArrayList<>(result);
         }
         // 2. Spring 工厂机制加载 AutoConfigurationImportFilter 的过滤器配置
         protected List<AutoConfigurationImportFilter> getAutoConfigurationImportFilters() {
             return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class, this.beanClassLoader);
         }
     }
     ```

     &emsp;&emsp;查看自动装配项目 `spring-boot-autoconfigure` JAR 包中 `META-INF/spring.factories` 文件中声明注册的过滤器：

     ```properties
     # Auto Configuration Import Filters
     org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
     org.springframework.boot.autoconfigure.condition.OnClassCondition
     ```

     &emsp;&emsp;可以看到目前仅注册了一个过滤器实现 ***OnClassCondition***：

     ```java
     @Order(Ordered.HIGHEST_PRECEDENCE)
     class OnClassCondition extends SpringBootCondition
             implements AutoConfigurationImportFilter, BeanFactoryAware, BeanClassLoaderAware {
     
         @Override
         public boolean[] match(String[] autoConfigurationClasses,
                                AutoConfigurationMetadata autoConfigurationMetadata) {
     		...
             // 1. 委托 getOutcomes 方法获取过滤结果数组 ConditionOutcome[]
             ConditionOutcome[] outcomes = getOutcomes(autoConfigurationClasses, autoConfigurationMetadata);
             boolean[] match = new boolean[outcomes.length];
             for (int i = 0; i < outcomes.length; i++) {
                 // 某个类在过滤结果为空或匹配为真即需要自动装配
                 match[i] = (outcomes[i] == null || outcomes[i].isMatch());
     			...
             }
             // 将结果数组返回
             return match;
         }
     
         // 2. 对候选配置类集合进行条件判断断定获得结果数组
         private ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
                                                AutoConfigurationMetadata autoConfigurationMetadata) {
             // 分成两半
             int split = autoConfigurationClasses.length / 2;
             // 前半部分交给另外线程处理，其实也是对内部类 StandardOutcomesResolver 处理器的异步封装
             OutcomesResolver firstHalfResolver = createOutcomesResolver(
                     autoConfigurationClasses, 0, split, autoConfigurationMetadata);
             // 后半部分由当前线程使用内部类 StandardOutcomesResolver 处理器处理
             OutcomesResolver secondHalfResolver = new StandardOutcomesResolver(
                     autoConfigurationClasses, split, autoConfigurationClasses.length,
                     autoConfigurationMetadata, this.beanClassLoader);
             // 调用 StandardOutcomesResolver 的 resolveOutcomes 方法
             ConditionOutcome[] secondHalf = secondHalfResolver.resolveOutcomes();
             ConditionOutcome[] firstHalf = firstHalfResolver.resolveOutcomes();
             ConditionOutcome[] outcomes = new ConditionOutcome[autoConfigurationClasses.length];
             // 将两部分结果合并返回
             System.arraycopy(firstHalf, 0, outcomes, 0, firstHalf.length);
             System.arraycopy(secondHalf, 0, outcomes, split, secondHalf.length);
             return outcomes;
         }
     
         // 3. 内部类 StandardOutcomesResolver 处理器
         private final class StandardOutcomesResolver implements OutcomesResolver {
             ...
             @Override
             public ConditionOutcome[] resolveOutcomes() {
                 // resolveOutcomes 又交由 getOutcomes 处理
                 return getOutcomes(this.autoConfigurationClasses, this.start, this.end,
                         this.autoConfigurationMetadata);
             }
     
             private ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
                                                    int start, int end, AutoConfigurationMetadata autoConfigurationMetadata) {
                 ConditionOutcome[] outcomes = new ConditionOutcome[end - start];
                 for (int i = start; i < end; i++) {
                     String autoConfigurationClass = autoConfigurationClasses[i];
                     // 4. 从元数据中获取当前候选配置类的 ConditionalOnClass 属性配置集合
                     Set<String> candidates = autoConfigurationMetadata.getSet(autoConfigurationClass, "ConditionalOnClass");
                     if (candidates != null) {
                         // 将属性方法的类集合继续交给下面的方法 5 处理
                         outcomes[i - start] = getOutcome(candidates);
                     }
                 }
                 return outcomes;
             }
             // 5. 将当前配置类所需依赖的类结合传入此方法处理
             private ConditionOutcome getOutcome(Set<String> candidates) {
     			...
                 // 依赖类集合交给外部类的 getMatches 方法做判断，返回没找到来的类名称集合
                 List<String> missing = getMatches(candidates, MatchType.MISSING, this.beanClassLoader);
                 // 一旦候选配置类依赖的类集合有任意一个没找到，即返回条件未满足结果
                 if (!missing.isEmpty()) {
                     return ConditionOutcome.noMatch(
                             ConditionMessage.forCondition(ConditionalOnClass.class)
                                     .didNotFind("required class", "required classes")
                                     .items(Style.QUOTE, missing));
                 }
     			...
                 return null;
             }
         }
         // 6. 外部类的 getMatches 方法进行处理
         private List<String> getMatches(Collection<String> candidates, MatchType matchType, ClassLoader classLoader) {
             List<String> matches = new ArrayList<>(candidates.size());
             for (String candidate : candidates) {
                 // 遍历每个依赖的类，交给内部枚举类 MatchType.MISSING 进行匹配
                 if (matchType.matches(candidate, classLoader)) {
                     matches.add(candidate);
                 }
             }
             return matches;
         }
         // 内部枚举类
         private enum MatchType {
     
             PRESENT {
                 ...
             },
     
             MISSING {
                 // 交由此 MISSING 覆盖方法处理
                 @Override
                 public boolean matches(String className, ClassLoader classLoader) {
                     // 7. 判断是否依赖的类
                     return !isPresent(className, classLoader);
                 }
             };
             // 8. 通过类装载机制判断依赖的类是否存在
             private static boolean isPresent(String className, ClassLoader classLoader) {
                 if (classLoader == null) {
                     classLoader = ClassUtils.getDefaultClassLoader();
                 }
                 try {
                     forName(className, classLoader);
                     return true;
                 }
                 catch (Throwable ex) {
                     return false;
                 }
             }
         }
     }
     ```

     &emsp;&emsp;由上面八个方法的调用可知，判断某个自动装载配置类是否要被过滤，就看其所配置的元数据属性 **ConditionalOnClass** 设置的依赖类是否在当前应用中存在，如果不存在那么该自动装载配置类将被过滤不进行装载。至此候选装配类列表仅存在需要被自动装配的配置类了，接下来在返回给 ***AutoConfigurationGroup*** 进行排序之前，还将 **需要被装配的配置类** 和 **手动剔除的配置类** 封装成自动装配事件交由监听器处理。

     

  5. 触发自动装配事件

     &emsp;&emsp;回到 *AutoConfigurationImportSelector* 类的方法 `selectImports`，其中调用的触发事件方法：

     ```java
     public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
             BeanFactoryAware, EnvironmentAware, Ordered {
         // 触发自动装配事件
         private void fireAutoConfigurationImportEvents(List<String> configurations,
                                                        Set<String> exclusions) {
             // Spring 工厂加载机制获取 AutoConfigurationImportListener 监听器列表
             List<AutoConfigurationImportListener> listeners = getAutoConfigurationImportListeners();
             if (!listeners.isEmpty()) {
                 AutoConfigurationImportEvent event = new AutoConfigurationImportEvent(this, configurations, exclusions);
                 // 依次触发调用监听器的 onAutoConfigurationImportEvent 方法
                 for (AutoConfigurationImportListener listener : listeners) {
                     invokeAwareMethods(listener);
                     listener.onAutoConfigurationImportEvent(event);
                 }
             }
         }
         // 获取监听器列表
         protected List<AutoConfigurationImportListener> getAutoConfigurationImportListeners() {
             return SpringFactoriesLoader.loadFactories(AutoConfigurationImportListener.class,this.beanClassLoader);
         }
     }
     ```

     &emsp;&emsp;配置文件显示监听器只配置了 ***ConditionEvaluationReportAutoConfigurationImportListener*** 用来记录自动装载报告使用，具体就不深究。不过由于是基于工厂机制加载，Spring Boot 也算留给开发人员进行自定义监听器的扩展点，不过封装的事件中的所有自动装配类的数组都被封装只读的数据结构，所以不能通过自定义监听器对自动装配做任何修改处理。

     ```properties
     # Auto Configuration Import Listeners
     org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
     org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener
     ```

  &emsp;&emsp;执行完事件处理后，*AutoConfigurationGroup* 委托 *AutoConfigurationImportSelector* 类获取所需导入的候选配置类名数组已经完成，并将它们封装成候选类名称为索引，注解元数据位值的 *LinkedHashMap* 成员变量  `entries`，然后调用方法 `selectImports()` 进行排序操作：

  ```java
  private static class AutoConfigurationGroup implements DeferredImportSelector.Group,
          BeanClassLoaderAware, BeanFactoryAware, ResourceLoaderAware {
      // 导入配置类名称为 key, @EnableAutoConfiguration 所在配置类的元注解为 value 的索引缓存对象    
      private final Map<String, AnnotationMetadata> entries = new LinkedHashMap<>();
  
      @Override
      public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
          // 1. 委托 AutoConfigurationImportSelector 类获取所需导入的候选配置类名数组完成
          String[] imports = deferredImportSelector.selectImports(annotationMetadata);
          for (String importClassName : imports) {
              // 2. 建立导入配置类名称及元注解信息的索引
              this.entries.put(importClassName, annotationMetadata);
          }
      }
  
      // 3. 执行该方法完成最终排序，并将排序后的候选配置类封装成 Iterable<Entry> 给 ConfigurationClassParser 进行解析注册
      @Override
      public Iterable<Entry> selectImports() {
          // 利用 sortAutoConfigurations 方法排序，然后封装成可迭代 Iterable<Entry> 对象返回
          return sortAutoConfigurations().stream()
                  .map((importClassName) -> new Entry(this.entries.get(importClassName),
                          importClassName))
                  .collect(Collectors.toList());
      }
      // 4. 排序方法
      private List<String> sortAutoConfigurations() {
          List<String> autoConfigurations = new ArrayList<>(this.entries.keySet());
          
          ...
  
          // 此处又重新读取了一遍自动配置元数据文件，即 META-INF/spring-autoconfigure-metadata.properties
          // 里面包含排序的信息：	
          //      绝对排序的属性 AutoConfigureOrder
          //      相对排序的 AutoConfigureAfter、AutoConfigureBefore
          AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
          // 交由 AutoConfigurationSorter 提取排序元信息做最后的排序操作
          return new AutoConfigurationSorter(getMetadataReaderFactory(),
                  autoConfigurationMetadata).getInPriorityOrder(autoConfigurations);
      }
  }
  ```

  &emsp;&emsp;接下来关注最后处理进行排序的类 *AutoConfigurationSorter* 的排序方法 `getInPriorityOrder`：

  ```java
  class AutoConfigurationSorter {
  	...
      // 5. 排序逻辑
      public List<String> getInPriorityOrder(Collection<String> classNames) {
          // 将待排序候选类、自动装配元数据信息、元数据读取工厂类封装成 classes
          AutoConfigurationClasses classes = new AutoConfigurationClasses(
                  this.metadataReaderFactory, this.autoConfigurationMetadata, classNames);
          List<String> orderedClassNames = new ArrayList<>(classNames);
          // 5.1 先按类全限定名进行字母顺序排序
          Collections.sort(orderedClassNames);
          // 5.2 在从自动装配元数据信息提取绝对定位属性 @AutoConfigureOrder 进行绝对位置排序
          orderedClassNames.sort((o1, o2) -> {
              int i1 = classes.get(o1).getOrder();
              int i2 = classes.get(o2).getOrder();
              return Integer.compare(i1, i2);
          });
          // 5.3 最后从自动装配元数据信息提取属性 @AutoConfigureBefore @AutoConfigureAfter 进行相对位置排序
          orderedClassNames = sortByAnnotation(classes, orderedClassNames);
          // 返回排序后的结果列表
          return orderedClassNames;
      }
  }
  ```

  &emsp;&emsp;候选自动装配类全限定名列表先后经过三次排序，由先到后依次为：

  1. **字母排序**

  2. **@AutoConfigureOrder 值绝对排序，数值越小优先级越高**

  3. **@AutoConfigureBefore @AutoConfigureAfter 值相对排序**

     

  &emsp;&emsp;排序完成后将会回到配置类解析器 *ConfigurationClassParser* 按照顺序依次解析每个配置类的 Bean 定义信息，进而完成所有自动配置类的装配。

  

* *@AutoConfigurationPackage* 自动装配 BasePackages 原理

  &emsp;&emsp;在上小节处理过程完成后，*@EnableAutoConfiguration* 中的元注解 *@Import* 的处理逻辑算是完成，细心的读者可能还发现还有另外一个元注解 ***@AutoConfigurationPackage*** 尚未做说明。从类名可知它的作用是自动配置包，那具体配置什么包呢？这些包有何用处呢？先从它的定义分析：

  ```java
  @Target(ElementType.TYPE)
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Inherited
  @Import(AutoConfigurationPackages.Registrar.class)
  public @interface AutoConfigurationPackage {
  }
  ```

  &emsp;&emsp;可以看出它包含 *@Import* 元注解，故也属于配置类型的组件。而 *@Import* 引入的 ***AutoConfigurationPackages.Registrar*** 会被 *ConfigurationClassParser* 解析处理，而该类正好实现了接口 *ImportBeanDefinitionRegistrar*，故会由解析过程中调用其接口方法：

  ```java
  public abstract class AutoConfigurationPackages {
  
      private static final Log logger = LogFactory.getLog(AutoConfigurationPackages.class);
  
      private static final String BEAN = AutoConfigurationPackages.class.getName();
  
  
      public static boolean has(BeanFactory beanFactory) {
          return beanFactory.containsBean(BEAN) && !get(beanFactory).isEmpty();
      }
  
      public static List<String> get(BeanFactory beanFactory) {
          try {
              return beanFactory.getBean(BEAN, BasePackages.class).get();
          }
          catch (NoSuchBeanDefinitionException ex) {
              throw new IllegalStateException(
                      "Unable to retrieve @EnableAutoConfiguration base packages");
          }
      }
  
      // 内部类 Registrar 实现 ImportBeanDefinitionRegistrar 接口
      static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
  
          @Override
          // 1. 由解析器调用该方法
          public void registerBeanDefinitions(AnnotationMetadata metadata,
                                              BeanDefinitionRegistry registry) {
              // 转调外部类的注册方法，将 @AutoConfigurationPackage 所标注的类的原信息构建内部类 PackageImport 的对象
              register(registry, new PackageImport(metadata).getPackageName());
          }
  		...
      }
  
      // 内部类 PackageImport
      private static final class PackageImport {
          private final String packageName;
          PackageImport(AnnotationMetadata metadata) {
              // 2. 可以看到提取出 @AutoConfigurationPackage 所标注的类所在的包路径
              this.packageName = ClassUtils.getPackageName(metadata.getClassName());
          }
  		...
      }
  
      // 3. 把 @AutoConfigurationPackage 所标注的类所在的包路径注册记录到一个特殊的 Bean 中
      //		该 Bean 类型为 BasePackages
      //      该 Bean 名称为 AutoConfigurationPackages.class.getName()
      // 		如果 @AutoConfigurationPackage 被标注在多个类上，那么这些类所在的包名会被记录到此 Bean 中
      public static void register(BeanDefinitionRegistry registry, String... packageNames) {
          if (registry.containsBeanDefinition(BEAN)) {
              BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
              ConstructorArgumentValues constructorArguments = beanDefinition
                      .getConstructorArgumentValues();
              constructorArguments.addIndexedArgumentValue(0,
                      addBasePackages(constructorArguments, packageNames));
          }
          else {
              GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
              beanDefinition.setBeanClass(BasePackages.class);
              beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0,
                      packageNames);
              beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
              registry.registerBeanDefinition(BEAN, beanDefinition);
          }
      }
  }
  ```

  &emsp;&emsp;由此可以解答上面的两个问题了：

  1. *@AutoConfigurationPackage* 具体配置什么包呢？

     答：*@AutoConfigurationPackage* 将**被它注解的类所在的包名**保存到 Spring 容器的一个特殊的 Bean 中，该 Bean 类型为 ***org.springframework.boot.autoconfigure.AutoConfigurationPackages.BasePackages***，Bean 名称为 ***org.springframework.boot.autoconfigure.AutoConfigurationPackages***。

     

  2. *@AutoConfigurationPackage* 配置的这些包有何用处呢？

     答：*@AutoConfigurationPackage* 把被注解的类所在的包名保存在 Spring 容器中，可以方便开发人员在任何地方通过容器获取这些包名。具体用处就看具体利用这些包名做了什么操作，在 Spring Boot JPA 场景中，JPA 会搜索这些包名下的实体类。

### 自定义 Spring Boot 自动装配

&emsp;&emsp;由上面原理章节可以得知，要自定义一个自动装配模块，首先要**定义一个自动装配配置类**，其次将该自动装配配置类配置到 `META-INF/spring.factories` 文件里键名为 **org.springframework.boot.autoconfigure.EnableAutoConfiguration** 中，这样该配置类即可被框架识别并进行自动装配逻辑。那么运用自动装配机制可以被运用在什么应用场景？其实 Spring Boot 运用自动装配机制实现了 [***Spring Boot Starters***](https://www.geeksforgeeks.org/spring-boot-starters/#:~:text=Spring%20Boot%20Starters%20are%20dependency,dependencies%20under%20a%20single%20name.)， 使得开发人员仅需引入某个 *Starter* 就能自动引入该 *Starter* 所需的依赖，从而让开发人员无需花费过多时间进行手动依赖管理。例如，想使用 Spring Data JPA 进行数据库访问，仅需引入 **`spring-boot-starter-data-jpa`** 依赖即可。由于 **Spring Boot Starters** 是运用 Spring Boot 自动装配机制的一个典型应用场景，那么下面讲围绕如何自定义实现一个 Spring Boot Starter 来阐述自定义自动装配详细过程。

* 自动装配 Class 命名潜规则

  &emsp;&emsp;观察 Spring Boot 内建的自动装配 Class 的命名可以发现，都是以 ***AutoConfiguration*** 作为后缀。

* 自动装配 package 命名潜规则

  &emsp;&emsp;观察内建的 `META-INF/spring.factories` 文件里键名为 **org.springframework.boot.autoconfigure.EnableAutoConfiguration** 的自动配置类全限定名，可以发现 package 基本模式为：

  ```
  org.springframework.boot.autoconfigure  -> 根包名
      |- ${module-package}                -> 模块名
          |- *AutoConfiguration           -> 自动装配 Class
          |- ${sub-module-package}        -> 子模块包名
              |- ...
          
  例如：
      org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration
      
      根包名：org.springframework.boot.autoconfigure
      模块名：data.jpa
      自动装配类名：JpaRepositoriesAutoConfiguration
  ```

  &emsp;&emsp;那么按这个规则，可以给出自定义 package 命名规则：

  ```
  ${root-package}                         -> 根包名
      |- autoconfigure                    -> 固定名
          |- ${module-package}            -> 模块名
              |- *AutoConfiguration       -> 自定义自动装配 Class
              |- ${sub-module-package}    -> 子模块包名
                  |- ...
  
  例如：
      org.reion.autoconfigure.example.module.ExampleAutoConfiguration
  
      根包名：org.reion
      固定名：autoconfigure
      模块名：example.module
      自动装配类名：ExampleAutoConfiguration
  ```

* 自定义 Spring Boot Starter 命名规则

  &emsp;&emsp;自定义 *Starter* 模块的命名规则就不是潜规则了，Spring Boot 官方明确要求开发人员采用 **`${module}-spring-boot-starter`** 的形式进行命名，用来区别 Spring Boot 官方内建的 *Starter* 命名规则 **`spring-boot-starter-${module}`**。另外值得注意的是，官方的 *Starter* 将自动装配的代码放在了 `autoconfigure` 模块中，通过 `spring-boot-starter` 模块依赖该模块的方式最终实现完成的 *Starter*。一般较大较复杂的自定义 *Starter* 也推荐效仿这种方式，否则可以将自动装配的代码一并放入 *Starter* 模块中。

* 自定义 Spring Boot Starter

  &emsp;&emsp;遵照前面框架对自动配置类、包名以及 *Starter* 命名规范，实现一个自定义的 *Starter* 项目，名称为 **`formatter-spring-boot-starter`**：

  1.  构建 Maven 模块工程，名为 `formatter-spring-boot-starter`，[pom.xml](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/formatter-spring-boot-starter/pom.xml) 文件如下：

     ```xml
     <?xml version="1.0" encoding="UTF-8"?>
     <project xmlns="http://maven.apache.org/POM/4.0.0"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
         ...
         <groupId>thinking-in-spring-boot</groupId>
         <artifactId>formatter-spring-boot-starter</artifactId>
         <name>《Spring Boot 编程思想》Spring Boot 2.0 示例工程 - 走向自动配置 - 对象格式化 Spring Boot Starter</name>
     
         <dependencies>
             <!-- Spring Boot Starter 基础依赖 -->
             <dependency>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-starter</artifactId>
               	<!-- 务必设置 optional 为 true，使得引入该包的项目不会传递依赖此包的 spring-boot-starter 版本，防止冲突 -->	
                 <optional>true</optional>
             </dependency>
             ...
         </dependencies>
     
     </project>
     ```

  2. 新建格式化接口 *Formatter*

     ```java
     package thinking.in.spring.boot.samples.autoconfigure.formatter;
     
     public interface Formatter {
         /**
          * 格式化操作
          *
          * @param object 待格式化对象
          * @return 返回格式化后的内容
          */
         String format(Object object);
     }
     ```

     &emsp;&emsp;注意包名，根包名 `thinking.in.spring.boot.samples`，模块名 `formatter`

  3. 实现 *Formatter* 接口类 *DefaultFormatter*

     ```java
     package thinking.in.spring.boot.samples.autoconfigure.formatter;
     
     public class DefaultFormatter implements Formatter {
     
         @Override
         public String format(Object object) {
             return String.valueOf(object); // null 安全实现
         }
     }
     ```

  4. 实现 *DefaultFormatter* 自动装配配置类 *FormatterAutoConfiguration*

     ```java
     package thinking.in.spring.boot.samples.autoconfigure.formatter;
     
     ...
     
     @Configuration
     public class FormatterAutoConfiguration {
     
         /**
          * 构建 {@link DefaultFormatter} Bean
          *
          * @return {@link DefaultFormatter}
          */
         @Bean
         public Formatter defaultFormatter() {
             return new DefaultFormatter();
         }
     }
     ```

     &emsp;&emsp;注意类名，遵照命名规范，以 **AutoConfiguration** 后缀。

  5. 在 `META-INF/spring.factories` 资源声明上面定义的自动装配配置类

     ```properties
     # FormatterAutoConfiguration 自动装配声明
     org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
     thinking.in.spring.boot.samples.autoconfigure.formatter.FormatterAutoConfiguration
     ```

  6. 构建打包此 *Starter* 项目

     ```shell
     $ mvn -Dmaven.test.skip -U clean install
     ```

     &emsp;&emsp;执行上面的 Maven 打包安装命令，将自定义的 *Starter* 安装到本地 Maven 库，供后面的测试项目引入。

  7. 新建 Spring Boot 测试项目，并引入此 *Starter* 包依赖，[pom.xml](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/auto-configuration-sample/pom.xml) 定义如下

     ```xml
     <?xml version="1.0" encoding="UTF-8"?>
     <project xmlns="http://maven.apache.org/POM/4.0.0"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
         ...
         <groupId>thinking-in-spring-boot</groupId>
         <artifactId>auto-configuration-sample</artifactId>
         <name>《Spring Boot 编程思想》Spring Boot 2.0 示例工程 - 走向自动配置</name>
     
         <dependencies>
             <!-- Spring Boot 基础依赖 -->
             <dependency>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-starter</artifactId>
             </dependency>
             <!-- 添加 formatter-spring-boot-starter -->
             <dependency>
                 <groupId>thinking-in-spring-boot</groupId>
                 <artifactId>formatter-spring-boot-starter</artifactId>
                 <version>1.0.0-SNAPSHOT</version>
             </dependency>
         </dependencies>
     
     </project>
     ```

  8. 在测试项目建立引导类 [*FormatterBootstrap*](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/auto-configuration-sample/src/main/java/thinking/in/spring/boot/samples/auto/configuration/bootstrap/FormatterBootstrap.java)

     ```java
     @EnableAutoConfiguration
     public class FormatterBootstrap {
     
         public static void main(String[] args) {
             ConfigurableApplicationContext context = new SpringApplicationBuilder(FormatterBootstrap.class)
                     .web(WebApplicationType.NONE)  // 非 Web 应用
                     .run(args);                    // 运行
             // 待格式化对象
             Map<String, Object> data = new HashMap<>();
             data.put("name", "小马哥");
             // 获取 Formatter 来自于 FormatterAutoConfiguration
             Formatter formatter = context.getBean(Formatter.class);
             System.out.printf("formatter.format(data) : %s\n", formatter.format(data))
             //  关闭当前上下文
             context.close();
         }
     }
     ```

  &emsp;&emsp;运行后，可以发现容器中确实已经自动装配了名为 `defaultFormatter` 的格式化 Bean。不过 `formatter-spring-boot-starter` 还比较简单，如果添加不同的格式化方式，根据运行环境的不同动态选择相应的格式化实现类，那么就需要支持**条件化自动装配**，将在下一小节继续完善。

  

  > [formatter-spring-boot-starter 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/tree/master/spring-boot-2.0-samples/formatter-spring-boot-starter)
  >
  > [FormatterBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/auto-configuration-sample/src/main/java/thinking/in/spring/boot/samples/auto/configuration/bootstrap/FormatterBootstrap.java)

### Spring Boot 条件化自动装配

&emsp;&emsp;Spring Boot 实现条件化自动装配，其底层还是利用派生 Spring Framework 引入的 *@Conditional* 条件注解来实现。自动化条件装配的候选类都是被 *@Configuration* 注解的类，这些配置类解析之后被 ***ConfigurationClassBeanDefinitionReader*** 的方法 `loadBeanDefinitions(Set<ConfigurationClass>)` 进行 Bean 定义的注册。在注册中会分别处理 **候选配置类** 和 类中被 ***@Bean* 注解的方法** 上的 ***@Conditional* 派生注解**，将其转交给 ***ConditionEvaluator*** 判断是否满足条件，而该条件评估器底层逻辑就是收集 *@Conditional* 派生注解所指定的 ***Condition*** 接口实现类，统一调用该接口声明的 `matches` 方法，具体可参考 [Spring 条件装配对 *@Conditional* 条件装配原理 的阐述](https://reionchan.github.io/2022/06/19/thinking-in-springboot-part-2/#spring-%E6%9D%A1%E4%BB%B6%E8%A3%85%E9%85%8D)。

&emsp;&emsp;下面是 *ConfigurationClassBeanDefinitionReader* 类判断 **候选配置类** 和 ***@Bean* 注解的方法** 中的条件注解的方法：

```java
class ConfigurationClassBeanDefinitionReader {

	...
    // 条件注解评估器
    private final ConditionEvaluator conditionEvaluator;

    public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
        TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
        for (ConfigurationClass configClass : configurationModel) {
            // 遍历候选配置类列表，对每个配置类注册 BeanDefinition
            loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
        }
    }

    private void loadBeanDefinitionsForConfigurationClass(
            ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
        // 1. 【候选配置类】此处委托 TrackedConditionEvaluator 提取配置类上的所有条件注解，并判断是否满足
        //      TrackedConditionEvaluator 最终还是委托 ConditionEvaluator 进行判断
        if (trackedConditionEvaluator.shouldSkip(configClass)) {
			...
        }
		...
        for (BeanMethod beanMethod : configClass.getBeanMethods()) {
            // 处理 @Bean 注解的方法
            loadBeanDefinitionsForBeanMethod(beanMethod);
        }
		...
    }

    private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
        ConfigurationClass configClass = beanMethod.getConfigurationClass();
        MethodMetadata metadata = beanMethod.getMetadata();
        String methodName = metadata.getMethodName();
        // 2. 【@Bean 注解的方法】委托 ConditionEvaluator 提取方法上的所有条件注解，并判断是否满足
        if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
            configClass.skippedBeanMethods.add(methodName);
            return;
        }
		...
    }
}
```

&emsp;&emsp;Spring Boot 引入和很多派生至 *@Conditional* 的条件注解，这些条件注解使得 Spring Boot 自动装配具备很强的灵活性，下面通过对前面自定义 *Starter* 的不断完善来依次讲解这些派生条件注解。值得注意的是，这些条件注解所指定的条件处理类都继承于 ***SpringBootCondition*** 抽象类，而后者实现 *Condition* 接口。抽象类 *SpringBootCondition* 将接口 *Condition* `matches` 方法转化为对模板方法 **`getMatchOutcome(ConditionContext, AnnotatedTypeMetadata)`** 的调用，而该抽象方法是所有条件处理子类都必须实现的方法。

* Class 条件注解

  &emsp;&emsp;Class 条件注解有一对语义相反的注解：*@ConditionalOnClass*、*@ConditionalOnMissingClass*，分别表达 “当指定类存在时” 和 “当指定类缺失时” 的语义。**值得注意的是 *@ConditionalOnMissingClass* 的 `value` 属性方法返回值在 Spring Boot 1.3+ 之后才调整为 `String[]`，并且在 Spring Boot 1.4+ 之后删除了 `name` 属性方法，运用到自动装配时，要顾及版本兼容问题**。 而 *@ConditionalOnClass* 从 Spring Boot 1.0+ 一直稳定，包含 **`Class<?>[] value()`**、**`String[] name()`** 两个方法，使用**前者时要求依赖的类必须在类路径存在**，使用**后者时要求依赖类的包名在不同版本不能有变动**，有时同一个 *@Bean* 定义可能需要两者配合使用来解决不同版本包名变动的兼容性问题。

  

  &emsp;&emsp;下面重构 `formatter-spring-boot-starter`, 使其检测类路径存在 `com.fasterxml.jackson.databind.ObjectMapper` 类时，装载 *JsonFormatter* 格式化器取代默认的 *DefaultFormatter* 格式化器。

  1. 增加 Jackson 依赖到 [pom.xml](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/formatter-spring-boot-starter/pom.xml)

     ```xml
     ...
       <dependencies>
         ...
         <!-- Jackson 依赖 -->
         <dependency>
           <groupId>com.fasterxml.jackson.core</groupId>
           <artifactId>jackson-databind</artifactId>
           <!-- 注意此处务必设置为 true，避免把该包传递依赖到引入本 starter 的测试项目中 -->
           <optional>true</optional>
         </dependency>
         ...
       </dependencies>
     ...
     ```

     

  2. 新增 [*JsonFormatter*](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/formatter-spring-boot-starter/src/main/java/thinking/in/spring/boot/samples/autoconfigure/formatter/JsonFormatter.java) 格式化器实现类

     ```java
     public class JsonFormatter implements Formatter {
     
         private final ObjectMapper objectMapper;
     
         public JsonFormatter() {
             this.objectMapper = new ObjectMapper();
         }
     
         @Override
         public String format(Object object) {
             try {
                 return objectMapper.writeValueAsString(object);
             } catch (JsonProcessingException e) {
                 // 解析失败返回非法参数异常
                 throw new IllegalArgumentException(e);
             }
         }
     }
     ```

     

  3. 调整自动化装配类，增加 *JsonFormatter* 的 Bean 声明，并整合 Class 条件化注解

     ```java
     @Configuration
     public class FormatterAutoConfiguration {
     
         @Bean
         // 利用 @ConditionalOnMissingClass 指定 ObjectMapper 不存在时，装载默认格式化器 DefaultFormatter
         @ConditionalOnMissingClass(value = "com.fasterxml.jackson.databind.ObjectMapper")
         public Formatter defaultFormatter() {
             return new DefaultFormatter();
         }
     
         @Bean
         // 利用 @ConditionalOnClass 指定 ObjectMapper 存在时，装载 JsonFormatter
         // 这里使用 name 属性是为了与上面对应方便阅读，其实使用 value = ObjectMapper.class 可以避免手动拼写全限定名错误的低级失误
         @ConditionalOnClass(name = "com.fasterxml.jackson.databind.ObjectMapper")
         public Formatter jsonFormatter() {
             return new JsonFormatter();
         }
     }
     ```

     

  4. 调整测试工程 *FormatterBootstrap*，对比在此工程 pom.xml 中不引入与不引入 Jackson 依赖时，格式化器的实现是否会自动变化

     ```java
     @EnableAutoConfiguration
     public class FormatterBootstrap {
     
         public static void main(String[] args) {
             ConfigurableApplicationContext context = new SpringApplicationBuilder(FormatterBootstrap.class)
                     .web(WebApplicationType.NONE)  // 非 Web 应用
                     .run(args);                    // 运行
             // 待格式化对象
             Map<String, Object> data = new HashMap<>();
             data.put("name", "小马哥");
             // 获取 Formatter 来自于 FormatterAutoConfiguration
             Formatter formatter = context.getBean(Formatter.class);
             System.out.printf("%s.format(data) : %s\n", formatter.getClass().getSimpleName(), formatter.format(data))
             //  关闭当前上下文
             context.close();
         }
     }
     ```

     

* Bean 条件注解

  &emsp;&emsp;Bean 条件注解也是成对出现的语义相反的注解：*@ConditionalOnBean*、*@ConditionalOnMissingBean*，分别表达 “指定 Bean 存在时” 和 “指定 Bean 不存在时” 的语义。值得注意的是，它们仅匹配 BeanFactory 中 Bean 的类型和名字，且在实现层面**仅匹配条件判断时应用上下文已处理的 BeanDefinition**。同时，官方强烈建议开发人员仅在自动装配中使用该条件注解。

  &emsp;&emsp;两者的属性方法基本形同，只是在 **Spring Boot 1.2.5** 时，*@ConditionalOnMissingBean* 中新增了 **`ignored`、`ignoredType`** 两个额外方法。下面就以该注解做说明，*@ConditionalOnBean* 同名注解类似只是语义相反：

  ```java
  @Target({ ElementType.TYPE, ElementType.METHOD })
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Conditional(OnBeanCondition.class)
  public @interface ConditionalOnMissingBean {
  
  	// 指定某些类型的 Bean 不存在时
  	Class<?>[] value() default {};
  
  	// 指定某些全限定名字符串类型的 Bean 不存在时
  	String[] type() default {};
  
  	// 指定某些类型的 Bean 被排除或忽略（ConditionalOnMissingBean 独有）
  	Class<?>[] ignored() default {};
  
  	// 指定某些全限定名字符串类型的 Bean 被排除或忽略（ConditionalOnMissingBean 独有）
  	String[] ignoredType() default {};
  
  	// 指定所有 Bean 没有指定的注解标注时
  	Class<? extends Annotation>[] annotation() default {};
  
  	// 指定某些 Bean 名称 不存在时
  	String[] name() default {};
  
  	// 指定上下文搜索策略：当前、父辈、所有，默认所有
  	SearchStrategy search() default SearchStrategy.ALL;
  }
  ```

  

  &emsp;&emsp;&emsp;下面重构 `formatter-spring-boot-starter`, 使其检测类型为 `com.fasterxml.jackson.databind.ObjectMapper` 的 Bean 存在时，装载另一个名为 `objectMapperFormatter` 的 *JsonFormatter* 格式化器，该格式化器构造时使用上下文获取的 *ObjectMapper* 取代自己实例化的 *ObjectMapper* 参数的 Json 格式化器。

  1. 由于上下文要装载包含 *ObjectMapper* 类型的 Bean，需要依赖 `spring-web` 工程，故修改测试工程 [pom.xml](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/auto-configuration-sample/pom.xml) 文件：

     ```xml
     ...
         <dependencies>
           ...
             <!-- Spring Boot Web 依赖 -->
             <dependency>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-starter-web</artifactId>
             </dependency>
           ...
         </dependencies>
     ...
     ```

     

  2. 调整 *JsonFormatter* 构造器，支持外部指定 *ObjectMapper* 传参

     ```java
     public class JsonFormatter implements Formatter {
     
         private final ObjectMapper objectMapper;
         // 自己实例化 ObjectMapper 构造
         public JsonFormatter() {
             this(new ObjectMapper());
         }
         // 上下文获得 ObjectMapper 构造
         public JsonFormatter(ObjectMapper objectMapper) {
             this.objectMapper = objectMapper;
         }
     
         @Override
         public String format(Object object) {
             try {
                 return objectMapper.writeValueAsString(object);
             } catch (JsonProcessingException e) {
                 // 解析失败返回非法参数异常
                 throw new IllegalArgumentException(e);
             }
         }
     }
     ```

     

  3. 调整自动化装配类，添加支持带参数的 `objectMapperFormatter` 格式化器 Bean 定义方法

     ```java
     @Configuration
     public class FormatterAutoConfiguration {
     
         ...
         @Bean
         @ConditionalOnClass(name = "com.fasterxml.jackson.databind.ObjectMapper")
         // 上下文没有 ObjectMapper 类型的 Bean 时，自己实例化构造
         @ConditionalOnMissingBean(type = "com.fasterxml.jackson.databind.ObjectMapper")
         public Formatter jsonFormatter() {
             return new JsonFormatter();
         }
     
         @Bean
         // 上下文有 ObjectMapper 类型的 Bean 时，取带参数的构造
         @ConditionalOnBean(ObjectMapper.class)
         public Formatter objectMapperFormatter(ObjectMapper objectMapper) {
             return new JsonFormatter(objectMapper);
         }
     }
     ```

     

  4. 调整测试工程 *FormatterBootstrap*, 使用 `getBeansOfType` 方法以便于拿到 Formatter 类型的 Bean 的同时也能获取 Bean 名称：

     ```java
     @EnableAutoConfiguration
     public class FormatterBootstrap {
     
         public static void main(String[] args) {
             ConfigurableApplicationContext context = new SpringApplicationBuilder(FormatterBootstrap.class)
                     .web(WebApplicationType.NONE)  // 非 Web 应用
                     .run(args);                    // 运行
             // 待格式化对象
             Map<String, Object> data = new HashMap<>();
             data.put("name", "小马哥");
             // 获取 Formatter 来自于 FormatterAutoConfiguration
             Map<String, Formatter> beans = context.getBeansOfType(Formatter.class);
             if (beans.isEmpty()) { // 如果 Bean 不存在，抛出异常
                 throw new NoSuchBeanDefinitionException(Formatter.class);
             }
             beans.forEach((beanName, formatter) -> {
                 System.out.printf("[Bean name : %s] %s.format(data) : %s\n", beanName, formatter.getClass().getSimpleName(),
                         formatter.format(data));
             });
     
             // 关闭当前上下文
             context.close();
         }
     }
     ```

  

* 属性条件注解

  &emsp;&emsp;属性条件注解只有一个注解 *@ConditionalOnProperty*, 其判断的属性来源于 ***Spring Environment***。在 *Spring Environment* 允许添加不同的 *属性配置源（PropertySource）*来形成最终的 *Environment*，比较典型的配置源有 **Java 系统属性** 和 **系统环境变量**。Spring Boot 对 *Spring Environment* 进行了扩展，另外添加了 `application.properties` 配置源到 *Environment* 中，换言之，只要在这些配置源中任意位置配置的属性，都能被属性条件注解读取。

  

  &emsp;&emsp;下面是注解 *@ConditionalOnProperty* 定义的属性方法：

  ```java
  @Retention(RetentionPolicy.RUNTIME)
  @Target({ ElementType.TYPE, ElementType.METHOD })
  @Documented
  @Conditional(OnPropertyCondition.class)
  public @interface ConditionalOnProperty {
  
      // name 的别名，与 name 不能同时出现
      String[] value() default {};
  
      // 属性前缀名
      String prefix() default "";
  
      // 需要判断的属性名列表，如果设定了前缀，那么每个属性名为：prifix+name[i]
      // 如果属性名含多个单词，使用 ‘-’ 分隔，设置多个属性名，将校验每一个属性名的值是否符合条件, 需结合 matchIfMissing 设置绝对最终结果
      // 	matchIfMissing = true 时：环境中未设置的属性默认为 true, 此时只要所有设置的属性与期望的一致最终结果为 true
      //  matchIfMissing = false 时: 此时即使所有设置的属性与期望值一致，缺失未设置的为 false 导致最终结果为 false
      String[] name() default {};
  
      // 指定属性期望值，不设置该属性时，非 'false' 的属性值都能使条件成立
      // 【特别注意】最好不要设置此属性为 'false', 如果设置，被判断的属性恰好为 ‘false’ 时还符合条件，有点反逻辑
      String havingValue() default "";
  
      // 指定属性在环境中未找到时，条件是否成立，默认不成立
      boolean matchIfMissing() default false;
  }
  ```

  &emsp;&emsp;特别注意 `name`、`havingValue` 属性上的注释说明。

  

  &emsp;&emsp;下面进一步重构 `formatter-spring-boot-starter`, 使其检查运行环境包含属性 `formatter.enabled` 且其值设置为 `true` 时才装载其中的所有配置类。此时需要将条件注解标注在类上即可：

  ```java
  @Configuration
  // prefix = "formatter" 表示：属性前缀为 formatter
  // name = "enabled" 表示：匹配属性名为 formatter.enabled 的属性
  // havingValue = "true" 表示：期望匹配值
  // matchIfMissing = true 表示：当属性配置不存在时，同样视作匹配
  @ConditionalOnProperty(prefix = "formatter", name = "enabled", havingValue = "true", matchIfMissing = true)
  public class FormatterAutoConfiguration {
  	...
  }
  ```

  &emsp;&emsp;可以在测试工程的 `application.properties` 文件添加目标属性进行测试：

  ```properties
  # FormatterAutoConfiguration 属性配置
  formatter.enabled = false
  ```

  &emsp;&emsp;也可以在启动类代码中设置属性参数进行测试，当然两者都配置时，外部配置将覆盖代码配置：

  ```java
  @EnableAutoConfiguration
  public class FormatterBootstrap {
  
      public static void main(String[] args) {
          ConfigurableApplicationContext context = new SpringApplicationBuilder(FormatterBootstrap.class)
                  .web(WebApplicationType.NONE)  
                  .properties("formatter.enabled=true") // 配置默认属性, "=" 前后不能有空格
                  .run(args);
         ...
      }
  }
  ```

  &emsp;&emsp;一般推荐自定义的 *Starter* 都配置此注解，并指定包含一个属性名为 `enabled`，`havingValue="true"` 的开关配置，如果想要默认不配置也能启用，加上 `matchIfMissing = true` 的参数配置。

  

* Resource 条件注解

  &emsp;&emsp;&emsp;Resource 条件注解 *@ConditionalOnResource*, 其语义为 “当指定资源存在时条件成立”。值得注意的是，查找资源是否存是委托给 ***ResourceLoader*** 实例，该实例**默认情况下**追本溯源最终实例对象为 ***DefaultResourceLoader***，不过在 **Spring Framework 4.3+** 或 **Spring Boot 1.4+** 时略有不同。在这个指定的版本之前，*ResourceLoader* 的实例要么是 DefaultResourceLoader 实例，要么是 *DefaultResourceLoader* 的子类即应用上下文 *AbstractApplicationContext* 的实现类，而后者未对父类做特别变动，所以行为与父类 *DefaultResourceLoader* 无差别，故说默认情况下其实就是 *DefaultResourceLoader* 的实例。然后在上述指定版本之后，*DefaultResourceLoader* 增加了新增自定义协议解析器的方法 `addProtocalResoler(ProtocalResolver)`，使得开发人员能借由上下文对象 *AbstractApplicationContext* 动态添加自定义的协议解析器，此时添加自定义解析器后的 *AbstractApplicationContext* 与默认的 *DefaultResourceLoader* 实例行为就有所差别，应予以关注。*ProtocalResolver* 扩展机制的出现也是有价值的，它能够简化复杂的 ***URLStreamHandler* 及 *URLStreamHandlerFactory*** 的扩展机制。*URLStreamHandler* 属于 Java 标准协议扩展机制，在 [运行 Spring Boot 应用](https://reionchan.github.io/2022/06/18/thinking-in-springboot-part-1/#%E8%BF%90%E8%A1%8C-spring-boot-%E5%BA%94%E7%94%A8) 章节中关于 **JarLauncher 实现原理** 描述中知道 Spring Boot 正是通过扩展该机制，实现了对 Jar 的增强处理从而引导 Spring Boot 应用 Jar 包的运行。

   

  &emsp;&emsp;回到 Resource 条件注解 *@ConditionalOnResource* 本身，它的定义如下：

  ```java
  @Target({ ElementType.TYPE, ElementType.METHOD })
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Conditional(OnResourceCondition.class)
  public @interface ConditionalOnResource {
      // 资源路径数组，当所有指定资源都存在时，条件成立
      // 资源路径支持 Spring 占位符
      String[] resources() default {};
  }
  ```

  

  &emsp;&emsp;下面进一步重构 `formatter-spring-boot-starter`, 使用注解 *@ConditionalOnResource* 指定资源 `META-INF/spring.factories` 存在时才自动装配：

  ```java
  @Configuration
  @ConditionalOnProperty(prefix = "formatter", name = "enabled", havingValue = "true", matchIfMissing = true)
  // 指定资源 META-INF/spring.factories
  @ConditionalOnResource(resources = "META-INF/spring.factories")
  public class FormatterAutoConfiguration {
    ...
  }
  ```

  

* Web 应用条件注解

  &emsp;&emsp; Web 应用条件注解包含一对语义相反的注解：*@ConditionalOnWebApplication*、*@ConditionalOnNotWebApplication*，它们的语义分别为 “当前为 Web 应用类型时” 和 “当前为非 Web 应用类型时”。

  

  &emsp;&emsp;*@ConditionalOnWebApplication* 注解定义：

  ```java
  @Target({ ElementType.TYPE, ElementType.METHOD })
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Conditional(OnWebApplicationCondition.class)
  public @interface ConditionalOnWebApplication {
      // Web 类型子类型枚举，默认 ANY 表示任意子类型
      Type type() default Type.ANY;
  
      enum Type {
          // 任意子类型，SERVLET 或 REACTIVE
          ANY,
          // 指定 SERVLET 类型
          SERVLET,
          // 指定 REACTIVE 类型
          REACTIVE
      }
  }
  ```

  &emsp;&emsp;*@ConditionalOnNotWebApplication* 注解定义：

  ```java
  @Target({ ElementType.TYPE, ElementType.METHOD })
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Conditional(OnWebApplicationCondition.class)
  public @interface ConditionalOnNotWebApplication {
      // 没有属性方法，只要标识该注解就排除 Web 类型（SERVLET、REACTIVE）外，全部符合
  }
  ```

  

  &emsp;&emsp;下面进一步重构 `formatter-spring-boot-starter`, 将制动装配类设置为只在非 Web 应用下进行装载：

  ```java
  @Configuration
  @ConditionalOnProperty(prefix = "formatter", name = "enabled", havingValue = "true", matchIfMissing = true)
  @ConditionalOnResource(resources = "META-INF/spring.factories")
  // 设置为只在非 Web 应用下装载
  @ConditionalOnNotWebApplication
  public class FormatterAutoConfiguration {
  }
  ```

  

* Spring 表达式条件注解

  &emsp;&emsp;上面讨论的条件注解语义比较单一，而 Spring 表达式条件注解能够支持 *SpringEL（Spring Expression Language）* 表达式来实现复杂语义的条件判断。当然，使用自定义 *@Conditional* 也是一种可选方案，但本节只关注 Spring 表达式条件注解 *@ConditionalOnExpression*：

  ```java
  @Retention(RetentionPolicy.RUNTIME)
  @Target({ ElementType.TYPE, ElementType.METHOD })
  @Documented
  @Conditional(OnExpressionCondition.class)
  public @interface ConditionalOnExpression {
  	// SpringEL 表达式，支持占位符 表达式的结果最终返回 true 或者 false
  	String value() default "true";
  }
  ```

  

  &emsp;&emsp;&emsp;下面进一步重构 `formatter-spring-boot-starter`, 使用 SpringEL 表达式读取属性 `formatter.enabled` 的值来实现是否自动装载该配置类：

  ```java
  @Configuration
  @ConditionalOnProperty(prefix = "formatter", name = "enabled", havingValue = "true", matchIfMissing = true)
  @ConditionalOnResource(resources = "META-INF/spring.factories")
  @ConditionalOnNotWebApplication
  // 使用 SpringEL 方式读取，其实与 @ConditionalOnProperty 功能相似，但与后者略有不同的是：如果没有配置此属性时，不会装载
  @ConditionalOnExpression("${formatter.enabled:true}")
  public class FormatterAutoConfiguration {
  }
  ```

  

* 指定自动装载配置类的顺序

  &emsp;&emsp;也许是篇幅太长，小马哥在讲完这些条件注解后漏掉了 [自定义 Spring Boot 自动装配](https://reionchan.github.io/2022/06/19/thinking-in-springboot-part-2/#%E8%87%AA%E5%AE%9A%E4%B9%89-spring-boot-%E8%87%AA%E5%8A%A8%E8%A3%85%E9%85%8D) 小节最后所提出的问题：

  > 如果加入的配置 Class 之间存在初始化前后依赖关系，那么又应该如何实现呢？
  >
  > （引至原书 P337）

  

  &emsp;&emsp;其实这个问题在 [Spring Boot 自动装配原理](https://reionchan.github.io/2022/06/19/thinking-in-springboot-part-2/#spring-boot-%E8%87%AA%E5%8A%A8%E8%A3%85%E9%85%8D%E5%8E%9F%E7%90%86) 小节有关 “*AutoConfigurationGroup* 排序及装载候选组件” 原理讲解中已有答案。自动装配类的排序支持 **绝对排序 （@AutoConfigureOrder）** 和 **相对排序 （@AutoConfigureBefore @AutoConfigureAfter）** 两种注解方式。

  

  &emsp;&emsp;排序注解的标定方式有两种，一种将注解之间标注到自动装配类上，还有一种是将其设置在 **`spring-autoconfigure-metadata.properties`** 资源文件中。如果配置类不多时，单纯只将注解标注到自动装配类上即可。如果装配类较多时，出于性能考虑把标注在类上的排序元信息提取出来放入资源文件中能够提高一些加载性能。值得注意，**Spring Boot 优先读取资源文件的配置，没找到时再转而读取配置类上的注解**。一旦引入了资源文件，类上的排序注解其实可以不用，但是为了统一资源文件及类的排序语义，最好把两者信息保持同步一致。

  

  &emsp;&emsp;在自动装配类标注排序注解：
  
  ```java
  @Configuration
  @ConditionalOnProperty(prefix = "formatter", name = "enabled", havingValue = "true", matchIfMissing = true)
  @ConditionalOnResource(resources = "META-INF/spring.factories")
  @ConditionalOnNotWebApplication
  @ConditionalOnExpression("${formatter.enabled:true}")
  // 绝对位置排序注解
  @AutoConfigureOrder(-100)
  // 相对位置排序注解，此例设置为本配置类在 MessageSourceAutoConfiguration 之前装配
  @AutoConfigureBefore(name = "org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration")
  public class FormatterAutoConfiguration {
  }
  ```

  &emsp;&emsp;在资源文件 `spring-autoconfigure-metadata.properties` 配置排序注解：
  
  ```properties
  # 此行空值行需要配置，否则 AutoConfigurationMetadata#wasProcessed 方法判断为 false 就不会读取该资源文件的配置，还是读取类上的注解了
  thinking.in.spring.boot.samples.autoconfigure.formatter.FormatterAutoConfiguration=
  # 设置绝对排序值，数值越小优先级越高
  thinking.in.spring.boot.samples.autoconfigure.formatter.FormatterAutoConfiguration.AutoConfigureOrder=-100
  # 设置相对位置排序，值为另一个相对位置的自动装配类全限定名，当然还可指定 AutoConfigureAfter 属性，本例只演示 AutoConfigureBefore
  # 支持配置多个，以英文逗号 , 隔开
  thinking.in.spring.boot.samples.autoconfigure.formatter.FormatterAutoConfiguration.AutoConfigureBefore=org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration
  ```
  
  &emsp;&emsp;经过实测，按上面设置时 *FormatterAutoConfiguration* 将被放在首位装载。


> [formatter-spring-boot-starter 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/tree/master/spring-boot-2.0-samples/formatter-spring-boot-starter)
> [FormatterBootstrap 源码](https://github.com/ReionChan/thinking-in-spring-boot-samples/blob/master/spring-boot-2.0-samples/auto-configuration-sample/src/main/java/thinking/in/spring/boot/samples/auto/configuration/bootstrap/FormatterBootstrap.java)



## 推荐阅读

[《Spring Boot 编程思想（核心篇）》上篇 —— 总览 Spring Boot](https://reionchan.github.io/2022/06/18/thinking-in-springboot-part-1)

[《Spring Boot 编程思想（核心篇）》下篇 —— 理解 SpringApplication](https://reionchan.github.io/2022/06/20/thinking-in-springboot-part-3)



## 脚注

[^1]: 原文描述 “属性方法声明” 存在歧义，详细解释参考下篇勘误中有关 <a href='https://reionchan.github.io/2022/06/20/thinking-in-springboot-part-3/#%E5%8B%98%E8%AF%AF'>Page 209 描述歧义</a> 部分。