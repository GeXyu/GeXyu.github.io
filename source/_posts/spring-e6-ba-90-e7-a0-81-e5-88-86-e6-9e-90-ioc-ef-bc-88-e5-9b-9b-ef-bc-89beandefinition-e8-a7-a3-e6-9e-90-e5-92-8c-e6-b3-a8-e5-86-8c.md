---
title: Spring源码分析-IOC（四）BeanDefinition解析和注册
comments: true
tags:
  - Spring源码分析-IOC（四）BeanDefinition解析和注册
categories:
  - Spring源码分析-IOC（四）BeanDefinition解析和注册
url: 63.html
id: 63
date: 2017-04-21 17:02:43
---

**BeanDefinition解析**
--------------------

上文说到按照Spring的Bean规则将document转换成BeanDefinition是由BeanDefinitionDocumentReader完成。BeanDefinitionDocumentReader调用了registerBeanDefinitions方法而这个方法又调用了doRegisterBeanDefinitions方法。

protected void doRegisterBeanDefinitions(Element root) {
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String\[\] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI\_VALUE\_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					return;
				}
			}
		}
         // 解析Bean定义之前, 增强解析过程的可扩展性 
		preProcessXml(root);
 // 从Document的根元素开始进行Bean定义的Document对象		
parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}

	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED\_BEANS\_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}

由于篇幅原因我们就看一下Bean标签的的解析

//由BeanDefinitionParserDelegate进行解析，将结果存放到BeanDefinitionHolder
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		//BeanDefinitionHolder是BeanDefinition的封装，封装了BeanDefinition，bean的名字别名等
		//用BeanDefinitionHolder来向IOC容器注册
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				//向IOC容器注册Bean
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}

spring的BeanDefinition解析是在BeanDefinitionParserDelegate完成的。解析完成以后会把解析结果放到BeanDefinition对象中并设置到BeanDefinitionHolder中去。继续向下就会看到对Bean的id name等属性的处理。

//调用入口
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}

	/\*\*
	 \* Parses the supplied {@code <bean>} element. May return {@code null}
	 \* if there were errors during parse. Errors are reported to the
	 \* {@link org.springframework.beans.factory.parsing.ProblemReporter}.
	 */
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
		//取得<Bean>元素中定义的ID name 和Alaiase属性的值
		String id = ele.getAttribute(ID_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

		List<String> aliases = new ArrayList<String>();
		if (StringUtils.hasLength(nameAttr)) {
			String\[\] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI\_VALUE\_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isDebugEnabled()) {
				logger.debug("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}

		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}
		//这个方法是对Bean元素的详细解析
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
				try {
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						// Register an alias for the plain bean class name, if still possible,
						// if the generator returned the class name plus a suffix.
						// This is expected for Spring 1.2/2.0 backwards compatibility.
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name \[" + beanName + "\]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String\[\] aliasesArray = StringUtils.toStringArray(aliases);
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}

上面介绍了对bean元素进行解析的过程，也就是BeanDefinition根据xml对bean定义而被创建的过程。其实这个BeanDefinition就是对Bean的抽象定义。继续看一下如何对Bean元素的解析

public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));
		//这里只读取<Bean>中定义的class名字，然后载入到BeanDefinition
		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}

		try {
			String parent = null;
			if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
				parent = ele.getAttribute(PARENT_ATTRIBUTE);
			}
			//这里生成需要的BeanDefinition对象，为Bean定义信息的载入做准备
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
			//这里对Bean元素进行属性解析
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
			
			//对各种BEAN元素信息进行解析
			parseMetaElements(ele, bd);
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

			parseConstructorArgElements(ele, bd);
			parsePropertyElements(ele, bd);
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class \[" + className + "\] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class \[" + className + "\] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}

 <bean>元素解析已经完了，而<bean>元素属性及其子元素的解析顺序为：

*   1，解析<bean>元素的属性。
*   2，解析<description>子元素。
*   3，解析<meta>子元素。
*   4，解析<lookup-method/>子元素。
*   5，解析<replaced-method>子元素。
*   6，解析<constructor-arg>子元素。
*   7，解析<property>子元素。
*   8，解析<qualifier>子元素。解析过程中像<meta>、<qualifier>等子元素都很少使用。

而下面就直接解析最常用的子元素<property>子元素。这里我们只对Property进行解析，点开parsePropertyElements方法解析property子元素来完成对BeanDefinition载入过程。

	
public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
// 遍历<bean>所有的子元素
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
				parsePropertyElement((Element) node, bd);
			}
		}
	}

	/\*\*
	 \* Parse a property element.
	 */
	public void parsePropertyElement(Element ele, BeanDefinition bd) {
 // <property>元素name属性
		String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
		if (!StringUtils.hasLength(propertyName)) {
			error("Tag 'property' must have a 'name' attribute", ele);
			return;
		}
		this.parseState.push(new PropertyEntry(propertyName));
		try {
// 如果同一个Bean中已经有相同名字的<property>存在, 直接返回  
// 也就是说, 如果一个Bean中定义了两个名字一样的<property>元素, 只有第一个起作用.
			if (bd.getPropertyValues().contains(propertyName)) {
				error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
				return;
			}
// 解析<property>元素, 返回的对象对应<property>元素的解析结果, 最终封装到PropertyValue中, 并设置到BeanDefinitionHolder中 
			Object val = parsePropertyValue(ele, bd, propertyName);
			PropertyValue pv = new PropertyValue(propertyName, val);
			parseMetaElements(ele, pv);
			pv.setSource(extractSource(ele));
			bd.getPropertyValues().addPropertyValue(pv);
		}
		finally {
			this.parseState.pop();
		}
	}
/\*\*
	 \* Get the value of a property element. May be a list etc.
	 \* Also used for constructor arguments, "propertyName" being null in this case.
	 */
	public Object parsePropertyValue(Element ele, BeanDefinition bd, String propertyName) {
		String elementName = (propertyName != null) ?
						"<property> element for property '" + propertyName + "'" :
						"<constructor-arg> element";

		 // 检查<property>的子元素, 只能是ref, value, list等(description, meta除外)其中的一个. 
		NodeList nl = ele.getChildNodes();
		Element subElement = null;
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
					!nodeNameEquals(node, META_ELEMENT)) {
				// Child element is what we're looking for.
				if (subElement != null) {
					error(elementName + " must not contain more than one sub-element", ele);
				}
				else {
					subElement = (Element) node;
				}
			}
		}
// 判断property元素是否含有ref和value属性, 不允许既有ref又有value属性.  
// 同时也不允许ref和value属性其中一个与子元素共存.  
		boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
		boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
		if ((hasRefAttribute && hasValueAttribute) ||
				((hasRefAttribute || hasValueAttribute) && subElement != null)) {
			error(elementName +
					" is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
		}

 // 如果属性是ref属性, 创建一个ref的数据对象RuntimeBeanReference, 封装了ref信		
if (hasRefAttribute) {
			String refName = ele.getAttribute(REF_ATTRIBUTE);
			if (!StringUtils.hasText(refName)) {
				error(elementName + " contains empty 'ref' attribute", ele);
			}
			RuntimeBeanReference ref = new RuntimeBeanReference(refName);
			ref.setSource(extractSource(ele));
			return ref;
		}
// 如果属性是value属性, 创建一个数据对象TypedStringValue, 封装了value信息 
		else if (hasValueAttribute) {
			TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
			valueHolder.setSource(extractSource(ele));
			return valueHolder;
		}
// 如果当前<property>元素还有子元素 
		else if (subElement != null) {
			return parsePropertySubElement(subElement, bd);
		}
		else {
 // 以上都不是, 说明配置错误, 返回null. 
			// Neither child element nor "ref" or "value" attribute found.
			error(elementName + " must specify a ref or value", ele);
			return null;
		}
	}

如果property子元素还有子元素

public Object parsePropertySubElement(Element ele, BeanDefinition bd, String defaultValueType) {
 // 如果子元素没有使用Spring默认命名空间, 则使用用户自定义的规则解析 
		if (!isDefaultNamespace(ele)) {
			return parseNestedCustomElement(ele, bd);
		}
// 如果子元素是bean元素, 则使用解析<bean>元素的方法解析  
		else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
			BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
			if (nestedBd != null) {
				nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
			}
			return nestedBd;
		}
 // 如果子元素是ref, 有且只能有bean、local和parent 3个属性中的一个
		else if (nodeNameEquals(ele, REF_ELEMENT)) {
			// A generic reference to any name of any bean.
// 引用普通任意的bean
			String refName = ele.getAttribute(BEAN\_REF\_ATTRIBUTE);
			boolean toParent = false;
			if (!StringUtils.hasLength(refName)) {
				// A reference to the id of another bean in the same XML file.
                  // 引用同一个XML配置文件中的bean 
				refName = ele.getAttribute(LOCAL\_REF\_ATTRIBUTE);
				if (!StringUtils.hasLength(refName)) {
					// A reference to the id of another bean in a parent context.
                       // 引用父容器中的bean 
					refName = ele.getAttribute(PARENT\_REF\_ATTRIBUTE);
					toParent = true;
					if (!StringUtils.hasLength(refName)) {
						error("'bean', 'local' or 'parent' is required for <ref> element", ele);
						return null;
					}
				}
			}
// ref元素没有bean、local和parent 3个属性中的一个, 返回null. 
			if (!StringUtils.hasText(refName)) {
				error("<ref> element contains empty target attribute", ele);
				return null;
			}
			RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
			ref.setSource(extractSource(ele));
			return ref;
		}
		else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
			return parseIdRefElement(ele);
		}
		else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
			return parseValueElement(ele, defaultValueType);
		}
		else if (nodeNameEquals(ele, NULL_ELEMENT)) {
			// It's a distinguished null value. Let's wrap it in a TypedStringValue
			// object in order to preserve the source location.
			TypedStringValue nullHolder = new TypedStringValue(null);
			nullHolder.setSource(extractSource(ele));
			return nullHolder;
		}
		else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
			return parseArrayElement(ele, bd);
		}
//如果是List子元素
		else if (nodeNameEquals(ele, LIST_ELEMENT)) {
			return parseListElement(ele, bd);
		}
		else if (nodeNameEquals(ele, SET_ELEMENT)) {
			return parseSetElement(ele, bd);
		}
		else if (nodeNameEquals(ele, MAP_ELEMENT)) {
			return parseMapElement(ele, bd);
		}
		else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
			return parsePropsElement(ele);
		}
		else {
			error("Unknown property sub-element: \[" + ele.getNodeName() + "\]", ele);
			return null;
		}
	}

我们以解析List为例

/\*\*
	 \* Parse a list element.
	 */
	public List<Object> parseListElement(Element collectionEle, BeanDefinition bd) {
// 获取<list>元素中的value-type属性, 即集合元素的数据类型		
String defaultElementType = collectionEle.getAttribute(VALUE\_TYPE\_ATTRIBUTE);
		
// <list>元素所有子节点
NodeList nl = collectionEle.getChildNodes();
		ManagedList<Object> target = new ManagedList<Object>(nl.getLength());
		target.setSource(extractSource(collectionEle));
		target.setElementTypeName(defaultElementType);
		target.setMergeEnabled(parseMergeAttribute(collectionEle));

// 具体解析List元素中的值
		parseCollectionElements(nl, target, bd, defaultElementType);
		return target;
	}

	/\*\*
	 \* Parse a set element.
	 */
	protected void parseCollectionElements(
			NodeList elementNodes, Collection<Object> target, BeanDefinition bd, String defaultElementType) {
// 遍历集合所有子节点
		for (int i = 0; i < elementNodes.getLength(); i++) {
			Node node = elementNodes.item(i);
			if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT)) {
// 如果子节点是Element且不是<description>节点, 则添加进ManagedList.  
// 同时触发对下层子元素的解析, 递归调用.
				target.add(parsePropertySubElement((Element) node, bd, defaultElementType));
			}
		}
	}

经过这样逐层的分析，我们在xml文件中定义的BeanDefinition就被整个载入到IOC容器中，并在容器中建立了数据映射。在IOC容器中建立了对应的数据结构，或者说可以看成POJO对象在IOC容器中的抽象，这些数据结构可以以AbstractBeanDefinition为入口让IOC容器执行索引，查询和操作。

**BeanDefinition注册**
--------------------

**DefaultBeanDefinitionDocumentReader****的**processBeanDefinition方法对Bean进行回调注册

protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 注册BeanDefinition
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}

点开registerBeanDefinition方法

public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
// 向IoC容器注册BeanDefinition
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// 如果解析的BeanDefinition有别名, 向容器为其注册别名. .
		String\[\] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String aliase : aliases) {
				registry.registerAlias(beanName, aliase);
			}
		}
	}

上面的registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition())是调用的注册位置，而BeanDefinitionRegistry仅仅是一个接口，而真正实现它的却是最原本的DefaultListableBeanFactory.registerBeanDefinition方法

public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");
// 对解析得到的BeanDefinition校验
		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition oldBeanDefinition;

		oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		if (oldBeanDefinition != null) {
			if (!this.allowBeanDefinitionOverriding) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition \[" + beanDefinition + "\] for bean '" + beanName +
						"': There is already \[" + oldBeanDefinition + "\] bound.");
			}
			else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE\_APPLICATION, now overriding with ROLE\_SUPPORT or ROLE_INFRASTRUCTURE
				if (this.logger.isWarnEnabled()) {
					this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
							" with a framework-generated bean definition ': replacing \[" +
							oldBeanDefinition + "\] with \[" + beanDefinition + "\]");
				}
			}
			else {
				if (this.logger.isInfoEnabled()) {
					this.logger.info("Overriding bean definition for bean '" + beanName +
							"': replacing \[" + oldBeanDefinition + "\] with \[" + beanDefinition + "\]");
				}
			}
		}
		else {
			this.beanDefinitionNames.add(beanName);
			this.manualSingletonNames.remove(beanName);
			this.frozenBeanDefinitionNames = null;
		}
		this.beanDefinitionMap.put(beanName, beanDefinition);

		if (oldBeanDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}

完成了BeanDefinition的注册，就完成了IOC容器的初始化过程，此时使用的IOC容器DefinListableBeanFactory中已经建立了整个Bean的配置信息，而且这些BeanDefinition已经可以被容器使用了，他们都BeanDefintionMap被检索和使用。容器的作用就是对这些信息进行处理和维护，这些信息就是容器建立依赖反转的基础。