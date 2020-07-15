---
title: springCloud Alibaba—nacos：简单使用
date: 2020-07-15 17:03:25
tags:
  - SpringCloud Alibaba
  - nacos
summary: nacos的简单介绍及使用(持久化、集群配置)
categories: SpringCloud Alibaba
---

**Nacos**就是服务注册中心+配置中心+前端UI的管理中心，相当于代替了springcloud中的eureka+config，nacos支持ap和cp的切换

1、下载安装后，直接启动，默认的启动端口为8848
* 启动方式：
	* 加-m standalone表示以单机模式启动：
		* sh startup.sh -m standalone   #单机模式启动，并默认使用内嵌数据库
	* 不加-m standalone表示以集群模式启动
		* 加-p embedded表示使用内嵌数据库
			* sh startup.sh -p embedded   #以集群模式启动，且集群中每个nacos都使用内嵌数据库
		* 加-p embedded表示使用外置数据库
			* sh startup.sh  #以集群模式启动，且集群中每个nacos都使用外置数据库
	* 解释：
{% asset_img Image1.png  %}
{% asset_img Image2.png  %}

2、nacos作为注册中心时，springboot作为nacos客户端
* 依赖
``` xml
<dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```
* 启动类
	* 添加@EnableDiscoveryClient
* 配置文件：application.properties

``` yaml
server:
  port: 9001

spring:
  application:
    name: nacos-provider
  cloud:
    nacos:
      discovery.server-addr: localhost:8848 #作为服务注册中心的nacos

management:
  endpoints:
    web:
      exposure:
        include: "*"  #暴露所有端点
```
3、nacos作为配置中心时，springboot作为nacos的config客户端
* 依赖
``` xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```
* 配置文件：bootstrap.properties

``` yaml
spring:
  application:
    name: nacos-consumer

  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848    
      config:
        server-addr: localhost:8848     #作为config中心的nacos地址
        file-extension: yaml    #支持的配置文件yaml
        group: mygroup          #组名，不指定即默认gourp：public
        namespace: mynamespace  #命名空间，不指定即默认namespace

  profiles:
    active: dev
```
* config客户端将会获取nacos中的DataID为${spring.cloud.nacos.config.prefix}-${spring.profile.active}.${spring.cloud.nacos.config.file.extension}的配置内容
	* ${spring.cloud.nacos.config.prefix}默认是${spring.application.name}
	* 当配置了namespace和group时，会去对应的namespace下的对应group中的DataID对应的配置
	* 该DataID必须提前在nacos中配置好
{% asset_img Image3.png  %}
* 当nacos中配置更新了，之后会自动去刷新config客户端的bean（在springcloud config中还得去post一个refresh请求，这里会自动请求）

4、nacos持久化
* nacos中这些配置文件将会默认保存在nacos的内嵌数据库derby中。
* nacos也支持用外置数据库，目前只支持mysql，且版本是5.6.5+，不支持8
* 配置外置数据库
	* 首先准备数据库，创建一个新的数据库，并执行nacos/conf下的nacos-mysql.sql文件中的建表语句
	* 修改nacos/conf/application.properties

``` 
spring.datasource.platform=mysql  #使用mysql

db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=xxx
db.password=xxx
```
*  重启nacos

5、nacos集群配置（3个及3个以上才能运行集群，可以nginx实现反向代理和负载均衡）
* 修改集群中每个nacos的conf/cluster.conf，添加集群中每个服务的ip:port（该ip是本机ip，必须是通过hostname -i 获取的本地ip）

```
172.26.226.176:3333
172.26.226.176:4444
172.26.226.176:5555
```
* 启动每个nacos，即可。
* 示例：当前演示是同一台机器的同一个nacos。由一个nacos启动不同端口的nacos实例（注意的是，由一个nacos创建多个实例，必须用外置数据库，不然会出现内置数据库已经被创建的错误提示，导致启动失败）
	* 在修改完cluster.conf后，修改启动脚本startup.sh
		* 添加变量：PORT，默认为8848。 添加参数P：和对应的赋值
		* 修改JAVA_OPT变量，添加-Dserver.port=${PORT}指定启动端口
{% asset_img Image4.png  %} 
	* 启动
		* sh startup.sh -P 8001
		* sh startup.sh -P 8002
		* sh startup.sh -P 8003

部分参考： https://www.cnblogs.com/larscheng/p/11427733.html
