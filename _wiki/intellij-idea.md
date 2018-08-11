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
 
以后只需在类、方法声明的**行前**输入 “`快捷键`+ `Tab`”组合键即可插入所设置的模板注释：  

* 方法声明行前键入 “`*`+ `Tab`”快速插入方法注释
* 类、接口、枚举声明行前键入 “`**`+ `Tab`”快速插入类注释
 
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
	
* IntelliJ IDEA 启动参数配置文件，可配置内存、字符编码等  
	
	```
$CONF_HOME/idea.vmoptions 
	```

## 快捷键  
  
&emsp;&emsp; 本文涉及的快捷键都是基于 Mac 平台，Linux、Windows 平台的快捷键请参考下方给出的官方文档。此外本文对快捷键的分组铺排是基于官方学习辅助插件，按下方给出的链接安装插件并依照它的提示完成所有步骤，即可熟悉默认的大多数快捷键操作。 

* [IntelliJ IDEA Keymap <img src="https://raw.githubusercontent.com/ReionChan/nand2tetris/master/res/pdf.png" alt="pdf"  width="16" />](https://resources.jetbrains.com/storage/products/intellij-idea/docs/IntelliJIDEA_ReferenceCard.pdf)
* [IDE Features Trainer](https://plugins.jetbrains.com/plugin/8554?pr=idea)
* [插件安装教程](http://wiki.jikexueyuan.com/project/intellij-idea-tutorial/plugins-settings.html)

### 标记说明

<style type="text/css">
	.red {background-color: rgba(255, 150, 150, .8)}
	.purple {background-color: rgba(150, 150, 255, .8)}
	.green {background-color: rgba(150, 255, 150, .8)}
</style>

颜色标记	| 含义
:-----------:	| ------------ 
<span class='red'> &emsp;&emsp; </span>| 牢记
<span class='purple'> &emsp;&emsp; </span>| 熟练
<span class='green'> &emsp;&emsp; </span>| 会用


### 基础编辑

* 选择  

	图标		|	组合键				| 说明
	----------	| -------------------------------	| -----------	------------------ 
	⌥⇧→	| Option-Shift-Right Arrow	| 从光标处向后选择文本
	⌥⇧← 	| Option-Shift-Left Arrow	| 从光标处向前选择文本
	⌥⇧↑	| Option-Shift-Up Arrow	| 光标所在行或选中行上移
	⌥⇧↓ 	| Option-Shift-Down Arrow	| 光标所在行或选中行下移
	<span class='red'>⌥↑</span>	 	| Option-Up Arrow			| 扩大光标选中行范围
	<span class='red'>⌥↓</span>	 	| Option-Down Arrow		| 缩小光标选中行范围
	⇧↑ 		| Shift-Up Arrow			| 向上选择多行
	⇧↓ 		| Shift-Down Arrow		| 向下选择多行
	⌘A		| Command-A				| 全选当前文件所有文本

* 注释行

	图标		|	组合键				| 说明
	----------	| -------------------------------	| -----------	------------------ 
	⌘/		| Command-Slash			| 注释光标所在行（*单行注释*）
	
	选中多行后，敲此快捷键将以*单行注释* 的方式注释所选中的每一行。  
	**单行注释：**Java 语言中行前加入双反斜杠 `\\` 即被称作单行注释。

* 删除行

	图标		|	组合键				| 说明
	----------	| -------------------------------	| -----------	------------------ 
	⌘⌫ 	| Command-Delete			| 删除光标所在行

* 复制

	图标		|	组合键				| 说明
	----------	| -------------------------------	| -----------	------------------ 
	⌘D		| Command-D				| 将光标所在行粘贴到下方
	
	选中多行后，敲此快捷键将粘贴所选的多行到多行的下方。

* 移动

	图标		|	组合键					| 说明
	----------	| -------------------------------		| -----------	------------------------------------ 
	⌥⇧↑	| Option-Shift-Up Arrow		| 光标所在行或选中行上移
	⌥⇧↓ 	| Option-Shift-Down Arrow		| 光标所在行或选中行下移
	⌘⇧↑	| Command-Shift-Up Arrow		| 光标所在行或选中行上移（*考虑语法*）
	⌘⇧↓ 	| Command-Shift-Down Arrow	| 光标所在行或选中行下移（*考虑语法*）
	
	**考虑语法：**选中行能否上、下移动如果会导致语法错误就不允许移动。

* 展开折叠

	图标		|	组合键				| 说明
	----------	| -------------------------------	| -----------	------------------------------------ 
	⌘- 		| Command-Minus sign		| 折叠光标所处代码块
	⌘+ 		| Command-Plus sign		| 展开光标所处代码块（⌘= 效果相同）
	⌘⇧- 	| Command-Shift-Minus sign| 折叠当前文件所有代码块
	⌘⇧+ 	| Command-Shift-Plus sign	| 展开当前文件所有代码块（⌘⇧= 效果相同）

* 多项选择
	
	图标		|	组合键			| 说明
	----------	| ------------------------------	| -----------	------------------------------------------------ 
	⌃G 		| Control-G			| 首次选中光标所在符号，之后追加下个相同符号
	⌃⇧G 	| Control-Shift-G		| 取消最后选中的相同符号
	⌃⌘G 	| Control-Command-G	| 选中文件中所有相同符号

### 代码完成

* 基础完成

	图标		|	组合键			| 说明
	----------	| ------------------------------	| -----------	------------------------------------------------
	⏎ 		| Return				| 敲字母后首个提示满足要求直接回车确认完成
	⌃Space 	| Control-Space		| 弹出基础完成提示框，连续两次访问更深层次建议

* 智能类型完成

	图标			|	组合键			| 说明
	----------		| ------------------------------	| -----------	------------------------------------------------
	<span class='red'>⌃⇧Space</span> 	| Control-Shift-Space	| 根据当前上下文类型智能给出类型建议，也适用return 语句

* 语句完成

	图标		|	组合键				| 说明
	----------	| ------------------------------		| -----------	-
	⌘⇧⏎ 	| Command-Shift-Return	| *语句完成*
	
	**语句完成：**能自动辅助完成 if、for 等语句结构与书写定位，还包括语句末尾添加分号等操作。

* Tab 完成

	图标	|	组合键	| 说明
	------	| ---------------	| --------------------------
	⇥  	| Tab		| *覆盖光标后的内容*
	
	**所谓覆盖光标后内容，即：**当敲下 `⌃Space` 弹出提示选项时，敲 `⇥ ` 键会将提示内容覆盖光标后的内容而非简单插入。  

### 代码重构

* 重命名

	图标		|	组合键	| 说明
	------		| ---------------	| ---------------------------------
	<span class='red'>⇧F6</span>  	| Shift-F6		| 选中并编辑光标所在的变量
	
	以此方式重命名的变量，将同步修改其 Getter、Setter 方法，以及所有引用了该变量的地方。

* 提取变量、字段

	图标		|	组合键			| 说明
	------		| ---------------------------	| ------------------------------------------
	⌘⌥V 	| Command-Option-V	| 将光标所在变量抽取为变量、字段

* 提取方法

	图标		|	组合键			| 说明
	------		| ---------------------------	| ----------------------------
	⌘⌥M 	| Command-Option-M	| 将所选语句抽取为方法

* 提取常量

	图标		|	组合键			| 说明
	------		| ---------------------------	| -------------------------------
	⌘⌥C 	| Command-Option-C	| 将所选表达式抽取为常量
	
* 提取参数

	图标		|	组合键			| 说明
	------		| ---------------------------	| ------------------------------------
	⌘⌥P 	| Command-Option-P	| 将所选表达式抽取为方法参数


### 代码辅助

* 格式化代码

	图标		|	组合键			| 说明
	------		| ---------------------------	| ------------------------------------
	⌘⌥L 	| Command-Option-L	| 格式化代码

* 查看方法参数

	图标	|	组合键		| 说明
	------	| --------------------	| --------------------------------------------
	<span class='red'>⌘P</span> 	| Command-P		| 光标在方法括号内敲击显示参数列表
	
* 查看类、方法被使用情况

	图标		|	组合键			| 说明
	------		| ---------------------------	| ------------------------------------
	⌘⌥F7 	| Command-Option-F7	| 显示使用情况
	⌥F7 	| Option-F7		| 在项目、库中查找使用情况
	⌘F7 	| Command-F7	| 在当前文件中查找使用情况
	⌘⇧F7 	| Command-Shift-F7	| 在当前文件中高亮使用情况

* 快速访问文档注释

	图标		|	组合键		| 说明
	------		| -----------------		| --------------------------------------------
	F1 		| F1				| 弹窗显示光标所在符号的文档注释
	⎋ 		| Esc			| 关闭文档注释窗口
	⌥Space	| Option-Space	| 弹窗显示光标所在符号的*定义信息*
	
	**定义信息：** 变量、方法的声明、实现代码等信息。

* 编辑器编程辅助

	图标		|	组合键			| 说明
	------		| -----------------			| --------------------------------------------
	F2 		| F2					| 定位到当前文件下一个错误位置
	⌘F1		| Command-F1		| 查看错误描述信息
	<span class='red'>⌥⏎</span> 		| Option-Enter		| 显示意向操作及快速修复
	⌘⏎		| Command-Enter		| 智能拆分行
	⌃⇧J		| Control-Shift-J		| 智能拼接行
	<span class='red'>⌘N</span>		| Command-N		| 生成构造、Getter/Setter/覆盖 等方法（⌃⏎ 效果等同）
	⌘⌥T	| Command-Option-T	| 将选择的代码块用语句包围
	⌘⇧F7	| Command-Option-F7	| 高亮当前文件中所有光标所在位置的符号
	
* 动态代码模板

	图标		|	组合键		| 说明
	------		| -----------------		| ----------------------------------
	⌘J 		| Command-J		| 插入预定义的动态代码模板
	⌘⌥J 	| Command-Option-J	| 插入预定义的可包围所选代码行的动态代码模板

* 版本控制、本地历史

	图标		|	组合键		| 说明
	------		| -----------------		| ----------------------------------
	⌘K 		| Command-K		| 提交本项目到版本控制系统
	⌘T	 	| Command-T		| 从版本控制系统更新本项目
	⌘⇧K 	| Command-Shift-K	| 推送提交请求
	⌃V	 	| Control-V	| 弹出版本控制操作窗口	

### 代码导航

* 查看源代码

	图标		|	组合键				| 说明
	------		| -----------------				| ----------------------------------
	⌘↓ 		| Command-Down Arrow	| 查看光标所在类型的源代码
	<span class='red'>⌘E</span> 		| Command-E				| 弹出最近使用的文件窗口
	⌘L 		| Command-L				| 跳转到输入的代码行
	⌃↑		| Control-Up Arrow		| 跳转到上一个方法
	⌃↓  	| Control-Down Arrow		| 跳转到下一个方法
	⌘⌥[	| Command-Option-Left Bracket		| 跳到代码块开始处
	⌘⌥]	| Command-Option-Right Bracket	| 跳到代码块结束处
	⌘[		| Command-Left Bracket	| 跳到上一个光标所在位置处
	⌘]		| Command-Right Bracket	| 跳到下一个光标所在位置处

* 声明与实现

	图标		|	组合键			| 说明
	------		| -----------------			| ----------------------------------
	⌘B 		| Command-B			| 查看光标所在类型的声明定义处（⌘Click 相同效果）
	⌘⌥B  	| Command-Option-B	| 查看光标所在类型的实现
	⌘U 		| Command-U			| 跳转到超类、超类方法
	⌃⇧B	| Control-Shift-B		| 跳转到

* 文件结构

	图标		|	组合键			| 说明
	------		| -----------------			| ----------------------------------
	⌘F12 	| Command-F12		| 查看当前类文件的*代码结构*
	
	**代码结构：** 当前类的成员变量、方法的声明信息。
	

* 查找、替换

	图标		|	组合键			| 说明
	------		| -----------------			| ----------------------------------
	<span class='red'>Double⇧</span>	| Shift-Shift			| *任何位置*查找关键字
	⌘R 		| Command-R			| 替换当前选择的变量
	⌘⇧R 	| Command-Shift-R	| 替换所选路径下所有当前选择的变量
	⌘F 		| Command-F			| 查找当前选择的变量
	⌘⇧F 	| Command-Shift-F	| 查找所选路径下所有当前选择的变量
	⌘G		| Command-G			| 定位到下一个发现点 （⏎ 相同效果）
	⌘⇧G 	| Command-Shift-G	| 定位到上一个发现点
	⎋ 		| Esc				| 关闭查找面板（关闭后查找快捷键仍可用）
	
	**任何位置：** 匹配类、文件、变量、方法等等。
	
### 代码调试

图标		| 组合键	| 说明
------		| -----------------	| ----------------------------------
F7		| F7			| 单步执行，遇到函数进入 
F8		| F8			| 单步执行，遇到函数跨过，视子函数为一步
⇧F7		| Shift-F7		| 智能单步执行，断点所在行有多个方法调用，会弹出进入这些方法
⇧F8		| Shift-F8		| 跳出
⌥F9		| Option-F9	| 运行到光标所在行
⌥F8		| Option-F8	| 表达式求值
⌘⌥R 	| Command-Option-R	| 恢复程序
⌘F8 	| Command-F8	| 断点开关
⌘⇧F8 	| Command-Shift-F8	| 查看所有断点

### 编译运行

图标		|	组合键		| 说明
------		| -----------------		| ----------------------------------
⌘F9		| Command-F9	| 编译项目 
⌘⇧F9	| Command-Shift-F9	| 编译所选的文件、包或模块
⌃R		| Control-R	| 运行
⌃D		| Control-D	| 调试
⌃⌥R	| Control-Option-R	| 弹出 Run 的可选配置菜单运行
⌃⌥D	| Control-Option-D	| 弹出 Debug 的可选配置菜单调试
⌃⇧R	| Control-Shift-R	| 从编辑器上下文配置来运行
⌃⇧D	| Control-Shift-D	| 从编辑器上下文配置来调试

































