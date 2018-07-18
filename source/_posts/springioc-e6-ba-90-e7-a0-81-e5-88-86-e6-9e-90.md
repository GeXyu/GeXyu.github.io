---
title: Spring源码分析-IOC（二）入门
comments: true
tags:
  - Spring源码分析-IOC（二）入门
categories:
  - Spring源码分析-IOC（二）入门
url: 32.html
id: 32
date: 2017-04-20 09:31:54
---

**ClassPathXmlApplicationContext介绍**
------------------------------------

ApplicationContext是spring中较高级的容器。和BeanFactory类似，它可以加载配置文件中定义的bean，将所有的bean集中在一起，当有请求的时候分配bean。 另外，它增加了企业所需要的功能，比如，从属性文件从解析文本信息和将事件传递给所指定的监听器。这个容器在org.springframework.context.ApplicationContext接口中定义。ApplicationContext包含BeanFactory所有的功能，一般情况下，相对于BeanFactory，ApplicationContext会被推荐使用。但BeanFactory仍然可以在轻量级应用中使用，比如移动设备或者基于applet的应用程序。

### ApplicationContext接口关系

1.支持不同的信息源。扩展了MessageSource接口，这个接口为ApplicationContext提供了很多信息源的扩展功能，比如：国际化的实现为多语言版本的应用提供服务。  
2.访问资源。这一特性主要体现在ResourcePatternResolver接口上，对Resource和ResourceLoader的支持，这样我们可以从不同地方得到Bean定义资源。  
   这种抽象使用户程序可以灵活地定义Bean定义信息，尤其是从不同的IO途径得到Bean定义信息。这在接口上看不出来，不过一般来说，具体ApplicationContext都是继承了DefaultResourceLoader的子类。因为DefaultResourceLoader是AbstractApplicationContext的基类，关于Resource后面会有更详细的介绍。  
 3.支持应用事件。继承了接口ApplicationEventPublisher，为应用环境引入了事件机制，这些事件和Bean的生命周期的结合为Bean的管理提供了便利。  
 4.附件服务。EnvironmentCapable里的服务让基本的Ioc功能更加丰富。  
 5.ListableBeanFactory和HierarchicalBeanFactory是继承的主要容器。

  最常被使用的ApplicationContext接口实现类：  
   1，FileSystemXmlApplicationContext：该容器从XML文件中加载已被定义的bean。在这里，你需要提供给构造器XML文件的完整路径。  
   2，ClassPathXmlApplicationContext：该容器从XML文件中加载已被定义的bean。在这里，你不需要提供XML文件的完整路径，只需正确配置CLASSPATH   
   环境变量即可，因为，容器会从CLASSPATH中搜索bean配置文件。  
   3，WebXmlApplicationContext：该容器会在一个 web 应用程序的范围内加载在XML文件中

**准备工作**
--------

需要一个message类

public class Message {
	public void sayMessage(){
		System.out.println("IOC Message!!!");
	}
}
还需要一个写一个spring的配置文件spring.xml

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
      <bean id="message" class="ioc.Message"></bean>
</beans>

还需要一个测试类

public class Message_Test {	
	@Test
	public void test1(){
		ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
		Message bean = context.getBean("message", Message.class);
		bean.sayMessage();
	}
}

好了准备工作完成好了，我们先看一下这个ClassPathXmlApplicationContext里面是做了什么。

public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String\[\] {configLocation}, true, null);
	}
//这个构造方法实际上是调用了下面
public ClassPathXmlApplicationContext(String\[\] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}

先看一下这个super() 里面做了什么（注意，这里传递过来的parent为null），追踪发现在AbstractApplicationContext类中调用了setParent(parent);

private ApplicationContext parent;
//其实就是把ApplicationContext作为父容器
@Override
	public void setParent(ApplicationContext parent) {
		this.parent = parent;
		if (parent != null) {
			Environment parentEnvironment = parent.getEnvironment();
			if (parentEnvironment instanceof ConfigurableEnvironment) {
				getEnvironment().merge((ConfigurableEnvironment) parentEnvironment);
			}
		}
	}

接着回到ClassPathXmlApplicationContext的构造方法看一下 下一句代码setConfigLocations()，追踪代码可以找到在PropertyPlaceholderHelper类中的parseStringValue()方法进行了字符串替换  ，path 路径中含有 ${user.dir} ，则将替换为： System.getProperty(user.dir);

protected String parseStringValue(
			String strVal, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {

		StringBuilder result = new StringBuilder(strVal);

		int startIndex = strVal.indexOf(this.placeholderPrefix);
		while (startIndex != -1) {
			int endIndex = findPlaceholderEndIndex(result, startIndex);
			if (endIndex != -1) {
				String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
				String originalPlaceholder = placeholder;
				if (!visitedPlaceholders.add(originalPlaceholder)) {
					throw new IllegalArgumentException(
							"Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
				}
				// Recursive invocation, parsing placeholders contained in the placeholder key.
				//这里递归查找
				placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
				// Now obtain the value for the fully resolved key...
				String propVal = placeholderResolver.resolvePlaceholder(placeholder);
				if (propVal == null && this.valueSeparator != null) {
					int separatorIndex = placeholder.indexOf(this.valueSeparator);
					if (separatorIndex != -1) {
						String actualPlaceholder = placeholder.substring(0, separatorIndex);
						String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
						//这里是调用了实现PropertyPlaceholderHelper内部接口PlaceholderResolver方法resolvePlaceholder
						//：占位符 key -> value
						propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
						if (propVal == null) {
							propVal = defaultValue;
						}
					}
				}
				if (propVal != null) {
					// Recursive invocation, parsing placeholders contained in the
					// previously resolved placeholder value.
					propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
					//// 替换占位符具体值
					result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
					if (logger.isTraceEnabled()) {
						logger.trace("Resolved placeholder '" + placeholder + "'");
					}
					startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
				}
				else if (this.ignoreUnresolvablePlaceholders) {
					// Proceed with unprocessed value.
					startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
				}
				else {
					throw new IllegalArgumentException("Could not resolve placeholder '" +
							placeholder + "'" + " in string value \\"" + strVal + "\\"");
				}
				visitedPlaceholders.remove(originalPlaceholder);
			}
			else {
				startIndex = -1;
			}
		}

		return result.toString();
	}

再回到ClassPathXmlApplicationContext中，因为refresh为true，所以要执行refresh方法，这个refresh会牵扯一系列复杂操作，对于不同容器，实现都是类似的，所以封装在一个基类中完成，而在ClassPathXmlApplicationContext看到的只是一些简单的调用。