---
layout: post
title:  "写一个“特殊”的查询构造器 - (七、DML 语句、事务)"
date:   2018-05-22 11:40:16
categories: PHP
excerpt: 查询语句 (DQL) 的构造功能开发完毕，我们再给查询构造器增加一些对 DML (Data Manipulation Language) 语句的支持，如简单的 insert、update、delete 操作。
---

查询语句 (DQL) 的构造功能开发完毕，我们再给查询构造器增加一些对 DML (Data Manipulation Language) 语句的支持，如简单的 insert、update、delete 操作。

## insert

我们先回顾下 PDO 原生的 insert 操作怎么进行：

```php
// 预编译
$pdoSt = $pdo->prepare("INSERT INTO test_table ('username', 'age') VALUES (:username, :age);");
// 绑定参数
$pdoSt->bindValue(':username', 'Jack', PDO::PARAM_STR)
$pdoSt->bindValue(':age', 18, PDO::PARAM_INT)
// 执行
$pdoSt->execute(); 
// 获取执行数据
$count = $pdoSt->rowCount(); // 返回被影响的行数
$lastId = $pdo->lastInsertId(); // 获取最后插入行的 ID
```

**数据插入**

和查询语句的执行过程并没有太大差别，只是语法不同。回想第二篇，我们新建了 _buildQuery() 方法去构造最终的 SQL，对于 insert，我们在基类新建 _buildInsert() 方法：

```php
protected function _buildInsert()
{
    $this->_prepare_sql = 'INSERT INTO '.$this->_table.$this->_insert_str;
}
```

基类添加 \_insert\_str 属性：

```php
protected $_insert_str = '';
```

修改 _reset() 函数，将 \_insert\_str 属性的初始化过程添加进去：

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
    $this->_insert_str = ''; // 重置 insert 语句
    $this->_bind_params = [];
}
```

基类中添加 insert() 方法：

```php
public function insert(array $data)
{
    // 构建字符串
    $field_str = '';
    $value_str = '';
    foreach ($data as $key => $value) {
        $field_str .= ' '.self::_wrapRow($key).',';
        $plh = self::_getPlh(); // 生产占位符
        $this->_bind_params[$plh] = $value; //保存绑定数据
        $value_str .= ' '.$plh.',';
    }
    // 清除右侧多余的逗号
    $field_str = rtrim($field_str, ',');
    $value_str = rtrim($value_str, ',');

    $this->_insert_str = ' ('.$field_str.') VALUES ('.$value_str.') ';
    // 构造 insert 语句
    $this->_buildInsert();
    // 执行
    $this->_execute();
    // 获取影响的行数
    return $this->_pdoSt->rowCount();
}
```

对上述代码，我们申明了 insert() 方法的参数是一个键值数组，用来传入要插入的字段、值映射。默认返回被影响的行数 (比较通用)。

**测试**

试着插入一条数据吧：
```php

$insert_data = [
    'username' => 'jack',
    'age'      => 18,
];

$results = $driver->table('test_table')->insert($insert_data);
```


**获取最后插入行的 ID**

当一个表中有自增 id 且为主键时，这个 id 可以被看作区分数据的唯一标识。而在插入一条数据后获取这条新增数据的 id 也是常见的业务需求。

PDO 提供了一个简单的获取最后插入行的 ID 的方法 [PDO::lastInsertId()](http://php.net/manual/en/pdo.lastinsertid.php) 供我们使用。

基类添加 insertGetLastId() 方法：

```php
public function insertGetLastId(array $data)
{
    $this->insert($data);

    return $this->_pdo->lastInsertId();
}
```

测试：
```php
$insert_data = [
    'username' => 'jack',
    'age'      => 18,
];

$lastId = $driver->table('test_table')->insertGetLastId($insert_data);
```

**个体差异**

然而，上述的 insertGetLastId() 方法在 PostgreSql 中并不奏效。PostgreSql 中，使用 PDO::lastInsertId() 获取结果需要传入正确的自增序列名 (PostgreSQL 中创建表时，如果使用 serial 类型，默认生成的自增序列名为：表名 + _ + 字段名 + _ + seq)。【1】

但是这个方式并不好用，因为访问 insertGetLastId() 方法时必须手动传入这个序列名称，这样 insertGetLastId() 方法对底层的依赖严重，比如当底层驱动从 postgresql 换到 mysql 时，需要更改上层应用。而我们希望无论是 mysql 还是 postgresql，上层应用调用 insertGetLastId() 方法时是无差别的，即底层对上层透明。

为了解决这个问题，就需要用到 postgresql 的 returning 语法了。postgresql 中 insert、update 和 delete 操作都有一个可选的 returning 子句，可以指定最后执行的字段进行返回，返回的数据可以像 select 一样取结果。【2】

对于我们返回最后插入行的 ID 的需求，只需 returning id 就好。

当然，基类的 insertGetLastId() 方法对于 postgresql 而言已经无效了，我们在 Pgsql 驱动类中重写 insertGetLastId() 方法：

```php
public function insertGetLastId(array $data)
{
    // 构建语句字符串、绑定数据
    $field_str = '';
    $value_str = '';
    foreach ($data as $key => $value) {
        $field_str .= ' '.self::_wrapRow($key).',';
        $plh = self::_getPlh();
        $this->_bind_params[$plh] = $value;
        $value_str .= ' '.$plh.',';
    }

    $field_str = rtrim($field_str, ',');
    $value_str = rtrim($value_str, ',');
    // 使用 returning 子句返回 id
    $this->_insert_str = ' ('.$field_str.') VALUES ('.$value_str.') RETURNING id ';
    // execute
    $this->_buildInsert();
    $this->_execute();
    // 使用 returning 子句后，可以像使用 SELECT 一样获取一个 returning 指定字段的结果集。 
    $result = $this->_pdoSt->fetch(PDO::FETCH_ASSOC);
    // 返回 id
    return $result['id'];
}
```

OK，我们再来测试看看，是不是就好用了呢？

## update

做完 insert，update 就很简单了，不同的是为了防止全局更新的失误发生，update 构造时强行要求使用 where 子句。

同样的，添加 \_update\_str 属性，修改 _reset() 函数：

```php
protected $_update_str = '';

...

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

构造 update 语句的方法：

```php
protected function _buildUpdate()
{
    $this->_prepare_sql = 'UPDATE '.$this->_table.$this->_update_str.$this->_where_str;
}
```

基类中添加 update() 方法：

```php
public function update(array $data)
{
    // 检测有没有设置 where 子句
    if(empty($this->_where_str)) {
        throw new \InvalidArgumentException("Need where condition");
    }
    // 构建语句、参数绑定
    $this->_update_str = ' SET ';
    foreach ($data as $key => $value) {
        $plh = self::_getPlh();
        $this->_bind_params[$plh] = $value;
        $this->_update_str .= ' '.self::_wrapRow($key).' = '.$plh.',';
    }

    $this->_update_str = rtrim($this->_update_str, ',');

    $this->_buildUpdate();
    $this->_execute();
    // 返回影响的行数
    return $this->_pdoSt->rowCount();
}
```

更新数据示例：

```php
$update_data = [
    'username' => 'lucy',
    'age'      => 22,
];

$results = $driver->table('test_table')
            ->where('username', 'jack')
            ->update($update_data);
```

## delete

相比 insert、update，delete 语句更为简单，只需 where 子句即可。和 update 一样，需要防止误操作删除所有数据。

构造 delete 语句的方法：

```php
protected function _buildDelete()
{
    $this->_prepare_sql = 'DELETE FROM '.$this->_table.$this->_where_str;
}
```

基类中添加 delete() 方法：

```php
public function delete()
{
    // 检测有没有设置 where 子句
    if(empty($this->_where_str)) {
        throw new \InvalidArgumentException("Need where condition");
    }

    $this->_buildDelete();
    $this->_execute();

    return $this->_pdoSt->rowCount();
}
```

删除数据示例：

```php
$results = $driver->table('test_table')
            ->where('age', 18)
            ->delete();
```

## 事务

既然有了 DML 操作，那么就少不了事务。对于事务，我们可以直接使用 PDO 提供的 [PDO::beginTransaction()](http://php.net/manual/en/pdo.begintransaction.php)、[PDO::commit()](http://php.net/manual/en/pdo.commit.php)、[PDO::rollBack()](http://php.net/manual/en/pdo.rollback.php)、[PDO::inTransaction()](http://php.net/manual/en/pdo.intransaction.php) 方法来实现。

基类添加 beginTrans() 方法：

```php
// 开始事务
public function beginTrans()
{
    try {
        return $this->_pdo->beginTransaction();
    } catch (PDOException $e) {
        // 断线重连
        if ($this->_isTimeout($e)) {

            $this->_closeConnection();
            $this->_connect();

            try {
                return $this->_pdo->beginTransaction();
            } catch (PDOException $e) {
                throw $e;
            }

        } else {
            throw $e;
        }
    }
}
```

> 注：因为 PDO::beginTransaction() 也是和 PDO::prepare() 一样会连接数据库的方法，所以需要做断线重连的操作。

commitTrans() 方法：

```php
// 提交事务
public function commitTrans()
{
    return $this->_pdo->commit(); 
}
```
rollBackTrans() 方法：

```php
// 回滚事务
public function rollBackTrans()
{
    if ($this->_pdo->inTransaction()) {
        // 如果已经开始了事务，则运行回滚操作
        return $this->_pdo->rollBack();
    }
}
```

事务使用示例：

```php
// 注册事务
$driver->beginTrans();
$results = $driver->table('test_table')
            ->where('age', 18)
            ->delete();
$driver->commitTrans(); // 确认删除

// 回滚事务
$driver->beginTrans();
$results = $driver->table('test_table')
            ->where('age', 18)
            ->delete();
$driver->rollBackTrans(); // 撤销删除
```

## 参考

【1】[PHP Manual - PDO::lastInsertId](http://php.net/manual/en/pdo.lastinsertid.php)

【2】[PostgreSQL Documentation - Returning Data From Modified Rows](https://www.postgresql.org/docs/10/static/dml-returning.html)
