---
layout: post
title:  "写一个“特殊”的查询构造器 - (二、第一条语句)"
date:   2018-05-09 16:08:22
categories: PHP
excerpt: 上一篇完成了代码结构的搭建和 PDO 的基础封装，这一篇我们来讲如何构造一个最基本的 SQL 语句并执行得到结果。
---

## 构造、执行第一条语句

上一篇完成了代码结构的搭建和 PDO 的基础封装，这一篇我们来讲如何构造一个最基本的 SQL 语句，并执行得到结果。

**query sql 构造目标：** `SELECT * FROM test_table;`

**查询构造器执行语法构造目标：** `$drivers->table('test_table')->select('\*')->get();`

测试用的数据表请大家自己建立，这里就不单独演示了。

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

### query sql 字符串构造：

要构造 query sql 语句，那么不妨先观察一下它的基本构造：

> SELECT、 要查找的字段(列)、 FROM、 要查找的表、 关联子句、 条件子句、 分组子句、 排序子句、 LIMIT 子句。

除了 SELECT 和 FROM 是固定不变，我们只需构造好查询字段、表名和一系列子句的字符串，然后按照 query sql 的语法拼接在一起即可。

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

### 基础方法的创建

有了基本的构造字符串属性，可以开始构造一条 sql 了。

添加 _buildQuery() 方法，用来构造 sql 字符串：

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

添加 table() 方法，用来设置表名：

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

### 构造、执行

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

### 测试

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

$driver = new Mysql($config);
// 执行 SELECT * FROM test_table; 的查询
$results = $driver->table('test_table')->select('*')->get();

var_dump($results);
```

>注：上述代码中由于 \_cols\_str 属性默认为 ' * '，所以在查询全部字段时省略 select() 方法的调用也是可以的。

之后为了节省篇幅，一些通用的方法只使用 Mysql 驱动类作为测试对象，PostgreSql 和 Sqlite 请读者自己进行测试，之后不会再单独说明。

## 优化

### 1、解耦

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

### 2、参数重置

使用查询构造器一次查询后，各个构造字符串的内容已经被修改，为了不影响下一次查询，需要将这些构造字符串恢复到初始状态。

>注：在常驻内存单例模式下，这种多次用一个类进行查询的情形很常见。

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

## row() 方法

上述的 get() 方法是直接取得整个结果集。而有一些业务逻辑希望只取一行结果，那么就需要一个 row() 方法来实现这个需求了。

row() 方法并不难，只需把 get() 方法中的 PDOStatement::fetchAll() 方法改为 PDOStatement::fetch() 方法即可：

```php
public function row()
{
    $this->_buildQuery();
    $this->_execute();

    return $this->_pdoSt->fetch(PDO::FETCH_ASSOC);
}
```
这里就不多说了，大家可以自己测试一下结果。


## 断线重连

对于典型 web 环境而言，一次 sql 的查询已经随着 HTTP 的请求而结束，PHP 的垃圾回收功能会回收一次请求周期内的数据。而一次 HTTP 请求的时间也相对较短，基本不用考虑数据库断线的问题。

但在常驻内存的环境下，尤其是单例模式下，数据库驱动类可能一直在内存中不被销毁。如果很长时间内没有对数据库进行访问的话，由数据库驱动类建立的数据库连接会被数据库作为空闲连接切断 (具体时间由数据库设置决定)，此时如果依旧使用旧的连接对象，会出现持续报错的问题。也就是说，我们要对数据库断线的情况进行处理，在检测到断线的同时新建一个连接代替旧的连接继续使用。

在 PDO 中，数据库断线后继续访问会相应的抛出一个 PDOException 异常 (也可以是一个错误，由 PDO 的错误处理设置决定)。

当数据库出现错误时，PDOException 实例的 errorInfo 属性中保存了错误的详细信息数组，第一个元素返回 SQLSTATE error code，第二个元素是具体驱动错误码，第三个元素是具体的错误信息。参见 [PDO::errorInfo](http://php.net/manual/en/pdo.errorinfo.php)

**Mysql 断线相关的错误码有两个：**
- [2006 CR_SERVER_GONE_ERROR](https://dev.mysql.com/doc/refman/5.5/en/error-messages-client.html#error_cr_server_gone_error) 
- [2013 CR_SERVER_LOST](https://dev.mysql.com/doc/refman/5.5/en/error-messages-client.html#error_cr_server_lost)

**PostgreSql 断线相关的错误码有一个：**

当具体驱动错误码为 7 时 PostgreSql 断线 (此驱动错误码根据 PDOException 实测得出，暂时未找到相关文档)

Sqlite 基于内存和文件，不存在断线一说，不做考虑。


这里我们使用 PDO 的具体驱动错误码作为判断断线的依据。

基类添加 _isTimeout() 方法：

```php
protected function _isTimeout(PDOException $e)
{
    // 异常信息满足断线条件，则返回 true
    return (
        $e->errorInfo[1] == 2006 ||   // MySQL server has gone away (CR_SERVER_GONE_ERROR)
        $e->errorInfo[1] == 2013 ||   // Lost connection to MySQL server during query (CR_SERVER_LOST)
        $e->errorInfo[1] == 7         // no connection to the server (for postgresql)
    );
}
```

修改 _execute() 方法，添加断线重连功能：

```php
protected function _execute()
{
    try {
        $this->_pdoSt = $this->_pdo->prepare($this->_prepare_sql);
        $this->_pdoSt->execute();
        $this->_reset();
    } catch (PDOException $e) {
        // PDO 抛出异常，判断是否是数据库断线引起
        if($this->_isTimeout($e)) { 
            // 断线异常，清除旧的数据库连接，重新连接
            $this->_closeConnection();
            $this->_connect();
            // 重试异常前的操作    
            try {
                $this->_pdoSt = $this->_pdo->prepare($this->_prepare_sql);
                $this->_pdoSt->execute();
                $this->_reset();
            } catch (PDOException $e) {
                // 还是失败、向外抛出异常
                throw $e;
            }
        } else {
            // 非断线引起的异常，向外抛出，交给外部逻辑处理
            throw $e;
        }
    }
}
```

顺便把之前暴露的 PDO 的原生接口也支持断线重连：

```php
public function query($sql)
{
    try {
        return $this->_pdo->query($sql);
    } catch (PDOException $e) {
        // when time out, reconnect
        if($this->_isTimeout($e)) {

            $this->_closeConnection();
            $this->_connect();

            try {
                return $this->_pdo->query($sql);
            } catch (PDOException $e) {
                throw $e;
            }

        } else {
            throw $e;
        }
    }
}

public function exec($sql)
{
    try {
        return $this->_pdo->exec($sql);
    } catch (PDOException $e) {
        // when time out, reconnect
        if($this->_isTimeout($e)) {

            $this->_closeConnection();
            $this->_connect();

            try {
                return $this->_pdo->exec($sql);
            } catch (PDOException $e) {
                throw $e;
            }

        } else {
            throw $e;
        }
    }
}

public function prepare($sql, array $driver_options = [])
{
    try {
        return $this->_pdo->prepare($sql, $driver_options);
    } catch (PDOException $e) {
        // when time out, reconnect
        if($this->_isTimeout($e)) {

            $this->_closeConnection();
            $this->_connect();

            try {
                return $this->_pdo->prepare($sql, $driver_options);
            } catch (PDOException $e) {
                throw $e;
            }

        } else {
            throw $e;
        }
    }
}
```


**如何模拟断线？**

在内存常驻模式中 (如 workerman 的 server 监听环境下)：

- 访问数据库
- 重启服务器的数据库软件
- 再次访问数据库，看看是否能正常获取数据。

Just do it

## 参考

【1】[PHP Document: The PDO class](http://php.net/manual/en/class.pdo.php)

【2】[Workerman mysql](https://github.com/walkor/mysql)
