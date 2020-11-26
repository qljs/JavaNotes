# Spring事务源码学习



## 一 Spring事务三大核心接口

### 1. PlatformTransactionManager：事务管理器

Spring通过这个接口向第三方（例如：Jdbc）提供了管理事务的功能。

```java
public interface PlatformTransactionManager extends TransactionManager {

   	/**
   	* 获取事务状态
   	*/
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
        throws TransactionException;

    /**
    * 提交事务
    */
    void commit(TransactionStatus status) throws TransactionException;

    /**
    * 回滚事务
    */ 
    void rollback(TransactionStatus status) throws TransactionException;
}
```



### 2. TransactionDefinition：事务定义

在该接口中定了事务的常量，例如隔离界别、传播行为。

```java
public interface TransactionDefinition {

	// 支持当前事务，没有则新建一个事务
	int PROPAGATION_REQUIRED = 0;

	// 如果当前有事务，加入当前事务；若没有，则以非事务的方式执行官
	int PROPAGATION_SUPPORTS = 1;

	// 如果挡墙有事务，加入当前事务；若没有则抛出异常
	int PROPAGATION_MANDATORY = 2;

	// 创建一个新的事务，若当前有事务，则将当前事务挂起
	int PROPAGATION_REQUIRES_NEW = 3;

	// 以非事务的方式执行，若当前有事务则将事务挂起
	int PROPAGATION_NOT_SUPPORTED = 4;

	// 以非事务方式运行，当前存在事务抛错
	int PROPAGATION_NEVER = 5;

	// 若当前存在事务，则在嵌套事务中执行，否则类似于PROPAGATION_REQUIRED
	int PROPAGATION_NESTED = 6;


	// 使用数据库默认隔离级别；MySQL:可重复读；Oracle：读已提交
	int ISOLATION_DEFAULT = -1;

	// 读未提交；允许读取未提交的事务，可能会导致脏读、幻读、不可重复读
	int ISOLATION_READ_UNCOMMITTED = 1;  // same as java.sql.Connection.TRANSACTION_READ_UNCOMMITTED;

	// 读已提交，可以防止脏读，但可能会有幻读、不可重复读
	int ISOLATION_READ_COMMITTED = 2;  // same as java.sql.Connection.TRANSACTION_READ_COMMITTED;

	// 可重复度；防止脏读和不可重复读，但可能有幻读
	int ISOLATION_REPEATABLE_READ = 4;  // same as java.sql.Connection.TRANSACTION_REPEATABLE_READ;

	// 防止脏读、不可重读读、幻读
	int ISOLATION_SERIALIZABLE = 8;  // same as java.sql.Connection.TRANSACTION_SERIALIZABLE;


	// 超时时间
	int TIMEOUT_DEFAULT = -1;


	/**
	 * 获取事务传播行为
	 */
	default int getPropagationBehavior() {
		return PROPAGATION_REQUIRED;
	}

	/**
	 * 获取事务隔离级别
	 */
	default int getIsolationLevel() {
		return ISOLATION_DEFAULT;
	}

	/**
	 * 获取超时时间
	 */
	default int getTimeout() {
		return TIMEOUT_DEFAULT;
	}

	/**
	 * 当前是否是只读事务
	 */
	default boolean isReadOnly() {
		return false;
	}

	/**
	 * 获取事务名称
	 */
	@Nullable
	default String getName() {
		return null;
	}


	// Static builder methods

	/**
	 * 获取默认值的事务定义
	 */
	static TransactionDefinition withDefaults() {
		return StaticTransactionDefinition.INSTANCE;
	}

}
```



### 3. TransactionStatus/TransactionExecution：事务状态

该接口用于获取或判断事务状态，从5.2版本开始将通用部分抽取到了`TransactionExecution`。

```java
public interface TransactionStatus extends TransactionExecution, SavepointManager, Flushable {

    //============================ TransactionExecution ==================
    /**
	 * 是否是新事务
	 */
	boolean isNewTransaction();

	/**
	 * 设置为只回滚
	 */
	void setRollbackOnly();

	/**
	 * 是否只回滚
	 */
	boolean isRollbackOnly();

	/**
	 * 是否已提交
	 */
	boolean isCompleted();
    //============================ TransactionExecution ==================
    
	/**
	 * 是否有保留点，用于事务嵌套
	 */
	boolean hasSavepoint();

	/**
	 * 刷新
	 */
	@Override
	void flush();

}
```

