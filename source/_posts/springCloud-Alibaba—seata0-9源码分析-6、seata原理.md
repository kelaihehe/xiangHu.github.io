---
title: springCloud Alibaba—seata0.9源码分析 6、seata原理总结
date: 2020-07-15 22:14:09
tags:
  - SpringCloud Alibaba
  - seata
summary: seata0.9版本源码解析：seata原理总结
categories: SpringCloud Alibaba
---

## 从全局事务角色的角度 ##
* 全局事务发起者：@GlobalTransactional 注解的方法为全局事务发起者
	* 1、由TM向TC发起Begin全局事务，得到Xid，绑定到RootContext上
	* 2、执行业务
		* sql相关
			* 1、如果RootContext中没有Xid，则直接执行sql，否则开启本地事务
				* 对于insert/update/delete操作：生成beforeImage，执行sql语句，afterImage，创建undolog，保存在内存中，创建lockKey用于申请全局锁
				* 对于select for update操作：先向TC查询查询的全局锁是否被占有，来判断是否返回结果
				* 对于select操作：直接执行并返回查询结果
			* 2、由RM向TC注册分支事务（需要申请全局锁），将undolog持久化，提交本地事务
			* 如果本地事务抛出异常，则report(false)，向TC报告一阶段失败的状态，并抛出异常（一直往上抛，知道第三步）
			* 如果一切顺利，则report(ture)，向TC报告一阶段成功的状态，并context中的Xid等信息
		* 服务调用
			* 在调用远程服务之前，会将RootContext中Xid，加入到request请求头中，进行服务间的传播
			* 服务提供者会在提前将请求头中Xid绑定到自己的RootContext中，进而实现了全局事务的传播
	* 3、若业务抛出异常，则由TM向TC发起rollback请求 
		* TC接收到请求后，会关闭session，防止其它分支事务注册
		* 向当前所有的RM客户端发送rollback命令  
		* RM接受到rollback命令后  （二阶段回滚）
			* 首先执行undo，从undolog表中查询出对应的log，解析后执行完成反向补偿
			* 最后批量删除undolog表中的相关记录
	* 4、若一切顺利，则由TM向TC发起全局commit请求  
		* TC接收到请求后，会关闭session，防止其它分支事务注册
		* 向当前所有的RM客户端发送commit命令  
		* RM接受到命令后，异步删除undolog表中的相关记录  （二阶段提交）
* 全局事务参与者（调用链中非入口的服务）：
	* 通过SeataHandlerInterceptor拦截器的拦截，会将请求头中Xid绑定到自己的RootContext中，进而实现了全局事务的传播
	* 在RootContext有Xid后，经过PreparedStatementProxy的代理时，会自动开启本地事务，并向TC注册分支事
	* 每个全局事务参与者都会经过，上面全局事务发起者中的sql相关部分。这就是为什么除了全局事务发起者都不需要@GlobalTransactional 注解



## 从事务阶段的角度 ##
* 一阶段：
	* 各分支事务完成本地事务提交以及undolog的持久化
* 二阶段：
	* 调用链最后回到全局事务发起者，再根据整个执行过程中是否抛异常，来决定是全局回滚还是全局提交



## 从TC、TM和RM的角度 ##
* TC存于seata 的Server端，主要工作是：
	* 维护全局session、分支session，及它们的状态。
	* 接收TM和RM的请求，并响应，包括发送分支事务的提交和回滚的命令
* TM存于全局事务发起者端，主要工作是：
	* 向TC发起开启全局事务的请求，全局rollback和全局commit命令
* RM存于每个分支事务端，主要工作是：
	* 向TC注册分支事务，report一阶段状态
	* 接收并处理TC发送的branchRollback请求和branchCommit请求

