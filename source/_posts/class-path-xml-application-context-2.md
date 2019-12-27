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
### 3.2、getBean方法

```java
// 2、
public Object getBean(String name) throws BeansException {
	 return doGetBean(name, null, null, false);
}
// 3、从容器中获取Bean，已经初始化过了就从容器中直接返回，否则就先初始化再返回。
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
		// 获取一个beanName，分为两种情况，一种是FactoryBean(前面带‘&’)；
    // 另外一种是别名问题，获取bean时，可以通过别名获取。
		final String beanName = transformedBeanName(name);
  	// 返回值。
		Object bean;

		// 检查bean是否已经被创建过。
		Object sharedInstance = getSingleton(beanName);
  	// args参数数组，分为两种情况，一种是通过getBean(beanName)进来的，此时args为null；
  	// 另一种情况是，args不为空，此时意味着调用方不是获取bean，而是创建bean。
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
      // 如果是普通bean，直接返回sharedInstance；
      // 如果是FactoryBean，则返回FactoryBean创建的bean。
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// 如果创建过此beanName的prototype类型的bean，则抛出异常；
      // 有可能是陷入循环引用。
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 检查一下这个BeanDifinition是否存在。
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
        // 如果父容器是AbstractBeanFactory类型，则有父容器负责查询该bean。
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
        // 如果args不为空，返回父容器查询结果。
				else if (args != null) {
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
        // 将当前beanName放到一个alreadyCreated的Set集合中。
				markBeanAsCreated(beanName);
			}

      // 此时要准备创建bean实例了，对于singleton的bean来说，容器中还没创建过此bean；
      // 对于prototype的bean来说，本来就要创建一个新的bean。
			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// 先初始化该bean依赖的所有bean。此处的依赖指的是depends-on中定义的依赖。
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
            // 检查是否有循环依赖。
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
            // 注册依赖关系。
						registerDependentBean(dep, beanName);
						try {
              // 初始化依赖的bean。
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 此时开始实例化bean。
        // 创建singleton类型的bean。
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
              // 创建bean。
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				// 创建prototype类型的bean。
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
            // 创建bean。
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
				// 如果不是singleton和prototype类型的bean，将bean的创建委托给对应的实现类来处理。
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
                // 创建bean。
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

		// 检查一下类型是否匹配。
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

### 3.3 createBean方法

`AbstractAutowireCapableBeanFactory.java`

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// 确保BeanDefinition中的Class被加载。
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// 准备方法覆盖。
  	// 这里关于MethodOverrides，它来自于bean定义中的 <lookuo-method/>、<replaced-method/>两个标签。
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// 让InstantiationAwareBeanPostProcessor在这里有机会返回代理。
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
      // 创建bean。
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isDebugEnabled()) {
				logger.debug("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException ex) {
			// A previously detected exception with proper bean creation context already...
			throw ex;
		}
		catch (ImplicitlyAppearedSingletonException ex) {
			// An IllegalStateException to be communicated up to DefaultSingletonBeanRegistry...
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```

`doCreateBean方法`创建bean，即bean的初始化。

该方法主要包含三个步骤：

1. 创建bean实例，createBeanInstance方法。
2. 属性赋值，populateBean方法。
3. 初始化bean，initiazeBean方法。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
      // 1、说明不是FactoryBean，这里实例化Bean。
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
  	// 这个bean就是 BeanDefinition 对应的类实例。
		final Object bean = instanceWrapper.getWrappedInstance();
  	// 类型。
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
  	// 解决循环依赖。
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
      // 2、属性赋值。
			populateBean(beanName, mbd, instanceWrapper);
      // 3、初始化bean。
      // 此时处理bean初始化之后的各种回调。
      // 如：init-method、InitializingBean接口、以及BeanPostProcessor等。
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		} catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			} else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				} else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

1、createBeanInstance方法，实例化bean。

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// 确保此时已经加载该class。
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
		// 校验类的访问权限。
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
		// 判断是否有实例化提供者，有的话，使用提供者进行bean的实例化。
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
		// 采用工厂方法实例化。
		if (mbd.getFactoryMethodName() != null)  {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
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
        // 构造函数依赖注入。
				return autowireConstructor(beanName, mbd, null, null);
			} else {
        // 无参构造函数。
				return instantiateBean(beanName, mbd);
			}
		}

		// 判断是否采用有参构造函数。
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
      // 构造函数依赖注入。
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// 无参构造函数。
		return instantiateBean(beanName, mbd);
	}
```

`instantiateBean`方法，无参构造函数。

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						getInstantiationStrategy().instantiate(mbd, beanName, parent),
						getAccessControlContext());
			}
			else {
        // 实例化。
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
      // 包装一下，返回。
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
	}
```

`instantiate`方法，实例化过程。`SimpleInstantiationStrategy.java` 

```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		// 如果没有方法覆盖，则使用Java反射进行实例化，狗则使用CGLIB。
  	// 方法覆盖参考 lookup-method 和 replaced-method。
		if (!bd.hasMethodOverrides()) {
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
							constructorToUse = AccessController.doPrivileged(
									(PrivilegedExceptionAction<Constructor<?>>) () -> clazz.getDeclaredConstructor());
						}
						else {
							constructorToUse =	clazz.getDeclaredConstructor();
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
      // 利用构造方法进行实例化。
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// 存在方法覆盖，利用CGLIB来完成实例化，需要依赖于CGLIB生成子类。
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}
```

2、`populateBean`方法，参数赋值。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}
  
  	// 此时 bean 实例化已完成(通过工厂方法或构造方法)，但还没开始属性赋值。
  	// InstantiationAwareBeanPostProcessor 的实现类在此处对bean进行修改。
		boolean continueWithPropertyPopulation = true;
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
          // 如果返回false，代表不需要进行后续的属性赋值，也不需要再经过其他的 BeanPostProcessor 的处理。
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
		// bean实例的所有属性。
		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// 通过名字找到所有属性值，如果存在bean依赖，先初始化依赖的bean，记录依赖关系。
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// 通过类型装配。
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);
		// 判断是否有InstantiationAwareBeanPostProcessor或需要检查依赖。
		if (hasInstAwareBpps || needsDepCheck) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 这里有个非常有用的 BeanPostProcessor，AutowiredAnnotationBeanPostProcessor。
            // 对采用 @Autowired、@Value 注解的依赖进行赋值。
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}

		if (pvs != null) {
      // 设置 bean 实例的属性值。
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}
```



3、`initiazeBean`方法，初始化Bean。属性赋值完成之后，这一步其实就是处理各种回调了。

`AbstractAutowireCapableBeanFactory.java`

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		} else {
      // 如果 bean 实现了 BeanNameAware、BeanClassLoaderAware或BeanFactoryAware接口，回调。
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
      // BeanPostProcessor 的 postProcessBeforeInitialization 回调。
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
      // 处理 bean 中定义的 init-method,
      // 或者如果 bean 实现了 InitializingBean 接口，调用 afterPropertiesSet() 方法。
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
      // BeanPostPorcessor 的 postProcessAfterInitiazation 回调。
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```













