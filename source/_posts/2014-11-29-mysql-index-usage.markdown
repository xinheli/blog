---
layout: post
title: "从一道面试题目谈谈MySQL查询优化"
date: 2014-11-29 14:27:57 +0800
comments: true
categories: MySQL
---

##背景
公司校园招聘的一道笔试题目，引起了大家的广泛讨论。这里结合题目深入说一下MySQL是怎么做查询优化的，到底是怎么使用索引(Index)的。

题目如下：
```
若在数据库中对user表中的两个INT字段a,b建立了复合索引INDEX(a,b)，则以下查询中能完全利用到这个索引的是_____

A. SELECT * FROM user WHERE a=0 AND b=0;

B. SELECT * FROM user WHERE a=0 OR b=0;

C. SELECT * FROM user WHERE a>0 AND b=0;

D. SELECT * FROM user WHERE a=0 AND b>0;
```
给出的答案是AD。我认为这题有一些漏洞，下面详细说来
<!--more-->

##分析
###表的结构：
```
CREATE TABLE `t4` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL DEFAULT '0',
  `b` int(11) NOT NULL DEFAULT '0',
  `c` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`),
  KEY `index1` (`a`,`b`)
) ENGINE=InnoDB;
```
###初步分析
A和D可以完全用到索引，不存在争论，“一般情况”下都是正确的(后面会解释什么叫“一般情况”)。B，也不存在争议，是用不到索引的。问题是C呢？

我们执行explain的结果如下：
```
mysql> explain select * from t4 where a =0 and b > 0;
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key    | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
|  1 | SIMPLE      | t4    | range | index1        | index1 | 8       | NULL |    1 | Using where |
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
mysql> explain select * from t4 where a > 0 and b = 0;
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key    | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
|  1 | SIMPLE      | t4    | range | index1        | index1 | 4       | NULL |    1 | Using where |
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
```
可以看到a=0&b>0 和 a>0&b=0 explain的结果“几乎“相同，都用到索引index1,都是range这个索引（即对索引区间扫描）得到的结果。唯一不同的是key_len。key_len说明了查找时用到的索引长度，可以根据长度，推测多维索引用到了几维。（MySQL索引都是前缀索引）

index1是二维索引KEY `index1` (`a`,`b`)，因此长度应该是4+4。 

a=0&b>0 key_len是8，说明仅仅用到了索引就能得到结果，先用a=0找到树节点，然后在其下面根据b>0过滤，得到结果。即”完全“用到索引就能得到结果。

a>0&b=0 key_len是4，说明仅仅用到了前缀索引的第一维，仅仅用a>0得到结果（主键），然后去主键索引（聚簇索引）里面读取整个行，根据b=0过滤相关数据，得到结果。即”不完全“用到索引才能得到结果。

这样看来，C选项a>0&b=0 ，应该是不正确的。但是有一些”特例“
###特例1：Index_Condition_Pushdown
[Index Condition Pushdown](http://www.cnblogs.com/zhoujinyi/archive/2013/04/16/3016223.html) (ICP)是MySQL用索引去表里取数据的一种优化，在MySQL 5.6才有的功能。

如果禁用ICP，引擎层会穿过索引在基表中寻找数据行，然后返回给MySQL Server层，再去为这些数据行进行WHERE后的条件的过滤。

ICP启用，如果部分WHERE条件能使用索引中的字段，MySQL Server 会把这部分下推到引擎层。存储引擎通过使用索引条目，然后推索引条件进行评估，使用这个索引把满足的行从表中读取出。

ICP能减少引擎层访问基表的次数和MySQL Server 访问存储引擎的次数。总之是 ICP的优化在引擎层就能够过滤掉大量的数据，这样无疑能够减少了对base table和mysql server的访问次数。

对于上面的例子：a>0&b=0。首先通过可以从index1里面得到所有a>0的结果。然后，如果没有启用ICP, 则存储引擎会把结果反给MySQL， MySQL在获取主键数据过滤，就如之前的例子分析。如果启动了ICP，因为index里面还是有b的信息，可以直接遍历索引a>0的子树下面是否有b=0的节点，就直接得到了数据，这样更快，减少一次查询和交互，是“完全”用到索引就能拿到数据。
```
mysql> explain select * from t4 where a > 0 and b = 0;
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key    | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
|  1 | SIMPLE      | t4    | range | index1        | index1 | 4       | NULL |    1 | Using index condition; Using where |
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
```
在特例1的情况下，C选项是正确的，答案是ACD
###特例2：数据分布
数据库最智慧，最难懂的地方就是“查询优化器”，一般来说，数据库会根据实际情况，使用最快的方法来查询数据。而最快的方法有时候不一定是用到索引，或者不一定“完全“用到索引。下面的例子。

t4里面插入2000条数据，都是a=0,b=100的数据，这个时候执行a=0&b>0 explain结果如下。
```
mysql>  explain select * from t4 where a =0 and b > 0;
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key    | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
|  1 | SIMPLE      | t4    | range | index1        | index1 | 4       | NULL |    1 | Using where |
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
```
似乎没什么区别，仔细看key_len是4，而不是8了。说明此处没有”完全“用到索引，这是为什么呢？这是数据分布导致。可以看到这个表所有的数据都符合查询条件。

如果采用之前的方法，key_len = 8，则
1，查询二维索引，找到符合条件的主键ID，因为所有数据都符合，等于遍历一次index。
2，根据第一步结果，读取相关行，因为所有数据都符合，等于遍历了一次数据。

如果采用key_len=4的做法。
1，查询二维索引，第一维度，只有一个节点，然后把整个节点的主键返回，不需要完全遍历index。
2，和上面一样，需要遍历一次数据。

后面的方法比之前的方法，其实还要快，少了一次全index的遍历。

不难分析出，如果查询条件变为a=0&b>100，则又会”完全“用到索引了。
```
mysql> explain select * from t2 where a =0 and b >100;
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key    | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
|  1 | SIMPLE      | t2    | range | index1        | index1 | 8       | NULL |    1 | Using where |
+----+-------------+-------+-------+---------------+--------+---------+------+------+-------------+
```
在特例2的情况下，D选项（a=0&b>0）是不对的。答案是A
###特例3：表结构
题目没有说表的结果，如果表结构仅仅有三个字段，主键, a, b. 即索引index1的实际结构和主键聚簇索引一样，则任何查询都仅仅会用到索引，而不会用到主键。
```
CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL DEFAULT '0',
  `b` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`),
  KEY `index1` (`a`,`b`)
) ENGINE=InnoDB;
```
```
mysql> explain select * from t1 where a =0 or  b=0;
+----+-------------+-------+-------+---------------+--------+---------+------+------+--------------------------+
| id | select_type | table | type  | possible_keys | key    | key_len | ref  | rows | Extra                    |
+----+-------------+-------+-------+---------------+--------+---------+------+------+--------------------------+
|  1 | SIMPLE      | t1    | index | index1        | index1 | 8       | NULL |    1 | Using where; Using index |
+----+-------------+-------+-------+---------------+--------+---------+------+------+--------------------------+
```
注意，最后一样的extra信息”Using index“，表示仅仅使用索引，就能拿到数据。最夸张的a=0 or b =0 都是对的。

在特例3的情况下，所有答案都是对的，答案是ABCD

###特例4：存储引擎
因为MySQL是插件式存储引擎，题目并没有说明具体数据库是什么，存储引擎是什么。因此如果存储引擎是某种”非主流“的引擎，完全没有实现索引的功能，或者用hash去实现索引，而不是Btree，这种情况下，可能没有正确答案，或者仅仅A是正确的。

在特例4的情况下，只有A是正确的，答案是A

当然特例4，有点算狡辩了，一般默认数据库肯定索引都是BTree了。

##总结
一道小小的笔试题目，引出了4个特例的情况。这道题给出的参考答案是AD，应该还是想考察绝大多数的情况。那如何修改题目，减少特例呢？

我的建议是，修改很困难，要说清楚比较麻烦。笔试题目还是不要出这里索引分析的题目，还是那句话，数据库查询优化的结果，影响因素很多，不同的数据库可能都不太一样。

