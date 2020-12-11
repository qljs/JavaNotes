[TOC]





## 1. Atomic

​		Atomic 是指一个操作是不可中断的操作，Atomic包里的类基本都是使用`Unsafe`实现的包装类。根据操作的数据类型，可以将JUC包中的原子类分为4类：

> #### 基本类型

- AtomicInteger：整型原子类
- AtomicLong：长整型原子类
- AtomicBoolean ：布尔型原子类

