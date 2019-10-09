---
title: Spring核心
date: 2019-03-20 23:23:19
tags: Java, Spring
top: true
copyright: true #新增,开启
---
## Spring 的本质系列之依赖注入

前言： Spring 这个轻量级的框架已经成为Web开发事实上的标准，阅读本篇文章之前希望你对OO,设计模式，单元测试，XML，反射等技术有一定了解。

## 概念：什么是IOC？

IoC(Inversion of Control)，意为控制反转，不是什么技术，而是一种设计思想。Ioc意味着**将你设计好的对象交给容器控制，而不是传统的在你的对象内部直接控制**。

如何理解好Ioc呢？理解好Ioc的关键是要明确“谁控制谁，控制什么，为何是反转（有反转就应该有正转了），哪些方面反转了”，那我们来深入分析一下：

- **谁控制谁，控制什么**：传统Java程序设计，我们直接在对象内部通过new进行创建对象，是程序主动去创建依赖对象；而IoC是有专门一个容器来创建这些对象，即由Ioc容器来控制对象的创建；谁控制谁？当然是IoC 容器控制了对象；控制什么？那就是主要控制了外部资源获取（不只是对象包括比如文件等）。

- **为何是反转，哪些方面反转了**：有反转就有正转，传统应用程序是由我们自己在对象中主动控制去直接获取依赖对象，也就是正转；而反转则是由容器来帮忙创建及注入依赖对象；为何是反转？因为由容器帮我们查找及注入依赖对象，对象只是被动的接受依赖对象，所以是反转；哪些方面反转了？依赖对象的获取被反转了。

### 1.对象的创建

面向对象的编程语言是用类(Class)来对现实世界进行抽象，在运行时这些类会生成对象(Object)。 

当然，单独的一个或几个对象根本没办法完成复杂的业务， 实际的系统是由千千万万个对象组成的， 这些对象需要互相协作才能干活，例如对象A调用对象B的方法，那必然会提出一个问题：对象A怎么才能获得对象B的引用呢？

最简单的办法无非是： 当对象A需要使用对象B的时候，把它给new 出来 ，这也是最常用的办法，java 不就是这么做的？例如：  

Apple a = new Apple();

后来业务变复杂了， 抽象出了一个水果(Fruit)的类， 创建对象会变成这个样子：

```java
Fruit f1 = new Apple();
Fruit f2 = new Banana();
Fruit f3 = ......
```

很自然的，类似下面的代码就会出现：

![](https://ww1.sinaimg.cn/large/007rAy9hgy1g1mdn91ta0j309i06iq2v.jpg)

这样的代码如果散落在各处，维护起来将会痛苦不堪， 例如你新加一个水果的类型Orange, 那得找到系统中所有的这些创建Fruit的地方，进行修改， 这绝对是一场噩梦。
 
解决办法也很简单，前辈们早就总结好了：工厂模式 

![](https://ww1.sinaimg.cn/large/007rAy9hly1g1mdorug29j30cg07xq30.jpg)

工厂模式，以及其他模式像抽象工厂， Builder模式提供的都是创建对象的方法。

这背后体现的都是**“封装变化”**的思想。
 
这些模式只是一些最佳实践而已： 起了一个名称、描述一下解决的问题、使用的范围和场景，码农们在项目中还得自己去编码实现他们。

### 2.解除依赖

我们再来看一个稍微复杂一点，更加贴近实际项目的例子：

一个订单处理类，它会被定时调用：查询数据库中订单的处理情况，必要时给下订单的用户发信。

![](https://ww1.sinaimg.cn/large/007rAy9hgy1g1mdqhsk4oj30hm0giq3r.jpg)

看起来也没什么难度， 需要注意的是很多类一起协作了， 尤其是OrderProcessor , 它依赖于
OrderService 和 EmailService这两个服务，它获取依赖的方式就是通过单例方法。

如果你想对这个process方法进行单元测试--这也是很多优秀的团队要求的-- 麻烦就来了。 

首先OrderService 确实会从真正的数据库中取得Order信息，你需要确保数据库中有数据， 数据库连接没问题，实际上如果数据库连接Container（例如Tomcat）管理的， 你没有Tomcat很难建立数据库连接。

其次这个EmailService 真的会对外发邮件， 你可不想对真正的用户发测试邮件，当然你可以修改数据库，把邮件地址改成假的，但那样很麻烦， 并且EmailService 会抛出一堆错误来，很不爽。

所有的这些障碍，最终会导致脆弱的单元测试： 速度慢， 不可重复，需要手工干预，不能独立运行。

想克服这些障碍，一个可行的办法就是不在方法中直接调用OrderService和EmailService的getInstance()方法， 而是把他们通过setter方法传进来。

![](https://ww1.sinaimg.cn/large/007rAy9hly1g1mdry36e0j30g003rq32.jpg)

通过这种方式，你的单元测试就可以构造一个假的OrderService 和假的EmailService 了。

例如OrderService 的冒牌货可以是MockOrderService , 它可以返回你想要的任何Order 对象， 而不是从数据库取。

MockEmailService 也不会真的发邮件， 而是把代码中试图发的邮件保存下来， 测试程序可以检查是否正确。

你的测试代码可能是这样的：

![](https://ww1.sinaimg.cn/large/007rAy9hly1g1mdsipj9kj30db07odg1.jpg)

当然， 有经验的你马上就会意识到：需要把OrderService 和 EmailService 变成 接口或者抽象类， 这样才可以把Mock对象传进来。 

这其实也遵循了面向对象编程的另外一个要求：对接口编程，而不是对实现编程。


### 3.Spring 依赖注入

啰啰嗦嗦说了这么多，快要和Spring扯上关系了。
 
上面的代码其实就是实现了一个依赖的注入，把两个冒牌货注入到业务类中(通过set方法)， 这个注入的过程是在一个测试类中通过代码完成的。

既然能把冒牌货注入进去，那毫无疑问肯定也能把一个正经的类安插进去，因为setter 方法接受的是接口，而不是具体类。

![](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6ULBhfFyXnPmY8sanSNqJpptbMJLeUGwPniawtrZg1Ut1UyNLJdj0HLlTftibTWnn25Q89iblLhwWjWYQ/640)

用这种方式来处理对象之间的依赖，会强迫你对接口编程，好处显而易见。 

随着系统复杂度的增长，这样的代码会越来越多，最后也会变得难于维护。 

能不能把各个类之间的依赖关系统一维护呢？
能不能把系统做的更加灵活一点，用声明的方式而不是用代码的方式来描述依赖关系呢？

肯定可以，在Java 世界里，如果想描述各种逻辑关系，XML是不二之选：

![](https://ww1.sinaimg.cn/large/007rAy9hgy1g1mdv0jo1qj30hs06cmxf.jpg)

这个xml 挺容易理解的，但是仅仅有它还不够，还缺一个解析器（假设叫做XmlAppContext）来解析，处理这个文件，基本过程是：

0. 解析xml, 获取各种元素

1. 通过Java反射把各个bean 的实例创建起来：com.coderising.OrderProcessor, OrderServiceImpl, EmailServiceImpl. 

2. 还是通过Java反射调用OrderProcessor的两个方法：setOrderService(....)  和 setEmailService(...) 把orderService, emailService 实例 注入进去。

应用程序使用起来就简单了：

```java
XmlAppContext ctx = new XmlAppContext("c:\\bean.xml");

OrderProcessor op = (OrderProcessor) ctx.getBean("order-processor");

op.process();
```

其实Spring的处理方式和上面说的非常类似，当然Spring 处理了更多的细节，例如不仅仅是setter方法注入， 还可以构造函数注入，init 方法，destroy方法等等，基本思想是一致的。

![](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6ULBhfFyXnPmY8sanSNqJppt9fcU8bxJwTh3zLDfAEFcTWBfmxEOCszBia5rWzkOTvkr7HGibJaOMffQ/640)

既然对象的创建过程和装配过程都是Spring做的，那Spring 在这个过程中就可以玩很多把戏了， 比如对你的业务类做点字节码级别的增强，搞点AOP什么的，这都不在话下了。 

### 4.IoC vs DI

“不要给我们打电话，我们会打给你的(don‘t call us, we‘ll call you)”这是著名的好莱坞原则。

在好莱坞，把简历递交给演艺公司后就只有回家等待。由演艺公司对整个娱乐项目完全控制，演员只能被动式的接受公司的差使,在需要的环节中，完成自己的演出。

这和软件开发有一定的相似性，演员们就像一个个Java Object, 最早的时候自己去创建自己所依赖的对象，   有了演艺公司（Spring容器）的介入，所有的依赖关系都是演艺公司搞定的，于是控制就翻转了 
Inversion of Control, 简称IoC。 

但是IoC这个词不能让人更加直观和清晰的理解背后所代表的含义，于是Martin Flower先生就创造了一个新词 : 依赖注入 (Dependency Injection，简称DI), 是不是更加贴切一点？

### 总结

DI—Dependency Injection，即“依赖注入”：组件之间依赖关系由容器在运行期决定，形象的说，即由容器动态的将某个依赖关系注入到组件之中。依赖注入的目的并非为软件系统带来更多功能，而是为了提升组件重用的频率，并为系统搭建一个灵活、可扩展的平台。通过依赖注入机制，我们只需要通过简单的配置，而无需任何代码就可指定目标需要的资源，完成自身的业务逻辑，而不需要关心具体的资源来自何处，由谁实现。

理解DI的关键是：“谁依赖谁，为什么需要依赖，谁注入谁，注入了什么”，那我们来深入分析一下：

- 谁依赖于谁：当然是应用程序依赖于IoC容器；

- 为什么需要依赖：应用程序需要IoC容器来提供对象需要的外部资源；

- 谁注入谁：很明显是IoC容器注入应用程序某个对象，应用程序依赖的对象；

- 注入了什么：就是注入某个对象所需要的外部资源（包括对象、资源、常量数据）。

IoC和DI由什么关系呢？其实它们是同一个概念的不同角度描述，由于控制反转概念比较含糊（可能只是理解为容器控制对象这一个层面，很难让人想到谁来维护对象关系），所以2004年大师级人物Martin Fowler又给出了一个新的名字：“依赖注入”，相对IoC 而言，“依赖注入”明确描述了“被注入对象依赖IoC容器配置依赖对象”。

对于Spring Ioc这个核心概念，我相信每一个学习Spring的人都会有自己的理解。这种概念上的理解没有绝对的标准答案，仁者见仁智者见智。理解了IoC和DI的概念后，一切都将变得简单明了，剩下的工作只是在框架中堆积木而已。

## Spring本质系列之AOP

### 问题来源

我们在做系统设计的时候，一个非常重要的工作就是把一个大系统做分解， 按业务功能分解成一个个低耦合、高内聚的模块，就像这样：

![](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6ULZa5xYnbCfBfOJSbTaBouH6HyhEOAO3QdfCXZ1HCMb4vOlch7P2yYGoOltv9LqRA4ibrH9QC5ibxUw/640)

但是分解以后就会发现有些很有趣的东西， 这些东西是通用的，或者是跨越多个模块的：

日志： 对特定的操作输出日志来记录

安全：在执行操作之前进行操作检查

性能：要统计每个方法的执行时间

事务：方法开始之前要开始事务， 结束后要提交或者回滚事务
等等....

这些可以称为是非功能需求， 但他们是多个业务模块都需要的， 是跨越模块的， 把他们放到什么地方呢？

最简单的办法就是把这些通用模块的接口写好， 让程序员在实现业务模块的时候去调用就可以了，码农嘛，辛苦一下也没什么。

![](https://ww1.sinaimg.cn/large/007rAy9hgy1g1mdu7gaq2j307p09p3yi.jpg)

这样做看起来没问题， 只是会产生类似这样的代码：

![](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6ULZa5xYnbCfBfOJSbTaBouHjGpR6CRuCerJx6KZJGfs355xXIf4eErVfnjgHWe2nKK0O1OuPUMnwA/640)

这样的代码也实现了功能，但是看起来非常的不爽， 那就是日志，性能，事务 相关的代码几乎要把真正的业务代码给淹没了。

不仅仅这一个类需要这么干， 其他类都得这么干， 重复代码会非常的多。

有经验的程序员还好， 新手忘记写这样的非业务代码简直是必然的。

### 设计模式：模板方法

用设计模式在某些情况下可以部分解决上面的问题，例如著名的模板方法：

![](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6ULZa5xYnbCfBfOJSbTaBouHwJHO3hL1CEcUNxoLUCicvbK7gdm5DMhzDUh582SYD7CUflzOV1LJxicg/640)

在父类（BaseCommand）中已经把那些“乱七八糟“的非功能代码都写好了， 只是留了一个口子（抽象方法doBusiness()）让子类去实现。

子类变的清爽， 只需要关注业务逻辑就可以了。
调用也很简单，例如：
BaseCommand  cmd = ...  获得PlaceOrderCommand的实例...
cmd.execute();

但是这样方式的巨大缺陷就是父类会定义一切： 要执行哪些非功能代码， 以什么顺序执行等等
子类只能无条件接受，完全没有反抗余地。

如果有个子类， 根本不需要事务， 但是它也没有办法把事务代码去掉。

### 设计模式：装饰者

如果利用装饰者模式， 针对上面的问题，可以带来更大的灵活性：

![](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6ULZa5xYnbCfBfOJSbTaBouH3LeZ2K6tDxLU5zCOpMG5kVEXpe3DOGbAa2JHvyJjNmHhHvvjibRy7fg/640)

现在让这个PlaceOrderCommand 能够打印日志，进行性能统计
Command cmd = new LoggerDecorator(
    new PerformanceDecorator(
        new PlaceOrderCommand()));
cmd.execute();

如果PaymentCommand 只需要打印日志，装饰一次就可以了：
Command cmd = new LoggerDecorator(
    new PaymentCommand());
cmd.execute();

可以使用任意数量装饰器，还可以以任意次序执行（严格意义上来说是不行的）， 是不是很灵活？ 

### AOP

如果仔细思考一下就会发现装饰者模式的不爽之处:

(1)  一个处理日志/性能/事务 的类为什么要实现 业务接口（Command）呢?

(2) 如果别的业务模块，没有实现Command接口，但是也想利用日志/性能/事务等功能，该怎么办呢？

最好把日志/安全/事务这样的代码和业务代码完全隔离开来，因为他们的关注点和业务代码的关注点完全不同 ，他们之间应该是正交的，他们之间的关系应该是这样的：

![](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6ULZa5xYnbCfBfOJSbTaBouH41wpaX8Q1uzbCCmT92GvcHCcUpX81YgeP1UqORm0rAKaeSU9eQAakQ/640)

如果把这个业务功能看成一层层面包的话， 这些日志/安全/事务 像不像一个个“切面”(Aspect) ？

如果我们能让这些“切面“能和业务独立，  并且能够非常灵活的“织入”到业务方法中， 那就实现了面向切面编程(AOP)！

### 实现AOP

现在我们来实现AOP吧， 首先我们得有一个所谓的“切面“类(Aspect)， 这应该是一个普通的java 类 ， 不用实现什么“乱七八糟”的接口。

以一个事务类为例：

![](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6ULZa5xYnbCfBfOJSbTaBouHE5ZBsjKibDxMONJEXOMDzp5OEgLIoUcibRZDicp0KQ8geUEwaEjKyj7Ww/640)

我们想达到的目的只这样的： 对于com.coderising这个包中所有类的execute方法， 在方法调用之前，需要执行Transaction.beginTx()方法， 在调用之后， 需要执行Transaction.commitTx()方法。

暂时停下脚步分析一下。

“对于com.coderising这个包中所有类的execute方法” ， 用一个时髦的词来描述就是切入点（PointCut） , 它可以是一个方法或一组方法（可以通过通配符来支持，你懂的）

”在方法调用之前/之后 ， 需要执行xxx“ , 用另外一个时髦的词来描述就是通知（Advice）

码农翻身认为，PointCut,Advice 这些词实在是不直观， 其实Spring的作者们也是这么想的 :  These terms are not Spring-specific… unfortunately, AOP terminology is not particularly intuitive; however, it would be even more confusing if Spring used its own terminology.

当然，想描述这些规则， xml依然是不二之选：

![](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6ULZa5xYnbCfBfOJSbTaBouH583lia6vRhPRJXzHCZXGicKF0Mk5vqxTdlYLlWq1JJicrDdsKD3SWMMKA/640)

注意：现在Transaction这个类和业务类在源代码层次上没有一点关系，完全隔离了。

隔离是一件好事情， 但是马上给我们带来了大麻烦 。
 
Java 是一门静态的强类型语言， 代码一旦写好， 编译成java class 以后 ，可以在运行时通过反射（Reflection）来查看类的信息， 但是想对类进行修改非常困难。 

而AOP要求的恰恰就是在不改变业务类的源代码（其实大部分情况下你也拿不到）的情况下， 修改业务类的方法, 进行功能的增强，就像上面给所有的业务类增加事务支持。

为了突破这个限制，大家可以说是费尽心机， 现在基本是有这么几种技术：

(1) 在编译的时候， 根据AOP的配置信息，悄悄的把日志，安全，事务等“切面”代码 和业务类编译到一起去。

(2) 在运行期，业务类加载以后， 通过Java动态代理技术为业务类生产一个代理类， 把“切面”代码放到代理类中，  Java 动态代理要求业务类需要实现接口才行。

(3) 在运行期， 业务类加载以后， 动态的使用字节码构建一个业务类的子类，将“切面”逻辑加入到子类当中去, CGLIB就是这么做的。

Spring采用的就是(1) +(2) 的方式，限于篇幅，这里不再展开各种技术了， 不管使用哪一种方式， 在运行时，真正干活的“业务类”其实已经不是原来单纯的业务类了， 它们被AOP了 ！

本文转载自“码农翻身”公众号