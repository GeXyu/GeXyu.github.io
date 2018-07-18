---
title: Spring源码分析-AOP（七）AOP小结
comments: true
tags:
  - Spring源码分析-AOP（七）AOP小结
categories:
  - Spring源码分析-AOP（七）AOP小结
url: 181.html
id: 181
date: 2017-05-04 09:08:12
---

在我们看到的AOP部分源码中，可以看到 Proxy 代理对象的使用，在程序中是一个非常重要的部分，Spring AOP 充分利用 Java 的Proxy、反射以及第三方的 CGLIB 这些方案，通过这些技术，完成了 AOP 代理对象的生成。

个人觉得可以分为三个部分，IOC容器初始化，代理对象初始化和创建，执行代理对象。

### **1,IOC容器初始化**

![AOPinit](http://www.zzcode.cn/wp-content/uploads/2017/05/AOPinit.png)

在IOC容器初始化的时候解析XML遇到命名空间就会去找命名空间对应的Handler处理，解析不同的标签调用不同的parse,这里找到的是AopNamespaceHandler调用的ConfigBeanDefinitionParser，在其Parser方法中注册了一个AspectJAwareAdvisorAutoProxyCreator类，解析了AOP相关的标签。譬如<aop:pointcut><aop:aspect>等。

### **2,代理对象初始化和创建**

![AOPCreateObject](http://www.zzcode.cn/wp-content/uploads/2017/05/AOPCreateObject.png)因为我们向容器注册了一个AspectJAwareAdvisorAutoProxyCreator类，所以我们在创建Bean的时候会执行postProcessBeforeInstantiation方法。所以当我们第一次getBean的时候（也就是创建Bean的时候），会调用postProcessBeforeInstantiation方法。这个方法干了3件事：

*   1,获取Bean对应的Advisor。
*   2,获取所有匹配Bean的Advisor，使用AspectJExpressionPointcut 匹配。
*   3,创建代理对象，如果被对象继承接口就是用JDK代理，否则默认使用CGLIB。

### ** 执行代理对象**

![AOPJDK](http://www.zzcode.cn/wp-content/uploads/2017/05/AOPJDK.png)

这里用JDK为例，因为JdkDynamicAopProxy实现了InvocationHandler接口，这就说明每个代理类的实例都关联到了一个handler，当我们通过代理对象调用一个方法的时候，这个方法的调用就会被转发为由InvocationHandler这个接口的 invoke 方法来进行调用。

invoke方法做了这几件事：

*   1，获取所有的Advisor，封装到ReflectiveMethodInvocation中形成拦截器链。
*   2，如果连接器链为空就执行代理方法。
*   3，如果有拦截器，通过InterceptorAndDynamicMethodMatcher 匹配，匹配成功就在拦截器类进行逻辑调用。

### **Spring AOP 源码分析 **

在 Spring AOP 的基本实现中，我们可以看到 Proxy 代理对象的使用，在程序中是一个非常重要的部分，Spring AOP 充分利用 Java 的Proxy、反射以及第三方的 CGLIB 这些方案，通过这些技术，完成了 AOP AopProxy 代理对象的生成。

        回顾整个源码实现过程我们可以看到，首先在容器初始化时解析标签注册自动代理创建器AspectJAwareAdvisorAutoProxyCreator。在创建初始化和创建Bean的时候，通过BeanFactoryAdvisorRetrievalHelper 获取所有Bean相关的Advisor，使用AspectJExpressionPointcut 进行匹配Advisor，从而获得匹配的Advisor，最后通过ProxyFactory 创建代理对象。

        而最终 AopProxy 代理对象的产生，会交给 JdkDynamicAopProxy 和 CglibAopProxy 这两个工厂来完成，用的就是我们最开始说到的技术。

        在完成 AopProxy 代理对象后，我们就可以对 AOP 切面逻辑进行实现了，首先会对这些方法进行拦截，从而为这些方法提供工作空间，随后在进行回调，从而完成 AOP 切面实现的一整个逻辑。而这里的拦截 JdkDynamicAopProxy  主要是通过内部的 invoke 方法来实现，而 CGLIB 是通过 getCallbacks 方法来完成的。他们为 AOP 切面的实现提供了舞台。

### **总结**

        Spring AOP 秉承 Spring 一贯的设计理念，致力于 AOP 框架与 IOC 容器的紧密集成，以此来为 J2EE 的开发人员服务。这里仅仅介绍了一部分，还有很多地方都很值得大家去仔细专研。当然，AOP 的实现时一个三足鼎立的世界：AspectJ、JBoss AOP、Spring AOP。如果你有兴趣，除了 Spring AOP 以外 AspectJ 和 JBoss AOP 是非常值得大家研究以及借鉴的。

        特别是 AspectJ。我们知道 Spring 的增强都是用标准的 Java 类编写的。我们可以用一般的 Java 开发环境进行开发切面，虽然好用，但是开发人员必须对 Java 开发相当熟悉，仅仅使用 Java 也有一定的局限性。而 AspectJ 与之相反，他专注于切面的开发，虽然最初也仅仅是作为 Java 语言的扩展方式来实现，但通过特有的 AOP 语言，我们可以获得更强大以及细颗粒的控制，从而丰富了 AOP 工具集。所以 Spring AOP 为弥补自身的不足，在源码中集成了 AspectJ 框架，有兴趣大家也可以去研究一下。