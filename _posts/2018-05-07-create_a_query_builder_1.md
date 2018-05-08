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

代码文件结构如下：

- ConnectorInterface.php  -- 接口文件
- PDODriver.php -- 基类
- Mysql.php -- Mysql 驱动类
- Pgsql.php -- PostgreSql 驱动类
- Sqlite.php -- Sqlite 驱动类

## 基类的创建，PDO 连接

首先要开始的就是基类，对 PDO 的连接进行封装的工作。

一个基础的基类 PDODriver.php 的代码如下：

```PHP
// 声明命名空间
namespace Drivers;
// 使用绝对命名空间的 PDO 类 
use PDO;
use PDOException;
use Closure;

class PDODriver implements ConnectorInterface
{
    protected $_pdo = NULL;

    protected $_config = [];

    public function __construct($config)
    {
        $this->_config = $config;

        $this->_connect();
    }

    protected function _connect()
    {
        extract($this->_config, EXTR_SKIP);

        $dsn = 'mysql:dbname='.$dbname.
               ';host='.$host.
               ';port='.$port;

        $options = isset($options) ? $options + $this->_options : $this->_options;

        try {

            $this->_pdo = new PDO($dsn, $user, $password, $options);

        } catch (PDOException $e) {
            throw $e;
        }
    }

    protected function _closeConnection()
    {
        $this->_pdo = NULL;
    }
    
}

```


