---
title: Spring源码分析-IOC（一）容器
comments: true
tags:
  - Spring源码分析-IOC（一）容器
categories:
  - Spring源码分析-IOC（一）容器
url: 39.html
id: 39
date: 2017-04-19 11:02:21
---

**容器概述**
--------

        IoC也被称作依赖注入(DI)。它是一个处理对象依赖项的过程，也就是将他们一起工作的其他的对象，只有通过构造参数、工厂方法参数或者（属性注入）通过构造参数实例化或通过工厂方法返回对象后再设置属性。当创建bean后，IoC容器再将这些依赖项注入进去。这个过程基本上是反转的，因此得名控制反转（IoC）。

**IoC容器利用Java的POJO类和配置元数据来生成 完全配置和可执行 的系统或应用程序。而Bean在Spring中就是POJO，也可以认为Bean就是对象。**

**接口设计**
--------

![1](http://www.zzcode.cn/wp-content/uploads/2017/04/1-1.png)

这里主要是接口体系，而具体实现体系，比如DefaultListableBeanFactory就是为了实现ConfigurableBeanFactory，从而成为一个简单Ioc容器实现。与其他Ioc容器类似，XmlBeanFactory就是为了实现BeanFactory，但都是基于DefaultListableBeanFactory的基础做了扩展。同样的，ApplicationContext也一样。        

        从图中我们可以简要的做出以下分析：

            1.从接口BeanFactory到HierarchicalBeanFactory，再到ConfigurableBeanFactory,这是一条主要的BeanFactory设计路径。在这条接口设计路径中，BeanFactory，是一条主要的BeanFactory设计路径。在这条接口设计路径中，BeanFactory接口定义了基本的Ioc容器的规范。在这个接口定义中，包括了getBean()这样的Ioc容器的基本方法（通过这个方法可以从容器中取得Bean）。而HierarchicalBeanFactory接口在继承了BeanFactory的基本接口后，增加了getParentBeanFactory()的接口功能，使BeanFactory具备了双亲Ioc容器的管理功能。在接下来的ConfigurableBeanFactory接口中，主要定义了一些对BeanFactory的配置功能，比如通过setParentBeanFactory()设置双亲Ioc容器，通过addBeanPostProcessor()配置Bean后置处理器，等等。通过这些接口设计的叠加，定义了BeanFactory就是最简单的Ioc容器的基本功能。

            2.第二条接口设计主线是，以ApplicationContext作为核心的接口设计，这里涉及的主要接口设计有，从BeanFactory到ListableBeanFactory，再到ApplicationContext，再到我们常用的WebApplicationContext或者ConfigurableApplicationContext接口。我们常用的应用基本都是org.framework.context 包里的WebApplicationContext或者ConfigurableApplicationContext实现。在这个接口体现中，ListableBeanFactory和HierarchicalBeanFactory两个接口，连接BeanFactory接口定义和ApplicationContext应用的接口定义。在ListableBeanFactory接口中，细化了许多BeanFactory的接口功能，比如定义了getBeanDefinitionNames()接口方法；对于ApplicationContext接口，它通过继承MessageSource、ResourceLoader、ApplicationEventPublisher接口，在BeanFactory简单Ioc容器的基础上添加了许多对高级容器的特性支持。

            3.这个接口系统是以BeanFactory和ApplicationContext为核心设计的，而BeanFactory是Ioc容器中最基本的接口，在ApplicationContext的设计中，一方面，可以看到它继承了BeanFactory接口体系中的ListableBeanFactory、AutowireCapableBeanFactory、HierarchicalBeanFactory等BeanFactory的接口，具备了BeanFactory Ioc容器的基本功能；另一方面，通过继承MessageSource、ResourceLoadr、ApplicationEventPublisher这些接口，BeanFactory为ApplicationContext赋予了更高级的Ioc容器特性。对于ApplicationContext而言，为了在Web环境中使用它，还设计了WebApplicationContext接口，而这个接口通过继承ThemeSource接口来扩充功能。

** BeanFactory的接口体系**
---------------------

### **最原始的容器：BeanFactory**

package org.springframework.beans.factory;

import org.springframework.beans.BeansException;
import org.springframework.core.ResolvableType;

/\*\*
 \* BeanFactory作为最原始同时也最重要的Ioc容器,它主要的功能是为依赖注入 （DI） 提供支持， BeanFactory 和相关的接口，比如，BeanFactoryAware、 
 \* DisposableBean、InitializingBean，仍旧保留在 Spring 中，主要目的是向后兼容已经存在的和那些 Spring 整合在一起的第三方框架。在 Spring 中
 \* ，有大量对 BeanFactory 接口的实现。其中，最常被使用的是 XmlBeanFactory 类。这个容器从一个 XML 文件中读取配置元数据，由这些元数据来生成一
 \* 个被配置化的系统或者应用。在资源宝贵的移动设备或者基于applet的应用当中， BeanFactory 会被优先选择。否则，一般使用的是 ApplicationContext
 \* 
 \* 这里定义的只是一系列的接口方法，通过这一系列的BeanFactory接口，可以使用不同的Bean的检索方法很方便地从Ioc容器中得到需要的Bean，从而忽略具体
 \* 的Ioc容器的实现，从这个角度上看，这些检索方法代表的是最为基本的容器入口。
 \*
 \* @author Rod Johnson
 \* @author Juergen Hoeller
 \* @author Chris Beams
 \* @since 13 April 2001
 */
public interface BeanFactory {

	/\*\*
	 \* 转定义符"&" 用来引用实例，或把它和工厂产生的Bean区分开，就是说，如果一个FactoryBean的名字为a，那么，&a会得到那个Factory
	 \*
	 \* FactoryBean和BeanFactory 是在Spring中使用最为频繁的类，它们在拼写上很相似。一个是Factory，也就是Ioc容器或对象工厂；一个
	 \* 是Bean。在Spring中，所有的Bean都是由BeanFactory（也就是Ioc容器）来进行管理的。但对FactoryBean而言，这个Bean不是简单的Be
	 \* an，而是一个能产生或者修饰对象生成的工厂Bean，它的实现与设计模式中的工厂模式和修饰器模式类似。
	 */
	String FACTORY\_BEAN\_PREFIX = "&";

	/\*\*
	 \* 五个不同形式的getBean方法，获取实例
	 \* @param name 检索所用的Bean名
	 \* @return Object（<T> T） 实例对象
	 \* @throws BeansException 如果Bean不能取得
	 */
	Object getBean(String name) throws BeansException;
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;
	<T> T getBean(Class<T> requiredType) throws BeansException;
	Object getBean(String name, Object... args) throws BeansException;
	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	/\*\*
	 \* 让用户判断容器是否含有指定名字的Bean.
	 \* @param name 搜索所用的Bean名
	 \* @return boolean 是否包含其中
	 */
	boolean containsBean(String name);

	/\*\*
	 \* 查询指定名字的Bean是否是Singleton类型的Bean.
	 \* 对于Singleton属性，可以在BeanDefinition指定.
	 \* @param name 搜索所用的Bean名
	 \* @return boolean 是否包是Singleton
	 \* @throws NoSuchBeanDefinitionException 没有找到Bean
	 */
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	/\*\*
	 \* 查询指定名字的Bean是否是Prototype类型的。
	 \* 与Singleton属性一样，可以在BeanDefinition指定.
	 \* @param name 搜索所用的Bean名
	 \* @return boolean 是否包是Prototype
	 \* @throws NoSuchBeanDefinitionException 没有找到Bean
	 */
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	/\*\*
	 \* 查询指定了名字的Bean的Class类型是否是特定的Class类型.
	 \* @param name 搜索所用的Bean名
	 \* @param typeToMatch 匹配类型
	 \* @return boolean 是否是特定类型
	 \* @throws NoSuchBeanDefinitionException 没有找到Bean
	 */
	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	/\*\*
	 \* 查询指定名字的Bean的Class类型.
	 \* @param name 搜索所用的Bean名
	 \* @return 指定的Bean或者null(没有找到合适的Bean)
	 \* @throws NoSuchBeanDefinitionException 没有找到Bean
	 */
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	/\*\*
	 \* 查询指定了名字的Bean的所有别名，这些都是在BeanDefinition中定义的
	 \* @param name 搜索所用的Bean名
	 \* @return 指定名字的Bean的所有别名 或者一个空的数组
	 */
	String\[\] getAliases(String name);
}

### **容器的基础：XmlBeanFactory**

package org.springframework.beans.factory.xml;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.core.io.Resource;

/\*\*
 \* XmlBeanFactory是BeanFactory的最简单实现类
 \* 
 \* XmlBeanFactory的功能是建立在DefaultListableBeanFactory这个基本容器的基础上的，并在这个基本容器的基础上实行了其他诸如
 \* XML读取的附加功能。XmlBeanFactory使用了DefaultListableBeanFactory作为基础类，DefaultListableBeanFactory是一个很重
 \* 要的Ioc实现，会在下一章进行重点论述。
 \*
 \* @author Rod Johnson
 \* @author Juergen Hoeller
 \* @author Chris Beams
 \* @since 15 April 2001
 */
public class XmlBeanFactory extends DefaultListableBeanFactory {
	
	private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);

	/\*\*
	 \* 根据给定来源，创建一个XmlBeanFactory
	 \* @param resource  Spring中对与外部资源的抽象，最常见的是对文件的抽象，特别是XML文件。而且Resource里面通常
	 \* 是保存了Spring使用者的Bean定义，比如applicationContext.xml在被加载时，就会被抽象为Resource来处理。
	 \* @throws BeansException 载入或者解析中发生错误
	 */
	public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
	}

	/\*\*
	 \* 根据给定来源和BeanFactory，创建一个XmlBeanFactory
	 \* @param resource  Spring中对与外部资源的抽象，最常见的是对文件的抽象，特别是XML文件。而且Resource里面通常
	 \* 是保存了Spring使用者的Bean定义，比如applicationContext.xml在被加载时，就会被抽象为Resource来处理。
	 \* @param parentBeanFactory 父类的BeanFactory
	 \* @throws BeansException 载入或者解析中发生错误
	 */
	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
		this.reader.loadBeanDefinitions(resource);
	}
}

在XmlBeanFactory中实例化了一个XmlBeanDefinitionReader，这个Reader对象就是用来处理以xml形式的持有类信息的****BeanDefinitionl****类。在XmlBeanFactory这个容器中还需要****BeanDefinitionl****信息，这些信息被封装成**Resource**，然后把**Resource**传递给构造方法，然后再使用XmlBeanDefinitionReader进行处理，完成容器的初始化和注入。

下图是**XmlBeanFactory**的继承关系 

![zz](http://www.zzcode.cn/wp-content/uploads/2017/04/zz-2-300x114.png)

作为简单ioc容器系列最底层的实现XmlBeanFactory是建立在defaultListableBeanFactory容器的基础之上的，并在这个基本容器的基础上实现了其他诸如xml读取的附加功能。

**Resource****的接口体系**
---------------------

![20130711125429078](http://www.zzcode.cn/wp-content/uploads/2017/04/20130711125429078.jpg)

资源的原始接口为Resource，它继承自InputStreamResource，实现了其getInstream方法，这样所有的资源就是通过该方法来获取输入流的。对于资源的加载，也实现了统一，定义了一个资源加载顶级接口ResourceLoader，它默认的加载就是DefaultResourceLoader。

### **InputStreamSource接口**

package org.springframework.core.io;

import java.io.IOException;
import java.io.InputStream;

/\*\*
 \* InputStreamSource 封装任何能返回InputStream的类，比如File、Classpath下的资源和Byte Array等
 \*
 \* @author Juergen Hoeller
 \* @since 20.01.2004
 */
public interface InputStreamSource {
	
	/\*\*
	 \* 返回InputStream的类，比如File、Classpath下的资源和Byte Array等
	 \* @return InputStream 返回一个新的InputStream的对象
	 \* @throws IOException 如果资源不能打开则抛出异常
	 */
	InputStream getInputStream() throws IOException;
}

### **Resource接口**

package org.springframework.core.io;

import java.io.File;
import java.io.IOException;
import java.net.URI;
import java.net.URL;

/\*\*
 \* Resource接口抽象了所有Spring内部使用到的底层资源：File、URL、Classpath等。
 \* 同时，对于来源不同的资源文件，Resource也有不同实现：文件(FileSystemResource)、Classpath资源(ClassPathResource)、
 \* URL资源(UrlResource)、InputStream资源(InputStreamResource)、Byte数组(ByteArrayResource)等等。
 \*
 \* @author Juergen Hoeller
 \* @since 28.12.2003
 */
public interface Resource extends InputStreamSource {

	/\*\*
	 \* 判断资源是否存在
	 \* @return boolean 是否存在
	 */
	boolean exists();

	/\*\*
	 \* 判断资源是否可读
	 \* @return boolean 是否可读
	 */
	boolean isReadable();

	/\*\*
	 \* 是否处于开启状态
	 \* @return boolean 是否开启
	 */
	boolean isOpen();

	/\*\*
	 \* 得到URL类型资源，用于资源转换
	 \* @return URL 得到URL类型
	 \* @throws IOException 如果资源不能打开则抛出异常
	 */
	URL getURL() throws IOException;

	/\*\*
	 \* 得到URI类型资源，用于资源转换
	 \* @return URI 得到URI类型
	 \* @throws IOException 如果资源不能打开则抛出异常
	 */
	URI getURI() throws IOException;

	/\*\*
	 \* 得到File类型资源，用于资源转换
	 \* @return File 得到File类型
	 \* @throws IOException 如果资源不能打开则抛出异常
	 */
	File getFile() throws IOException;

	/\*\*
	 \* 获取资源长度
	 \* @return long 资源长度
	 \* @throws IOException 如果资源不能打开则抛出异常
	 */
	long contentLength() throws IOException;

	/\*\*
	 \* 获取lastModified属性
	 \* @return long 获取lastModified
	 \* @throws IOException 如果资源不能打开则抛出异常
	 */
	long lastModified() throws IOException;

	/\*\*
	 \* 创建一个相对的资源方法
	 \* @param relativePath 相对路径
	 \* @return Resource 返回一个新的资源
	 \* @throws IOException 如果资源不能打开则抛出异常
	 */
	Resource createRelative(String relativePath) throws IOException;

	/\*\*
	 \* 获取文件名称
	 \* @return String 文件名称或者null
	 */
	String getFilename();

	/\*\*
	 \* 得到错误处理信息，主要用于错误处理的信息打印
	 \* @return String 错误资源信息
	 */
	String getDescription();
}

根据上面的推论，我们可以理解为这两段代码，在某种程度来说是完全等效的

    ClassPathResource resource = new ClassPathResource("applicationContext.xml");
    InputStream inputStream = resource.getInputStream();

    Resource resource = new ClassPathResource("applicationContext.xml");
    InputStream inputStream = resource.getInputStream();

这样得到InputStream以后，我们可以拿来用了。值得一提是，不同的实现有不同的调用方法，这里就不展开了。

**编程式使用Ioc容器**
--------------

说完了BeanFactory和resource体系，可以看到XmlBeanFactory使用DefaultListableBeanFactory作为基类，DefaultListableBeanFactory也是一个很重要的ioc容器，在其他的ioc容器中，譬如ApplicationContext的实现和XmlBeanFactory实现是类似的。编程式的使用DefaultListableBeanFactory可以了解容器的基本过程，还可以看一下几个关键类（resource，DefaultListableBeanFactory，XmlBeanDefinitionReader）之间的关系，如何分离解耦又如何结合为ioc容器服务

public class ProgramBeanFactory{
	public static void main(String\[\] args) {
		ClassPathResource resource = new ClassPathResource("applicationContext.xml");
		DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
		XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
		reader.loadBeanDefinitions(resource);
		Message message = factory.getBean("message", Message.class);	//Message是自己写的测试类
		message.printMessage();
	}
}

 以上，可以简单说明我们在使用Ioc容器时，需要如下几个步骤：

            1，创建Ioc配置文件的抽象资源，这个抽象资源包含了BeanDefinition的定义信息。

            2，创建一个BeanFactory，这里使用了DefaultListableBeanFactory。

            3，创建一个载入BeanDefinition的读取器，这里使用XmlBeanDefinitionReader来载入XML文件形式的BeanDefinition，通过一个回调配置给BeanFactory。

            4，从定义好的资源位置读入配置信息，具体的解析过程由XmlBeanDefinitionReader来完成。完成整个载入和注册Bean定义之后，需要的Ioc容器就建立起来了。这个时候我们就可以直接使用Ioc容器了。