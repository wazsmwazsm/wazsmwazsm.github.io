---
layout: post
title:  "写一个“特殊”的查询构造器 - (二)"
date:   2018-05-09 16:08:22
categories: PHP
excerpt: 上一篇完成了代码结构的搭建和 PDO 的基础封装，这一篇我们来讲如何构造一个最基本的 SQL 语句并执行得到结果。
---

## 构造、执行第一条语句

上一篇完成了代码结构的搭建和 PDO 的基础封装，这一篇我们来讲如何构造一个最基本的 SQL 语句并执行得到结果。

**构造目标，query sql ："SELECT * FROM test_table;"**

**构造器执行语法：$drivers->table('test_table')->select('\*')->get();**

我们回顾下 PDO 执行这个 query 语句的基本用法：

1、PDO::query() 方法获取结果集：

```php
$pdo->query("SELECT * FROM test_table;");
```

2、PDO::prepare()、PDOStatement::execute() 方法：

```php
$pdoSt = $pdo->prepare("SELECT * FROM test_table;");
$pdoSt->execute();
$pdoSt->fetchAll(PDO::FETCH_ASSOC);
```

PDO::prepare() 方法提供了防注入、参数绑定的机制，可以指定结果集的返回格式，更加灵活易于封装，我们选这种。

**query sql 字符串构造：**

要构造 query sql 语句，那么不妨先观察一下它的基本构造：

SELECT、 要查找的字段(列)、 FROM、 要查找的表、 关联子句、 条件子句、 分组子句、 排序子句、 limit 子句。

除了 SELECT 和 FROM 是固定不变，我们只需构造好查询字段、表和一系列子句的字符串，然后按照 query sql 的语法拼接在一起即可。

在基类 PDODriver.php 中添加属性作为构造字符串：

```php

protected $_table = '';        // table 名
protected $_prepare_sql = '';  // prepare 方法执行的 sql 语句
protected $_cols_str = ' * ';  // 需要查询的字段，默认为 * (全部)
protected $_where_str = '';    // where 子句
protected $_orderby_str = '';  // order by 子句
protected $_groupby_str = '';  // group by 子句
protected $_having_str = '';   // having 子句 (配合 group by 使用)
protected $_join_str = '';     // join 子句
protected $_limit_str = '';    // limit 子句

```

**基础方法的创建**

有了基本的构造字符串属性，可以开始构造一条 sql 了。

为构建构建 sql 功能添加方法：

```php

protected function _buildQuery()
{
    $this->_prepare_sql = 'SELECT '.$this->_cols_str.' FROM '.$this->_table.
        $this->_join_str.
        $this->_where_str.
        $this->_groupby_str.$this->_having_str.
        $this->_orderby_str.
        $this->_limit_str;
}


```

添加 table() 方法：

```php
public function table($table)
{
    $this->_table = $table;

    return $this; // 为了链式调用，返回当前实例
}

```

添加 select() 方法，这里使用可变参数灵活处理传入：

```php
public function select()
{
    // 获取传入方法的所有参数
    $cols = func_get_args();

    if( ! func_num_args() || in_array('*', $cols)) {
        // 如果没有传入参数，默认查询全部字段
        $this->_cols_str = ' * ';
    } else {

        $this->_cols_str = ''; // 清除默认的 * 值
        // 构造 "field1, filed2 ..." 字符串
        foreach ($cols as $col) {
            $this->_cols_str .= ' '.$col.',';
        }
        $this->_cols_str = rtrim($this->_cols_str, ',');
    }

    return $this;
}
```

**构造、执行**

sql 字符串构造完毕，接下来就需要一个执行 sql 并取得结果的方法来收尾。

添加 get() 方法：

```php
public function get()
{
    try {
        $this->_buildQuery();   // 构建 sql
        // prepare 预处理
        $pdoSt = $this->_pdo->prepare($this->_prepare_sql);
        // 执行
        $pdoSt->execute();
    } catch (PDOException $e) {
        throw $e;
    }

    return $pdoSt->fetchAll(PDO::FETCH_ASSOC); // 获取一个以键值数组形式的结果集
}

```

**测试**

修改 test/test.php：

```php
require_once dirname(dirname(__FILE__)) . '/vendor/autoload.php';

use Drivers\Mysql;

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
];

$driver = new PDODriver($config);
// 执行 SELECT * FROM test_table; 的查询
$results = $driver->table('test_table')->select('*')->get();

var_dump($results);

```

## 优化

**1、解耦**

get 方法中的 prepare、execute 过程是通用的 (查询、插入、删除、更新等操作)，我们可以将这部分代码提出来，在其它执行 sql 取结果的方法中复用。

基类中新建 _execute() 方法：

```php
protected function _execute()
{
    try {
        $this->_pdoSt = $this->_pdo->prepare($this->_prepare_sql);
        $this->_pdoSt->execute();

    } catch (PDOException $e) {
        throw $e;
    }

}

```

由于将逻辑分离到另一个方法中，get() 方法获取不到 PDOStatement 实例，因此将 PDOStatement 实例保存到基类的属性中：

```php
protected $_pdoSt = NULL;
```

修改后的 get() 方法：

```php
public function get()
{
    $this->_buildQuery();
    $this->_execute();

    return $this->_pdoSt->fetchAll(PDO::FETCH_ASSOC);
}
```

**2、参数重置**

使用查询构造器一次查询后，各个构造字符串的内容已经被修改，为了不影响下一次查询，需要将这些构造字符串恢复到初始状态。

注：在常驻内存单例模式下，这种多次用一个类进行查询的情形很常见。

添加 _reset() 方法：

```php
protected function _reset()
{   
    $this->_table = '';
    $this->_prepare_sql = '';
    $this->_cols_str = ' * ';
    $this->_where_str = '';
    $this->_orderby_str = '';
    $this->_groupby_str = '';
    $this->_having_str = '';
    $this->_join_str = '';
    $this->_limit_str = '';
    $this->_insert_str = '';
    $this->_update_str = '';
    $this->_bind_params = [];
}
```

修改 _execute() 方法：

```php
protected function _execute()
{
    try {
        $this->_pdoSt = $this->_pdo->prepare($this->_prepare_sql);
        $this->_pdoSt->execute();
        $this->_reset(); // 每次执行 sql 后将各构造字符串恢复初始状态，保证下一次查询的正确性
    } catch (PDOException $e) {
        throw $e;
    }

}

```

## 断线重连




## 参考

【1】[PDO](http://php.net/manual/en/book.pdo.php)
