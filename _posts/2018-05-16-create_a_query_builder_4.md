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

我们先看下 laravel 中的 where exists 构造语句是什么样的：

```php
DB::table('users')
            ->whereExists(function ($query) {
                $query->select(DB::raw(1))
                      ->from('orders')
                      ->whereRaw('orders.user_id = users.id');
            })
            ->get();
```

laravel 查询构造器的 whereExists() 方法接受一个闭包，闭包接收一个查询构造器实例，用于在闭包中构造子句。

使用闭包的好处是：
- 给接受闭包参数的函数扩展功能 (进行子查询语句构造)
- 闭包传入函数中，函数可以控制这个闭包的执行方式，在闭包的执行前后可以做相应操作 (现场保护、恢复)

**基本结构**

所以参考 laravel，我们也使用传入闭包的方式，我们先确定一下 whereExists() 方法的基本结构：

```php
// $callback 闭包参数
// $condition exists、not exists 模式
// $operator and、or 模式
public function whereExists(Closure $callback, $condition = 'EXISTS', $operator = 'AND')
{
    // 判断模式是否合法
    if( ! in_array($condition, ['EXISTS', 'NOT EXISTS']) || ! in_array($operator, ['AND', 'OR'])) {
        throw new \InvalidArgumentException("Error whereExists mode");
    }
    // 初次调用？
    if($this->_where_str == '') {
        $this->_where_str = ' WHERE '.$condition.' ( ';
    } else {
        $this->_where_str .= ' '.$operator.' '.$condition.' ( ';
    }

    // 进行现场保护
    ...
    // 闭包调用，传入当前实例
    ...
    // 现场恢复
    ...

    // 返回当前实例
    return $this;
}
```

因为使用到了 Closure 限制参数类型，要在基类文件的顶部加上：
```php
use Closure;
```


**现场的保护和恢复**

上面一直再说现场的保护和恢复，那么我们保护、恢复的这个现场是什么呢？

我们先理一下构造一个普通的 SQL 的步骤：依次构造各个查询子句、使用 _buildQuery() 方法将这些子句按照固定顺序组合成 SQL。

那么在有子查询的过程中，意味着这样的步骤要经过两次，但是由于要传入当前实例 (另外新建实例需要创建连接，更复杂)，第二次查询构建会覆盖掉第一次构建的结果。所以，我们这里的现场就是这些构造用的子句字符串。

而且有了现场的保护和恢复，即使在闭包中调用闭包 (即子查询中嵌套子查询) 的情形下也能正确的构造需要的 SQL 语句。(有没有觉得很想递归呢？的确这里是借鉴了栈的使用思路。)

首先我们需要一个保存构建字符串名称的数组 (用来获取构造字符串属性)，在基类添加属性 _buildAttrs：
```php
// 这里保存了需要保护现场的构造字符串名称
protected $_buildAttrs = [
    '_table',
    '_prepare_sql',
    '_cols_str',
    '_where_str',
    '_orderby_str',
    '_groupby_str',
    '_having_str',
    '_join_str',
    '_limit_str',
];
```

然后，添加保护现场和恢复现场的方法：

```php
// 保护现场
protected function _storeBuildAttr()
{
    $store = [];
    // 将实例的相关属性保存到 $store，并返回
    foreach ($this->_buildAttrs as $buildAttr) {
        $store[$buildAttr] = $this->$buildAttr;
    }

    return $store;
}
//恢复现场
protected function _reStoreBuildAttr(array $data)
{
    // 从 $data 取数据恢复当前实例的属性
    foreach ($this->_buildAttrs as $buildAttr) {
        $this->$buildAttr = $data[$buildAttr];
    }
}
```

当然，保护了现场后，子查询要使用实例的属性时，需要的是一个初始状态的属性，所以我们还需要一个可以重置这些构造字符串的方法：

```php
protected function _resetBuildAttr()
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
}
```


**完成 whereExists()**

有了保护、恢复现场的方法，我们继续完成 whereExists() 方法：

```php
public function whereExists(Closure $callback, $condition = 'EXISTS', $operator = 'AND')
{
    if( ! in_array($condition, ['EXISTS', 'NOT EXISTS']) || ! in_array($operator, ['AND', 'OR'])) {
        throw new \InvalidArgumentException("Error whereExists mode");
    }

    if($this->_where_str == '') {
        $this->_where_str = ' WHERE '.$condition.' ( ';
    } else {
        $this->_where_str .= ' '.$operator.' '.$condition.' ( ';
    }

    // 保护现场，将构造字符串属性都保存起来
    $store = $this->_storeBuildAttr();

    /**************** 开始子查询 SQL 的构建 ****************/
        // 复位构造字符串
        $this->_resetBuildAttr();
        // 调用闭包，将当前实例作为参数传入
        call_user_func($callback, $this);
        // 子查询构造字符串数组    
        $sub_attr = [];
        // 构建子查询 SQL
        $this->_buildQuery();
        // 保存子查询构造字符串，用于外层调用
        foreach ($this->_buildAttrs as $buildAttr) {
            $sub_attr[$buildAttr] = $this->$buildAttr;
        }
    /**************** 结束子查询 SQL 的构建 ****************/

    // 恢复现场
    $this->_reStoreBuildAttr($store);

    // 获取子查询 SQL 字符串，构造外层 SQL
    $this->_where_str .= $sub_attr['_prepare_sql'].' ) ';
    
    return $this;
}
```

**测试**

构造语句 `SELECT * FROM student WHERE EXISTS ( SELECT * FROM classes WHERE id = 3);`：

```php
$results = $driver->table('student')
                  ->whereExists(function($query) {
                      $query->table('classes')
                            ->where('id', 3);
                  })
                  ->get();
```

大家在测试文件中试试看吧！

whereNotExists()、orWhereExists() 等模式就不单独演示了。完整代码请看 [WorkerF - PDODriver.php](https://github.com/wazsmwazsm/WorkerF/blob/master/src/WorkerF/DB/Drivers/PDODriver.php#L936)。

**优化**

where exists 子句用到了子查询，单并不只有 where exists 使用子查询。最直接的 `SELECT * FROM (SELECT * FROM table);` 子查询语句，where in 子查询语句也用到子查询，那么重复的逻辑要提出来，Don't Repeat Yourself！

基类中新建 _subBuilder() 方法，用来进行现场的保护恢复、子查询 SQL 的构造：
```php
protected function _subBuilder(Closure $callback)
{
    // 现场保护
    $store = $this->_storeBuildAttr();

    /**************** begin sub query build ****************/

        $this->_resetBuildAttr();

        call_user_func($callback, $this);
        
        $sub_attr = [];

        $this->_buildQuery();

        foreach ($this->_buildAttrs as $buildAttr) {
            $sub_attr[$buildAttr] = $this->$buildAttr;
        }
    /**************** end sub query build ****************/

    // 现场恢复
    $this->_reStoreBuildAttr($store);

    return $sub_attr;
}
```

修改 whereExists() 方法：
```php
public function whereExists(Closure $callback, $condition = 'EXISTS', $operator = 'AND')
{
    if( ! in_array($condition, ['EXISTS', 'NOT EXISTS']) || ! in_array($operator, ['AND', 'OR'])) {
        throw new \InvalidArgumentException("Error whereExists mode");
    }

    if($this->_where_str == '') {
        $this->_where_str = ' WHERE '.$condition.' ( ';
    } else {
        $this->_where_str .= ' '.$operator.' '.$condition.' ( ';
    }

    $sub_attr = $this->_subBuilder($callback);

    $this->_where_str .= $sub_attr['_prepare_sql'].' ) ';

    return $this;
}
```

### where in 子查询

有了上面 where exists 的基础，where in 子查询的如出一辙：

```php
public function whereInSub($field, Closure $callback, $condition = 'IN', $operator = 'AND')
{
    if( ! in_array($condition, ['IN', 'NOT IN']) || ! in_array($operator, ['AND', 'OR'])) {
        throw new \InvalidArgumentException("Error whereIn mode");
    }
    
    if($this->_where_str == '') {
        $this->_where_str = ' WHERE '.self::_wrapRow($field).' '.$condition.' ( ';
    } else {
        $this->_where_str .= ' '.$operator.' '.self::_wrapRow($field).' '.$condition.' ( ';
    }

    $sub_attr = $this->_subBuilder($callback);
    $this->_where_str .= $sub_attr['_prepare_sql'].' ) ';

    return $this;
}
```

构造 SQL `SELECT * FROM student WHERE class_id IN (SELECT id FROM class);`：

```php
$results = $driver->table('student')
                ->whereInSub('class_id', function($query) {
                    $query->table('class')->select('id');
                })
                ->get();
```


同样，where not in、or where in 这些模式就不单独展示了。

### 单纯的子查询

单纯的 SELECT * FROM (子查询) 语句的构造就很简单了：

```php
public function fromSub(Closure $callback)
{
    $sub_attr = $this->_subBuilder($callback);
    $this->_table .= ' ( '.$sub_attr['_prepare_sql'].' ) AS tb_'.uniqid().' ';

    return $this;
}
```

上述代码需要注意的地方：
- FROM 子查询的语句需要给子查询一个别名做表名，否则是语法错误，这里我们选择 uniqid() 函数生成一个随机的别名。
- 这里是用 _table 属性报存了子查询字符串，如果同时调用了 table() 方法会有冲突。

构造 SQL `SELECT username, age FROM (SELECT * FROM test_table WHERE class_id = 3)`：

```php
$results = $driver->select('username', 'age')
            ->fromSub(function($query) {
                $query->table('test_table')->where('class_id', 3);
            })
            ->get();
```

## 复杂的 where 逻辑

在基本的 where 子句中，有时候会出现复杂的逻辑运算，比如多个条件用 OR 和 AND 来组合： 

```sql
WHERE a = 1 OR a = 2 AND b = 1; 
```

AND 的优先级是大于 OR 的，如果想要先执行 OR 的条件，需要圆括号进行包裹：

```sql
WHERE a = 1 AND (b = 2 OR c = 3); 
```
AND 和 OR 我们可以用 where() 和 orWhere() 方法连接，但是圆括号的包裹还需要增加方法来实现。

**思路**

参考含有子查询的子句，我们可以把圆括号包裹的内部作为一个“子查询”字符串来看待，区别在于，我们不像是子查询中取整个子查询的 SQL，而是只取 where 子句的构造字符串。

Ok，有了思路，那就编码吧：

```php
public function whereBrackets(Closure $callback, $operator = 'AND')
{
    if( ! in_array($operator, ['AND', 'OR'])) {
        throw new \InvalidArgumentException("Logical operator");
    }
    
    if($this->_where_str == '') {
        $this->_where_str = ' WHERE ( '; // 开头的括号包裹
    } else {
        $this->_where_str .= ' '.$operator.' ( '; // 开头的括号包裹
    }
    $sub_attr = $this->_subBuilder($callback);
    // 这里只取子查询构造中的 where 子句
    // 由于子查询的 where 子句会带上 WHERE 关键字，这里要去掉
    $this->_where_str .= preg_replace('/WHERE/', '', $sub_attr['_where_str'], 1).' ) '; // 结尾的括号包裹

    return $this;
}
```

构造 SQL `SELECT * FROM test_table WHERE a = 1 AND (b = 2 OR c IS NOT NULL);`：

```php
$results = $driver->table('test_table')
            ->where('a', 1)
            ->whereBrackets(function($query) {
                $query->where('b', 2)
                      ->orWhereNotNull('c');
            })
            ->get();
```

orWhereBrackets() 就不单独演示了。
