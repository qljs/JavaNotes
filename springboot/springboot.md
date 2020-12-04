

## SpringBoot启动



## 一  @SpringBootApplication

`@SpringBootApplication`是springboot的核心注解，它主要的功能就是开启自动装配。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};
	
	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "nameGenerator")
	Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

	@AliasFor(annotation = Configuration.class)
	boolean proxyBeanMethods() default true;
}
```



其中最重要的三个注解分别是：

- <font color="red">@SpringBootConfiguration</font>：定义配置类，包含了`@Configuration`注解；
- <font color="red">@EnableAutoConfiguration</font>：开启自动装配功能，springboot中很重要的一个注解，通过`@Import`注解导入了`AutoConfigurationImportSelector`，该类会去扫描`META-INF/spring.factories`中的组件，注入到容器中；
- <font color="red">@ComponentScan</font>：扫描包的注解，默认的路径是启动类下的包，这也是为什么启动类一般会放在最外层包。



### 1. 自动装配：@EnableAutoConfiguration

`AutoConfigurationImportSelector`实现了`ImprotSelector`接口，其中重要的方法是`getAutoConfigurationEntry`，在该方法中，会去扫描`META-INF/spring.factories`下的组件。

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    // 获取@EnableAutoConfiguration注解中exclude和excludeName的值，用来排除组件
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 扫描META-INF/spring.factories下的组件
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    // 删除重复的组件
    configurations = removeDuplicates(configurations);
    // 获取需要排除的组件配置
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    // 检查需要排除的组件配置
    checkExcludedClasses(configurations, exclusions);
    // 移除需要排除的
    configurations.removeAll(exclusions);
    // 获取配置类过滤器进行过滤，移除pom文件未引入且不需要的组件
    configurations = getConfigurationClassFilter().filter(configurations);
    // 触发自动装配导入的监听事件，
    // 如果有实现BeanClassLoaderAware、BeanFactoryAware、EnvironmentAware、ResourceLoaderAware接口的，会在这里执行
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}
```





> ### getCandidateConfigurations()

根绝断言也可以看出，该方法会去扫描META-INF/spring.factories下的组件配置

```java
// org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#getCandidateConfigurations
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
                                                                         getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
                    + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}

// org.springframework.core.io.support.SpringFactoriesLoader#loadFactoryNames
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    // 从上面方法进来时，factoryTypeName的值
    // factoryTypeName：org.springframework.boot.autoconfigure.EnableAutoConfiguration
    String factoryTypeName = factoryType.getName();
    // 扫描META-INF/加载spring.factories下的组件配置
    return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}
```



> #### 加载spring.factories组件配置：loadSpringFactories()

```java
// org.springframework.core.io.support.SpringFactoriesLoader#loadFactoryNames
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    // 先尝试从缓存中取
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        // FACTORIES_RESOURCE_LOCATION：META-INF/spring.factories
        // 从META-INF/spring.factories加载组件配置，
        Enumeration<URL> urls = (classLoader != null ?
                                 classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                                 ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        // 循环获取路径中的配置组件
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                                           FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```





## 二 启动类的run()方法

在启动类中，出了`@SpringBootApplication`注解之外，还有一个就是`run()`方法，接下来就看一下`run()`方法做了什么。

```java
// SpringApplication.java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    // primarySource：传进来的启动类
    return run(new Class<?>[] { primarySource }, args);
}

// SpringApplication.java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```



### 1 new SpringApplication()

新建spring上下文中主要功能：

- 获取webApplication类型；
- 扫描META-INF/spring.factories下的组件（主要是一些组件的初始化器和监听器），创建实例；
- 找到main方法所在类的类型。

```java
// SpringApplication.java
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}

// SpringApplication.java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 获取WebApplication的类型
    // NONE：不作为web应用运行，也不启动内嵌web服务器
    // SERVLET：作为基于Servlet的web应用程序运行，并启动嵌入的Servlet web服务
    // REACTIVE：作为反应式Web服务器，并启动嵌入的反应式wbe服务器
    // 这种模式应该是spring5.0中新提出的webflux，该部分不在主流程中，笔者也未深入研究。
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 扫描META-INF/spring.factories下的组件，通过反射方法获取构造方法构建实例，放入springApplication中
    // key：org.springframework.context.ApplicationContextInitializer
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 扫描META-INF/spring.factories下的组件
    // key：org.springframework.context.ApplicationListener
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 获取main方法的class
    this.mainApplicationClass = deduceMainApplicationClass();
}
```



> ### 获取扫描到的组件实例：getSpringFactoriesInstances()

```java
// SpringApplication.java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

// SpringApplication.java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    // 获取类加载器
    ClassLoader classLoader = getClassLoader();
    // loadFactoryNames(type, classLoader)方法已经分析过
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 通过反射创建实例
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```



> #### 创建实例：createSpringFactoriesInstances（

```java
// SpringApplication.java
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
       ClassLoader classLoader, Object[] args, Set<String> names) {
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            // 通过反射回去class
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            // 获取构造器
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            // 创建实例
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```



> #### 获取含main方法的class：deduceMainApplicationClass

```java
// SpringApplication.java
private Class<?> deduceMainApplicationClass() {
    try {
        // 获取堆栈中的元素，循环判断
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```





### 2. run()方法

```java
// SpringApplication.java
public ConfigurableApplicationContext run(String... args) {
    // 创建stopwatch记录当前任务的名称和起始时间
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // 设置java.awt.headless属性
    configureHeadlessProperty();
    // 获取监听
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 注册到多播器上并执行监听
    listeners.starting();
    try {
        // 处理参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 根据获取到的webApplicationType，构造环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);
        context = createApplicationContext();
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                                                         new Class[] { ConfigurableApplicationContext.class }, context);
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
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



#### 2.1 准备环境：prepareEnvironment()

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
                                                   ApplicationArguments applicationArguments) {
    // 根据获取到的webApplicationType，创建ConfigurableEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    listeners.environmentPrepared(environment);
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                                                                                               deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

