---
layout: post
title:  "写一个“特殊”的查询构造器 - (一)"
date:   2018-05-07 17:22:01
categories: PHP
excerpt: 本篇开始，正式进入编码实践中。首先，简单的规划下程序的结构。
 如上一篇所说的，我们需要一个基类 PDODriver 用来封装 PDO 的一些公用的方法，Mysql 等每个数据库都新建一个类 (这里我们简称为驱动类)，均继承自基类。
---

## 程序的结构

本篇开始，正式进入编码实践中。首先，简单的规划下程序的结构。

如上一篇所说的，我们需要一个基类 PDODriver 用来封装 PDO 的一些公用的方法，Mysql 等每个数据库都新建一个类 (这里我们简称为驱动类)，均继承自基类。

为了开发的扩展性和规范化，我们还需要一个接口 ConnectorInterface，接口中声明了必须要实现的方法，每个驱动类和基类都实现了该接口。

需要建立的源码文件如下：
```
项目根目录/
    ConnectorInterface.php  -- 接口文件
    PDODriver.php -- 基类
    Mysql.php -- Mysql 驱动类
    Pgsql.php -- PostgreSql 驱动类
    Sqlite.php -- Sqlite 驱动类
```


## 基类的创建，PDO 连接

首先要开始的就是基类，对 PDO 的连接进行封装的工作。

- 新建 PDO 连接，初始化参数 (dsn、username、password、options) 应该交给构造函数。
- PDO 连接不对外暴露，隐藏 pdo 的内部使用。 
- 为了方便灵活控制，独立连接和断开连接的方法

一个基础的基类 PDODriver.php 代码如下：

```php
// 声明命名空间，方便代码管理
namespace Drivers;
// 使用绝对命名空间的 PDO 类 
use PDO;
use PDOException;
// 声明基类
class PDODriver implements ConnectorInterface
{
    // 用来保存 PDO 连接，内部和派生类使用，对外隐藏
    protected $_pdo = NULL;
    // 保存初始化参数
    protected $_config = [];
    // PDO 的 options 信息，这里配置一些基础设置
    protected $_options = [
        PDO::ATTR_CASE => PDO::CASE_NATURAL,           // 保留数据库驱动返回的列名
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,   // 如果发生错误，则抛出一个 PDOException 异常
        PDO::ATTR_ORACLE_NULLS => PDO::NULL_NATURAL,   // 获取数据时将空字符串转换成 SQL 中的 NULL
        PDO::ATTR_STRINGIFY_FETCHES => FALSE,          // 禁止将数值转换为字符串
        PDO::ATTR_EMULATE_PREPARES => FALSE,           // 禁用预处理语句的模拟
    ];
    // 构造函数，保存初始化参数，创建连接
    public function __construct($config)
    {
        $this->_config = $config;

        $this->_connect();
    }
    // 创建连接，这里以 mysql 为例
    protected function _connect()
    {   // 将初始化参数数组结构为单独变量
        extract($this->_config, EXTR_SKIP);
        // 构建 dsn
        $dsn = 'mysql:dbname='.$dbname.
               ';host='.$host.
               ';port='.$port;
        // 构建 options
        $options = isset($options) ? $options + $this->_options : $this->_options;

        try {
            // 创建 pdo 连接
            $this->_pdo = new PDO($dsn, $user, $password, $options);

        } catch (PDOException $e) {
            // 如果失败，向外抛出异常
            throw $e;
        }
    }
    // 关闭连接
    protected function _closeConnection()
    {
        $this->_pdo = NULL;
    }
    
}

```

基类实现了接口 ConnectorInterface

接口的代码如下：

```php

namespace Drivers;

interface ConnectorInterface {
    // 使用接口指明必须实现的共有方法
    public function __construct($config);
}

```

## PDO 原生方法的暴露

虽然将 PDO 的操作封装起来，但是也有一些复杂的查询需要调用原始的接口来执行。所以，我们要将 PDO 基础的 prepare、exec、query 暴露出来，以提供一个更直接、原始的接口。

在基类 PDODriver.php 中添加方法：

```php
public function query($sql)
{
    return $this->_pdo->query($sql);
}

public function exec($sql)
{
    return $this->_pdo->exec($sql);
}

public function prepare($sql, array $driver_options = [])
{
    return $this->_pdo->prepare($sql, $driver_options);
}

```

## 自动加载

目前对 PDO 的一个简单的封装工作完成了，现在我们要测试一下这个东西好不好用。但是每次使用一个文件就 require 很没有开发效率，如果能根据命名空间自动加载文件是最好不过了。

怎么实现？有请 [composer](https://www.phpcomposer.com/) 和 [PSR-4](https://www.php-fig.org/psr/psr-4/) 登场。

- composer 是 PHP 的一个强大好用的依赖管理工具，这里不做详细解释，更多请看[相关文档](https://www.phpcomposer.com/)。
- PSR-4 是 PHP 的一个自动加载规范，composer 已经支持 PSR-4。

OK，开工

1、安装 compooser

```shell
# 任意目录下
curl -sS https://getcomposer.org/installer | php # 下载源文件并执行 (注意 composer 下载要翻墙)
cp composer.phar /usr/local/bin/composer # 将可执行文件放到已经设置环境变量的目录中
```

2、在你的项目目录中新建 src 目录，将所有的源码放到 src 中。在项目根目录中运行 composer init，随意填写一些基本信息后，会在根目录生成一个 composer.json 文件，内容类似：

```json
{
    "name": "vagrant/query-builder",
    "require": {}
}

```

修改 composer.json，添加 autoload 字段，指定自动加载规范为 psr-4 ( 这里我的命名空间是 Drivers，所以设置为 Drivers 到 src 目录的映射 )。

```json
{
    "name": "vagrant/query-builder",
    "require": {},
    "autoload": {
        "psr-4": {
            "Drivers\\": "src/" 
        }
    }
}

```

3、项目目录下运行 composer install，生成 vendor 目录，现在只要引入 vendor/autoload.php 后就可以直接通过命名空间自动加载 src 下的文件啦

项目目录下新建 test 目录 (方便对源码做一些测试)，test 目录下新建 test.php，对基类进行测试

```php
// 引入 composer 的自动加载文件
require_once dirname(dirname(__FILE__)) . '/vendor/autoload.php';
// 使用基类
use Drivers\PDODriver;
// 使用你的数据库替换配置
$config = [
    'host'        => 'localhost',
    'port'        => '3306',
    'user'        => 'username',
    'password'    => 'password',
    'dbname'      => 'database',
];

$driver = new PDODriver($config);

$results = $driver->query('select * from your_table');

foreach ($results as $result) {
    var_dump($result);
}

```

4、根目录下运行 php test/test.php，查看结果是否如同你的预期

此时的项目目录如下：

```
项目根目录/
    src/
        ConnectorInterface.php
        PDODriver.php
        Mysql.php
        Pgsql.php
        Sqlite.php
    test/
        test.php
    vendor/
        ...
    composer.json
```

## Mysql 驱动类的创建

由于 Mysql 的字符集、时区设置语句和其他数据库存在差异，同时拥有一些其它数据库没有的特性 (unix_socket 连接，严格模式等)，所以，Mysql 的驱动类有基类的功能，但又有异于基类的部分。

Mysql 继承自基类，实现了 ConnectorInterface 接口，Mysql.php 代码如下：

```php
namespace Drivers;
use PDO;
use PDOException;
use Drivers\PDODriver;

class Mysql extends PDODriver implements ConnectorInterface
{
    // 重写基类的 _connect 方法
    protected function _connect()
    {
        // 解包配置数组
        extract($this->_config, EXTR_SKIP);
        // 构建 dsn，判断是否使用 unix_socket 创建 mysql 连接
        $dsn = isset($unix_socket) ?
               'mysql:unix_socket='.$unix_socket.';dbname='.$dbname :
               'mysql:dbname='.$dbname.';host='.$host.(isset($port) ? ';port='.$port : '');
        // options 选项
        $options = isset($options) ? $options + $this->_options : $this->_options;

        try {
            // 创建连接
            $this->_pdo = new PDO($dsn, $user, $password, $options);

            // 如果需要，设置字符集
            if(isset($charset)) {
                $this->_pdo->prepare("set names $charset ".(isset($collation) ? " collate '$collation'" : ''))->execute();
            }
            // 如果需要，设置时区
            if(isset($timezone)) {
                $this->_pdo->prepare("set time_zone='$timezone'")->execute();
            }
            // 如果需要，设置严格模式
            if(isset($strict)) {
                if($strict) {
                    $this->_pdo->prepare("set session sql_mode='STRICT_ALL_TABLES'")->execute();
                } else {
                    $this->_pdo->prepare("set session sql_mode=''")->execute();
                }
            }
        } catch (PDOException $e) {
            // 创建连接失败，向外抛出异常
            throw $e;
        }
    }

}

```

测试：

修改 test.php

```php
// 引入 composer 的自动加载文件
require_once dirname(dirname(__FILE__)) . '/vendor/autoload.php';
// 使用 Mysql 驱动类
use Drivers\Mysql;
// 使用你的数据库替换配置
$config = [
    'host'        => 'localhost',
    'port'        => '3306',
    'user'        => 'username',
    'password'    => 'password',
    'dbname'      => 'database',
    'charset'     => 'utf8',
    'timezone'    => '+8:00',
    'collection'  => 'utf8_general_ci',
    'strict'      => false,
    // 'unix_socket' => '/var/run/mysqld/mysqld.sock',
];

$driver = new PDODriver($config);

$results = $driver->query('select * from your_table');

foreach ($results as $result) {
    var_dump($result);
}

```

## PostgreSql、Sqlite 驱动类的创建

PostgreSql、Sqlite 驱动类的创建和 Mysql 驱动类的创建类似，这里就不再赘述，just show code

Pgsql.php 代码如下：

```php

namespace Drivers;

use PDO;
use PDOException;
use Drivers\PDODriver;

class Pgsql extends PDODriver implements ConnectorInterface
{
    // ATTR_EMULATE_PREPARES 常量 不适用 postgresql，对基类的 _options 属性进行重写
    protected $_options = [
        PDO::ATTR_CASE => PDO::CASE_NATURAL,
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_ORACLE_NULLS => PDO::NULL_NATURAL,
        PDO::ATTR_STRINGIFY_FETCHES => false,
    ];

    // 连接方法重写
    protected function _connect()
    {
        extract($this->_config, EXTR_SKIP);

        $dsn = 'pgsql:dbname='.$dbname.
               (isset($host) ? ';host='.$host : '').
               (isset($port) ? ';port='.$port : '').
               (isset($sslmode) ? ';sslmode='.$sslmode : '');

        $options = isset($options) ? $options + $this->_options : $this->_options;

        try {

            $this->_pdo = new PDO($dsn, $user, $password, $options);

            // 字符集设置
            if(isset($charset)) {
                $this->_pdo->prepare("set names '$charset'")->execute();
            }
            // 时区设置
            if(isset($timezone)) {
                $this->_pdo->prepare("set time zone '$timezone'")->execute();
            }
            // postgresql 的 schema 路径设置
            if(isset($schema)) {
                $this->_pdo->prepare("set search_path to $schema")->execute();
            }
            // postgresql 的应用名设置
            if(isset($application_name)) {
                $this->_pdo->prepare("set application_name to '$applicationName'")->execute();
            }
        } catch (PDOException $e) {
            throw $e;
        }
    }
}


```

Sqlite 基于内存或者文件，相对简单一些

Sqlite.php 代码如下：

```php

namespace Drivers;
use PDO;
use PDOException;
use Drivers\PDODriver;

class Sqlite extends PDODriver implements ConnectorInterface
{
    protected function _connect()
    {
        extract($this->_config, EXTR_SKIP);

        // 构建 dsn
        // 判断内存模式还是文件模式
        if($dbname == ':memory:') {
            $dsn = 'sqlite::memory:';
        } else {
            // 获得 db 文件的绝对路径
            $path = realpath($dbname);

            if($path === FALSE) {
                throw new InvalidArgumentException("Database $dbname does not exist.");
            }
            $dsn = 'sqlite:'.$path;
        }

        $options = isset($options) ? $options + $this->_options : $this->_options;

        try {
            // sqlite 无需用户名密码
            $this->_pdo = new PDO($dsn, '', '', $options);

        } catch (PDOException $e) {
            throw $e;
        }
    }

}

```

实践出真知，那么测试看看吧！
