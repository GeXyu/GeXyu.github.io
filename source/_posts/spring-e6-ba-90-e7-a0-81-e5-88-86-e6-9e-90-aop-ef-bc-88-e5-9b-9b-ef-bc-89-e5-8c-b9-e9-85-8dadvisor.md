---
title: Spring源码分析-AOP（四）匹配Advisor
comments: true
tags:
  - Spring源码分析-AOP（四）匹配Advisor
categories:
  - Spring源码分析-AOP（四）匹配Advisor
url: 113.html
id: 113
date: 2017-04-27 15:01:56
---

上文中讲到了如何解析config标签，并把XML文件中定义的属性设置到IOC容器中。在[Spring源码分析-AOP（二）入门](http://www.zzcode.cn/springaop/thread-83.html)文章中讲到了在解析config标签之前，先注册一个AspectJAwareAdvisorAutoProxyCreator类型的类。从这个类的继承关系中看到它实现了BeanPostProcessor接口。

BeanPostProcessor接口作用是：**如果我们需要在Spring容器完成Bean的实例化、配置和其他的初始化前后添加一些自己的逻辑处理，我们就可以定义一个或者多个BeanPostProcessor接口的实现，然后注册到容器中。**所以在创建类之前会调用BeanPostProcessor的postProcessBeforeInstantiation方法。

**当调用者通过 getBean（ name ）向 容器寻找Bean 时，如果容器注册了org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor接口，在实例 bean 之前，将调用该接口的 postProcessBeforeInstantiation （）方法。AspectJAwareAdvisorAutoProxyCreator实现了InstantiationAwareBeanPostProcessor。所以当getBean时就会执行 postProcessBeforeInstantiation方法。**

public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        Object cacheKey = getCacheKey(beanClass, beanName);
 
        if (beanName == null || !this.targetSourcedBeans.contains(beanName)) {
            if (this.advisedBeans.containsKey(cacheKey)) {
                return null;
            }
            if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
                this.advisedBeans.put(cacheKey, Boolean.FALSE);
                return null;
            }
        }
 
        // Create proxy here if we have a custom TargetSource.
        // Suppresses unnecessary default instantiation of the target bean:
        // The TargetSource will handle target instances in a custom fashion.
        if (beanName != null) {
            TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
            if (targetSource != null) {
                this.targetSourcedBeans.add(beanName);
                //获取所有的通知
                Object\[\] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
                //把所有的通知配置到代理类
                Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
                this.proxyTypes.put(cacheKey, proxy.getClass());
                return proxy;
            }
        }
 
        return null;
    }

getAdvicesAndAdvisorsForBean该方法是AbstractAutoProxyCreator的抽象方法，实现由子类实现。从继承的关系来看，这里的子类就是AbstractAdvisorAutoProxyCreator。

       //调用入口
	protected Object\[\] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {
		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO\_NOT\_PROXY;
		}
		return advisors.toArray();
	}
//查找适合的Advisors
protected List<Advisor> findEligibleAdvisors(Class beanClass, String beanName) {
   List<Advisor> candidateAdvisors = findCandidateAdvisors();
   List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
   extendAdvisors(eligibleAdvisors);//交给子类实现扩展
   if (!eligibleAdvisors.isEmpty()) {
      eligibleAdvisors = sortAdvisors(eligibleAdvisors);
   }
   return eligibleAdvisors;
}

### **解析getAdvicesAndAdvisorsForBean**

protected List<Advisor> findEligibleAdvisors(Class beanClass, String beanName) {
   List<Advisor> candidateAdvisors = findCandidateAdvisors();//获取所有的Advisors
   if (advisors.isEmpty()) {
			return DO\_NOT\_PROXY;
		}
   return advisors.toArray();
}

protected List<Advisor> findCandidateAdvisors() {
	    //委托给BeanFactoryAdvisorRetrievalHelper处理
		return this.advisorRetrievalHelper.findAdvisorBeans();
	}
	
	public List<Advisor> findAdvisorBeans() {
		// Determine list of advisor bean names, if not cached already.
		String\[\] advisorNames = null;
		synchronized (this) {
			advisorNames = this.cachedAdvisorBeanNames;
			if (advisorNames == null) {
				// Do not initialize FactoryBeans here: We need to leave all regular beans
				// uninitialized to let the auto-proxy creator apply to them!
				//获取容器中声明的Advisor
				advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
						this.beanFactory, Advisor.class, true, false);
				this.cachedAdvisorBeanNames = advisorNames;
			}
		}
		if (advisorNames.length == 0) {
			return new LinkedList<Advisor>();
		}

		List<Advisor> advisors = new LinkedList<Advisor>();
		for (String name : advisorNames) {
			//判断返回
			if (isEligibleBean(name)) {
				if (this.beanFactory.isCurrentlyInCreation(name)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipping currently created advisor '" + name + "'");
					}
				}
				else {
					try {
						advisors.add(this.beanFactory.getBean(name, Advisor.class));
					}
					catch (BeanCreationException ex) {
						Throwable rootCause = ex.getMostSpecificCause();
						if (rootCause instanceof BeanCurrentlyInCreationException) {
							BeanCreationException bce = (BeanCreationException) rootCause;
							if (this.beanFactory.isCurrentlyInCreation(bce.getBeanName())) {
								if (logger.isDebugEnabled()) {
									logger.debug("Skipping advisor '" + name +
											"' with dependency on currently created bean: " + ex.getMessage());
								}
								// Ignore: indicates a reference back to the bean we're trying to advise.
								// We want to find advisors other than the currently created bean itself.
								continue;
							}
						}
						throw ex;
					}
				}
			}
		}
		return advisors;
	}

从上面的代码可以看到是如何获取所有的Advisor。回到AbstractAdvisorAutoProxyCreator的findAdvisorsThatCanApply方法，看他是如何进行匹配的。

**匹配​findAdvisorsThatCanApply**

protected List<Advisor> findAdvisorsThatCanApply(
			List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

		ProxyCreationContext.setCurrentProxiedBeanName(beanName);
		try {
			//定位到AopUtils.findAdvisorsThatCanApply
			return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
		}
		finally {
			ProxyCreationContext.setCurrentProxiedBeanName(null);
		}
	}
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
		List<Advisor> eligibleAdvisors = new LinkedList<Advisor>();
		//遍历
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				//是否IntroductionAdvisor
				eligibleAdvisors.add(candidate);
			}
		}
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}
			//执行
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		return eligibleAdvisors;
	}
	
	public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
		//如果是IntroductionAdvisor则调用AspectJExpressionPointcut的matches方法
		if (advisor instanceof IntroductionAdvisor) {
			return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
		}
		//如果是PointcutAdvisor则调用AspectJExpressionPointcut的matches方法（与上面相比非同一个方法）
		else if (advisor instanceof PointcutAdvisor) {
			PointcutAdvisor pca = (PointcutAdvisor) advisor;
			return canApply(pca.getPointcut(), targetClass, hasIntroductions);
		}
		else {
			// It doesn't have a pointcut so we assume it applies.
			return true;
		}
	}

![1](http://www.zzcode.cn/wp-content/uploads/2017/04/1-2.png)

这里的AspectJExpressionPointcut同时实现了MethodMatcher和ClassFilter两个接口。

public class AspectJExpressionPointcut extends AbstractExpressionPointcut
      implements ClassFilter, IntroductionAwareMethodMatcher, BeanFactoryAware {
       
      //ClassFilter实现
   public boolean matches(Class targetClass) {
           checkReadyToMatch();
       try {
          return this.pointcutExpression.couldMatchJoinPointsInType(targetClass);
       } catch (ReflectionWorldException e) {
          logger.debug("PointcutExpression matching rejected target class", e);  
       }
 }
//MethodMatcher实现
public boolean matches(Method method, Class targetClass, boolean beanHasIntroductions) {
   checkReadyToMatch();
   Method targetMethod = AopUtils.getMostSpecificMethod(method, targetClass);
   ShadowMatch shadowMatch = getShadowMatch(targetMethod, method);
   // Special handling for this, target, @this, @target, @annotation
   // in Spring - we can optimize since we know we have exactly this class,
   // and there will never be matching subclass at runtime.
   if (shadowMatch.alwaysMatches()) {
      return true;
   }
   else if (shadowMatch.neverMatches()) {
      return false;
   }
   else {
      // the maybe case
      return (beanHasIntroductions || matchesIgnoringSubtypes(shadowMatch) || matchesTarget(shadowMatch, targetClass));
   }
}
 
}
 
public class PointcutExpressionImpl implements PointcutExpression {
//AspectJExpressionPointcut 实现ClassFilter接口时候调用。
public boolean couldMatchJoinPointsInType(Class aClass) {
   ResolvedType matchType = world.resolve(aClass.getName());
   ReflectionFastMatchInfo info = new ReflectionFastMatchInfo(matchType, null, this.matchContext, world);
   boolean couldMatch = pointcut.fastMatch(info).maybeTrue();
   if (MATCH_INFO) {
      System.out.println("MATCHINFO: fast match for '" + this.expression + "' against '" + aClass.getName() + "': "
            \+ couldMatch);
   }
   return couldMatch;
}}

看一下整个的执行流图时序图

![2](http://www.zzcode.cn/wp-content/uploads/2017/04/2.png)

具体实现逻辑，通过上面的分析就是表达式的使用。确定一个方法是否被应用于advice,其实就是根据PointCut与方法进行匹配。