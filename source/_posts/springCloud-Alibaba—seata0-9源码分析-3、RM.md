---
title: springCloud Alibaba—seata0.9源码分析 3、RM
date: 2020-07-15 20:53:36
tags:
  - SpringCloud Alibaba
  - seata
summary: seata0.9版本源码解析：seata的RM资源管理器
categories: SpringCloud Alibaba
---

seata的RM实现了对数据源的代理
* 1、执行Statement
	* PreparedStatementProxy对PreparedStatement进行代理

``` java
@Override
public boolean execute(String sql) throws SQLException {
    this.targetSQL = sql;
    return ExecuteTemplate.execute(this, new StatementCallback<Boolean, T>() {  //交给ExecuteTemplate执行
        @Override
        public Boolean execute(T statement, Object... args) throws SQLException {
            return statement.execute((String) args[0]); // 回调，执行原始的Statement的execute方法 
        }
    }, sql);
}
```
- - ExecuteTemplate

``` java
public static <T, S extends Statement> T execute(SQLRecognizer sqlRecognizer,
                                                 StatementProxy<S> statementProxy,
                                                 StatementCallback<T, S> statementCallback,
                                                 Object... args) throws SQLException {
    //RootContext中没有Xid，没有TX_LOCK，直接执行原始方法
    if (!RootContext.inGlobalTransaction() && !RootContext.requireGlobalLock()) {  //不存在全局事务，不存在全局锁
        return statementCallback.execute(statementProxy.getTargetStatement(), args);  // 直接执行原始方法
    }
    //若全局事务，则进行代理
    if (sqlRecognizer == null) {
        // sql解析
        sqlRecognizer = SQLVisitorFactory.get(statementProxy.getTargetSQL(), statementProxy.getConnectionProxy().getDbType());
    }

    Executor<T> executor = null;
    if (sqlRecognizer == null) {
        executor = new PlainExecutor<T, S>(statementProxy, statementCallback);
    } else {
        // 根据解析出来的类型选择Executor
        switch (sqlRecognizer.getSQLType()) {
            case INSERT:
                executor = new InsertExecutor<T, S>(statementProxy, statementCallback, sqlRecognizer);
                break;
            case UPDATE:
                executor = new UpdateExecutor<T, S>(statementProxy, statementCallback, sqlRecognizer);
                break;
            case DELETE:
                executor = new DeleteExecutor<T, S>(statementProxy, statementCallback, sqlRecognizer);
                break;
            case SELECT_FOR_UPDATE:
                executor = new SelectForUpdateExecutor<T, S>(statementProxy, statementCallback, sqlRecognizer);
                break;
            default:
                executor = new PlainExecutor<T, S>(statementProxy, statementCallback);
                break;
        }
    }
    T rs = null;
    try {
        rs = executor.execute(args);  // executor执行
    } catch (Throwable ex) {
        if (!(ex instanceof SQLException)) {
            ex = new SQLException(ex);
        }
        throw (SQLException)ex;
    }
    return rs;
}
```

- - BaseTransactionalExecutor

``` java
public Object execute(Object... args) throws Throwable {
    if (RootContext.inGlobalTransaction()) {  //若是全局事务，先获取从TM那里获取Xid，绑定到ConnectionProxy上
        String xid = RootContext.getXID();
        this.statementProxy.getConnectionProxy().bind(xid);
    }

    if (RootContext.requireGlobalLock()) {
        this.statementProxy.getConnectionProxy().setGlobalLockRequire(true);
    } else {
        this.statementProxy.getConnectionProxy().setGlobalLockRequire(false);
    }

    return this.doExecute(args);  
}
```
- - （对于insert/update/delete的操作）AbstractDMLBaseExecutor

``` java
public T doExecute(Object... args) throws Throwable {
    AbstractConnectionProxy connectionProxy = statementProxy.getConnectionProxy();
    
    if (connectionProxy.getAutoCommit()) {    //判断自动提交标志位
        return executeAutoCommitTrue(args);   //默认是自动提交
    } else {
        return executeAutoCommitFalse(args);   
    }
}

protected T executeAutoCommitTrue(Object[] args) throws Throwable {
    ConnectionProxy connectionProxy = this.statementProxy.getConnectionProxy();
    Object var3;
    try {
        connectionProxy.setAutoCommit(false);  //这里开启本地事务
        var3 = (new AbstractDMLBaseExecutor.LockRetryPolicy(connectionProxy.getTargetConnection())).execute(() -> {
            T result = this.executeAutoCommitFalse(args);  //再执行executeAutoCommitFalse
            connectionProxy.commit();  //最后提交事务
            return result;
        });
    } catch (Exception var7) {
        if (!AbstractDMLBaseExecutor.LockRetryPolicy.isLockRetryPolicyBranchRollbackOnConflict()) {  
            connectionProxy.getTargetConnection().rollback();   //默认不执行
        }
        throw var7;
    } finally {
        ((ConnectionProxy)connectionProxy).getContext().reset();
        connectionProxy.setAutoCommit(true);
    }
    return var3;
}

protected T executeAutoCommitFalse(Object[] args) throws Exception {
    
    TableRecords beforeImage = beforeImage();  // 生成beforeImage

    T result = statementCallback.execute(statementProxy.getTargetStatement(), args);  //执行本地sql语句
    
    TableRecords afterImage = afterImage(beforeImage);  // 生成afterImage
    prepareUndoLog(beforeImage, afterImage);  // 生成undoLog，保存在connectionProxy；创建lockKey
    return result;
}
```
- - （对于select for update的操作）SelectForUpdateExecutor

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

* 总结：
	* 对于RootContext中没有Xid，则直接执行SQL语句，即不开启本地事务和全局事务
	* 对于RootContext中有Xid，即表示出于全局事务中，则对其代理
		* 1、先绑定Xid到ConnectionProxy
		* 2、开启本地事务
		* 3、执行sql操作
			* 对于insert/update/delete操作：生成beforeImage，执行sql语句，afterImage，创建undolog，保存在connectionProxy，创建lockKey
			* 对于select for update操作：先向TC查询全局锁是否被占有，来判断是否返回结果
			* 对于select操作：直接执行并返回查询结果
		* 4、commit

- 2、执行commit——connectionProxy

``` java
public void commit() throws SQLException {   //提交事务
    try {
        LOCK_RETRY_POLICY.execute(() -> {
            this.doCommit();  //执行提交
            return null;
        });
    } catch (SQLException var2) {
        throw var2;
    } catch (Exception var3) {
        throw new SQLException(var3);
    }
}

private void doCommit() throws SQLException {
    if (this.context.inGlobalTransaction()) {
        this.processGlobalTransactionCommit();   //如果是全局事务，则执行全局事务提交
    } else if (this.context.isGlobalLockRequire()) {
        this.processLocalCommitWithGlobalLocks();
    } else {
        this.targetConnection.commit();
    }
}

private void processGlobalTransactionCommit() throws SQLException {
    try {
        this.register();  //注册分支事务
    } catch (TransactionException var2) {
        this.recognizeLockKeyConflictException(var2, this.context.buildLockKeys());
    }

    try {
        if (this.context.hasUndoLog()) {  //是否有undoLog
            UndoLogManagerFactory.getUndoLogManager(this.getDbType()).flushUndoLogs(this);   //将undoLog持久化
        }
        this.targetConnection.commit();  //本地事务提交
    } catch (Throwable var3) {
        LOGGER.error("process connectionProxy commit error: {}", var3.getMessage(), var3);
        this.report(false);  //如果事务出现错误，则向TC发送一阶段失败
        throw new SQLException(var3);
    }


    this.report(true);  //若本地事务未出错，则向TC发送一阶段完成
    this.context.reset();  // 清除当前ConnectionProxy中的xid、branchId、undoLog buffer之类的，本次提交结束
}


private void register() throws TransactionException {   //由RM向TC注册分支事务，对应Xid，获取分支事务ID
    Long branchId = DefaultResourceManager.get().branchRegister(BranchType.AT, this.getDataSourceProxy().getResourceId(), (String)null, this.context.getXid(), (String)null, this.context.buildLockKeys());
    this.context.setBranchId(branchId);
}
```

* commit总结：
	* 1、让RM拿Xid去TC注册分支事务（获取全局锁）
	* 2、将undolog持久化 
	* 3、提交本地事务  
	* 若一切顺利，则向TC发送一阶段完成；否则发送一阶段失败，并向上抛



