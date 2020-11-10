[TOC]

##  Spring IOC源码（一）



## 1. IOC是什么？

IOC即控制反转，是**依赖倒置原则**的一种实现思路，而DI（依赖注入）则是其一种实现方式。

简单来说，IOC是为了删除对象间的直接依赖关系，避免因底层类的改动，导致所有依赖它的类都要改变，就是我们常说的解耦。

**依赖倒置原则**：高层模块不应该依赖低层模块，两者都应该依赖其抽象；抽象不应该依赖细节，细节应该依赖抽象。

关于IOC更详细解释可以看看这两篇文章：

https://martinfowler.com/articles/injection.html#InversionOfControl

https://www.zhihu.com/question/23277575



## 2. 基于注解的Spring IOC源码分析

### 2.1 构造方法

先看下`AnnotationConfigApplicationContext`的构造方法

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    // 构造方法
    this();
    // 解析传入的类并注入到容器中
    register(componentClasses);
    // IOC的核心方法
    refresh();
}

public AnnotationConfigApplicationContext() {
    // 新建Bean定义(BeanDefinition)的读取器
    this.reader = new AnnotatedBeanDefinitionReader(this);
    // 新建扫描器
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

在`AnnotationConfigApplicationContext`的构造方法中，新建了两个对象，分别为`AnnotatedBeanDefinitionReader`和`ClassPathBeanDefinitionScanner`.



> ### new AnnotatedBeanDefinitionReader(this)

AnnotatedBeanDefinitionReader`这个方法主要是注册了一些解析注解的组件。

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
    this(registry, getOrCreateEnvironment(registry));
}

public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    Assert.notNull(environment, "Environment must not be null");
    this.registry = registry;
    // 这个类用来处理@Conditional注解
    this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
    // 注册注解解析器
    AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```

`@Conditional`：用来做条件的判断，当一个对象需要另一个对象创建之后才能创建时，可以使用该注解。

```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
    BeanDefinitionRegistry registry, @Nullable Object source) {
	// 获取bean工厂
    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
    if (beanFactory != null) {
        // 注册处理@Order、@Priority注解的解析器
        if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
            beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
        }
        // 注册处理@Lazy注解的解析器
        if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
            beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        }
    }

    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
	// 注册ConfigurationClassPostProcessor后置处理器，非常重要的一个类。
    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }
	// 注册处理@Autowired、@Value的解析器
    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 检查是否支持JSR-250，注册处理@Resource、@PostConstruct等注解解析器
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 检查是否支持JPA，
    if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition();
        try {
            def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
                                                AnnotationConfigUtils.class.getClassLoader()));
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
        }
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 注册解析@EventListener的解析器,将有该注解的方法解析为ApplicationListener实例
    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
    }

   	// 注册解析@EventListenerFactory的解析器,对@EventListener提供支持
    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
    }

    return beanDefs;
}
```

`EventListenerMethodProcessor`实现了`SmartInitializingSingleton`，`ApplicationContextAware`和`BeanFactoryPostProcessor`接口。

`SmartInitializingSingleton`：该接口中只有一个`afterSingletonsInstantiated()`方法，该方法会在bean实例化后触发。

`ApplicationContextAware`：该接口中`setApplicationContext`会在`init`方法之前触发。

`BeanFactoryPostProcessor`：在bean定义信息被加载到容器中之后，初始化之前。



> ###  new ClassPathBeanDefinitionScanner(this)

`ClassPathBeanDefinitionScanner`就简单了很多，主要是设置上下文环境和资源加载器，解析@Component、@Repository、@Service、@Controller注解。

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
                                      Environment environment, @Nullable ResourceLoader resourceLoader) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    this.registry = registry;
	// @ComponentScan注解中有一个FilterType属性，可以自定义扫描的类型
    if (useDefaultFilters) {
        registerDefaultFilters();
    }
    // 设置环境
    setEnvironment(environment);
    // 设置资源加载器
    setResourceLoader(resourceLoader);
}

/**
* 使用默认FilterType
*/
protected void registerDefaultFilters() {
    // 注册处理@Component的过滤器
    this.includeFilters.add(new AnnotationTypeFilter(Component.class));
    ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
    try {
        this.includeFilters.add(new AnnotationTypeFilter(
            ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
        logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
    }
    catch (ClassNotFoundException ex) {
        // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
    }
    try {
        this.includeFilters.add(new AnnotationTypeFilter(
            ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
        logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
    }
    catch (ClassNotFoundException ex) {
        // JSR-330 API not available - simply skip.
    }
}
```

`@Controller`、`@Service`、`@Repository`都是`@Component`的子注解，所以过滤器只需注入`@Component`，就可以。



### 2.2 register(componentClasses)

该方法主要是解析bean定义信息并注入其中，就是上一步构造方法中注册，包括自己在`AnnotationConfigApplicationContext`构造方法中传的那个。

![1604671150217](SpringIOC.assets/1604671150217.png)

该方法最终是调用`AnnotatedBeanDefinitionReader`类的`doRegisterBean`方法

```java
/**
* @param beanClass the class of the bean
* @param name an explicit name for the bean
* @param qualifiers 除bean级别之外，特殊注解
* @param supplier bean实例的回调
* @param customizers 工厂的回调，设置延迟初始化或主标志
* @since 5.0
*/
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name, @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier, @Nullable BeanDefinitionCustomizer[] customizers) {
    // 新建一个bean定义
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    // 根据@Conditional判断是否需要跳过
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    abd.setInstanceSupplier(supplier);
    // 解析获取作用域信息
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    // 解析获取beanName，若没有指定就会截取类名
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
	// 处理一些公共的注解，@Lazy、@Primary等
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    // 解析创建来的注解
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    if (customizers != null) {
        for (BeanDefinitionCustomizer customizer : customizers) {
            customizer.customize(abd);
        }
    }
	// 将bean定义信息放到一个BeanDefinitionHolder中
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    // 创建作用域代理，默认不创建
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 将bean注册到bean工厂中
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

`BeanDefinition`：bean定义，用来描述bean实例的属性值、构造函数等元数据信息。

`BeanDefinitionReaderUtils.registerBeanDefinition()`：将bean定于注册到容器中。

```java
public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
    throws BeanDefinitionStoreException {
	// 获取beanDefinition名称
    String beanName = definitionHolder.getBeanName();
    // 将bean注册到容器中
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // 注册别名，只是简单的属性赋值检查等操作，不展开
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}

/**
* =>DefaultListableBeanFactory.registerBeanDefinition()
*/
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException {

    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");
	
    if (beanDefinition instanceof AbstractBeanDefinition) {
        // 如果是抽象的bean定义，进行校验
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                                                   "Validation of bean definition failed", ex);
        }
    }
	
    // beanDefinitionMap：bean定义的缓存，key是bean名称，value是bean定义
    // 先去缓存中查一下。
    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {
        // 检查是否可以覆盖
        if (!isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
        }
        // 一些判断检查
        else if (existingDefinition.getRole() < beanDefinition.getRole()) {
            // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
            if (logger.isInfoEnabled()) {
                logger.info("Overriding user-defined bean definition for bean '" + beanName +
                            "' with a framework-generated bean definition: replacing [" +
                            existingDefinition + "] with [" + beanDefinition + "]");
            }
        }
        else if (!beanDefinition.equals(existingDefinition)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Overriding bean definition for bean '" + beanName +
                             "' with a different definition: replacing [" + existingDefinition +
                             "] with [" + beanDefinition + "]");
            }
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("Overriding bean definition for bean '" + beanName +
                             "' with an equivalent definition: replacing [" + existingDefinition +
                             "] with [" + beanDefinition + "]");
            }
        }
        // 将缓存中原来的beanDefinition覆盖
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }
    else {
        if (hasBeanCreationStarted()) {
            // 已经开始创建
            synchronized (this.beanDefinitionMap) {
                // 放入bean定义的缓存中
                this.beanDefinitionMap.put(beanName, beanDefinition);
                // 更新bean定义的集合
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                // 移除手动创建的bean
                removeManualSingletonName(beanName);
            }
        }
        else {
            // 已经开始创建，但是还未完成，先放入缓存中
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            removeManualSingletonName(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }
	// 如果bean定义缓存或单例bean缓存中，则重置bean定义的信息
    if (existingDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
}
```

除了通过xml或注解的方式向Spring容器中添加bean外，还可以获取beanFactory，通过`registerXX()`方法来手工注入bean。

`resetBeanDefinition()`方法：

```java
protected void resetBeanDefinition(String beanName) {
    // 清理合并的bean定义缓存
    clearMergedBeanDefinition(beanName);

    // 销毁bean定义
    destroySingleton(beanName);

    // 重置后置处理器
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        if (processor instanceof MergedBeanDefinitionPostProcessor) {
            ((MergedBeanDefinitionPostProcessor) processor).resetBeanDefinition(beanName);
        }
    }

    // 重置所有子bean的父类
    for (String bdName : this.beanDefinitionNames) {
        if (!beanName.equals(bdName)) {
            BeanDefinition bd = this.beanDefinitionMap.get(bdName);
            // Ensure bd is non-null due to potential concurrent modification
            // of the beanDefinitionMap.
            if (bd != null && beanName.equals(bd.getParentName())) {
                resetBeanDefinition(bdName);
            }
        }
    }
}

```

