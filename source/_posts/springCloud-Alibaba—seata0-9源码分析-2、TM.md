---
title: springCloud Alibaba—seata0.9源码分析 2、TM
date: 2020-07-15 20:21:41
tags:
  - SpringCloud Alibaba
  - seata
summary: seata0.9版本源码解析：TM事务管理器
categories: SpringCloud Alibaba
---

当方法被@GlobalTransactional注解后，会被aop代理，会被GlobalTransactionalInterceptor拦截

``` java
public Object invoke(MethodInvocation methodInvocation) throws Throwable {
    Class<?> targetClass = methodInvocation.getThis() != null ? AopUtils.getTargetClass(methodInvocation.getThis()) : null;
    Method specificMethod = ClassUtils.getMostSpecificMethod(methodInvocation.getMethod(), targetClass);
    Method method = BridgeMethodResolver.findBridgedMethod(specificMethod);
    GlobalTransactional globalTransactionalAnnotation = (GlobalTransactional)this.getAnnotation(method, GlobalTransactional.class); //获取方法上的GlobalTransactional注解
    GlobalLock globalLockAnnotation = (GlobalLock)this.getAnnotation(method, GlobalLock.class);
    if (globalTransactionalAnnotation != null) {
        return this.handleGlobalTransaction(methodInvocation, globalTransactionalAnnotation);  //若有@GlobalTransactional注解，则进入handleGlobalTransaction
    } else {
        return globalLockAnnotation != null ? this.handleGlobalLock(methodInvocation) : methodInvocation.proceed();
    } 
}

private Object handleGlobalTransaction(final MethodInvocation methodInvocation, final GlobalTransactional globalTrxAnno) throws Throwable {
    try {
        return this.transactionalTemplate.execute(new TransactionalExecutor() {  //由transactionalTemplate执行execute方法
            public Object execute() throws Throwable {
                return methodInvocation.proceed();
            }

            public String name() {  
                String name = globalTrxAnno.name();
                return !StringUtils.isNullOrEmpty(name) ? name : GlobalTransactionalInterceptor.this.formatMethod(methodInvocation.getMethod());
            }

            public TransactionInfo getTransactionInfo() {  //将@GlobalTransactional注解解析成TransactionInfo
                TransactionInfo transactionInfo = new TransactionInfo();
                //解析GlobalTransactional注解上的信息
                return transactionInfo;
            }
        });
    } catch (ExecutionException var5) {
        //处理错误...
    }
}
```
- TransactionalTemplate#execute

``` java
public Object execute(TransactionalExecutor business) throws Throwable {
    GlobalTransaction tx = GlobalTransactionContext.getCurrentOrCreate();  //1、获取GlobalTransaction对象
    TransactionInfo txInfo = business.getTransactionInfo();
    if (txInfo == null) {
        throw new ShouldNeverHappenException("transactionInfo does not exist");
    } else {
        Object var5;
        try {
            this.beginTransaction(txInfo, tx);  //2、开始全局事务
            Object rs = null;

            try {
                rs = business.execute();   //3、执行被代理的原方法，即处理本地事务
            } catch (Throwable var9) {
                this.completeTransactionAfterThrowing(txInfo, tx, var9);  //如果本地事务出现错误，则进行rollback
                throw var9;
            }

            this.commitTransaction(tx);  //4、提交全局事务
            var5 = rs;
        } finally {
            this.triggerAfterCompletion();
            this.cleanUp();
        }

        return var5;
    }
}
```
- - 1、GlobalTransactionContext  获取GlobalTransaction对象

```java
public static GlobalTransaction getCurrentOrCreate() {
    GlobalTransaction tx = getCurrent(); //初始时为空
    return tx == null ? createNew() : tx; //当为空时，创建一个GlobalTransaction，其xid为null，状态为UnKnown，角色为发起者
}

private static GlobalTransaction getCurrent() {
    String xid = RootContext.getXID(); //从ThreadLocal中取出XID，初始时为空
    return xid == null ? null : new DefaultGlobalTransaction(xid, GlobalStatus.Begin, GlobalTransactionRole.Participant); //初始化时为空，所以返回null；若不为空，表示在此之前已经有一个xid，即创建DefaultGlobalTransaction实例，设置状态为Begin，角色为参与者
}

private static GlobalTransaction createNew() { //
    GlobalTransaction tx = new DefaultGlobalTransaction(); //最后调用构造方法this((String)null, GlobalStatus.UnKnown, GlobalTransactionRole.Launcher);
    return tx;
}
```
- - RootContext

``` java
public static String getXID() {
    return CONTEXT_HOLDER.get("TX_XID");  //CONTEXT_HOLDER是一个维护了ThreadLocal<Map<String, String>>类型的实例，初始时map为空
}
```

- - 2、TransactionalTemplate#beginTransaction  开启全局事务

``` java
private void beginTransaction(TransactionInfo txInfo, GlobalTransaction tx) throws ExecutionException {
    try {
        this.triggerBeforeBegin();
        tx.begin(txInfo.getTimeOut(), txInfo.getName());  //
        this.triggerAfterBegin();
    } catch (TransactionException var4) {
        throw new ExecutionException(tx, var4, Code.BeginFailure);
    }
}
```

	- 	- GlobalTransaction

``` java
public void begin(int timeout, String name) throws TransactionException {
    if (this.role != GlobalTransactionRole.Launcher) {  //begin应该有发起者执行
        this.check();
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Ignore Begin(): just involved in global transaction [" + this.xid + "]");
        }

    } else if (this.xid != null) {  //发起者在还未执行begin时，xid为null
        throw new IllegalStateException();
    } else if (RootContext.getXID() != null) {
        throw new IllegalStateException();
    } else {
        this.xid = this.transactionManager.begin((String)null, (String)null, name, timeout);  //向TC发起begin请求，获取Xid
        this.status = GlobalStatus.Begin;  //改变状态为Begin
        RootContext.bind(this.xid);   //将xid保存在threadLocal变量中
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Begin new global transaction [" + this.xid + "]");
        }
    }
}
```
- - DefaultTransactionManager

``` java
public String begin(String applicationId, String transactionServiceGroup, String name, int timeout) throws TransactionException {
    GlobalBeginRequest request = new GlobalBeginRequest();
    request.setTransactionName(name);
    request.setTimeout(timeout);
    GlobalBeginResponse response = (GlobalBeginResponse)this.syncCall(request);  //向TC发起begin请求
    if (response.getResultCode() == ResultCode.Failed) {
        throw new TmTransactionException(TransactionExceptionCode.BeginFailed, response.getMsg());
    } else {
        return response.getXid();  //获取TC颁发的Xid
    }
}
```
- * 3、business.execute();   处理业务
-	* 4、当业务中抛出异常，则执行completeTransactionAfterThrowing，由TM向TC发起回滚请求

``` java
private void completeTransactionAfterThrowing(TransactionInfo txInfo, GlobalTransaction tx, Throwable ex) throws ExecutionException {
    if (txInfo != null && txInfo.rollbackOn(ex)) {  
        try {
            this.rollbackTransaction(tx, ex);  //执行事务回滚，向TC发送事务回滚请求
        } catch (TransactionException var5) {
            throw new ExecutionException(tx, var5, Code.RollbackFailure, ex);
        }
    } else {
        this.commitTransaction(tx);
    }
}
```
- 	- 5、TransactionalTemplate#commitTransaction   由TM向RM发起全局提交事务

``` java
private void commitTransaction(GlobalTransaction tx) throws ExecutionException {
    try {
        this.triggerBeforeCommit();
        tx.commit();
        this.triggerAfterCommit();
    } catch (TransactionException var3) {
        throw new ExecutionException(tx, var3, Code.CommitFailure);
    }
}
```
- - DefaultGlobalTransaction

``` java
public void commit() throws TransactionException {
    if (this.role == GlobalTransactionRole.Participant) {
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Ignore Commit(): just involved in global transaction [" + this.xid + "]");
        }
    } else if (this.xid == null) {
        throw new IllegalStateException();
    } else {
        int retry = COMMIT_RETRY_COUNT;
        try {
            while(retry > 0) {
                try {
                    this.status = this.transactionManager.commit(this.xid); //执行commit
                    break;
                } catch (Throwable var6) {
                    LOGGER.error("Failed to report global commit [{}],Retry Countdown: {}, reason: {}", new Object[]{this.getXid(), retry, var6.getMessage()});
                    --retry;
                    if (retry == 0) {
                        throw new TransactionException("Failed to report global commit", var6);
                    }
                }
            }
        } finally {
            if (RootContext.getXID() != null && this.xid.equals(RootContext.getXID())) {
                RootContext.unbind(); //将threadLocal中的Xid移除
            }
        }

        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("[" + this.xid + "] commit status:" + this.status);
        }
    }
}
```
- - DefaultTransactionManager

``` java
public GlobalStatus commit(String xid) throws TransactionException {
    GlobalCommitRequest globalCommit = new GlobalCommitRequest();
    globalCommit.setXid(xid);
    GlobalCommitResponse response = (GlobalCommitResponse)this.syncCall(globalCommit);  //向TC发起commit请求
    return response.getGlobalStatus();
}
```

* **总结**：
	* GlobalTransactionalInterceptor会对@GlobalTransactional注解的方法进行拦截，主要工作transactionalTemplate的execute方法
	* transactionalTemplate的execute方法的主要逻辑如下：
		* 获取GlobalTransaction对象
			* 从ThreadLocal变量中取，在当前线程还未开启全局事务之前为空。从而会创建一个xid=null，状态为unknow，角色为发起者的GlobalTransaction对象
		* beginTransaction
			* 由GlobalTransaction对象中的TransactionManager来执行begin操作，向TC发起begin请求，返回Xid
		* business.execute()：执行业务
		* completeTransactionAfterThrowing：如果业务出现异常，则执行。
			* 由TM向TC发起全局回滚请求
		* commitTransaction
			* 由GlobalTransaction对象中的TransactionManager来执行commit操作，向TC发起commit请求，返回GlobalStatus





