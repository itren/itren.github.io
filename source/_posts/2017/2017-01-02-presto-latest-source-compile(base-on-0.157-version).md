---
title: Presto 最新源码编译（基于 0.157 版本）
categories:
  - bigdata
tags:
  - presto
abbrlink: fa36cdd2
date: 2017-01-02 23:29:53
updated: 2020-01-01 21:24:35
---

Presto 在国内主要是京东和美团在使用，其中 Presto 在中国的官网是由京东维护的。当然，京东为了满足自己的需求对原生的 Presto 进行了一些改在，所有京东版的 Presto。我这边文章介绍的是原生 Presto 的编译流程，也就是 Facebook 公司的 Presto。

<!--more-->

## 1. 去 GitHub 查看 Presto 的[主页](https://github.com/prestodb/presto) 查看说明文档

我们来看看主页上边的说明

* * *

Requirements

Mac OS X or Linux
Java 8 Update 60 or higher (8u60+), 64-bit
Maven 3.3.9+ (for building)
Python 2.4+ (for running with the launcher script)

* * *

从“Requirements”我们可以知道编译需要在 Mac OS X or Linux 环境下编译，所以 Presto 的源码是无法在 Windows 环境下编译成功的。**当然有人会想，我只是想看那看源代码，博主当初也是这么想的，将 Presto 的源码导入到 IntelliJ IDEA 中，结果因为没有编译导致源码中缺失很多的类，具体原因还不知道**，所以要想查看完整的源码需要先编译 Presto 再导入到 IDE（我使用的是 eclipse ）中查看。

## 2. 准备编译环境

安装 maven，版本3.3.9以上，官网这么说明的，照着做就行了。至于 maven 怎么安装读者可以自行百度，跟安装 JDK 差不多，我安装的就是3.3.9版本。

```
[root@cdh5 presto]# mvn -version
    Apache Maven 3.3.9 (bb52d8502b132ec0a5a3f4c09453c07478323dc5; 2015-11-11T00:41:47+08:00)
    Maven home: /root/app/apache-maven-3.3.9
    Java version: 1.8.0_111, vendor: Oracle Corporation
    Java home: /usr/java/jdk1.8.0_111/jre
    Default locale: zh_CN, platform encoding: UTF-8
    OS name: "linux", version: "2.6.32-642.11.1.el6.x86_64", arch: "amd64", family: "unix"
```

安装 python，版本2.4以上，安装请自行百度，我的版本是2.6 。

```
[root@cdh5 presto]# python -V
    Python 2.6.6
```

安装 git，这个官网上没有说，但是我们要使用 git 将代码从 GitHub 上 clone 下来。**需要说明的是最好使用 clone 方式下载源码，如果你下载的是 ZIP 压缩包，那么源码中是不会有 .git 文件夹的，这也会导致 Presto 编译失败**。我的 git 版本是1.7.1。

```
[root@cdh5 presto]# git --version
    git version 1.7.1
```

## 3. 开始编译 Presto

进入 Presto 源码的根目录

```
[root@cdh5 presto]# ls
    CONTRIBUTING.md  presto-base-jdbc         presto-client        presto-hive-hadoop2  presto-mongodb         presto-raptor                   presto-spi                      src
    LICENSE          presto-benchmark         presto-docs          presto-jdbc          presto-mysql           presto-rcfile                   presto-teradata-functions       target
    mvnw             presto-benchmark-driver  presto-example-http  presto-jmx           presto-orc             presto-record-decoder           presto-testing-server-launcher
    pom.xml          presto-blackhole         presto-hive          presto-kafka         presto-parser          presto-redis                    presto-tests
    presto-accumulo  presto-bytecode          presto-hive-cdh4     presto-local-file    presto-plugin-toolkit  presto-resource-group-managers  presto-tpch
    presto-array     presto-cassandra         presto-hive-cdh5     presto-main          presto-postgresql      presto-server                   presto-verifier
    presto-atop      presto-cli               presto-hive-hadoop1  presto-ml            presto-product-tests   presto-server-rpm               README.md

```

执行编译命令

```
mvn clean install -DskipTests
```

“DskipTests”可以跳过 presto 中的一些单元测试，加快编译速度。如果你的 maven 下载速度很慢，你可以自行配置 maven 的中央仓库。我配置的阿里云的 maven 仓库，速度杠杠的，配置方法也很简单，读者可以自行百度配置方法。

* * *

一般 maven 编译成功是不会出现 error 的日志信息的，读者可以对比判断自己的源码是否编译成功。
* * *

## 4. 在 eclipse 中运行 presto

> 我们首先将编译后的 presto 以 maven 工程的形式导入到 eclipse ，但如的过程中 eclipse 会安装一些 maven 的插件，具体是什么博主也不怎么清楚，如果网络没什么问题慢慢等待就是了。

找到 presto 的主函数
目录结构如下：

![](2017-01-02-presto-latest-source-compile(base-on-0.157-version)/15778854788139.jpg)

设置要运行的 main 方法

![](2017-01-02-presto-latest-source-compile(base-on-0.157-version)/15778855025192.jpg)

设置“VM Options”，这个官网也有说明，我就复制一下，因为 presto 运行需要一些配置信息，所以这里要设置一下。

```
-ea -XX:+UseG1GC -XX:G1HeapRegionSize=32M -XX:+UseGCOverheadLimit -XX:+ExplicitGCInvokesConcurrent -Xmx2G -Dconfig=etc/config.properties -Dlog.levels-file=etc/log.properties
```

![](2017-01-02-presto-latest-source-compile(base-on-0.157-version)/15778855232958.jpg)

设置完之后点击运行就可以了，如果有什么异常读者可以根据 eclipse 的日志来分析原因，博主将这些设置好了之后可以正常运行。

![](2017-01-02-presto-latest-source-compile(base-on-0.157-version)/15778855416967.jpg)

可以看到控制台的日志中“SERVER STARTED”的字样，这说明 presto 已经正常启动了，和我们通过部署的方式运行 presto 所打印出来的日志相同。