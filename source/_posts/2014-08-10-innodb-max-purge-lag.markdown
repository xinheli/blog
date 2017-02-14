---
layout: post
title: "innodb_max_purge_lag"
date: 2014-08-10 15:27:28 +0800
comments: true
categories: MySQL
---

##现象：

线上数据库每个表分配一个ibdata,但是总的ibdata文件很大，超过10G，用相关工具查看，大部分空间都是undo_log

分析了ibdata1的记录
``` mysql
Total number of page: 2398464: 2.4M的page * 16K = 38G
Insert Buffer Free List: 2659
B-tree Node: 5720
Freshly Allocated Page: 12725
Undo Log Page: 2352027
File Segment inode: 25333
```
可以看到绝大部分的空间，都是被undo log占用了....
<!--more-->
##分析：

undo log 过大 看来是Mysql的一个“problematic feature” 从03年到现在一直都有人抱怨这个问题。有补丁(需要自己修改源码编译)，但是不知道是否可靠... 参考[这里](http://bugs.mysql.com/bug.php?id=57611)和[这里](http://bugs.mysql.com/bug.php?id=1341)

出现的原因的一个可能原因：purge 线程赶不上速度，没有即使回收不用的undo page。可以增加一些. 这[里面](http://bugs.mysql.com/bug.php?id=45173 )有人提出一些优化参考的方案：

1，增大`innodb_io_capacity` （mysq 5.0不支持 5.1才有）

2，设置独立的`purge thread` mysql 5.0不支持，5.5才有

调节 `innodb_max_purge_lag` 这个值表明，当purge赶不上写操作的时候，写操作delay的时间指标，我们是0，表示不等待，有可能大并发写，purge落后

查看了一下文档，结论：不推荐用`innodb_max_purge_lag`来实现undo log扩充的问题。

####主要原因如下：

1，`innodb_max_purge_lag`调节参数不好设定，调整不好会强烈影响到正常insert,update的时效，得不偿失

2，`innodb_max_purge_lag`，[有人](http://mysqldump.azundris.com/archives/81-DELETE,-innodb_max_purge_lag-and-a-case-for-PARTITIONS.html)实验证明，该参数调节影响不是很大，对delete,insert没有作用，副作用大，强烈不推荐 

####undo log较大的原因是：

1，Mysql 每10S操作一次purge

2，每次purge mysql做多回收20 page 的undo log

如果10S之内删除,update的数据操作20 page，也就是320K的东西,就会出现purge 回收不及时的情况，就会出现undo log过大。

对应业务数据表，数据长度为292 因此只要1S删除超过100行，就会出现上述情况。因此只要线上大规模 delete 数据就会出现删除不干净的情况。

##解决方案：

1，delete操作，脚本控制，不要一口气删除感觉，要sleep，控制在1S删除100条的速度  这个已经证明是一个非常好的方案。删除短信数据的时候，如果速度过快，ibdata显著增加，如果控速适当，该文件是根本不会增加的

2, 如果是全表删除，推荐truncate，ddl不会写undo

3, 如果是delete + where 删除的需求，也可以考虑建立新表，导入部分旧表数据＋truncate 旧表的方式，

4, 业务层面支持分表操作，彻底去掉删除大表数据这种事情

