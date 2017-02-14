---
layout: post
title: "使用MySQL产生分表自增ID方法总结"
date: 2014-09-13 14:27:57 +0800
comments: true
categories: MySQL
---

##背景
-------------
公司线上数据增长很快，用户评论表（comment）已经增到到快10亿行，数据文件大小也快100G了。

MySQL表行数超过一个值，性能就会大幅度下滑，主要原因是B-Tree索引在数据多的时候，会多增加深度，导致查询耗时。以前的经验告诉我们数据表超过1亿行就要考虑分表了。但是这个数值并不是固定的，跟具体的每一行的数据长度有关（因为MySQL是按照page（size=16K) 存放在B-Tree上）。如果行的数据长度较小，没有什么varchar大字段，可能上亿都OK。但是comment表里面还实际存储了用户的留言信息，所以分表迫在眉睫。

分表所引入的一个问题就是就是各个分表自增ID的分配问题。
##具体方案
目前主流的方案有三种

1. 使用MySQL分配

2. 使用类MySQL方案分配（Redis/Memcached）

3. 写一个自增ID分配服务

方案2之前做其他项目的时候，有过类似经验。大体思路是使用memcached的Add 和 Increment方法来确保自增ID的加锁。具体可以参考[这里](http://abhinavsingh.com/blog/2009/12/how-to-use-locks-for-assuring-atomic-operation-in-memcached/)，但是从后面的性能上，并不是很强（1000 qps）

方案3需要自己造轮子，虽然不复杂，但是，ID分配服务必然是S级服务，要考虑持久化，单点恢复，高可用等情况，还增加了部署的复杂性等。

于是我们采用方案1，且方案1历史悠久，且生产环境使用经验丰富。
<!--more-->
使用MySQL分配ID，就是有一个专门的自增ID表，负责分配ID给分表使用。

#### 最初方案 方案一
``` mysql
自增ID表：
CREATE TABLE `ticket_mutex` (
  `id` bigint(20) unsigned NOT NULL auto_increment,
  `stub` char(1) NOT NULL default '',
  PRIMARY KEY  (`id`),
  UNIQUE KEY `stub` (`stub`)
) ENGINE＝InnoDB

获取ID REPLACE INTO ticket_mutex (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
```
这个方案，也是网上最多的方案之一，据说[Twitter](http://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)也在用，之前在美团也广泛使用，所以线上经验丰富。

#####优点：
采用了Unique Key的限制，确保这个表永远只有一行。
#####缺点：
1, 仅仅负责一套业务分表，如果线上有多套业务需要分表，需要多个这样的分配ID表（其实问题也不大）

2, 每次update都会锁住primary key和unique key，所以REPLCAE语句无法并发执行。高并发的情况下，会出现大量锁住的情况，导致性能问题。

最初，没有意识到缺点，之前的线上压力也没有暴露这个缺点。目前线上高峰的写入，能到600/s，甚至极端超过1000。大量的写入操作，导致上线没多久，MySQL就出现大量的lock wait(通过 `show engine innodb status` 查看)。之后查找资料，找到了另外一种可行方案。

####优化的方案 方案二
``` mysql
自增ID表：
CREATE TABLE ticket_mutex (
    name char(8) NOT NULL PRIMARY KEY,
    value bigint(20) UNSIGNED NOT NULL
)Engine=InnoDB DEFAULT CHARSET=UTF8;

初始化：
INSERT INTO ticket_mutex(name, value) values('USER', 0),('ITEM', 0);

获取ID
UPDATE ticket_mutex SET value=LAST_INSERT_ID(value+1) WHERE name='USER';
SELECT LAST_INSERT_ID();
```
[网上](http://www.cnblogs.com/obullxl/archive/2011/06/24/mysql-last-insert-id.html)也有人在生产环境用过。LAST_INSERT_ID仅仅和connection相关，所以不同的connection，取得的LastID不会冲突。具体LAST_INSERT_ID参考[这里](http://dev.mysql.com/doc/refman/5.5/en/information-functions.html#function_last-insert-id)

#####优点：
 1, 采用了Primary Key的限制，确保这个表永远只有一行。
 
 2, 一个业务逻辑一行，多个业务逻辑可以共用1个表来分配自增ID。

#####缺点：
 1,  每次update还是会锁住，不过第一个方案，会锁住primary key和unique key，而本方案仅仅锁住primary key，性能肯定会优于方案一。

上线之后，果然没有lock了，不过线上运行了几天，发现有不少慢查询。大并发下，性能还是有问题。无奈只有采用性能最好，但是缺点也很突出的方案三。
#### 最终方案 方案三
``` mysql
自增ID表：
CREATE TABLE `ticket_mutex` (
  `id` bigint(20) unsigned NOT NULL auto_increment,
  PRIMARY KEY  (`id`)
) ENGINE＝InnoDB

获取ID INSERT INTO ticket_mutex () values ();
SELECT LAST_INSERT_ID();
```
这个方案，最简单粗暴，效率最高。

#####优点：
性能最高，几乎没有锁。如果非要说有锁，就是获取自增ID的时候，MySQL会产生一个局部锁，但是这个锁的性能损耗远远小于对index 的锁
#####缺点：
每获取一次，都会产生一行，长时间跑，导致该表越来越大。

#####后续问题：
上线该方案之后，再也没有任何性能问题。遗留问题，这个自增表也会越来越大，如果瘦身?  

定期删除看似可行，其实不可以。因为MySQL本身删除，由于[purge](/blog/2014/09/13/innodb-max-purge-lag/)的问题，每次删除控制在500qps左右，而之前说了，该表每秒写入能到600多。写的速度比删都快。

最简单的方案，就是定期，切表。比如三个月切换一次。旧表可以直接快速Drop掉

##总结
由于应用逻辑并发太高，最后采用了最不优雅，但是性能最好的方案。对于普通的使用(写操作不会超过500 qps)，方案二，干净简洁，值得考虑。方案一，现在看来，完全没有存在的必要了。

