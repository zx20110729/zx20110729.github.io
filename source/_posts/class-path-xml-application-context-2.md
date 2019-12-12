---
title: ClassPathXmlApplicationContext启动过程分析（二）-- 从配置文件加载bean的流程
date: 2019-12-12 20:52:25
categories:
    - java
    - spring
tags: spring
excerpt: ClassPathXmlApplicationContext启动过程分析（二）-- 从配置文件加载bean的流程


---

> 上篇降到Bean容器实例化，接着看实例化之后的流程。
>
> 本篇讲Bean的初始化。

# 1. 还是refresh方法

上篇讲到obtainFreshBeanFactory方法。`AbstractApplicationContext.java`

```java
@Override
	public void refresh() throws BeansException, IllegalStateException {
    // 加锁，必须的，不然在并发的情况下就乱了
		synchronized (this.startupShutdownMonitor) {
			// 准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
			prepareRefresh();

			// 关键步骤，该方法会把配置文件解析成一个个Bean定义，注册到BeanFactory中，
      // 这里的Bean还没有初始化，只是把配置信息提取出来，
      // 注册也只是把这些信息保存到注册中心。（beanName -> beanDifinition 的 map）
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 设置BeanFactory的类加载器，添加一些BeanPostProcessor，手动注册几个特殊的Bean。
			prepareBeanFactory(beanFactory);

			try {
				// 知识点：BeanFactoryPostProcessor，若Bean实现了该接口，那么容器初始化后，
        // 会调用postProcessBeanFactory方法。
        // 提供子类的拓展点，到这里的时候，所有的Bean都"加载"、"注册"完成了，【此时Bean还没有初始化】
        // 具体的子类可以在这里添加一些自定义的BeanFactoryPostProcessor做些事情。
				postProcessBeanFactory(beanFactory);

				// 调用BeanFactoryPostProcessor 各个实现类的postProcessBeanFactory方法
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册BeanPostProcess的实现类，【与BeanFactoryPostProcessor不一样哦】
        // 此接口有两个方法：postProcessBeforeInitialization、postProcessAfterInitialization
        // 两个方法分别在Bean初始化之前和初始化之后得到执行。【此时Bean还没有初始化】
				registerBeanPostProcessors(beanFactory);

				// 初始化当前ApplicationContext的MessageSource，国际化。
				initMessageSource();

				// 初始化当前ApplicationContext的事件广播器。
				initApplicationEventMulticaster();

				// 模版方法（钩子方法）
        // 具体的子类可以在这里初始化一些特殊的Bean【在初始化singleton beans之前】
				onRefresh();

				// 注册时间监听器，即实现了ApplicationListener接口的类。
				registerListeners();

				// 核心方法，重点、重点、重点
        // 初始化所有的singleton beans【lazy-init的除外】
				finishBeanFactoryInitialization(beanFactory);

				// 最后，广播事件，ApplicationContext初始化完成。
				finishRefresh();
			} catch (...){
        ......
      }
	}
```

# 2、prepareBeanFactory方法

---

准备Bean容器，Spring对一些特殊的bean进行处理。

`AbstractApplicationContext.java`

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// 设置beanFactory的类加载器。
    // 这里设置当前ApplicationContext类的类加载器。
		beanFactory.setBeanClassLoader(getClassLoader());
  	// 设置BeanExpressionResolver。
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// 添加一个BeanPostProcessor(ApplicationContextAwareProcessor)。
  	// 该Processor负责在beans初始化时，回调实现了Aware接口的beans。
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
  	
  	// 下面代码的意思是，如果某个bean依赖一下几个接口实现，在自动装配时忽略他们，容器通过其他方式处理这些依赖。
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// 下面代码的意思是，如果有bean依赖于一下几个类型，在这里进行设置相应的值。
  	// ApplicationContext 继承BeanFactory、 ResourceLoader、
    // ApplicationEventPublisher、MessageSource，
    // 所以对于这几个依赖，可以赋值为 this，注意 this 是一个 ApplicationContext，
    // 那这里怎么没看到为 MessageSource 赋值呢？那是因为 MessageSource 被注册成为了一个普通的 bean。
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// 添加一个BeanPostProcessor(ApplicationListenerDetector)。
  	// 该Processor负责在ApplicationListener的实现bean初始化时，将其添加到listener列表中。
  	// 即负责注册事件监听器。
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// 添加特定的Processor。
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// 注册默认的bean。
  	// 注册environment这个bean。
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
  	// 注册systemProperties这个bean。
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
  	// 注册systemEnvironment这个bean。
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

# 3、finishBeanFactoryInitialization方法

---

初始化(预初始化/实例化)所有的singleton bean。到该阶段BeanFactory已经创建完成，并且所有实现了BeanFactoryPostProcessor接口的bean都已经实现了实例化，其中的 postProcessBeanFactory(factory) 方法已经得到回调执行了。Spring 自己注册了一些特殊的 Bean，如 ‘environment’、‘systemProperties’ 等。

`AbstractApplicationContext.java`

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// 实例化 conversionService 的 bean for this context.
  	// 注册：实例化的操作在beanFactory.getBean方法中。
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// 先实例化 LoadTimeWeaverAware beans 为了让其更早的注册transformers.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Spring 已经开始预初始化 singleton beans 了，冻结配置信息。
		beanFactory.freezeConfiguration();

  	// 实例化所有的 singleton beans，不包含lazy-init的beans。
		beanFactory.preInstantiateSingletons();
	}
```

### 3.1、preInstantiateSingletons方法

预初始化singleton beans，`DefaultListableBeanFactory.java`

```java
// 1、
public void preInstantiateSingletons() throws BeansException {
   if (logger.isDebugEnabled()) {
      logger.debug("Pre-instantiating singletons in " + this);
   }

   // beanDefinitionNames保存了所有的bean name。
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

   // 触发所有的singlton beans的初始化
   for (String beanName : beanNames) {
     	// 合并父 Bean 中的配置，注意 <bean id="" class="" parent="" /> 中的 parent。
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
     	// 非抽象、非懒加载的 singltons。
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
        	// FactoryBean会在beanName前面加上'&'符号，再调用getBean。
         if (isFactoryBean(beanName)) {
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
            if (bean instanceof FactoryBean) {
               final FactoryBean<?> factory = (FactoryBean<?>) bean;
               // 判断当前FactoryBean是否为SmartFactoryBean。
               boolean isEagerInit;
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean){
                  isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                              ((SmartFactoryBean<?>) factory)::isEagerInit,
                        getAccessControlContext());
               } else {
                  isEagerInit = (factory instanceof SmartFactoryBean &&
                        ((SmartFactoryBean<?>) factory).isEagerInit());
               }
               if (isEagerInit) {
                  getBean(beanName);
               }
            }
         }
         else {
            // 对于普通的bean(非FactoryBean)，只需调用该方法。
            getBean(beanName);
         }
      }
   }

   // 到这里说明所有的非懒加载的 singleton beans 已经完成了初始化
   // 如果我们定义的 bean 是实现了 SmartInitializingSingleton 接口的，
   for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton) {
         final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
         if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
               smartSingleton.afterSingletonsInstantiated();
               return null;
            }, getAccessControlContext());
         }
         else {
            smartSingleton.afterSingletonsInstantiated();
         }
      }
   }
}
```
`getBean方法`

```java
// 2、
public Object getBean(String name) throws BeansException {
	 return doGetBean(name, null, null, false);
}
// 3、
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}

```

