---
title: 一次简单的SQL注入经历
date: 2019-06-08 12:00:02
tags:
- mysql注入
categories: 注入

---

一次简单的sql注入经历。

<!--more-->

资产收集发现个APP，这是一个内部员工<-->商家的一个APP，大概是主要处理单据的系统。

提一点收集资产的**小姿势**

site:pgyer.com XX

通过蒲公英找到该APP。

一般的数据库语句都是使用单引号闭合，因此奇数单引号的回显跟偶数单引号的回显是不一样的。

![](/images/easy-sql/1.png)

![](/images/easy-sql/2.png)

![](/images/easy-sql/3.png)

```
验证码已过期-----------------True    //默认不加语句的情况下回显，如果加了语句为真的话显示下面的不正确

OPT_TYPE不正确--------------True

请先获取短信验证码------------False

NULL------------------------False
```

这有个奇葩的点就不展开说了，简单来说就是该注入点如果带入验证码进入了查询，硬waf就会起作用，如果验证码过期，该硬waf就会失效。猜测该waf应该是只拦截数据查询。

OTP叫one time password，也就是一次性密码，即是手机验证码。那么可以推测出注入点存在于查询验证码的语句中，而不是在用户查询语句中。而且验证码查询是优先于用户查询，如果不存在验证码或者验证码失效就不会跳转到用户查询语句中，即不会触发waf。

然后通过爆破的方式识别数据库。TABLE_PRIVILEGE_MAP是Oracle的特征表，可以得出该数据库类型为Oracle。

![](/images/easy-sql/4.png)

接着用sqlmap应该就能跑了，之前没设置延时请求过快被ban，就不继续证明了

在burp上挂上sock代理，然后就能出结果了，因为有4种回显，必须加string标识才能可以跑。

```
python sqlmap.py -r 1.txt --dbms=Oracle --threads=1 --random-agent --dbs --proxy=http://127.0.0.1:8080 --string "不正确" --delay 1 --force-ssl
```

![](/images/easy-sql/5.png)