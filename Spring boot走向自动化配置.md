# Spring boot走向自动化配置
### 前言
这篇文章的题目都改过了好几遍，<font color='red'>原因是我把自动化配置和自动装配这两个概念给弄混淆了。</font>现在的视频和书籍真是误导人，这么基础的概念都不说清楚，一不小心就被带沟里了。

### Spring boot VS Spring Framework
自动化配置是Spring boot相较于Spring Framework的一个全新特性，其内部实现就涉及到了很多Spring Framework自动装配的内容。而且Spring boot的设计也遵循“约定大于配置”的原则，那么到底什么是约定大于配置呢？我之前对这个概念也是一脸懵逼，今天终于理出了一点头绪，以下就是我个人的理解：

> 所谓约定大于配置就是对于一些常用的功能和组件，Spring boot自动提供了相关的配置，并不需要我们人为地去配置，这样就能达到开箱即用的目的。

由此可见在这个语境下，所谓”约定“就是Spring boot自身提供的配置，并不是开发人员参与定义的配置（这里之所以叫”约定“，我想是Spring boot默认开发人员已经知道了Spring boot提供的配置的使用规则，这些规则就是Spring boot与开发人员之间的约定）。当约定不能满足开发要求时，开发人员可以用相关的显式配置覆盖约定提供的自动化配置来达到修改配置的目的。

### Spring Framework的各种装配套路
Spring Framework的装配方式按照开发人员参与的程度可以分为两大类，手动装配和自动装配。

#### 手动装配
<font color=red>手动装配区别于自动装配的不同之处在于需要开发人员自己实现创建bean的细节，</font>这里我以JavaConfig显式配置为例：

```java
@Configuration
public class CDPlayerConfig {
    @Bean
    public CompactDisc sgtPeppers() {
        return new SgtPeppers();
    }
}
```
配置类需要用`@Configuration`注解标注，表明这是一个配置类，这样在运行时，Spring应用上下文就能将标注了`@Bean`注解的方法返回的对象注册为组件bean。方法中对象创建的细节需要开发人员来实现，这也是被称为手动装配的理由。

#### 自动装配
#### 1. 模式注解
模式注解是一种声明在应用中的扮演组件角色的注解，典型的模式注解有`@Component`，诸如`@Service`、`@Repository`、`@Controller`和`@Configuration`都是由`@Component`派生而来。使用模式注解标识的组件类，在运行时**该类的实例**就能自动在Spring应用上下文中注册为bean，<font color='red'>而不需要开发人员去实现具体的创建bean的细节，</font>只需要用`@ComponentScan`设置组件扫描的基础包即可，具体代码如下：

```java 
package com.lucida.repository;

//Bean名称
@Component(value = "myFirstLevelRepository")
public class MyRepository {
}

```
```java
@Configuration
@ComponentScan(basePackages = "com.lucida.repository")
public class RepositoryConfig {
}
```

#### 2. @Enable模块装配
<font color='red'>模块是指具有相同领域功能的组件的集合</font>，比如WebMvc功能就涉及多个相互依赖的组件，这些组件的集合就是一个模块。由此可见`@Enable`模块装配是一次性装配一个具有相同领域功能的组件集合。常见的`@Enable`模块注解有`@EnableWebMvc`、`@EnableAutoConfiguration`、`@EnableWebFlux`等，常见的实现方式无非有以下两种：

##### 2.1 注解驱动方式
`@EnableWebMvc`的实现就是典型的注解驱动方式，运行注解的同时会加载一个标注了`@Configuration`的配置类，这个配置类中实现了创建bean的细节，相当于由Spring显式配置的JavaConfig，所以就能一次性将相关组件全部进行加载，下面就来看看`@EnableWebMvc`的源码：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
// Indicates one or more {@link Configuration @Configuration} classes to import.
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```
```java
@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
}
```
```java
public class WebMvcConfigurationSupport implements ApplicationContextAware, ServletContextAware {
    @Bean
	public PathMatcher mvcPathMatcher() {
		PathMatcher pathMatcher = getPathMatchConfigurer().getPathMatcher();
		return (pathMatcher != null ? pathMatcher : new AntPathMatcher());
	}
}
```


##### 2.2 接口编程方式
这种实现方式比注解驱动方式多了一个中间的处理过程，典型的实现有`@EnableAutoConfiguration`注解。运行该注解的同时会加载一个实现了ImportSelector接口的类，该类实现了ImportSelector接口的selectImports方法，**用于返回符合要求的配置类的全限定名**。相比较注解驱动就多了一个这样的中间步骤，更具有灵活性，下面来看具体的代码（由于`@EnableAutoConfiguration`太复杂，这里展示的就是我用接口编程的方式自定义实现的@Enable模块注解的代码，但原理是一样的）

```java
Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(HelloWorldImportSelector.class)
public @interface EnableHelloWorld {
}
```
```java
public class HelloWorldImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[] {HelloWorldConfiguration.class.getName()};
    }
}
```
```java
public class HelloWorldConfiguration {

    @Bean
    public String helloWorld() {//方法名即Bean名称
        return "Hello, World 2018";
    }
}
```

### 3. 条件装配
顾名思义，满足既定的条件才会进行装配，主要是用`@Conditional`编程的方式来实现。典型的实现有`@ConditionOnClass`。运行这类注解的同时会加载一个OnXXXCondition的类，这个类实现了Condition接口，因此也就实现了Condition接口的matches方法。**这个方法用于判断赋给注解的name和value的值是否与应用或者系统中对应的name的value值一致**，如果一致则返回true，表示满足条件可以进行装配，如果不一致，则不满足条件不进行装配。同样这里给出我自定义的`@Condition`编程的实现代码：

```java
/**
 * Java 系统属性条件判断
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(OnSystemPropertyCondition.class)
public @interface ConditionalOnSystemProperty {

    /**
     * Java 系统属性名称
     * @return
     */
    String name();

    /**
     * Java 系统属性值
     * @return
     */
    String value();
}
```
```java
public class OnSystemPropertyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Map<String, Object> attributes = metadata.getAnnotationAttributes(ConditionalOnSystemProperty.class.getName());

        String propertyName = String.valueOf(attributes.get("name"));
        String propertyValue = String.valueOf(attributes.get("value"));

        String javaPropertyValue = System.getProperty(propertyName);
        return propertyValue.equals(javaPropertyValue);
    }
}
```
使用的时候就直接给注解的属性赋值，表示需要满足的条件：

```java
@ConditionalOnSystemProperty(name = "user.name", value = "ricardolc")
```

### Spring boot自动化配置的实现
铺垫了这么多，终于可以进入正题了，虽然说是Spring boot自动化配置的实现，实际就是`@EnableAutoConfiguration`注解的实现。上述提到的自动装配的方式，在实现`@EnableAutoConfiguration`注解的时候都会用到。以下就`@EnableAutoConfiguration`的源码：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}
```
由上可知，`@EnableAutoConfiguration`是使用接口编程的方式实现的`@Enable`模块注解，根据之前对`@Enable`模块注解的讲解，我们知道加载的AutoConfigurationImportSelector实现了ImportSelector接口的selectImports方法，用于返回符合要求的配置类的全限定名，这些配置类的全限定名从何而来？从配置资源：META-INF/spring.factories中加载而来！怎么加载，用Spring工厂加载机制（实现类SpringFactoriesLoader）进行加载！具体源码如下：

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
List<String> configurations = getCandidateConfigurations(annotationMetadata,
			attributes);
}
```
```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
		AnnotationAttributes attributes) {
	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
			getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
	Assert.notEmpty(configurations,
			"No auto configuration classes found in META-INF/spring.factories. If you "
					+ "are using a custom packaging, make sure that file is correct.");
	return configurations;
}
```
```java
protected Class<?> getSpringFactoriesLoaderFactoryClass() {
		return EnableAutoConfiguration.class;
}
```
这样就能将META-INF/spring.factories资源文件中键为EnableAutoConfiguration全限定名的的值都取出来，而这些值就对应着上述的配置类的全限定名，这里我截取一部分配置文件：

```xml
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
```
随便点开一个配置类都能看到我们熟悉的那些自动装配的套路：

```java
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
}
```
```java
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(MessageDispatcherServlet.class)
@ConditionalOnMissingBean(WsConfigurationSupport.class)
@EnableConfigurationProperties(WebServicesProperties.class)
@AutoConfigureAfter(ServletWebServerFactoryAutoConfiguration.class)
public class WebServicesAutoConfiguration {
}
```
这些配置类的装配策略就是采用了我们上述的那些自动装配的套路。

### 总结
终于写到总结了，这篇文章应该是我迄今为止写过的篇幅最长的技术文章，花的时间也是最多的。原以为这篇文章会比较好写，没想到走了很多的弯路，还好最后写完了，Spring boot真是博大精深，就一个自动化配置的技术细节都要我用这么长的篇幅去叙述。越来越感受到写技术文章妙处了，真是绝好的整理知识的方式。





