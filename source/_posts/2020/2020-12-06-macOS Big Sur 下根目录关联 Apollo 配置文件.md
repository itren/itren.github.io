---
title: macOS Big Sur 下根目录关联 Apollo 配置文件
categories:
  - tools
tags:
  - macOS
  - Apollo
abbrlink: f9c08d5a
date: 2020-12-06 22:27:38
---

从 macOS Catalina 开始收紧了系统目录的权限，不能像以前那样使用 root 账户在系统根目录创建子目录与文件，这种限制对于开发人员来说非常不友好。Catalina 这个版本还可以通过软连接达到目的，但是最新的 Big Sur 这种方式已经失效了，好在 Apple 的官网讨论区已经有人给出了解决方案。

<!--more-->

这次的解决方案与之前的方式类似，也是通过映射的方式实现，在 Catalina 上是通过软链接实现的，在 Big Sur 中是通过配置文件实现的，配置文件方式猜测应该是 Apple 官方支持的方式。

## 在用户目录新建文件夹

因为像 Apollo 配置中心需要使用系统目录保存配置文件，我们可以先在用户目录新建同名的配置目录。

```
/Users/{user_dir}/opt
```
上面就是我在自己用户名下新建的 opt 目录，专门用来保存 Apollo 相关的配置文件。

## 使用 root 权限新建 synthetic.conf 文件

因为要在系统根目录创建配置文件，所以需要 root权限，mac 上如何开启 root 权限可以自行搜索，下面给出 synthetic.conf 配置的示例：

```
opt	/Users/{user_dir}/opt
```

上面的 opt 就是你想在根目录创建的文件夹，后面用户目录的文件夹是实际的配置目录。需要注意的是两个目录中间是使用 "tab" 作为分隔符的，不要使用空格作为分隔符，配置完成后重启生效。

## 参考

-[1] [big sur 根目录无法创建文件夹](https://discussionschinese.apple.com/thread/252048297)







