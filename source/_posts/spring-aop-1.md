---
title: Spring-AOP（一）-- AOP简介
date: 2019-12-27 22:10:59
categories:
    - java
    - spring
tags: spring
excerpt: Spring-AOP（一）-- AOP简介
---

# Spring AOP简介

## Spring AOP

- 它基于**动态代理**来实现。默认地，如果使用接口的，用 JDK 提供的动态代理实现；如果没有接口，使用 CGLIB 实现；
- Spring 3.2 以后，spring-core 添加了 CGLIB 和 ASM 依赖；
- Spring 的 IOC 容器和 AOP 都很重要，Spring AOP 需要依赖于 IOC 容器来管理；
- 如果开发web程序，有时候可能需要的是一个 Filter 或一个 Interceptor，而不一定是 AOP；
- Spring AOP 只能作用于 Spring 容器中的 Bean，它是使用纯粹的 Java 代码实现的，只能作用于 bean 的方法；
- Spring 提供了 AspectJ 的支持；
- Spring AOP 是基于代理实现的，在容器启动的时候需要生成代理实例，在方法调用上也会增加栈的深度，使得 Spring AOP 的性能不如 AspectJ 那么好；

## AspectJ

- AspectJ来自Eclipse基金会，[AspectJ](https://www.eclipse.org/aspectj)；
- 属于静态织入，它是通过修改代码来实现的，它的织入时机可以是：
  - Compile-time weaving：编译期织入，如类 A 使用 AspectJ 添加了一个属性，类 B 引用了它，这个场景就需要编译期的时候就进行织入，否则没法编译类 B；
  - Post-compile weaving：也就是已经生成了 .class 文件，或已经打成 jar 包了，这种情况我们需要增强处理的话，就要用到编译后织入；
  - **Load-time weaving**：指的是在加载类的时候进行织入，要实现这个时期的织入，有几种常见的方法。1、自定义类加载器来干这个，这个应该是最容易想到的办法，在被织入类加载到 JVM 前去对它进行加载，这样就可以在加载的时候定义行为了。2、在 JVM 启动的时候指定 AspectJ 提供的 agent：`-javaagent:xxx/xxx/aspectjweaver.jar`。
- AspectJ 能干很多 Spring AOP 干不了的事情，它是 **AOP 编程的完全解决方案**。Spring AOP 致力于解决的是企业级开发中最普遍的 AOP 需求（方法织入），而不是力求成为一个像 AspectJ 一样的 AOP 编程完全解决方案；
- 因为 AspectJ 在实际代码运行前完成了织入，所以大家会说它生成的类是没有额外运行时开销的。

## AOP术语

Advice、Advisor、Pointcut、Aspect、Joinpoint；

#  Spring AOP的使用

Spring AOP使用有三种方式：

- Spring 1.2 **基于接口的配置**：最早的 Spring AOP 是完全基于几个接口的；
- Spring 2.0 **schema-based 配置**：Spring 2.0 以后使用 XML 的方式来配置，使用 命名空间 `<aop />`；
- Spring 2.0 **@AspectJ 配置**：使用注解的方式来配置。

## 基于接口配置实现AOP

首先定义两个接口`UserService`和`OrderService` ， 以及它们的实现类 `UserServiceImpl` 和 `OrderServiceImpl`：

```java
public interface UserService{
   User createUser(String name, int age);
   User queryUser();
}
```

```java
public interface OrderService{
   Order createOrder(String username, String product);
   Order queryOrder();
}
```

```java
public interface UserServiceImpl implements UserService{
   private static User user = null;
  
   @Override
   public User createUser(String name, int age){
     user = new User();
     user.setName(name); 
     user.setAge(age);
     return user;
   }
   @Override
   public User queryUser(){
     return user;
   }
}
```

```java
public interface OrderServiceImpl{
   private static Order order = null;
   @Override
   public Order createOrder(String username, String product){
     order = new Order();
     order.setUsername(username);
     order.setProduct(product);
     return order;
   }
   @Override
   public Order queryOrder(){
     return order;
   }
}
```

定义两个advice，分别用于拦截方法执行前和方法返回后。
```java
public class LogArgsAdvice implements MethodBeforeAdvice{
  @Override
  public void before(Method method, Object[] args, @Nullable Object target) throws Throwable{
    System.out.println("准备执行方法："+method.getName()+"，参数列表："+Arrays.toString(args));
  }
}
```

```java
public class LogResultAdvice implements AfterReturningAdvice{
  @Override
  public void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable{
    System.out.println("方法返回："+returnValue);
  }
}
```

上面的两个 Advice 分别用于方法调用前输出参数和方法调用后输出结果。

XML配置Advice：

```xml
<bean id="userServiceImpl"  class="com.javalife.service.UserServiceImpl" />
<bean id="orderServiceImpl"  class="com.javalife.service.OrderServiceImpl" />

<bean id="logArgsAdvice"  class="com.javalife.aop.LogArgsAdvice" />
<bean id="logResultAdvice"  class="com.javalife.aop.LogResultAdvice" />

<bean id="userServiceProxy"  class="org.springframework.aop.framework.ProxyFactoryBean" >
  	<!-- 代理的接口 -->
		<property name="proxyInterfaces">
      	<list>
          	<value>com.javalife.service.UserService</value>
      	</list>
  	</property>
    <!-- 代理的具体实现 -->
  	<property name="target" ref="userServiceImpl" />
  	<!-- 配置拦截器，这里可以配置 advice、advisor、interceptor -->
    <property>
      	<list>
          <value>logArgsAdvice</value>
          <value>logResultAdvice</value>
      	</list>
  	</property>
</bean> 
```

运行：

```java
public class Demo {
  public static void main(String[] args){
    	// 启动 spring 容器
    	ApplicationContext context = new ClassPathXmlApplicationContext("xxx.xml");
    	// 这里获取 AOP 代理
    	UserService userService = (UserService)context.getBean("userServiceProxy");
    	
    	userService.createUser("Jerry",5);
    	userService.queryUser();
  }
}
```

输入结果：

>准备执行方法：createUser，参数列表：[Jerry,5]
>
>方法返回：User{name='Jerry',age=5}
>
>准备执行方法：queryUser，参数列表：[]
>
>方法返回：User{name='Jerry',age=5}

从结果可以看到，对 UserService 中的两个方法都做了前、后拦截。这个例子理解起来应该非常简单，就是一个代理实现。

> 代理模式需要一个接口、一个具体实现类，然后就是定义一个代理类，用来包装实现类，添加自定义逻辑，在使用的时候，需要用代理类来生成实例。

此中方法有个致命的问题，如果我们需要拦截 OrderService 中的方法，那么我们还需要定义一个 OrderService 的代理。如果还要拦截 PostService，得定义一个 PostService 的代理......而且拦截器的粒度只控制到了类级别，类中所有的方法都进行了拦截。























