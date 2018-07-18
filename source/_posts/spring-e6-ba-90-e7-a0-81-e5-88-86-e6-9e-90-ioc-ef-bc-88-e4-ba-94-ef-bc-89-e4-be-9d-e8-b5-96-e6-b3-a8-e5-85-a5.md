---
title: Spring源码分析-IOC（五）依赖注入
comments: true
tags:
  - Spring源码分析-IOC（五）依赖注入
categories:
  - Spring源码分析-IOC（五）依赖注入
url: 65.html
id: 65
date: 2017-04-22 09:05:04
---

上篇文章看完的同学，估计已经了解了ioc容器是如何初始化并载入了用户定义的Bean信息。那么依赖注入是什么时候触发的？其实依赖注入是当用户getBean的时候触发依赖注入，当然也有例外，就是当Bean配置了lazy-init的时候，依赖注入在ioc容器初始化的时候注入。可以从测试类的getBean出发去看看具体是的实现。

**依赖注入**
--------

//调用了AbstractBeanFactory的getBean方法
@Override
	public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
		return doGetBean(name, requiredType, null, false);
	}
@SuppressWarnings("unchecked")
// doGetBean方法是实际触发依赖注入的地方
	protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object\[\] args, boolean typeCheckOnly)
			throws BeansException {
//指定名称获取被管理的Bean名称，并且指定名称中对容器的依赖
//如果是别名，将别名转变为Bean的名称
		final String beanName = transformedBeanName(name);
		Object bean;

		//先缓存取得的Bean以及处理创建过的单件模式的Bean
        //单态模式的Bean只能创建一次.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
//如果指定名称的Bean在容器中已经存在，则直接返回
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
              //这里完成FactoryBean的相关处理
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			//缓存中如果已经存在创建的单态模式，因为循环而创建失败，则抛出异常
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			//对容器中的BeanDefinition进行检查，检查能否在当前的BeanFactory中取得Bean.
//如果在当前的工厂中取不到，则到父类的BeanFactory中去取
//如果在父类中取不到，则到父类的父类中取.
			BeanFactory parentBeanFactory = getParentBeanFactory();
//当前容器的父容器存在，且当前容器中不存在指名的Bean
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				//  //解析Bean的原始名.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// //委派给父类容器查找，根据指定的名称和显示参数.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// //委派给父类容器查找，根据指定的名称和类型.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}
//创建的Bean是否需要进行验证，一般不需要
			if (!typeCheckOnly) {
                   //向容器指定的Bean已被创建
				markBeanAsCreated(beanName);
			}

			try {
                 //根据Bean的名字获取BeanDefinition
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				//获取当前Bean的所有依赖的名称.
				String\[\] dependsOn = mbd.getDependsOn();
//如果当前Bean有依赖Bean
				if (dependsOn != null) {
					for (String dependsOnBean : dependsOn) {
						if (isDependent(beanName, dependsOnBean)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
						}
                             //把被依赖的Bean注册给当前依赖的Bean
						registerDependentBean(dependsOnBean, beanName);
                            //触发递归
						getBean(dependsOnBean);
					}
				}

				//创建单态模式的Bean实例对象.
				if (mbd.isSingleton()) {
                       //使用一个内部匿名类，创建Bean实例对象，并且注册对所依赖的对象
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
                                   //创建一个指定Bean实例对象，如果有父级继承，则合并资对象
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.
//清除显示容器单例模式Bean缓存中的实例对象
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
                            //获取给定Bean实例对象
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
                       // 判断是否是原型模式
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
                       //原型模式会每次创建一个新的对象
					Object prototypeInstance = null;
					try {
                            //回调方法，注册原型对象
						beforePrototypeCreation(beanName);
                            //创建指定Bean对象实例
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
                           //回调方法，Bean无法再次创建
						afterPrototypeCreation(beanName);
					}
                       //获取给定Bean的实例对象
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
                      //Bean定义资源中没有配置生命周期范围，则Bean不合法
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
					}
					try {
                       //这里使用一个匿名类，获取一个指定的生命周期范围
						Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
							@Override
							public Object getObject() throws BeansException {
								beforePrototypeCreation(beanName);
								try {
									return createBean(beanName, mbd, args);
								}
								finally {
									afterPrototypeCreation(beanName);
								}
							}
						});
                         //获取给定Bean的实例对象
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; " +
								"consider defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

	// 对创建的Bean进行类型检查，如果没有问题，就返回这个新建的Bean，这个Bean已经包含了依赖关系.
		if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
			try {
				return getTypeConverter().convertIfNecessary(bean, requiredType);
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type \[" +
							ClassUtils.getQualifiedName(requiredType) + "\]", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}

从上面，可以看出doGetBean，是依赖注入的实际入口，他定义了Bean的定义模式，单例模式（Singleton）和原型模（Prototype），而依赖注入触发的前提是BeanDefinition数据已经建立好的前提下。其实对于Ioc容器的使用，Spring提供了许多的参数配置，每一个参数配置实际上代表了一个Ioc实现特性，而这些特性的实现很多都需要在依赖注入的过程中或者对Bean进行生命周期管理的过程中完成。Spring Ioc容器作为一个产品，其真正的价值体现在一系列产品特征上，而这些特征都是以依赖反转模式作为核心，他们为控制反转提供了很多便利，从而实现了完整的Ioc容器。

### **bean****实例的创建和初始化：****createBean**

getBean是依赖注入的起点，之后就会调用createBean，createBean可以生成需要的Bean以及对Bean进行初始化，但对createBean进行跟踪，发现他在AbstractBeanFactory中仅仅是声明，而具体实现是在AbstractAutowireCapableBeanFactory类里。

@Override
	protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object\[\] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		//判断创建的Bean是否需要实例化，以及这个类是否需要通过类来装载.
		resolveBeanClass(mbd, beanName);

		//校验和准备中的方法覆盖.
		try {
			mbd.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbd.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			//如果Bean配置了初始化前和后的处理器，则返回一个Bean对象.
			Object bean = resolveBeforeInstantiation(beanName, mbd);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

 //创建Bean的入口		
Object beanInstance = doCreateBean(beanName, mbd, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}

//Bean的真正创建位置
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object\[\] args) {
		/// 这个BearWrapper是用来封装创建出来的Bean对象的.
		BeanWrapper instanceWrapper = null;
// 如果是单态模式，就先把缓存中同名的Bean清除
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
         //创建Bean，由createBeanInstance来完成
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

		// Allow post-processors to modify the merged bean definition.
           //调用PostProcessor后置处理器
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
          //缓存单态模式的Bean对象，以防循环引用
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
               //匿名类，防止循环引用，可尽早的持有引用对象
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
		}

		// Initialize the bean instance.
//初始化Bean的地方，依赖注入发生的地方
        //exposedObject在初始化完成后，会作为依赖注入完成后的Bean
		Object exposedObject = bean;
		try {
              //将Bean实例对象进行封装，并将Bean定义的配置属性赋值给实例对象
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
                   //初始化Bean
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
              //获取指定名称的单态模式对象
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
                  //判断注册的实例化Bean和正在实例化的Bean是同一个
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
             //当前Bean依赖其他Bean，并且当发生循环引用时不允许创建新实例
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String\[\] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
                        //获取当前Bean的所有依赖
					for (String dependentBean : dependentBeans) {
                                 //对依赖bean进行检查
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans \[" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"\] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		//注册完成依赖注入的Bean.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}

在对doCreateBean的追踪中我们发现Bean的创建方法createBeanInstance与BeanDefinition的载入与解析方法populateBean方法是最为重要的。因为控制反转原理的实现就是在这两个方法中实现的。

### **生成Bean中的对象：createBeanInstance**

在createBeanInstance中生成了Bean所包含的Java对象，对象的生成有很多种不同的方法：工厂方法+反射，容器的autowire特性等等，这些生成方法都是由相关BeanDefinition来指定的。

protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object\[\] args) {
		// Make sure bean class is actually resolved at this point.
         // 确认需要创建的Bean实例的类可以实例化
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
        
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
         //这里使用工厂方法对Bean进行实例化
		if (mbd.getFactoryMethodName() != null)  {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
        //使用容器自动装配方法进行实例化
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
                  //配置自动装配属性，使用容器自动实例化
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
                   //无参构造方法实例化
				return instantiateBean(beanName, mbd);
			}
		}

		// Need to determine the constructor...
         //使用构造函数进行实例化
		Constructor<?>\[\] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
//使用容器自动装配，调用构造方法实例化			
return autowireConstructor(beanName, mbd, ctors, args);
		}

		// No special handling: simply use no-arg constructor.
          //使用默认构造函数对Bean进行实例化
		return instantiateBean(beanName, mbd);
	}
/\*\*
 \* CGLIB是一个常用的字节码生成器的类库，他提供了一系列的API来提供生成和转换Java的字节码功能。
 \* 在Spring AOP中也使用CGLIB对Java的字节码进行了增强。
 */ 
private InstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();

protected InstantiationStrategy getInstantiationStrategy() {
        return this.instantiationStrategy;
}

/\*\*
 \* 最常见的实例化过程instantiateBean
 \* 使用默认的实例化策略对Bean进行实例化，默认的实例化策略是
 \* CglibSubclassingInstantiationStrategy，也就是用CGLIB来对Bean进行实例化
 */
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
             //获取系统的安全管理接口
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
					@Override
					public Object run() {
						return getInstantiationStrategy().instantiate(mbd, beanName, parent);
					}
				}, getAccessControlContext());
			}
			else {
                 //实例化的对象封装
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
	}

我们对CglibSubclassingInstantiationStrategy进行跟踪，发现Spring中的CGLIB生成，是由SimpleInstantiationStrategy.instantiate方法来完成的，所以我们就直接看SimpleInstantiationStrategy.instantiate

@Override
	public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
		// Don't override the class with CGLIB if no overrides.
		if (bd.getMethodOverrides().isEmpty()) {
              /指定构造器或者生成的对象工厂方法来对Bean进行实例化
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
								@Override
								public Constructor<?> run() throws Exception {
									return clazz.getDeclaredConstructor((Class\[\]) null);
								}
							});
						}
						else {
							constructorToUse =	clazz.getDeclaredConstructor((Class\[\]) null);
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Exception ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
              //通过BeanUtils进行实例化，这个BeanUtils的实例化通过Constructor来完成
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.
               //使用CGLIB来实例化对象
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}

从上面可以看到**SimpleInstantiationStrategy**是sprng用来生成Bean对象的默认类，它提供了两种方式一种是BeanUtils，它是使用了jvm的反射机制，另外一种就是cglib。

### **属性依赖注入实现：****populateBean**

Bean对象进行实例化以后。怎么把这些Bean对象之间的依赖关系处理好，以完成整个依赖注入，而这里就涉及到各种Bean对象依赖关系的处理过程了，而这些依赖关系都已经解析到了BeanDefinition。如果要仔细理解这个过程，我们必须从前面提到的populateBean方法入手。

protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
		//这里取得BeanDefinition中的property值
		PropertyValues pvs = mbd.getPropertyValues();
		//实例对象为NULL
		if (bw == null) {
			if (!pvs.isEmpty()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				//实例对象为NULL，属性值也为空，不需设置，直接返回
				return;
			}
		}


		//在设置属性之前调用Bean的PostProcessor后置处理器  
		boolean continueWithPropertyPopulation = true;

		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}

		if (!continueWithPropertyPopulation) {
			return;
		}
		//开始进行依赖注入过程，先处理autowire的注入
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE\_BY\_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE\_BY\_TYPE) {
			
			
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			//根据Bean的名字或者类型自动装配处理
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE\_BY\_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			 //根据类型自动装配注入
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE\_BY\_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}
		//检查日期是否持有用于单态模式Bean关闭时的后置处理器
		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		//Bean实例对象没有依赖，也没有继承
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY\_CHECK\_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			 //从实例对象中提取属性描述符
			PropertyDescriptor\[\] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						//使用BeanPostProcessor处理器处理属性值  
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
				//对要配置的属性进行依赖检查
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}
		 //对属性进行依赖注入
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
	
	protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		if (pvs == null || pvs.isEmpty()) {
			return;
		}
		//封装属性值
		MutablePropertyValues mpvs = null;
		List<PropertyValue> original;

		if (System.getSecurityManager() != null) {
			if (bw instanceof BeanWrapperImpl) {
				//设置安全上下文，JDK安全机制  
				((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
			}
		}

		if (pvs instanceof MutablePropertyValues) {
			mpvs = (MutablePropertyValues) pvs;
			//属性值已经转换  
			if (mpvs.isConverted()) {
				// Shortcut: use the pre-converted values as-is.
				try {
					//为实例化对象设置属性值  
					bw.setPropertyValues(mpvs);
					return;
				}
				catch (BeansException ex) {
					throw new BeanCreationException(
							mbd.getResourceDescription(), beanName, "Error setting property values", ex);
				}
			}
			 //获取属性值对象的原始类型值  
			original = mpvs.getPropertyValueList();
		}
		else {
			original = Arrays.asList(pvs.getPropertyValues());
		}

		 //获取用户自定义的类型转换  
		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}
		
		//创建一个Bean定义属性值解析器，将Bean定义中的属性值解析为Bean实例对象  的实际值 
		BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);


		//这里为解析值创建的一个副本，然后通过副本注入Bean
		List<PropertyValue> deepCopy = new ArrayList<PropertyValue>(original.size());
		boolean resolveNecessary = false;
		//转换属性值
		for (PropertyValue pv : original) {
			if (pv.isConverted()) {
				deepCopy.add(pv);
			}
			else {
				String propertyName = pv.getName();
				 //原始的属性值，即转换之前的属性值  
				Object originalValue = pv.getValue();
                   //转换属性值，例如将引用转换为IoC容器中实例化对象引用（属性值解析）				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				if (convertible) {
					 //使用用户自定义的类型转换器转换属性值  
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}
				// Possibly store converted value in merged bean definition,
				// in order to avoid re-conversion for every created bean instance.
				
				//存储转换后的属性值，避免每次属性注入时的转换工作  
				if (resolvedValue == originalValue) {
					if (convertible) {
						pv.setConvertedValue(convertedValue);
					}
					deepCopy.add(pv);
				}
				
				//属性是可转换的，且属性原始值是字符串类型，且属性的原始类型值不是  
                //动态生成的字符串，且属性的原始值不是集合或者数组类型  
				else if (convertible && originalValue instanceof TypedStringValue &&
						!((TypedStringValue) originalValue).isDynamic() &&
						!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
					pv.setConvertedValue(convertedValue);
					deepCopy.add(pv);
				}
				else {
					resolveNecessary = true;
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
		if (mpvs != null && !resolveNecessary) {
			mpvs.setConverted();
		}

		// 这里是依赖注入发生的地方
		try {
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Error setting property values", ex);
		}
	}

以上就是属性依赖注入的过程，我们再看一下属性值的解析，点开valueResolver.resolveValueIfNecessary()方法。

### **属性值的解析：****resolveValueIfNecessary**

进而看一下BeanDefinitionValueResolver的resolveValueIfNecessary

public Object resolveValueIfNecessary(Object argName, Object value) {
		// We must check each value to see whether it requires a runtime reference
		// to another bean to be resolved.
         //对引用类型的属性进行解析
		if (value instanceof RuntimeBeanReference) {
			RuntimeBeanReference ref = (RuntimeBeanReference) value;
              //调用引用类型属性的解析方法
			return resolveReference(argName, ref);
		}
//对属性值是引用容器中另一个Bean名称的解析		
else if (value instanceof RuntimeBeanNameReference) {
			String refName = ((RuntimeBeanNameReference) value).getBeanName();
			refName = String.valueOf(doEvaluate(refName));
              //从容器中获取指定名称的Bean 
			if (!this.beanFactory.containsBean(refName)) {
				throw new BeanDefinitionStoreException(
						"Invalid bean name '" + refName + "' in bean reference for " + argName);
			}
			return refName;
		}
          //对Bean类型属性的解析，主要是Bean中的内部类 
		else if (value instanceof BeanDefinitionHolder) {
			// Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
			BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
			return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
		}
        		else if (value instanceof BeanDefinition) {
			// Resolve plain BeanDefinition, without contained name: use dummy name.
			BeanDefinition bd = (BeanDefinition) value;
			String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED\_BEAN\_NAME_SEPARATOR +
					ObjectUtils.getIdentityHexString(bd);
			return resolveInnerBean(argName, innerBeanName, bd);
		}
            //对集合数组类型的属性解析 

		else if (value instanceof ManagedArray) {
			// May need to resolve contained runtime references.
			ManagedArray array = (ManagedArray) value;
             //获取数组的类型  
			Class<?> elementType = array.resolvedElementType;
			if (elementType == null) {
                   //获取数组元素的类型  
				String elementTypeName = array.getElementTypeName();
				if (StringUtils.hasText(elementTypeName)) {
					try {
                            //使用反射机制创建指定类型的对象 
						elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
						array.resolvedElementType = elementType;
					}
					catch (Throwable ex) {
						// Improve the message by showing the context.
						throw new BeanCreationException(
								this.beanDefinition.getResourceDescription(), this.beanName,
								"Error resolving array type for " + argName, ex);
					}
				}
				else {
                     //没有获取到数组的类型，也没有获取到数组元素的类型，则直接设置数 组的类型为Object
					elementType = Object.class;
				}
			}
//创建指定类型的数组 
			return resolveManagedArray(argName, (List<?>) value, elementType);
		}
           
		else if (value instanceof ManagedList) {
			// May need to resolve contained runtime references.
			return resolveManagedList(argName, (List<?>) value);
		}
		else if (value instanceof ManagedSet) {
			// May need to resolve contained runtime references.
			return resolveManagedSet(argName, (Set<?>) value);
		}
		else if (value instanceof ManagedMap) {
			// May need to resolve contained runtime references.
			return resolveManagedMap(argName, (Map<?, ?>) value);
		}

           //解析props类型的属性值，props其实就是key和value均为字符串的map  
		else if (value instanceof ManagedProperties) {
			Properties original = (Properties) value;
              //创建一个拷贝，用于作为解析后的返回值
			Properties copy = new Properties();
			for (Map.Entry<Object, Object> propEntry : original.entrySet()) {
				Object propKey = propEntry.getKey();
				Object propValue = propEntry.getValue();
				if (propKey instanceof TypedStringValue) {
					propKey = evaluate((TypedStringValue) propKey);
				}
				if (propValue instanceof TypedStringValue) {
					propValue = evaluate((TypedStringValue) propValue);
				}
				copy.put(propKey, propValue);
			}
			return copy;
		}
            //解析字符串类型的属性值  
		else if (value instanceof TypedStringValue) {
			// Convert value to target type here.
			TypedStringValue typedStringValue = (TypedStringValue) value;
			Object valueObject = evaluate(typedStringValue);
			try {
                    //获取属性的目标类型  
				Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
				if (resolvedTargetType != null) {
					return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
				}
				else {
                        //没有获取到属性的目标对象，则按Object类型返回 
					return valueObject;
				}
			}
			catch (Throwable ex) {
				// Improve the message by showing the context.
				throw new BeanCreationException(
						this.beanDefinition.getResourceDescription(), this.beanName,
						"Error converting typed String value for " + argName, ex);
			}
		}
		else {
			return evaluate(value);
		}
	}

### **属性值的依赖注入setPropertyValue**

点开AbstractAutowireCapableBeanFactory类的applyPropertyValues方法最后setPropertyValues  实现类AbstractPropertyAccessor

@Override
	public void setPropertyValue(PropertyValue pv) throws BeansException {
		setPropertyValue(pv.getName(), pv.getValue());
	}
@Override
	public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid)
			throws BeansException {

		List<PropertyAccessException> propertyAccessExceptions = null;
		List<PropertyValue> propertyValues = (pvs instanceof MutablePropertyValues ?
				((MutablePropertyValues) pvs).getPropertyValueList() : Arrays.asList(pvs.getPropertyValues()));
		for (PropertyValue pv : propertyValues) {
			try {
				// This method may throw any BeansException, which won't be caught
				// here, if there is a critical failure such as no matching field.
				// We can attempt to deal only with less serious exceptions.
				setPropertyValue(pv);
			}
			catch (NotWritablePropertyException ex) {
				if (!ignoreUnknown) {
					throw ex;
				}
				// Otherwise, just ignore it and continue...
			}
			catch (NullValueInNestedPathException ex) {
				if (!ignoreInvalid) {
					throw ex;
				}
				// Otherwise, just ignore it and continue...
			}
			catch (PropertyAccessException ex) {
				if (propertyAccessExceptions == null) {
					propertyAccessExceptions = new LinkedList<PropertyAccessException>();
				}
				propertyAccessExceptions.add(ex);
			}
		}

		// If we encountered individual exceptions, throw the composite exception.
		if (propertyAccessExceptions != null) {
			PropertyAccessException\[\] paeArray =
					propertyAccessExceptions.toArray(new PropertyAccessException\[propertyAccessExceptions.size()\]);
			throw new PropertyBatchUpdateException(paeArray);
		}
	}

好了，自此，依赖注入部分就完了。