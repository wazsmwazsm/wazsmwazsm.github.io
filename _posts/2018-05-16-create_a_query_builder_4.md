---
layout: post
title:  "写一个“特殊”的查询构造器 - (四、条件查询：复杂条件)"
date:   2018-05-16 14:01:12
categories: PHP
excerpt: 在 SQL 的条件查询中，不只有 where、or where 这些基本的子句，还有 where in、where exists、where between 等复杂一些的子句。而且即使是 where 这种基础的子句，也有多个条件的多种逻辑组合。这篇我们就来讲一下查询构造器如何构造这些复杂的查询语句。
---

## 复杂的条件

在 SQL 的条件查询中，不只有 where、or where 这些基本的子句，还有 where in、where exists、where between 等复杂一些的子句。而且即使是 where 这种基础的子句，也有多个条件的多种逻辑组合。这篇我们就来讲一下查询构造器如何构造这些复杂的查询语句。

## where 系列

### where in 子句

我们回想一下使用 where in 子句的 SQL 是什么样的：

```sql
-- 从一个数据范围获取
SELECT * FROM test_table WHERE age IN (18, 20, 22, 24);
-- 从一个子查询获取
SELECT * FROM test_table WHERE username IN (SELECT username FROM test_name_table);
```

从一个子查询获取的模式有些复杂我们稍后再说，先分析下从数据范围获取的方式。

where in 子句判断字段是否属于一个数据集合，有 where in、where not in、or where in、or where not in 四种模式。我们只需构造好这个数据集合，并对集合中的数据进行数据绑定即可。

基类中添加 whereIn() 方法:

```php
// $field where in 要查的字段
// $data  进行判断的数据集合
// $condition in、not in 模式
// $operator AND、OR 分隔符
public function whereIn($field, array $data, $condition = 'IN', $operator = 'AND')
{
    // 判断模式和分隔符是否合法
    if( ! in_array($condition, ['IN', 'NOT IN']) || ! in_array($operator, ['AND', 'OR'])) {
        throw new \InvalidArgumentException("Error whereIn mode");
    }
    // 生产占位符，绑定数据
    foreach ($data as $key => $value) {
        $plh = self::_getPlh();
        $data[$key] = $plh;
        $this->_bind_params[$plh] = $value;
    }
    // 第一次调用该方法，需要 WHERE 关键字
    if($this->_where_str == '') {
        $this->_where_str = ' WHERE '.self::_wrapRow($field).' '.$condition.' ('.implode(',', $data).')';
    } else { // 非初次调用，使用分隔符连接
        $this->_where_str .= ' '.$operator.' '.self::_wrapRow($field).' '.$condition.' ('.implode(',', $data).')';
    }
    // 方便链式调用，返回当前实例
    return $this;
}
```

关于上述代码，由于 where in、where not in、or where in、or where not in 这写方法的区别只是关键字的区别，对于字符串来说只需替换关键字即可。所有对于这些方法，为了方便我们把这些模式的关键字作为方法的参数传入，可以提高代码的重用性。

那么，另外三种模式的代码可以这么写：

```php
public function orWhereIn($field, array $data)
{
    return $this->whereIn($field, $data, 'IN', 'OR');
}

public function whereNotIn($field, array $data)
{
    return $this->whereIn($field, $data, 'NOT IN', 'AND');
}

public function orWhereNotIn($field, array $data)
{
    return $this->whereIn($field, $data, 'NOT IN', 'OR');
}
```
**构造测试**

```php
$driver->table('test_table')
       ->whereIn('age', [18, 20, 22, 24])
       ->get();

$driver->table('test_table')
       ->Where('age', '!=', 12)
       ->orWhereNotIn('age', [13, 23, 26, 25])
       ->get();
```

### where between 子句

where between 子句的构造和 where in 相差无几，只有语法上的区别，而且只有 where between and、or where between and 两种模式。

whereBetween 系列方法代码：

```php
public function whereBetween($field, $start, $end, $operator = 'AND')
{
    // 检测模式是否合法
    if( ! in_array($operator, ['AND', 'OR'])) {
        throw new \InvalidArgumentException("Logical operator");
    }
    // 生产占位符，绑定数据
    $start_plh = self::_getPlh();
    $end_plh = self::_getPlh();
    $this->_bind_params[$start_plh] = $start;
    $this->_bind_params[$end_plh] = $end;

    // 是否初次访问？
    if($this->_where_str == '') {
        $this->_where_str = ' WHERE '.self::_wrapRow($field).' BETWEEN '.$start_plh.' AND '.$end_plh;
    } else {
        $this->_where_str .= ' '.$operator.' '.self::_wrapRow($field).' BETWEEN '.$start_plh.' AND '.$end_plh;
    }

    return $this;
}

public function orWhereBetween($field, $start, $end)
{
    return $this->whereBetween($field, $start, $end, 'OR');
}
```

### where null 子句

前面的 where 子句中，使用单条件模式时，数据为 NULL 时则进行 IS NULL 的判断。但是我们想要一个更灵活和语义清晰的接口，所以这里为 NULL 的判断单独编写方法。

where null 系列代码：

```php
public function whereNull($field, $condition = 'NULL', $operator = 'AND')
{
    if( ! in_array($condition, ['NULL', 'NOT NULL']) || ! in_array($operator, ['AND', 'OR'])) {
        throw new \InvalidArgumentException("Logical operator");
    }
    if($this->_where_str == '') {
        $this->_where_str = ' WHERE ';
    } else {
        $this->_where_str .= ' '.$operator.' ';
    }

    $this->_where_str .= self::_wrapRow($field).' IS '.$condition.' ';

    return $this;
}

public function whereNotNull($field)
{
    return $this->whereNull($field, 'NOT NULL', 'AND');
}

public function orWhereNull($field)
{
    return $this->whereNull($field, 'NULL', 'OR');
}

public function orWhereNotNull($field)
{
    return $this->whereNull($field, 'NOT NULL', 'OR');
}
```

### where exists

到 where exists 子句时，构造就有些难度了。我们回忆一下使用 where exists 子句的 SQL：

```sql
SELECT * FROM table1 where exists (SELECT * FROM table2);
```

没错，和之前构造的语句不同，where exists 子句存在子查询。之前的 sql 构造都是通过 _buildQuery() 方法按照查询的顺序构造的，那么如何对子查询进行构造呢？子查询中的 where 子句和外层查询的 where 子句同时存在时，又该怎么区分呢？

首先，观察一下有子查询的 SQL，可以看出：**子查询是一个独立的查询语句。**

那么，能不能将子查询语句和外层查询语句各自单独构造，然后再组合到一起成为一条完整的 SQL 呢？

当然是可以的。不过，如何去单独构造子查询语句呢？如果子查询中还有子查询语句呢？

这里一个常见的编程技巧就派上用场了：**递归**

递归的底层是栈来实现的，每次进入下一层的时候，保存当前层的数据 (保护现场)，当每次退出当前的时候，恢复上一层的数据 (恢复现场)，递归的深度即栈的深度。

那在我们的查询构造器中，应该使用什么实现递归呢？当然是函数。我们先看下 laravel 中是如何实现 where exists 的构造的：




