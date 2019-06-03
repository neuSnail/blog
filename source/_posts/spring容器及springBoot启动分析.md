---
title: spring容器及springBoot启动分析
date: 2019-05-02 12:30:53
tags: 
- spring
- springboot
categories: spring
---

## ApplicationContext结构

ApplicationContext接口继承HierarchicalBeanFactory和ListableBeanFactory, 二者又都继承自Beanfactory接口.

所以容器的实现要实现BeanFactory中的方法
![image-20190429105542606](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190429105542606.png)



![image-20190429110059201](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190429110059201.png)

大部分常用的容器都是继承AbstractApplicationContext,该抽象类实现了ConfigurableApplicationContext.

springBoot获取beanFactory得到的是org.springframework.beans.factory.support.DefaultListableBeanFactory
最后执行到org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean







## SpringBoot启动

#### @SpringBootApplication

```java
@SpringBootApplication(scanBasePackages = "test")
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited

@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
      @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
      @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
  
	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};

	/**
	 * Base packages to scan for annotated components. Use {@link #scanBasePackageClasses}
	 * for a type-safe alternative to String-based package names.
	 * @return base packages to scan
	 * @since 1.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	/**
	 * Type-safe alternative to {@link #scanBasePackages} for specifying the packages to
	 * scan for annotated components. The package of each class specified will be scanned.
	 * <p>
	 * Consider creating a special no-op marker class or interface in each package that
	 * serves no purpose other than being referenced by this attribute.
	 * @return base packages to scan
	 * @since 1.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

}

```

**@SpringBootConfiguration**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

}
```

看起来就是和@Configuration一样

**@EnableAutoConfiguration**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

   String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

   /**
    * Exclude specific auto-configuration classes such that they will never be applied.
    * @return the classes to exclude
    */
   Class<?>[] exclude() default {};

   /**
    * Exclude specific auto-configuration class names such that they will never be
    * applied.
    * @return the class names to exclude
    * @since 1.3.0
    */
   String[] excludeName() default {};

}
```

Import了`AutoConfigurationImportSelector`  实现了`DeferredImportSelector`(延时导入选择器)接口,关于DeferredImportSelector定义如下

```
A variation of {@link ImportSelector} that runs after all {@code @Configuration} beans
* have been processed. This type of selector can be particularly useful when the selected
* imports are {@code @Conditional}.
```

@EnableAutoConfiguration内部包含@Import(AutoConfigurationImportSelector.class)注解，AutoConfigurationImportSelector为DeferredImportSelector接口的实现类，会在BeanFactory对所有BeanDefinition处理后执行来进行SpringBoot自动配置类的加载、导入，并基于@Conditional条件化配置来决定是否将该配置类内部定义的Bean注册到Spring容器。

@EnableAutoConfiguration-- 依赖于META-INF/spring.factories配置文件，spring.factories文件中键key为 org.springframework.boot.autoconfigure.EnableAutoConfiguration ,对应的配置项(注：全类名)加载到spring容器。

springboot 提供一些常用组件的自动配置比如RedisAutoConfiguration

**@ComponentScan**

component的扫描路径,不指定的话以当前目录为跟路径. 会在容器refresh时候处理目录下的组件



#### SpringApplication.run()

SpringApplication.run(Application.class, args)这个方法其实就是构造一个SpringApplication对象 然后调用他的run方法.

**construct**

```java
public SpringApplication(Class<?>... primarySources) {
   this(null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  //推断web应用类型:webFlux,servlet,none
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
  //查找并加载 classpath下 META-INF/spring.factories文件中所有可用的ApplicationContextInitializer
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
  //使用 SpringFactoriesLoader查找并加载 classpath下 META-INF/spring.factories文件中的所有可用的 ApplicationListener
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  //推断main方法所在的类
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

总结：

* 推断web应用类型

* 加载spring.factories中的initializer和listener



**run方法**

```java
public ConfigurableApplicationContext run(String... args) {
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();
   ConfigurableApplicationContext context = null;
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
   configureHeadlessProperty();
  //获取监听器集合对象
   SpringApplicationRunListeners listeners = getRunListeners(args);
  //发出start事件
   listeners.starting();
   try {
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
     //准备环境 springboot会根据args对PropertySources和profile做一些配置
      ConfigurableEnvironment environment = prepareEnvironment(listeners,
            applicationArguments);
      configureIgnoreBeanInfo(environment);
      Banner printedBanner = printBanner(environment);
     //创建容器
      context = createApplicationContext();
      exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
     //执行Initializers 准备容器
      prepareContext(context, environment, listeners, applicationArguments,
            printedBanner);
     //1.refresh容器
     //2.注册容器的registerShutdownHook(就是jvm关闭的时候调用context的close()方法关闭容器)
      refreshContext(context);
     //empty do nothing 
      afterRefresh(context, applicationArguments);
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass)
               .logStarted(getApplicationLog(), stopWatch);
      }
     //发送started事件
      listeners.started(context);
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```

**prepareEnvironment**

解析配置 绑定到spring Environment

```java
	private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		configureEnvironment(environment, applicationArguments.getSourceArgs());
    //发出environmentPrepared事件
		listeners.environmentPrepared(environment);
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader())
					.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```





**createApplicationContext**

根据之前推断的webApplicationType来创建对应的applicationContext

```java
protected ConfigurableApplicationContext createApplicationContext() {
   Class<?> contextClass = this.applicationContextClass;
   if (contextClass == null) {
      try {
         switch (this.webApplicationType) {
         case SERVLET:
             //AnnotationConfigServletWebServerApplicationContext
            contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
            break;
         case REACTIVE:
             //AnnotationConfigReactiveWebServerApplicationContext
            contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
            break;
         default:
             //AnnotationConfigApplicationContext
            contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
         }
      }
   return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
 

public static <T> T instantiateClass(Class<T> clazz) throws BeanInstantiationException {
   Assert.notNull(clazz, "Class must not be null");
   if (clazz.isInterface()) {
      throw new BeanInstantiationException(clazz, "Specified class is an interface");
   }
   try {
       //clazz.getDeclaredConstructor()获取无参的构造器，然后进行实例化
      return instantiateClass(clazz.getDeclaredConstructor());
   }
   catch (NoSuchMethodException ex) {
    .......
}
 //设置访问权限accessible,对kotlin做特殊处理
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
		Assert.notNull(ctor, "Constructor must not be null");
		try {
			ReflectionUtils.makeAccessible(ctor);
			return (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass()) ?
					KotlinDelegate.instantiateClass(ctor, args) : ctor.newInstance(args));
		}
  ....
}
```

**prepareContext** 

调用初始化器，执行initialize方法

```java
private void prepareContext(ConfigurableApplicationContext context,
      ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
      ApplicationArguments applicationArguments, Banner printedBanner) {
   context.setEnvironment(environment);
  //配置Bean生成器以及资源加载器
   postProcessApplicationContext(context);
   //调用初始化器，执行initialize方法
   applyInitializers(context);
  //发布contextPrepared事件
   listeners.contextPrepared(context);
   if (this.logStartupInfo) {
      logStartupInfo(context.getParent() == null);
      logStartupProfileInfo(context);
   }
   // Add boot specific singleton beans
   ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
   beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
   if (printedBanner != null) {
      beanFactory.registerSingleton("springBootBanner", printedBanner);
   }
   if (beanFactory instanceof DefaultListableBeanFactory) {
      ((DefaultListableBeanFactory) beanFactory)
            .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
   }
   // Load the sources (主类)
   Set<Object> sources = getAllSources();
   Assert.notEmpty(sources, "Sources must not be empty");
  //将sources load为bean
   load(context, sources.toArray(new Object[0]));
  //发送contextLoaded事件
   listeners.contextLoaded(context);
}
```

run方法的主要行为:

1. 构造一个StopWatch，观察SpringApplication的执行
2. 找出所有的SpringApplicationRunListener并封装到SpringApplicationRunListeners中，用于监听run方法的执行。监听的过程中会封装成事件并广播出去让初始化过程中产生的应用程序监听器进行监听
3. 构造Spring容器(ApplicationContext)，并返回
   3.1 创建Spring容器的判断是否是web环境，是的话构造AnnotationConfigEmbeddedWebApplicationContext，否则构造AnnotationConfigApplicationContext
   3.2 初始化过程中产生的初始化器在这个时候开始工作
   3.3 Spring容器的刷新(完成bean的解析、各种processor接口的执行、条件注解的解析等等)
4. 从Spring容器中找出ApplicationRunner和CommandLineRunner接口的实现类并排序后依次执行



### 容器的Refresh

org.springframework.context.support.AbstractApplicationContext#refresh

```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
     //初始化property source
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

     //设置classloader
     // 添加ApplicationContextAwareProcessor 该对象处理各种EnvironmentAware,
     //ApplicationContextAware等aware接口实现
     // beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
      prepareBeanFactory(beanFactory);

      try {
         // 执行AbstractApplicationContext不同子类的不同后继处理方法
         postProcessBeanFactory(beanFactory);

         // 在Spring容器中找出实现了BeanFactoryPostProcessor接口的processor并执行
        //ConfigurationClassPostProcessor这个processor是最先执行的,下面详细说
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         //从容器中找出实现了BeanPostProcessor接口的bean，并注册到BeanFactory的属性中。下面有详细说明
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
        //初始化事件广播器,如果使用的是springboot的话在启动过程中已经初始化过了
        //直接从beanFactory中拿就行
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         // 给子类实现的模板方法
         // 比如ServletWebServerApplicationContext会调用createWebServer
         onRefresh();

         // 把Spring容器内的事件监听器和BeanFactory中的事件监听器都添加的事件广播器中。
				 //然后如果存在early event的话，广播出去。
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
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

#### invokeBeanFactoryPostProcessors

```java
//上面的invokeBeanFactoryPostProcessors后面会调用ConfigurationClassPostProcessor的
//processConfigBeanDefinitions方法
//截取一段核心代码
// Parse each @Configuration class
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

//这时的candidates就是我们springboot的启动类
		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		do {
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);
```

parse方法最后会调到这里:

`org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass`

doProcessConfigurationClass()会对一个配置类执行真正的处理：

一个配置类的成员类(配置类内嵌套定义的类)也可能适配类，先遍历这些成员配置类，调用processConfigurationClass处理它们;
处理@PropertySources注解：进行一些配置信息的解析

1. **处理@PropertySources注解：** <br>进行一些配置信息的解析
2. **处理@ComponentScan注解**：<br>使用ComponentScanAnnotationParser扫描basePackage下的需要解析的类(@SpringBootApplication注解也包括了@ComponentScan注解，只不过basePackages是空的，空的话会去获取当前@Configuration修饰的类所在的包)，并注册到BeanFactory中(这个时候bean并没有进行实例化，而是进行了注册。具体的实例化在finishBeanFactoryInitialization方法中执行)。对于扫描出来的类，递归解析
3. **处理@Import注解：** <br>先递归找出所有的注解，然后再过滤出只有@Import注解的类，得到@Import注解的值。比如查找@SpringBootApplication注解的@Import注解数据的话，首先发现@SpringBootApplication不是一个@Import注解，然后递归调用修饰了@SpringBootApplication的注解，发现有个@EnableAutoConfiguration注解，再次递归发现被@Import(EnableAutoConfigurationImportSelector.class)修饰，还有@AutoConfigurationPackage注解修饰，再次递归@AutoConfigurationPackage注解，发现被@Import(AutoConfigurationPackages.Registrar.class)注解修饰，所以@SpringBootApplication注解对应的@Import注解有2个，分别是@Import(AutoConfigurationPackages.Registrar.class)和@Import(EnableAutoConfigurationImportSelector.class)。找出所有的@Import注解之后，开始处理逻辑
4. **处理@ImportResource注解：** <br>获取@ImportResource注解的locations属性，得到资源文件的地址信息。然后遍历这些资源文件并把它们添加到配置类的importedResources属性中
5. **处理@Bean注解：**获取被@Bean注解修饰的方法，然后添加到配置类的beanMethods属性中
6. **处理DeferredImportSelector:** <br>处理第3步@Import注解产生的DeferredImportSelector，进行selectImports方法的调用找出需要import的类，然后再调用第3步相同的处理逻辑处理

这里@SpringBootApplication注解被@EnableAutoConfiguration修饰，@EnableAutoConfiguration注解被@Import(EnableAutoConfigurationImportSelector.class)修饰，所以在第3步会找出这个@Import修饰的类EnableAutoConfigurationImportSelector，这个类刚好实现了DeferredImportSelector接口，接着就会在第6步被执行。第6步selectImport得到的类就是自动化配置类。

EnableAutoConfigurationImportSelector的selectImport方法会在spring.factories文件中找出key为EnableAutoConfiguration对应的值，有81个，这81个就是所谓的自动化配置类(XXXAutoConfiguration)。

ConfigurationClassParser解析完成之后，被解析出来的类会放到configurationClasses属性中。然后使用ConfigurationClassBeanDefinitionReader去解析这些类。



#### registerBeanPostProcessors

从容器中找出实现了BeanPostProcessor接口的bean，并设置到BeanFactory的属性中。之后bean被实例化的时候会调用这个BeanPostProcessor。
这些BeanPostProcessor包括有AutowiredAnnotationBeanPostProcessor(处理被@Autowired注解修饰的bean并注入)、RequiredAnnotationBeanPostProcessor(处理被@Required注解修饰的方法)、CommonAnnotationBeanPostProcessor(处理@PreDestroy、@PostConstruct、@Resource等多个注解的作用)等。



#### finishBeanFactoryInitialization

实例化BeanFactory中已经被注册但是未实例化的所有实例(懒加载的不需要实例化)。

这个过程最后也是调用org.springframework.beans.factory.support.AbstractBeanFactory#getBean(java.lang.String)进行实例化的

```java
// Instantiate all remaining (non-lazy-init) singletons.
beanFactory.preInstantiateSingletons();
```

比如invokeBeanFactoryPostProcessors方法中根据各种注解解析出来的类，在这个时候都会被初始化。

实例化的过程各种BeanPostProcessor开始起作用,比如AutowiredAnnotationBeanPostProcessor开始对@Autowired进行装配(这个过程太长了就不详细说了,最后会在

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean中实现)



#### finishRefresh

1. 清理context-level resource caches 
2. 初始化生命周期处理器，并设置到Spring容器中(LifecycleProcessor)
3. 调用生命周期处理器的onRefresh方法，这个方法会找出Spring容器中实现了SmartLifecycle接口的类并进行start方法的调用
4. 发布ContextRefreshedEvent事件告知对应的ApplicationListener进行响应的操作`publishEvent(new ContextRefreshedEvent(this));`
5. 调用LiveBeansView的registerApplicationContext方法：如果设置了JMX相关的属性，则就调用该方法

### Bean的装载

bean的装载是通过`AbstractBeanFactory#getBean(java.lang.String)`方法执行的,它还有几个重载,最全的参数如下

```java
public <T> T getBean(String name, @Nullable Class<T> requiredType, @Nullable Object... args)
```

getBean方法最后会调到doGetBean方法:

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
	 //去除 FactoryBean 的修饰符"&"
   final String beanName = transformedBeanName(name);
   Object bean;

   // Eagerly check singleton cache for manually registered singletons.
   Object sharedInstance = getSingleton(beanName);
   if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }
```

**getSingleton**

DefaultSingletonBeanRegistry 中有三个cache,有人称之为"三级缓存"

```java
/** Cache of singleton objects: bean name to bean instance. */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of early singleton objects: bean name to bean instance. */
//这里面放的是还在创建中的早期bean
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

/** Cache of singleton factories: bean name to ObjectFactory. */
//中间态的beanFactory
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  //先尝试从singletonObjects缓存中获取
   Object singletonObject = this.singletonObjects.get(beanName);
  //获取不到且当前单例对象在创建中
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
        //尝试从earlySingletonObjects中获取
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
           //从singletonFactories中获取
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               singletonObject = singletonFactory.getObject();
              //缓存升级,从从singletonFactories中获取中移出,添加到earlySingletonObjects中
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return singletonObject;
}
```

**回到doGetBean方法**

原型的处理和parentBeanFactory加载

```java
else {
   //如果是原型模式且当前bean在中间态,说明有循环依赖
   //原型模式无法通过缓存解决循环依赖,直接抛异常
   if (isPrototypeCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(beanName);
   }

   // Check if bean definition exists in this factory.
   BeanFactory parentBeanFactory = getParentBeanFactory();
  // 如果有parentBeanFactory,使用parentBeanFactory加载bean
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
      else if (requiredType != null) {
         // No args -> delegate to standard getBean method.
         return parentBeanFactory.getBean(nameToLookup, requiredType);
      }
      else {
         return (T) parentBeanFactory.getBean(nameToLookup);
      }
   }

   if (!typeCheckOnly) {
      markBeanAsCreated(beanName);
   }
```

**dependsOn处理以及createBean**

createBean()->doCreateBean()这个流程太长了就不详细说了

值得一提的是doCreateBean方法中会调用populateBean()方法来填充一些属性,包括依赖的注入.

在该过程中会调用postBeanProcessor对bean属性做后置处理

```java
try {
      final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
      checkMergedBeanDefinition(mbd, beanName, args);

      // Guarantee initialization of beans that the current bean depends on.
  		// 这里的depends on 不是指@Resource这种依赖
   		// 是用户显示使用的@DependsOn指定加载顺序
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

      //创建单例bean
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

  		//创建原型bean
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
```

**最后对类型做转换返回Bean**

```java
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
      if (logger.isTraceEnabled()) {
         logger.trace("Failed to convert bean '" + name + "' to required type '" +
               ClassUtils.getQualifiedName(requiredType) + "'", ex);
      }
      throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
   }
}
return (T) bean;
```

#### 对循环依赖的处理

上面的`getSingleton`方法中包含了对单例循环依赖的处理:尝试从一级缓存获取到三级缓存,而三级缓存在单例bean创建时会提前曝光,在`doCreateBean()`中有这样一行代码

```java
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

该代码执行在populateBean()之前,此时添加的单例缓存是一个中间态,依赖还没有填充,但是在堆上已经分配了地址.

所以假如有A->B,B->A这种循环依赖,在创建A将中间态的A添加到缓存,然后发现A依赖B,于是去创建B,在创建B的时候发现缓存中有A,于是直接拿来用了,就中断了循环.

**只有单例模式且是field依赖时Spring才会解决循环依赖**

原型模式不会使用缓存,无法解决. 在构造函数中有依赖的话在new的过程中就挂了,无法使用中间态的方式解决.