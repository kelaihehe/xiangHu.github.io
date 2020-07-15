---
title: springCloud Alibaba—seata0.9 简单使用
date: 2020-07-15 17:53:34
tags:
  - SpringCloud Alibaba
  - seata
summary: seata的简单介绍及使用
categories: SpringCloud Alibaba
---

seata服务端和seata客户端都需要注册到服务中心，这里选择nacos作为注册中心。
1、seata服务端
* 下载seata-server，解压
* 1、修改registry.conf

``` 
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"   #选择注册中心方式
  nacos {
    serverAddr = "39.99.209.200"
    namespace = ""
    cluster = "default"
  }

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file" #指定config的方式，这里演示以file.conf的形式读取

  nacos {
    serverAddr = "localhost"
    namespace = ""
  }

  file {
    name = "file.conf"
  }
}
```
- 2、修改file.conf

```
service {
  #vgroup->rgroup
                                                    #vgroup_mapping.key=value，当value为default时，value默认=key。
  vgroup_mapping.my_test_tx_group = "my_tx_group"   #指定服务的事务组，如果value是default，表示value默认=key
  #only support single node
  default.grouplist = "127.0.0.1:8091"
  #degrade current not support
  enableDegrade = false
  #disable
  disable = false
  #unit ms,s,m,h,d represents milliseconds, seconds, minutes, hours, days, default permanent
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
}
store {  #事务日志存储
  ## store mode: file、db
  mode = "db"  #指定存储方式

  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp) etc.
    datasource = "dbcp"
    ## mysql/oracle/h2/oceanbase etc.
    db-type = "mysql"
    driver-class-name = "com.mysql.cj.jdbc.Driver"
    url = "jdbc:mysql://localhost:3306/seata"
    user = "root"
    password = "root"
    min-conn = 1
    max-conn = 3
    global.table = "global_table"
    branch.table = "branch_table"
    lock-table = "lock_table"
    query-limit = 100
  }
}
```
- 3、准备分布式事务日志存储数据库（file.conf中指定的db，这里是mysql）

``` mysql
-- the table to store GlobalSession data
drop table if exists `global_table`;
create table `global_table` (
  `xid` varchar(128)  not null,
  `transaction_id` bigint,
  `status` tinyint not null,
  `application_id` varchar(32),
  `transaction_service_group` varchar(32),
  `transaction_name` varchar(128),
  `timeout` int,
  `begin_time` bigint,
  `application_data` varchar(2000),
  `gmt_create` datetime,
  `gmt_modified` datetime,
  primary key (`xid`),
  key `idx_gmt_modified_status` (`gmt_modified`, `status`),
  key `idx_transaction_id` (`transaction_id`)
);

-- the table to store BranchSession data
drop table if exists `branch_table`;
create table `branch_table` (
  `branch_id` bigint not null,
  `xid` varchar(128) not null,
  `transaction_id` bigint ,
  `resource_group_id` varchar(32),
  `resource_id` varchar(256) ,
  `lock_key` varchar(128) ,
  `branch_type` varchar(8) ,
  `status` tinyint,
  `client_id` varchar(64),
  `application_data` varchar(2000),
  `gmt_create` datetime,
  `gmt_modified` datetime,
  primary key (`branch_id`),
  key `idx_xid` (`xid`)
);

-- the table to store lock data
drop table if exists `lock_table`;
create table `lock_table` (
  `row_key` varchar(128) not null,
  `xid` varchar(96),
  `transaction_id` long ,
  `branch_id` long,
  `resource_id` varchar(256) ,
  `table_name` varchar(32) ,
  `pk` varchar(36) ,
  `gmt_create` datetime ,
  `gmt_modified` datetime,
  primary key(`row_key`)
);
```
- 目前server只支持mysql5，如果是mysql8的话，可以先替换lib中的mysql-connect的jar包，换成8版本。同时file.conf中修改：driver-class-name = "com.mysql.cj.jdbc.Driver"


2、seata客户端
* 依赖（排除默认的seata，加上对应seata-server版本的seata依赖）。注意：同时需要注册中心，这里演示是使用nacos
``` xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
    <exclusions>
        <exclusion>
            <artifactId>seata-all</artifactId>
            <groupId>io.seata</groupId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-all</artifactId>
    <version>0.9.0</version>
</dependency>
```
- 配置：

```yaml
spring:
  cloud:
    alibaba:
      seata:
        tx-service-group: my_tx_group  #自定义事务组，该事务组名应该和conf上的对应vgroup_mapping的key有对应
```
* 添加registry.conf。（当conf获取方式为是file时，还需要file.conf）。这里演示的是file方式
	* 可以复制server上的registry.conf，和file.conf
	* 注意，可能需要修改file.conf

```
service {
  #vgroup->rgroup
                                            #vgroup_mapping.key=value，当value为default时，value默认=key。
  vgroup_mapping.my_tx_group = "default"    #这里是客户端的事务组，其value应该和server上需要有对应
  #only support single node
  default.grouplist = "127.0.0.1:8091" 
```
- 配置seata的对数据源的代理

``` java
@Configuration
public class DataSourceProxyConfig {

    @Value("${mybatis.mapper-locations}")
    private String mapperLocations;

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource druidDataSource(){
        return new DruidDataSource();
    }

    @Bean
    public DataSourceProxy dataSourceProxy(DataSource dataSource) {
        return new DataSourceProxy(dataSource);
    }

    @Bean
    public SqlSessionFactory sqlSessionFactoryBean(DataSourceProxy dataSourceProxy) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSourceProxy);
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources(mapperLocations));
        sqlSessionFactoryBean.setTransactionFactory(new SpringManagedTransactionFactory());
        return sqlSessionFactoryBean.getObject();
    }

}
```
或

``` java
@Configuration
public class DataSourceProxyConfig {

    @Autowired
    DataSourceProperties dataSourceProperties;

    @Bean
    public DataSource druidDataSource(){
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl(dataSourceProperties.getUrl());
        dataSource.setUsername(dataSourceProperties.getUsername());
        dataSource.setPassword(dataSourceProperties.getPassword());
        dataSource.setDriverClassName(dataSourceProperties.getDriverClassName());

        return new DataSourceProxy(dataSource);   //对原有的dataSource生成代理
    }

}
```
- 去除默认的数据源配置
    @SpringBootApplication(exclude = DataSourceAutoConfiguration.class)

* 由于是分布式事务，所以每个服务对应一个数据库。这些数据库都需要配置回滚日志表


3、测试：
* 配置三个seata客户端，每个客户端配置一个数据库，每个数据库分别对应订单Order、库存stroage、账户account，这样简单的实现分布式数据库
* 注意：每个数据库都要都有回滚日志表

```
CREATE TABLE t_order(
`id` BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
`user_id` BIGINT(11) DEFAULT NULL COMMENT '用户id',
`product_id` BIGINT(11) DEFAULT NULL COMMENT '产品id',
`count` INT(11) DEFAULT NULL COMMENT '数量',
`money` DECIMAL(11, 0) DEFAULT NULL COMMENT '金额',
`status` INT(1) DEFAULT NULL COMMENT '订单状态：0：创建中；1：已完成'
)ENGINE=INNODB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8;

CREATE TABLE t_storage(
    `id` BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `product_id` BIGINT(11) DEFAULT NULL COMMENT '产品id',
    `total` INT(11) DEFAULT NULL COMMENT '总库存',
    `used` INT(11) DEFAULT NULL COMMENT '已用库存',
    `residue` INT(11) DEFAULT NULL COMMENT '剩余库存'
)ENGINE=INNODB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;


INSERT INTO t_storage VALUES(1, 1, 100, 0, 100);  #初始化库存

CREATE TABLE t_account(
    `id` BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY COMMENT 'id',
    `user_id` BIGINT(11) DEFAULT NULL COMMENT '用户id',
    `total` DECIMAL(10, 0) DEFAULT NULL COMMENT '总额度',
    `used` DECIMAL(10, 0) DEFAULT NULL COMMENT '已用余额',
    `residue` DECIMAL(10, 0) DEFAULT '0' COMMENT '剩余可用额度'
)ENGINE=INNODB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

INSERT INTO t_account VALUES(1, 1, 1000, 0, 1000); #初始化账户
```
- 回滚日志表
```
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```
* 服务调用链：Order->storage ->Account
* 使用@GlobalTransactional实现分布式事务
``` java
@GlobalTransactional(name="create-order",rollbackFor=Exception.class) #表示当出现方法中抛出Exception异常，则进行全局回滚
public void create(Order order) {...}
```

