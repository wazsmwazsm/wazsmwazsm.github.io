---
layout: post
title:  "写一个“特殊”的查询构造器 - (五、聚合函数、分组、排序、分页)"
date:   2018-05-20 20:01:01
categories: PHP
excerpt: where 相关的子句构造完成后，我们继续构造其它子句。这一篇我们进行聚合函数、分组、排序、分页等子句的构造。
---

where 相关的子句构造完成后，我们继续构造其它子句。这一篇我们进行聚合函数、分组、排序、分页等子句的构造。

## 聚合函数

在 SQL 中，有一些用来统计、汇总的函数，被称作聚合函数，如 SUM、COUNT、AVG 等。

使用 select() 方法时，我们可以用 select('COUNT(id)') 这种写法来使用聚合函数，但是这种方式有缺点：
- 语义上并不直观
- 在防止关键字冲突时需要手动添加引号(参见第三篇条件查询)
- 拿到聚合数据需要一串繁琐的方法调用

为了更方便的获得聚合数据，我们需要为其单独编写方法。

### count() 方法

基类添加 count() 方法：

```php
public function count($field = '*')
{
    // 判断是否时 * 
    // 非 * 时给字段添加引号
    if(trim($field) != '*') {
        $field = self::_quote($field);
    }
    // 构造列查询字符串
    $this->_cols_str = ' COUNT('.$field.') AS count_num ';
    // 取结果
    return $this->row()['count_num'];
}
```

构造 SQL `SELECT COUNT(id) FROM test_table;`

```php
$results = $driver->table('test_table')
            ->count('id')
```

### 其它方法

sum()、avg() 等方法没有 COUNT('*') 这样的场景，比 count() 方法的编写更简单。

sum() 方法：

```php
public function sum($field)
{
    $this->_cols_str = ' SUM('.self::_quote($field).') AS sum_num ';

    return $this->row()['sum_num'];
}
```

其它方法如 avg()、max() 等的编写就不一一展示了，代码可以在 [WorkerF - 聚合函数](https://github.com/wazsmwazsm/WorkerF/blob/master/src/WorkerF/DB/Drivers/PDODriver.php#L1321) 找到。


## 分组：group by 和 having

group by 子句指定了要分组的字段，可以是一个或多个 (用逗号隔开)。

having 子句和 group by 一起使用，作为分组的筛选条件，可以为空，除了关键字外语法和 where 子句基本相同。

基类新增 groupBy() 方法：

```php
public function groupBy($field)
{
    // 是否初次调用 ?
    if($this->_groupby_str == '') {
        $this->_groupby_str = ' GROUP BY '.self::_wrapRow($field);
    } else { // 非初次调用(多个分组字段)，使用逗号分隔
        $this->_groupby_str .= ' , '.self::_wrapRow($field);
    }

    return $this;
}
```
基类新增 having()、orHaving() 方法：

```php
public function having()
{
    $operator = 'AND';

    // 初次调用 ?
    if($this->_having_str == '') {
        $this->_having_str = ' HAVING ';
    } else {
        $this->_having_str .= ' '.$operator.' ';
    }
    // 和 where 子句一样进行条件构造
    $this->_condition_constructor(func_num_args(), func_get_args(), $this->_having_str);

    return $this;
}

public function orHaving()
{
    $operator = 'OR';

    if($this->_having_str == '') {
        $this->_having_str = ' HAVING ';
    } else {
        $this->_having_str .= ' '.$operator.' ';
    }

    $this->_condition_constructor(func_num_args(), func_get_args(), $this->_having_str);

    return $this;
}
```

这里我们也留一个处理原生字符串的 havingRaw() 方法，方便构造 `HAVING SUM(price) > 1000 ` 这类子句：

```php
public function havingRaw($string)
{
    $this->_having_str = ' HAVING '.$string.' ';

    return $this;
}
```

构造 SQL `SELECT id, SUM(price) FROM test_table GROUP BY price HAVING SUM(price) > 1000;`：

```php
$results = $driver->table('test_table')
            ->select('id', 'SUM(price)')
            ->groupBy('price')
            ->having('SUM(price)', '>', 1000)
            ->get();
// 使用 havingRaw()
$results = $driver->table('test_table')
            ->select('id', 'SUM(price)')
            ->groupBy('price')
            ->havingRaw('SUM(price) > 1000')
            ->get();
```

## 排序

排序就很简单了，固定的语法和正序倒序两个模式，多个字段排序时使用逗号隔开：

```php
public function orderBy($field, $mode = 'ASC')
{
    if( ! in_array($mode, ['ASC', 'DESC'])) {
        throw new \InvalidArgumentException("Error order by mode");
    }
    // 初次调用？
    if($this->_orderby_str == '') {
        $this->_orderby_str = ' ORDER BY '.self::_wrapRow($field).' '.$mode;
    } else {
        // 多个排序字段时，逗号隔开
        $this->_orderby_str .= ' , '.self::_wrapRow($field).' '.$mode;
    }

    return $this;
}
```

构造 SQL `SELECT * FROM tes_table ORDER BY price DESC, id ASC;`：

```php
$results = $driver->table('test_table')
            ->orderBy('price', 'DESC')
            ->orderBy('id', 'ASC')
            ->get();
```

## limit 和 分页

### limit 子句

标准 SQL 中的 limit 子句是和 limit、offset 关键字一起使用的，Mysql 中有 `LIMIT 0, 10` 的简写形式，但是 PostgreSql 和 Sqlite 并不适用。所以我们选用 limit、offset 语法：

```php
public function limit($offset, $step)
{
    $this->_limit_str = ' LIMIT '.$step.' OFFSET '.$offset.' ';

    return $this;
}
```

构造 SQL `SELECT * FROM test_table LIMIT 10 OFFSET 1;`：

```php
$results = $driver->table('test_table')
            ->limit(1, 10)
            ->get();
```

### 分页


