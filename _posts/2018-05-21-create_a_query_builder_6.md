---
layout: post
title:  "写一个“特殊”的查询构造器 - (六、关联)"
date:   2018-05-21 19:38:01
categories: PHP
excerpt: 关联查询是关系型数据库典型的查询语句，根据两个或多个表中的列之间的关系，从这些表中查询数据。在 SQL 标准中使用 JOIN 和 ON 关键字来实现。
---

关联查询是关系型数据库典型的查询语句，根据两个或多个表中的列之间的关系，从这些表中查询数据。在 SQL 标准中使用 JOIN 和 ON 关键字来实现关联查询。

## Join 子句

join 子句的构造并不难，注意事项就是关联查询的注意事项：

- 写对语法和关联的条件
- 使用 table.field 模式防止字段重名

基类中新建 join() 方法：

```php
// $table 要关联的表
// $one   作为关联条件的一个表的字段
// $two   作为关联条件的另一个表的字段
// $type  关联模式 inner、left、right
public function join($table, $one, $two, $type = 'INNER')
{
    // 判断模式是否合法
    if( ! in_array($type, ['INNER', 'LEFT', 'RIGHT'])) {
        throw new \InvalidArgumentException("Error join mode");
    }
    // 构建 join 子句字符串
    $this->_join_str .= ' '.$type.' JOIN '.self::_wrapRow($table).
        ' ON '.self::_wrapRow($one).' = '.self::_wrapRow($two);
    return $this;
}
```

leftJoin() 和 rightJoin() 方法：

```php
public function leftJoin($table, $one, $two)
{
    return $this->join($table, $one, $two, 'LEFT');
}

public function rightJoin($table, $one, $two)
{
    return $this->join($table, $one, $two, 'RIGHT');
}
```

> 注：Sqlite 是不支持 right join 的，所以 rightJoin() 方法在 Sqlite 驱动类中无效。

构建 `SELECT student.name, class.name FROM student INNER JOIN class ON student.class_id = class.id;`：

```php
$results = $driver->table('student')
            ->select('student.name', 'class.name')
            ->join('class', 'student.class_id', 'class.id')
            ->get();
```

## 表前缀

### 为什么要有表前缀

以前很多数据表放在一个数据库中的时候，需要表前缀来区分功能。虽然现在这样的情况已经很少，但是对于查询构造器而言，还是要提供一个方便的方法来对表前缀进行设置，特别是当你没有权限修改表名的时候。

### 自动添加表前缀的方法

对于有表前缀的表，我们并不想每次都写一个前缀，这样会导致前缀更改后，应用层要跟着修改。所以我们将表前缀作为一个配置参数传入查询构造器，在查询构造器的底层进行自动前缀添加。

表前缀的配置，假设表前缀为 'test_' ：

```php
// 以 mysql 为例
$config = [
    'host'        => 'localhost',
    'port'        => '3306',
    'user'        => 'username',
    'password'    => 'password',
    'dbname'      => 'dbname',
    'charset'     => 'utf8',
    'prefix'      => 'test_',
    'timezone'    => '+8:00',
    'collection'  => 'utf8_general_ci',
    'strict'      => false,
    // 'unix_socket' => '/var/run/mysqld/mysqld.sock',
];
$db = new Mysql($config);
```

进行自动添加前缀的方法：

```php
protected function _wrapTable($table)
{
    // 构造函数传入的配置中有前缀参数吗？
    $prefix = array_key_exists('prefix', $this->_config) ?
            $this->_config['prefix'] : '';
    // 拼接前缀
    return $prefix.$table;
}
```

修改 table() 方法：

```php
public function table($table)
{
    // 自动添加前缀
    $this->_table = self::_wrapRow($this->_wrapTable($table));

    return $this;
}
```

join 子句中也涉及到表，所以修改 join() 方法：

```php
public function join($table, $one, $two, $type = 'INNER')
{
    if( ! in_array($type, ['INNER', 'LEFT', 'RIGHT'])) {
        throw new \InvalidArgumentException("Error join mode");
    }
    // 添加表前缀
    $table = $this->_wrapTable($table);
    
    $this->_join_str .= ' '.$type.' JOIN '.self::_wrapRow($table).
        ' ON '.self::_wrapRow($one).' = '.self::_wrapRow($two);
    return $this;
}
```

### table.field 模式的表前缀添加

增加了表前缀后，我们会发现一个问题：

使用 table()、join() 方法传入的表可以自动的添加前缀，但是 table.field 格式中的表没法自动添加前缀，如上面的 `join('class', 'student.class_id', 'class.id')`，我们总不能每次都写成 `join('class', 'test_student.class_id', 'test_class.id')` 这种 (这样的话和全部手工添加前缀没什么两样)，必须找到一个自动添加前缀的办法。

观察 table.field 模式，它出现的位置不定，可能在列、任何一个子句中出现，所以在固定的位置去添加前缀是不大可能的。那么我们反过来想一下，如果在 SQL 已经构造完成但还未执行时，这时已经知道有哪些地方使用了这种格式，去一一替换即可。那么如何知道有哪些地方使用了这种格式？

**使用正则**

我们用正则表达式找到 table.field 的 table 部分，给 table 加上表前缀即可 (这里不考虑跨库查询时三个点的情况)。

基类新增 _wrapPrepareSql() 方法：

```php
// 替换 table.field 为 prefixtable.field
protected function _wrapPrepareSql()
{
    $quote = static::$_quote_symbol;
    $prefix_pattern = '/'.$quote.'([a-zA-Z0-9_]+)'.$quote.'(\.)'.$quote.'([a-zA-Z0-9_]+)'.$quote.'/';
    $prefix_replace = self::_quote($this->_wrapTable('$1')).'$2'.self::_quote('$3');

    $this->_prepare_sql = preg_replace($prefix_pattern, $prefix_replace, $this->_prepare_sql);
}
```

修改 _execute() 方法：

```php
protected function _execute()
{
    try {
        // table.field 模式添加表前缀
        $this->_wrapPrepareSql();
        $this->_pdoSt = $this->_pdo->prepare($this->_prepare_sql);
        $this->_bindParams();
        $this->_pdoSt->execute();
        $this->_reset();
    } catch (PDOException $e) {
        if($this->_isTimeout($e)) { 

            $this->_closeConnection();
            $this->_connect();
            
            try {
                // table.field 模式添加表前缀
                $this->_wrapPrepareSql();
                $this->_pdoSt = $this->_pdo->prepare($this->_prepare_sql);
                $this->_bindParams();
                $this->_pdoSt->execute();
                $this->_reset();
            } catch (PDOException $e) {
                throw $e;
            }
        } else {
            throw $e;
        }
    }

}
```

最后我们进行一个完整的测试：

```php
require_once dirname(dirname(__FILE__)) . '/vendor/autoload.php';

use Drivers\Mysql;

$config = [
    'host'        => 'localhost',
    'port'        => '3306',
    'user'        => 'username',
    'password'    => 'password',
    'dbname'      => 'database',
    'prefix'      => 'test_',
    'charset'     => 'utf8',
    'timezone'    => '+8:00',
    'collection'  => 'utf8_general_ci',
    'strict'      => false,
];

$driver = new Mysql($config);

$results = $driver->table('student')
    ->select('student.name', 'class.name')
    ->join('class', 'student.class_id', 'class.id')
    ->get();

var_dump($results);
```

试试看吧！

## 复杂语句的构造

到目前位置，查询相关的 SQL 构造方法基本开发完毕，我们进行一些复杂的 SQL 构造吧。

> 注：这里只是以我的测试环境举例，大家可以按照自己的思路去建表

构造语句 `SELECT * FROM t_user WHERE username = 'Jackie aa' OR ( NOT EXISTS ( SELECT * FROM t_user WHERE username = 'Jackie aa' ) AND (username = 'Jackie Conroy' OR username = 'Jammie Haag')) AND g_id IN ( SELECT id FROM t_user_group) ORDER BY id DESC LIMIT 1 OFFSET 0 `：

```php
$results = $driver->table('user')
            ->where('username', 'Jackie aa')
            ->orWhereBrackets(function($query) {
                $query->whereNotExists(function($query) {
                    $query->table('user')->where('username', 'Jackie aa');
                })->WhereBrackets(function($query) {
                    $query->where('username', 'Jackie Conroy')
                            ->orWhere('username', 'Jammie Haag');
                });
            })
            ->whereInSub('g_id', function($query) {
                $query->table('user_group')->select('id');
            })
            ->orderBy('id', 'DESC')
            ->limit(0, 1)
            ->get();
```
构造语句 `SELECT t_user.username, t_user_group.groupname FROM t_user LEFT JOIN t_user_group ON t_user.g_id = t_user_group.id WHERE username = 'Jackie aa' OR ( NOT EXISTS ( SELECT * FROM t_user WHERE username = 'Jackie aa' ) AND username = 'Jackie Conroy' )`：

```php
$results = $driver->table('user')
            ->select('user.username', 'user_group.groupname')
            ->leftJoin('user_group', 'user.g_id', 'user_group.id')
            ->where('user.username', 'Jackie aa')
            ->orWhereBrackets(function($query) {
                $query->whereNotExists(function($query) {
                    $query->table('user')->where('username', 'Jackie aa');
                })->where('user.username', 'Jackie Conroy');
            })
            ->get();
```

更多例子参考 [WorkerF 单元测试 - PDODQL](https://github.com/wazsmwazsm/WorkerF/blob/master/tests/DB/PDODQL.php#L675)
