---
title: "极限环境Certutil加Powershell配合Burp快速落地文件"
date: 2020-09-28T14:54:25+08:00
draft: false
tags:
- Certutil
- Powershell
series:
-
categories:
- 渗透测试
---

碰到一些极限环境，比如站库分离只出dns的时候，想上线cs的马，但是文件迟迟不能落地，相信很多人都会想到certutil等工具。
<!--more-->

而在使用certutil base64通过echo写文件时，echo会在每行的末尾追加一个空格，加上http传输的URL编码问题，有一些傻逼环境总是decode时候出错，而且一些几十几百k的文件，一行一行echo实在是拉跨。所以用powershell配合bp的爆破模块来写文件，然后 `certutil -decode` 就完事了，轻松省心。

```powershell
powershell -c "'a' | Out-File C:\1.txt -Append"
```

写文件的时候通过bp的爆破模块去单线程写入文件，举一个请求包的例子。

```
/login HTTP/1.1
Host: baidu.com

cmd=powershell -c "'§§' | Out-File C:\1.txt -Append"
```

设置参数
![image.png](https://y4er.com/img/uploads/20200928158664.png)



设置certutil encode的txt字典
![image.png](https://y4er.com/img/uploads/20200928155864.png)

勾上URL编码
![image.png](https://y4er.com/img/uploads/20200928158567.png)


设置单线程，你也可以设置每次请求之后sleep 1秒。
![image.png](https://y4er.com/img/uploads/20200928152618.png)

冲完之后落地到目标的txt文件和本地的txt文件hash一致，decode之后的文件hash仍然一致。

本地还原文件的hash
![image.png](https://y4er.com/img/uploads/20200928152292.png)

落地到目标还原之后的文件hash
![image.png](https://y4er.com/img/uploads/20200928150771.png)



**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**