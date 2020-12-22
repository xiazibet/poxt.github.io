---
title: "Kerberos协议之基于资源的约束委派"
date: 2020-12-12T12:47:18+08:00
draft: false
tags:
- Kerberos
series:
- Windows协议
categories:
- 渗透测试
---

在之前的约束委派文章中提到，如果配置受约束的委派，必须拥有SeEnableDelegation特权，该特权是敏感的，通常仅授予域管理员。为了使用户/资源更加独立，Windows Server 2012中引入了基于资源的约束委派。基于资源的约束委派允许资源配置受信任的帐户委派给他们。

<!--more-->
# 理解资源约束委派

基于资源的约束委派和传统的约束委派非常相似，但是作用的方向实际上是相反的。引用[《Wagging the Dog: Abusing Resource-Based Constrained Delegation to Attack Active Directory》](https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html)文中的图解。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/847d037e-2396-e3b2-ab1d-7d4b0740048b.png)

传统的约束委派：在ServiceA的msDS-AllowedToActOnBehalfOfOtherIdentity属性中配置了对ServiceB的信任关系，定义了到ServiceB的传出委派信任。

资源约束委派：在ServiceB的msDS-AllowedToActOnBehalfOfOtherIdentity属性中配置了对ServiceA的信任关系，定义了从ServiceA的传入信任关系。

而重要的一点在于，资源本身可以为自己配置资源委派信任关系，资源本身决定可以信任谁，该信任谁。

在那篇文章中给出了一种利用环境。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/2a91e9ec-adff-9327-c443-e905cc546449.png)

1. 攻击者攻陷了一个有TrustedToAuthForDelegation标志位的账户(ServiceA)。
2. 攻击者拥有一个账户，该账户对ServiceB有权限，可以在ServiceB上配置资源约束委派。
3. 那么攻击者可以在ServiceB上配置从ServiceA到ServiceB的资源约束委派。
4. 攻击者使用S4U2Self模拟administrator向域控请求可转发的TGS
5. 然后将上一步的TGS通过S4U2Proxy模拟administrator向域控请求访问ServiceB的TGS。
6. 此时攻击者拿到了ServiceA->ServiceB的TGS。

# 实际利用 滥用MachineAccountQuota

1. 需要一个SPN账户，因为S4U2Self只适用于具有SPN的账户
2. 需要一个对目标机器有写入权限的账户GenericAll、GenericWrite、WriteProperty、WriteDacl等等都可以。

在域中有一个属性MachineAccountQuota，这个值表示的是允许域用户在域中创建的计算机帐户数，默认为10，这意味着我们如果拥有一个普通的域用户那么我们就可以利用这个用户最多可以创建十个新的计算机帐户，而计算机账户默认是注册RestrictedKrbHost/domain和HOST/domain这两个SPN的，所以第一个条件就满足了。

但当我们只是一个普通的域用户时，并没有权限（如GenericAll、GenericWrite、WriteDacl等）为服务修改msDS-AllowedToActOnBehalf OfOtherIdentity属性，不满足第二个条件。所以这里一般用的是中继，通过中继机器账户来修改该机器账户自身的msDS-AllowedToActOnBehalf属性。

ateam的[《这是一篇“不一样”的真实渗透测试案例分析文章》](https://blog.ateam.qianxin.com/post/zhe-shi-yi-pian-bu-yi-yang-de-zhen-shi-shen-tou-ce-shi-an-li-fen-xi-wen-zhang/)中对于基于资源的约束委派利用中，非常经典。

1. 首先通过webdav的xxe发送ntlm请求，实现中继webdav，因为是system权限，所以`webdav$`账户到手。
2. 然后通过密码喷洒拿到了一个普通域账号
3. 通过域账号添加了一个机器账户`evilpc$`
4. 通过xxe中继webdav到域控的ldap中，在webdav$机器上设置基于资源的约束委派（我改我自己）
5. 然后通过s4u发起从`evilpc$`到`webdav$`的资源约束委派请求，就有了webdav$的tgs。
6. 通过上一步申请的tgs ptt到`webdav$`

本地没环境，有机会再补上。

# 参考

感谢daiker师傅的一顿指点。这个东西写了很长一段时间，写完了还不是很明白，只是明白了ateam的那种利用场景，没参透有无别的场景，自己以后有空回过头来再看一下吧。

1. [这是一篇“不一样”的真实渗透测试案例分析文章](https://blog.ateam.qianxin.com/post/zhe-shi-yi-pian-bu-yi-yang-de-zhen-shi-shen-tou-ce-shi-an-li-fen-xi-wen-zhang/)
2. https://daiker.gitbook.io/windows-protocol/kerberos/2
3. https://cloud.tencent.com/developer/article/1552171
4. https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html
5. https://github.com/Kevin-Robertson/Powermad


**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**