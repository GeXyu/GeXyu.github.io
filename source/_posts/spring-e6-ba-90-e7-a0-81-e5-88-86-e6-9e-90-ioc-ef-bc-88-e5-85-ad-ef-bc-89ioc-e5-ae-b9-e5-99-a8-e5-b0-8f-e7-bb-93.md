---
title: Spring源码分析-IOC（六）IOC容器小结
comments: true
tags:
  - Spring源码分析-IOC（六）IOC容器小结
categories:
  - Spring源码分析-IOC（六）IOC容器小结
url: 71.html
id: 71
date: 2017-04-23 13:07:49
---

 如果把Ioc容器比喻成一个人的话，Bean对象们就构成了他的骨架，依赖注入就是他的血肉，各种组件和支持则汇成了他的筋脉和皮肤，而各种特性则是他的灵魂。**各种特性真正的使****Spring Ioc****有别于其他****Ioc****框架，也成就了应用开发的丰富多彩，****Spring Ioc** **作为一个产品，可以说，他的各种特性才是它真正的价值所在。**

        Spring Ioc的特性非常的多，了解了Spring Ioc容器整个运行原理后，按照相同思路分析这些特性相信也不是一件难事。如果读者感兴趣的话，也可以按照相同的思路进行研究。这里仅仅举个例子：  
 

 **例子：****Bean****的完整生命周期**

        Spring Bean的完整生命周期从创建Spring容器开始，直到最终Spring容器销毁Bean，这其中包含了一系列关键点。

![TIM4](http://www.zzcode.cn/wp-content/uploads/2017/04/TIM4.png)

Bean的完整生命周期经历了各种方法调用，这些方法可以划分为以下几类：

*   1、Bean自身的方法：这个包括了Bean本身调用的方法和通过配置文件中<bean>的init-method和destroy-method指定的方法
*   2、Bean级生命周期接口方法：这个包括了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这些接口的方法
*   3、容器级生命周期接口方法：这个包括了InstantiationAwareBeanPostProcessor 和 BeanPostProcessor 这两个接口实现，一般称它们的实现类为“后处理器”。
*   4、工厂后处理器接口方法：这个包括了AspectJWeavingEnabler, ConfigurationClassPostProcessor, CustomAutowireConfigurer等等非常有用的工厂后处理器接口的方法。工厂后处理器也是容器级的。在应用上下文装配配置文件之后立即调用。

**总结：**
-------

        Spring Ioc容器的核心是BeanFactory和BeanDefinition。分别对应对象工厂和依赖配置的概念。虽然我们通常使用的是ApplicationContext的实现类，但ApplicationContext只是封装和扩展了BeanFactory的功能。XML的配置形式只是Spring依赖注入的一种常用形式而已，而AnnotationConfigApplicationContext配合Annotation注解和泛型，早已经提供了更简易的配置方式，AnnotationConfigApplicationContext和AnnotationConfigWebApplicationContext则是实现无XML配置的核心接口，但无论你使用任何配置，最后都会映射到BeanDefinition。

        其次，这里特别要注意的还是BeanDefinition， Bean在XML文件里面的展现形式是<bean id="...">...</bean>，当这个节点被加载到内存中，就被抽象为BeanDefinition了，在XML Bean节点中的那些关键字，在BeanDefinition中都有相对应的成员变量。如何把一个XML节点转换成BeanDefinition，这个工作自然是由BeanDefinitionReader来完成的。Spring通过定义BeanDefinition来管理基于Spring的应用中的各种对象以及它们之间的相互依赖关系。BeanDefinition抽象了我们对Bean的定义，是让容器起作用的主要数据类型。我们知道在计算机世界里，所有的功能都是建立在通过数据对现实进行抽象的基础上的。Ioc容器是用BeanDefinition来管理对象依赖关系的，对Ioc容器而言，BeanDefinition就是对控制反转模式中管理的对象依赖关系的数据抽象，也是容器实现控制反转的核心数据结构，有了他们容器才能发挥作用。

IOC容器启动执行流程：  ![](http://ool3owzgf.bkt.clouddn.com/IOC%E5%AE%B9%E5%99%A8%E5%90%AF%E5%8A%A8.png)
-----------------------------------------------------------------------------------------------

**   ioc容器启动其实就干了5件事**

*   1，启动容器refresh()
*   2，完成resource的定位
*   3，使用DefaultBeanDefinitionDocumentReader读入xml文件，并将xml转换为BeanDefinition
*   4，使用BeanDefinitionParserDelegate进行解析
*   5，使用DefaultListableBeanFactory 进行注册

IOC依赖注入流程图
----------

![](http://ool3owzgf.bkt.clouddn.com/%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5.png)

** 依赖注入也干这几件事**

*   1，得到bean
*   2，在父子容器中寻找bean，没有找到就创建bean
*   3，默认使用SimpleInstantiationStrategy类通过cglib技术生成对象
*   4，通过BeanDefinitionValueResolver 类来解析依赖属性
*   5，使用AbstractPropertyAccessor 来注入依赖属性

**最后，****其实****IoC****从原理上说是非常简单的，就是把****xml****文件解析出来，然后放到内存的****map****里，最后在内置容器里管理****bean****。但是看****IoC****的源码，却发现其非常庞大，看着非常吃力。这是因为****spring****加入了很多特性和为扩展性预留很多的接口，这些特性和扩展，造就了它无与伦比的功能以及未来无限的可能性，可以说正是他们将技术的美学以最简单的方法呈现在了人们面前，当然这也导致了他的复杂性**。