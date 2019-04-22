---
layout: post
title:  "搞懂依赖注入, 用 PHP 手写简易 IOC 容器"
date:   2019-04-21 15:36:08
categories: PHP
excerpt: 好的设计会提高程序的可复用性和可维护性，也间接的提高了开发人员的生产力。今天，我们就来说一下在很多框架中都使用的依赖注入。
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

控制反转的一种具体实现方法。通过参数的方式从外部传入依赖，将依赖的创建由主动变为被动 (实现了控制反转)。

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

上述代码的依赖关系是 Controller 依赖 Service，Service 依赖 Model。从控制的角度来看，Controller 主动创建依赖 Service，Service 主动创建依赖 Model。依赖是由需求方内部产生的，需求方需要关心依赖的具体实现。这样的设计使代码耦合性变高，每次底层发生改变(如参数变动)，顶层就必须修改代码。


接下来，我们使用依赖注入实现控制反转，使依赖关系倒置：
```php
class Controller
{
    protected $service;
    // 依赖被动传入。申明要 Service 类的实例 (抽象接口)
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

依赖注入是以构造函数参数的形式传入的，想要自动注入：
- 我们需要知道需求方需要哪些依赖，使用反射来获得
- 只有类的实例会被注入，其它参数不受影响

如何自动进行注入呢？当然是 PHP 自带的反射功能！
> 注：关于反射是否影响性能，答案是肯定的。但是相比数据库连接、网络请求的时延，反射带来的性能问题在绝大多数情况下并不会成为应用的性能瓶颈。

### 1.雏形
首先，创建 Container 类，getInstance 方法：
```php
class Container
{
    public static function getInstance($class_name, $params = [])
    {
        // 获取反射实例
        $reflector = new ReflectionClass($class_name);
        // 获取反射实例的构造方法
        $constructor = $reflector->getConstructor();
        // 获取反射实例构造方法的形参
        $di_params = [];
        if ($constructor) {
            foreach ($constructor->getParameters() as $param) {
                $class = $param->getClass();
                if ($class) { // 如果参数是一个类，创建实例
                    $di_params[] = new $class->name;
                }
            }
        }
        
        $di_params = array_merge($di_params, $params);
        // 创建实例
        return $reflector->newInstanceArgs($di_params);
    }
}

```

这里我们获取构造方法参数时用到了 [ReflectionClass](https://www.php.net/manual/zh/class.reflectionclass.php "ReflectionClass") 类，大家可以到官方文档了解一下该类包含的方法和用法，这里就不再赘述。

ok，有了 getInstance 方法，我们可以试一下自动注入依赖了：
```php
class A
{
    public $count = 100;
}

class B
{
    protected $count = 1;

    public function __construct(A $a, $count)
    {
        $this->count = $a->count + $count;
    }

    public function getCount()
    {
        return $this->count;
    }
}

$b = Container::getInstance(B::class, [10]);
var_dump($b->getCount()); // result is 110
```

### 2.进阶

虽然上面的代码可以进行自动依赖注入了，但是问题是只能构注入一层。如果 A 类也有依赖怎么办呢？

ok，我们需要修改一下代码：
```php
class Container
{
    public static function getInstance($class_name, $params = [])
    {
        // 获取反射实例
        $reflector = new ReflectionClass($class_name);
        // 获取反射实例的构造方法
        $constructor = $reflector->getConstructor();
        // 获取反射实例构造方法的形参
        $di_params = [];
        if ($constructor) {
            foreach ($constructor->getParameters() as $param) {
                $class = $param->getClass();
                if ($class) { // 如果参数是一个类，创建实例，并对实例进行依赖注入
                    $di_params[] = self::getInstance($class->name);
                }
            }
        }
        
        $di_params = array_merge($di_params, $params);
        // 创建实例
        return $reflector->newInstanceArgs($di_params);
    }
}
```

测试一下：
```php

class C 
{
    public $count = 20;
}
class A
{
    public $count = 100;

    public function __construct(C $c)
    {
        $this->count += $c->count;
    }
}

class B
{
    protected $count = 1;

    public function __construct(A $a, $count)
    {
        $this->count = $a->count + $count;
    }

    public function getCount()
    {
        return $this->count;
    }
}

$b = Container::getInstance(B::class, [10]);
var_dump($b->getCount()); // result is 130

```

> 上述代码使用递归完成了多层依赖的注入关系，程序中依赖关系层级一般不会特别深，递归不会造成内存遗漏问题。

### 3.单例

有些类会贯穿在程序生命周期中被频繁使用，为了在依赖注入中避免不停的产生新的实例，我们需要 IOC 容器支持单例模式，已经是单例的依赖可以直接获取，节省资源。

为 Container 增加单例相关方法：
```php
class Container
{
    protected static $_singleton = []; 

    // 添加一个实例到单例
    public static function singleton($instance)
    {
        if ( ! is_object($instance)) {
            throw new InvalidArgumentException("Object need!");
        }
        $class_name = get_class($instance);
        // singleton not exist, create
        if ( ! array_key_exists($class_name, self::$_singleton)) {
            self::$_singleton[$class_name] = $instance;
        }
    }
    // 获取一个单例实例
    public static function getSingleton($class_name)
    {
        return array_key_exists($class_name, self::$_singleton) ?
                self::$_singleton[$class_name] : NULL;
    }
    // 销毁一个单例实例
    public static function unsetSingleton($class_name)
    {
        self::$_singleton[$class_name] = NULL;
    }

}
```

改造 getInstance 方法：
```php

public static function getInstance($class_name, $params = [])
{
    // 获取反射实例
    $reflector = new ReflectionClass($class_name);
    // 获取反射实例的构造方法
    $constructor = $reflector->getConstructor();
    // 获取反射实例构造方法的形参
    $di_params = [];
    if ($constructor) {
        foreach ($constructor->getParameters() as $param) {
            $class = $param->getClass();
            if ($class) { 
                // 如果依赖是单例，则直接获取
                $singleton = self::getSingleton($class->name);
                $di_params[] = $singleton ? $singleton : self::getInstance($class->name);
            }
        }
    }
    
    $di_params = array_merge($di_params, $params);
    // 创建实例
    return $reflector->newInstanceArgs($di_params);
}

```

### 4.以依赖注入的方式运行方法

类之间的依赖注入解决了，我们还需要一个以依赖注入的方式运行方法的功能，可以注入任意方法的依赖。这个功能在实现路由分发到控制器方法时很有用。

增加 run 方法
```php
public static function run($class_name, $method, $params = [], $construct_params = [])
{
    if ( ! class_exists($class_name)) {
        throw new BadMethodCallException("Class $class_name is not found!");
    }

    if ( ! method_exists($class_name, $method)) {
        throw new BadMethodCallException("undefined method $method in $class_name !");
    }
    // 获取实例
    $instance = self::getInstance($class_name, $construct_params);

    // 获取反射实例
    $reflector = new ReflectionClass($class_name);
    // 获取方法
    $reflectorMethod = $reflector->getMethod($method);
    // 查找方法的参数
    $di_params = [];
    foreach ($reflectorMethod->getParameters() as $param) {
        $class = $param->getClass();
        if ($class) { 
            $singleton = self::getSingleton($class->name);
            $di_params[] = $singleton ? $singleton : self::getInstance($class->name);
        }
    }

    // 运行方法
    return call_user_func_array([$instance, $method], array_merge($di_params, $params));
}

```

测试：
```php
class A
{
    public $count = 10;
}

class B
{
    public function getCount(A $a, $count)
    {
        return $a->count + $count;
    }
}

$result = Container::run(B::class, 'getCount', [10]);
var_dump($result); // result is 20
```

ok，一个简单好用的 IOC 容器完成了，动手试试吧！

## 完整代码

IOC Container 的完整代码请见 [wazsmwazsm/IOCContainer](https://github.com/wazsmwazsm/IOCContainer "wazsmwazsm/IOCContainer")， 原先是在我的框架 [wazsmwazsm/WorkerA](https://github.com/wazsmwazsm/WorkerA "wazsmwazsm/WorkerA") 中使用，现在已经作为单独的项目，有完善的单元测试，可以使用到生产环境。

