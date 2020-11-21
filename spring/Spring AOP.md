[TOC]



# SpringAOP源码



## 一 前言

看过SpringIOC源码的应该知道，代理对象是通过后置处理器`InstantiationAwareBeanPostProcessor`生成的。

```java
// AbstractAutowireCapableBeanFactory.java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {

    // ..................
    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        // 给后置处理一个返回代理而不是目标bean实例的机会。
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    // ............
}
```

查看`InstantiationAwareBeanPostProcessor`接口的实现，可以看到其中有一个`AbstractAutoProxyCreator`的实现类，根据名字可以推断出，这是一个代理创建器。

<img src="../images/springAOP/1605928485465.png" align="left">

可以先看下该类的注释：

> {@link org.springframework.beans.factory.config.BeanPostProcessor} implementation that wraps each eligible bean with an AOP proxy, delegating to specified interceptors  before invoking the bean itself.

翻译后：

> 实现了BeanPostProcessor，用一个AOP代理包装每个合格的bean，在调用bean本身之前委托给指定的拦截器。

可以基本确定，代理是在该类或子类中创建的。



本文是基于注解的方式分析的，所以先看下`AspectJAwareAdvisorAutoProxyCreator`的类关系图。

![1605927083780](../images/springAOP/1605927083780.png)



可以看到`AbstractAutoProxyCreator`实现了`BeanFactoryAware`接口，在该接口中只有一个`setBeanFactory(BeanFactory beanFactory)`方法。

下面看下`AbstractAutoProxyCreator`在`setBeanFactory`方法中做了什么。

```java
// AbstractAutoProxyCreator.java
@Override
public void setBeanFactory(BeanFactory beanFactory) {
    // 给beanFactory赋值
    this.beanFactory = beanFactory;
}
```

可以看到`AbstractAutoProxyCreator`类中只是设置beanFactory，而在其子类`AbstractAdvisorAutoProxyCreator`重写了`setBeanFactory`方法。

```java
// AbstractAdvisorAutoProxyCreator.java
@Override
public void setBeanFactory(BeanFactory beanFactory) {
    // 调用父类setBeanFactory()方法
    super.setBeanFactory(beanFactory);
    if (!(beanFactory instanceof ConfigurableListableBeanFactory)) {
        throw new IllegalArgumentException(
            "AdvisorAutoProxyCreator requires a ConfigurableListableBeanFactory: " + beanFactory);
    }
    // 初始化bean工厂
    initBeanFactory((ConfigurableListableBeanFactory) beanFactory);
}
```

而`initBeanFactory()`方法又被子类`AnnotationAwareAspectJAutoProxyCreator`重写了，该方法主要是创建了AOP的增强器，以`AspectJ`注入。

```java
// AnnotationAwareAspectJAutoProxyCreator.java
@Override
protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    super.initBeanFactory(beanFactory);
    if (this.aspectJAdvisorFactory == null) {
        this.aspectJAdvisorFactory = new ReflectiveAspectJAdvisorFactory(beanFactory);
    }
    this.aspectJAdvisorsBuilder =
        new BeanFactoryAspectJAdvisorsBuilderAdapter(beanFactory, this.aspectJAdvisorFactory);
}
```



同时`AbstractAutoProxyCreator`也实现了`BeanPostProcessor`，在`BeanPostProcessor`接口中只有两个方法，分别是`postProcessBeforeInitialization`()和`postProcessAfterInitialization`()，用于bean初始化前后操作bean。

先看下`postProcessBeforeInitialization()`方法做了什么。

```java
// AbstractAutoProxyCreator.java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    // 判断是否是FactoryBean，若是就为名字加上&前缀
    Object cacheKey = getCacheKey(beanClass, beanName);

    // beanName不为空或者targetSourcedBeans缓存中不包含该beanName
    if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
        // 是否被代理过
        if (this.advisedBeans.containsKey(cacheKey)) {
            return null;
        }
        // 如果是基础类或者需要跳过的，表示不需要被代理
        if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
            // 放入缓存
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return null;
        }
    }

    // 获取自定义的TargetSource
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    if (targetSource != null) {
        if (StringUtils.hasLength(beanName)) {
            this.targetSourcedBeans.add(beanName);
        }
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        // 创建代理
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    return null;
}
```

