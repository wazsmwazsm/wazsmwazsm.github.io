---
layout: post
title:  "写一个“特殊”的查询构造器 - (三)"
date:   2018-05-12 21:12:22
categories: PHP
excerpt: 
---

## 构造 where 条件

如果单单是执行 SELECT * FROM test_table; 这样的语句，使用原生扩展就好了，使用查询构造器就是杀鸡用牛刀。当然，在实际的业务需求中，大部分的 SQL 都没这么简单，有各种条件查询、分组、排序、连表等操作，尤其是条件查询，占到了查询业务的大多数。

这一篇，我们来讲讲如何使用查询构造器进行条件查询。

### 条件的核心：参数绑定

首先，我们回顾一下用 PDO 来写条件查询该怎么做：

1、构造语句、预编译：

PDO 可以通过占位符绑定参数，占位符可以使用 :name 的形式或者 ? 的形式。

```php

$pdoSt = $pdo->prepare("SELECT * FROM test_table WHERE username = :username AND age = :age;");

```

2、进行参数绑定，执行语句：

[PDOStatement::bindParam()](http://php.net/manual/en/pdostatement.bindparam.php) 和 [PDOStatement::bindValue()](http://php.net/manual/en/pdostatement.bindvalue.php) 方法可以绑定一个 PHP 变量到指定的占位符。 

```php

$username = 'test';
$age = 18;

$pdoSt->bindValue(':username', $username, PDO::PARAM_STR);
$pdoSt->bindValue(':age', $age, PDO::PARAM_INT);
$pdoSt->execute();
```

由此我们得知，只要搞定了参数绑定，就可以构造一个简单的 where 子句了。

### 占位符和绑定方法的选择

**占位符选择：**

? 占位符必须按照顺序去绑定，而 :name 占位符只要占位符和数据的映射关系确定，绑定的数据就不会出错。所以我们选择 :name 占位符。

**绑定方法的选择：**

PDOStatement::bindValue() 方法把一个值绑定到一个参数。

PDOStatement::bindParam() 不同于 PDOStatement::bindValue()，绑定变量作为引用被绑定，并只在 PDOStatement::execute() 被调用的时候才取其值。

这里我们选择 PDOStatement::bindValue() 方法，因为参数绑定过程和 execute 执行过程可能被封装到不同的方法中，我们需要简单的值传递去传值而不是引用。

### 为基类编写参数绑定方法

先回顾下基类现在执行 sql 的过程：在 get()、row() 这些取结果的方法中，先执行构造 sql 的方法，再执行 _execute() 方法执行 sql。那么也就是说，我们只要在这两个方法中间进行参数的绑定即可。

当然，并非只有 where 子句需要参数绑定，having 子句也涉及到条件等式的编写。为了程序结构的灵活和清晰，我们在基类新加一个 \_bind\_params 属性，键值数组类型，用来存储占位符和其绑定数据的映射。这样，我们只需：

- 构造 where (having) 子句字符串时生成占位符，并将占位符和其绑定数据存入 \_bind\_params 数组中。
- 等待构造 sql 完毕后执行参数绑定方法。
- 最后执行 _execute() 方法执行 sql。

talk is cheap， just show code：

基类添加 \_bind\_params 属性，用于存储占位符和其绑定数据的映射：

```php
protected $_bind_params = [];
```

添加参数绑定方法：

```php
protected function _bindParams()
{
    if(is_array($this->_bind_params)) {
        // 将占位符绑定数据数组迭代绑定
        foreach ($this->_bind_params as $plh => $param) {
            // 默认为字符串类型
            $data_type = PDO::PARAM_STR;
            // 如果绑定数据为数字
            if(is_numeric($param)) {
                $data_type = PDO::PARAM_INT;
            }
            // 如果绑定数据为 null
            if(is_null($param)) {
                $data_type = PDO::PARAM_NULL;
            }
            // 如果绑定数据为 Boolean
            if(is_bool($param)) {
                $data_type = PDO::PARAM_BOOL;
            }
            // 执行绑定
            $this->_pdoSt->bindValue($plh, $param, $data_type);
        }
    }
}

```

修改 _execute() 方法：

```php
protected function _execute()
{
    try {
        $this->_pdoSt = $this->_pdo->prepare($this->_prepare_sql);
        // 进行参数绑定
        $this->_bindParams();
        $this->_pdoSt->execute();
        $this->_reset();
    } catch (PDOException $e) {
        if($this->_isTimeout($e)) { 

            $this->_closeConnection();
            $this->_connect();
            
            try {
                $this->_pdoSt = $this->_pdo->prepare($this->_prepare_sql);
                // 进行参数绑定
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

### where 方法

现在我们开始开发条件查询的主要对外方法 where() 

where 方法应该包含如下的功能：

- 构造 where 子句字符串，支持链式访问
- 生成占位符，保存占位符和绑定数据的映射
- 支持几种方便的条件模式 (单条件、多条件、是否为 NULL、比较运算符的判断)


**1、构造 where 子句字符串**

我们希望 where() 方法支持多种模式的条件，如 where('name', 'jack')、where('age', '<', '30')、where(['name' => 'jack', 'age' => 18])。可以观察得到，方法的参数是变动的，那么我们可以使用可变参数，用 func_num_args() 函数得到传入参数的数量进行模式判断，用 func_get_args() 函数得到传入的参数，这样就可以实现对多个模式的支持。(当然使用可变参数也是优缺点的，使用可变参数，接口 ConnectorInterface 中就无法限制该方法参数的个数和类型了，这里要根据个人需求取舍)

基类增加 where() 方法：

```php
public function where()
{
    // 多个条件的默认连接符为 AND，即与的关系
    $operator = 'AND';
    // 在一次查询构造过程中，是否是第一次调用此方法？
    // 在链式访问中有效
    if($this->_where_str == '') { // 第一次调用，where 子句需要 WHERE 关键字 
        $this->_where_str = ' WHERE ';
    } else { // 非初次访问，用连接符拼接上一个条件
        $this->_where_str .= ' '.$operator.' ';
    }
    // 获取参数数量和参数数组
    $args_num = func_num_args();
    $params = func_get_args();

    // argurment mode
    switch ($args_num) {
        case 1: // 只有一个参数：传入数组，多条件模式，如 a = b AND c = d ... 默认 AND 连接
            ...
            break;
        case 2: // 两个参数：单条件模式
            ...
            break;
        case 3: // 三个参数：比较运算符判断模式
            ...
            break;
        }
    // 实现链式操作，返回当前实例
    return $this;
}


```

**2、生成占位符，保存占位符和绑定数据的映射**

对于 :name 形式的占位符，只要保证占位符唯一即可。但是如何保证其唯一性呢？占位符不光是在 where 子句中出现，还在 where in 、where bewteen 这些需要参数绑定的子句中出现。那么按照功能和数据拼接字符串来生产吗？但是问题又来了，对于 where，有 where 和 or where 子句，where in 有 where in、where not in、or where in、or where not in 等组合，两个要绑定的参数也可能拥有相同的值，用功能加数据拼接字符串来生成占位符很复杂，而且因为和方法、参数本身的依赖度高，没法独立出来，程序的可维护性也不行。

当然使用 ? 占位符不用考虑那么多 (知名框架 laravel 的查询构造器就是这么做的，然而源码太多，原谅我没时间看完)，但是使用 ? 占位符对参数绑定的顺序有很大的要求。对于目前我的程序结构来说，_bindParams() 方法只是一股脑的迭代绑定参数，并不能分清楚各个参数的顺序，容易导致绑错参数的状况。

那么，就说说我最后决定的做法：使用唯一 ID 生成。

首先将生产占位符的过程独立出来，作为一个独立方法，这样即使有了更好的生成方法后，也不用更改其他程序。

使用 PHP 的 [uniqid()](http://php.net/manual/zh/function.uniqid.php) 函数生产一个唯一字符串，加前缀和熵值 (提高唯一性)，用 MD5 签一下名 (生成 :name 占位符可接受的字符)。

代码如下：

```php
// 生成占位符的方法
// 考虑次方法和类实例本身无关，所以写为 static 方法提高效率
protected static function _getPlh() // get placeholder
{
    return ':'.md5(uniqid(mt_rand(), TRUE));
}

```

**性能相关的思考：**

Q：uniqid()、mt_rand()、md5() 这些函数的性能如何？会不会拖慢查询构造器的速度？

A：这几个函数性能不怎么样，但是就像前言那一篇所说的，这些函数的使用的影响是否超过了系统性能的平衡点？对于此查询构造器程序来讲，并不是频繁使用这些函数做密集运算，系统的瓶颈还是在和数据库交互的网络 IO 上，所以，这些函数是可以使用的。

注：我在一些测试机上测试过拼接字符串做占位符和随机 ID 做占位符的压测 AB 对比，并没有什么性能差距。当然可能在一个处理速度超快的服务器、数据库组合上能看到差距，如果各位有更好的方法欢迎提出，对我的方法的不足也欢迎指正。

Q：能保证生产的占位符是唯一的吗？

A：如果在多线程的环境下，存在数据竞争，PHP 又没有好用的线程库进行数据加锁，会出现重复的状况。但是在其他环境下不会。传统 web 环境下每次执行随着一次 HTTP 请求结束，PHP-FPM 是多进程模型，处理请求时每个进程中的数据独立，互不影响。而在 workerman 这个常驻内存的框架里，多任务也是一个任务开启一个进程，数据相互独立，每个进程中使用 epoll (linux)、select (windows) 来处理并发，并不会出现并行和数据竞争的状况。所以说只要没有多线程的需求，则占位符不会重复。


**3、支持几种方便的条件模式**

OK，占位符的生成方式搞定，那么我们开始在 where() 方法中使用吧。

```php

public function where()
{
    // 多个条件的默认连接符为 AND，即与的关系
    $operator = 'AND';
    // 在一次查询构造过程中，是否是第一次调用此方法？
    // 在链式访问中有效
    if($this->_where_str == '') { // 第一次调用，where 子句需要 WHERE 关键字 
        $this->_where_str = ' WHERE ';
    } else { // 非初次访问，用连接符拼接上一个条件
        $this->_where_str .= ' '.$operator.' ';
    }
    // 获取参数数量和参数数组
    $args_num = func_num_args();
    $params = func_get_args();

    // 判断传入的参数数量是否合法
    if( ! $args_num || $args_num > 3) {
        throw new \InvalidArgumentException("Error number of parameters");
    }

    // argurment mode
    switch ($args_num) {

        // 只有一个参数：传入数组，多条件模式，如 a = b AND c = d ... 默认 AND 连接
        case 1: 
            if( ! is_array($params[0])) { // 传入非法参数，抛出异常提醒
                throw new \InvalidArgumentException($params[0].' should be Array');
            }
            // 遍历构造多条件 where 子句
            $this->_where_str .= '(';
            foreach ($params[0] as $field => $value) {
                $plh = self::_getPlh(); // 生成占位符
                $this->_where_str .= ' '.$field.' = '.$plh.' AND'; // 将占位符添加到子句字符串中
                $this->_bind_params[$plh] = $value; // 保存占位符和待绑定数据
            }
            // 清除最后一个 AND 连接符
            $this->_where_str = substr($this->_where_str, 0, strrpos($this->_where_str, 'AND'));
            $this->_where_str .= ')';
            break;

        // 两个参数：单条件模式
        case 2: 
            if(is_null($params[1])) { // 如果数据为 null，则使用 IS NULL 语法
                $this->_where_str .= ' '.$params[0].' IS NULL ';
            } else {
                $plh = self::_getPlh(); // 生成占位符
                $this->_where_str .= ' '.$params[0].' = '.$plh.' '; // 将占位符添加到子句字符串中
                $this->_bind_params[$plh] = $params[1]; // 保存占位符和待绑定数据
            }
            break;

        // 三个参数：比较运算符判断模式
        case 3: 
            // 判断使用的比较运算符是否合法 (各数据库的运算符支持并不相同)
            if( ! in_array(strtolower($params[1]), $this->_operators)) {
                throw new \InvalidArgumentException('Confusing Symbol '.$params[1]);
            }
            $plh = self::_getPlh(); // 生成占位符
            $this->_where_str .= ' '.$params[0].' '.$params[1].' '.$plh.' '; // 将占位符添加到子句字符串中
            $this->_bind_params[$plh] = $params[2]; // 保存占位符和待绑定数据
            break;
        }
    // where 子句构造完毕
    return $this;
}

```

关于上述代码这里有几点要提一下：

在多条件模式下，需要判断一下传入的是不是一个数组 (过滤非法参数，方便开发)，多个条件之间的连接符默认是 AND (大部分多个相等条件之间都是以与的形式连接的，如果有 OR 的需求请用 or where 子句)。

单条件模式下需要判断绑定数据是否为 null，如果为 null，则使用 IS NULL 的语法 (SQL 中不能用 < > = 判断 null )。

比较运算符判断模式下，首先要判断一下传入的比较运算符是否非法，这里各个数据库提供的比较运算符是有差异的，所以我们要单独设置一个属性 _operators 保存这些运算符，Mysql、PostgreSql、Sqlite 这些驱动类中进行重写。

Mysql 驱动类中：

```php
// Mysql 提供的比较运算符
protected $_operators = [
    '=', '<', '>', '<=', '>=', '<>', '!=', '<=>',
    'like', 'not like', 'like binary', 'rlike', 'regexp', 'not regexp',
    '&', '|', '^', '<<', '>>',
];

```

PostgreSql 驱动类中：

```php
// PostgreSql 提供的比较运算符
protected $_operators = [
    '=', '<', '>', '<=', '>=', '<>', '!=',
    'like', 'not like', 'ilike', 'similar to', 'not similar to',
    '&', '|', '#', '<<', '>>',
];

```

Sqlite 驱动类中：

```php
// Sqlite 提供的比较运算符
protected $_operators = [
    '=', '<', '>', '<=', '>=', '<>', '!=',
    'like', 'not like', 'ilike',
    '&', '|', '<<', '>>',
];

```

### 测试

打开 test/test.php，修改代码：

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

// 单条件模式测试
$results = $driver->table('test_table')
                  ->select('*')
                  ->where('username', 'jack')
                  ->get();

var_dump($results);

// 链式访问 + 比较运算符判断模式测试
$results = $driver->table('test_table')
                  ->select('*')
                  ->where('username', 'jack')
                  ->where('age', '<', 30)
                  ->get();

var_dump($results);

// 多条件模式测试
$results = $driver->table('test_table')
                  ->select('*')
                  ->where([
                      'username' => 'jack',
                      'age' => 18,
                  ])
                  ->get();

var_dump($results);

```

执行看看，数据是不是如你所想。

## 优化

虽然完成了 where() 方法的编写，但是我们发现 where() 方法中的代码很臃肿，而且后续编写 orWhere()、having() 这些都需要用到条件查询的方法时，很多的代码都是重复的。既然如此，那么就把这部分代码提出来。

基类添加 \_condition\_constructor 方法：

```php

// $args_num 为 where() 传入参数的数量
// $params 为 where() 传入的参数数组
// $construct_str 为要构造的子句的字符串，在 where() 方法中调用会传入 $this->_where_str 
// 因为要改变该子句字符串，所以这里使用引用传递
protected function _condition_constructor($args_num, $params, &$construct_str)
{
    if( ! $args_num || $args_num > 3) {
        throw new \InvalidArgumentException("Error number of parameters");
    }

    switch ($args_num) {
        case 1: 
            if( ! is_array($params[0])) {
                throw new \InvalidArgumentException($params[0].' should be Array');
            }
            $construct_str .= '(';
            foreach ($params[0] as $field => $value) {
                $plh = self::_getPlh();
                $construct_str .= ' '.$field.' = '.$plh.' AND';
                $this->_bind_params[$plh] = $value;
            }
            $construct_str = substr($construct_str, 0, strrpos($construct_str, 'AND'));
            $construct_str .= ')';
            break;
        case 2: 
            if(is_null($params[1])) {
                $construct_str .= ' '.$params[0].' IS NULL ';
            } else {
                $plh = self::_getPlh();
                $construct_str .= ' '.$params[0].' = '.$plh.' ';
                $this->_bind_params[$plh] = $params[1];
            }
            break;
        case 3: 
            if( ! in_array(strtolower($params[1]), $this->_operators)) {
                throw new \InvalidArgumentException('Confusing Symbol '.$params[1]);
            }
            $plh = self::_getPlh();
            $construct_str .= ' '.$params[0].' '.$params[1].' '.$plh.' ';
            $this->_bind_params[$plh] = $params[2];
            break;
    }

}

```

修改后的 where() 方法：

```php
public function where()
{
    // 多个条件的默认连接符为 AND，即与的关系
    $operator = 'AND';
    // 在一次查询构造过程中，是否是第一次调用此方法？
    // 在链式访问中有效
    if($this->_where_str == '') { // 第一次调用，where 子句需要 WHERE 关键字 
        $this->_where_str = ' WHERE ';
    } else { // 非初次访问，用连接符拼接上一个条件
        $this->_where_str .= ' '.$operator.' ';
    }
    // 进行占位符生产、参数绑定、生成子句字符串操作
    $this->_condition_constructor(func_num_args(), func_get_args(), $this->_where_str);

    return $this;
}

```

这样我们就把可以通用的逻辑提出来了，趁热打铁，我们把 orWhere() 方法也添加到基类中。

对于 orWhere() 方法，和 where() 方法的区别只有链式操作进行多条件查询时的连接符不同：

```php
public function orWhere()
{
    $operator = 'OR';
    
    if($this->_where_str == '') {
        $this->_where_str = ' WHERE ';
    } else {
        $this->_where_str .= ' '.$operator.' ';
    }
    
    $this->_condition_constructor(func_num_args(), func_get_args(), $this->_where_str);

    return $this;
}
```

