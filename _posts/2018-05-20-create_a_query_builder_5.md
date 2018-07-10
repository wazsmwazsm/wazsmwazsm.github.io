---
layout: post
title:  "写一个“特殊”的查询构造器 - (五、聚合函数、分组、排序、分页)"
date:   2018-05-20 20:01:01
categories: PHP
excerpt: where 相关的子句构造完成后，我们继续构造其它子句。这一篇我们进行聚合函数、分组、排序等子句的构造。
---

where 相关的子句构造完成后，我们继续构造其它子句。这一篇我们进行聚合函数、分组、排序等子句的构造。

## 聚合函数

在 SQL 中，有一些用来统计、汇总的函数，被称作聚合函数，如 SUM、COUNT、AVG 等。

使用 select() 方法时，我们可以用 select('COUNT(id)') 这种写法来使用聚合函数，但是这种方式有缺点：
- 语义上并不直观
- 在防止关键字冲突时需要手动添加引号(参见第三篇条件查询)
- 拿到聚合数据需要一串繁琐的方法调用

为了更方便的获得聚合数据，我们需要为其单独编写方法。


### getList() 方法

获得某一列的方法可以由 PDO::FETCH_COLUMN 来完成。

基类添加 getList() 方法：

```php
public function getList($field)
{
    $this->_cols_str = ' '.self::_quote($field).' ';
    $this->_buildQuery();
    $this->_execute();
    // 获取一列数据
    return $this->_pdoSt->fetchAll(PDO::FETCH_COLUMN, 0);
}
```


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
            ->count('id');
```

由于聚合函数方法和 get()、row() 方法一样是取结果的，链式调用时切记要放到最后执行。

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

其它方法如 avg()、max() 之类的编写就不一一展示了，代码请看 [WorkerF - 聚合函数](https://github.com/wazsmwazsm/WorkerF/blob/master/src/WorkerF/DB/Drivers/PDODriver.php#L1321) 。


## 分组：group by 和 having

group by 子句指定了要分组的字段，可以是一个或多个 (用逗号隔开)。

having 子句和 group by 子句一起使用，作为分组的筛选条件，可以为空，除了关键字外，语法和 where 子句基本相同。

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

这里我们也留一个处理原生字符串的 havingRaw() 方法 (手动填写数据，不进行数据绑定)：

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

排序就很简单了，固定的语法和正序、倒序两个模式，多个字段排序时使用逗号隔开：

```php
public function orderBy($field, $mode = 'ASC')
{
    $mode = strtoupper($mode);
    if ( ! in_array($mode, ['ASC', 'DESC'])) {
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

标准 SQL 中的 limit 子句是 limit、offset 关键字一起使用的，Mysql 中有 `LIMIT 0, 10` 的简写形式，但是 PostgreSql 和 Sqlite 并不适用。所以我们选用 limit、offset 语法：

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

在数据请求时会遇到请求资源数据量大的问题，对于这个问题，普遍的解决方案就是分页。

有了 limit() 方法，可以进行分页功能的实现了。

为了灵活的访问分页，我们要返回以下的数据：

- 数据的总数
- 每页的数据个数
- 当前页
- 下一页
- 上一页
- 第一页
- 最后一页
- 当前页的数据集合

**对于获取数据的总数，这里有一些问题。**

如何获取总数？当然是使用上面讲到的聚合函数 count() 来处理。但是，使用了 count() 方法后相当于进行了一次 SQL 的构造和执行 (执行后会将构造字符串设置为初始状态)，那么还如何进行当页数据集合的获取呢？

**两次执行**

当然是有解决方案的，就是构造两次。

我们的查询构造器结构每次新建一个实例就会得到一个 PDO 的连接，所以为了构造两次而新建实例的话，代价太大，那么如何在一个实例中实现两次执行？

回顾上一篇，含有子查询的 SQL 构造时经过了两次构造，对构造字符串进行了保护、恢复。那么换到分页中，还是构造两次 SQL，对构造字符串进行保护、恢复，区别就是：因为要执行两次，所以要对绑定的数据也进行保护、恢复。

基类中添加绑定数据保存、恢复的方法：

```php
// 保存绑定数据
protected function _storeBindParam()
{
    return $this->_bind_params;
}
// 恢复绑定数据
protected function _reStoreBindParam($bind_params)
{
    $this->_bind_params = $bind_params;
}
```

基类中添加 paginate() 方法： 

```php
public function paginate($step, $page = NULL)
{
    // 保存构造字符串和绑定数据
    $store = $this->_storeBuildAttr();
    $bind_params = $this->_storeBindParam();
    // 获取数据总数
    $count = $this->count();
    // 恢复构造字符串和绑定数据
    $this->_reStoreBuildAttr($store);
    $this->_reStoreBindParam($bind_params);

    // 创建分页数据
    $page = $page ? $page : 1; // 第一页
    $this->limit($step * ($page - 1), $step);

    $rst['total']        = $count;
    $rst['per_page']     = $step;
    $rst['current_page'] = $page;
    $rst['next_page']    = ($page + 1) > ($count / $step) ? NULL : ($page + 1);
    $rst['prev_page']    = ($page - 1) < 1 ? NULL : ($page - 1);
    $rst['first_page']   = 1;
    $rst['last_page']    = $count / $step;
    $rst['data']         = $this->get();
    // 返回结果
    return $rst;
}
```

测试一下，获取从第五页的数据，每页 10 条数据：

```php

$results = $driver->table('test_table')
            ->paginate(10, 5);
```

Just do it!
