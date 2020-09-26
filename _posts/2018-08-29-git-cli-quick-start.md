---
layout: post
title: Git CLI 快速入门
categories: Tool
excerpt: Git CLI 实用命令快速上手
image: https://git-scm.com/images/logo@2x.png
description: Git CLI 实用命令快速上手
keywords: git GIT CLI 版本控制
licences: cc
---

![git](https://git-scm.com/images/logo@2x.png)

### 帮助文档
	
	man git-log
	git help log
	
### 设置 name & email

	git config --global user.name "your name"
	git config --global user.email you@example.com

### 导入新工程
	
	cd your_project_path
	git init
		创建一个空的 Git 仓库 .git/
	git add .
		为目录下的所有文件创建一个快照，存储在 `index` 区
	git commit
		要求输入提交信息
		
### 更改
	git add file1 file2 file3
		将三个文件的更改添加到 `index` 区
	git diff --cached
		比较将要提交的做了哪些改变，不加 cached 不会显示
		未通过 add 添加到 index 区的改动
	git status
		查看简要概括信息
	git commit
		提交index区变动到库，后接 -a 参数，可以掠过 git add 操作
		提交备注信息是个好习惯，一般要求简短，50个字以内的标题  
		后接一个空行，再最佳一些说明。
		
### 查看历史
	git log
		查看变更记录，-p 查看每一步的全部不同之处
		--stat --summary 可以对每一步有一个感官上的概览
		
### 管理分支
	git branch experimental
		创建名为 experimental 的分支
	git branch
		将列出所有已经存在的分支
		* 号表示目前所处的分支
	git checkout experimental
		切换到 experimental 分支
	git merge experimental
		如果在主、分支中分别修改了同一文件不同的地方
		将会把分支上的变动合并到主分支，如果有冲突
		将会有冲突标记在冲突位置
	git diff
		将会显示冲突位置，可以通过编辑解决冲突
		git commit -a 将最终提交最终解决冲突后的变动
	gitk
		将会展示一个友好的图形化的历史结果
	git branch -d experimental 
		将会删除该分支，应确保分支的更改已经存在于当前的分支中
		-D 如果仅让分支做一些实验性的尝试，此命令忽略所有变动
		
### 协同合作
#### 本地库共享

	git pull  /home/bob/myrepo master
		将 bob 项目 master 分支中的更改拉取到当前操作的的分支
		如果当前操作分支本身也有变动，那么就要手动处理冲突
		pull 包含两操作：从远程分支获取改动，合并到当前分支
		
		如果 bob 的更改与当前已更改的有冲突，
		将会使用工作树和 index 来解决冲突，已经存在的本地更改
		将会干涉冲突解决处理，git 仍会拉取 bob 的更改，
		但不会合并，应该采取一些方式放弃当前本地更改，
		然后再次尝试 pull
		
	git fetch /home/bob/myrepo master
	git log -p HEAD..FETCH_HEAD
		fetch 可以预览 bob 的更改而无需合并，这项操作是安全的
		即使当前本地有未提交的变动。
		“HEAD..FETCH_HEAD”表示展示所有从 FETCH_HEAD
		可到达的开始但不包括 HEAD 可到达的。
	
	gitk HEAD..FETCH_HEAD
		将查看 bob 自他 forked 以来所有更改
		使用三个点的范围，将查看他们自 fork 以来所有更改
		
	gitk HEAD...FETCH_HEAD
		表示查看他们中任意一个可达的更改，但不包括他们都可达的
		部分
		范围标记可以被 gitk 和 git log 两个命令使用
		
#### 远程库共享
	
	git remote add bob /home/bob/myrepo
		远程仓库添加 bob 的更改
	git fetch bob
		使用远程库的简略写法来 fetch bob 的 master 分支
	git log -p master..bob/master
		查看当前master分支与bob 从当前master分支后的变动
	git merge bob/master
		将 bob 的变动合并到当前master分支，也可以
		git pull . remotes/bob/master
	之后，bob 可以更新它的库来使用当前合并的更新，bob 命令
		git pull
		注意，bob不需要给定他所fork的库的路径，因为 bob 克隆
		库是，git 已经保存了库路径
		
		git config --get remote.origin.url
		即可查看该克隆原始路径地址
		
		git 同样保存一个全新的原始库的拷贝
		git branch -r 即可查看
		
### 浏览历史

	git log
	git show commit_id
	git show HEAD 当前分支的顶端
	git show experimental 名为experimental分支的顶端
	
	git show HEAD^ 
		查看 HEAD 的上一次提交
	git show HEAD^^
		查看 HEAD 的上上一次提交
	git show HEAD~4
		查看 HEAD 的上上上上次提交
		
	合并提交可能有多个上一次提交
	git show HEAD^1
		查看 HEAD 的第一个上一次提交
	git show HEAD^2
		查看 HEAD 的第二个上一次提交
	
	git tag v2.5 1b2e1d63ff
		给 1b2e1d63ff 提交命名为 v2.5
	git diff v2.5 HEAD
		比较当前 HEAD 与 v2.5 的不同
	git branch stable v2.5
		基于 v2.5 创建一个新的分支，名为 stable
	git reset --hard HEAD^
		重置当前分支和工作目录到 HEAD^
		谨慎使用，将会丢失所有当前目录的更改
		并且删除所有最近在这个分支上的提交
		它会导致其它开发者不变要的合并来清理历史
		如果想要取消所有已经推送的更改，使用
		git revert
	git grep "hello" v2.5
		在 v2.5 上搜索字符串 hello
		
	git log v2.5..v2.6
	git log v2.5..
	git log --since="2 weeks ago"
	git log v2.5.. Makefile 浏览至v2.5以来 Makefile 文件所有变动
	
	gitk --since="2 weeks ago" drivers/
		列出至两个星期以来 drivers 目录下的所有更新
		
### Git GUI 客户端推荐

* [GitHub Desktop](https://desktop.github.com/) `快捷`

	![GitKraken](https://desktop.github.com/images/github-desktop-screenshot-mac.png)
	
* [GitKraken](https://www.gitkraken.com/) `酷炫`

	![GitKraken](https://www.gitkraken.com/img/index/gk-product-2.png)