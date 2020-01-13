---
title: Canal Admin 搭建 Canal 集群以及体验
categories:
  - etl
tags:
  - canal-admin
  - canal
abbrlink: 99c08147
date: 2020-01-10 18:03:03
---

Canal 是阿里巴巴开源的一套分布式数据库同步系统，目前主要支持 MySQL、RDS。Canal 的主要原理是伪装成 MySQL 的 Slave 节点，监听 MySQL 主库的 binlog 文件，根据 binglog 将数据库事件发送到 MQ 中，消费端可以订阅 MQ 中的消息。为了方便 Canal 的运维人员，阿里还提供了 Canal Admin 这个运维平台，使用户可以快速和安全的操作。

<!--more-->

## 准备环境

Canal Admin 的使用需要依赖 MySQL、Canal 、Zookeeper 这三个服务，在使用 Canal Admin 之前我已经体验过单机节点的搭建，这里列一下三个服务使用的版本：

1. MySQL 5.6
2. Canal 1.1.4
3. Zookeeper 3.4.10

为了更加方便的了解 Canal Admin 的作用，下面根据理解画了一下整体的架构图，根据架构图可以帮助我们理解整个系统运行的逻辑，出现问题也更加方便排查。

![](https://itgrocery.cn/2020/media/15786508860209.jpg)

## 部署 Canal Admin

Canal Admin 是跟随着 Canal 一起发布的，可以去 Canal 的 Release 模块下面下载对应版本的 Canal Admin，我这里下载的是”canal.admin-1.1.4.tar.gz“。

### 配置 application.xml

当压缩包下来之后进行加压缩，比较重要的配置 conf 目录下面的 application.yml 文件，里面涉及到 Canal Admin 的数据库连接配置以及管理员信息。

```
server:
  port: 8089
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8


spring.datasource:
  address: 127.0.0.1:3306
  database: canal_manager
  username: root
  password: root
  driver-class-name: com.mysql.jdbc.Driver
  url: jdbc:mysql://${spring.datasource.address}/${spring.datasource.database}?useUnicode=true&characterEncoding=UTF-8&useSSL=false
  hikari:
    maximum-pool-size: 30
    minimum-idle: 1


canal:
  adminUser: admin
  adminPasswd: 123456
```
### 初始化 MySQL

因为 Canal Admin 是一个管理系统，需要使用数据库存放配置信息，只用在 MySQL 中执行 Canal Admin 提供的数据库初始化文件即可，该文件在“conf/canal_manager.sql”路径下面。

### 新建集群

上面的 Canal Admin 配置好了之后直接根据“/bin/startup.sh”启动 Canal Admin 即可，在浏览器上面输入"ip:8089"即可进入到管理页面，如果使用的默认的配置信息，用户名入"admin"，密码输入"123456"即可访问首页。

进入到首页点击集群的菜单栏，然后选择新建集群。

![](https://itgrocery.cn/2020/media/15786509043984.jpg)

在里面输入集群的名称以及 Zookeeper 即可，这里的集群目前还没有任务节点，后续通过配置 Canal Server 的自动注册功能，便可以查看该集群下面拥有的节点。

## 启动 Canal Server

因为这里使用 Canal Admin 部署集群，所以 Canal Server 节点只需要关注 manager 相关的信息即可，具体的任务信息后续都通过 Canal Admin 下发，这一点与单机部署区别很大。

在 Canal 的配置目录下，有两个 canal 前缀的配置项，其中一个文件名是"canal_local.properties"，这是 Canal Admin 官网介绍的集群部署需要修改的配置文件，里面的配置信息相比"canal.properties"要少很多，多余的信息在集群模式下都由 Canal Admin 管理。

```
# register ip

canal.register.ip = 10.37.129.3




# canal admin config

canal.admin.manager = 10.37.129.3:8089

canal.admin.port = 11110

canal.admin.user = admin

canal.admin.passwd = 6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9

# admin auto register

canal.admin.register.auto = true

canal.admin.register.cluster = xxx_cluster
```
上面配置信息中有几个地方要注意一下，这里简短介绍一下。第一个是"canal.register.ip"，这个配置用来指定当前 Canal Server 的 IP 信息，如果主机是多网卡，可以避免 IP 信息错乱的问题。第二个是"canal.admin.passwd"，这里的密码就是之前配置 Canal Admin 里面配置的管理员密码，只不过这里并不是明文展示，使用 MySQL 的"select password("123456")"语句查询处理过的密码，注意查询结果前面的"*"要去掉。第三个是"canal.admin.register.auto"，这里是自动注册的意思，如果没有配置，Canal Server 启动后需要自行在 Canal Admin 上面添加。第四个是"canal.admin.register.cluster"，这个配置如果不写代表当前的 Canal Server 是一个单机节点，如果添加的名字在 Canal Admin 上面没有提前注册，Canal Server 启动时会报错。

如果你的集群需要部署多个 Canal Server，将上面的配置复制到另外几台机器上面，主要别忘记修改 IP 信息，配置好所有的节点之后启动即可，这些节点会自动注册到 Canal Admin。

## 配置 Canal Server

通过上面的操作之后所有的 Canal Server 便可以在 Canal Admin 上面看到，接着可以通过 UI 界面配置 Canal Server。

![](https://itgrocery.cn/2020/media/15786510267775.jpg)

从上面的截图可以看到两个节点都归属于同一个集群，如果我们点击单个节点编辑配置，Admin 会提示我们集群模式下不允许修改单个节点的配置，需要在集群下面修改配置。

![](https://itgrocery.cn/2020/media/15786510367353.jpg)

下面通过集群管理页面修改 Canal Server 的配置，主要是添加 Canal Server 需要对接的 MQ。一开始进入的时候是空白的，我们可以选择载入模板，然后根据模板修改自己关注的配置。为了对接 MQ，需要在配置中指定"canal.serverMode"，我这里配置的是“RocketMQ”，另外一个配置就是 MQ 的连接信息“canal.mq.servers”。

指定消息队列为 RocketMQ。

![](https://itgrocery.cn/2020/media/15786510463073.jpg)

配置 RocketMQ 的连接信息。

![](https://itgrocery.cn/2020/media/15786510581747.jpg)

## 创建 Canal Instance

Canal Admin 提供了 Canal Instance 的管理功能，我们可以通过 UI 界面添加需要监听的数据库，让该 Instance 消费 binlog 并将事件发送到 MQ。

点击“新建 Instance”按钮创建 Instance。

![](https://itgrocery.cn/2020/media/15786510677512.jpg)

修改"canal.instance.mysql.slaveId"和"canal.instance.master.address"。

![](https://itgrocery.cn/2020/media/15786510768090.jpg)

查看 Instance 日志，判断 Instance 是否配置正确。

```
2020-01-10 14:46:50.800 [destination = parallels_mysql5.6 , address = /10.37.129.3:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - ---> find start position successfully, EntryPosition[included=false,journalName=mysql-bin.000007,position=104805,serverId=1,gtid=,timestamp=1578638732000] cost : 7ms , the next step is binlog dump
```

## 测试 MySQL 事物

创建好了 Canal Instance 之后，就可以通过消费者消费 MQ 里面的数据库事件了。对于普通的操作例如新增、修改、删除等操作都是没有问题，下面可以来测试一下事物比较特殊的场景。其实只需要验证未提交的事物是否会产生 binlog 即可推测出结果。

### 关闭自动提交

为了测试事物，可以将当前会话的自动提交关闭。

```
set @global.autocommit=0;
```
查询当前自动提交是否关闭。

```
set @@session.autocommit=0;
```

### 开始测试事物

当我们关闭自动提交之后就要定位 binlog 当前最后的位置，这样后续写入数据时但是没有提交时可以判定 binlog 是否有写入。

查找最后一个 binlog 的文件名。

```
show binary logs;
```
查看最后一个 binlog 最后的位置。

```
show binlog events in 'mysql-bin.000007';
```
我这里查找到的结果是 106667。

![](https://itgrocery.cn/2020/media/15786510995237.jpg)

执行数据库插入语句。

```
INSERT INTO `test`.`tb_user`(`username`, `password`, `age`, `nickname`) VALUES ('testtttttttt', 'tttttttttt', 3, 'ttttttttttt');
```
再次查看 binlog 最后的位置。

![](https://itgrocery.cn/2020/media/15786511093998.jpg)

可以看到位置没有发生变化，我的消费端也没有收到数据添加的消息，最后将该次事物提交，然后查看下最后的位置。

![](https://itgrocery.cn/2020/media/15786511178983.jpg)

可以看到 binlog 的位置发生了变化，证明只有事物提交之后才能 binlog 才会写入，我的消费端也接收到了数据库变化的消息。

![](https://itgrocery.cn/2020/media/15786511271747.jpg)