---
title: ClassPathXmlApplicationContext启动过程分析（一）-- 从配置文件加载bean的流程
date: 2019-12-12 11:52:25
categories:
    - java
    - spring
tags: spring
excerpt: ClassPathXmlApplicationContext启动过程分析（一）-- 从配置文件加载bean的流程

---



# 1、构造函数

---

```java
public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {
		// 如果已经有 ApplicationContext 并需要配置成父子关系，那么调用这个构造方法
		super(parent);
  	// 根据提供的路径，处理成配置文件数组(以分号、逗号、空格、tab、换行符分割)
		setConfigLocations(configLocations);
		if (refresh) {
			refresh(); //核心方法
		}
	}
```

接下来，就是 refresh()，这里简单说下为什么是 refresh()，而不是 init() 这种名字的方法。因为 ApplicationContext 建立起来以后，其实我们是**可以通过调用 refresh() 这个方法重建**的，refresh() 会将原来的 ApplicationContext 销毁，然后再重新执行一次初始化操作。

# 2、refresh方法

---

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
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// 销毁已经初始化的singleton beans，以免有些bean会一直占用资源。
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// 继续往外抛出异常。
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

下面开始一步步解释refresh执行的各个方法。

# 3、prepareRefresh方法

---

创建Bean容器前的准备工作。

```java
protected void prepareRefresh() {
  	// 记录启动时间
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false); // 将AtomicBoolean的closed属性设置为false
		this.active.set(true); // 将AtomicBoolean的active属性设置为true

		if (logger.isInfoEnabled()) {
			logger.info("Refreshing " + this);
		}

		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		// 校验 xml 配置文件
		getEnvironment().validateRequiredProperties();
  
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```

# 4、obtainFreshBeanFactory方法

---

创建Bean容器，加载并注册Bean。该方法是全文最重要的部分之一，会初始化BeanFactory、加载Bean、注册Bean等等。但是并未初始化Bean，即未生成Bean实例。

`AbstractApplicationContext.java`

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
  	// 关闭旧的BeanFactory，创建新的BeanFactory，加载Bean定义、注册Bean等等。
		refreshBeanFactory();
  	// 返回刚刚创建的 BeanFactory
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```

`AbstractRefreshableApplicationContext.java` 第124行。

```java
@Override
	protected final void refreshBeanFactory() throws BeansException {
		// 如果 ApplicationContext中已经加载过BeanFactory，销毁所有Bean，并关闭BeanFactory。
    // 注意：容器中可以有多个BeanFactory，这里的BeanFactory并不是全局的，而是当前ApplicationContext的。
    if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
      // 初始化一个BeanFactory。
			DefaultListableBeanFactory beanFactory = createBeanFactory();
      // 用于BeanFactory的序列化。
			beanFactory.setSerializationId(getId());
      
      // 以下两个重要方法
      // 设置BeanFactory的两个配置属性：是否允许Bean被覆盖，是否允许循环引用。
			customizeBeanFactory(beanFactory);
      // 加载Bean到BeanFactory。
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

> 看到这里的时候，我们应该站在高处看 ApplicationContext ，ApplicationContext 继承自 BeanFactory，但是它不应该被理解为 BeanFactory 的实现类，而是说其内部持有一个实例化的 BeanFactory（DefaultListableBeanFactory）。以后所有的 BeanFactory 相关的操作其实是委托给这个实例来处理的。

> 如果想要在程序运行的时候动态往 Spring IOC 容器注册新的 bean，就会使用到这个类(DefaultListableBeanFactory)。可以通过 ApplicationContext 接口能获取到 AutowireCapableBeanFactory，然后它向下转型就能得到 DefaultListableBeanFactory 了。

**BeanFactory是Bean的容器，那么什么是Bean？**

在Spring容器中BeanDefinition就是我们所说的Bean，在配置文件或类中定义的Bean都会会转换为BeanDefinition存在与Spring的BeanFactory中。

> BeanDefinition 中保存了我们的 Bean 信息，比如这个 Bean 指向的是哪个类、是否是单例的、是否懒加载、这个 Bean 依赖了哪些 Bean 等等。

### **BeanDefinition接口定义**

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

	// 默认只有singleton和prototype两种作用域。
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;


	// 不重要的参数
	int ROLE_APPLICATION = 0;
	int ROLE_SUPPORT = 1;
	int ROLE_INFRASTRUCTURE = 2;


	// 设置父Bean，这里涉及到bean的继承【不是Java的继承】。
  // 可以理解为继承父Bean的配置信息。
	void setParentName(@Nullable String parentName);
	// 获取父Bean
	String getParentName();

	// 指定Bean的类名称，将来通过反射来生成实例。
	void setBeanClassName(@Nullable String beanClassName);
	// 获取Bean的类名称。
	String getBeanClassName();

	// 设置/获取作用域。
	void setScope(@Nullable String scope);
	String getScope();

	// 设置/获取懒加载。
	void setLazyInit(boolean lazyInit);
	boolean isLazyInit();

	// 设置or获取Bean所依赖的Bean。
  // 这里的依赖不是属性依赖（如 @Autowire标记的），是depend-on=""设置的值。
	void setDependsOn(@Nullable String... dependsOn);
	String[] getDependsOn();

	// 设置or获取可否注入到其他Bean中。
  // 只对根据类型注入有效，对根据名称注入无效。
	void setAutowireCandidate(boolean autowireCandidate);
	boolean isAutowireCandidate();

	// 设置or获取是否是主要的
  // 同一接口的多个实现，Spring会优先设置primary为true的Bean。 
	void setPrimary(boolean primary);
	boolean isPrimary();

	// 设置or获取工厂类名称。
  // 类如果是通过工厂方法生成的，指定工厂名称。
	void setFactoryBeanName(@Nullable String factoryBeanName);
	String getFactoryBeanName();
	// 设置or获取工程类中的工厂方法。
	void setFactoryMethodName(@Nullable String factoryMethodName);
	String getFactoryMethodName();

	// 构造器参数。
	ConstructorArgumentValues getConstructorArgumentValues();
	// 是否包含构造器参数。
	default boolean hasConstructorArgumentValues() {
		return !getConstructorArgumentValues().isEmpty();
	}

	// Bean的属性值。
	MutablePropertyValues getPropertyValues();
	// 是否包含属性值。
	default boolean hasPropertyValues() {
		return !getPropertyValues().isEmpty();
	}

	// 是否singleton。
	boolean isSingleton();
	// 是否prototype。
	boolean isPrototype();
	// 是否抽象。
	boolean isAbstract();

	int getRole();
	String getDescription();
	String getResourceDescription();
	BeanDefinition getOriginatingBeanDefinition();
}
```

# 5、customizeBeanFactory方法

---

该方法用来设置是否允许BeanDefinition覆盖、是否允许循环引用。

`AbstractRefreshableApplicationContext.java`第224行。

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
		if (this.allowBeanDefinitionOverriding != null) {
      // 是否允许Bean定义覆盖。
			beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		if (this.allowCircularReferences != null) {
      // 是否允许Bean之间的循环依赖。
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}
```

BeanDefinition的覆盖指的是在配置文件中使用了相同的id或name。默认情况下allowBeanDefinitionOverriding为null，如果在同一配置文件出现覆盖，会出错；如果出现在不同的配置文件中，就会发生覆盖。

循环引用：A依赖B，B也依赖A；A依赖B，B依赖C，C依赖A。默认情况下，Spring 允许循环依赖，当然如果你在 A 的构造方法中依赖 B，在 B 的构造方法中依赖 A 是不行的。

# 6、loadbeanDefinitions方法

---

根据配置，加载各个 Bean，然后放到 BeanFactory 中。

读取配置的操作在 XmlBeanDefinitionReader 中，其负责加载配置、解析。

`AbstractXmlApplicationContext.java`

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
   //给传入的BeanFactory实例话一个XmlBeanDefinitionReader对象。
   XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

   // Configure the bean definition reader with this context's
   // resource loading environment.
   beanDefinitionReader.setEnvironment(this.getEnvironment());
   beanDefinitionReader.setResourceLoader(this);
   beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

   // 初始化BeanDefinitionReader，抽象方法
   initBeanDefinitionReader(beanDefinitionReader);
   // 加载BeanDefinition【重要】。 
   loadBeanDefinitions(beanDefinitionReader);
}

protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
      // 1.加载BeanDefinition。
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
      // 2.会转换为resource，跟1一样。
			reader.loadBeanDefinitions(configLocations);
		}
	}
```

`AbstractBeanDefinitionReader.java`

```java
	@Override
	public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int counter = 0;
		for (Resource resource : resources) {
			counter += loadBeanDefinitions(resource);
		}
		return counter; // 返回加载的BeanDefinition的数量。
	}
```

`XmlBeanDefinitionReader.java`

```java
// 1、
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
   return loadBeanDefinitions(new EncodedResource(resource));
}
// 2、
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
   Assert.notNull(encodedResource, "EncodedResource must not be null");
   if (logger.isInfoEnabled()) {
      logger.info("Loading XML bean definitions from " + encodedResource.getResource());
   }
	 // 使用ThreadLocal存放配置文件资源。 
   Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
   if (currentResources == null) {
      currentResources = new HashSet<>(4);
      this.resourcesCurrentlyBeingLoaded.set(currentResources);
   }
   if (!currentResources.add(encodedResource)) {
      throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
   }
   try {
      InputStream inputStream = encodedResource.getResource().getInputStream();
      try {
         InputSource inputSource = new InputSource(inputStream);
         if (encodedResource.getEncoding() != null) {
            inputSource.setEncoding(encodedResource.getEncoding());
         }
         // 核心加载部分。
         return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
      }
      finally {
         inputStream.close();
      }
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException(
            "IOException parsing XML document from " + encodedResource.getResource(), ex);
   }
   finally {
      currentResources.remove(encodedResource);
      if (currentResources.isEmpty()) {
         this.resourcesCurrentlyBeingLoaded.remove();
      }
   }
}
// 3、
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
      // 把XML文件转换为Document对象。
			Document doc = doLoadDocument(inputSource, resource);
			return registerBeanDefinitions(doc, resource);
		} catch ......
}

// 4、
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
  	// 创建一个DefaultBeanDefinitionDocumentReader对象，由该对象进行注册BeanDefinition。
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

`DefaultBeanDefinitionDocumentReader.java`

```java
// 1、
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
   this.readerContext = readerContext;
   logger.debug("Loading bean definitions");
   Element root = doc.getDocumentElement();
   //从XML根节点解析文件。
   doRegisterBeanDefinitions(root);
}
// 2、
protected void doRegisterBeanDefinitions(Element root) {
   // 该类负责解析BeanDefinition。
   // 定义parent用来解决递归问题，
   // 因为<beans />内部还可以定义<beans />
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
      // 环境变量profile是否匹配环境，<beans ... profile="dev"> 。
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root);//钩子
  	// 解析BeanDefinitions。
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);//钩子

		this.delegate = parent;
	}
// 3、
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	// default namespace 涉及四个标签 <import/> <alias/> <bean/> <beans/>
  // 其他属于custom
  if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
            // 解析 default namespace下的元素。
						parseDefaultElement(ele, delegate);
					}
					else {
            // 解析其他namespace下的元素。
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
// 4、
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	// 处理 <import /> 标签	
  if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
   // 处理 <alias /> 标签定义
   // <alias name="fromName" alias="toName"/>
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		// 处理 <bean /> 标签定义，这也算是我们的重点吧
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// 如果碰到的是嵌套的 <beans /> 标签，需要递归
			doRegisterBeanDefinitions(ele);
		}
	}
// 5、
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
  	// 1、将<bean />节点信息提取出来，然后封装到一个BeanDefinition里。
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
      // 2、解析自定义属性。
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 2、注册Bean。
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// 3、注册完成，发送事件。
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

`<bean/>`标签中的属性

| Property                 |                                                              |
| ------------------------ | ------------------------------------------------------------ |
| class                    | 类的全限定名                                                 |
| name                     | 可指定 id、name(用逗号、分号、空格分隔)                      |
| scope                    | 作用域                                                       |
| constructor arguments    | 指定构造参数                                                 |
| properties               | 设置属性的值                                                 |
| autowiring mode          | no(默认值)、byName、byType、 constructor                     |
| lazy-initialization mode | 是否懒加载(如果被非懒加载的bean依赖了那么其实也就不能懒加载了) |
| initialization method    | bean 属性设置完成后，会调用这个方法                          |
| destruction method       | bean 销毁后的回调方法                                        |

### 把`<bean/>` 解析为BeanDefinition对象

`BeanDefinitionParserDelegate.java`

```java
	// 1、
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}
	// 2、
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
		String id = ele.getAttribute(ID_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		// 把name属性按 "逗号、分号、空格"切分，形成一个别名列表。
		List<String> aliases = new ArrayList<>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

		String beanName = id;
    // 如果id为空，选择别名里的第一个元素作为id。
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
		// 根据<bean />标签创建出一个BeanDefinition。
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
        // id和name均未设置进入这里。
				try {
					if (containingBean != null) {
            //自动生成beanName。
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
            // beanName：全类名#编号
            // beanClassName：全类名
						beanName = this.readerContext.generateBeanName(beanDefinition);
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
              // 把beanClassName设置为别名。
							aliases.add(beanClassName);
						}
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
      // 返回BeanDefinitionHolder。
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}

	// 3、
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
      // 创建BeanDefinition。
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
			// 设置BeanDefinition的属性，这些属性定义在AbstractBeanDefinition中。
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
			
      // 以下代码解析<bean/>内部的子元素，解析结果放在bd属性中。
      // 解析<meta/>、<lookup-method/>、<replaced-method/>
			parseMetaElements(ele, bd);
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
			// 解析<constructor-arg/>、<property/>、<qulifier/>
			parseConstructorArgElements(ele, bd);
			parsePropertyElements(ele, bd);
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```

### 把BeanDefinition对象注册

`BeanDefinitionReaderUtils.java`

```java
public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {
		String beanName = definitionHolder.getBeanName();
  	// 注册primary bean。
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// 注册bean的别名
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
        // alias -> beanName 保存别名信息，使用map。
        // 获取时，会先把alias转换为beanName，通过beanName查找。
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

`DefaultListableBeanFactory.java`

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}
		// 已经存在的BeanDefinition，allowBeanDefinitionOverriding是否允许覆盖 -_-||
		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
  	// 处理名称重复的BeanDefinition。
		if (existingDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
        // 不允许覆盖抛出异常。
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + existingDefinition + "] bound.");
			}
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
        // log 用框架定义的bean覆盖用户定义的bean。
				if (logger.isWarnEnabled()) {
					logger.warn("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							existingDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(existingDefinition)) {
        // log 用新的bean覆盖旧的bean。
				if (logger.isInfoEnabled()) {
					logger.info("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (logger.isDebugEnabled()) {
          // log 用同等的bean覆盖旧的bean。
					logger.debug("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
      // 判断是否已有其他的bean 开始初始化了。
      // 注册：注册bean结束后，bean仍没有初始化。
      // 在容器启动的最后，会预初始化所有的singlton beans。
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// 最正常应该进入该分支。
        
        // 将BeanDefinition放入map中，该map保存所有的BeanDefinition。
				this.beanDefinitionMap.put(beanName, beanDefinition);
        // 按bean配置顺序保存bean的名字。
				this.beanDefinitionNames.add(beanName);
        // 这是个 LinkedHashSet，代表的是手动注册的 singleton bean，
         // 注意这里是 remove 方法，到这里的 Bean 当然不是手动注册的
         // 手动指的是通过调用以下方法注册的 bean ：
         //     registerSingleton(String beanName, Object singletonObject)
         // 这不是重点，解释只是为了不让大家疑惑。Spring 会在后面"手动"注册一些 Bean，
         // 如 "environment"、"systemProperties" 等 bean，我们自己也可以在运行时注册 Bean 到容器中的
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
```

到这里已经初始化了 Bean 容器，`` 配置也相应的转换为了一个个 BeanDefinition，然后注册了各个 BeanDefinition 到注册中心，并且发送了注册事件。





















































