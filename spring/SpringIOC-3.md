[TOC]

## Spring IOC学习（三）getBean()

在`refresh()`方法中，`finishBeanFactoryInitialization()`方法会对所有单例非懒加载的bean进行初始化。

## 一. 初始化非懒加载的单例bean

```java
// AbstractApplicationContext.java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
        beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        // 设置属性转换
        beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    if (!beanFactory.hasEmbeddedValueResolver()) {
        // 添加嵌入式值解析器，用于解析注解属性的值
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // 处理需要织入切面的bean，如果需要使用spring底层组件，可以通过实现xxAware接口
    // LoadTimeWeaverAware：用于加载时织入
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // 将临时的ClassLoader置为空
    // 该ClassLoader通常用于加载时编织，确保尽可能的延迟加载真正的bean类
    beanFactory.setTempClassLoader(null);

    // 冻结所有的bean定义，不允许再修改
    beanFactory.freezeConfiguration();

    // 准备初始化所有非懒加载的单例bean
    beanFactory.preInstantiateSingletons();
}
```



## 二 preInstantiateSingletons()出事化之前的一些准备

```java
// DefaultListableBeanFactory.java
public void preInstantiateSingletons() throws BeansException {
    if (logger.isTraceEnabled()) {
        logger.trace("Pre-instantiating singletons in " + this);
    }

    // 拿到工厂中所有beanDefinition名称
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    for (String beanName : beanNames) {
        // 尝试从已合并beanDefinition
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 初始化非抽象，非懒加载的bean
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                // 如果是工厂bean，给beanName加上&前缀，之后调用getBean()
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    // 如果是工厂bean，类型强转后做一些判断
                    FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged(
                            (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                            getAccessControlContext());
                    }
                    else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                       ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
            }
            else {
                // 非工厂也调用getBean()
                getBean(beanName);
            }
        }
    }

    // 触发bean初始化后回调
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
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



> 获取合并后的RootBeanDefinition：getMergedLocalBeanDefinition()

合并beanDefinition并将非RootBeanDefinition转换为RootBeanDefinition。

```java
// AbstractBeanFactory.java
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
    // 先尝试从缓存取
    RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
    // 从缓存中拿到，且不需要重新合并，直接返回
    if (mbd != null && !mbd.stale) {
        return mbd;
    }
    return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
}

//  AbstractBeanFactory.java
protected RootBeanDefinition getMergedBeanDefinition(String beanName, BeanDefinition bd)
    throws BeanDefinitionStoreException {
    return getMergedBeanDefinition(beanName, bd, null);
}

// AbstractBeanFactory.java
protected RootBeanDefinition getMergedBeanDefinition(
    String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
    throws BeanDefinitionStoreException {
	// 对缓存已合并beanDefintion加锁，防止并发操作
    synchronized (this.mergedBeanDefinitions) {
        // 合并之后的beanDefinition
        RootBeanDefinition mbd = null;
        // 之前的beanDefinition
        RootBeanDefinition previous = null;

        // 进行合并前再次尝试从缓存中回去，若其他线程已经做过合并，则不处理
        if (containingBd == null) {
            mbd = this.mergedBeanDefinitions.get(beanName);
        }

        // mbd为空或者需要重新合并
        if (mbd == null || mbd.stale) {
            previous = mbd;
            if (bd.getParentName() == null) {
                // 没有父类，即不需要合并
                if (bd instanceof RootBeanDefinition) {
                    // 如果是RootBeanDefinition类型，强转之后赋值给mbd
                    mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
                }
                else {
                    // 不是RootBeanDefinition类型，将
                    mbd = new RootBeanDefinition(bd);
                }
            }
            else {
                // 存储父类beanDefinition
                BeanDefinition pbd;
                try {
                    // 获取父类名称并处理
                    String parentBeanName = transformedBeanName(bd.getParentName());
                    if (!beanName.equals(parentBeanName)) {
                        // 与父类不同名，递归向上合并
                        pbd = getMergedBeanDefinition(parentBeanName);
                    }
                    else {
                        // 与父类同名，获取父类bean工厂
                        BeanFactory parent = getParentBeanFactory();
                        if (parent instanceof ConfigurableBeanFactory) {
                            // 父工厂是ConfigurableBeanFactory，递归向上合并
                            pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
                        }
                        else {
                            // 省略
                        }
                    }
                }
                catch (NoSuchBeanDefinitionException ex) {
                    // 省略
                }
                // 深拷贝
                mbd = new RootBeanDefinition(pbd);
                // 用子类属性重写父类属性
                mbd.overrideFrom(bd);
            }

            // 合并之后若没有作用域，默认设置为单例
            if (!StringUtils.hasLength(mbd.getScope())) {
                mbd.setScope(SCOPE_SINGLETON);
            }

            // 合并之后的beanDefinition是单例，但是containingBd不是单例，用containingBd的作用覆盖合并后的
            // containingBd不为空时，是一个内部的beanDefinition
            if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
                mbd.setScope(containingBd.getScope());
            }

            // 放入缓存，用于之后合并
            if (containingBd == null && isCacheBeanMetadata()) {
                this.mergedBeanDefinitions.put(beanName, mbd);
            }
        }
        // 当需要重新合并是
        if (previous != null) {
            copyRelevantMergedBeanDefinitionCaches(previous, mbd);
        }
        return mbd;
    }
}
```



## 三 getBean()方法

```java
// AbstractBeanFactory.java
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}

// AbstractBeanFactory.java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    // 如果传入的是工厂bean或者别名，进行处理转换
    final String beanName = transformedBeanName(name);
    Object bean;

    // ===================== 1. getSingleton() =====================
    // 1. 从缓存中获取bean，拿到的可能是工厂bean，也可能是bean实例
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        if (logger.isTraceEnabled()) {
          // 省略
        }
        // ===================== 2. getObjectForBeanInstance() =====================
        // 主要是处理工厂bean，如果是正常bean或者期望返回的就是工厂bean则会直接返回。
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {
        // spring只能解决单例的循环依赖，对于多例直接抛错
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 获取父类bean工厂
        BeanFactory parentBeanFactory = getParentBeanFactory();
        // 父bean工厂不为空，且不在beanDefinition缓存map中。
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // 名字转换成工厂bean
            String nameToLookup = originalBeanName(name);
            // 从父工厂中获取bean
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                    nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {
                // 如果创建bean是有传入参数，则委派给父工厂
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else if (requiredType != null) {
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
            else {
                // 从父工厂中查找bean
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }
		// 不需要进行类型检查，且已创建的集合set中也没有，则将该bean放进集合中
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            // 合并beanDefintion
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            // 检查beanDefinition是不是抽象的，如果是抛错
            checkMergedBeanDefinition(mbd, beanName, args);

            // 获取通过@DependsOn引入的所有beanDefinition名称
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    // 检查依赖循环
                    if (isDependent(beanName, dep)) {
                        // 省略
                    }
                    // 注册依赖关系
                    registerDependentBean(dep, beanName);
                    try {
                        // 获取依赖
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        // 省略
                    }
                }
            }

            // 2. 创建单例bean
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        // 创建出错销毁bean，并清理缓存
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
			// 创建多例bean
            else if (mbd.isPrototype()) {
             
                Object prototypeInstance = null;
                try {
                    // 放入ThreadLocal中
                    beforePrototypeCreation(beanName);
                    // 创建
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    // 从ThreadLocal中移除
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }
			// 创建其他作用域的bean
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
                    // 省略
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
           // 省略
        }
    }
    return (T) bean;
}
```



#### 1. getSingleton()：从缓存中查找bean ======TODO==========

```java
// DefaultSingletonBeanRegistry.java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 先从缓存map中取实例化好的bean
    Object singletonObject = this.singletonObjects.get(beanName);
    // 没有取到且正在创建
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 加锁，防止并发创建
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            // 早期bean缓存也没有
            if (singletonObject == null && allowEarlyReference) {
                // 从工厂bean缓存中取，如果有的话放入早期bean缓存中
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```



#### 2. getObjectForBeanInstance()

```java
/**
* beanInstance: 从缓存中取出来的bean
* name: 传进来的bean名称
* beanName: 对name处理过的beanName
* mbd：合并后的bean
*/
// AbstractBeanFactory.java
protected Object getObjectForBeanInstance(
    Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

    // beanName是不是&开头
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        // 空的bean直接返回
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }
        // 不是工厂bean直接报错
        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
        }
        // 工厂bean标志设为true
        if (mbd != null) {
            mbd.isFactoryBean = true;
        }
        // 直接返回，在spring低版本中，并没有在这返回，而是在后面又重新判断了是否以&开头
        return beanInstance;
    }

    // 普通的bean直接返回
    if (!(beanInstance instanceof FactoryBean)) {
        return beanInstance;
    }

    Object object = null;
    if (mbd != null) {
        // mdb工厂bean标识设置未true
        mbd.isFactoryBean = true;
    }
    else {
        // 根据传入beanName从缓存中取出工厂bean
        object = getCachedObjectForFactoryBean(beanName);
    }
    if (object == null) {
        // beanInstance强转为工厂bean
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        // 从beanDefinitionMap查看传入的bean定义是否加载过
        if (mbd == null && containsBeanDefinition(beanName)) {
            // 将工厂bean的引用合并放入已合并beanDefinition缓存中
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        // 检测bean是否是合成的
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        // ==========TODO===========
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
```

如果一个bean是工厂bean，那么通过beanName获取到的是工厂bean调用`getObject()`方法返回的引用，要获取bean的实例，需要加上`&`来获取真正bean。



## 四 创建bean

如果从缓存中没有获取到要查找的bean，那么接下来就要进行创建bean的流程了，在创建之前会做一些检查处理。

### 1. 检查处理 @DependsOn 循环依赖

```java
// DefaultSingletonBeanRegistry.java
protected boolean isDependent(String beanName, String dependentBeanName) {
    synchronized (this.dependentBeanMap) {
        return isDependent(beanName, dependentBeanName, null);
    }
}

// DefaultSingletonBeanRegistry.java
private boolean isDependent(String beanName, String dependentBeanName, @Nullable Set<String> alreadySeen) {
    // 如果已经检查过，直接返回
    // alreadySeen：存放的是已经检查过@DependsOn的
    if (alreadySeen != null && alreadySeen.contains(beanName)) {
        return false;
    }
    // 若传入的是别名，得到真正的名称
    String canonicalName = canonicalName(beanName);
    // 获取通过@DependsOn注解引入了自己的beanName缓存
    Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
    if (dependentBeans == null) {
        return false;
    }
    // 存在循环，返回true在外层抛错
    // 因为是先检查依赖关系，之后再放入缓存中的，所以当A和B存在依赖循环时，先检查A的时候并不会发现，
    // A只会把自己的依关系放入缓存dependentBeanMap中，B->A，当检查B时，获取到的set中存在A，则说明有依赖循环
    if (dependentBeans.contains(dependentBeanName)) {
        return true;
    }
    // dependentBeanName放入缓存前，检查是否与缓存中的存在依赖循环
    for (String transitiveDependency : dependentBeans) {
        if (alreadySeen == null) {
            alreadySeen = new HashSet<>();
        }
        // 放入已经检查过的缓存中  
        alreadySeen.add(beanName);
        // 递归检查
        if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
            return true;
        }
    }
    return false;
}


/**
* beanName: @DependsOn注解中的beanName
* dependentBeanName: 被@DependsOn注解修饰的beanName
*/
// DefaultSingletonBeanRegistry.java
public void registerDependentBean(String beanName, String dependentBeanName) {
    String canonicalName = canonicalName(beanName);

    synchronized (this.dependentBeanMap) {
        // computeIfAbsent：key不存在put，存在不做操作，返回旧值
        Set<String> dependentBeans =
            this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
        if (!dependentBeans.add(dependentBeanName)) {
            return;
        }
    }

    synchronized (this.dependenciesForBeanMap) {
        Set<String> dependenciesForBean =
            this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
        dependenciesForBean.add(canonicalName);
    }
}
```



### 2. getSingleton()

```java
// AbstractBeanFactory.java
getSingleton(beanName, () -> {
    try {
        return createBean(beanName, mbd, args);
    }
    catch (BeansException ex) {
        // 销毁bean
        destroySingleton(beanName);
        throw ex;
    }
});

// DefaultSingletonBeanRegistry.java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
        // 再次尝试从缓存中获取
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            // 正在销毁
            if (this.singletonsCurrentlyInDestruction) {
                // 抛错，省略
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
            // 检查是否在创建中
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet<>();
            }
            try {
                // 进入真正创建bean的方法
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            catch (IllegalStateException ex) {
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw ex;
                }
            }
            catch (BeanCreationException ex) {
                if (recordSuppressedExceptions) {
                    for (Exception suppressedException : this.suppressedExceptions) {
                        ex.addRelatedCause(suppressedException);
                    }
                }
                throw ex;
            }
            finally {
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                // 如果是新的单例bean，放入单例bean缓存中，并从早期bean缓存和工厂bean缓存中移除
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```



### 3. createBean()

```java
// AbstractAutowireCapableBeanFactory.java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// 获取bean class
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
            // 获取需要重写的父类方法
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// 处理BeanPostProcessors后置处理器，生成代理，放在之后AOP
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
            // 创建bean
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		// 省略catch块
	}

```



### 4. doCreateBean()

```java
// AbstractAutowireCapableBeanFactory.java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        // beanDefinition是单例，先尝试从工厂bean实例缓存中获取，同时删除缓存中的实例
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // ========================== 1. createBeanInstance() ===================
        // 从缓存中没有获取到，创建bean实例的包装
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                // ===========2. applyMergedBeanDefinitionPostProcessors() ===================
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
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isTraceEnabled()) {
            logger.trace("Eagerly caching bean '" + beanName +
                         "' to allow for resolving potential circular references");
        }
        
        // =========== 3. addSingletonFactory() ===================
        // 放入单例工厂缓存池，用于解决循环依赖
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // =========== 4. addSingletonFactory() ===================
        // 填充属性
        populateBean(beanName, mbd, instanceWrapper);
        // =========== 5. initializeBean() ===================
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        }
        else {
            throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    // 抛错省略
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



#### 4.1 TODO===createBeanInstance()

```java
// AbstractAutowireCapableBeanFactory.java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // 获取bean class
    Class<?> beanClass = resolveBeanClass(mbd, beanName);
	
    // bean class不为空，且修饰不是public，且不允许访问非public构造函数
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        // 抛错省略
    }
	// 获取bean的回调，如果有，下次通过回调创建
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }
	
    // ==========TODO
    if (mbd.getFactoryMethodName() != null) {
        // 如果有工厂方法，通过工厂方法创建bean
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // 是否解析过构造方法或工厂方法
    boolean resolved = false;
    // 自动装配
    boolean autowireNecessary = false;
    // args：构造方法的参数
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            // resolvedConstructorOrFactoryMethod用于缓存的解析过构造方法或工厂方法
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                // 已经解析过
                resolved = true;
                // 是否解析过构造方法参数
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            // 构造方法已经解析过且构造方法参数也解析过，使用自动装配创建
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            // 默认的无法构造创建
            return instantiateBean(beanName, mbd);
        }
    }

    // 通过后置处理器获取构造方法
    // SmartInstantiationAwareBeanPostProcessor接口继承了BeanPostProcessor，是一种bean后置处理器
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
        mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // Preferred constructors for default construction?
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
        return autowireConstructor(beanName, mbd, ctors, null);
    }

    // 无参构造方法创建
    return instantiateBean(beanName, mbd);
}
```



#### 4.2  TODO==applyMergedBeanDefinitionPostProcessors()

```java

```



#### 4.3 addSingletonFactory()

```java
// DefaultSingletonBeanRegistry.java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            this.singletonFactories.put(beanName, singletonFactory);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}
```





#### 4.4 populateBean()

```java
// AbstractAutowireCapableBeanFactory.java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {

    if (bw == null) {
         // 实例为空且有属性值
        if (mbd.hasPropertyValues()) {
            // 省略报错
        }
        else {
            // 空的实例直接返回.
            return;
        }
    }

    
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		// 获取所有的bean后置处理器
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                // 执行实例化后，填充属性前的回调
                // 自动装配注入字段前也是在这里回调
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    return;
                }
            }
        }
    }

    // 获取所有属性
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // 获取@Autowire
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // 处理根据name注入的
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // 处理根据type注入的
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    // 又是后置处理器，用于处理@Autowire
    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    if (filteredPds == null) {
                        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                    }
                    pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvsToUse == null) {
                        return;
                    }
                }
                pvs = pvsToUse;
            }
        }
    }
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    if (pvs != null) {
        // 将属性设置到bean中
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```





#### 4.5 initializeBean()

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        // 回调 xxAware接口
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 执行bean初始化前的后置处理方法
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 执行自定义的初始化方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 初始化后的后置处理器方法
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```





## 1.2 初始化之后回调：afterSingletonsInstantiated()