---
title: Spring源码分析-AOP（三）XML解析
comments: true
tags:
  - Spring源码分析-AOP（三）XML解析
categories:
  - Spring源码分析-AOP（三）XML解析
url: 100.html
id: 100
date: 2017-04-26 15:09:28
---

上文在ico容器初始化时，注册了一个AspectJAwareAdvisorAutoProxyCreator类，也提到了容器初始化时解析<aop:config>标签会用AopNamespaceHandler调用ConfigBeanDefinitionParser的Parser方法。

   @Override
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        CompositeComponentDefinition compositeDef =
                new CompositeComponentDefinition(element.getTagName(), parserContext.extractSource(element));
        parserContext.pushContainingComponent(compositeDef);

        // 创建AOP自动代理创建器
        configureAutoProxyCreator(parserContext, element);

        //遍历并解析<aop:config>的子标签
        List<Element> childElts = DomUtils.getChildElements(element);
        for (Element elt: childElts) {
            String localName = parserContext.getDelegate().getLocalName(elt);
            if ("pointcut".equals(localName)) {
                // 解析<aop:pointcut>标签
                parsePointcut(elt, parserContext);
            } else if ("advisor".equals(localName)) {
                // 解析<aop:advisor>标签
                parseAdvisor(elt, parserContext);
            } else if ("aspect".equals(localName)) {
                // 解析<aop:aspect>标签
                parseAspect(elt, parserContext);
            }
        }

        parserContext.popAndRegisterContainingComponent();
        return null;
    }

parse方法首先调用解析器ConfigBeanDefinitionParser的configureAutoProxyCreator方法来向容器中注册一个自动代理构建器AspectJAwareAdvisorAutoProxyCreator对象，然后调用parsePointcut方法解析<aop:pointcut>标签，调用parseAdvisor方法解析<aop:advisor>标签，调用parseAspect方法解析<aop:aspect>标签。

### **解析<aop:config>属性** 

想容器注册了AspectJAwareAdvisorAutoProxyCreator对象之后，接着调用AopNamespaceUtils中useClassProxyingIfNecessary方法来解析<aop:config>的两个属性。

private static void useClassProxyingIfNecessary(BeanDefinitionRegistry registry, Element sourceElement) {
		if (sourceElement != null) {
			 // 解析proxy-target-class属性
			boolean proxyTargetClass = Boolean.valueOf(sourceElement.getAttribute(PROXY\_TARGET\_CLASS_ATTRIBUTE));
			if (proxyTargetClass) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
			// 解析expose-proxy属性
			boolean exposeProxy = Boolean.valueOf(sourceElement.getAttribute(EXPOSE\_PROXY\_ATTRIBUTE));
			if (exposeProxy) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}

在是对两个属性的处理

public static final String AUTO\_PROXY\_CREATOR\_BEAN\_NAME =
			"org.springframework.aop.config.internalAutoProxyCreator";
public static void forceAutoProxyCreatorToUseClassProxying(BeanDefinitionRegistry registry) {
		if (registry.containsBeanDefinition(AUTO\_PROXY\_CREATOR\_BEAN\_NAME)) {
			BeanDefinition definition = registry.getBeanDefinition(AUTO\_PROXY\_CREATOR\_BEAN\_NAME);
			definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE);
		}
	}

	public static void forceAutoProxyCreatorToExposeProxy(BeanDefinitionRegistry registry) {
		if (registry.containsBeanDefinition(AUTO\_PROXY\_CREATOR\_BEAN\_NAME)) {
			BeanDefinition definition = registry.getBeanDefinition(AUTO\_PROXY\_CREATOR\_BEAN\_NAME);
			definition.getPropertyValues().add("exposeProxy", Boolean.TRUE);
		}
	}

### **解析<aop:pointcut>标签**

回到ConfigBeanDefinitionParser的Parser方法，他会调用parsePointcut方法解析<aop:pointcut>标签。

private AbstractBeanDefinition parsePointcut(Element pointcutElement, ParserContext parserContext) {
        // 获取id属性值
        String id = pointcutElement.getAttribute("id");
        // 获取expression属性值
        String expression = pointcutElement.getAttribute("expression");

        AbstractBeanDefinition pointcutDefinition = null;

        try {
            this.parseState.push(new PointcutEntry(id));
            //根据切点表达式来创建一个Pointcut对象的BeanDefinition
            pointcutDefinition = createPointcutDefinition(expression);
            pointcutDefinition.setSource(parserContext.extractSource(pointcutElement));

            String pointcutBeanName = id;
            if (StringUtils.hasText(pointcutBeanName)) {
                //id属性值不为空时，使用id值为Pointcut的bean名称然后注册到容器中
                parserContext.getRegistry().registerBeanDefinition(pointcutBeanName, pointcutDefinition);
            } else {
                //id属性值为空时，使用bean名称生成器来为Pointcut创建bean名称然后注册到容器中
                pointcutBeanName = parserContext.getReaderContext().registerWithGeneratedName(pointcutDefinition);
            }

            parserContext.registerComponent(
                    new PointcutComponentDefinition(pointcutBeanName, pointcutDefinition, expression));
        } finally {
            this.parseState.pop();
        }

        return pointcutDefinition;
    }

我们看一下createPointcutDefinition方法创建了什么？

 protected AbstractBeanDefinition createPointcutDefinition(String expression) {
        RootBeanDefinition beanDefinition = new RootBeanDefinition(AspectJExpressionPointcut.class);
        //创建AspectJExpressionPointcut对象，域为SCOPE_PROTOTYPE
        beanDefinition.setScope(BeanDefinition.SCOPE_PROTOTYPE);
        beanDefinition.setSynthetic(true);
        beanDefinition.getPropertyValues().add("expression", expression);
        return beanDefinition;
    }

从上面可以看到他创建了一个AspectJExpressionPointcut对象，域为SCOPE_PROTOTYPE。

### ** 解析<aop:advisor>标签**

继续回到ConfigBeanDefinitionParser类的parse方法，看他如何解析<aop:advisor>标签

private void parseAdvisor(Element advisorElement, ParserContext parserContext) {

        //创建一个Advisor对象对应的BeanDefintion对象
        AbstractBeanDefinition advisorDef = createAdvisorBeanDefinition(advisorElement, parserContext);
        //获取id属性值
        String id = advisorElement.getAttribute(ID);

        try {
            this.parseState.push(new AdvisorEntry(id));
            String advisorBeanName = id;
            if (StringUtils.hasText(advisorBeanName)) {
                //id属性值不为空时，使用id值为Advisor的bean名称，并注册到容器中
                parserContext.getRegistry().registerBeanDefinition(advisorBeanName, advisorDef);
            } else {
                //id属性值为空时，使用bean名称生成器来为Advisor创建bean名称，并注册到容器中
                advisorBeanName = parserContext.getReaderContext().registerWithGeneratedName(advisorDef);
            }

            //获取Advisor的Pointcut
            //解析pointcut和poincut-ref属性
            Object pointcut = parsePointcutProperty(advisorElement, parserContext);
            if (pointcut instanceof BeanDefinition) {
                //获取的是一个根据pointcut属性所指定的切点表达式来创建的的一个Poincut bean
                advisorDef.getPropertyValues().add("pointcut", pointcut);
                parserContext.registerComponent(
                        new AdvisorComponentDefinition(advisorBeanName, advisorDef, (BeanDefinition) pointcut));
            } else if (pointcut instanceof String) {
                //获取的是pointcut-ref属性值指向的一个Pointcut bean。
                advisorDef.getPropertyValues().add("pointcut", new RuntimeBeanReference((String) pointcut));
                parserContext.registerComponent(
                        new AdvisorComponentDefinition(advisorBeanName, advisorDef));
            }
        } finally {
            this.parseState.pop();
        }
    }

其实这个parseAdvisor方法就干两件事：

一：通过调用createAdvisorBeanDefinition方法创建并注册了Advisor对象对应的BeanDefintion对象。

二：通过调用parsePointcutProperty方法解析通过pointcut或者pointcut-ref属性获取Advisor的Pointcut

#### **createAdvisorBeanDefinition方法**

 private AbstractBeanDefinition createAdvisorBeanDefinition(Element advisorElement, ParserContext parserContext) {

        // 指定向容器中注入DefaultBeanFactoryPointcutAdvisor对象作为Advisor
        RootBeanDefinition advisorDefinition = new RootBeanDefinition(DefaultBeanFactoryPointcutAdvisor.class);
        advisorDefinition.setSource(parserContext.extractSource(advisorElement));

        // 指定Advisor的Advice对象
        // 获取advice-ref属性值
        String adviceRef = advisorElement.getAttribute("advice-ref");
        if (!StringUtils.hasText(adviceRef)) {
            parserContext.getReaderContext().error(
                    "'advice-ref' attribute contains empty value.", advisorElement, this.parseState.snapshot());
        }
        else {
            advisorDefinition.getPropertyValues().add(
                    "adviceBeanName", new RuntimeBeanNameReference(adviceRef));
        }

        // 获取order值，用于指定Advise的执行顺序
        if (advisorElement.hasAttribute("order")) {
            advisorDefinition.getPropertyValues().add(
                    "order", advisorElement.getAttribute("order"));
        }

        return advisorDefinition;
    }

可以从上面的代码看到createAdvisorBeanDefinition方法不仅创建了Advisor有关的BeanDefinitiion对象，还获取了order和adice-ref属性来设置Advisor的相应属性。

#### **parsePointcutProperty方法**

private Object parsePointcutProperty(Element element, ParserContext parserContext) {
        // poincut和pointcut-ref属性不能同时定义
        if (element.hasAttribute("pointcut") && element.hasAttribute("pointcut-ref")) {
            parserContext.getReaderContext().error(
                    "Cannot define both 'pointcut' and 'pointcut-ref' on <advisor> tag.",
                    element, this.parseState.snapshot());
            return null;
        } else if (element.hasAttribute("pointcut")) {
            String expression = element.getAttribute("pointcut");
            // 根据切点表达式来创建一个Pointcut的BeanDefinition对象
            AbstractBeanDefinition pointcutDefinition = createPointcutDefinition(expression);
            pointcutDefinition.setSource(parserContext.extractSource(element));
            return pointcutDefinition;
        } else if (element.hasAttribute("pointcut-ref")) {
            // 获取pointcut-ref属性值并返回
            String pointcutRef = element.getAttribute("pointcut-ref");
            if (!StringUtils.hasText(pointcutRef)) {
                parserContext.getReaderContext().error(
                        "'pointcut-ref' attribute contains empty value.", element, this.parseState.snapshot());
                return null;
            }
            return pointcutRef;
        } else {
            parserContext.getReaderContext().error(
                    "Must define one of 'pointcut' or 'pointcut-ref' on <advisor> tag.",
                    element, this.parseState.snapshot());
            return null;
        }
    }

<aop:advisor>有id、advice-ref、order、pointcut和pointcut-ref共5个属性，其中前三个属性只需要简单获取属性值就可以了，而这段代码的作用就是用来解析pointcut和pointcut-ref属性。

### ** 解析<aop:aspect>标签**

ConfigBeanDefinitionParser解析器调用它的parseAspect方法解析<aop:aspect>标签

private void parseAspect(Element aspectElement, ParserContext parserContext) {
        // 获取id属性值
        String aspectId = aspectElement.getAttribute("id");
        // 获取ref属性值
        String aspectName = aspectElement.getAttribute("ref");

        try {
            this.parseState.push(new AspectEntry(aspectId, aspectName));
            List<BeanDefinition> beanDefinitions = new ArrayList<BeanDefinition>();
            List<BeanReference> beanReferences = new ArrayList<BeanReference>();

            List<Element> declareParents = DomUtils.getChildElementsByTagName(aspectElement, "declare-parents");
            // 遍历并解析<aop:declare-parents>标签
            // 定义有METHOD_INDEX=0
            for (int i = METHOD_INDEX; i < declareParents.size(); i++) {
                Element declareParentsElement = declareParents.get(i);
                // 解析<aop:declare-parents>标签
                beanDefinitions.add(parseDeclareParents(declareParentsElement, parserContext));
            }

            // 遍历并解析before、after、after-returning、after-throwing和around标签
            NodeList nodeList = aspectElement.getChildNodes();
            boolean adviceFoundAlready = false;
            for (int i = 0; i < nodeList.getLength(); i++) {
                Node node = nodeList.item(i);
                // 判断当前
                if (isAdviceNode(node, parserContext)) {
                    if (!adviceFoundAlready) {
                        adviceFoundAlready = true;
                        if (!StringUtils.hasText(aspectName)) {
                            parserContext.getReaderContext().error(
                                    "<aspect> tag needs aspect bean reference via 'ref' attribute when declaring advices.",
                                    aspectElement, this.parseState.snapshot());
                            return;
                        }
                        beanReferences.add(new RuntimeBeanReference(aspectName));
                    }
                    // 解析adice相关的标签，并创建和注册相应的BeanDefinition对象
                    AbstractBeanDefinition advisorDefinition = parseAdvice(
                            aspectName, i, aspectElement, (Element) node, parserContext, beanDefinitions, beanReferences);
                    beanDefinitions.add(advisorDefinition);
                }
            }

            AspectComponentDefinition aspectComponentDefinition = createAspectComponentDefinition(
                    aspectElement, aspectId, beanDefinitions, beanReferences, parserContext);
            parserContext.pushContainingComponent(aspectComponentDefinition);

            // 遍历并解析<aop:pointcut>标签
            List<Element> pointcuts = DomUtils.getChildElementsByTagName(aspectElement, POINTCUT);
            for (Element pointcutElement : pointcuts) {
                parsePointcut(pointcutElement, parserContext);
            }

            parserContext.popAndRegisterContainingComponent();
        } finally {
            this.parseState.pop();
        }
    }

parseAspect方法解析aspect标签下的pointcut、declare-parents和5个advice标签，其中pointcut标签的解析已经在前面看过了，这里我们只需要看后面两种标签的解析。

（1）解析<aop:declare-parents>标签是会调用parseDeclareParents方法

private AbstractBeanDefinition parseDeclareParents(Element declareParentsElement, ParserContext parserContext) {
        // 使用BeanDefinitionBuilder对象来构造一个BeanDefinition对象
        BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(DeclareParentsAdvisor.class);
        builder.addConstructorArgValue(declareParentsElement.getAttribute("implement-interface"));
        builder.addConstructorArgValue(declareParentsElement.getAttribute("types-matching"));

        String defaultImpl = declareParentsElement.getAttribute("default-impl");
        String delegateRef = declareParentsElement.getAttribute("delegate-ref");

        // default-impl和delegate-ref不能同时定义
        if (StringUtils.hasText(defaultImpl) && !StringUtils.hasText(delegateRef)) {
            builder.addConstructorArgValue(defaultImpl);
        } else if (StringUtils.hasText(delegateRef) && !StringUtils.hasText(defaultImpl)) {
            builder.addConstructorArgReference(delegateRef);
        } else {
            parserContext.getReaderContext().error(
                    "Exactly one of the " + DEFAULT\_IMPL + " or " + DELEGATE\_REF + " attributes must be specified",
                    declareParentsElement, this.parseState.snapshot());
        }

        AbstractBeanDefinition definition = builder.getBeanDefinition();
        definition.setSource(parserContext.extractSource(declareParentsElement));
        // 向容器注册BeanDefinitiion对象
        parserContext.getReaderContext().registerWithGeneratedName(definition);
        return definition;
    }

parseDeclareParents方法作用是向容器注册一个DeclareParentsAdvisor对象。

（2）解析advice标签。spring提供了<aop:before>、<aop:after>、<aop:after-returning>、<aop:after-throwing>、<aop:around>5个advice标签，parseAspect调用parseAdvice方法来处理这些标签。

private AbstractBeanDefinition parseAdvice(
            String aspectName, int order, Element aspectElement, Element adviceElement, ParserContext parserContext,
            List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {

        try {
            this.parseState.push(new AdviceEntry(parserContext.getDelegate().getLocalName(adviceElement)));

            // 创建一个方法工厂bean
            RootBeanDefinition methodDefinition = new RootBeanDefinition(MethodLocatingFactoryBean.class);
            methodDefinition.getPropertyValues().add("targetBeanName", aspectName);
            methodDefinition.getPropertyValues().add("methodName", adviceElement.getAttribute("method"));
            methodDefinition.setSynthetic(true);

            // 创建一个用于获取aspect实例的工厂
            RootBeanDefinition aspectFactoryDef =
                    new RootBeanDefinition(SimpleBeanFactoryAwareAspectInstanceFactory.class);
            aspectFactoryDef.getPropertyValues().add("aspectBeanName", aspectName);
            aspectFactoryDef.setSynthetic(true);

            // 创建Advice
            AbstractBeanDefinition adviceDef = createAdviceDefinition(
                    adviceElement, parserContext, aspectName, order, methodDefinition, aspectFactoryDef,
                    beanDefinitions, beanReferences);

            // 配置Advicor
            RootBeanDefinition advisorDefinition = new RootBeanDefinition(AspectJPointcutAdvisor.class);
            advisorDefinition.setSource(parserContext.extractSource(adviceElement));
            advisorDefinition.getConstructorArgumentValues().addGenericArgumentValue(adviceDef);
            if (aspectElement.hasAttribute("order")) {
                advisorDefinition.getPropertyValues().add(
                        "order", aspectElement.getAttribute("order"));
            }

            // 向容器中注册Advisor
            parserContext.getReaderContext().registerWithGeneratedName(advisorDefinition);

            return advisorDefinition;
        } finally {
            this.parseState.pop();
        }
    }

1，创建一个用于获取指定aspect实例方法的MethodLocatingFactoryBean对应的BeanDefinition。

2，创建一个用于获取指定aspect实例的SimpleBeanFactoryAwareAspectInstanceFactory对应的BeanDefinition。

3，调用ConfigBeanDefinitionParser解析器的createAdviceDefinition方法创建Advice的BeanDefinition，并注册这个BeanDefinition。

我们看一下是如何创建创建Advice的BeanDefinition

  private AbstractBeanDefinition createAdviceDefinition(
            Element adviceElement, ParserContext parserContext, String aspectName, int order,
            RootBeanDefinition methodDef, RootBeanDefinition aspectFactoryDef,
            List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {

        RootBeanDefinition adviceDefinition = new RootBeanDefinition(getAdviceClass(adviceElement, parserContext));
        adviceDefinition.setSource(parserContext.extractSource(adviceElement));

        adviceDefinition.getPropertyValues().add("aspectName", aspectName);
        adviceDefinition.getPropertyValues().add("declarationOrder", order);

        if (adviceElement.hasAttribute("returning")) {
            adviceDefinition.getPropertyValues().add(
                    "returningName", adviceElement.getAttribute("returning"));
        }
        if (adviceElement.hasAttribute("throwing")) {
            adviceDefinition.getPropertyValues().add(
                    "throwingName", adviceElement.getAttribute("throwing"));
        }
        if (adviceElement.hasAttribute("arg-names")) {
            adviceDefinition.getPropertyValues().add(
                    "argumentNames", adviceElement.getAttribute("arg-names"));
        }

        ConstructorArgumentValues cav = adviceDefinition.getConstructorArgumentValues();
        // 定义有METHOD_INDEX=0
        cav.addIndexedArgumentValue(METHOD_INDEX, methodDef);

        // 解析poincut和pointcut-ref属性
        // 定义有POINTCUT_INDEX = 1
        Object pointcut = parsePointcutProperty(adviceElement, parserContext);
        if (pointcut instanceof BeanDefinition) {
            cav.addIndexedArgumentValue(POINTCUT_INDEX, pointcut);
            beanDefinitions.add((BeanDefinition) pointcut);
        } else if (pointcut instanceof String) {
            RuntimeBeanReference pointcutRef = new RuntimeBeanReference((String) pointcut);
            cav.addIndexedArgumentValue(POINTCUT_INDEX, pointcutRef);
            beanReferences.add(pointcutRef);
        }
        // 定义有ASPECT\_INSTANCE\_FACTORY_INDEX = 2
        cav.addIndexedArgumentValue(ASPECT\_INSTANCE\_FACTORY_INDEX, aspectFactoryDef);

        return adviceDefinition;
    }

createAdviceDefinition方法主要是解析adive标签上的属性值。不过在处理属性之前，还需要判断标签类型是5中advice标签中的哪种，下面是getAdviceClass方法的源码。

private Class<?> getAdviceClass(Element adviceElement, ParserContext parserContext) {
        String elementName = parserContext.getDelegate().getLocalName(adviceElement);
        if ("before".equals(elementName)) {
            return AspectJMethodBeforeAdvice.class;
        } else if ("after".equals(elementName)) {
            return AspectJAfterAdvice.class;
        } else if ("after-returning".equals(elementName)) {
            return AspectJAfterReturningAdvice.class;
        } else if ("after-throwing".equals(elementName)) {
            return AspectJAfterThrowingAdvice.class;
        } else if ("around".equals(elementName)) {
            return AspectJAroundAdvice.class;
        } else {
            throw new IllegalArgumentException("Unknown advice kind \[" + elementName + "\].");
        }
    }

至此，我们就完成了探索spring遇到aop命名空间下的config标签时会创建哪些类型对应的BeanDefiniton，所有的Adivce也注册到了容器中。IOC容器也初始化完成了。