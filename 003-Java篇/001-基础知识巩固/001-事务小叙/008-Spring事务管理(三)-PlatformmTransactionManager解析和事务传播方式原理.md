# <font color='#FF6347'>Spring事务管理(三)-PlatformmTransactionManager解析和事务传播方式原理</font>

Spring在事务管理时，对事务的处理做了极致的抽象，即PlatformTransactionManager。对事务的操作，简单地来说，只有三步操作：获取事务，提交事务，回滚事务。

```java
public interface PlatformTransactionManager {
	// 获取事务
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;
	// 提交事务
	void commit(TransactionStatus status) throws TransactionException;
	// 回滚事务
	void rollback(TransactionStatus status) throws TransactionException;
}
```

当然Spring不会仅仅只提供一个接口，同时会有一个抽象模版类，实现了事务管理的具体骨架。AbstractPlatformTransactionManager类可以说是Spring事务管理的控制台，决定事务如何创建，提交和回滚。

在[Spring事务管理(二)-TransactionProxyFactoryBean原理](https://my.oschina.net/u/2377110/blog/1611531)中，分析TransactionInterceptor增强时，在invoke方法中最重要的三个操作：

- 创建事务 createTransactionIfNecessary
- 异常后事务处理 completeTransactionAfterThrowing
- 方法执行成功后事务提交 commitTransactionAfterReturning

在具体操作中，最后都是通过事务管理器PlatformTransactionManager的接口实现来执行的，其实也就是上面列出的三个接口方法。我们分别介绍这三个方法的实现，并以DataSourceTransactionManager为实现类观察JDBC方式事务的具体实现。

## 1. 获取事务

getTransaction方法根据事务定义来获取事务状态，事务状态中记录了事务定义，事务对象及事务相关的资源信息。对于事务的获取，除了调用事务管理器的实现来获取事务对象本身外，另外的很重要的一点是处理了事务的传播方式。

```java
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
  // 1.获取事务对象
  Object transaction = doGetTransaction();

  // Cache debug flag to avoid repeated checks.
  boolean debugEnabled = logger.isDebugEnabled();

  if (definition == null) {
    // Use defaults if no transaction definition given.
    definition = new DefaultTransactionDefinition();
  }

  // 2.如果已存在事务，根据不同的事务传播方式处理获取事务
  if (isExistingTransaction(transaction)) {
    // Existing transaction found -> check propagation behavior to find out how to behave.
    return handleExistingTransaction(definition, transaction, debugEnabled);
  }

  // Check definition settings for new transaction.
  if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
    throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
  }

  // 3. 如果当前没有事务，不同的事务传播方式不同处理方式
  // 3.1 事务传播方式为mandatory(强制必须有事务)，则抛出异常
  // No existing transaction found -> check propagation behavior to find out how to proceed.
  if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
    throw new IllegalTransactionStateException(
        "No existing transaction found for transaction marked with propagation 'mandatory'");
  }
  // 3.2 事务传播方式为required或required_new或nested(嵌套)，创建一个新的事务状态
  else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
      definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
      definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
    SuspendedResourcesHolder suspendedResources = suspend(null);
    if (debugEnabled) {
      logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
    }
    try {
      boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
      // 创建新的事务状态对象
      DefaultTransactionStatus status = newTransactionStatus(
          definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
      // 事务初始化
      doBegin(transaction, definition);
      // 准备其他同步操作
      prepareSynchronization(status, definition);
      return status;
    }
    catch (RuntimeException | Error ex) {
      resume(null, suspendedResources);
      throw ex;
    }
  }
  // 3.3 其他事务传播方式，返回一个事务对象为null的事务状态对象
  else {
    // Create "empty" transaction: no actual transaction, but potentially synchronization.
    if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
      logger.warn("Custom isolation level specified but no actual transaction initiated; " +
          "isolation level will effectively be ignored: " + definition);
    }
    boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
    return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
  }
}
```

获取事务的方法主要做两件事情：

1. 获取事务对象
2. 根据事务传播方式返回事务状态对象

获取事务对象，在DataSourceTransactionManager的实现中，返回一个DataSourceTransactionObject对象

```java
protected Object doGetTransaction() {
  DataSourceTransactionObject txObject = new DataSourceTransactionObject();
  txObject.setSavepointAllowed(isNestedTransactionAllowed());
  ConnectionHolder conHolder =
      (ConnectionHolder) 
  // 从事务同步管理器中根据DataSource获取数据库连接资源		
  TransactionSynchronizationManager.getResource(obtainDataSource());
  txObject.setConnectionHolder(conHolder, false);
  return txObject;
}
```

每次执行doGetTransaction方法，即会创建一个DataSourceTransactionObject对象txObject，并从事务同步管理器中根据DataSource获取数据库连接持有对象ConnectionHolder，然后存入txObject中。**事务同步管理类持有一个ThreadLocal级别的resources对象，存储DataSource和ConnectionHolder的映射关系。**因此返回的txObject中持有的ConnectionHolder可能有值，也可能为空。而不同的事务传播方式下，事务管理的处理根据txObejct中是否存在事务有不同的处理方式。

关于关注事务传播方式的实现，很多人对事务传播方式都是一知半解，只是因为没有了解源码的实现。现在就来看看具体的实现。事务传播方式的实现分为两种情况，事务不存在和事务已经存在。isExistingTransaction方法判断事务是否存在，默认在AbstractPlatformTransactionManager抽象类中返回false，而在DataSourceTransactionManager实现中，则根据是否有数据库连接来决定。

```java
protected boolean isExistingTransaction(Object transaction) {
  DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
  return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
}
```

### 当前无事务

如果当前没有事务，则不同事务传播方式的处理如下：

1. 事务传播方式为mandatory(强制必须有事务)，当前没有事务，即抛出异常。
2. 事务传播方式为required或required_new或nested(嵌套)，当前没有事务，即会创建一个新的事务状态。
3. 其他事务传播方式时，直接返回事务对象为null的事务状态对象，即不在事务中执行。

如何创建一个新的事务状态

```java
// 1. 新建事务状态,返回DefaultTransactionStatus对象
DefaultTransactionStatus status = newTransactionStatus(
    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
// 2. 事务初始化
doBegin(transaction, definition);
// 3. 准备同步操作
prepareSynchronization(status, definition);
return status;
```

第一步，新建事务状态，就是构建一个DefaultTransactionStatus对象

```java
protected DefaultTransactionStatus newTransactionStatus(
    TransactionDefinition definition, @Nullable Object transaction, boolean newTransaction,
    boolean newSynchronization, boolean debug, @Nullable Object suspendedResources) {

  boolean actualNewSynchronization = newSynchronization &&
      !TransactionSynchronizationManager.isSynchronizationActive();
  return new DefaultTransactionStatus(
      transaction, newTransaction, actualNewSynchronization,
      definition.isReadOnly(), debug, suspendedResources);
}
```

第二步，事务初始化，AbstractPlatformTransactionManager没有实现，来看DataSourceTransactionManager的实现：**获取一个新的数据库连接并开启事务，完成事务的基本属性设置。**

```java
protected void doBegin(Object transaction, TransactionDefinition definition) {
  DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
  Connection con = null;

  try {
  	// 如果当前没有事务，由DataSource获取一个新的数据库连接，并赋予txObject
    if (!txObject.hasConnectionHolder() ||
        txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
      Connection newCon = obtainDataSource().getConnection();
      if (logger.isDebugEnabled()) {
        logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
      }
      txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
    }

	// 设置数据库连接与事务同步
    txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
    con = txObject.getConnectionHolder().getConnection();

	// 设置事务隔离级别
    Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
    txObject.setPreviousIsolationLevel(previousIsolationLevel);

    // Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
    // so we don't want to do it unnecessarily (for example if we've explicitly
    // configured the connection pool to set it already).
    // 非常重要的一点，JDBC通过设置自动提交为false，开启一个新的事务
    if (con.getAutoCommit()) {
      txObject.setMustRestoreAutoCommit(true);
      if (logger.isDebugEnabled()) {
        logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
      }
      con.setAutoCommit(false);
    }

	// 如果设置事务只读属性，执行Statement设置只读
    prepareTransactionalConnection(con, definition);
    // 激活事务状态
    txObject.getConnectionHolder().setTransactionActive(true);

	// 设置超时时间
    int timeout = determineTimeout(definition);
    if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
      txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
    }

    // Bind the connection holder to the thread.
    // 如果为新连接，绑定DataSource和数据库连接持有者的映射关系
    if (txObject.isNewConnectionHolder()) {
      TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
    }
  }

  catch (Throwable ex) {
    if (txObject.isNewConnectionHolder()) {
      DataSourceUtils.releaseConnection(con, obtainDataSource());
      txObject.setConnectionHolder(null, false);
    }
    throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
  }
}
```

第三步：准备同步操作，如果事务状态开启同步，则在事务同步管理器中设置事务基础属性

```java
protected void prepareSynchronization(DefaultTransactionStatus status, TransactionDefinition definition) {
  if (status.isNewSynchronization()) {
    TransactionSynchronizationManager.setActualTransactionActive(status.hasTransaction());
    TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(
        definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT ?
            definition.getIsolationLevel() : null);
    TransactionSynchronizationManager.setCurrentTransactionReadOnly(definition.isReadOnly());
    TransactionSynchronizationManager.setCurrentTransactionName(definition.getName());
    TransactionSynchronizationManager.initSynchronization();
  }
}
```

### 当前有事务

如果当前已经有事务存在，由handleExistingTransaction方法完成事务操作。

1. 传播方式为never（不允许事务），抛出异常

2. 传播方式为not_supported（不支持），则挂起当前事务，以无事务方式运行

3. 传播方式为required_new，挂起原有事务，并开启新的事务

4. 传播方式为nested（嵌套）,创建嵌套事务。这里一般方式都是通过savepoint保存点的完成嵌套，但Spring对JTA事务单独做了另一种处理。

5. 传播方式为supports或required，返回当前事务

6. ```java
   private TransactionStatus handleExistingTransaction(
       TransactionDefinition definition, Object transaction, boolean debugEnabled)
       throws TransactionException {
   
     // 传播方式为never（不允许事务），抛出异常
     if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
       throw new IllegalTransactionStateException(
           "Existing transaction found for transaction marked with propagation 'never'");
     }
   
     // 传播方式为not_supported（不支持），则挂起当前事务，以无事务方式运行
     if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
       if (debugEnabled) {
         logger.debug("Suspending current transaction");
       }
       Object suspendedResources = suspend(transaction);
       boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
       return prepareTransactionStatus(
           definition, null, false, newSynchronization, debugEnabled, suspendedResources);
     }
   
     // 传播方式为required_new，挂起原有事务，并开启新的事务
     if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
       if (debugEnabled) {
         logger.debug("Suspending current transaction, creating new transaction with name [" +
             definition.getName() + "]");
       }
       SuspendedResourcesHolder suspendedResources = suspend(transaction);
       try {
         boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
         DefaultTransactionStatus status = newTransactionStatus(
             definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
         doBegin(transaction, definition);
         prepareSynchronization(status, definition);
         return status;
       }
       catch (RuntimeException | Error beginEx) {
         resumeAfterBeginException(transaction, suspendedResources, beginEx);
         throw beginEx;
       }
     }
   
     // 传播方式为nested（嵌套）,创建嵌套事务
     if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
       if (!isNestedTransactionAllowed()) {
         throw new NestedTransactionNotSupportedException(
             "Transaction manager does not allow nested transactions by default - " +
             "specify 'nestedTransactionAllowed' property with value 'true'");
       }
       if (debugEnabled) {
         logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
       }
       // 使用保存点支持嵌套事务
       if (useSavepointForNestedTransaction()) {
         // Create savepoint within existing Spring-managed transaction,
         // through the SavepointManager API implemented by TransactionStatus.
         // Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
         DefaultTransactionStatus status =
             prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
         status.createAndHoldSavepoint();
         return status;
       }
       // 只适用于JTA事务：通过嵌套的begin和commit/rollback创建嵌套事务
       else {
         // Nested transaction through nested begin and commit/rollback calls.
         // Usually only for JTA: Spring synchronization might get activated here
         // in case of a pre-existing JTA transaction.
         boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
         DefaultTransactionStatus status = newTransactionStatus(
             definition, transaction, true, newSynchronization, debugEnabled, null);
         doBegin(transaction, definition);
         prepareSynchronization(status, definition);
         return status;
       }
     }
   
     // Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
     // 传播方式为supports或required，返回当前事务
     if (debugEnabled) {
       logger.debug("Participating in existing transaction");
     }
     if (isValidateExistingTransaction()) {
       if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
         Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
         if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
           Constants isoConstants = DefaultTransactionDefinition.constants;
           throw new IllegalTransactionStateException("Participating transaction with definition [" +
               definition + "] specifies isolation level which is incompatible with existing transaction: " +
               (currentIsolationLevel != null ?
                   isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
                   "(unknown)"));
         }
       }
       if (!definition.isReadOnly()) {
         if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
           throw new IllegalTransactionStateException("Participating transaction with definition [" +
               definition + "] is not marked as read-only but existing transaction is");
         }
       }
     }
     boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
     return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
   }
   ```

   这里关注两个点

   第一是事务的挂起，Spring并不是真的对数据库连接做了什么挂起操作，而是在逻辑上由事务同步管理器做了事务信息和状态的重置，并将原事务信息和状态返回，并记录在新的事务状态对象中，从而形成一种链式结构。

   ```java
   protected final SuspendedResourcesHolder suspend(@Nullable Object transaction) throws TransactionException {
     if (TransactionSynchronizationManager.isSynchronizationActive()) {
       List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();
       try {
         Object suspendedResources = null;
         if (transaction != null) {
           suspendedResources = doSuspend(transaction);
         }
         String name = TransactionSynchronizationManager.getCurrentTransactionName();
         TransactionSynchronizationManager.setCurrentTransactionName(null);
         boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
         TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
         Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
         TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
         boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
         TransactionSynchronizationManager.setActualTransactionActive(false);
         return new SuspendedResourcesHolder(
             suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
       }
       catch (RuntimeException | Error ex) {
         // doSuspend failed - original transaction is still active...
         doResumeSynchronization(suspendedSynchronizations);
         throw ex;
       }
     }
     else if (transaction != null) {
       // Transaction active but no synchronization active.
       Object suspendedResources = doSuspend(transaction);
       return new SuspendedResourcesHolder(suspendedResources);
     }
     else {
       // Neither transaction nor synchronization active.
       return null;
     }
   }
   ```

   第二是嵌套事务设置保存点，通常由JDBC3.0支持的savepoint API完成，然后将保存点记录在事务状态中。

   ```java
   DefaultTransactionStatus status =
       prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
   status.createAndHoldSavepoint();
   return status;
   ```

   至此，完成了获取事务。

   

   ## 2.提交事务

   首先要说的，**commit方法并不是一定提交事务，也可能回滚。**

   ```java
   public final void commit(TransactionStatus status) throws TransactionException {
     // 事务已经提交，再次提交抛出异常
     if (status.isCompleted()) {
       throw new IllegalTransactionStateException(
           "Transaction is already completed - do not call commit or rollback more than once per transaction");
     }
   
     DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
     // 如果事务状态设置了回滚标识，则执行回滚
     if (defStatus.isLocalRollbackOnly()) {
       if (defStatus.isDebug()) {
         logger.debug("Transactional code has requested rollback");
       }
       processRollback(defStatus, false);
       return;
     }
   
     // 设置全局回滚标识为true，则执行回滚
     if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
       if (defStatus.isDebug()) {
         logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
       }
       processRollback(defStatus, true);
       return;
     }
   
     // 提交事务
     processCommit(defStatus);
   }
   ```

   processCommit执行事务的提交，但事务的提交也分为几种情况：

   1. 存在保存点，即嵌套事务，则释放保存点
   2. 如果事务是由当前事务状态开启的，即事务传播的第一层，执行事务提交
   3. 其他情况（比如事务是继承自上一层），则不做任何操作

   且在processCommit方法中，不同时候设置了不同状态的触发监控，用来提示事务同步相关资源，触发需要的操作。

   ```java
   private void processCommit(DefaultTransactionStatus status) throws TransactionException {
     try {
       boolean beforeCompletionInvoked = false;
   
       try {
         boolean unexpectedRollback = false;
         prepareForCommit(status);
         // 提交前提示
         triggerBeforeCommit(status);
         // 完成前提示
         triggerBeforeCompletion(status);
         beforeCompletionInvoked = true;
   
         if (status.hasSavepoint()) {
           if (status.isDebug()) {
             logger.debug("Releasing transaction savepoint");
           }
           unexpectedRollback = status.isGlobalRollbackOnly();
           status.releaseHeldSavepoint();
         }
         else if (status.isNewTransaction()) {
           if (status.isDebug()) {
             logger.debug("Initiating transaction commit");
           }
           unexpectedRollback = status.isGlobalRollbackOnly();
           // 执行事务提交
           doCommit(status);
         }
         else if (isFailEarlyOnGlobalRollbackOnly()) {
           unexpectedRollback = status.isGlobalRollbackOnly();
         }
   
         // Throw UnexpectedRollbackException if we have a global rollback-only
         // marker but still didn't get a corresponding exception from commit.
         if (unexpectedRollback) {
           throw new UnexpectedRollbackException(
               "Transaction silently rolled back because it has been marked as rollback-only");
         }
       }
       catch (UnexpectedRollbackException ex) {
         // can only be caused by doCommit
         // 回滚完成提示
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
         throw ex;
       }
       catch (TransactionException ex) {
         // can only be caused by doCommit
         if (isRollbackOnCommitFailure()) {
           doRollbackOnCommitException(status, ex);
         }
         else {
           // 未知状态完成提示
           triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
         }
         throw ex;
       }
       catch (RuntimeException | Error ex) {
         if (!beforeCompletionInvoked) {
           triggerBeforeCompletion(status);
         }
         doRollbackOnCommitException(status, ex);
         throw ex;
       }
   
       // Trigger afterCommit callbacks, with an exception thrown there
       // propagated to callers but the transaction still considered as committed.
       try {
         // 事务提交完成提示
         triggerAfterCommit(status);
       }
       finally {
         // 操作完成完成提示
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
       }
   
     }
     finally {
       // 完成后清理
       cleanupAfterCompletion(status);
     }
   }
   ```

   DataSourceTransactionManager对doCommit的实现，就是执行数据库连接的提交

   ```java
   protected void doCommit(DefaultTransactionStatus status) {
     DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
     Connection con = txObject.getConnectionHolder().getConnection();
     if (status.isDebug()) {
       logger.debug("Committing JDBC transaction on Connection [" + con + "]");
     }
     try {
       // 提交事务
       con.commit();
     }
     catch (SQLException ex) {
       throw new TransactionSystemException("Could not commit JDBC transaction", ex);
     }
   }
   ```

   ## 3.回滚事务

   回滚事务时也分为几种情况：

   1. 存在保存点（嵌套事务），则回滚到保存点

   2. 如果事务是由当前事务状态开启的，则执行回滚操作

   3. 其他情况下，如果事务状态设置了回滚标识，则设置事务对象的状态也为回滚，否则不做任何操作

      ```java
      private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
         try {
            boolean unexpectedRollback = unexpected;
      
            try {
               triggerBeforeCompletion(status);
      
               if (status.hasSavepoint()) {
                  if (status.isDebug()) {
                     logger.debug("Rolling back transaction to savepoint");
                  }
                  // 回滚保存点
                  status.rollbackToHeldSavepoint();
               }
               else if (status.isNewTransaction()) {
                  if (status.isDebug()) {
                     logger.debug("Initiating transaction rollback");
                  }
                  // 回滚事务
                  doRollback(status);
               }
               else {
                  // Participating in larger transaction
                  if (status.hasTransaction()) {
                     if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                        if (status.isDebug()) {
                           logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
                        }
                        doSetRollbackOnly(status);
                     }
                     else {
                        if (status.isDebug()) {
                           logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
                        }
                     }
                  }
                  else {
                     logger.debug("Should roll back transaction but cannot - no transaction available");
                  }
                  // Unexpected rollback only matters here if we're asked to fail early
                  if (!isFailEarlyOnGlobalRollbackOnly()) {
                     unexpectedRollback = false;
                  }
               }
            }
            catch (RuntimeException | Error ex) {
               triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
               throw ex;
            }
      
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
      
            // Raise UnexpectedRollbackException if we had a global rollback-only marker
            if (unexpectedRollback) {
               throw new UnexpectedRollbackException(
                     "Transaction rolled back because it has been marked as rollback-only");
            }
         }
         finally {
            cleanupAfterCompletion(status);
         }
      }
      ```

      对于DataSourceTransactionManager实现，回滚保存点和回滚事务都由JDBC的API来完成。

      至此，事务管理器对事务的三种操作就简单地介绍完了，但其中事务同步资源的控制十分精妙，这里就不做详细的介绍。有兴趣的自己去研究源码。于我而言，任何强大的机制都是由一行行地源码精妙地组建出来的，深入进去，一切都将真相大白。