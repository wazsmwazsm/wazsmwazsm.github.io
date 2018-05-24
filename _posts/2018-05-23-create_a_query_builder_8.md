---
layout: post
title:  "写一个“特殊”的查询构造器 - (八、单元测试、收尾工作)"
date:   2018-05-23 14:23:08
categories: PHP
excerpt: 查询构造器的编写到此结束，这一篇我们说一说 debug、测试相关的问题
---

## debug 模式

对查询构造器进行调试并不难，从其构造 SQL -> 数据绑定 -> SQL 执行的过程中就能发现，要方便调试，只要可以观察以下信息：

- 构造的 SQL 
- 绑定的数据

PDO 提供了一个方便的 debug 方法 [PDOStatement::debugDumpParams()](http://php.net/manual/en/pdostatement.debugdumpparams.php) 来打印 SQL 和绑定的数据。我们就使用它来做 debug 的工作。

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

这样，在任何一个语句构造过程中使用 withDebug() 方法 (在 get()、row() 等取结果的方法调用之前)，就能打印出 debug 的信息。

> 注：因为我在常驻内存模式下使用，所以选择直接打印到 stdout 中，这样可以直接在终端界面上调试。传统 web 模式中可以使用 [Output Control 系列函数](http://php.net/manual/zh/ref.outcontrol.php) 来获取 debug 信息。

## 单元测试

### 单元测试的必要性

**从项目的角度看：**

当项目的规模很小的时候，单元测试没什么用。但是如果是写底层框架或者项目发展到一定的规模时，单元测试对于提高生产力有很明显的贡献。

**从程序设计的角度上看：**

单元测试可以让你更好的拆分程序为最小单元，帮助你更好的解耦。

单元测试的好处是给开发人员的，并不是给机器的。

拿我们编写的查询构造器为例，where()、get() 等方法依赖了很多底层的方法，底层方法之间也有互相的调用。

>**情况一：**你要为一个底层方法添加功能，改完后如何判断是否会影响上层调用呢？把所有调用它的方法都调用一遍看结果吗？不用，只需使用单元测试，确定这个方法的输入输出、可能的运行情况和边界状态，即保证最小单元可用。只要通过单元测试，则这个方法就没有问题 (当然这里的程序结构必须设计合理、测试必须准确有效)。
>
>**情况二：**有一天，你想为你的查询构造器再支持一个新的数据库，这个数据库的驱动类继承自基类。但是你不清楚基类的这些方法是否对新的数据库还有效 (比如 postgresql 中 lastInsertId 的不同)，要把所有方法跑一遍吗？不用，你只需事先把这些通用方法写好单元测试，把驱动类换作新数据库的驱动类执行单元测试即可，跑一遍你就会发现有哪些方法是有问题的。
>
>**情况三：**和情况一类似，当一个方法出 bug 时，你并不能马上定位此 bug 出在此方法还是此方法依赖的方法上。而且当你定位了 bug 并进行修复时，发现其它方法因为修复出现了新的 bug，又是一轮 bug 查找。使用单元测试后，每次有修改后，都跑一遍单元测试，可以很快的发现此次修改对整个程序的影响，为我们节省很多时间。

当然，单元测试中还有 stub、mock 之类的模式可以很好的解决依赖不确定、难重现的问题，这里不做重点，我们就不多说了。

### 使用 PHPUnit

到现在之前，我们都是使用 test/test.php 这个文件写一些测试，这种简单的方式虽然做一些简单测试没有问题，但是完成单元测试就要大费周章了。而 PHP 有著名的单元测试框架 [PHPUnit](http://www.phpunit.cn/)，能很好的完成我们进行测试的需求，所以单元测试这块儿，我们使用 PHPUnit。

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

当然，对于这个单元测试，还是要做一些说明的。

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

首先看 PDODML 和 PDODQL 类，包含了通用的 DQL 和 DML 方法的测试，通过原生 PDO 执行 SQL 得出的结果和查询构造器构造执行得出的结果相比较。

**MysqlPDODMLTest.php、MysqlPDODQLTest.php 等数据库开头测试文件：**

MysqlPDODMLTest 继承自 PDODML，MysqlPDODQLTest 继承自 PDODQL，Pgsql 和 Sqlite 同样道理。

MysqlPDODMLTest、MysqlPDODQLTest 这些测试类中使用 phpunit 的 setUpBeforeClass() 方法和 dbunit 的 getConnection() 方法等创建了一个全局可用的数据库连接，方便测试时对数据库的访问。

**test.xml：**

test.xml 中写好了 dbunit 要求的固定格式的模拟数据，用来测试时自动填充、恢复数据表 (因为 insert、update 等会更改数据表，这也是要用 dbunit 的原因)。

**PDODriverTest.php：**

里面包含了对基类所有方法的测试。这里要说明一下，基类中有很多 protected 方法，我的测试方案是写一个新的类，继承自基类，然后新建 public 方法包裹要测试的 protected 方法，对新建的 public 方法进行测试，即达到了测试 protected 方法的目的。

这个文件中的测试更多是测试各个方法构造的 SQL 字符串是否符合预期，使用的正则匹配断言比较多。

**sql 文件：**

几个数据库测试表的建表 sql。

**本地测试**

想要在你的本地跑这些测试的话，打开 MysqlPDODMLTest.php、MysqlPDODQLTest.php 等数据库开头的文件，把数据库配置中的 username、password、dbname 等改成你自己的即可。


## 集成测试 Travis CI

### 什么是 Travis CI？

Travis CI 提供的是持续集成服务（Continuous Integration，简称 CI）。它绑定 Github 上面的项目，只要有新的代码，就会自动抓取。然后，提供一个运行环境，执行测试，完成构建，还能部署到服务器。【2】

持续集成指的是只要代码有变更，就自动运行构建和测试，反馈运行结果。确保符合预期以后，再将新代码"集成"到主干。【2】

持续集成的好处在于，每次代码的小幅变更，就能看到运行结果，从而不断累积小的变更，而不是在开发周期结束时，一下子合并一大块代码。【2】

### 使用 Travis CI

如果你要将项目推到 Github 上，可以接入 [Travis CI](https://www.travis-ci.org/)。通过编写 .travis.yml 配置文件，可以实现远程运行环境的语言多版本切换、软件安装、脚本执行等操作。

对于查询构造器这个项目，我们可以让其在远程运行环境安装相关数据库软件，执行数据表建立，数据导入，执行单元测试等操作。

我的框架项目 WorkerA 就集成了 Travis CI ，相关配置见 [WorkerF - .travis.yml](https://github.com/wazsmwazsm/WorkerF/blob/master/.travis.yml)，感兴趣的可以了解下。

## 注释

PHP 中对方法的注释一是为了提示，二是为了生成文档。我这里的注释写法是标明功能、参数、返回值和抛出的异常。一个清晰好懂的注释对于项目来说还是很必要的。

例如：
```php
/**
 * get paginate data
 *
 * @param  int $step
 * @param  int $page
 * @return  array
 * @throws  \PDOException
 */
public function paginate($step, $page = NULL)
{
    ...
}
```

## 尾声

一个查询构造器的创建到此结束，希望对大家有用。如果发现文中的书写和思路有错误，或者对此项目有什么好的建议的话，欢迎提出。对文中的解释有不解的地方，也欢迎提问。

查询构造器的完整代码：[WorkerF - DB](https://github.com/wazsmwazsm/WorkerF/tree/master/src/WorkerF/DB/Drivers)

查询构造器的单元测试完整代码：[WorkerF - tests - DB](https://github.com/wazsmwazsm/WorkerF/tree/master/tests/DB)。

## 参考
【1】[PHPUnit Doc](http://www.phpunit.cn/)

【2】[持续集成服务 Travis CI 教程 - 阮一峰](http://www.ruanyifeng.com/blog/2017/12/travis_ci_tutorial.html)
