---
layout: post
title:  "写一个“特殊”的查询构造器 - (六、关联)"
date:   2018-05-21 19:38:01
categories: PHP
excerpt: 关联查询是关系型数据库典型的查询语句，根据两个或多个表中的列之间的关系，从这些表中查询数据。在 SQL 标准中使用 JOIN 和 ON 关键字来实现。
---

关联查询是关系型数据库典型的查询语句，根据两个或多个表中的列之间的关系，从这些表中查询数据。在 SQL 标准中使用 JOIN 和 ON 关键字来实现。

## Join 子句

join 子句的构造并不难，注意的事项就是关联查询的事项：

- 写对语法和关联的条件
- 使用表前缀防止字段重名

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

对于有表前缀的表，我们并不想每次都写一个前缀，而且当前缀更改后，应用层要跟着修改。所以我们将表前缀作为一个配置参数，在查询构造器的底层进行自动前缀添加。

表前缀可以作为新建查询构造器实例的时候作为配置参数传入，假设表前缀为 'test_' ：

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
