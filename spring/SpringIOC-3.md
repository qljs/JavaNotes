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





## 二 preInstantiateSingletons()

```java
// DefaultListableBeanFactory.java
public void preInstantiateSingletons() throws BeansException {
    if (logger.isTraceEnabled()) {
        logger.trace("Pre-instantiating singletons in " + this);
    }

    // 拿到工厂中所有beanDefinition名称
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    for (String beanName : beanNames) {
        // ================== 1. getMergedLocalBeanDefinition ===========
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
        // 获取bean实例
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
                // 执行SmartInitializingSingleton的afterSingletonsInstantiated方法
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```



#### 1. getMergedLocalBeanDefinition()

`getMergedLocalBeanDefinition()`在spring中是个频繁使用的方法，主要用于获取合并的beanDefinition，返回`RootBeanDefinition`类型的beanDefinition。

```java
// AbstractBeanFactory.java
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
    // 先尝试从缓存取
    // mergedBeanDefinitions是一个final ConcurrentHashMap，用于存储已合并的beanDefinition
    // key：beanName, value为RootBeanDefinition
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



## 三 doGetBean

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
        // 主要用于处理工厂bean
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

    // 如果有传入bean的类型，进行类型转换
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



#### 1. getSingleton(String beanName, boolean allowEarlyReference)

`allowEarlyReference`：是否允许早期引用，true的时候，若存在循环依赖则去`singletonFactories`中查找。

`singletonObjects`：用于存储实例化的bean；

`earlySingletonObjects`：存储早期的bean，在处理循环依赖后，bean会从；

`singletonFactories`：存储工厂bean的引用，用于处理循环依赖。

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

`getObjectForBeanInstance()`也是spring中一个高频的方法，主要是用来判断bean是否是一个正常的bean或者工厂bean，并处理工厂bean的引用，获取真正得到工厂bean实例。

如果一个bean是工厂bean，那么通过beanName获取到的是工厂bean调用`getObject()`方法返回的引用，要获取bean的实例，需要加上`&`来获取真正bean。

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
        // ========== 2.1 getObjectFromFactoryBean() =====
        // 获取实例
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
```

1. 如果是普通bean或者工厂bean直接返回；
2. 处理工厂bean的引用。



#### 2.1 getObjectFromFactoryBean()

```java
// FactoryBeanRegistrySupport.java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    // FactoryBean是单例且在缓存中
    if (factory.isSingleton() && containsSingleton(beanName)) {
        synchronized (getSingletonMutex()) {
            // factoryBeanObjectCache：由FactoryBean创建的单例缓存
            Object object = this.factoryBeanObjectCache.get(beanName);
            if (object == null) {
                // =========== 2.2 doGetObjectFromFactoryBean() =====
                // 缓存中没有，通过FactoryBean.getObject()获取或得到一个NullBean
                object = doGetObjectFromFactoryBean(factory, beanName);
                // 再次从缓存总获取，
                Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                if (alreadyThere != null) {
                    object = alreadyThere;
                }
                else {
                    // 缓存中没有取到，且需要执行后置处理
                    if (shouldPostProcess) {
                        if (isSingletonCurrentlyInCreation(beanName)) {
                            // 正在创建中，返回
                            return object;
                        }
          				// 进行检查，并尝试放入正在创建的bean缓存
                        beforeSingletonCreation(beanName);
                        try {
                            // 执行后置处理器
                            object = postProcessObjectFromFactoryBean(object, beanName);
                        }
                        catch (Throwable ex) {
                            // 省略抛错
                        }
                        finally {
                            // 检查并尝试从缓存删除
                            afterSingletonCreation(beanName);
                        }
                    }
                    if (containsSingleton(beanName)) {
                        // 放入缓存中
                        this.factoryBeanObjectCache.put(beanName, object);
                    }
                }
            }
            return object;
        }
    }
    else {
        // 不是单例或者没在缓存中，返回FactoryBean.getObject()或NullBean
        Object object = doGetObjectFromFactoryBean(factory, beanName);
        if (shouldPostProcess) {
            try {
                // 执行后置处理器
                object = postProcessObjectFromFactoryBean(object, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
            }
        }
        return object;
    }
}
```

1. 如果是单例且在单例对象缓存中，从FactoryBean的单例缓存中获取，没有取到调用真正的获取实例方法`doGetObjectFromFactoryBean()`；
2.  如果需要执行后置处理，则执行`postProcessObjectFromFactoryBean`进行后置处理，并尝试放入缓存；
3. 若非单例或在单例缓存池中不存在，同样执行`doGetObjectFromFactoryBean`方法来获取实例，之后执行后置处理器。



#### 2.2  doGetObjectFromFactoryBean()

```java
private Object doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName) throws BeanCreationException {
    Object object;
    try {
        // 安全检查
        if (System.getSecurityManager() != null) {
            AccessControlContext acc = getAccessControlContext();
            try {
                // 获取FactoryBean的上下文
                object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            // 获取工厂bean的实例
            object = factory.getObject();
        }
    }
    catch (FactoryBeanNotInitializedException ex) {
        throw new BeanCurrentlyInCreationException(beanName, ex.toString());
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
    }

    // 获取到的实例为空，返回NullBean
    if (object == null) {
        if (isSingletonCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(
                beanName, "FactoryBean which is currently in creation returned null from getObject");
        }
        object = new NullBean();
    }
    return object;
}
```





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

创建之前，先尝试从缓存中获取，没有获取到，则进行创建。

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
            // 判断是否忽略检查并尝试放入正在创建中的缓存
            beforeSingletonCreation(beanName);
            // 是否是一个新的单例bean
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
                // 尝试从正在创建中的缓存里移除
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
        // 处理BeanPostProcessors后置处理器，生成代理
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
        // ========================== 4.1 createBeanInstance() ===================
        // 从缓存中没有获取到，创建bean实例的包装
        // 有工厂方法，先用工厂方法创建，否则根据参数确定有参或无参构造创建。
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // 获取bean实例
    final Object bean = instanceWrapper.getWrappedInstance();
    // 获取bean class
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                // MergedBeanDefinitionPostProcessor后置处理处理@Autowire,@Value注解
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }

    // 单例、允许循环引用且在创建中
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isTraceEnabled()) {
            logger.trace("Eagerly caching bean '" + beanName +
                         "' to allow for resolving potential circular references");
        }
        
        // =========== 4.2 addSingletonFactory() ===================
        // 放入单例工厂缓存池，用于解决循环依赖
        // 为了解决循环依赖，将bean放入singletonFactories缓存中，若在后置处理中没有处理，
        // SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference() 默认返回bean的引用
        // 该接口通常用于spring内部
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // =========== 4.3 populateBean() ===================
        // 填充属性
        populateBean(beanName, mbd, instanceWrapper);
        // =========== 4.4 initializeBean() ===================
        // 调用bean的后置处理，执行自定的初始化方法
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
       // 省略
    }

    if (earlySingletonExposure) {
        // getSingleton()的第二个参数allowEarlyReference是false时，不会去singletonFactories去查找
        // 所以，earlySingletonReference不为空时，代表bean存在循环依赖且已经拿到早期引用了
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                // 在后置处理中被增强过
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

    // 把销毁方法注册到缓存中，用于销毁bean时调用
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

1. 先尝试从FactoryBean创建的bean缓存中获取，没取到，则调用`createBeanInstance()`方法创建实例；
2. 调用`MergedBeanDefinitionPostProcessor`后置处理器，处理`@AutoWire`注解；
3. 如果允许循环引用，则将bean的引用放入缓存中；
4. 调用`populateBean()`方法进行属性填充，在这里会发现有没有循环依赖，有的话进行处理；
5. 调用`initializeBean()`方法进行初始化操作；
6. 注册销毁方法。



#### 4.1 createBeanInstance()

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
	
    if (mbd.getFactoryMethodName() != null) { 
        // 如果有工厂方法，通过工厂方法创建bean，即factory-method配置的方法
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
    // 主要是通过参数数量以及类型来确认构造方法，之后反射执行构造方法获取实例
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
        mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // 使用默认的构造函数
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
        return autowireConstructor(beanName, mbd, ctors, null);
    }

    // 无参构造方法创建
    return instantiateBean(beanName, mbd);
}
```

1. 若实现了`Supplier<T>`接口，则通过回调创建实例;
2. 配置了`factory-method`则通过工厂方法创建实例；
3. 根据参数或后置处理器，确定构造方法来创建实例。



#### 4.2 addSingletonFactory()

如果bean不存在于singletonObjects缓存中，将其放入singletonFactories，用于处理依赖循环

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



#### 4.3 populateBean() 

在该方法中进行属性注入

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
                // 执行实例化后，填充属性前的回调，自动装配注入前也是在这里回调
                // 用户可以在这里自定义属性注入
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    return;
                }
            }
        }
    }

    // 获取所有属性
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // 处理注入模型
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // 处理根据name注入的
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            // ======== 4.3.1 autowireByName() ======
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // 处理根据type注入的
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            // ======== 4.3.2 autowireByName() ======
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

    // 是否实现了InstantiationAwareBeanPostProcessorsi接口
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    // 是否需要依赖检查
    // DEPENDENCY_CHECK_NONE：不检查
    // DEPENDENCY_CHECK_OBJECTS：检查对象引用
    // DEPENDENCY_CHECK_SIMPLE：检查属性依赖
    // DEPENDENCY_CHECK_ALL：全都检查
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    // 该后置处理器用于对属性的值修改
    // 例如用于处理@AutoWired和@Value的接口：AutowiredAnnotationBeanPostProcessor
    // 该接口继承了InstantiationAwareBeanPostProcessorAdapter，会在这里处理给属性设置值
    // 如果是@AutoWired注入造成成依赖循环的，在这里会处理
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
                // 替换原来的属性
                pvs = pvsToUse;
            }
        }
    }
    
    // 检查依赖
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    if (pvs != null) {
        // ============== 4.3.3 applyPropertyValues() ===========
        // 将属性设置到bean包装器中
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

1. 获取beanWrapper属性列表；
2. 若是实现了`InstantiationAwareBeanPostProcessor`接口，调用`postProcessAfterInstantiation()`处理属性；
3. 根据类型判断自动装配方法；
4. 若实现了`InstantiationAwareBeanPostProcessor`接口，调用`postProcessProperties`修改属性值；
5. 讲属性设置到beanWrapper中。



##### 4.3.1 autowireByName()

```java
// AbstractAutowireCapableBeanFactory.java
protected void autowireByName(
    String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

    // 获取非简单的属性名称
    // spring中简单属性：
    // 1. CharSequence
    // 2. Enum
    // 3. Date
    // 4. URI/URL
    // 5. Number的继承类
    // 6. 基本类型
    // 7. Locale
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        // 检查该属性是否在单例bean缓存或beanDefinition缓存中
        if (containsBean(propertyName)) {
            // 调用getBean()
            Object bean = getBean(propertyName);
            // 设置属性
            pvs.add(propertyName, bean);
            // 注册依赖关系
            registerDependentBean(propertyName, beanName);
            if (logger.isTraceEnabled()) {
                logger.trace("Added autowiring by name from bean name '" + beanName +
                             "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
            }
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                             "' by name: no matching bean found");
            }
        }
    }
}
```

1. 获取所有属性名称，判断是否在单例bean缓存或beanDefinition缓存中；
2. 调用`getBean()`获取或创建bean；
3. 注册依赖关系。



##### 4.3.2 autowireByType()

```java
protected void autowireByType(
    String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

    // 获取类型转换器
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }

    Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
    // 获取非简单bean的属性
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        try {
            // 获取属性描述器
            PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
            // Object类型不处理
            if (Object.class != pd.getPropertyType()) {
                // 获取setter方法中的参数信息
                MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
                // 实现PriorityOrdered接口的是组优先级最高的bean，这种bean在自动装配是不会去检查factoryBean
                boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
                // 将参数封装成依赖描述
                DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                // 解析出依赖，
                Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
                if (autowiredArgument != null) {
                    // 解析出来的bean放入属性值列表中国
                    pvs.add(propertyName, autowiredArgument);
                }
                // 注册依赖关系
                for (String autowiredBeanName : autowiredBeanNames) {
                    registerDependentBean(autowiredBeanName, beanName);
                    if (logger.isTraceEnabled()) {
                        logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
                                     propertyName + "' to bean named '" + autowiredBeanName + "'");
                    }
                }
                autowiredBeanNames.clear();
            }
        }
        catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
        }
    }
}
```



##### 4.3.3  applyPropertyValues()

在将属性解析获取到属性列表pvs后，本方法会属性设置到bean中。

```java
// AbstractAutowireCapableBeanFactory.java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    if (pvs.isEmpty()) {
        return;
    }

    if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
        ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
    }

    MutablePropertyValues mpvs = null;
    // 属性值列表
    List<PropertyValue> original;
    if (pvs instanceof MutablePropertyValues) {
        mpvs = (MutablePropertyValues) pvs;
        if (mpvs.isConverted()) {
            // 属性列表被转换过，直接返回
            try {
                bw.setPropertyValues(mpvs);
                return;
            }
            catch (BeansException ex) {
                throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Error setting property values", ex);
            }
        }
        // 获取属性值列表
        original = mpvs.getPropertyValueList();
    }
    else {
        original = Arrays.asList(pvs.getPropertyValues());
    }

    // 获取自定义的转化器
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }
    // 获取beanDefinition值的解析器
    BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

    // 存储属性值的引用
    List<PropertyValue> deepCopy = new ArrayList<>(original.size());
    boolean resolveNecessary = false;
    for (PropertyValue pv : original) {
        if (pv.isConverted()) {
            // 属性值被转换过，直接拿
            deepCopy.add(pv);
        }
        else {
            String propertyName = pv.getName();
            // 原值
            Object originalValue = pv.getValue();
            // AutowiredPropertyMarker：用于单独自动装配属性值的简单标记类
            if (originalValue == AutowiredPropertyMarker.INSTANCE) {
                // 获取set方法
                Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
                if (writeMethod == null) {
                    throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
                }
                // 新建属性描述器，将依赖设置为：eager
                originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
            }
            // ========== 4.3.4 resolveValueIfNecessary() ========
            // 解析工厂中的其他bean引用，如果有循环依赖，将在这里发现处理
            Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
            Object convertedValue = resolvedValue;
            // 是否可以转换
            // isWritableProperty：是否可写
            // isNestedOrIndexedProperty：是否是indexed或nested(嵌套)
            boolean convertible = bw.isWritableProperty(propertyName) &&
                !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
            if (convertible) {
                // 转化属性值的类型
                convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
            }
            // resolvedValue是autowireByName或autowireByType解析出来的，结果为true
            if (resolvedValue == originalValue) {
                if (convertible) {
                    // 转化后的值和原值相同，缓存起来，下次获取直接从缓存中取
                    pv.setConvertedValue(convertedValue);
                }
                deepCopy.add(pv);
            }
            // 可以转换，且原始值是TypedStringValue类型，且转换后的不是集合或数组
            else if (convertible && originalValue instanceof TypedStringValue &&
                     !((TypedStringValue) originalValue).isDynamic() &&
                     !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
                pv.setConvertedValue(convertedValue);
                deepCopy.add(pv);
            }
            else {
                resolveNecessary = true;
                deepCopy.add(new PropertyValue(pv, convertedValue));
            }
        }
    }
    if (mpvs != null && !resolveNecessary) {
        mpvs.setConverted();
    }

    // Set our (possibly massaged) deep copy.
    try {
        // 将属性值设置到bean中，最终调用setter方法进行赋值。
        bw.setPropertyValues(new MutablePropertyValues(deepCopy));
    }
    catch (BeansException ex) {
        throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Error setting property values", ex);
    }
}
```

1. 获取所有属性值，判断是否转换过，若转换过，则直接返回；
2. 循环处理所有的属性值，`resolveValueIfNecessary()`会解析对其他bean的引用，如果存在循环依赖，则会在这里处理；
3. 将属性值注入到bean中。



##### 4.3.4 resolveValueIfNecessary()

解析其他bean的引用

```java
// BeanDefinitionValueResolver.java
public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
    // c处理RuntimeBeanReference类型，必须解析的类型
    if (value instanceof RuntimeBeanReference) {
        RuntimeBeanReference ref = (RuntimeBeanReference) value;
        // ============ 4.3.5 resolveReference() ===========
        // 解析引用
        return resolveReference(argName, ref);
    }
    // 处理RuntimeBeanNameReference
    else if (value instanceof RuntimeBeanNameReference) {
        String refName = ((RuntimeBeanNameReference) value).getBeanName();
        refName = String.valueOf(doEvaluate(refName));
        if (!this.beanFactory.containsBean(refName)) {
            throw new BeanDefinitionStoreException(
                "Invalid bean name '" + refName + "' in bean reference for " + argName);
        }
        return refName;
    }
    // 处理BeanDefinitionHolder
    else if (value instanceof BeanDefinitionHolder) {
        // Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
        BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
        return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
    }
    // 处理resolveInnerBean
    else if (value instanceof BeanDefinition) {
        // Resolve plain BeanDefinition, without contained name: use dummy name.
        BeanDefinition bd = (BeanDefinition) value;
        String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR +
            ObjectUtils.getIdentityHexString(bd);
        return resolveInnerBean(argName, innerBeanName, bd);
    }
    // 处理DependencyDescriptor
    else if (value instanceof DependencyDescriptor) {
        Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
        Object result = this.beanFactory.resolveDependency(
            (DependencyDescriptor) value, this.beanName, autowiredBeanNames, this.typeConverter);
        for (String autowiredBeanName : autowiredBeanNames) {
            if (this.beanFactory.containsBean(autowiredBeanName)) {
                this.beanFactory.registerDependentBean(autowiredBeanName, this.beanName);
            }
        }
        return result;
    }
    // 处理数组
    else if (value instanceof ManagedArray) {
        // May need to resolve contained runtime references.
        ManagedArray array = (ManagedArray) value;
        Class<?> elementType = array.resolvedElementType;
        if (elementType == null) {
            String elementTypeName = array.getElementTypeName();
            if (StringUtils.hasText(elementTypeName)) {
                try {
                    elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
                    array.resolvedElementType = elementType;
                }
                catch (Throwable ex) {
                    // Improve the message by showing the context.
                    throw new BeanCreationException(
                        this.beanDefinition.getResourceDescription(), this.beanName,
                        "Error resolving array type for " + argName, ex);
                }
            }
            else {
                elementType = Object.class;
            }
        }
        return resolveManagedArray(argName, (List<?>) value, elementType);
    }
    // 处理list
    else if (value instanceof ManagedList) {
        // May need to resolve contained runtime references.
        return resolveManagedList(argName, (List<?>) value);
    }
    // 处理set
    else if (value instanceof ManagedSet) {
        // May need to resolve contained runtime references.
        return resolveManagedSet(argName, (Set<?>) value);
    }
    // 处理map
    else if (value instanceof ManagedMap) {
        // May need to resolve contained runtime references.
        return resolveManagedMap(argName, (Map<?, ?>) value);
    }
    else if (value instanceof ManagedProperties) {
        Properties original = (Properties) value;
        Properties copy = new Properties();
        original.forEach((propKey, propValue) -> {
            if (propKey instanceof TypedStringValue) {
                propKey = evaluate((TypedStringValue) propKey);
            }
            if (propValue instanceof TypedStringValue) {
                propValue = evaluate((TypedStringValue) propValue);
            }
            if (propKey == null || propValue == null) {
                throw new BeanCreationException(
                    this.beanDefinition.getResourceDescription(), this.beanName,
                    "Error converting Properties key/value pair for " + argName + ": resolved to null");
            }
            copy.put(propKey, propValue);
        });
        return copy;
    }
    else if (value instanceof TypedStringValue) {
        // Convert value to target type here.
        TypedStringValue typedStringValue = (TypedStringValue) value;
        Object valueObject = evaluate(typedStringValue);
        try {
            Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
            if (resolvedTargetType != null) {
                return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
            }
            else {
                return valueObject;
            }
        }
        catch (Throwable ex) {
            // Improve the message by showing the context.
            throw new BeanCreationException(
                this.beanDefinition.getResourceDescription(), this.beanName,
                "Error converting typed String value for " + argName, ex);
        }
    }
    else if (value instanceof NullBean) {
        return null;
    }
    else {
        return evaluate(value);
    }
}
```



##### 4.3.5 resolveReference()

解析bean的依赖引用

```java
// BeanDefinitionValueResolver.java
private Object resolveReference(Object argName, RuntimeBeanReference ref) {
    try {
        Object bean;
        // 获取引用类型
        Class<?> beanType = ref.getBeanType();
        // 是否对父工厂的引用
        if (ref.isToParent()) {
            // 获取父工厂
            BeanFactory parent = this.beanFactory.getParentBeanFactory();
            if (parent == null) {
                throw new BeanCreationException(
                    this.beanDefinition.getResourceDescription(), this.beanName,
                    "Cannot resolve reference to bean " + ref +
                    " in parent factory: no parent factory available");
            }
            if (beanType != null) {
                // 类型不为空，根据类型获取bean
                bean = parent.getBean(beanType);
            }
            else {
                // 根据name获取bean
                bean = parent.getBean(String.valueOf(doEvaluate(ref.getBeanName())));
            }
        }
        else {
            String resolvedName;
            if (beanType != null) {
                NamedBeanHolder<?> namedBean = this.beanFactory.resolveNamedBean(beanType);
                bean = namedBean.getBeanInstance();
                resolvedName = namedBean.getBeanName();
            }
            else {
                resolvedName = String.valueOf(doEvaluate(ref.getBeanName()));
                bean = this.beanFactory.getBean(resolvedName);
            }
            // 注册依赖
            this.beanFactory.registerDependentBean(resolvedName, this.beanName);
        }
        if (bean instanceof NullBean) {
            bean = null;
        }
        return bean;
    }
    catch (BeansException ex) {
        throw new BeanCreationException(
            this.beanDefinition.getResourceDescription(), this.beanName,
            "Cannot resolve reference to bean '" + ref.getBeanName() + "' while setting " + argName, ex);
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
        // 执行 xxAware接口的回调
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 执行后置处理器的before方法
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
        // 执行后置处理器after方法
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```



