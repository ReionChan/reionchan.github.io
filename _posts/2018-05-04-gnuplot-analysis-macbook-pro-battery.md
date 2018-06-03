---
layout: post
title: Gnuplot 案例入门
categories: Tools
tags: Gnuplot
excerpt: Gnuplot 绘图分析 MacBook Pro 电池耗损
image: http://www.gnuplot.info/figs/gaussians.png
description: Gnuplot 绘图分析 MacBook Pro 电池耗损
keywords: Gnuplot, plot, MacBook, 入门, 案例
licences: cc
---

## 简介  
> &emsp;&emsp;Gnuplot 是一款适用于 Linux，OS/2，MS Windows，OSX，VMS 和许多其他平台的便携式命令行驱动图形工具。源代码受版权保护，但免费分发（即，您不必为此付费）。它最初创建的目的是让科学家和学生能够交互式地将数学函数和数据可视化，但已经发展到支持许多非交互式用途，例如Web脚本。它也被Octave等第三方应用程序用作绘图引擎。自1986年以来，Gnuplot一直得到支持和积极发展。
<div style="text-align: right;"> —— 摘自 <a href="http://www.gnuplot.info/">gnuplot 官网</a> &emsp;&emsp;</div>

* 支持许多不同类型的 2D、3D 图

* 支持许多不同类型的格式输出  
	- 交互式屏幕显示：跨平台（Qt、wxWidgets、X11）、特定平台（Windows、OS/2）
	- 丰富的输出格式：postscript（含 eps）、PDF、PNG、LaTex ……
	- 可鼠标操作的 web 显示格式：HTML5、SVG

## 资源
* [软件下载](https://sourceforge.net/projects/gnuplot/files/gnuplot/5.2.3/)
	- 源码：[gnuplot-5.2.3.tar.gz](https://jaist.dl.sourceforge.net/project/gnuplot/gnuplot/5.2.3/gnuplot-5.2.3.tar.gz)  
	- MacOS：[gnuplot-5.2.3-quartz.pkg](http://ricardo.ecn.wfu.edu/pub/gnuplot/gnuplot-5.2.3-quartz.pkg) 或 [Plot2 on AppStore](https://itunes.apple.com/cn/app/plot2/id846509360?mt=12)
	- Windows：[gp523-win64-mingw.7z](https://jaist.dl.sourceforge.net/project/gnuplot/gnuplot/5.2.3/gp523-win64-mingw.7z)、[gp523-win32-mingw.7z](https://netix.dl.sourceforge.net/project/gnuplot/gnuplot/5.2.3/gp523-win32-mingw.7z)  

* [![pdf](http://www.nand2tetris.org/icons/acrobat.gif) 官方用户手册](http://www.gnuplot.info/docs_5.2/Gnuplot_5.2.pdf)

*  《使用 gnuplot 科学作图 — Gnuplot 中文教程》  
	- 作者：[马欢](mailto:yusufma77@yahoo.com) 
	- [![pdf](http://www.nand2tetris.org/icons/acrobat.gif) 下载](https://github.com/ReionChan/PhotoRepo/raw/master/gnuplot/gnuplot_tutorial.pdf)&emsp;[![CC BY-NC-SA 3.0](https://reionchan.github.io/images/posts/cc-by-nc-sa.png)](https://creativecommons.org/licenses/by-nc-sa/3.0/)  

<div style="text-align: right;">🌹由衷感谢资源作者！🌹&emsp;&emsp;</div>

## 安装

&emsp;&emsp;本文案例使用 MacOS 环境，故分别尝试了 [gnuplot-5.2.3-quartz.pkg](http://ricardo.ecn.wfu.edu/pub/gnuplot/gnuplot-5.2.3-quartz.pkg) 及 [Plot2 on AppStore](https://itunes.apple.com/cn/app/plot2/id846509360?mt=12) 。读者可以自行选择，只是前者跨平台交互终端是 wxt，后者是 qt，且默认是纯图形化的绘图界面。双击按提示进行安装即可。对比发现 Plot2 分辨率更高，故下文都用它来演示。
  
&emsp;&emsp;本文主要演示命令行方式，图形方式绘图在此就不做介绍，读者自行探索。安装完成后，打开系统终端，输入命令：  

```sh
Reion@MBPr ~$ gnuplot
```

&emsp;&emsp;显示版本、帮助等信息：  

```

	G N U P L O T
	Version 5.2 patchlevel 0    last modified 2017-09-01 

	Copyright (C) 1986-1993, 1998, 2004, 2007-2017
	Thomas Williams, Colin Kelley and many others

	gnuplot home:     http://www.gnuplot.info
	faq, bugs, etc:   type "help FAQ"
	immediate help:   type "help"  (plot window: hit 'h')

Terminal type is now 'qt'
gnuplot> 
```  

&emsp;&emsp;在出现的 `gnuplot>` 处输入 `test` 命令，弹出 qt 终端的风格代码测试图：

```sh
gnuplot> test
```

<center><img src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/gnuplot/test.png" width="600" /><br /> 图一 qt 终端的风格代码测试图</center>

## 案例
### 介绍
&emsp;&emsp;本案例将用 Gnuplot 来绘制笔者的 MBP 的电池容量耗损图。基础实验数据来源于 [**coconutBattery**](https://www.coconut-flavour.com/coconutbattery/)，它是一款能够记录 Mac、iOS设备的电池健康度的软件，对软件作者表示由衷感谢。用此软件采集了 40 个月中 MBP 的电池容量数据，将分别以时间、充电循环次数两个维度来绘制展现电池的容量曲线。

### 学习背景知识 (20 min)
&emsp;&emsp;获取上面资源里的《使用 gnuplot 科学作图 — Gnuplot 中文教程》，本案例涉及的必备知识点都囊括在下面，根据知识点序号定位查阅，作者写得很详细，在此就不做过多赘述：  

* 前 10 个知识点，涉及安装启动、`plot`、`set`、`replot`、点线风格等命令知识
* 第 17、18、35、43 知识点，涉及边框、坐标、填充及脚本导入执行命令

### 基本实验数据  

* 基于时间维度的电池容量数据&emsp;[点击下载](https://github.com/ReionChan/PhotoRepo/blob/master/gnuplot/capacity_base_on_age_of_battery.dat)

&emsp;&emsp;数据为四列，以 `tab` 符号分隔的文本格式：

```json
#循环次数  容量  月数       时间
102     6019     12     2014-12
111     5725     13 	2015-01
122     5920     14 	2015-02
134     5850     15 	2015-03
144     5997     16 	2015-04
145     6057     17 	2015-05
156     6036     18 	2015-06
166     6033     19 	2015-07	
```


* 基于充电循环次数的电池容量数据&emsp;[点击下载](https://github.com/ReionChan/PhotoRepo/blob/master/gnuplot/capacity_base_on_loadcycles.dat) 

&emsp;&emsp;数据为三列，以 `tab` 符号分隔的文本格式：

```json
#循环次数 容量        时间	
98      5899     2014-12-24
99      5947     2014-12-24
100     5896     2014-12-25
100     5901     2014-12-25
101     6017     2014-12-26
101     6007     2014-12-27
102     5830     2014-12-28	
```

### 脚本编写运行 (10 min)

* 电池容量随充电循环次数变化脚本
 
```sh
# 设置图表主标题
set title "MacBook Pro（Retina，13 英寸，2013 年末）电池容量充电循环耗损表"
# 显示图例说明
set key
# 显示背景格子
set grid
# 设置 X 轴标签
set xlabel "充电循环次数"
# 设置 X 轴数值范围
set xrange [90:310]
# 设置 X 轴数值刻度精度
set xtics 90,10,310
# 取消 X 轴刻度镜像显示
set xtics nomirror
# 设置 Y 轴标签
set ylabel "电池容量（mAh）"
# 设置 Y 轴数值范围
set yrange [4500:6500]
# 设置 Y 轴数值刻度精度
set ytics 4500,500,6500
# 取消 Y 轴刻度镜像显示
set ytics nomirror
# 绘制单条折线图
plot "capacity_base_on_loadcycles.dat" using 1:2 with lp pt -1 lw 3 lc 2 title "电池容量"
```

&emsp;&emsp;脚本存到数据文件同目录下以 **`.gnu`** 结尾的文件中，**在此目录打开终端**输入：  
 
 ```sh
 # 运行 gnuplot
 Reion@MBPr ~$ gnuplot
 # 进入 gnuplot 会话，继续输入 load 命令
 gnuplot> load "capacity_base_on_loadcycles.gnu"
 ``` 

&emsp;&emsp;绘制的结果如下图所示，可以选择导出多种格式，此例为 SVG 格式：  

<center><picture>
    <source type="image/svg+xml" srcset="https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/gnuplot/capacity_base_on_loadcycles.svg?sanitize=true">
    <img src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/gnuplot/capacity_base_on_loadcycles.png" alt="capacity_base_on_loadcycles"  width="800">
</picture> 图二 电池容量随充电循环次数变化</center>

* 电池容量随时间变化脚本

```sh
# 设置图表主标题
set title "MacBook Pro（Retina，13 英寸，2013 年末）电池容量时间耗损表"
# 显示图例说明
set key
# 显示背景格子
set grid
# 设置 X 轴标签
set xlabel "使用月数"
# 设置 X 轴数值范围
set xrange [12:52]
# 设置 X 轴数值刻度精度
set xtics 12,1,52
# 取消 X 轴刻度镜像显示
set xtics nomirror
# 设置 Y 轴标签
set ylabel "电池容量（mAh）"
# 设置 Y 轴数值范围
set yrange [4500:6500]
# 设置 Y 轴数值刻度精度
set ytics 4500,500,6500
# 取消 Y 轴刻度镜像显示
set ytics nomirror
# 设置填充透明度
set style fill solid 0.2
# 绘出电池容量填充图、损耗点折线图，复合成最终效果
plot "capacity_base_on_age_of_battery.dat" using 3:2 with filledcurves y1=0 lc 2 title "电池容量", "capacity_base_on_age_of_battery.dat" using 3:2 axis x1y1 with lp pt 7 ps 2 lw 1 lc 2 title "损耗折线"
```
&emsp;&emsp;脚本存到数据文件同目录下以 **`.gnu`** 结尾的文件中，**在此目录打开终端**输入：  
 
 ```sh
 # 运行 gnuplot
 Reion@MBPr ~$ gnuplot
 # 进入 gnuplot 会话，继续输入 load 命令
 gnuplot> load "capacity_base_on_age_of_battery.gnu"
 ``` 

&emsp;&emsp;绘制的结果如下图所示，可以选择导出多种格式，此例为 SVG 格式：  

<center><picture>
    <source type="image/svg+xml" srcset="https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/gnuplot/capacity_base_on_age_of_battery.svg?sanitize=true">
    <img src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/gnuplot/capacity_base_on_age_of_battery.png" alt="capacity_base_on_age_of_battery"  width="800">
</picture> 图三 电池容量随时间变化</center>

## 总结

&emsp;&emsp;案例使用 gnuplot 绘制简单的 2D 图表，更多复杂有意思的图表可以点击 [Demo](http://gnuplot.sourceforge.net/demo_5.2/) 进行学习。

* 参考资料：
	- 官网 <http://www.gnuplot.info/>
	- gnuplot manpage by `man gnuplot`
	- 书籍《使用 gnuplot 科学作图 — Gnuplot 中文教程》  
