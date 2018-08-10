---
layout: wiki
title: IntelliJ IDEA
categories: IDE
description: IntelliJ IDEA for Mac 配置及快捷键
keywords: IntelliJ IDEA
---

## 配置
### 类、接口、枚举模板
* 打开 **Preferences** `⌘,` 后，展开 **Editor**
* 点选 **File and Code Templates**
* 右侧 **Files** 选项卡选择 **Class**
* 模板代码中，类声明行前添加下面注释 [`Code_01`](#code_01)
*  **Interface**、**Enum** 依次同样设置
* 点击 **Apply** 保存，**OK** 退出
  
<div id="code_01" style="font-size: 0.8em;color: blue;text-align: right;background-color: rgb(220,220,220);">Code_01 类、接口、枚举模板&emsp;&emsp;</div>
```java
/**
 * ${description}
 *
 * @author ${USER}
 * @date ${YEAR}-${MONTH}-${DAY} ${HOUR}:${MINUTE}
 * @version 1.0
 **/
```
  

###  类、方法文档注释

* 创建自定义模板组
	* 打开 **Preferences** `⌘,` 后，展开 **Editor**
	* 点选 **Live Templates**
	* 点击右侧 **+** 按钮，选择 **Template Group**
	* 输入自定义模板组组名，例如：**`Customize`**
	* 点击 **OK** 后，选中该组

* 自定义组添加类文档注释模板
	* 点击右侧 **+** 按钮，选择 **Live Template**
	* **Abbreviation** 项输入 `**`
	* **Description** 项输入 `Comment for Class, Interface, Enum`
	* **Template text** 项输入下面注释 [`Code_02`](#code_02)
	* 点击 “*⚠️ No applicable contexts yet.*” 后的 **Define**
	* 勾选 **Java** 复选框
	* 右下方点击按钮 **Edit variables** 弹出表格
	* **author**、**date** 变量的 **Expression** 栏分别填入 `user()`、`date("yyyy-MM-dd")`
	* **OK** 退出
* 自定义组添加方法文档注释模板
	* 点击右侧 **+** 按钮，选择 **Live Template**
	* **Abbreviation** 项输入 `*`
	* **Description** 项输入 `Comment for method`
	* **Template text** 项输入下面注释 [`Code_03`](#code_03)
	* 点击 “*⚠️ No applicable contexts yet.*” 后的 **Define**
	* 勾选 **Java** 复选框
	* 右下方点击按钮 **Edit variables** 弹出表格
	* **author**、**params**、**return** 变量的 **Expression** 栏分别填入 `user()`、`methodParameters()` 及 `methodReturnType()`
	* **OK** 退出到上层后，继续 **OK** 保存并退出
 
<div id="code_02" style="font-size: 0.8em;color: blue;text-align: right;background-color: rgb(220,220,220);">Code_02 类、接口、枚举注释模板&emsp;&emsp;</div>
```java
**
 * $description$
 *
 * @author $author$
 * @date $date$
 * @version 1.0
 **/
```

Code_03 方法注释模板 
<div id="code_03" style="font-size: 0.8em;color: blue;text-align: right;background-color: rgb(220,220,220);">Code_03 方法注释模板 &emsp;&emsp;</div>
```java
** 
 * $description$
 * 
 * @author $author$       
 * @param $params$ 
 * @return $return$
 */
 ```
 
###  配置文件
&emsp;&emsp;上述配置完成后，IDEA 会将修改信息保存到配置文件当中，具体位置如下：  
  
* 配置文件主目录（下文简写为 **CONF_HOME**）  
	
	```
/Users/$USER/Library/Preferences/IntelliJIdeaXXXX  
	```

* 类、接口、枚举模板配置文件  
	
	```
$CONF_HOME/fileTemplates/internal
——————————————————
internal 中包含上面修改的几个模板：
└── internal
      └── Class.java
      └── Enum.java
      └── Interface.java  
	```
	
* 类、方法等自定义模板配置文件  
	
	```
$CONF_HOME/templates
——————————————————
templates 中以模板组为单位的 XML 文件组成，
例如：上节创建的模板组 `Customize`
└── templates
      └── Customize.xml  
	```
	

* 类、接口、枚举模板配置文件  
	
	```
$CONF_HOME/fileTemplates/internal
——————————————————
internal 中包含上面修改的几个模板：
└── internal
      └── Class.java
      └── Enum.java
      └── Interface.java  
	```
	
* IntelliJ IDEA 启动参数配置文件，可配置内存、字符编码等  
	
	```
$CONF_HOME/idea.vmoptions 
	```
