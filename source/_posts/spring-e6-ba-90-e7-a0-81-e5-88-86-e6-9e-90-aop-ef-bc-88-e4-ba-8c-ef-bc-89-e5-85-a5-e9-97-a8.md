---
title: Spring源码分析-AOP（二）入门
comments: true
tags:
  - Spring源码分析-AOP（二）入门
categories:
  - Spring源码分析-AOP（二）入门
url: 83.html
id: 83
date: 2017-04-24 11:08:13
---

**Spring AOP **

AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP是OOP的延续，是软件开发中的一个热点，也是Spring框架中的一个重要内容，是函数式编程的一种衍生范型。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。

### **Spring AOP 架构**

**        先是生成代理对象，然后是拦截器的作用，最后是编织的具体实现。这是AOP实现的三个步骤，当然Spring AOP也是一样。**

**        而从**Spring AOP整体架构上看****，**其核心都是建立在代理上的。**当我们建立增强实例时，我们必须先使用 ProxyFactory 类加入我们需要织入该类的所有增强，然后为该类创建代理。一般而言，AOP实现代理的方法有三种，而 **Spring 内部用到了其中的两种方法：动态代理和静态代理，而动态代理又分为JDK动态代理和CGLIB代理**。下面是 Spring AOP 生成代理的原理图：

![231946_FGbT_2528735](http://www.zzcode.cn/wp-content/uploads/2017/04/231946_FGbT_2528735.png)

而特别需要注意的是，**在 Spring AOP 中，一个 Advisor（通知者，也翻作 通知器或者增强器）就是一个切面，他的主要作用是整合切面增强设计（Advice）和切入点设计（Pointcut）**。Advisor有两个子接口：IntroductionAdvisor 和 PointcutAdvisor，基本所有切入点控制的 Advisor 都是由 PointcutAdvisor 实现的。而下面的是Spring AOP的整体架构图

![041131_h0UQ_2528735](http://www.zzcode.cn/wp-content/uploads/2017/04/041131_h0UQ_2528735.jpg)

   乍一看，非常的复杂，但如果结合上面的Spring AOP生成代理的原理图一起看，也就那么回事，只是丰富的许多属性了，我们会慢慢介绍的。

准备一个代理对象

public class ProxyClass {
	public void sayProxy(){
		System.out.println("sayProxy......");
	}
}

一个通知类

public class AdviceClass {
	public void before(){
		System.out.println("sayBefore......");
	}
	public void after(){
		System.out.println("sayAfter......");
	}
}

spring的配置文件spring.xml

<?xml version="1.0" encoding="UTF-8"?>
<beans 
      xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:context="http://www.springframework.org/schema/context"
      xmlns:aop="http://www.springframework.org/schema/aop"	
      xsi:schemaLocation="
	  http://www.springframework.org/schema/beans 
	  http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
	  http://www.springframework.org/schema/context
      http://www.springframework.org/schema/context/spring-context-3.0.xsd	  
	  http://www.springframework.org/schema/aop 
	  http://www.springframework.org/schema/aop/spring-aop-3.0.xsd     
      ">
      
      
      <bean id="proxy" class="aop.ProxyClass" />
      <bean id="advice" class="aop.AdviceClass"/>
      <aop:config>
      	<aop:pointcut expression="execution(* aop.*.*(..))" id="pt"/>
      	<aop:aspect ref="advice">
      		<aop:before method="before" pointcut-ref="pt"/>
      		<aop:after method="after" pointcut-ref="pt"/>
      	</aop:aspect>	
      </aop:config>
</beans>

测试类

public class Aop_Test {
	@Test
	public void test1(){
		ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
		ProxyClass bean = context.getBean("proxy", ProxyClass.class);
		bean.sayProxy();
	}
}

上文说道，<aop:config>标签是在AopNamespaceHandler的init方法中解析。

@Override
	public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}

由此，我们知道只要配置文件中的“aop:config”的注解时就会使用解析器对 ConfigBeanDefinitionParser进行解析。上文也提到过得到NamespaceHandler方法后会调用parse方法。其实在**解析<aop:config>标签时parse方法中得的就是ConfigBeanDefinitionParser的解析器**，然后调用了它的parse方法。

    private static final String ASPECT = "aspect";
	private static final String EXPRESSION = "expression";
	private static final String ID = "id";
	private static final String POINTCUT = "pointcut";
	private static final String ADVICE\_BEAN\_NAME = "adviceBeanName";
	private static final String ADVISOR = "advisor";
	private static final String ADVICE_REF = "advice-ref";
	private static final String POINTCUT_REF = "pointcut-ref";
	private static final String REF = "ref";
	private static final String BEFORE = "before";
	private static final String DECLARE_PARENTS = "declare-parents";
	private static final String TYPE_PATTERN = "types-matching";
	private static final String DEFAULT_IMPL = "default-impl";
	private static final String DELEGATE_REF = "delegate-ref";
	private static final String IMPLEMENT_INTERFACE = "implement-interface";
	private static final String AFTER = "after";
	private static final String AFTER\_RETURNING\_ELEMENT = "after-returning";
	private static final String AFTER\_THROWING\_ELEMENT = "after-throwing";
	private static final String AROUND = "around";
	private static final String RETURNING = "returning";
	private static final String RETURNING_PROPERTY = "returningName";
	private static final String THROWING = "throwing";
	private static final String THROWING_PROPERTY = "throwingName";
	private static final String ARG_NAMES = "arg-names";
	private static final String ARG\_NAMES\_PROPERTY = "argumentNames";
	private static final String ASPECT\_NAME\_PROPERTY = "aspectName";
	private static final String DECLARATION\_ORDER\_PROPERTY = "declarationOrder";
	private static final String ORDER_PROPERTY = "order";
	private static final int METHOD_INDEX = 0;
	private static final int POINTCUT_INDEX = 1;
	private static final int ASPECT\_INSTANCE\_FACTORY_INDEX = 2;

	private ParseState parseState = new ParseState();


	@Override
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		CompositeComponentDefinition compositeDef =
				new CompositeComponentDefinition(element.getTagName(), parserContext.extractSource(element));
		parserContext.pushContainingComponent(compositeDef);
        //1 . 注册一个AspectJAwareAdvisorAutoProxyCreator类型的bean
		configureAutoProxyCreator(parserContext, element);
        //2\. 解析主标签下面的advisor标签，并且注册advisor.
		List<Element> childElts = DomUtils.getChildElements(element);
		for (Element elt: childElts) {
			String localName = parserContext.getDelegate().getLocalName(elt);
			if (POINTCUT.equals(localName)) {
				parsePointcut(elt, parserContext);
			}
			else if (ADVISOR.equals(localName)) {
				parseAdvisor(elt, parserContext);
			}
			else if (ASPECT.equals(localName)) {
				parseAspect(elt, parserContext);
			}
		}

		parserContext.popAndRegisterContainingComponent();
		return null;
	}

不难看出，其实上面就看你两件事，第一件事注册一个AspectJAwareAdvisorAutoProxyCreator类型的bean，第二件事就是解析主标签下面的advisor标签，并且注册advisor。

下面看一下是如何注册一个AspectJAwareAdvisorAutoProxyCreator，实现在AopNamespaceUtils

	private void configureAutoProxyCreator(ParserContext parserContext, Element element) {
		AopNamespaceUtils.registerAspectJAutoProxyCreatorIfNecessary(parserContext, element);
	}
	public static void registerAspectJAutoProxyCreatorIfNecessary(
			ParserContext parserContext, Element sourceElement) {

		BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAutoProxyCreatorIfNecessary(
				parserContext.getRegistry(), parserContext.extractSource(sourceElement));
        // 解析<aop:config>标签的proxy-target-class和expose-proxy属性
		useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
		registerComponentIfNecessary(beanDefinition, parserContext);
	}
	
	public static BeanDefinition registerAspectJAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
		//这里传递了一个类型为AspectJAwareAdvisorAutoProxyCreator的class
		return registerOrEscalateApcAsRequired(AspectJAwareAdvisorAutoProxyCreator.class, registry, source);
	}
	
	public static final String AUTO\_PROXY\_CREATOR\_BEAN\_NAME =
			"org.springframework.aop.config.internalAutoProxyCreator";
	private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, Object source) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
        // 定义有AUTO\_PROXY\_CREATOR\_BEAN\_NAME="org.springframework.aop.config.internalAutoProxyCreator"
		if (registry.containsBeanDefinition(AUTO\_PROXY\_CREATOR\_BEAN\_NAME)) {
            // 如果容器中已经存在自动代理构建器，则比较两个构建器的优先级
			BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO\_PROXY\_CREATOR\_BEAN\_NAME);
			if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
				int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
				int requiredPriority = findPriorityForClass(cls);
                // 保存优先级高的构建器
				if (currentPriority < requiredPriority) {
					apcDefinition.setBeanClassName(cls.getName());
				}
			}
			return null;
		}
       // 如果容器中还没有自动代理构建器则创建构建器相应的BeanDefinition对象 
		RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
		beanDefinition.setSource(source);
		beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
		beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
       // 向容器中注册代理构建器的BeanDefinition对象
		registry.registerBeanDefinition(AUTO\_PROXY\_CREATOR\_BEAN\_NAME, beanDefinition);
		return beanDefinition;
	}

这里就是注册了一个AspectJAwareAdvisorAutoProxyCreator类，需要注意的是这里的cls是调用方法的时候传递过来的，类型是AspectJAwareAdvisorAutoProxyCreator。那么AspectJAwareAdvisorAutoProxyCreator又有什么特殊的地方呢？通过查看继承关系，发现在他父类的父类AbstractAutoProxyCreator实现了SmartInstantiationAwareBeanPostProcessor接口，而这个接口继承了InstantiationAwareBeanPostProcessor。这个InstantiationAwareBeanPostProcessor是BeanPostProcessor的一个子类，在bean初始化的时候调用postProcessBeforeInstantiation方法。