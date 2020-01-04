---
title: Presto 向分区表快速插入数据时出现“target directory already exists”的原因
categories:
  - bigdata
tags:
  - presto
abbrlink: d99010b6
date: 2017-05-01 23:29:53
updated: 2020-01-04 13:02:59
---

因为项目使用Presto作为ETL使用，需要将关系库中的数据导入到Hive中。目前关系库中的数据每天导入一次，在Hive中以天为间隔创建新的分区。思路是正确的，但是在使用的过程中，发现将少量关系库中的数据通过Presto快速并多次导入到Hive中时会出现如下错误：

<!--more-->

    com.facebook.presto.spi.PrestoException: Unable to rename from hdfs://cloud171:8020/tmp/presto-root/34923b62-7933-46f8-b016-8b05c7a6dd0e/liutest01_dept_part=2017-08-04 00%3A00%3A00.0 to hdfs://cloud171:8020/user/hive/warehouse/liutest01_dept/liutest01_dept_part=2017-08-04 00%3A00%3A00.0: target directory already exists
        at com.facebook.presto.hive.metastore.SemiTransactionalHiveMetastore.renameDirectory(SemiTransactionalHiveMetastore.java:1543)
        at com.facebook.presto.hive.metastore.SemiTransactionalHiveMetastore.access$2500(SemiTransactionalHiveMetastore.java:81)
        at com.facebook.presto.hive.metastore.SemiTransactionalHiveMetastore$Committer.prepareAddPartition(SemiTransactionalHiveMetastore.java:984)
        at com.facebook.presto.hive.metastore.SemiTransactionalHiveMetastore$Committer.access$700(SemiTransactionalHiveMetastore.java:819)
        at com.facebook.presto.hive.metastore.SemiTransactionalHiveMetastore.commitShared(SemiTransactionalHiveMetastore.java:749)
        at com.facebook.presto.hive.metastore.SemiTransactionalHiveMetastore.commit(SemiTransactionalHiveMetastore.java:671)
        at com.facebook.presto.hive.HiveMetadata.commit(HiveMetadata.java:1312)
        at com.facebook.presto.hive.HiveConnector.commit(HiveConnector.java:177)
        at com.facebook.presto.transaction.TransactionManager$TransactionMetadata$ConnectorTransactionMetadata.commit(TransactionManager.java:578)
        at java.util.concurrent.CompletableFuture$AsyncRun.run(CompletableFuture.java:1626)
        at io.airlift.concurrent.BoundedExecutor.drainQueue(BoundedExecutor.java:77)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

因为之前看过Presto的一些源码，对于上述错误的原因因该是Presto中元数据的缓存造成的。了解Hive的读者因该知道，Hive中的每张表都对应一个文件夹，分区也不例外，如果要新建分区的话会在该表的目录下新建文件夹，文件夹的名称为该分区的名称，对于一些特殊的字符串Hive会做处理，避免非法的文件名。当Presto向Hive中插入数据时会首先在内存中查看当前表的元数据，如果已经有了这个分区就直接向这个分区插入新的数据，如果没有这个分区首先会将数据写入到Hive的tmp目录下，等操作结束后修改为目标路径。但是Presto的缓存不可能这么快的感知到Hive中元数据的变化，第二次向这个分区插入数据时，因为它依然认为该表没有这个分区，又会执行新建分区的操作，文件夹是不可能重名的，所以会抛出这个异常。这个猜想也很好证实，首次创建这个分区之后等几秒在对这个分区做数据的插入操作就不会出现这个异常了，因为Presto已经刷新了缓存。

![](https://www.itren.tech/2017/media/15781142609533.jpg)