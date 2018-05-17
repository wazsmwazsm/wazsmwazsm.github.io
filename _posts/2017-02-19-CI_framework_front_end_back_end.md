---
layout: post
title:  "CI 框架网站前后台目录搭建"
date:   2017-02-19 22:46:02
categories: WEB框架
excerpt: 根据功能的不同，我们将网站分为前台和后台。前台用于展示内容给用户，后台用于管理员管理网站内容。
 同样，在网站应用的编码实现时，我们也需要根据前台、后台的功能不同来进行代码的安排和组织。
 那么，使用CodeIgniter(以3.x版本为例)搭建的网站，前后台应该怎么划分呢？
---

###  前台与后台
根据功能的不同，我们将网站分为前台和后台。前台用于展示内容给用户，后台用于管理员管理网站内容。
同样，在网站应用的编码实现时，我们也需要根据前台、后台的功能不同来进行代码的安排和组织。
那么，使用CodeIgniter(以3.x版本为例)搭建的网站，前后台应该怎么划分呢？

### 分开前后台的几种方式
如果有使用过ThinkPHP的朋友，肯定会熟悉下面这张图(TP3.2)

![](/assets/images/2017-02-19-CI_framework_front_end_back_end-1.png)

TP中实现多应用是很简单的，框架本省支持应用分组，创建一个新的应用只需在application中新建一个文件夹复制相关内容即可，而且支持公有模型、配置，且支持配置文件优先级。

比起来CI框架并不支持这样的功能，CI提供了两种方案给用户解决多应用问题：
#### 1、创建子目录
在Model、Controller等文件夹下建立子文件夹，加载相关模型、控制器时只需加上子目录即可，如下

![](/assets/images/2017-02-19-CI_framework_front_end_back_end-2.png)

#### 2、多应用多入口
在application下建立多个应用文件夹，每个文件夹下是一个应用，为每个应用创建入口文件，定义application路径，如下

![](/assets/images/2017-02-19-CI_framework_front_end_back_end-3.png)

![](/assets/images/2017-02-19-CI_framework_front_end_back_end-4.png)

#### 两种方式的特点
1、创建子目录方式： 属于一个CI应用，共享配置文件，无法进行单独的配置设置，比如后台要开钩子功能但是前台不需要，或者前后台需要分别加载各自的模块时，这种搭建方式就不是那么友好了。

2、多应用多入口： 前后台分为单独的CI应用，可以单独进行配置，通过各自的入口文件访问，应用完全分离，但是无法进行模型、自定义类库的共享。

### 方案的选择
无论选择哪种方案，都要跟着实际需求去选择，你的项目前后台是否需要单独的配置？是否是两个队伍分别开发前后台？等等。

就以我的博客为例，我选择了第2种方案。

那么第2种方案无法共享模型、类库的问题怎么解决呢？同样的数据，难道我要为了前后台写两份模型出来吗？

OK，显然CI并没有给我们提供分组、共享模型的功能，但是CI的特点之一就是“可扩展”，需要你自己动手做一些东西，这个框架没那么丰富，但却小巧、灵活，这也是CI的乐趣之一。

### 多入口应用搭建

#### 搭建目录、设置入口文件
将application种的文件复制两份，分别为home和admin(前后台)

![](/assets/images/2017-02-19-CI_framework_front_end_back_end-5.png)

设置入口文件的 $application_folder 变量

前台： index.php

![](/assets/images/2017-02-19-CI_framework_front_end_back_end-6.png)

后台： admin.php

![](/assets/images/2017-02-19-CI_framework_front_end_back_end-7.png)

此时在两个应用中创建不同的welcome控制器、视图，分别访问index.php
，admin,php就能分别访问到不同的应用了。

#### CI框架扩展
##### 1、Loader.php代码分析
CI的model、视图、类库等的加载都是靠核心类Loader.php完成的，代码文路径为system/core/Loader.php。

在Loader.php的80行左右，有如下代码，设置了model、helper、librariy的初始寻找路径
```php
 /**
  * List of paths to load libraries from
  *
  * @var array
  */
 protected $_ci_library_paths = array(APPPATH, BASEPATH);

 /**
  * List of paths to load models from
  *
  * @var array
  */
  // 模型初始寻找路径数组
 protected $_ci_model_paths = array(APPPATH);

 /**
  * List of paths to load helpers from
  *
  * @var array
  */
 protected $_ci_helper_paths = array(APPPATH, BASEPATH);
```

以model的加载为例，在320行左右，CI_Loader类的model方法中，读取了$_ci_model_paths属性，并依次读取初(第一个元素优先级最高)始的model路径数组，拼接成要加载的model路径，对model进行加载。

```php
// 循环路径数组
foreach ($this->_ci_model_paths as $mod_path)
{
    //如果不存在该路径，继续查找下一个路径
    if ( ! file_exists($mod_path.'models/'.$path.$model.'.php'))
    {
        continue;
    }

    // 加载model文件
    require_once($mod_path.'models/'.$path.$model.'.php');
    if ( ! class_exists($model, FALSE))
    {
        throw new RuntimeException($mod_path."models/".$path.$model.".php exists, but doesn't declare class ".$model);
    }
    // 加载文件后不继续加载其他寻找路径中的文件
    // 寻找路径中的优先级：左边最高
    break;
}
```

分析代码后我们很容易看到，只要修改$_ci_model_paths 属性的初始值，添加一个寻找路径(公有路径)到寻找路径数组的最前，就能优先在这个路径中寻找、加载模型了。

当然，这里有个问题，要直接修改Loader.php源码吗？

**NO**

CI提供了核心类扩展、替换功能，我们只需把Loader.php拷贝到home、admin两个应用中的core文件夹中，依次修改即可完成核心类的替换。

![](/assets/images/2017-02-19-CI_framework_front_end_back_end-8.png)

##### 2、共享路径的实现，定义常量

要实现共享路径，第一步，在application中添加一个common应用，为model、helper等添加文件夹

![](/assets/images/2017-02-19-CI_framework_front_end_back_end-9.png)

第二部，入口文件index.php和admin.php中定义常量，参考CI的常量定义代码来定义

```php
// 公共搜索路径
$common_path = 'application/common/';

if (is_dir($common_path)) {
 // 取得绝对路径
    if (($_temp = realpath($common_path)) !== FALSE) {
        $common_path = $_temp;
    } else {
        $common_path = strtr(rtrim($common_path, '/\\'), '/\\', DIRECTORY_SEPARATOR.DIRECTORY_SEPARATOR;
    }
} else {
    header('HTTP/1.1 503 Service Unavailable.', TRUE, 503);
    echo 'Your common folder path does not appear to be set correctly. Please open the following file and correct this: '.SELF;
    exit(3); // EXIT_CONFIG
}
// 定义路径为常量
define('APP_COMMON', $common_path.DIRECTORY_SEPARATOR);
```

打开home、admin应用的core/Loader.php，修改相关路径
注意代码优先级，我的设置为APPPATH, APP_COMMON，即最优先使用APPPATH下的model、libraries, 如果未找到，在APP_COMMON路径下搜索。


```php
/**
  * List of paths to load libraries from
  *
  * @var array
  */
 protected $_ci_library_paths = array(APPPATH, APP_COMMON, BASEPATH);

 /**
  * List of paths to load models from
  *
  * @var array
  */
 protected $_ci_model_paths = array(APPPATH, APP_COMMON);

 /**
  * List of paths to load helpers from
  *
  * @var array
  */
 protected $_ci_helper_paths = array(APPPATH, APP_COMMON, BASEPATH);
```
修改完成后，在common/model中新建模型文件，前后台应用就可以分别访问到啦。

### 最后的问题
比较遗憾的是虽然分开了多个应用，但是配置文件都是互相独立，无法实现TP那样的多层级多优先级配置设置，因为如果你查看config中的配置文件的话，很多配置文件的获取路径都是写死的，例如钩子类(system/core/Hooks.php)中的100行左右

```php
// Grab the "hooks" definition file.
if (file_exists(APPPATH.'config/hooks.php'))
{
    include(APPPATH.'config/hooks.php');
}

if (file_exists(APPPATH.'config/'.ENVIRONMENT.'/hooks.php'))
{
    include(APPPATH.'config/'.ENVIRONMENT.'/hooks.php');
}
```
相似的有很多配置文件的读取都是只能在APPPATH的config目录下读取，想要重写这个加载方案的话所要做的改动就太得不偿失了，当然这并不是CI的缺点，每个工具都有它的设计理念和使用场合，在实际需求中没有好坏与否，只说适不适用。

而且，对于对 .htaccess 重写支持不好的 nginx 来说，这种方式也不太友好。

当然这并不影响CI框架的好用和高扩展性，你甚至可以修改Loader.php给CI的MVC添加一个service层，只要你愿意的话。
