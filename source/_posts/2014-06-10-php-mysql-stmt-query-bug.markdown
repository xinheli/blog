---
layout: post
title: "DB execute诡异的类型转换问题"
date: 2014-06-10 16:06:45 +0800
comments: true
categories: PHP MySQL GDB 
---
##背景介绍

业务逻辑中，发现php对象连接MySQL，查询结果很诡异，和直接去数据库查询结果不一样。为了简化逻辑，这里把问题简单转换一下。

mysql 表信息如下
``` mysql
mysql> show create table tchar;
| tchar | CREATE TABLE `tchar` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `t` char(20) NOT NULL,
  `i` bigint(20) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=latin1 |
```
tchar这个表，里面的数据如下
``` mysql
mysql> select * from tchar;
+----+--------------------+--------------------+
| id | t                  | i                  |
+----+--------------------+--------------------+
|  3 | 13372241626481427  |  13372241626481427 |
|  4 | 13372241626481428  |  13372241626481428 |
+----+--------------------+--------------------+
```
php 代码如下
``` php
$DB_URL = "mysql://username:123@192.168.0.147:3306/lee#UTF8";
$SQL = "select id, t, i from tchar where i=?";
$db = DB::getInstance($DB_URL);
$i = "13372241626481427";
$arr = DB::getMultiRowsFromStmt($db->execute($SQL, array("i", $i)), array('id', 't', 'i'));
var_dump($arr);
```
故障描述
``` php
php 输出结果
array(1) {
  [0]=>
  array(3) {
    ["id"]=>
    int(4)
    ["t"]=>
    string(17) "13372241626481428"
    ["i"]=>
    int(13372241626481428)
  }
}
```
结果很诡异，明明查询的是 `i = 13372241626481427`，怎么出来的结果是 `i= 13372241626481428`； 而且正好错误的结果比正确的大1？
<!--more-->
##故障分析

###初步分析

先说错误最根本原因，$i变量是string类型，而execute执行的声明`bind_param`指定的却是int类型。导致出现某些诡异问题。

修改方法也很简答
``` php
//修改代码如下 对$i进行一次intval转换
$arr = DB::getMultiRowsFromStmt($db->execute($SQL, array("i", intval($i))), array('id', 't', 'i'));
//其实这样修改也没有问题
$arr = DB::getMultiRowsFromStmt($db->execute($SQL, array("s", $i)), array('id', 't', 'i'));
```
到此，基本就算知道原因了，使用框架的DB，特别要注意类型的问题，尤其是php是一种弱类型的语言，很容易犯这个错误。

###深入分析

不过，还没有找到本质原因。有几个疑问还没有解答。

1，当$i是string,而`bind`声明是int的时候，难道php不会进行`string->int`的转换吗？转换了可能就没问题了

2，直接在mysql client上执行如下语句，都是OK，为什么php执行就有问题？ 这个建议参考[mysql类型转换问题](/blog/2014/09/13/mysql-item-compare-bug/)
``` mysql
mysql> select * from tchar where i = 13372241626481427;
+----+-------------------+-------------------+
| id | t                 | i                 |
+----+-------------------+-------------------+
|  3 | 13372241626481427 | 13372241626481427 |
+----+-------------------+-------------------+
1 row in set (0.00 sec)

mysql> select * from tchar where i = "13372241626481427";
+----+-------------------+-------------------+
| id | t                 | i                 |
+----+-------------------+-------------------+
|  3 | 13372241626481427 | 13372241626481427 |
+----+-------------------+-------------------+
1 row in set (0.00 sec)
```
3，为什么恰好查询的错误结果是+1？怎么解释？
首先，看一下是哪里处理问题，是mysql还是php出了问题？ 联系sa,打开了db的query log,发现上述php执行后，query log 内容如下
``` mysql
243105 Prepare  select id, t, i from tchar where i=?
243105 Execute  select id, t, i from tchar where i=13372241626481428
```
看来mysql收到的查询请求，已经是错误的`i= 13372241626481428` ，而不是`i = 13372241626481427`。错误原因是php端。

看一下php DB的`execute`方法干了什么
``` php
public function &execute($sql, $param = null, $useCache = true, $renew = false)
{
    //$this->checkSpecialSQL($sql);
    $type = '';
    $stmt = $this->getStmt($sql, $type, $useCache, $renew);
    $stmt_param = array();
    if (is_array($param) && sizeof($param)) {
        // fix warning, see http://bugs.php.net/bug.php?id=43568
        foreach ($param as $k => &$v) {
            $stmt_param[$k] = &$v;
        }
        call_user_func_array(array($stmt, 'bind_param'), $stmt_param);
    }
    //......后面的不重要
}
```
还是没啥，看不出问题所在。只能祭出GDB神器了。
从源代码找出问题。GDB找起来，比较浪费时间，我也饶了一些弯路，这里直接写结论：

1，跟mysql相关的代码，都在`ext/mysqli`中，主要是`mysql_api.c`,`mysqlnd.c`, `mysqlnd_ps_codes.c` 这三个文件。

2，调用`bind_param`的时候，会把`bind_param`信息存在`stmt->param_bind` 这个数据结构。具体代码在 `mysqli_api.c` 的 `mysqli_stmt_bind_param_do_bind`函数 `param_bind`是一个数组，每一项是一个这样的结构体
``` c
struct param
{
    int  type， //声明的类型，就是执行execute的参数"i"的类型，是LONG形
    zval* zv   //存的数值，就是$i="13372241626481427"这个字符串
}
```
gdb 可以看到具体这个数的信息
``` c
1447    in /home/xxxxx/php-5.3.13/ext/mysqlnd/mysqlnd_ps.c
(gdb) p *(MYSQLND_STMT_DATA *) stmt
$10 = {conn = 0xb657d48, stmt_id = 1, flags = 0, state = MYSQLND_STMT_PREPARED, warning_count = 0, result = 0xb657b68, field_count = 3, param_count = 1, send_types_to_server = 0 '\0', param_bind = 0xb4f2258, result_bind = 0x0,
  result_zvals_separated_once = 0 '\0', persistent = 1 '\001', upsert_status = {warning_count = 0, server_status = 2, affected_rows = 18446744073709551615, last_insert_id = 0}, error_info = {error = '\0' <repeats 512 times>,
    sqlstate = "00000", error_no = 0}, update_max_length = 0 '\0', prefetch_rows = 1, cursor_exists = 0 '\0', default_rset_handler = 0, execute_cmd_buffer = {buffer = 0xb4f56d8 "p?W\v", length = 4096}, execute_count = 0}
(gdb) p stmt->param_bind[0]
$11 = {zv = 0xb653fe0, type = 8 '\b', flags = 0}
(gdb) p *(zval *)stmt->param_bind[0].zv
$13 = {value = {lval = 188509408, dval = 9.3136022410671048e-316, str = {val = 0xb3c6ce0 "13372241626481427", len = 17}, ht = 0xb3c6ce0, obj = {handle = 188509408, handlers = 0x11}}, refcount__gc = 5, type = 6 '\006',
  is_ref__gc = 1 '\001'}
```
可以看到zval里面存的是 string类型的数据，还没有类型转换。ZVAL 是php 里面表示数据的基础结构，是一个union,
``` c
typedef union _zvalue_value {
    long lval;                  /* long value */
    double dval;                /* double value */
    struct {
        char *val;
        int len;
    } str;
    HashTable *ht;              /* hash table value */
    zend_object_value obj;
} zvalue_value;
```
3，之后php 执行了`mysqlnd_stmt_execute_store_params` 这个函数，而类型转换就发生在了这一步。
``` c
/*  mysqlnd_stmt_execute_store_params */
static enum_func_status
mysqlnd_stmt_execute_store_params(MYSQLND_STMT * s, zend_uchar **buf, zend_uchar **p, size_t *buf_len  TSRMLS_DC)
{
   //....前面的不重要
   /* 1. Store type information */
    /*
      check if need to send the types even if stmt->send_types_to_server is 0. This is because
      if we send "i" (42) then the type will be int and the server will expect int. However, if next
      time we try to send > LONG_MAX, the conversion to string will send a string and the server
      won't expect it and interpret the value as 0. Thus we need to resend the types, if any such values
      occur, and force resend for the next execution.
    */
    for (i = 0; i < stmt->param_count; i++) {
        if (Z_TYPE_P(stmt->param_bind[i].zv) != IS_NULL &&
            (stmt->param_bind[i].type == MYSQL_TYPE_LONG || stmt->param_bind[i].type == MYSQL_TYPE_LONGLONG))
        {
            /* always copy the var, because we do many conversions */
            if (Z_TYPE_P(stmt->param_bind[i].zv) != IS_LONG &&
                PASS != mysqlnd_stmt_copy_it(&copies, stmt->param_bind[i].zv, stmt->param_count, i TSRMLS_CC))
            {
                SET_OOM_ERROR(stmt->error_info);
                goto end;
            }
            /*
              if it doesn't fit in a long send it as a string.
              Check bug #52891 : Wrong data inserted with mysqli/mysqlnd when using bind_param, value > LONG_MAX
            */
            if (Z_TYPE_P(stmt->param_bind[i].zv) != IS_LONG) {
                zval *tmp_data = (copies && copies[i])? copies[i]: stmt->param_bind[i].zv;
                convert_to_double_ex(&tmp_data);  //这里发生的转换
                if (Z_DVAL_P(tmp_data) > LONG_MAX || Z_DVAL_P(tmp_data) < LONG_MIN) {
                    stmt->send_types_to_server = resend_types_next_time = 1;
                }
            }
        }
    }
    //后面的也不重要
}
```
这一步 php 做了类型的转换，理论上，应该是把string转为int，但是代码里面却是 `convert_to_double_ex` 把string转为浮点数。

而浮点是不精确的表示，特别是当string特别长的时候，下面展示了`convert_to_double_ex`的输入和输出。可以看到输入 “13372241626481427”， 却输出了13372241626481428
``` c
2062    in /home/xxxxxx/php-5.3.13/Zend/zend_strtod.c
(gdb) finish
Run till exit from #0  zend_strtod (s00=0x1710f810 "13372241626481427", se=0x0) at /home/xxxxxx/php-5.3.13/Zend/zend_strtod.c:2062
convert_to_double (op=0x1739d1f0) at /home/xxxxx/php-5.3.13/Zend/zend_operators.c:420
420     /home/xxxxx/php-5.3.13/Zend/zend_operators.c: No such file or directory.
        in /home/xxxxx/php-5.3.13/Zend/zend_operators.c
Value returned is $30 = 13372241626481428
```
下面的代码也解释了php string->float会造成“误差”
``` php
$s = "13372241626481427";
$l = (float)$s;
var_dump($l);
//结果  float(1.3372241626481E+16)
```
到底，疑惑全部有了答案。
Q1:当$i是string,而`bind`声明是int的时候，难道php不会进行string->int的转换吗？转换了可能就没问题了

A1:php是转换了，不过用的是string->double

Q2:直接在mysql client上执行如下语句，都是OK，为什么php执行就有问题？

A2:问题发生在php端

Q3:为什么恰好查询的错误结果是+1？怎么解释？

A3:浮点数不精确，往往都是最后一位造成的，正好差了+1

##收获

1，执行类似的操作，最后加上`intval`的判断，如果id比较小，不加上没有问题的原因是，那时候还没有丢失精度的问题，大id就有问题

2，问题追根刨地，总能有意想不到的收获。源代码永远是我们最好的老师.
