---
layout: post
title:  "搞懂依赖注入, 用 PHP 手写简易 IOC 容器"
date:   2019-04-21 15:36:08
categories: PHP
excerpt: 
---

## 前言

好的设计会提高程序的可复用性和可维护性，也间接的提高了开发人员的生产力。今天，我们就来说一下在很多框架中都使用的依赖注入。

## 一些概念

要搞清楚什么是依赖注入如何依赖注入，首先我们要明确一些概念。

### DIP (Dependence Inversion Principle) 依赖倒置原则：

程序要依赖于抽象接口，不要依赖于具体实现。

### IOC (Inversion of Control) 控制反转：

遵循依赖倒置原则的一种代码设计方案，依赖的创建 (控制) 由主动变为被动 (反转)。

### DI (Dependency Injection) 依赖注入：

控制反转的一种具体实现方法。通过参数注入的方式，将依赖的创建由主动变为被动 (实现了控制反转)。

光说理论有点不好理解，我们用代码举个例子。

首先，我们看依赖没有倒置时的一段代码：

```php

class Controller
{
    protected $service;

    public function __construct()
    {
        // 主动创建依赖
        $this->service = new Service(12, 13); 
    }       
}

class Service
{
    protected $model;
    protected $count;

    public function __construct($param1, $param2)
    {
        $this->count = $param1 + $param2;
        // 主动创建依赖
        $this->model = new Model('test_table'); 
    }
}

class Model
{
    protected $table;

    public function __construct($table)
    {
        $this->table = $table;
    }
}

$controller = new Controller;

```

上述代码的依赖关系是 Controller 依赖 Service，Service 依赖 Model。从控制的角度来看，Controller 主动创建依赖 Service，Service 主动创建依赖 Model。依赖是由被需求方内部产生的，需求方需要关心依赖的具体实现。这样的设计使代码耦合性变高，每次底层发生改变(如参数变动)，顶层就必须修改代码。


接下来，我们使用依赖注入实现控制反转，使依赖关系倒置：
```php
class Controller
{
    protected $service;
    // 依赖被动传入
    public function __construct(Service $service)
    {
        $this->service = $service; 
    }       
}

class Service
{
    protected $model;
    protected $count;
    // 依赖被动传入
    public function __construct(Model $model, $param1, $param2)
    {
        $this->count = $param1 + $param2;
        $this->model = $model; 
    }
}

class Model
{
    protected $table;

    public function __construct($table)
    {
        $this->table = $table;
    }
}

$model = new Model('test_table');
$service = new Service($model, 12, 13);
$controller = new Controller($service);
```

将依赖通过参数的方式从外部传入(即依赖注入)，控制的角度上依赖的产生从主动创建变为被动注入，依赖关系变为了依赖于抽象接口而不依赖于具体实现。此时的代码得到了解耦，提高了可维护性。

## 如何依赖注入，自动注入依赖

有了上面的一些理论基础，我们大致了解了依赖注入是什么，能干什么。

不过虽然上面的代码可以进行依赖注入了，但是依赖还是需要手动创建。我们可不可以创建一个工厂类，用来帮我们进行自动依赖注入呢？OK，我们需要一个 IOC 容器。

## 实现一个简单的 IOC 容器

