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

### 使用 phpunit

到现在之前，我们都是使用 test/test.php 这个文件写一些测试，去执行我们的查询构造器方法是否正确。但是，这样简单的测试方法并不能保证整个查询构造器正确无误，
