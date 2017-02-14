---
layout: post
title: "MySQL的热备的全局锁库"
date: 2014-09-10 15:43:49 +0800
comments: true
categories: MySQL 
---
##背景
搭建从库，我们使用 innobackupex(percona公司出品)。两次备份从库数据库，都出现了全局锁库的问题，现象就是备份数据的时候，整个从库, 都无法对外提供写服务了。这里分析了一下innobackupex的原理，搞清楚为啥会全局锁库：

####具体工作原理
1，记录`Log Sequence Number LSN`

2，拷贝每个表的ibd文件，统一的ibdata1文件，和redo log，就是ib_logfile0,ib_logfile1,ib_logfile2 这三个文件。

3，在步骤2，可能会花费较长的时间，同时后台跑一个进程，检测ib_logfile的变化，拷贝变化的数据。

4，数据都拷贝完毕，利用innodb 自身的 crash recovery，修复bid数据。现在innodb数据已经保存好

5，`flush table with read lock`：对所有的数据库和表加锁(此时读OK，写被block)

6，记录`show master status` 的位置

7，拷贝.frm .MYD, .MYI,.CSV等其他表信息，引擎文件

8，unlock数据。拷贝完毕
<!--more-->
####全局锁库原因
如果数据表中，含有大量的MyISAM文件，第7步，会耗费较多时间，导致全库不可写，业务被hang住。

问1：为什么第5步，要加锁？

答1：MyISAM不同于InnoDB，MyISAM没有redo log机制，只能温备(lock数据备份)或者冷备（停机），无法热备。所以拷贝数据的时候，自然不能有新数据写入。

问2：那仅仅给MyISAM的表加锁，为什么全库全表都要加锁？

答2：因为InnoDB和MyISAM在这个脚本是分开备份的。步骤1~4，都仅仅备份了InnoDB，第7步，备份MyISAM数据。二者如果不对InnoDB加锁，会导致InnoDB备份的数据位置和MyISAM的位置不一致 , master的地址无法记录。

##结论
线上的数据库，尽可能是InnoDB，不要使用MyISAM了，不然备份会很麻烦。
