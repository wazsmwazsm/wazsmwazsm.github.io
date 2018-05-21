---
layout: post
title:  "写一个“特殊”的查询构造器 - (三、条件查询)"
date:   2018-05-12 21:12:22
categories: PHP
excerpt: 如果单单是执行 SELECT * FROM test_table; 这样的语句，使用原生扩展就好了，使用查询构造器就是杀鸡用牛刀。当然，在实际的业务需求中，大部分的 SQL 都没这么简单，有各种条件查询、分组、排序、连表等操作，尤其是条件查询，占到了查询业务的大多数。
---

## 构造 where 条件

如果单单是执行 `SELECT * FROM test_table;` 这样的语句，使用原生扩展就好了，使用查询构造器就是杀鸡用牛刀。当然，在实际的业务需求中，大部分的 SQL 都没这么简单，有各种条件查询、分组、排序、连表等操作，尤其是条件查询，占到了查询业务的大多数。

这一篇，我们来讲讲如何使用查询构造器进行条件查询。

### 条件查询的核心：参数绑定

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

PDOStatement::bindParam() 不同于 PDOStatement::bindValue()，绑定变量作为引用被传入，并只在 PDOStatement::execute() 被调用的时候才取值。

这里我们选择 PDOStatement::bindValue() 方法，因为参数绑定过程和 execute 执行过程可能被封装到不同的方法中，我们需要简单的传值传递而不是引用传递。

### 为基类编写参数绑定方法

先回顾下基类现在执行 sql 的过程：在 get()、row() 这些取结果的方法中，先执行构造 sql 的方法，再执行 _execute() 方法执行 sql。那么也就是说，我们只要在这两个方法中间进行参数的绑定即可。

当然，并非只有 where 子句需要参数绑定，having 子句、where in 子句等也涉及到参数的绑定。为了程序结构的灵活和清晰，我们在基类新加一个 \_bind\_params 属性，键值数组类型，用来存储占位符和其绑定数据的映射。这样，我们只需：

- 构造 where (having 等) 子句字符串时生成占位符，并将占位符和其绑定数据存入 \_bind\_params 数组中。
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
- 支持几种常用的条件模式 (单条件、多条件、是否为 NULL、比较运算符的判断)


**1、构造 where 子句字符串**

我们希望 where() 方法支持多种条件模式，如 `where('name', 'jack')、where('age', '<', '30')、where(['name' => 'jack', 'age' => 18])`。可以观察得到，方法的参数是变动的，那么我们可以使用可变参数，用 func_num_args() 函数得到传入参数的数量进行模式判断，用 func_get_args() 函数得到传入的参数，这样就可以实现对多个模式的支持。(当然使用可变参数也是有缺点的，使用可变参数，接口 ConnectorInterface 中就无法限制该方法参数的个数和类型了，这里要根据个人需求取舍)

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

对于 :name 形式的占位符，只要保证占位符唯一即可。但是如何保证其唯一性呢？占位符不光是在 where 子句中出现，还在 where in 、where between 这些需要参数绑定的子句中出现。那么按照功能和绑定数据拼接字符串来生成吗？但是问题又来了，对于 where，有 where 和 or where 子句，where in 有 where in、where not in、or where in、or where not in 等组合，多个要绑定的参数也可能拥有相同的值，用功能加绑定数据拼接字符串来生成占位符很复杂，而且因为和方法、参数本身的依赖度高，没法独立出来，程序的可维护性也不行。

当然使用 ? 占位符不用考虑那么多 (知名框架 laravel 的查询构造器就是这么做的，然而源码太多，原谅我没时间看完)，但是使用 ? 占位符对参数绑定的顺序有很大的要求。对于目前我的程序结构来说，_bindParams() 方法只是一股脑的迭代绑定参数，并不能分清楚各个参数的顺序，容易导致绑错数据的状况。

那么，就说说我最后决定的做法：使用唯一 ID 生成。

首先将生成占位符的过程独立出来，作为一个独立方法，这样即使以后有了更好的方案，也不用更改其他程序。

使用 PHP 的 [uniqid()](http://php.net/manual/zh/function.uniqid.php) 函数生成一个唯一字符串，加前缀和熵值 (提高唯一性)，用 MD5 签一下名 (生成 :name 占位符可接受的字符)。

代码如下：

```php
// 生成占位符的方法
// 考虑此方法和类实例本身无关，所以写为 static 方法提高效率
protected static function _getPlh() // get placeholder
{
    return ':'.md5(uniqid(mt_rand(), TRUE));
}
```

**性能相关的思考：**

Q：uniqid()、mt_rand()、md5() 这些函数的性能如何？会不会拖慢查询构造器的速度？

A：这几个函数性能不怎么样，但是就像前言那一篇所说的，这些函数的使用的影响是否超过了系统性能的平衡点？对于此查询构造器程序来讲，并不是频繁使用这些函数做密集运算，系统的瓶颈还是在和数据库交互的网络 IO 上，所以，这些函数是可以使用的。

>注：我在一些测试机上测试过拼接字符串做占位符和随机 ID 做占位符的压测 AB 对比，并没有什么性能差距。当然可能在一个处理速度超快的服务器、数据库组合上能看到差距，如果各位有更好的方法欢迎提出，对我的方法的不足也欢迎指正。

Q：能保证生成的占位符是唯一的吗？

A：如果在多线程的环境下，存在数据竞争，PHP 又没有好用的线程库进行数据加锁，会出现重复的状况。但是在其他环境下不会。传统 web 环境下每次执行随着一次 HTTP 请求结束而结束，解析 PHP 程序的 PHP-FPM、MOD_PHP 是多进程模型，处理请求时每个进程中的数据独立，互不影响。而在 workerman 这个常驻内存的框架里，多任务也是一个任务开启一个进程，数据相互独立，每个进程中使用 epoll (linux)、select (windows) 来处理并发，并不会出现并行和数据竞争的状况。所以说只要没有多线程的需求，则占位符不会重复。


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

在多条件模式下，需要判断一下传入的是不是一个数组 (过滤非法参数，方便开发)，多个条件之间的连接符默认是 AND (大部分多个相等判断条件之间都是以 AND 的形式连接的，如果有 OR 的需求请用链式访问的多个 orWhere() 方法)。

单条件模式下需要判断绑定数据是否为 null，如果为 null，则使用 IS NULL 的语法 (SQL 中不能用 < > = 判断 null )。

比较运算符判断模式下，首先要判断一下传入的比较运算符是否合法，这里各个数据库提供的比较运算符是有差异的，所以我们要单独设置一个属性 _operators 保存这些运算符，Mysql、PostgreSql、Sqlite 这些驱动类中进行重写。

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
    // 进行占位符生成、参数绑定、生成子句字符串操作
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

构造语句 `SELECT * FROM test_table WHERE username = 'jack' OR username = 'mike';`:

```php
$results = $driver->table('test_table')
                  ->select('*')
                  ->where('username', 'jack')
                  ->orWhere('username', 'mike')
                  ->get();
```


## 关键字冲突

熟悉数据库的朋友们应该知道，每种数据库都有一些关键字，一部分是 SQL 语句的关键字，另一部分是数据库自己的关键字。既然有关键字，那么就避免不了用户键入的数据和关键字重名的问题，比如表名和关键字重名、字段名 (别名) 和关键字重名等。

那么如何解决关键字冲突呢？

当然，建表的时候尽量注意命名，不要和关键字冲突是一种方法，但是如果这个表的建立、修改权限不在你手中，你又要访问这个表去拿数据的时候就没招了，所以我们常常要对历史遗留问题进行兼容处理。

各数据库使用了类似转义的做法。Mysql 使用反引号 ` 来包裹字符串避免数据库将这个字符解析为关键字，PostgreSql 和 Sqlite 则是用双引号 " 来做相应的工作。

而在使用查询构造器的过程中，总不能每次由用户手动来写这个符号 (如 where('\`count\`', 12) )，这样更换数据库驱动的时候会影响到上层的代码，可维护性差 (如 mysql 切到 pgsql，需要把所有 ` 改为 " )。所以，为可能出现关键字冲突的地方添加引号应该交给查询构造器底层去做。

既然各个数据库有差异，想必现在大家已经知道该怎么做了，基类添加属性 \_quote\_symbol，Mysql 类中进行重写。

Mysql 驱动类中添加：

```php
// 因为次属性不会改变，使用 static 关键字
protected static $_quote_symbol = '`';
```

PostgreSql 和 Sqlite 同理，这里不单独演示了。

下面我们给基类添加 _quote() 方法，用于给字符串添加引号：

```php
// static 方法
protected static function _quote($word)
{
    return static::$_quote_symbol.$word.static::$_quote_symbol;
}
```

有了这个方法，我们可以简单的防止一个字符串关键字冲突了。但是在实际应用中还远不够。

首先，在书写 SQL 时字段的表述有很多模式
- 别名：username as name (这里不对省略 as 的情况做处理，请不要省略 as 关键字)
- 点号：table_name.field
- SQL 函数做为列：COUNT(field) 

我们必须的对这些常用情形做处理，而不只是直接对这些字符串的两边加引号。

对字符串的匹配处理，那么我们首先想到的是正则表达式。至于正则的性能，还是参考前言所说的性能平衡点，这里每次请求用到正则的次数很少，并没有突破数据库连接和执行的网络 IO 瓶颈，所以可以使用。

在基类添加 _wrapRow 方法，用来处理 SQL 字段的字符串：

```php
// static 方法
protected static function _wrapRow($str)
{
    // 匹配模式
    $alias_pattern = '/([a-zA-Z0-9_\.]+)\s+(AS|as|As)\s+([a-zA-Z0-9_]+)/';
    $alias_replace = self::_quote('$1').' $2 '.self::_quote('$3');
    $prefix_pattern = '/([a-zA-Z0-9_]+\s*)(\.)(\s*[a-zA-Z0-9_]+)/';
    $prefix_replace = self::_quote('$1').'$2'.self::_quote('$3');
    $func_pattern = '/[a-zA-Z0-9_]+\([a-zA-Z0-9_\,\s\`\'\"\*]*\)/';
    // alias mode 别名模式
    if(preg_match($alias_pattern, $str, $alias_match)) {
        // 如果列是 aa.bb as cc 的模式
        if(preg_match($prefix_pattern, $alias_match[1])) {
            $pre_rst = preg_replace($prefix_pattern, $prefix_replace, $alias_match[1]);
            $alias_replace = $pre_rst.' $2 '.self::_quote('$3');
        }
        // 如果列是 aa as bb 的模式
        return preg_replace($alias_pattern, $alias_replace, $str);
    }
    // prefix mode 表.字段 模式
    if(preg_match($prefix_pattern, $str)) {
        return preg_replace($prefix_pattern, $prefix_replace, $str);
    }
    // func mode 函数模式，什么都不做，交给用户去处理
    if(preg_match($func_pattern, $str)) {
        return $str;
    }
    // field mode 简单的字段模式，直接加引号返回
    return self::_quote($str);
}
```

上诉代码有几点要说明：

别名模式是最复杂的，需要判断是 aa as bb 模式还是 aa.bb as cc 模式，匹配替换后的结果是 \`aa\` as \`bb\` 、\`aa\`.\`bb\` as \`cc\` (这里以 mysql 为例)。

函数模式如 count(aa.cc)、max(count) 这种，函数的参数数量不定，模式多变不好匹配，交给用户手动输入原生字符串去处理，而且诸如此类的聚合函数的话，后面的篇幅会增加聚合函数的相关方法去获得结果。

有了 _wrapRow() 方法，我们可以使关键字冲突的处理对上层应用完全透明。

修改 table() 方法：

```php
public function table($table)
{
    // 添加引号
    $this->_table = self::_wrapRow($table);

    return $this;
}
```

修改 select() 方法：

```php
public function select()
{
    $cols = func_get_args();

    if( ! func_num_args() || in_array('*', $cols)) {
        $this->_cols_str = ' * ';
    } else {
        $this->_cols_str = '';
        foreach ($cols as $col) {
            // 添加引号
            $this->_cols_str .= ' '.self::_wrapRow($col).',';
        }
        $this->_cols_str = rtrim($this->_cols_str, ',');
    }

    return $this;
}
```


修该用于条件构造的 \_condition\_constructor() 方法：

```php
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
                // 添加引号
                $construct_str .= ' '.self::_wrapRow($field).' = '.$plh.' AND';
                $this->_bind_params[$plh] = $value;
            }
            
            $construct_str = substr($construct_str, 0, strrpos($construct_str, 'AND'));
            $construct_str .= ')';
            break;
        case 2: 
            if(is_null($params[1])) {
                // 添加引号
                $construct_str .= ' '.self::_wrapRow($params[0]).' IS NULL ';
            } else {
                $plh = self::_getPlh();
                // 添加引号
                $construct_str .= ' '.self::_wrapRow($params[0]).' = '.$plh.' ';
                $this->_bind_params[$plh] = $params[1];
            }
            break;
        case 3: 
            if( ! in_array(strtolower($params[1]), $this->_operators)) {
                throw new \InvalidArgumentException('Confusing Symbol '.$params[1]);
            }
            $plh = self::_getPlh();
            // 添加引号
            $construct_str .= ' '.self::_wrapRow($params[0]).' '.$params[1].' '.$plh.' ';
            $this->_bind_params[$plh] = $params[2];
            break;
    }

}
```

现在我们给要查的数据表中添加一个名为 group 的字段，构造一下 `SELECT * FROM test_table where group = 'test';` 这个语句，看是否会报错呢？

```php
$results = $driver->table('test_table')
    ->select('*')
    ->where('group', 'test')
    ->get();
```

Just do it!

