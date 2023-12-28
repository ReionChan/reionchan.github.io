---
layout: post
title: 分布式事务框架 Seata 快速配置指南
categories: Java
excerpt: Seata 配置指南
image: https://seata.io/zh-cn/img/seata_logo.png
description: Seata 快速配置指南
keywords: Seata Quick Start Distributed Transaction Solution
licences: cc
---

<br/>

<img src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/seata/seata_logo.png" alt="Seata Logo" width="300"/>

> Seata 是一款开源的分布式事务解决方案，致力于在微服务架构下提供高性能和简单易用的分布式事务服务。


## 下载

* [Seata Server v2.0.0](https://github.com/apache/incubator-seata/releases/download/v2.0.0/seata-server-2.0.0.zip)
* 解压缩，约定之后将解压目录称为为：`seata_home`

## 目录结构

```sh
.
# 可执行文件
└── bin
│   └── seata-server.sh # Seata 启动脚本
│   └── startup.sh
│   └── seata-setup.sh
│   └── seata-server.bat
└── target
│   └── seata-server.jar # Seata 服务程序
└── script
│   └── server
│   │   └── db
│   │   │   └── mysql.sql # Seata 服务 DB 模式建表脚本
│   └── config-center
│   │   └── nacos # Seata 服务使用 Nacos 配置中心时，上传配置脚本（也可以手动拷贝配置）
│   │   │   └── nacos-config-interactive.sh
│   │   │   └── nacos-config.py
│   │   │   └── nacos-config.sh
│   │   │   └── nacos-config-interactive.py
│   │   └── config.txt # Seata 服务配置（可手动将此内容拷贝到 Nacos）
│   │   └── README.md
└── lib
└── conf
│   └── logback-spring.xml
│   └── application.example.yml # Seata 服务程序配置文件（模板）
│   └── application.yml # Seata 服务程序配置文件
│   └── application.raft.example.yml
```

## 配置

&emsp;&emsp;Seata 服务端是基于 SpringBoot，故其配置主要是修改 `seata_home/conf/application.yml` 文件，默认为：

```yaml
server:
  port: 7091

spring:
  application:
    name: seata-server

logging:
  config: classpath:logback-spring.xml
  file:
    path: ${log.home:${user.home}/logs/seata}
  extend:
    logstash-appender:
      destination: 127.0.0.1:4560
    kafka-appender:
      bootstrap-servers: 127.0.0.1:9092
      topic: logback_to_logstash
      
# Seata Web 控制台默认用户名密码
console:
  user:
    username: seata
    password: seata
seata:
  # Seata 配置加载方式，支持：nacos, consul, apollo, zk, etcd3
  config:
    type: file
  # Seata 服务注册方式，支持：nacos, eureka, redis, zk, consul, etcd3, sofa
  registry:
    type: file
  # Seata 服务存储方式，支持：file 、 db 、 redis 、 raft
  store:
    mode: file
  
  security:
    secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
    tokenValidityInMilliseconds: 1800000
    ignore:
      urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.jpeg,/**/*.ico,/api/v1/auth/login,/metadata/v1/**
```

&emsp;&emsp;对其配置一般关注三点：**配置方式**、**服务注册方式**、**存储方式**。下面示例演示将 Seata 的配置放入 Nacos 并同时将服务也注册到 Nacos，然后使用 ***db*** 存储模式。

&emsp;&emsp;本指南依赖 Nacos 服务、MySQL  数据库，请自行安装配置。它们的具体信息如下：

* Nacos 2.2.3
  * localhost:8848
  * 无需用户名密码认证
* MySQL 5.1.x
  * localhost:3306
  * 用户名、密码：root root

### seata.config 配置方式（Nacos）

* Nacos 配置设置

  1. 新建名称为 `seata` 的命名空间，产生**唯一命名空间 ID** 

     ![seata-nacos-namespace](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/seata/seata-nacos-namespace.png)

  2. 上传 Seata 配置文件到 Nacos

     上传有两种方式，脚本方式和手动创建方式，脚本方式将把每一条配置生成一个 Data Id，具体参考 `seata_home/script/config-center/README.md` 说明，本例采用手动方式，并将所有配置放入一个 Data Id 之中。

     ![seata-nacos-config](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/seata/seata-nacos-config-1.png)

     在配置管理中点击***创建配置***，并设置 `seata-server`、`SEATA_GROUP` 的 ***Data ID*** 和 ***Group***, 同时选择 ***Properties*** 配置格式，并将 `seata_home/script/config-center/config.txt` 配置内容拷贝到 ***配置内容***，然后点击 ***发布***：

     ![seata-nacos-config](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/seata/seata-nacos-config-2.png)

     

* Seata 配置

  修改 `seata_home/conf/application.yml` 中 **seata.config** 下的内容为：

  ```yaml
  seata:
    config:
      type: nacos
      nacos:
        # Nacos 服务地址 
        server-addr: 127.0.0.1:8848
        # Nacos 命名空间
        namespace: 8b4a9073-dc11-4302-aa02-6c02fc6d81fe
        # 分组
        group: SEATA_GROUP
        # 数据ID
        data-id: seata-server
        username:
        password:
        context-path:
  ```

### seata.registry 注册方式（Nacos）

&emsp;&emsp;修改 `seata_home/conf/application.yml` 中 **seata.registry** 下的内容为：

```yaml
seata:
  registry:
    type: nacos
    nacos:
      # Seata 服务注册到 Nacos 时服务名称
      application: seata-server
      # Nacos 服务地址
      server-addr: 127.0.0.1:8848
      # 注册分组，本例使用默认分组
      group: DEFAULT_GROUP
      # 不填，默认使用 public 命名空间
      namespace:
      # 集群名称，使用默认
      cluster: default
      username:
      password:
      context-path:
```

### seata.store 存储方式（DB，MySQL）

&emsp;&emsp;先在 MySQL 数据库创建名为 *seata* 的数据库

&emsp;&emsp;打开 `seata_home/server/db/mysql.sql` 文件，并在数据库中执行：

```sql
-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_status_gmt_modified` (`status` , `gmt_modified`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL,
    `xid`               VARCHAR(128) NOT NULL,
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32),
    `resource_id`       VARCHAR(256),
    `branch_type`       VARCHAR(8),
    `status`            TINYINT,
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(128),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `status`         TINYINT      NOT NULL DEFAULT '0' COMMENT '0:locked ,1:rollbacking',
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_status` (`status`),
    KEY `idx_branch_id` (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

CREATE TABLE IF NOT EXISTS `distributed_lock`
(
    `lock_key`       CHAR(20) NOT NULL,
    `lock_value`     VARCHAR(20) NOT NULL,
    `expire`         BIGINT,
    primary key (`lock_key`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('AsyncCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryRollbacking', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('TxTimeoutCheck', ' ', 0);
```

&emsp;&emsp;修改 `seata_home/conf/application.yml` 中有 **seata.store** 下的内容为：

```yaml
seata:
  store:
    mode: db
    session:
      mode: db
    lock:
      mode: db
    db:
      datasource: druid
      # 数据库类型
      db-type: mysql
      # 数据库驱动
      driver-class-name: com.mysql.jdbc.Driver
      # 数据库连接地址
      url: jdbc:mysql://127.0.0.1:3306/seata?rewriteBatchedStatements=true&useSSL=false&useUnicode=true&characterEncoding=utf8&serverTimezone=GMT%2B8
      user: root
      password: root
      min-conn: 10
      max-conn: 100
      global-table: global_table
      branch-table: branch_table
      lock-table: lock_table
      distributed-lock-table: distributed_lock
      query-limit: 1000
      max-wait: 5000
```

## 启动

&emsp;&emsp; 终端切换到 `seata_home/bin` 目录，执行如下命令：

```sh
# 执行如下命令
sh seata-server.sh -p 8091 -h 127.0.0.1 -m db
```

&emsp;&emsp;浏览器输入地址：http://localhost:7091，输入用户名 *seata* 密码 *seata* 即可进入 Web 控制台：

![seata-web-console](https://raw.githubusercontent.com/ReionChan/PhotoRepo/master/seata/seata-web-console.png)

## 推荐阅读

* [Seata 快速开始](https://seata.io/zh-cn/docs/user/quickstart/)

* [Seata 开发者指南](https://seata.io/zh-cn/docs/dev/mode/at-mode)
