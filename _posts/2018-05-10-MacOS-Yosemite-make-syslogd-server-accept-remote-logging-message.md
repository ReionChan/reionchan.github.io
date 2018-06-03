---
layout: post
title: Yosemite 开启日志服务器
categories: MacOS
excerpt: 使 syslogd 接收远程日志消息
image: https://developer.apple.com/assets/elements/icons/mac-os/mac-os.svg
description: 使 syslogd 接收远程日志消息
keywords: MacOS, Yosemite, syslog, syslogd, server, remote, message
licences: cc
---

## 配置

&emsp;&emsp; MacOS Yosemite 开启日志服务器，允许远程日志消息打印到系统控制台，可执行如下操作：

* 打开 UDP 日志接收端口：
```shell
cd /System/Library/LaunchDaemons/
sudo cp com.apple.syslogd.plist com.apple.syslogd.plist.bak
sudo /usr/libexec/PlistBuddy -c "add :Sockets:NetworkListener dict" com.apple.syslogd.plist
sudo /usr/libexec/PlistBuddy -c "add :Sockets:NetworkListener:SockServiceName string syslog" com.apple.syslogd.plist
sudo /usr/libexec/PlistBuddy -c "add :Sockets:NetworkListener:SockType string dgram" com.apple.syslogd.plist
sudo launchctl unload com.apple.syslogd.plist
sudo launchctl load com.apple.syslogd.plist
```
<center>
    <img src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/syslog/OS_X-Yosemite_enable_syslog_server.png" alt="Added Nodes"  width="400"> <br /> com.apple.syslogd.plist 新增的节点<br /><br /></center>


* 查看 UDP 端口是否已经打开：  
```shell
# 输出含 UDP 端口号 514，表示配置成功  
sudo lsof -i :514 -P  
```  

## Log4j 2 实践  
  
  
```properties
# Syslog Appender 定义
appender.sys.type = Syslog
appender.sys.name = syslog
appender.sys.format = RFC5424
appender.sys.host = Server_IP
appender.sys.port = 514
appender.sys.protocol = UDP
appender.sys.includeMDC = true
appender.sys.layout.type = PatternLayout
appender.sys.layout.pattern = %d{yyyy-MM-dd HH:mm:ss,SSS} [%t] %-5p %c - %m%n  
# 根日志配置成 syslog 的 Appender  
rootLogger.appenderRef.sys.ref = syslog
```


