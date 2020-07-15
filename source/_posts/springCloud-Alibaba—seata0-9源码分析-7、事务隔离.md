---
title: springCloud Alibaba—seata0.9源码分析 7、事务隔离
date: 2020-07-15 22:20:26
tags:
  - SpringCloud Alibaba
  - seata
summary: seata0.9版本源码解析：seata的事务隔离实现
categories: SpringCloud Alibaba
---
seata的全局事务的默认隔离级别是：读未提交。对于想要实现读已提交，使用select ... for update进行读操作即可
* 写隔离的实现
	* 每个分支事务的执行流程：
		* 全局事务正常：申请本地锁-->执行sql-->申请全局锁-->本地事务提交，释放本地锁-->二阶段提交，释放全局锁
		* 全局事务异常（下游出现异常）：申请本地锁-->执行sql-->申请全局锁-->本地事务提交，释放本地锁-->二阶段回滚，申请本地锁，反向补偿，执行sql语句，释放本地锁-->释放全局锁
	* 分支事务在尝试获取全局锁如果超时，自动放弃，并回滚本地事务，释放本地锁。
		* 为什么如果获取全局锁超时将自动放弃？
			* T1和T2都需要对同一表的的相同行进行更改操作。
			* 当T1先申请到全局锁，完成本地事务提交，释放本地锁。当下游出现异常，需要反向补偿，重新申请本地锁。此时T2已经申请本地锁，但全局锁被T1占有。
			* 此时的状态为：T1需要本地锁，占了全局锁；T2需要全局锁，占了本地锁。出现了死锁，必须要有人先放弃持有的资源，当然是T2放弃申请全局锁
	* 总结：
		* 申请本地锁-->申请全局锁-->释放本地锁-->释放全局锁 的顺序   + 申请全局锁的超时机制     最终实现了写隔离
* 读隔离的实现

	* SelectForUpdateExecutor 对select ... for update语句进行代理，在返回查询结果前，判断锁是否被占有，决定是否返回结果

## 写隔离的实现 ##
* 1、申请全局锁
	* 在执行完AfterImage后，执行prepareUndoLog，其中包括创建lockKeys
		* AbstractDMLBaseExecutor

``` java
protected void prepareUndoLog(TableRecords beforeImage, TableRecords afterImage) throws SQLException {
    if (beforeImage.getRows().size() != 0 || afterImage.getRows().size() != 0) {
        ConnectionProxy connectionProxy = this.statementProxy.getConnectionProxy();
        TableRecords lockKeyRecords = this.sqlRecognizer.getSQLType() == SQLType.DELETE ? beforeImage : afterImage;
        String lockKeys = this.buildLockKey(lockKeyRecords);  //创建 lockKeys
        connectionProxy.appendLockKey(lockKeys);    //将lockKeys添加到connectionProxy的context中
        SQLUndoLog sqlUndoLog = this.buildUndoItem(beforeImage, afterImage);
        connectionProxy.appendUndoLog(sqlUndoLog);
    }
}
```

- - 在注册分支事务时，会向TC发送lockKeys
	- ConnectionProxy

``` java
private void register() throws TransactionException {
    Long branchId = DefaultResourceManager.get().branchRegister(BranchType.AT, this.getDataSourceProxy().getResourceId(), (String)null, this.context.getXid(), (String)null, this.context.buildLockKeys()); //从context中取出lockkey，发送给TC
    this.context.setBranchId(branchId);
}
```
-  TC收到后，会查看该keys是否已经占有
	* DefaultCore

``` java
@Override
public Long branchRegister(BranchType branchType, String resourceId, String clientId, String xid,
                           String applicationData, String lockKeys) throws TransactionException {
    GlobalSession globalSession = assertGlobalSessionNotNull(xid);
    return globalSession.lockAndExcute(() -> {
        if (!globalSession.isActive()) {
            throw new GlobalTransactionException(GlobalTransactionNotActive,
                String.format("Could not register branch into global session xid = %s status = %s", globalSession.getXid(), globalSession.getStatus()));
        }
        //SAGA type accept forward(retry) operation, forward operation will register remaining branches
        if (globalSession.getStatus() != GlobalStatus.Begin && !BranchType.SAGA.equals(branchType)) {
            throw new GlobalTransactionException(GlobalTransactionStatusInvalid,
                String.format("Could not register branch into global session xid = %s status = %s while expecting %s", globalSession.getXid(), globalSession.getStatus(), GlobalStatus.Begin));
        }
        globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
        BranchSession branchSession = SessionHelper.newBranchByGlobal(globalSession, branchType, resourceId，applicationData, lockKeys, clientId);  //创建分支Session
        if (!branchSession.lock()) {  //申请全局锁
            throw new BranchTransactionException(LockKeyConflict,
                String.format("Global lock acquire failed xid = %s branchId = %s", globalSession.getXid(), branchSession.getBranchId()));
        }
        try {
            globalSession.addBranch(branchSession);
        } catch (RuntimeException ex) {
            branchSession.unlock();
            throw new BranchTransactionException(FailedToAddBranch,
                String.format("Failed to store branch xid = %s branchId = %s", globalSession.getXid(), branchSession.getBranchId()));
        }
        LOGGER.info("Successfully register branch xid = {}, branchId = {}", globalSession.getXid(), branchSession.getBranchId());
        return branchSession.getBranchId();
    });
}
```
- 	* 2、释放全局锁
		* TC接收到二阶段的全局提交或全局rollback，都会将各branchSession从全局Session中去除：globalSession.removeBranch(branchSession);
			* GlobalSession

``` java
public void removeBranch(BranchSession branchSession) throws TransactionException {
    for (SessionLifecycleListener lifecycleListener : lifecycleListeners) {
        lifecycleListener.onRemoveBranch(this, branchSession);
    }
    branchSession.unlock();  //释放锁
    remove(branchSession);
}
```
* 总结：
	* 注册分支事务时，申请全局锁
		* RM在执行完sql语句后，会生成sql对应表和行组成LockKey，保存在（形式为：tableName1：primaryKey value1,primaryKey value2... ）。
		* 在注册分支事务时，RM将context中的LockKey用;组成字符串，发送给TC，TC判断该锁是否占用（缓存中是否包含这些表的主键值），从而判断是否为其创建分支事务
	* 在二阶段的全局提交或全局rollback，释放全局锁

## 读隔离的实现 ##
* 对于select for update的查询语句，用SelectForUpdateExecutor执行

``` java
public static <T, S extends Statement> T execute(SQLRecognizer sqlRecognizer, StatementProxy<S> statementProxy, StatementCallback<T, S> statementCallback, Object... args) throws SQLException {
    if (!RootContext.inGlobalTransaction() && !RootContext.requireGlobalLock()) {
        return statementCallback.execute(statementProxy.getTargetStatement(), args);
    } else {
        if (sqlRecognizer == null) {
            sqlRecognizer = SQLVisitorFactory.get(statementProxy.getTargetSQL(), statementProxy.getConnectionProxy().getDbType());
        }
        Executor<T> executor = null;
        if (sqlRecognizer == null) {
            executor = new PlainExecutor(statementProxy, statementCallback);
        } else {
            switch(sqlRecognizer.getSQLType()) {
            case INSERT:
                executor = new InsertExecutor(statementProxy, statementCallback, sqlRecognizer);
                break;
            case UPDATE:
                executor = new UpdateExecutor(statementProxy, statementCallback, sqlRecognizer);
                break;
            case DELETE:
                executor = new DeleteExecutor(statementProxy, statementCallback, sqlRecognizer);
                break;
            case SELECT_FOR_UPDATE:
                executor = new SelectForUpdateExecutor(statementProxy, statementCallback, sqlRecognizer);
                break;
            default:
                executor = new PlainExecutor(statementProxy, statementCallback);
            }
        }

        Object rs = null;
        try {
            rs = ((Executor)executor).execute(args);
            return rs;
        } catch (Throwable var7) {
            Throwable ex = var7;
            if (!(var7 instanceof SQLException)) {
                ex = new SQLException(var7);
            }
            throw (SQLException)ex;
        }
    }
}
```
- SelectForUpdateExecutor 

``` java
public T doExecute(Object... args) throws Throwable {
    //...
    try {
        if (originalAutoCommit) {
            conn.setAutoCommit(false);
        }
        sp = conn.setSavepoint();
        while(true) {
            try { 
                rs = this.statementCallback.execute(this.statementProxy.getTargetStatement(), args);   //先执行查询语句
                TableRecords selectPKRows = this.buildTableRecords(this.getTableMeta(), selectPKSQL, paramAppenderList);
                String lockKeys = this.buildLockKey(selectPKRows);
                if (!StringUtils.isNullOrEmpty(lockKeys)) {
                    if (RootContext.inGlobalTransaction()) {
                        this.statementProxy.getConnectionProxy().checkLock(lockKeys);    //由RM向TC发送Lock查询，若锁被占有了，则抛出LockConflictException
                    } else {
                        if (!RootContext.requireGlobalLock()) {
                            throw new RuntimeException("Unknown situation!");
                        }
                        this.statementProxy.getConnectionProxy().appendLockKey(lockKeys);
                    }
                }
                break;
            } catch (LockConflictException var14) {
                conn.rollback(sp);
                lockRetryController.sleep(var14);
            }
        }
    }
    //...
    return rs;  //若锁未冲突，则直接返回查询结果
}
```







