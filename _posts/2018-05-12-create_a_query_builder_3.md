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

占位符选择：

? 占位符必须按照顺序去绑定，而 :name 占位符只要占位符和数据的映射关系确定，绑定的数据就不会出错。所以我们选择 :name 占位符。

绑定方法的选择：

PDOStatement::bindValue() 方法把一个值绑定到一个参数。

PDOStatement::bindParam() 不同于 PDOStatement::bindValue()，绑定变量作为引用被绑定，并只在 PDOStatement::execute() 被调用的时候才取其值。

这里我们选择 PDOStatement::bindValue() 方法，因为参数绑定过程和 execute 执行过程可能被封装到不同的方法中，我们需要简单的值传递去传值而不是引用。
