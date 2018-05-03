---
layout: post
title:  "Laravel 创建自己的 Facade"
date:   2017-06-04 22:06:59
categories: WEB框架
---

## 前言
laravel 提供了一个灵活的模式，那就是 facade 。框架内部的 DB、Auth、File 等功能也有相关的 facade 实现。那么，该如何写自己的 facade 呢？

## Facade 是什么？
首先，facade 并不是 laravel 独有的东西，它就是设计模式中的外观模式(Facade)。
当然，这里就不长篇大论去讨论外观模式的定义了。这篇文章写的很不错 ： [设计模式（九）外观模式Facade（结构型）](http://blog.csdn.net/hguisu/article/details/7533759 "设计模式（九）外观模式Facade（结构型）")。
那么，laravel 的 facade 做了什么？
同样的， laravel 实现了外观模式的开关功能，并且使用魔术方法 __callstatic 实现了静态方式调用、动态创建对象的功能。参考 ([官方文档](http://d.laravel-china.org/docs/5.1/facades "官方文档"))

当然你可能觉得这些概念很抽象，都什么玩意。那么其实简单的讲，laravel 的 facade 就是将某些功能封装成工具类，而且能以静态方式调用工具类的方法。

## 建立自己的 facade

首先、以 laravel 5.1 框架，我之前写过的 Geoip facade 为例，说一下怎么去建立自己的 facade。

### 下载 geoip 扩展
geoip 是一个可以更具 IP 获取国家、地域、城市信息的 PHP 扩展，基于 maxmind 数据库。 [github 在此](https://github.com/maxmind/GeoIP2-php "github 在此")。

首先，为 laravel 添加 geoip 扩展。
打开 composer.json，添加 "geoip2/geoip2": "~2.0" 到 require。
项目根目录运行 composer update ( 需要安装 composer )更新一下，geoip 的依赖和软件包就被下载到 vendor 文件夹中了。 

然后下载 geoip 依赖的数据库，免费库的地址 ： [GeoLite2](http://dev.maxmind.com/geoip/geoip2/geolite2/ "GeoLite2") 

我下载了 GeoLite2 Country 和 GeoLite2 City 库，放到了 storage/geoipdb 中。

### 建立 facade。

在 app 目录下新建 Facades 文件夹，里面新建 Facades/GeoIP/GeoIP.php 和 Facades/GeoIP/Facade/GeoIP.php (建议每个功能新建一个文件夹区分，比如我这里给 GeoIP 新建一个文件夹，关于GeoIP 的东西全放到这里)
注意，Facades/GeoIP 下的 GeoIP.php 是你要对 geoip 扩展进行封装的类， Facades/GeoIP/Facade 下的 GeoIP.php 是你的 facade，用来给 laravel 解析使用，这两个文件可以不同名。

目录结构如图:

![](/assets/images/2017-06-04-laravel_create_facade-1.png)

**Facades/GeoIP/Facade/GeoIP.php 如下**
```php
<?php

namespace App\Facades\GeoIP\Facade;

use Illuminate\Support\Facades\Facade;

class GeoIP extends Facade
{
    protected static function getFacadeAccessor()
    {
        return 'geoip';
    }
}

```

注意你的 facade 现在只有一个方法，返回了一个字符串 'geoip' , 这个字符串是一个标号，用来给 laravel 的服务提供者解析使用的。

**Facades/GeoIP/GeoIP.php 如下（吐槽：写的有点随意）**
```php
<?php

namespace App\Facades\GeoIP;

use GeoIp2\Database\Reader;

class GeoIP
{
    /**
     * GeoIP country db path (base on storage_path).
     *
     * @var GeoIP
     */
    private $_country_db = 'geoipdb/GeoLite2-Country.mmdb';

    /**
     * GeoIP city db path (base on storage_path).
     *
     * @var GeoIP
     */
    private $_city_db = 'geoipdb/GeoLite2-City.mmdb';

    /**
     * Instance for GeoIP .
     *
     * @var GeoIP
     */
    private $_instance;

    /**
     * Init instance.
     *
     */
    public function init($mode)
    {
        switch ($mode) {
          case 'getCountry':
            $path = $this->_country_db;
            break;
          case 'getCity':
            $path = $this->_city_db;
            break;
          default:
            break;
        }

        $this->_instance = new Reader(storage_path($path));
    }

    /**
     * Get Country infomations.
     *
     * @param  String  $ip
     * @return Array
     */
    public function getCountry($ip)
    {
      $this->init(__FUNCTION__);

      $record = $this->_instance->country($ip);

      // 国家信息
      $data['iso_code'] = $record->country->isoCode;
      $data['country_name'] = $record->country->name;
      $data['country_name_zh_cn'] = $record->country->names['zh-CN'];

      return $data;
    }
 
 /**
     * Get City infomations.
     *
     * @param  String  $ip
     * @return Array
     */
    public function getCity($ip)
    {
      $this->init(__FUNCTION__);

      $record = $this->_instance->city($ip);

      $data['iso_code'] = $record->country->isoCode;
      $data['country_name'] = $record->country->name;
      $data['country_name_zh_cn'] = $record->country->names['zh-CN'];

      // 省、州信息
      $data['sub_division_name'] = $record->mostSpecificSubdivision->name;
      $data['sub_division_name_zh_cn'] = $record->mostSpecificSubdivision->names['zh-CN'];
      $data['sub_division_code'] = $record->mostSpecificSubdivision->isoCode;

      // 城市信息
      $data['city_name'] = $record->city->name;
      $data['postal_code'] = $record->postal->code;

      // 经纬度
      $data['latitude'] = $record->location->latitude;
      $data['longitude'] = $record->location->longitude;

      return $data;
    }

}

```

OK，现在 geoip 的常用功能已经封装到方法中了。

## 注册服务
完成了 facade 的创建和功能封装，下面就要使用它了。自己创建的 facade 要在 laravel 使用是要进行注册的，以便 laraval 在启动时能自动注入依赖(请看 laravel 的依赖注入简介 : [laravel 依赖注入 学院君](http://laravelacademy.org/post/769.html "laravel 依赖注入 学院君"))

### 编写服务提供者
在 app/Providers 下新建 FacadesServiceProvider.php
可以手动建，也可以用 artisan 命令来生成，随你喜欢。
app/Providers/FacadesServiceProvider.php 代码如下:

```php
<?php

namespace App\Providers;

use App\Service\ApiService;
use Illuminate\Support\ServiceProvider;

// include the class facade binded
use App\Facades\GeoIP\GeoIP;

class FacadesServiceProvider extends ServiceProvider
{
    /**
     * 在容器中注册绑定。
     *
     * @return void
     */
    public function register()
    {
        $this->app->singleton('geoip', function ($app) {
            return new GeoIP($app);
        });
    }
}

```

上面代码可知，服务提供者注册时会注册一个单例，标号为 'geoip'，也就是我们自己的 facade 返回的那个，然后回调函数会返回一个对象，也就是我们封装 geoip 功能的那个类的实例，不明白的同学可以看看 laravel 的服务提供者和服务容器相关知识哦。(注意要 use 将 facade 和封装类的命名空间引用一下哦)


### 注册服务提供者

laravel 5.1 以上版本的话， config/app.php 中找到 providers 和 aliases ，将你的服务提供者和 facade 别名配置一下 : 

providers 加入 ： 
```php
App\Providers\FacadeServiceProvider::class,
```
aliases 加入(不用每次都写很长的命名空间前缀) :
```php
'GeoIP'      => App\Facades\GeoIP\Facade\GeoIP::class,
```
对于 lumen 5.2 以上，需要在 bootstrap/app.php 中添加


```php
$app->register(App\Providers\FacadesServiceProvider::class);
```

注册完毕后，每次使用 facade::function 的时候，laravel 会自动解析 facade， 然后创建一个对象给用户使用，，而无需用户自己去 new 一个对象出来。

## 使用

现在，在任何一个控制器，或者路由的回调函数中，使用
```php
$res = GeoIP::getCountry('75.101.195.215');
var_dump($res);
```
你会发现，facade 已经可以好好工作了，enjoy！

## 参考文章
【1】[设计模式（九）外观模式Facade（结构型）](http://blog.csdn.net/hguisu/article/details/7533759 "设计模式（九）外观模式Facade（结构型）")

【2】[Laravel 服务容器实例教程 —— 深入理解控制反转（IoC）和依赖注入（DI）](http://laravelacademy.org/post/769.html "laravel 依赖注入 学院君")

【3】[Laravel 服务提供者实例教程 —— 创建 Service Provider 测试实例](http://laravelacademy.org/post/796.html "Laravel 服务提供者实例教程 —— 创建 Service Provider 测试实例")
