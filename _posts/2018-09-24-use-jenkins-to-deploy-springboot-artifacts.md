---
layout: post
title: Jenkins 持续集成忠实管家
categories: Tools Spring
excerpt: 使用 Jenkins 自动化编译部署 Springboot 项目
image: https://camo.githubusercontent.com/fe0c9ecc354db49a376fa2a68f26bebdc75df14d/68747470733a2f2f6a656e6b696e732e696f2f73697465732f64656661756c742f66696c65732f6a656e6b696e735f6c6f676f2e706e67
description: 使用 Jenkins 自动化编译部署 Springboot 项目
keywords: Jenkins, Springboot, Git，CI, CD, Deploy, artifacts, Java, jar
licences: cc
---
<br/>
![Jenkins](https://camo.githubusercontent.com/fe0c9ecc354db49a376fa2a68f26bebdc75df14d/68747470733a2f2f6a656e6b696e732e696f2f73697465732f64656661756c742f66696c65732f6a656e6b696e735f6c6f676f2e706e67)

---
> &emsp;&emsp;Jenkins 是一款开源自动化服务器，通过提供众多插件来构建、部署及自动化任何项目。
<div style="text-align: right;"> —— 摘自 <a href="https://jenkins.io/">Jenkins 官网</a> &emsp;&emsp;</div>

## 安装及运行

### 所需环境
* 硬件
	- 推荐 512 MB 以上内存
	- 10 GB 的硬盘空间
	
* 软件
	- Java 8

### 下载运行
* 下载 [Jenkins](http://mirrors.jenkins.io/war-stable/latest/jenkins.war)
* 打开命令行终端，到下载目录
* 运行 `java -jar jenkins.war --httpPort=8080`
* 在浏览器输入 `http://localhost:8080`
* 根据首页提示，找到初始密码，解锁及下载常用插件

<div style="font-size: 0.8em;color: red;text-align: right;"> 注意：插件安装提示离线，请到 【插件管理】 -> 【高级】 修改升级站点为<br> http://updates.jenkins-ci.org/update-center.json&emsp;&emsp;</div>

## 插件安装及配置

### 全局工具配置
&emsp;&emsp; 在初始化页面安装完官方推荐的插件后，导航【系统管理】->【全局工具配置】下，进行 JDK、Git、Maven 的路径配置。

* JDK
	- 别名：`JDK_8`
	- 路径：`/usr/java/jdk1.8.0_181` （根据实际路径填）
	- 取消自动安装复选框
	
* Git
	- Name：`Git`
	- Path to Git executable：`/usr/bin/git` （根据实际路径填）
	- 取消自动安装复选框
	
* Maven
	- Name：`Maven_3.5.4`
	- Maven_HOME：`/usr/local/maven3/`（根据实际路径填）
	- 取消自动安装复选框

&emsp;&emsp; 点击保存即可完成全局工具配置。
 
### 插件安装及配置

&emsp;&emsp; 在初始化页面安装完官方推荐的插件后，需要额外安装如下几个插件：

* Git 插件
* Maven Integration 插件
* SSH 插件

&emsp;&emsp; 导航到【系统管理】->【插件管理】->【可选插件】,搜索并选择相应的插件，然后点击直接安装并重启即可。

### SSH 配置

&emsp;&emsp; 假设目前有 Git 源代码版本控制服务器 A ，以及现在运行 Jenkins 的服务器 B ，并同时将 B 服务器当做 Spring Boot 应用服务器。那么我们需要把 B 服务器的公钥复制到 A 服务器 **<span style="color:red">Git 账户</span>** 路径下的文件 `~/.ssh/authorized_keys` 中，从而使 B 服务器免密从源代码服务器 A 下载源代码。  
  
* B 服务器生成密钥对
	
	```sh
	$ ssh-keygen -t rsa
	$ cat .ssh/id_rsa.pub
	```
	将显示的内容复制保存到 A 服务器的相应文件中并重启 A 服务器的 SSH 服务。

* A 服务器重启 SSH 服务

	```sh
	$ service sshd restart
	```
	
* 添加全局凭证
	- 导航【凭据】->【系统】->【全局凭据】->【添加凭据】
	- 类型：`SSH Username with private key`
	- 范围：`全局（Jenkins，nodes，items，all child items，etc）`
	- Username：`gitUser` （根据实际情况填）
	- Private Key：选中单选框 `Enter directly`
	- Key：粘贴 Jenkins 服务器 B 的密钥，路径 `~/.ssh/id_rsa`  
		
		```sh
		# 密钥格式如：
		-----BEGIN RSA PRIVATE KEY-----
		SLDKJFLLKJSkjsldkjflkjLSJDFKSD
		JFLLKJSkjsldkjflkjLSJflkjLLKflkjLL
		kjsldkjflkjLLKJSkjslSJDFKSD==
		-----END RSA PRIVATE KEY-----
		```  

	- Passphrase：`gitPassword` （根据实际情况填）
	- ID：`Git ID` （凭据标识ID）
	- 描述：`Use for git` （简短描述凭据用途）
	- 点击确定即可	


## 部署项目
&emsp;&emsp;假设源代码服务器中包含一个 Spring Boot 项目，例如：demo ，且该项目下含有多个子模块，demo-a、demo-b 。

* 在项目各模块的 `pom.xml` 配置 `maven-compiler-plugin`、`spring-boot-maven-plugin` 插件
* Jenkins 点击【新建任务】输入任务名称：`demo`
* 选择 `构建一个 maven 项目`
* 跳转的界面中，依次填入：
	- 描述：（项目的描述）
	- JDK：下拉选择之前配置的 `JDK_8`
	- 源代码管理：选择 Git
		* Repository URL：`gitUser@host:/path/demo.git`
		* Credentials：下拉选择创建的凭据 `Git ID`
		* Branches to build：`*/master` （分支名根据实际填）
	- 构建触发器
		* 选中 `Build whenever a SNAPSHOT dependency is built`
		* 选中 `轮询 SCM`，编辑日程表，例如
		
			```
			TZ=Asia/Shanghai
			# 每隔5分钟查询是否有新代码提交
			H/5 * * * *
			```
	- Build
		* Root POM：`pom.xml`
		* Goals and options：`-B -DskipTests clean package`  
		（根据实际情况，目前是清理及打包）
	- Post Steps
		* 选中单选框 `Run only if build succeeds` 
		* 执行 shell，在命令文本框填写部署脚本  
		
			```sh
			/path/deploy.sh
			```  
		（根据实际情况填写，本例采用让其执行外部 shell 部署脚本，脚本详情参看 [**附录Ⅰ**](#%E9%99%84%E5%BD%95%E2%85%B0)）
		* 点击保存
		
<div style="font-size:1.5em;color: red;margin:0.5em;">&nbsp;注意：</div>  
**&emsp;&emsp;即使在脚本中使用 `nohup` 命令启动进程，也会被 Jenkins 的 `ProcessTreeKiller` 杀掉，以如下参数重新启动 Jenkins 可避免该情况发生：**   

```sh
java -Dhudson.util.ProcessTree.disable=true -jar jenkins.war
```	

* 详情参考：[ProcessTreeKiller](https://wiki.jenkins.io/display/JENKINS/ProcessTreeKiller)		

## 附录Ⅰ
### 部署脚本

```sh
echo ===== 执行第 $BUILD_NUMBER 次构建 =======
echo '[仓库] '$GIT_URL
echo '[分支] '$GIT_BRANCH
echo '[提交名称] '$GIT_COMMITTER_NAME
echo '[提交用户] '$GIT_AUTHOR_NAME 
echo -----------------------------------------
DEPLOY_DIR=/path/deployDir
export DEPLOY_DIR
PID_a=$(ps -ef | grep demo-a-1.0-SNAPSHOT.jar | grep -v grep | awk '{ print $2 }')
PID_b=$(ps -ef | grep demo-b-1.0-SNAPSHOT-exec.jar | grep -v grep | awk '{ print $2 }')
echo ---------- 停止当前运行的服务 --------------
if [ -z "$PID_a" ]
then
	echo '[demo-a] 服务已经停止，无需操作。'
else
	echo kill $PID_a
    kill $PID_a
fi
if [ -z "$PID_b" ]
then
	echo '[demo-b] 服务已经停止，无需操作。'
else
	echo kill $PID_b
    kill $PID_b
fi
echo ---------- 停止当前运行的服务完毕 --------------
echo ---------- 清理服务部署目录 --------------
cd $DEPLOY_DIR
rm -rf *.jar
rm -rf *.log
echo ---------- 清理服务部署完毕 --------------
echo ---------- 拷贝JAR到部署目录 --------------
WORKSPACE=/path/.jenkins/workspace/demo
cd $WORKSPACE
cp demo-a/target/demo-a-1.0-SNAPSHOT.jar $DEPLOY_DIR
cp demo-b/target/demo-b-1.0-SNAPSHOT.jar $DEPLOY_DIR
echo ---------- 拷贝JAR到部署目录完毕 --------------
echo ---------- 启动服务 --------------
cd $DEPLOY_DIR
echo 启动 demo-a-1.0-SNAPSHOT.jar
nohup java -jar $DEPLOY_DIR/demo-a-1.0-SNAPSHOT.jar >/dev/null 2>&1 &
echo 启动 demo-b-1.0-SNAPSHOT.jar
nohup java -jar $DEPLOY_DIR/demo-b-1.0-SNAPSHOT.jar >/dev/null 2>&1 &
echo ---------- 启动服务完毕 --------------
```
			


