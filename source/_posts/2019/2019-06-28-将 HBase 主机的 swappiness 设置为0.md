---
title: 将 HBase 主机的 swappiness 设置为0
categories:
  - bigdata
tags:
  - hbase
abbrlink: 9900b865
date: 2019-06-28 23:29:53
updated: 2020-01-04 13:35:57
---

HBase是一个对内存比较敏感的存储系统，所以需要将Linux系统的交换内存设置为0，否则当系统内存不足时如果出现内存交换会造成HBase与Zookeeper之间的会话超时，比较明显的问题就是ReginServer会经常挂掉。

<!--more-->

*   将Swappines临时设置为0

```
root# sysctl -w vm.swappiness=0
```

*   将Swappines永久设置为0

```
root# echo "vm.swappiness = 0" >> /etc/sysctl.conf
```

* * *

上述配置摘自：https://www.packtpub.com/mapt/book/big_data_and_business_intelligence/9781849517140/8/ch08lvl1sec05/setting-vm.swappiness-to-0-to-avoid-swap

* * *