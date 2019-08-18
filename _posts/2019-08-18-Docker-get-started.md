---
layout: post
title: Docker 快速入门
categories: Tools
excerpt: Docker 快速入门指南
image: https://www.docker.com/sites/default/files/social/docker_facebook_share.png
description: Docker 快速入门指南
keywords: Docker, docker, Image, image, Container, container, Build, build, Deploy, deploy
licences: cc
---
<br/>

![Docker Logo](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/docker/docker_facebook_share.png)

---
> &emsp;&emsp;调试你的应用，而不是你的环境！无论何地都能安全的构建、分享及运行任何的应用。
<div style="text-align: right;"> —— 摘自 <a href="https://www.docker.com">Docker 官网</a> &emsp;&emsp;</div>

## Docker 概念[^1]

&emsp;&emsp;Docker 为开发人员及系统管理员从事开发、部署及运行基于容器的应用提供一个通用平台。使用基于 Linux 容器来部署应用的操作被称为 *容器化*。容器本身虽不是新技术理念，但是**通过容器化来简化部署应用的这种理念还是蛮新奇的理念**。

&emsp;&emsp;容器化之所以越来越流行，是因为容器具备如下特点：

* **高灵活性**：即便是很复杂的应用也能被容器化
* **轻量级**：容器之间能有效共享及利用宿主机内核
* **高可替换性**：你可以在运行时进行更新、升级
* **高可移植性**：你可以本地构建、云端部署、到处运行
* **高可扩展性**：你可以增加并自动分发容器的副本
* **高可堆叠性**：你可以在运行时垂直堆叠更多的服务

### 镜像及容器

&emsp;&emsp;*容器 container* 通过运行一个 *镜像 image* 而被启动。而 **镜像** 是一个可执行的包，它包含要运行的应用所需的所有依赖，例如：代码、运行环境、库、环境变量及配置文件。

&emsp;&emsp;**容器** 是 **镜像** 运行时的具体实例，即：当 **镜像** 被执行后载入内存（它是一个内存中带状态的镜像或用户进程）。你可以通过执行 `docker ps` 命令来查看目前正在运行的容器列表，类似于 Linux 上查看进程。

### 容器及虚拟机

&emsp;&emsp;*容器 container* 运行在*原生的* Linux 上，并与其他容器共享宿主机的内核资源。它运行在一个独立的进程中，与其他可执行的其他进程一样，也不需其他额外的内存资源，这也是它具备轻量级的原因所在。

&emsp;&emsp;与容器相比，*虚拟机 Virtual Machine* 是通过 *hypervisor* 虚拟访问宿主机资源，来运行一个完整的寄生操作系统。总之，虚拟机提供了一个大而全的应用运行环境及资源，然而大多数应用可能使用不到这么多的环境资源，故而存在资源浪费。

<p>
	&emsp;&emsp;<image src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/docker/Container%402x.png" width="40%"></image>&emsp;&emsp;&emsp;&emsp;
	<image src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/docker/VM%402x.png" width="40%"></image>
</p>

## Docker 安装

* 注册 [Docker Hub](hub.docker.com) 账号

* 下载 [Docker Desktop for Mac](https://download.docker.com/mac/stable/Docker.dmg)
* 安装后，启动 Docker 应用

	附带有个快速入门示例 Demo `cheers2019`
	
	1. 克隆项目

		```
		git clone https://github.com/docker/doodle.git
		```
	2. 构建 Docker 镜像

		```
		cd doodle/cheers2019 && docker build -t docker/cheers2019 .
		```
		具体 `build` 命令的用法，执行 `docker build --help`
		
	3. 运行 Docker 镜像

		```
		docker run -it --rm docker/cheers2019
		```
		具体 `run` 命令的用法，执行 `docker run --help`
		
		运行结果如下：
		
		![Cheers2019](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/docker/docker-cheers.gif)
		
	4. 分享 Docker 镜像

		```
		docker login && docker push docker/cheers2019
		```
		如何感觉这个例子有趣，执行上面命令，将会在 [Docker Hub](hub.docker.com) 分享你的容器镜像。
		
## Docker CLI
### 检查版本
```
# docker 版本
$ docker -v
Docker version 19.03.1, build 74b1e89
	
# docker 详细信息
$ docker info
Client:
Debug Mode: false
	
# docker-compose 版本
$ docker-compose -v
docker-compose version 1.24.1, build 4667896b
	
# docker-machine 版本
$ docker-machine -v
docker-machine version 0.16.1, build cce350d7
```

### 基本命令

```
## 列出 docker 的所有命令行命令列表
docker
## 获得子命令的帮助
docker container --help
	
## 显示 docker 的版本及信息
docker --version
docker version
docker info
	
## 执行 docker 镜像
docker run hello-world
	
## 显示目前系统中的所有 docker 镜像列表
docker image ls
	
## 显示目前系统中的所有 docker 容器列表 (running, all, all in quiet mode)
docker container ls
docker container ls --all
docker container ls -aq
```

## Docker 实践

&emsp;&emsp;在 [Docker 安装](#docker-安装)中，我们通过一个小 Demo 大致感受到了采用 Docker 来编译部署及运行一个容器化的应用。在本小节我们将详细阐述 Docker 在各个阶段的主要任务的实现过程。

&emsp;&emsp;在着手以 Docker 的方式构建一个应用时，一般是要经过**应用开发**、**构建容器**、**发布服务**及**合成服务栈**的几个重要步骤。创建容器应用是基础，容器应用就可以跑在 Docker 提供的应用环境上，提供具体服务。将不同的服务巧妙地组合就能构造出复杂的服务栈，来满足实际的业务需求。所以它们的层级关系应该如图所示：

<p align="center">
	<image src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/docker/docker-layer.png" width="35%"></image>
</p>

> Containers 容器：可以视作包含应用代码及其运行时所需的依赖及环境的统称<br />
> Services 服务：分布式应用中常常指能提供某项功能的单独模块<br />
> Stacks 服务栈：将不同的服务组合堆叠从而能够完成业务需求的结构<br />


&emsp;&emsp;我们这里通过编写一个 Python 应用来实践整个过程。首先创建容器 Container，然后将其发布成服务 Service，最后组合多个 Service 形成一个服务栈 Stack。

### 创建容器 Containers

* 开发容器化应用

	&emsp;&emsp;在过去，如果要开发 Python 应用，首先得安装 Python 运行环境。Docker 方式就不同，只需获得一个可移植的 Python 运行环境的 Docker 镜像，而无需实际的安装。在应用构建时，应用所需的所有依赖都会和代码一同被打包到应用镜像中。像这种可移植镜像的声明被定义在一个叫做 `Dockerfile` 的文件中。
	
	* 创建文件 `Dockerfile`

		```
		# 使用 Docker 官方的 Python 运行期镜像当做父镜像
		FROM python:2.7-slim
		
		# 设置项目的工作目录设置为 /app
		WORKDIR /app
		
		# 将当前目录的所有内容复制到容器中的 /app 目录
		COPY . /app
		
		# 安装一些必要的依赖包，这些依赖包被指定在 requirements.txt 的文本文件中
		RUN pip install --trusted-host pypi.python.org -r requirements.txt
		
		# 开放容器的 80 端口提供给外部访问
		EXPOSE 80
		
		# 定义一个环境变量，变量名称 NAME，变量值 World
		ENV NAME World
		
		# 当容器启动时，运行 app.py
		CMD ["python", "app.py"]
		```
		
	* 编写依赖包声明文件 `requirements.txt`

		```
		Flask
		Redis
		```	
	* 编写应用代码 `app.py`

		```python
		from flask import Flask
		from redis import Redis, RedisError
		import os
		import socket
		
		# Connect to Redis
		redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)
		
		app = Flask(__name__)
		
		@app.route("/")
		def hello():
		    try:
		        visits = redis.incr("counter")
		    except RedisError:
		        visits = "<i>cannot connect to Redis, counter disabled</i>"
		
		    html = "<h3>Hello {name}!</h3>" \
		           "<b>Hostname:</b> {hostname}<br/>" \
		           "<b>Visits:</b> {visits}"
		    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)
		
		if __name__ == "__main__":
		    app.run(host='0.0.0.0', port=80)
		```
		
	&emsp;&emsp;编写完毕后，当前目录的结构如下所示：
	
		.
		└── requirements.txt		// 依赖包文件
		└── Dockerfile		// 容器定义文件
		└── app.py			// Python 代码

* 构建容器化应用镜像
	
	&emsp;&emsp;来到项目的根目录，执行如下命令构建应用：	
	```sh
	$ docker build --tag=friendlyhello .
	```
	
	&emsp;&emsp;使用`--tag` 为当前镜像设置标签，也可以使用简写 `-t`，标签的整体格式为 ***REPOSITORY_NAME*:*TAG***，其中 *REPOSITORY_NAME* 为库名称，*TAG* 为具体标签说明，可以是版本等说明信息，例如：`friendlyhello:v0.0.1`
	
	&emsp;&emsp;等待编译完成后，查看具体镜像：
	
	```sh
	$ docker image ls
	REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
	friendlyhello        latest              550941976752        9 minutes ago       148MB
	```
	
* 运行容器化应用

	&emsp;&emsp;执行下面命令，将镜像文件运行起来：
	
	```sh
	$ docker run -p 4000:80 friendlyhello
	 * Serving Flask app "app" (lazy loading)
	 * Environment: production
	   WARNING: This is a development server. Do not use it in a production deployment.
	   Use a production WSGI server instead.
	 * Debug mode: off
	 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
	172.17.0.1 - - [18/Aug/2019 15:16:26] "GET / HTTP/1.1" 200 -
	172.17.0.1 - - [18/Aug/2019 15:16:27] "GET /favicon.ico HTTP/1.1" 404 -
	```
	
	&emsp;&emsp;由于我们将`4000`端口映射为容器中的`80`端口，故而在浏览器中访问地址 `http://localhost:4000` 即可访问页面，如图所示：
	
	![app-in-brower.png](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/docker/app-in-browser.png)
	
	&emsp;&emsp;尝试执行如下命令，可以获取更多相关容器的一些信息：
	
	```sh
	# 列出当前正在运行的容器列表
	$ docker container ls
	CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                  NAMES
	6af27ed7e8d9        friendlyhello       "python app.py"     9 seconds ago       Up 8 seconds        0.0.0.0:4000->80/tcp   practical_grothendieck
	
	# 停止容器 6af27ed7e8d9 为上面展示的 CONTAINER ID
	$ docker container stop 6af27ed7e8d9
	
	```
	

* 发布容器化应用

	&emsp;&emsp;运行调试完毕，我们可以将镜像打上标签，并上传至 [Docker Hub](https://hub.docker.com) 进行镜像托管，当然你也可以选择不同的镜像托管服务器。具体操作步骤如下：
	
	```sh
	# 登录到 Docker Hub
	$ docker login
	Authenticating with existing credentials...
	Login Succeeded
	
	# 将本地镜像 friendlyhello 重新打上规范的标签
	# username 替换为你的 Docker Hub 账号用户名
	$ docker tag friendlyhello username/friendlyhello:v1.0
	
	# 重新标签后，我们能看到一个新的镜像文件
	$ docker image ls
	REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
	friendlyhello           latest              550941976752        51 minutes ago      148MB
	username/friendlyhello   v1.0                550941976752        51 minutes ago      148MB
	
	# 将新的镜像 push 到 Docker Hub
	$ docker push username/friendlyhello:v1.0
	
	# 此后，就可以在任何时候执行如下命令拉取远程镜像文件执行
	$ docker run -p 8081:80 username/friendlyhello:v1.0
	
	```

### 发布服务 Services

&emsp;&emsp;对于 Docker 而言，服务就是产品级的容器，服务指定了容器以什么方式运行，例如：容器运行在哪个端口、需要多少硬件资源、需要多少容器实例拷贝...... 所以符合 Docker 容器化的服务，天生就具备良好的可扩展性。Docker 通过 `docker-compose.yml` 文件来配置容器运行的参数，下面就上一节生成的镜像文件来编写这个文件。

* 编写 `docker-compose.yml` 文件

	```yml
	version: "3"
	services:
	  web:
	    # 根据自身的镜像信息替换 username/friendlyhello:v1.0
	    image: username/friendlyhello:v1.0
	    deploy:
	      # 容器实例个数
	      replicas: 5
	      # 资源配置
	      resources:
	        limits:
	          # 限制每个实例占用 cpu 时间为 10%
	          cpus: "0.1"
	          # 限制每个实例占用内存为 50M
	          memory: 50M
	      restart_policy:
	        condition: on-failure
	    # 端口映射，将 4000 端口映射为容器的 80 端口
	    ports:
	      - "4000:80"
	    # 将此 web 参与到负载均衡网络 webnet
	    networks:
	      - webnet
	# 采取默认方式定义一个负载均衡的网络
	networks:
	  webnet:
	```
	
* 发布具备负载均衡应用服务

	&emsp;&emsp;由于要使用负载均衡的功能，在部署时涉及后续小节的内容，现在如果不理解，先照着命令执行即可。安装如下命令发布配置好的服务：
	
	```sh
	# 将当前节点初始化为集群的管理节点，将在下节介绍。
	# 如果不初始化，下面的 docker stack 命令将会报错。
	$ docker swarm init
	Swarm initialized: current node (t6r6euslxnpwitw4yn0g92uaf) is now a manager.
	To add a worker to this swarm, run the following command:
	    docker swarm join --token ... 192.168.65.X:2377
	To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
	
	# 应用根目录执行，后面的 getstartedlab 是指定的当前服务栈的名称
	$ docker stack deploy -c docker-compose.yml getstartedlab
	Creating network getstartedlab_webnet
	Creating service getstartedlab_web
	
	# 查看当前的服务信息
	$ docker service ls
	ID                  NAME                MODE                REPLICAS            IMAGE                        PORTS
	kqibyn0286pu        getstartedlab_web   replicated          5/5                 username/friendlyhello:v1.0   *:4000->80/tcp

	# 除了上面的命令外，下面的可以只显示指定服务栈名称的服务信息
	$ docker stack services getstartedlab
	ID                  NAME                MODE                REPLICAS            IMAGE                        PORTS
	kqibyn0286pu        getstartedlab_web   replicated          5/5                 username/friendlyhello:v1.0   *:4000->80/tcp
	
	# 根据配置，当前服务运行了5个容器实例
	# 服务中的每个容器实例被称作任务 task，每个任务有一个唯一的自增长的 ID
	# 此 ID 编号一直增长到配置文件中指定的实例个数，类似下面的 getstartedlab_web.5
	# docker stack ps getstartedlab 也能显示相同任务内容
	$ docker service ps getstartedlab_web
	ID                  NAME                  IMAGE                        NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
	2j70ic6zxs55        getstartedlab_web.1   username/friendlyhello:v1.0   docker-desktop      Running             Running 4 minutes ago                       
	vretbeyraceh        getstartedlab_web.2   username/friendlyhello:v1.0   docker-desktop      Running             Running 4 minutes ago                       
	qig853sns67z        getstartedlab_web.3   username/friendlyhello:v1.0   docker-desktop      Running             Running 4 minutes ago                       
	chuipb76htpg        getstartedlab_web.4   username/friendlyhello:v1.0   docker-desktop      Running             Running 4 minutes ago                       
	mhcbn7dab9vt        getstartedlab_web.5   username/friendlyhello:v1.0   docker-desktop      Running             Running 4 minutes ago
	
	# 下面的命令显示不同的容器实例，只不过没有根据服务栈名称过滤
	# 注意：这里显示的是容器ID，上面命令是任务ID
	$ docker container ls
	CONTAINER ID        IMAGE                        COMMAND             CREATED             STATUS              PORTS               NAMES
	decd34f76501        username/friendlyhello:v1.0   "python app.py"     6 minutes ago       Up 6 minutes        80/tcp              getstartedlab_web.3.qig853sns67zlmdtpz83tl439
	eb04fb1c8146        username/friendlyhello:v1.0   "python app.py"     6 minutes ago       Up 6 minutes        80/tcp              getstartedlab_web.4.chuipb76htpgsbbk0f9e7zqnr
	4250101e9006        username/friendlyhello:v1.0   "python app.py"     6 minutes ago       Up 6 minutes        80/tcp              getstartedlab_web.2.vretbeyraceh2o7dd22jr4n20
	5a5c3a5f525a        username/friendlyhello:v1.0   "python app.py"     6 minutes ago       Up 6 minutes        80/tcp              getstartedlab_web.5.mhcbn7dab9vt5krt3vkpe35em
	4194f16e84a5        username/friendlyhello:v1.0   "python app.py"     6 minutes ago       Up 6 minutes        80/tcp              getstartedlab_web.1.2j70ic6zxs55cpekpusq4tjmp
	```
	
	&emsp;&emsp;执行完命令后，访问 URL 地址 `http://localhost:4000`。刷新几次页面，发现页面中 **Hostname** 是变化的，因为每次请求可能被负载均衡到不同的容器实例上。可以修改 `docker-compose.yml` 文件来改变应用的资源配置、实例数等参数，每次修改保存后，只要重复执行 `docker stack deploy -c docker-compose.yml getstartedlab` 命令就能更新实例。值得注意的是，Docker 平台采取一站式更新，因此你不必先停掉你的 stack 或者 kill 所有的容器，Awesome！
	
* 停止应用及集群

	```sh
	# 停止服务栈
	$ docker stack rm getstartedlab
	
	# 关闭集群
	$ docker swarm leave --force
	```

### 创建集群 Swarms

### 创建服务栈 Stacks

### 部署应用


## 脚注
[^1]: 本文参考翻译自 Docker 官方文档，阅读原文请移步 [Get started](https://docs.docker.com/get-started)。
	
