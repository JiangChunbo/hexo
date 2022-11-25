---
title: MyBatis Spring 关于 TransactionSynchronizationManager 的使用
date: 2022-11-24 17:15:02
tags:
- MyBatis-Spring
---

# 结论

MyBatis 使用 `TransactionSynchronizationManager` 在事务中保存 `SqlSession`。

# 分析

`org.mybatis.spring.SqlSessionUtils` 提供了一个获取 `SqlSession` 的方法 `getSqlSession()`:

```java{.line-numbers}
public static SqlSession getSqlSession(
    SqlSessionFactory sessionFactory, 
    ExecutorType executorType,
    PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
        return session;
    }

    LOGGER.debug(() -> "Creating a new SqlSession");
    session = sessionFactory.openSession(executorType);

    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
}
```

第`9~11`行通过 `TransactionSynchronizationManager` 尝试（从 `ThreadLocal`）获得一个 `SqlSession`。

首次运行，一定是无法获取到 `SqlSession` 的，因为 `ThreadLocal` 还没有注册 `SqlSession`。

执行到第`17`行才会执行注册 `SqlSession` 的逻辑:

```java
private static void registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType,
        PersistenceExceptionTranslator exceptionTranslator, SqlSession session) {
    SqlSessionHolder holder;
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        Environment environment = sessionFactory.getConfiguration().getEnvironment();

        if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
            LOGGER.debug(() -> "Registering transaction synchronization for SqlSession [" + session + "]");

            holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
            TransactionSynchronizationManager.bindResource(sessionFactory, holder);
            TransactionSynchronizationManager
                    .registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
            holder.setSynchronizedWithTransaction(true);
            holder.requested();
        } else {
            if (TransactionSynchronizationManager.getResource(environment.getDataSource()) == null) {
                LOGGER.debug(() -> "SqlSession [" + session
                        + "] was not registered for synchronization because DataSource is not transactional");
            } else {
                throw new TransientDataAccessResourceException(
                        "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
            }
        }
    } else {
        LOGGER.debug(() -> "SqlSession [" + session
                + "] was not registered for synchronization because synchronization is not active");
    }
}
```

但是，如果 `TransactionSynchronizationManager.isSynchronizationActive()` 返回 false（即，没有激活同步），也不会执行注册逻辑。这个 `isSynchronizationActive` 的判断逻辑如下:


```java
public static boolean isSynchronizationActive() {
    return (synchronizations.get() != null);
}
```

上面的代码表示，如果能够获取到 `synchronizations`，则表示同步被激活。`synchronizations` 的声明如下，它是 `TransactionSynchronizationManager` 内部的用于维护每个线程各自 `Set<TransactionSynchronization>` 的 `ThreadLocal` 对象:

```java
private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
        new NamedThreadLocal<Set<TransactionSynchronization>>("Transaction synchronizations");
```



何时会激活同步也就是何时会调用 `synchronizations` 的 `set()` 方法，答案是开启事务的时候。





以下是一段编程式开启事务的代码示例:

```java
TransactionStatus txStatus =
    transactionManager.getTransaction(new DefaultTransactionDefinition());
```

`getTransaction` 的逻辑如下: 

```java{.line-numbers}
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
    Object transaction = doGetTransaction();

    // Cache debug flag to avoid repeated checks.
    boolean debugEnabled = logger.isDebugEnabled();

    if (definition == null) {
        // Use defaults if no transaction definition given.
        definition = new DefaultTransactionDefinition();
    }

    if (isExistingTransaction(transaction)) {
        // Existing transaction found -> check propagation behavior to find out how to behave.
        return handleExistingTransaction(definition, transaction, debugEnabled);
    }

    // Check definition settings for new transaction.
    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }

    // No existing transaction found -> check propagation behavior to find out how to proceed.
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        SuspendedResourcesHolder suspendedResources = suspend(null);
        if (debugEnabled) {
            logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
        }
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
        catch (RuntimeException ex) {
            resume(null, suspendedResources);
            throw ex;
        }
        catch (Error err) {
            resume(null, suspendedResources);
            throw err;
        }
    }
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

第`39`行执行了`prepareSynchronization`方法，该方法就是操作 `TransactionSynchronizationManager`的，代码如下:

```java{.line-numbers}
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

第`9`行调用了 `TransactionSynchronizationManager.initSynchronization()` 方法，该方法对 `synchronizations` 进行了设置:

```java
public static void initSynchronization() throws IllegalStateException {
    if (isSynchronizationActive()) {
        throw new IllegalStateException("Cannot activate transaction synchronization - already active");
    }
    logger.trace("Initializing transaction synchronization");
    synchronizations.set(new LinkedHashSet<TransactionSynchronization>());
}
```