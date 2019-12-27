---
title: ClassPathXmlApplicationContext启动过程分析（三）-- 附录信息
date: 2019-12-27 21:10:59
categories:
    - java
    - spring
tags: spring
excerpt: ClassPathXmlApplicationContext启动过程分析（三）-- 附录信息
---

# id 和 name

每个bean在Spring容器中都由一个唯一的名字（beanName） 和 0或多个别名（aliases）。

从Spring容器中获取bean的时候，可以根据beanName，也可以通过别名。

```java
beanFactory.getBean("beanName or aliases");
```

在配置 `<bean />` 的过程中，我们可以配置id和name，如下：

```xml
<!-- beanName 为messageService，别名有3个，分别为m1、m2、m3 -->
<bean id="messageService" name="m1, m2, m3" class="com.javalife.example.MessageServiceImpl" />
```

```xml
<!-- beanName 为m1，别名有2个，分别为m2、m3 -->
<bean name="m1, m2, m3" class="com.javalife.example.MessageServiceImpl" />
```

```xml
<!-- beanName 为com.javalife.example.MessageServiceImpl#0，别名为com.javalife.example.MessageServiceImpl -->
<bean class="com.javalife.example.MessageServiceImpl" />
```

```xml
<!-- beanName 为messageService，没有别名。 -->
<bean id="messageService" class="com.javalife.example.MessageServiceImpl" />
```

# bean覆盖、循环依赖

我们说过，默认情况下，allowBeanDefinitionOverriding 属性为 null。如果在同一配置文件中 Bean id 或 name 重复了，会抛错，但是如果不是同一配置文件中，会发生覆盖。

可是有些时候我们希望在系统启动的过程中就严格杜绝发生 Bean 覆盖，因为万一出现这种情况，会增加我们排查问题的成本。

循环依赖说的是 A 依赖 B，而 B 又依赖 A。或者是 A 依赖 B，B 依赖 C，而 C 却依赖 A。默认 allowCircularReferences 也是 null。

它们两个属性是一起出现的，必然可以在同一个地方一起进行配置。

配置如下：

```java
public class NoBeanOverridingContextLoader extends ContextLoader {
  
  @Override
  protected void customizeContext(ServletContext servletContext, ConfigurableWebApplicationContext applicationContext) {
    super.customizeContext(servletContext, applicationContext);
    AbstractRefreshableApplicationContext arac = (AbstractRefreshableApplicationContext) applicationContext;
    // 不允许 bean 覆盖。
    arac.setAllowBeanDefinitionOverriding(false);
  }
}
```

```java
public class MyContextLoaderListener extends org.springframework.web.context.ContextLoaderListener {
  @Override
  protected ContextLoader createContextLoader() {
    return new NoBeanOverridingContextLoader();
  }
}
```

```xml
<listener>
    <listener-class>com.javalife.MyContextLoaderListener</listener-class>  
</listener>
```

# profile

我们可以把不同环境的配置分别配置到单独的文件中，举个例子：

```xml
<!-- development环境  -->
<beans profile="development"
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xsi:schemaLocation="...">

    <jdbc:embedded-database id="dataSource">
        <jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
        <jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
    </jdbc:embedded-database>
</beans>
```

```xml
<!-- production环境 -->
<beans profile="production"
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="...">

    <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
</beans>
```

也可以在一个配置文件中使用：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="...">

    <beans profile="development">
        <jdbc:embedded-database id="dataSource">
            <jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
            <jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
        </jdbc:embedded-database>
    </beans>

    <beans profile="production">
        <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
    </beans>
</beans>
```

Spring 在启动的过程中，会去寻找 `spring.profiles.active`的属性值，根据这个属性值来的。指定profile的方式有3种：

1. 命令行启动时，指定参数。

   `-Dspring.profiles.active="profile1,profile2"`

2. 代码中指定，通过代码的形式从 Environment 中设置 profile。

   ```java
   AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
   ctx.getEnvironment().setActiveProfiles("development");
   ctx.register(SomeConfig.class, StandaloneDataConfig.class, JndiDataConfig.class);
   ctx.refresh(); // 重启
   ```

3. 注解指定。如果是单元测试中使用的话，在测试类中使用 @ActiveProfiles 指定。

如果是 Spring Boot 的话更简单，一般会创建 application.properties、application-dev.properties、application-prod.properties 等文件，其中 application.properties 配置各个环境通用的配置，application-{profile}.properties 中配置特定环境的配置，然后在启动的时候指定 profile：`java -Dspring.profiles.active=prod -jar demo.jar`

# 工厂模式生成bean

这里的工厂模式指的是factory-bean 和 FactoryBean 中的前者，是说静态工厂或实例工厂，而后者是 Spring 中的特殊接口，代表一类特殊的 Bean。

设计模式里，工厂方法模式分静态工厂和实例工厂，我们分别看看 Spring 中怎么配置这两个，代码示例如下：

1. 静态工厂

   ```xml
   <bean id="clientService" class="examples.ClientService" factory-method="createInstance"/>
   ```

   ```java
   public class ClientService {
       private static ClientService clientService = new ClientService();
       private ClientService() {}
       // 静态方法
       public static ClientService createInstance() {
           return clientService;
       }
   }
   ```

   

2. 实例工厂

   ```xml
   <bean id="serviceLocator" class="examples.DefaultServiceLocator">
       <!-- inject any dependencies required by this locator bean -->
   </bean>
   
   <bean id="clientService" factory-bean="serviceLocator"
       factory-method="createClientServiceInstance"/>
   
   <bean id="accountService" factory-bean="serviceLocator"
       factory-method="createAccountServiceInstance"/>
   ```

   ```java
   public class DefaultServiceLocator {
   
       private static ClientService clientService = new ClientServiceImpl();
   
       private static AccountService accountService = new AccountServiceImpl();
   
       public ClientService createClientServiceInstance() {
           return clientService;
       }
   
       public AccountService createAccountServiceInstance() {
           return accountService;
       }
   }
   ```

# FactoryBean

FactoryBean 适用于 Bean 的创建过程比较复杂的场景，比如数据库连接池的创建。

```java
public interface FactoryBean<T> {
    T getObject() throws Exception;
    Class<T> getObjectType();
    boolean isSingleton();
}
```

```java
public class Person { 
    private Car car ;
    private void setCar(Car car){ this.car = car;  }  
}
```

假设现在需要创建一个 Person 的 Bean，首先需要一个 Car 的实例，我们这里假设 Car 的实例创建很麻烦，那么我们可以把创建 Car 的复杂过程包装起来：

```java
public class MyCarFactoryBean implements FactoryBean<Car>{
    private String make; 
    private int year ;

    public void setMake(String m){ this.make =m ; }

    public void setYear(int y){ this.year = y; }

    public Car getObject(){ 
      // 这里我们假设 Car 的实例化过程非常复杂，反正就不是几行代码可以写完的那种
      CarBuilder cb = CarBuilder.car();

      if(year!=0) cb.setYear(this.year);
      if(StringUtils.hasText(this.make)) cb.setMake( this.make ); 
      return cb.factory(); 
    }

    public Class<Car> getObjectType() { return Car.class ; } 

    public boolean isSingleton() { return false; }
}
```

FactoryBean的声明有两种：

1. xml配置

   ```xml
   <bean class = "com.javalife.MyCarFactoryBean" id = "car">
     <property name = "make" value ="Honda"/>
     <property name = "year" value ="1984"/>
   </bean>
   <bean class = "com.javalife.Person" id = "josh">
     <property name = "car" ref = "car"/>
   </bean>
   ```

   id 为 “car” 的 bean 其实指定的是一个 FactoryBean，不过配置的时候，我们直接让配置 Person 的 Bean 直接依赖于这个 FactoryBean 就可以了。中间的过程 Spring 已经封装好了。

2. JavaConfig配置

   ```java
   @Configuration 
   public class CarConfiguration { 
   
       @Bean 
       public MyCarFactoryBean carFactoryBean(){ 
         MyCarFactoryBean cfb = new MyCarFactoryBean();
         cfb.setMake("Honda");
         cfb.setYear(1984);
         return cfb;
       }
   
       @Bean
       public Person aPerson(){ 
       Person person = new Person();
         // 注意这里的不同
       person.setCar(carFactoryBean().getObject());
       return person; 
       } 
   }
   ```

   这个时候，其实我们的思路也很简单，把 MyCarFactoryBean 看成是一个简单的 Bean 就可以了，不必理会什么 FactoryBean，它是不是 FactoryBean 和我们没关系。

# 初始化 bean 的回调

初始化bean的回调有四种方案：

1. init-method属性

   ```xml
   <bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>
   ```

2. InitializingBean接口

   ```java
   public class AnotherExampleBean implements InitializingBean {
   
       public void afterPropertiesSet() {
           // do some initialization work
       }
   }
   ```

3. @Bean注解

   ```java
   @Bean(initMethod = "init")
   public Foo foo() {
       return new Foo();
   }
   ```

4. @PostConstruct注解

   ```java
   @PostConstruct
   public void init() {
   
   }
   ```

# 销毁 bean 的回调

销毁bean的回调也有四种方式：

1. destroy-method属性

   ```xml
   <bean id="exampleInitBean" class="examples.ExampleBean" destroy-method="cleanup"/>
   ```

2. DisposableBean接口

   ```java
   public class AnotherExampleBean implements DisposableBean {
   
       public void destroy() {
           // do some destruction work (like releasing pooled connections)
       }
   }
   ```

3. @Bean注解

   ```java
   @Bean(destroyMethod = "cleanup")
   public Bar bar() {
       return new Bar();
   }
   ```

4. @PreDestroy注解

   ```java
   @PreDestroy
   public void cleanup() {
   
   }
   ```



# ConversionService

最有用的场景就是，它用来将前端传过来的参数和后端的 controller 方法上的参数进行绑定的时候用。

像前端传过来的字符串、整数要转换为后端的 String、Integer 很容易，但是如果 controller 方法需要的是一个枚举值，或者是 Date 这些非基础类型（含基础类型包装类）值的时候，我们就可以考虑采用 ConversionService 来进行转换。

```xml
<bean id="conversionService"
  class="org.springframework.context.support.ConversionServiceFactoryBean">
  <property name="converters">
    <list>
      <bean class="com.javadoop.learning.utils.StringToEnumConverterFactory"/>
    </list>
  </property>
</bean>
```

ConversionService 接口很简单，所以要自定义一个 convert 的话也很简单。

下面再说一个实现这种转换很简单的方式，那就是实现 Converter 接口。

```java
public class StringToDateConverter implements Converter<String, Date> {

    @Override
    public Date convert(String source) {
        try {
            return DateUtils.parseDate(source, "yyyy-MM-dd", "yyyy-MM-dd HH:mm:ss", "yyyy-MM-dd HH:mm", "HH:mm:ss", "HH:mm");
        } catch (ParseException e) {
            return null;
        }
    }
}
```

只要注册这个 Bean 就可以了。这样，前端往后端传的时间描述字符串就很容易绑定成 Date 类型了，不需要其他任何操作。

# Bean继承

在初始化 Bean 的地方，有如下方法：

```java
RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
```

这里涉及到的就是 `` 中的 parent 属性，我们来看看 Spring 中是用这个来干什么的。

首先，我们要明白，这里的继承和 java 语法中的继承没有任何关系，不过思路是相通的。child bean 会继承 parent bean 的所有配置，也可以覆盖一些配置，当然也可以新增额外的配置。

Spring 中提供了继承自 AbstractBeanDefinition 的 `ChildBeanDefinition` 来表示 child bean。

如下：

```xml
<bean id="inheritedTestBean" abstract="true" class="org.springframework.beans.TestBean">
    <property name="name" value="parent"/>
    <property name="age" value="1"/>
</bean>

<bean id="inheritsWithDifferentClass" class="org.springframework.beans.DerivedTestBean"
        parent="inheritedTestBean" init-method="initialize">
    <property name="name" value="override"/>
</bean>
```

parent bean 设置了 `abstract="true"` 所以它不会被实例化，child bean 继承了 parent bean 的两个属性，但是对 name 属性进行了覆写。

child bean 会继承 scope、构造器参数值、属性值、init-method、destroy-method 等等。

如果parent bean 中的 abstract = true加上了以后 ，Spring 在实例化 singleton beans 的时候会忽略这个 bean。

比如下面这个极端 parent bean，它没有指定 class，所以毫无疑问，这个 bean 的作用就是用来充当模板用的 parent bean，此处就必须加上 abstract = true。

```xml
<bean id="inheritedTestBeanWithoutClass" abstract="true">
    <property name="name" value="parent"/>
    <property name="age" value="1"/>
</bean>
```



# 方法注入

一般来说，我们的应用中大多数的 Bean 都是 singleton 的。singleton 依赖 singleton，或者 prototype 依赖 prototype 都很好解决，直接设置属性依赖就可以了。

但是，如果是 singleton 依赖 prototype 呢？这个时候不能用属性依赖，因为如果用属性依赖的话，我们每次其实拿到的还是第一次初始化时候的 bean。

一种解决方案就是不要用属性依赖，每次获取依赖的 bean 的时候从 BeanFactory 中取。这个也是大家最常用的方式了吧。怎么取，我就不介绍了，大部分 Spring 项目大家都会定义那么个工具类的。

另一种解决方案就是这里要介绍的通过使用 Lookup method。

1. lookup-method

   ```java
   public abstract class CommandManager {
   
       public Object process(Object commandState) {
           // grab a new instance of the appropriate Command interface
           Command command = createCommand();
           // set the state on the (hopefully brand new) Command instance
           command.setState(commandState);
           return command.execute();
       }
   
       // okay... but where is the implementation of this method?
       protected abstract Command createCommand();
   }
   ```

   xml配置`<lookup-method />`

   ```xml
   <!-- a stateful bean deployed as a prototype (non-singleton) -->
   <bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype">
       <!-- inject dependencies here as required -->
   </bean>
   
   <!-- commandProcessor uses statefulCommandHelper -->
   <bean id="commandManager" class="fiona.apple.CommandManager">
       <lookup-method name="createCommand" bean="myCommand"/>
   </bean>
   ```

   Spring 采用 **CGLIB 生成字节码**的方式来生成一个子类。我们定义的类不能定义为 final class，抽象方法上也不能加 final。

   lookup-method 上的配置也可以采用注解来完成，这样就可以不用配置`<lookup-method />`了，其他不变：

   ```java
   public abstract class CommandManager {
   
       public Object process(Object commandState) {
           MyCommand command = createCommand();
           command.setState(commandState);
           return command.execute();
       }
   
       @Lookup("myCommand")
       protected abstract Command createCommand();
   }
   ```

   > 注意，既然用了注解，要配置注解扫描：`<context:component-scan base-package="com.javalife" />`

   甚至，我们可以像下面这样:

   ```java
   public abstract class CommandManager {
   
       public Object process(Object commandState) {
           MyCommand command = createCommand();
           command.setState(commandState);
           return command.execute();
       }
   
       @Lookup
       protected abstract MyCommand createCommand();
   }
   ```

   > 上面的返回值用了 MyCommand，当然，如果 Command 只有一个实现类，那返回值也可以写 Command。

   

2. replaced-method

   replaced-method的功能是替换掉 bean 中的一些方法。

   ```java
   public class MyValueCalculator {
   
       public String computeValue(String input) {
           // some real code...
       }
   
       // some other methods...
   }
   ```

   方法覆写，注意要实现 MethodReplacer 接口：

   ```java
   public class ReplacementComputeValue implements org.springframework.beans.factory.support.MethodReplacer {
   
       public Object reimplement(Object o, Method m, Object[] args) throws Throwable {
           // get the input value, work with it, and return a computed result
           String input = (String) args[0];
           ...
           return ...;
       }
   }
   ```

   xml配置：

   ```xml
   <bean id="myValueCalculator" class="x.y.z.MyValueCalculator">
       <!-- 定义 computeValue 这个方法要被替换掉 -->
       <replaced-method name="computeValue" replacer="replacementComputeValue">
           <arg-type>String</arg-type>
       </replaced-method>
   </bean>
   
   <bean id="replacementComputeValue" class="a.b.c.ReplacementComputeValue"/>
   ```

   > arg-type 明显不是必须的，除非存在方法重载，这样必须通过参数类型列表来判断这里要覆盖哪个方法。



# BeanPostProcessor接口

```java
public interface BeanPostProcessor {

   Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

   Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```

 bean 在初始化之前会执行 postProcessBeforeInitialization 这个方法，初始化完成之后会执行 postProcessAfterInitialization 这个方法。

除了自定义的 BeanPostProcessor 实现外，Spring 容器在启动时自动加几个。如在获取 BeanFactory 的 obtainFactory() 方法结束后的 prepareBeanFactory(factory)，Spring 往容器中添加了这两个 BeanPostProcessor：ApplicationContextAwareProcessor、ApplicationListenerDetector。

第一个方法接受的第一个参数是 bean 实例，第二个参数是 bean 的名字，重点在返回值将会作为新的 bean 实例，所以这里不能随便返回个 null。这里可以对一些 bean 实例做一些修饰。但是对于 Spring 框架来说，它会决定是不是要在这个方法中返回 bean 实例的代理。

BeanPostProcessor执行时机：在 bean 实例化完成、属性注入完成之后，会执行回调方法，具体请参见类 AbstractAutowireCapableBeanFactory#initBean 方法。**首先会回调几个实现了 Aware 接口的 bean，然后就开始回调 BeanPostProcessor 的 postProcessBeforeInitialization 方法，之后是回调 init-method，然后再回调 BeanPostProcessor 的 postProcessAfterInitialization 方法。**

































