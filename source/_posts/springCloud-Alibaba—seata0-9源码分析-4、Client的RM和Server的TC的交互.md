---
title: springCloud Alibaba—seata0.9源码分析 4、Client的RM和Server的TC的交互
date: 2020-07-15 21:13:16
tags:
  - SpringCloud Alibaba
  - seata
summary: seata0.9版本源码解析：seata Server和Client之间的通信
categories: SpringCloud Alibaba
---

Server的TC和client的RM的交互：Server的DefaultCore类 和 RM的DefaultRMHandler类

Server端：
* 1、当TM向TC发起begin请求时，DefaultCore#begin处理该请求

``` java
@Override
public String begin(String applicationId, String transactionServiceGroup, String name, int timeout)
    throws TransactionException {
    GlobalSession session = GlobalSession.createGlobalSession(applicationId, transactionServiceGroup, name, timeout);  //创建全局Session
    session.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
    session.begin();
    //transaction start event
    eventBus.post(new GlobalTransactionEvent(session.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
        session.getTransactionName(), session.getBeginTime(), null, session.getStatus()));

    LOGGER.info("Successfully begin global transaction xid = {}", session.getXid());
    return session.getXid();
}
```
* 2、当RM向TC发起分支事务注册请求时，DefaultCore#branchRegister处理该请求

``` java
@Override
public Long branchRegister(BranchType branchType, String resourceId, String clientId, String xid,
                           String applicationData, String lockKeys) throws TransactionException {
    GlobalSession globalSession = assertGlobalSessionNotNull(xid);
    return globalSession.lockAndExcute(() -> {
        //...
        globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
        BranchSession branchSession = SessionHelper.newBranchByGlobal(globalSession, branchType, resourceId,  //创建分支session
            applicationData, lockKeys, clientId);
        if (!branchSession.lock()) {
            throw new BranchTransactionException(LockKeyConflict,
                String.format("Global lock acquire failed xid = %s branchId = %s", globalSession.getXid(), branchSession.getBranchId()));
        }
        try {
            globalSession.addBranch(branchSession);  //添加分支session到全局session
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
- 3、当RM向TC发起report一阶段状态请求时，DefaultCore的branchReport处理该请求（只是将globalSession的状态改成对应的status）

``` java
@Override
public void branchReport(BranchType branchType, String xid, long branchId, BranchStatus status,
                         String applicationData) throws TransactionException {
    GlobalSession globalSession = assertGlobalSessionNotNull(xid);   //获取Xid对应的globalSession
    BranchSession branchSession = globalSession.getBranch(branchId); //获取globalSession的中的branchSession
    if (branchSession == null) {
        throw new BranchTransactionException(BranchTransactionNotExist, String.format("Could not found branch session xid = %s branchId = %s", xid, branchId));
    }
    globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());  //添加
    globalSession.changeBranchStatus(branchSession, status);  //改变branchSession的状态，包括改变持久化的Session的状态

    LOGGER.info("Successfully branch report xid = {}, branchId = {}", globalSession.getXid(), branchSession.getBranchId());
}
```
- 4、当TM向TC发起全局rollback，DefaultCore的rollback处理该请求（遍历branchSession，发送branchrollback命令）

``` java
@Override
public GlobalStatus rollback(String xid) throws TransactionException {
    GlobalSession globalSession = SessionHolder.findGlobalSession(xid);  //获取Xid对应的globalSession
    if (globalSession == null) {
        return GlobalStatus.Finished;
    }
    globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
    // just lock changeStatus
    boolean shouldRollBack = globalSession.lockAndExcute(() -> {
        globalSession.close();          // 关闭全局Session，其它分支就不能注册
        if (globalSession.getStatus() == GlobalStatus.Begin) {  //如果globalSession开启了，则可以回滚
            globalSession.changeStatus(GlobalStatus.Rollbacking);
            return true;
        }
        return false;
    });
    if (!shouldRollBack) {    
        return globalSession.getStatus();
    }


    doGlobalRollback(globalSession, false);    //执行
    return globalSession.getStatus();
}


@Override
public void doGlobalRollback(GlobalSession globalSession, boolean retrying) throws TransactionException {
    //start rollback event
    eventBus.post(new GlobalTransactionEvent(globalSession.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
            globalSession.getTransactionName(), globalSession.getBeginTime(), null, globalSession.getStatus()));

    if(isSaga(globalSession)){   //SAGA类型的分支
        //...
    }
    else{    //其他类型的分支包括AT类型

        for (BranchSession branchSession : globalSession.getReverseSortedBranches()) {  //遍历所有分支Session
            BranchStatus currentBranchStatus = branchSession.getStatus();
            if (currentBranchStatus == BranchStatus.PhaseOne_Failed) {
                globalSession.removeBranch(branchSession);
                continue;
            }
            try {
                BranchStatus branchStatus = resourceManagerInbound.branchRollback(branchSession.getBranchType(),
                        branchSession.getXid(), branchSession.getBranchId(),
                        branchSession.getResourceId(), branchSession.getApplicationData());   //
                //...
            } catch (Exception ex) {
                LOGGER.error("Exception rollbacking branch xid={} branchId={}", globalSession.getXid(), branchSession.getBranchId(), ex);
                if (!retrying) {
                    queueToRetryRollback(globalSession);
                }
                throw new TransactionException(ex);
            }
        }
    }

    SessionHelper.endRollbacked(globalSession);

    //rollbacked event
    eventBus.post(new GlobalTransactionEvent(globalSession.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
            globalSession.getTransactionName(), globalSession.getBeginTime(), System.currentTimeMillis(),
            globalSession.getStatus()));


    LOGGER.info("Successfully rollback global, xid = {}", globalSession.getXid());
}
```
- 5、当TM向TC发起全局commit，DefaultCore的commit处理该请求（遍历branchSession，发送branchcommit命令）

``` java
@Override
public GlobalStatus commit(String xid) throws TransactionException {
    GlobalSession globalSession = SessionHolder.findGlobalSession(xid);
    if (globalSession == null) {
        return GlobalStatus.Finished;
    }
    globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
    // just lock changeStatus
    boolean shouldCommit = globalSession.lockAndExcute(() -> {
        //the lock should release after branch commit
        globalSession
            .closeAndClean(); // Highlight: Firstly, close the session, then no more branch can be registered.
        if (globalSession.getStatus() == GlobalStatus.Begin) {
            globalSession.changeStatus(GlobalStatus.Committing);
            return true;
        }
        return false;
    });
    if (!shouldCommit) {
        return globalSession.getStatus();
    }
    if (globalSession.canBeCommittedAsync()) {
        asyncCommit(globalSession);
        return GlobalStatus.Committed;
    } else {
        doGlobalCommit(globalSession, false);
    }
    return globalSession.getStatus();
}


@Override
public void doGlobalCommit(GlobalSession globalSession, boolean retrying) throws TransactionException {
    //start committing event
    eventBus.post(new GlobalTransactionEvent(globalSession.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
        globalSession.getTransactionName(), globalSession.getBeginTime(), null, globalSession.getStatus()));

    if(isSaga(globalSession)){  //SAGA类型的分支
                //...
    }
    else{  //其他类型的分支包括AT类型
        for (BranchSession branchSession : globalSession.getSortedBranches()) {
            BranchStatus currentStatus = branchSession.getStatus();
            if (currentStatus == BranchStatus.PhaseOne_Failed) {
                globalSession.removeBranch(branchSession);
                continue;
            }
            try {
                BranchStatus branchStatus = resourceManagerInbound.branchCommit(branchSession.getBranchType(),  //向RM发送提交的命令
                        branchSession.getXid(), branchSession.getBranchId(),
                        branchSession.getResourceId(), branchSession.getApplicationData());
                //...
            } catch (Exception ex) {
                LOGGER.error("Exception committing branch {}", branchSession, ex);
                if (!retrying) {
                    queueToRetryCommit(globalSession);
                    throw new TransactionException(ex);
                }
            }
        }
        if (globalSession.hasBranch()) {
            LOGGER.info("Global[{}] committing is NOT done.", globalSession.getXid());
            return;
        }
    }
    SessionHelper.endCommitted(globalSession);

    //committed event
    eventBus.post(new GlobalTransactionEvent(globalSession.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
        globalSession.getTransactionName(), globalSession.getBeginTime(), System.currentTimeMillis(),
        globalSession.getStatus()));

    LOGGER.info("Global[{}] committing is successfully done.", globalSession.getXid());
}
```

Client端：
* DefaultRMHandler

``` java
public BranchCommitResponse handle(BranchCommitRequest request) {     //当TC向RM发来了BranchCommit命令时，调用
    return this.getRMHandler(request.getBranchType()).handle(request);  //根据类型获取RMHandler，AT模式是RMHandlerAT
}

public BranchRollbackResponse handle(BranchRollbackRequest request) {  //当TC向RM发来了BranchRollback命令时，调用
    return this.getRMHandler(request.getBranchType()).handle(request);
}
```
* RMHandlerAT只覆盖了handle(UndoLogDeleteRequest request)方法，大部分在AbstractRMHandler实现了
* 1、执行BranchCommit
	* AbstractRMHandler

``` java
public BranchCommitResponse handle(BranchCommitRequest request) {
    BranchCommitResponse response = new BranchCommitResponse();
    this.exceptionHandleTemplate(new AbstractCallback<BranchCommitRequest, BranchCommitResponse>() {
        public void execute(BranchCommitRequest request, BranchCommitResponse response) throws TransactionException {
            AbstractRMHandler.this.doBranchCommit(request, response);
        }
    }, request, response);
    return response;
}

protected void doBranchCommit(BranchCommitRequest request, BranchCommitResponse response) throws TransactionException {
    String xid = request.getXid();
    long branchId = request.getBranchId();
    String resourceId = request.getResourceId();
    String applicationData = request.getApplicationData();
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Branch committing: " + xid + " " + branchId + " " + resourceId + " " + applicationData);
    }


    BranchStatus status = this.getResourceManager().branchCommit(request.getBranchType(), xid, branchId, resourceId, applicationData);  #由RM执行branchCommit
    response.setXid(xid);
    response.setBranchId(branchId);
    response.setBranchStatus(status);
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Branch commit result: " + status);
    }
}
```

- - DataSourceManager
``` java
public BranchStatus branchCommit(BranchType branchType, String xid, long branchId, String resourceId, String applicationData) throws TransactionException {
    return this.asyncWorker.branchCommit(branchType, xid, branchId, resourceId, applicationData);  //异步执行
}
```
- - AsyncWorker

``` java
public BranchStatus branchCommit(BranchType branchType, String xid, long branchId, String resourceId, String applicationData) throws TransactionException {
    //将branchCommit任务放在阻塞队列中，等待执行
    if (!ASYNC_COMMIT_BUFFER.offer(new AsyncWorker.Phase2Context(branchType, xid, branchId, resourceId, applicationData))) {
        LOGGER.warn("Async commit buffer is FULL. Rejected branch [" + branchId + "/" + xid + "] will be handled by housekeeping later.");
    }
    
    return BranchStatus.PhaseTwo_Committed;
}

public synchronized void init() {
    LOGGER.info("Async Commit Buffer Limit: " + ASYNC_COMMIT_BUFFER_LIMIT);
    timerExecutor = new ScheduledThreadPoolExecutor(1, new NamedThreadFactory("AsyncWorker", 1, true));
    //AsyncWorker初始化了一个ScheduledThreadPoolExecutor，每1s执行一次doBranchCommits
    timerExecutor.scheduleAtFixedRate(new Runnable() {   
        public void run() {
            try {
                AsyncWorker.this.doBranchCommits();
            } catch (Throwable var2) {
                AsyncWorker.LOGGER.info("Failed at async committing ... " + var2.getMessage());
            }


        }
    }, 10L, 1000L, TimeUnit.MILLISECONDS);
}

private void doBranchCommits() {   //从ASYNC_COMMIT_BUFFER中取出Phase2Context来执行
    if (ASYNC_COMMIT_BUFFER.size() != 0) {
       //...

                    while(var9.hasNext()) {
                        AsyncWorker.Phase2Context commitContext = (AsyncWorker.Phase2Context)var9.next();
                        xids.add(commitContext.xid);
                        branchIds.add(commitContext.branchId);
                        int maxSize = xids.size() > branchIds.size() ? xids.size() : branchIds.size();
                        if (maxSize == 1000) {
                            try {
                                UndoLogManagerFactory.getUndoLogManager(dataSourceProxy.getDbType()).batchDeleteUndoLog(xids, branchIds, conn);  //一次行删除100份数据库中的undolog
                            } catch (Exception var25) {
                                LOGGER.warn("Failed to batch delete undo log [" + branchIds + "/" + xids + "]", var25);
                            }
                            xids.clear();
                            branchIds.clear();
                        }
                    }
                    if (!CollectionUtils.isEmpty(xids) && !CollectionUtils.isEmpty(branchIds)) {
                        try {
                            UndoLogManagerFactory.getUndoLogManager(dataSourceProxy.getDbType()).batchDeleteUndoLog(xids, branchIds, conn); 不满100份时，也需要删除
                        } catch (Exception var24) {
                            LOGGER.warn("Failed to batch delete undo log [" + branchIds + "/" + xids + "]", var24);
                        }
                        continue;
                    }
        }
    }
}
```
* 2、执行BranchRollback
	* AbstractRMHandler

``` java
protected void doBranchRollback(BranchRollbackRequest request, BranchRollbackResponse response) throws TransactionException {
    String xid = request.getXid();
    long branchId = request.getBranchId();
    String resourceId = request.getResourceId();
    String applicationData = request.getApplicationData();
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Branch Rollbacking: " + xid + " " + branchId + " " + resourceId);
    }

    BranchStatus status = this.getResourceManager().branchRollback(request.getBranchType(), xid, branchId, resourceId, applicationData);  
    response.setXid(xid);
    response.setBranchId(branchId);
    response.setBranchStatus(status);
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Branch Rollbacked result: " + status);
    }
}
```
- - DataSourceManager

``` java
public BranchStatus branchRollback(BranchType branchType, String xid, long branchId, String resourceId, String applicationData) throws TransactionException {
    DataSourceProxy dataSourceProxy = this.get(resourceId);
    if (dataSourceProxy == null) {
        throw new ShouldNeverHappenException();
    } else {
        try {
            UndoLogManagerFactory.    (dataSourceProxy.getDbType()).undo(dataSourceProxy, xid, branchId);
        } catch (TransactionException var9) {
            if (LOGGER.isInfoEnabled()) {
                LOGGER.info("branchRollback failed reason [{}]", var9.getMessage());
            }

            if (var9.getCode() == TransactionExceptionCode.BranchRollbackFailed_Unretriable) {
                return BranchStatus.PhaseTwo_RollbackFailed_Unretryable;
            }

            return BranchStatus.PhaseTwo_RollbackFailed_Retryable;
        }

        return BranchStatus.PhaseTwo_Rollbacked;
    }
}
```
- - AbstractUndoLogManager

``` java
public void undo(DataSourceProxy dataSourceProxy, String xid, long branchId) throws TransactionException {
    
    while(true) {
        try {
            selectPST = conn.prepareStatement(SELECT_UNDO_LOG_SQL);  //根据xid和branchId去数据库中查询undolog表
            selectPST.setLong(1, branchId);
            selectPST.setString(2, xid);
            rs = selectPST.executeQuery();
            boolean exists = false;

            while(rs.next()) {
                exists = true;
                //...
                BranchUndoLog branchUndoLog = parser.decode(rollbackInfo);  //解析成BranchUndoLog
                try {
                    setCurrentSerializer(parser.getName());
                    List<SQLUndoLog> sqlUndoLogs = branchUndoLog.getSqlUndoLogs();
                    if (sqlUndoLogs.size() > 1) {
                        Collections.reverse(sqlUndoLogs);
                    }
                    Iterator var19 = sqlUndoLogs.iterator();
                    while(var19.hasNext()) {
                        SQLUndoLog sqlUndoLog = (SQLUndoLog)var19.next();
                        TableMeta tableMeta = TableMetaCacheFactory.getTableMetaCache(dataSourceProxy).getTableMeta(dataSourceProxy, sqlUndoLog.getTableName());
                        sqlUndoLog.setTableMeta(tableMeta);
                        AbstractUndoExecutor undoExecutor = UndoExecutorFactory.getUndoExecutor(dataSourceProxy.getDbType(), sqlUndoLog);
                        undoExecutor.executeOn(conn);  //执行undolog进行反向补偿
                    }
                } finally {
                    removeCurrentSerializer();
                }
            }

            if (exists) {
                this.deleteUndoLog(xid, branchId, conn);  //如果undolog存在，则删除
                conn.commit();
                if (LOGGER.isInfoEnabled()) {
                    LOGGER.info("xid {} branch {}, undo_log deleted with {}", new Object[]{xid, branchId, AbstractUndoLogManager.State.GlobalFinished.name()});
                    break;
                }
            } else {
                this.insertUndoLogWithGlobalFinished(xid, branchId, UndoLogParserFactory.getInstance(), conn);  //若undolog不存在，则插入一条已完成事务状态的undolog
                conn.commit();
            //...
        }
    }
}
```







