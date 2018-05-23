---
layout: post
title:  "写一个“特殊”的查询构造器 - (八、单元测试、收尾工作)"
date:   2018-05-23 14:23:08
categories: PHP
excerpt: 
---

## debug 模式

对于查询构造器的调试不难，从其构造 SQL 到数据绑定执行的过程中就能发现，要方便调试，只要可以观察以下数据：

- 构造的 SQL 
- 绑定的数据

PDO 提供了一个简短的 debug 方法 [PDOStatement::debugDumpParams()](http://php.net/manual/en/pdostatement.debugdumpparams.php) 来打印 SQL 和绑定的数据。我们就使用它来做 debug 的工作。

在基类添加 _debug 属性和 withDebug() 方法：

```php
protected $_debug = FALSE;

...

public function withDebug()
{
    $this->_debug = TRUE;
    // 方便链式调用，返回当前实例
    return $this;
}
```

修改 _execute() 方法：

```php
protected function _execute()
{
    try {
        $this->_wrapPrepareSql();
        $this->_pdoSt = $this->_pdo->prepare($this->_prepare_sql);
        $this->_bindParams();
        $this->_pdoSt->execute();
        $this->_reset();  
        // 如果是 debug 模式，则打印相关信息
        if($this->_debug) {
            $this->_pdoSt->debugDumpParams(); // 打印 debug 信息
            $this->_debug = FALSE; // debug 只在当此访问有效，打印完就关闭
        }
    } catch (PDOException $e) {
        // when time out, reconnect
        if($this->_isTimeout($e)) {
            $this->_closeConnection();
            $this->_connect();
            // retry
            try {
                $this->_wrapPrepareSql();
                $this->_pdoSt = $this->_pdo->prepare($this->_prepare_sql);
                $this->_bindParams();
                $this->_pdoSt->execute();
                $this->_reset();
                // 如果是 debug 模式，则打印相关信息
                if($this->_debug) {
                    $this->_pdoSt->debugDumpParams(); // 打印 debug 信息
                    $this->_debug = FALSE; // debug 只在当此访问有效，打印完就关闭
                }
            } catch (PDOException $e) {
                throw $e;
            }

        } else {
            throw $e;
        }
    }

}
```

这样，在任何一个语句构造过程中使用 withDebug() 方法 (get()、row() 等取结果的方法之前)，就能打印出 debug 的信息。

> 注：因为我在常驻内存模式下，所以选择直接打印到 stdout 中。传统 web 模式中可以使用 [Output Control 系列函数](http://php.net/manual/zh/ref.outcontrol.php) 来获取 debug 信息。

## 单元测试

### 单元测试的必要性

从项目的角度看：

当项目的规模很小的时候，单元测试没什么用。但是如果是写底层框架或者项目发展到一定的规模时，单元测试对于提高生产力有很明显的贡献。

从程序设计的角度上看：单元测试可以让你更好的拆分程序为最小单元，帮助你更好的解耦。

单元测试的好处是给开发人员的，并不是给机器的。

拿我们编写的查询构造器为例，where()、get() 等方法依赖了很多底层的方法，底层方法之间也有互相的调用。

情况一：你要为一个底层方法添加功能，改完后如何判断是否会影响上层调用呢？把所有调用它的方法都调用一遍看结果吗？不用，只需使用单元测试，确定这个方法的输入输出、可能的运行情况和边界状态，即保证最小单元可用。只要通过单元测试，则这个方法就没有问题 (当然这里的程序结构必须设计合理、测试必须准确有效)。

情况二：有一天，你想为你的查询构造器再支持一个新的数据库，这个数据库的驱动类继承自基类。但是你不清楚基类的这些方法是否对新的数据库还有效 (比如 postgresql 的 lastInsertId 的不同)，要把所有方法一一跑一遍吗？不用，你只需使用一个可以自动化单元测试的框架，事先对这些通用方法写好单元测试，把驱动类换作新数据库的驱动类执行单元测试即可，跑一遍你就会发现有哪些方法是有问题的。

情况三：和情况一类似，当一个方法出 bug 时，你并不能马上定位此 bug 出在此方法还是此方法依赖的方法上。而且当你定位了 bug 并进行修复时，发现其它方法因为新的修改出了 bug，又是一轮 bug 查找。使用单元测试后，每次有修改后，都跑一遍单元测试，可以很快的发现此次修改对整个程序的影响，为我们节省很多时间。

当然，单元测试中还有 stub、mock 之类的模式可以很好的解决依赖不确定、难重现的问题，这里不做重点，我们就不多说了。

### 使用 PHPUnit

到现在之前，我们都是使用 test/test.php 这个文件写一些测试，这种简单的方式虽然做一些简单测试没有问题，但是完成单元测试就要大费周章了。而 PHP 有著名的单元测试框架 PHPUnit，能很好的完成我们进行测试的需求，所以单元测试这块儿，我们使用 PHPUnit。

**安装 PHPUnit**

PHPUnit 的安装很简单，在项目中执行：

```shell
composer require "phpunit/phpunit" "~4.0"
composer require "phpunit/dbunit" "~2.0" 
```

> 注：我们的测试需要连接数据库，所以要安装 dbunit

现在在项目目录下的 test 文件夹中新建以 Test.php 结尾的测试文件，命令行运行 phpunit 即可运行测试。【1】

### 单元测试的编写

单元测试的代码简单、代码量大，我就不在这里展示了，所有的测试代码见 [WorkerF - tests - DB](https://github.com/wazsmwazsm/WorkerF/tree/master/tests/DB)。

当然，对于这个单元测试，还是要做一些说明。

**单元测试的结构：**

```
项目目录/
    test/
        PDODML.php
        PDODQL.php
        MysqlPDODMLTest.php
        MysqlPDODQLTest.php
        PgsqlPDODMLTest.php
        PgsqlPDODQLTest.php
        SqlitePDODMLTest.php
        SqlitePDODQLTest.php
        PDODriverTest.php
        test.xml
        testMysql.sql
        testPgsql.sql
        testSqlite.sql
```

**PDODML.php 和 PDODQL.php 文件：**

首先看 PDODML 和 PDODQL 类，这里都是通用的 DQL 和 DML 方法的测试，通过原生 PDO 执行 SQL 得出的结果和查询构造器构造执行得出的结果相比较。

**MysqlPDODMLTest.php、MysqlPDODQLTest.php 等数据库开头测试文件：**

MysqlPDODMLTest 继承自 PDODML，MysqlPDODQLTest 继承自 PDODQL，Pgsql 和 Sqlite 同样道理。

MysqlPDODMLTest、MysqlPDODQLTest 这些测试类中使用 phpunit 的 setUpBeforeClass() 方法和 dbunit 的 getConnection() 方法等创建了一个全局可用的数据库连接，方便测试时对数据库的访问。

想要在你的本地跑这些测试的话，把这些文件中数据库配置中的 username、password、dbname 等改成你自己的即可。

**test.xml：**

test.xml 中写好了 dbunit 要求的固定格式的模拟数据，用来测试时自动填充、恢复数据表 (因为 insert、update 等会更改数据表，这也是要用 dbunit 的原因)。

**PDODriverTest.php：**

里面包含了对基类所有方法的测试。这里要说明一下，基类中有很多 protected 方法，我的测试方案是写一个新的类，继承自基类，然后新建 public 方法包裹要测试的 protected 方法，对新建的 public 方法进行测试，即达到了测试 protected 方法的目的。

这个文件中的测试更多是测试各个方法构造的 SQL 字符串是否符合预期，使用的正则匹配断言比较多。

**sql 文件：**

这是用来测试的数据表的几个数据库的建表 sql。


## 集成测试 Travis CI


## 注释规范


## 尾声
