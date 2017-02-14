---
layout: post
title: "mysql查询类型转换诡异问题"
date: 2014-07-10 15:53:57 +0800
comments: true
categories: MySQL GDB 
---

##背景
MYSQL查询数据查不到的情况，为了简化，把核心内容抽象出来，如下：

tchar表结构，只有两个字段 id(int), t(char(20)) ,相关数据如下：
``` mysql
| tchar | CREATE TABLE `tchar` (\\
 `id` int(11) NOT NULL,\\
 `t` char(20) NOT NULL\\
 ) ENGINE=CSV DEFAULT CHARSET=gbk |
```
通过select * 可以看到有如下的数据：
``` mysql
mysql> select * from tchar;
+\--\-+\-------------------\+
| id | t |
+\--\-+\-------------------\+
| 1 | 201012141549372150 |
| 2 | 201012151434141129 |
+\--\-+\-------------------\+
```
##现象与初步分析
###现象一

使用`select * from tchar where t=201012141549372150` 查询不到数据。（mysql记录中有该数据）

###分析一

因为t字段在数据库中是char类型，所以这样查询不对，应该写成`select * from tchar where t=“201012141549372150”`。果然数据出来了。这样看来是一个小问题，加上引号就OK了。如果只分析到这一步，就不会有这篇文章了....
<!--more-->
###现象二

使用`select * from tchar where t=201012151434141129` 却可以查询到数据。（mysql记录中有该数据）

###分析二

这个就有点奇怪了，有其他同学分析是不是因为这个插入数据库的程序做过更新，第一版本是`insert t=123456`，第二个版本是`insert t="123456"`，mysql存的东西不一样，所以会出现这样的诡异问题？ 思路很好，不过后面后证明，是不对的。

###现象三

使用`select * from tchar where t=201012151434141130` 竟然也可以查询到数据。（mysql记录没有这个数据）

###分析三

实际群里仅仅讨论了上面两个现象，第三个现象是自己线下测试发现的。第一感觉，可能是Mysql查询的时候，做了类型转换，这里面碰到了一些坑，要好好分析一下。

###现象四

使用`select * from tchar where t="xxxxx"` 查询数据是完成符合预期的

###分析四

说明很有可能是不带引号查询，出现的类型转换的错误。
##问题深入追查
首先排除“分析二”的说法不正确。这里使用了CSV存储引擎，数据直接打开看，通过insert不带引号，和带引号的两种方法，可以看到在数据引擎里面都会存为带引号的数据。

实际上在insert的时候，mysql会根据column的type，对传入的数据做类型转换的工作，不带引号insert，实际上mysql也会把它转为char类型，然后更新到存储引擎中。

看到查不到的问题，第一反应，是不是溢出了？会不会是因为这个数字太长了，查询的时候，作为int型，溢出了？？答案是否定的，首首首先t=201012141549372150 并没有超过int的长度
``` mysql
mysql> select cast(201012141549372150 as signed);
+\-----------------------------------\-+
| cast(201012141549372150 as signed) |
+\-----------------------------------\-+
| 201012141549372150 |
+\-----------------------------------\-+
```
另外可以看到 t= 201012151434141129 是可以出来数据的，而201012141549372150 查不到数据，能查到的数据比查不到的还要“大”，要是溢出，应该两个都查不到才对。

后续还补充做了一些其他查询的实验，发现当数据比较大的时候，会出现上面的查不到，或者差错的情况，如果t传入的数据比较小，是可以很准确查到数据的。但是并不存在一个绝对的门限，比如大于X，查询肯定有问题，小于X，查询肯定没有问题。

这些实验，让我非常困惑，看来只能从根本解决了,祭出神器GDB
MYSQL代码很庞杂，这里仅仅列些了有关和查询相关的函数。

这里首先说一下，MYSQL的一个大体查询过程。特别详细描述了`JOIN::prepare`和`JOIN::exec()` ，对于优化部分，不是本文重点。
``` c
handle_select()
mysql_select()          //查询主函数
JOIN::prepare()         //
setup_fields()
setup_without_group()
setup_conds
Item_func::fix_fields
Item_func::fix_length_and_dec
Item_bool_func2::fix_length_and_dec
item_cmp_type
setup_order
setup_group
JOIN::optimize()            /\* optimizer is from here ... \*/
optimize_cond()
opt_sum_query()
make_join_statistics()
get_quick_record_count()
choose_plan()
optimize_straight_join()
best_access_path()
greedy_search()
best_extension_by_limited_search()
best_access_path()
find_best()
make_join_select()        /\* ... to here \*/
JOIN::exec()
```
MYSQL查询里面，用ITEM类来存放where条件数据。ITEM类是MYSQL中最重要的一个类，里面的方法和字段很多，和本文有关系的只有2个

`name: char`类型，存放查询的value数据，如上面的例子中的201012141549372150。

`cmp_context`：存放最终where比较的类型,刚开始默认是 4294967295 0XFFFFFFFF

`result_type()`:存放初始的类型。

在`mysql_select` 这一步，系统会把初步解析的where条件，生成对应的ITEM对象`conds`：

如果执行查询是`select * from tchar where t=201012141549372150`，那么对应conds的内容如下：
``` c
(gdb) p conds->next->result_type()
$16 = INT_RESULT
(gdb) p conds->next->name
$17 = 0x2a9b704fb0 "201012141549372150"
(gdb) p conds->next->cmp_context
$18 = 4294967295
```
如果执行查询是`select * from tchar where t=“201012141549372150”`，那么对应conds的内容如下：
``` c
(gdb) p conds->next->result_type()
$19 = STRING_RESULT
(gdb) p conds->next->name
$20 = 0x2a9b705048 "201012141549372150"
(gdb) p conds->next->cmp_context
$21 = 4294967295
```
对比可以看到，带引号的查询条件是STRING类型，不带引号，mysql认为是INT类型。这个和我们之前的想法是一直的。

继续GDB跟踪，发现经过`JOIN::prepare()`之后，conds数据发生了变化。

对应引号查询的 `conds->cmp_context = STRING_RESULT`

而对应不带引号的查询，`conds->cmp_context = REAL_RESULT`

真相大白。对于不带引号的查询，MYSQL虽然知道是INT类型，但是实际和数据库比较的时候，是按照了REAL类型来比较的，REAL类型是一直近似的浮点表示，最多只能表示15~16位有效数字，通过后续观察，在do_select查询函数的时候，where条件 201012141549372150 变成了2.010121415493722*e17，这样自然就会出现了查询差不 到，或者查询出错的情况。之前的现象1,2,3,4都可以完美的解释了。

##再进一步

那为什么传入的INT类型，MYSQL却认为是REAL类型呢？这个和MYSQL的官网建议不符合啊。MYSQL官方建议：“在 WHERE 子句搜索条件中（特别是 = 和 <> 运算符）,应避免使用float或real列。最好限制使用float和real列做> 或 < 的比较。” 用户使用了INT类型，竟然还给转成了REAL类型。

通过进一步查看MYSQL代码，可以得到MYSQL类型转换的准则。具体代码如下：其中a是数据库表信息中该字段的类型，b是实际传入的类型。对应上面的例子,a 是t字段实际的类型，STRING_RESULT，对应不带引号的查询，b是INT_RESULT,对于带引号的查询，b是STRING_RESULT。
``` c
Item_result item_cmp_type(Item_result a,Item_result b)
{
if (a == STRING_RESULT && b == STRING_RESULT)
return STRING_RESULT;
if (a == INT_RESULT && b == INT_RESULT)
return INT_RESULT;
else if (a == ROW_RESULT \|\| b == ROW_RESULT)
return ROW_RESULT;
if ((a == INT_RESULT \|\| a == DECIMAL_RESULT) &&
(b == INT_RESULT \|\| b == DECIMAL_RESULT))
return DECIMAL_RESULT;
return REAL_RESULT;
}
```
原来，MYSQL类型转换的时候，转换条件比较严格，如果不符合，默认都是REAL_RESULT。不带引号的时候，a=STRING_RESULT,b=INT_RESULT,返回的就是REAL_RESULT。
后续通过修改item_cmp_type的代码，加上了一条判断
``` c
Item_result item_cmp_type(Item_result a,Item_result b)
{
if (a == STRING_RESULT && b == INT_RESULT)
return STRING_RESULT;
if (a == STRING_RESULT && b == STRING_RESULT)
return STRING_RESULT;
if (a == INT_RESULT && b == INT_RESULT)
return INT_RESULT;
else if (a == ROW_RESULT \|\| b == ROW_RESULT)
return ROW_RESULT;
if ((a == INT_RESULT \|\| a == DECIMAL_RESULT) &&
(b == INT_RESULT \|\| b == DECIMAL_RESULT))
return DECIMAL_RESULT;
return REAL_RESULT;
}
```
到此真相大白。个人认为这个是MYSQL设计欠妥当的地方。默认REAL_RESULT会出现很多令人费解的情况，默认是STRING可能更好，最稳妥的方式是如果不符合条件默认报错，这样从根本规避这个问题。

##一些总结
1，mysql查询的时候，还是格外注意一下column的类型和where的写法，特别是char类型的时候，很容易忽略掉引号。如果设计数据库的时候，可以预见未来的char数据都是int类型，则直接用int，比如上面的例子，完全可以用int字段，就不会有后面这些故事。

2，如果输入的类型和自身类型不符合，用mysql自带的转义，缺点很多，首先是上面说的隐隐患，其次，mysql底层查询的时候，会把数据库中所有的数据进行类型转换，上面的例子，char-->double的转换，涉及到字符串操作和循环，比较耗费CPU的操作，如果查询数量巨大，会极大影响速度。

3，实际GDB mysql的时候，要注意使用`select sql_no_cache * from xxx where xxx`，加上`sql_no_cache`是让sql不去查询cache,确保会执行`mysql_select`函数，方便调试。

4，小问题不容忽视，往下追查往往会有大收获。

