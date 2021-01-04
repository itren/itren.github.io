---
title: Centos7 安装 MySQL 5.7 教程
categories:
  - cookbook
  - database
tags:
  - mysql
date: 2021-01-04 22:17:50
---

MySQL 安装环境：

* Centos 7
* MySQL 5.7

<!--more-->

## 配置 YUM 源

* 下载 MySQL 源

```
wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
```

* 安装源文件

```
yum localinstall mysql57-community-release-el7-11.noarch.rpm
```

## 安装 MySQL

```
yum install -y mysql-community-server
```

## MySQL 常见操作

* 启动

```
systemctl start mysqld
```

* 查看状态

```
systemctl status mysqld
```

* 开机启动

```
systemctl enable mysqld
```

* 重新加载配置

```
systemctl daemon-reload
```

* 初次配置 root 密码

先使用命令查看默认生成的临时密码：

```
grep 'temporary password' /var/log/mysqld.log
```

使用临时密码登录：

```
mysql -uroot -p
```

修改 root 密码：

```
ALTER USER 'root'@'localhost' IDENTIFIED BY 'newpassword!'; 
```

* 允许 root 可以远程登录

```
use mysql;
UPDATE user SET Host='%' WHERE User='root';
flush privileges
```

* 指定用户名更新密码

```
use mysql;
update user set password=PASSWORD('newpassword!') where user='root';
flush privileges;
```

## 参考

- [1] [CentOS 7 下 MySQL 5.7 的安装与配置](https://www.jianshu.com/p/1dab9a4d0d5f)

