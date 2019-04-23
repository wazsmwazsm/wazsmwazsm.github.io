---
layout: post
title:  "写一个“特殊”的查询构造器 - (前言)"
date:   2018-05-06 17:22:01
categories: PHP
excerpt: 查询构造器 (query builder)，顾名思义，它的目的就是以简便的形式构造、执行 SQL，为查询数据库的业务提供了方便好用的接口，一些知名的 web 框架如 PHP 的 Laravel、CodeIgniter、ThinkPHP 等都提供了好用的查询构造器。

---
## 更新

此项目已经从 [WorkerA](https://github.com/wazsmwazsm/WorkerA) 中拆分为独立项目，完整代码请到 [wazsmwazsm/DB](https://github.com/wazsmwazsm/DB) 中查看。
## 文章目录

[写一个“特殊”的查询构造器 - (前言)]({{ site.baseurl }}/php/2018/05/06/create_a_query_builder_0.html)

[写一个“特殊”的查询构造器 - (一、程序结构，基础封装)]({{ site.baseurl }}/php/2018/05/07/create_a_query_builder_1.html)

[写一个“特殊”的查询构造器 - (二、第一条语句)]({{ site.baseurl }}/php/2018/05/09/create_a_query_builder_2.html)

[写一个“特殊”的查询构造器 - (三、条件查询)]({{ site.baseurl }}/php/2018/05/12/create_a_query_builder_3.html)

[写一个“特殊”的查询构造器 - (四、条件查询：复杂条件)]({{ site.baseurl }}/php/2018/05/16/create_a_query_builder_4.html)

[写一个“特殊”的查询构造器 - (五、聚合函数、分组、排序、分页)]({{ site.baseurl }}/php/2018/05/20/create_a_query_builder_5.html)

[写一个“特殊”的查询构造器 - (六、关联)]({{ site.baseurl }}/php/2018/05/21/create_a_query_builder_6.html)

[写一个“特殊”的查询构造器 - (七、DML 语句、事务)]({{ site.baseurl }}/php/2018/05/23/create_a_query_builder_7.html)

[写一个“特殊”的查询构造器 - (八、单元测试、收尾工作)]({{ site.baseurl }}/php/2018/05/23/create_a_query_builder_8.html)

## 前言

对于后端程序员来说，数据库操作是必备知识，对数据的增删查改也是业务中最普遍、频繁的操作。

对于关系型数据库，如 Mysql、Postgresql 等，我们可以使用 SQL (Structured Query Language) 来操作我们的数据，如 “SELECT * FROM test;” 这类的 SQL 语句，大部分的编程语言如 PHP、Python、go 等都提供了相应的数据库扩展，可以方便的进行数据库连接、执行 SQL 获取结果。

但是，直接使用基础的扩展效率并不高，每次都要创建连接、手动写 SQL、对执行结果进行判断、将查询得到的数据进行处理等，代码的重用性和维护性并不是很好，在多人开发的时候更是不能保证代码质量。而在项目开发中，我们想要的是一个更好用的可维护的工具，此时，对代码的封装、模块化就显得尤为重要，于是出现了两种方案：**查询构造器**，**对象关系映射**。

- **查询构造器 (query builder)**，顾名思义，它的目的就是以简便的形式构造、执行 SQL，为查询数据库的业务提供了方便好用的接口，一些知名的 web 框架如 PHP 的 Laravel、CodeIgniter、ThinkPHP 等都提供了好用的查询构造器。

- **对象关系映射 (ORM)** 是一种更面向对象的数据模型化操作，将数据库的数据映射成对象模型，数据库的直接操作对开发者透明，开发者只需关注对象模型即可，更符合面向对象程序思维。同样，很多框架也提供了 ORM 的方式去操作数据。

## 为什么要写查询构造器，我的需求是什么

写这个查询构造器的起因是 [workerman](http://www.workerman.net/ "workerman")，一个由 PHP 编写的、可以常驻内存的 Socket 框架。

当时选用 workerman 去做一个 webAPI 的项目，针对 workerman 的环境编写了一个简单的 http 框架 [WorkerA](https://github.com/wazsmwazsm/WorkerA "WorkerA")，一是没有找到合适的 (在一个非典型 web 环境中，想要一个类似 laravel 那样方便的查询构造器)，二是想要锻炼自己，于是选择了自己去写这个查询构造器。

如今该查询构造器已经完工，完整代码在我的框架核心代码中： [查看](https://github.com/wazsmwazsm/WorkerF/tree/master/src/WorkerF/DB/Drivers)

## Q&A

Q：为什么选择查询构造器而不是 ORM

A：对于我自己的需求，我需要一个简单好用又快速的工具，ORM 的性能和复杂性显然不合适。

***

Q：该查询构造器“特殊”在何处

A：区别于典型 web 的一次 HTTP 请求的代码运行周期，这个查询构造器除了可以使用在普通的 web 环境中，还支持常驻内存的环境，支持断线重连 (这个很重要)

***

Q：为什么选 PHP

A：因为我自己用 PHP 开发最多，重要的是 workerman 是 PHP 编写的。

***

Q：支持哪些数据库

A：目前支持了 Mysql、Postgresql、Sqlite 这三个数据库，而且全部通过了单元测试

***

Q：好的使用实践？

A：我在 workerman 的环境中使用的是单例模式，每个进程一个数据库连接单例模型。典型 web 环境下按照一般的查询构造器处理就行。同样常驻内存的情景可以自己写链接池之类的功能，当然这和查询构造器本身没有关系，一切由你的需求来决定。

## 需求和技术的选择 (想想自己想要什么)

**底层驱动的选择：**

- php 提供了 mysql 和 pgsql 这些常用数据库的扩展，但是每个扩展暴露出的接口都不大一样，封装不便，pass

- php 的 PDO 提供了多种数据库的底层驱动，统一了访问接口，prepare 方法可以有效的防止 SQL 注入，就选它

**错误异常处理：**

- 使用 PHP 的 try catch 机制，使用内置的异常和 PDO 的异常进行异常抛出

**代码结构的设计：**

- 查询构造器要支持多个数据库，我们要把相同的部分 (查询等) 封装起来，将不同的部分 (不同数据库的连接和设置) 独立开来，那么只需创建一个基类 (封装 PDO 的操作)，每个数据库单独建立一个类继承基类，将各自差异的部分重写即可。

**SQL 的构建：**

- sql 的构建属于字符串的操作，构建 sql 即是构造字符串，PHP 有丰富的字符串处理函数

**好用和性能的均衡考虑：**

- 有些工具好用但是性能不一定好 (比如正则表达式)，当然并不是说为了语言层面的极致性能而放弃一些便易性，查询构造器的瓶颈一般在于连接数据库的网络消耗和数据库对 SQL 的执行上。那么我们就以此作为平衡点，只要不比平衡点慢，我们可以选择更方便开发的方式写代码。

**测试：**

- 选择 phpunit 作为单元测试工具，以可测试为前提拆分代码为最小单元，提升代码的可维护性

## 总体思路

进行相关的需求分析和技术选择后，我们得出了构建查询构造器的大概的思路：

- 基于 PDO
- 通过字符串操作构建 SQL
- 将构造好的字符串交给 PDO 执行，获取结果
- 以继承、重写的模式搭建代码的架子
- 错误异常处理
- 编写单元测试

理清思路，开始干活。

(本系列文章默认您已经掌握 PHP、PDO、SQL 的基础知识)
