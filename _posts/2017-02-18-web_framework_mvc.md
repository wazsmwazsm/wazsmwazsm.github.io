---
layout: post
title:  "Web 框架的 MVC 符合标准的 MVC 吗？"
date:   2017-02-18 15:20:58
categories: WEB框架
excerpt: MVC是什么：一种设计模式？一种解决方案？
 MVC 设计模式最初由 Trygve Reenskaug 这位挪威计算机科学家在 70 年代提出并应用在 Xerox PARC 的 smalltalk 系统上，成功的将数据模型从系统内容中分离出来。MVC 设计模式普遍的应用在 GUI 应用程序上。
---

### 问题的开始
最近看了阮一峰老师的博文 [MVC，MVP 和 MVVM 的图示](http://www.ruanyifeng.com/blog/2015/02/mvcmvp_mvvm.html "MVC，MVP 和 MVVM 的图示")，很多人对 MVC 图示中的M和V之间的通信产生了疑问，那张图是这样的：

![](/assets/images/2017-02-18-web_framework_mvc-1.png)

> View 传送指令到 Controller【1】
> Controller 完成业务逻辑后，要求 Model 改变状态【1】
> **Model 将新的数据发送到 View，用户得到反馈** 【1】

问题来了。
What？Model 将新的数据发送到 View？不是 V->C->M -> C -> V 吗？

:joy:我可能用了假的 MVC。

当然，产生疑问的一定是写 web 应用的童鞋。

### MVC？

MVC是什么：一种设计模式？一种解决方案？...balabala

每个学习过 web 框架的朋友肯定会思考到这个问题

wiki 是这么解释的：[MVC_wiki](https://en.wikipedia.org/wiki/Model–view–controller "MVC_wiki")（这个词条在 wiki 上其实也是很有争议的）

当然后来 MVC 也有很多变体: MVP，MVVM...

MVC能做什么呢：业务逻辑和表现分离，数据展示相分离...balabala


我是分割线


MVC 设计模式最初由 Trygve Reenskaug 这位挪威计算机科学家在 70 年代提出并应用在 Xerox PARC 的 smalltalk 系统上，成功的将数据模型从系统内容中分离出来。MVC 设计模式普遍的应用在 GUI 应用程序上。

传统的 MVC 图示：
![](/assets/images/2017-02-18-web_framework_mvc-2.png)

没错，view 的更新是靠 model 的。

例如 Android 的 MVC 框架，model 只要发现自已属性有改变，就会发出事件广播，通知所有监听自已此属性的 view，view 收到通知后，会 update 自已。View 只管接收改变的通知，controller 只管改变 model，model 只管发出通知，这是非 web 应用的 MVC 框架常见的方案。【2】

其后这种设计模式被 web 开发者应用，成功的登陆 web 领域。而在 web 领域上的问题就是，客户端是浏览器，view 是HTML页面，应用层的协议是 HTTP ，而 HTTP 是一个无状态的协议，这种无状态行为使得 model 很难将更改通知 view。在 Web 上，为了发现对应用程序状态的修改，浏览器必须重新查询服务器。

### MVC2
90 年代 sun 公司为 java web 应用推出了一个设计模式 [JSP model 2 architecture](https://en.wikipedia.org/wiki/JSP_model_2_architecture "JSP model 2 architecture")，既 MVC2（MVC Model 2），它解决了全站 JSP 的 Model 1 模式下代码混乱的问题，并采用了 MVC 的设计思想。

同时，MVC 2的设计模式下：

> View接受用户输入,并传递到Controller.【3】
> Controller统一进行处理命令,交由Model处理具体的业务.【3】
> 经过处理Model更新后,Controller会选一个View并把Model内容传递给它.【3】

MVC2 图示

![](/assets/images/2017-02-18-web_framework_mvc-3.jpg)

在 MVC 2 模式下，Controller “掌管大权”，处理起了 model 和 view 之间的联系，model 和 view 变得互相不可见，无法单独通信了(当然这不是结果而是原因)。同时 MVC 2 也更好的解决了 MVC 在 Web 上的应用。

### MVP
当然这里不是指 most valuable player 啦（笑

MVC 2 更像是 Taligent 公司 90 年代提出的 [Model–view–presenter](https://en.wikipedia.org/wiki/Model–view–presenter "Model–view–presenter") 设计模式，既 MVP，当然，MVP 强调了 presenter 和 model、view 的双向通信。【6】

MVP设计模式：
![](/assets/images/2017-02-18-web_framework_mvc-4.png)

### 现在的 Web 框架是哪种设计模式呢？

举个栗子

CodeIgniter 的 MVC：
![](/assets/images/2017-02-18-web_framework_mvc-5.png)

Ruby on rails 的 MVC:
![](/assets/images/2017-02-18-web_framework_mvc-6.png)


是不是很熟悉呢？

同样，sitepoint 上的一篇关于 ROR 和 MVC 的文章[Getting Started with MVC](https://www.sitepoint.com/getting-started-with-mvc/ "Getting Started with MVC")很好的阐述了 Ruby on rails 和 MVC 设计模式的联系：

> #### History
> In order to understand the MVC framework in depth, we will explore its history first. The Model-View-Controller pattern was formulated in the 1970s by Trygve Reenskaug as part of a smalltalk system being developed at Xerox PARC. This was long before the World Wide Web saga and even before computers became personal. The smalltalk system with MVC design pattern focused on separation of concern, a design principle on which all modern web frameworks base their prioritization. They successfully separated the data model (the data processing part) from the content and content from presentation. They also implemented an intermediate driving force which was the first-in-command. Although it had some downsides, it attracted web developers, resulting in many early web frameworks using the MVC design pattern (and obviously evolving into a de facto standard for building Web applications).【7】

> #### So, is Rails a MVC framework?
> Theoretically, Rails doesn’t adhere to MVC standards. Classic MVC had models that can notify views about the changes directly. But, in Rails, model data gets sent to the views via the controller, and it is the controller that returns the HTML output back to the browser. This closely matches the Model2 design pattern initially developed by Sun Microsystems in the late 1990s for designing Java Web applications. But this wouldn’t bother Rails developers, as long as they have a delicious cup of hot coffee while busy crunching code. :)【7】

博主翻译:

#### 历史
为了深入理解 MVC 框架，先介绍一下 MVC 的历史。模型-视图-控制器 (model-view-controller) 模式是由 Trygve Reenskaug 在 1970 年代作为正在开发的 Xerox PARC smalltalk 系统的一部分而制定的。这远远早于万维网传奇的产生，甚至远早于个人电脑的产生。按照 MVC 设计模式的 smalltalk 系统关注关注点分离([separation of concern](https://en.wikipedia.org/wiki/Separation_of_concerns "separation of concern"))，一种所有现代 Web 框架都以其优先级为基础的设计原则。这个设计模式成功地将数据模型（数据处理部分）与演示内容从应用中分离出来，同时也首次实现了一个中间驱动层 (这里指 Cotroller )。虽然这个设计模式有一些缺点，但是瑕不掩瑜，很快的吸引了 web 开发者，使得很多早期的 web 框架使用了 MVC 设计模式(并且明显的演变为 web 应用搭建的事实标准)。


#### 所以说，Rails 是 MVC 框架吗？
理论上来说，Rails 不遵守 MVC 标准，传统的 MVC 有状态改变时可以通知 view 的 model， 但是在 Rails 中，模型数据通过控制器发送到视图，而控制器将 HTML 输出返回给浏览器，这与 Sun 公司最初于 20 世纪 90 年代开发的用于设计 Java Web 应用程序的 Model2 设计模式十分类似。不过这些并不会对 Rails 的开发者造成困扰，就像他们更希望在忙碌码代码时能有一杯好喝的热咖啡一样 :)

---

### 参考文章:

【1】[MVC，MVP 和 MVVM 的图示](http://www.ruanyifeng.com/blog/2015/02/mvcmvp_mvvm.html)

【2】[Android 设计模式之MVC模式](http://www.cnblogs.com/liqw/p/4175325.html)

【3】[MVC模式及MVC1和MVC2模式的区别](http://williamliu.iteye.com/blog/361628)

【4】[Model-View-Controller](https://en.wikipedia.org/wiki/Model–view–controller "MVC_wiki")

【5】[JSP model 2 architecture](https://en.wikipedia.org/wiki/JSP_model_2_architecture "JSP model 2 architecture")

【6】[Does PHP supports MVP pattern?](http://stackoverflow.com/questions/4530023/does-php-supports-mvp-pattern/15155046)

【7】[What is MVC in Ruby on Rails?](http://stackoverflow.com/questions/1931335/what-is-mvc-in-ruby-on-rails "What is MVC in Ruby on Rails?")
