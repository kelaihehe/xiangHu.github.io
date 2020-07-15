---
title: 分布式配置篇——springcloud Config
date: 2020-07-15 16:08:41
tags:
  - SpringCloud
  - Config
summary: SpringCloud Config的简单介绍及使用
categories: SpringCloud
---
## 一、简单的分布式配置 ##
### 1、创建config server中心 ###
依然需要添加到注册中心，前提是需要项目所在系统能够与github仓库建立通信
- 依赖
``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```
- 启动类：
	- 添加@EnableConfigServer 
- 配置

``` yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: git@github.com:kelaihehe/configCenterTest.git  #github仓库上面的git仓库名字
          search-paths:   ##搜索目录
            - configCenterTest
      label: master  #读取分支
```
* 测试：直接通过路径访问仓库的文件

	* 三种路径方式（label表示仓库分支，profile表示配置文件的的profile）
		* /label/application-profile.yml   例如 ：http://localhost:3344/master/config-dev.yml
		* /application-profile.yml         例如：http://localhost:3344/config-dev.yml
		* /application/profile/label.yml    例如：http://localhost:3344/config/dev/master.yml

### 2、各服务作为config客户端 ###
* 添加依赖
``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```
* 配置文件（两种方式）见 [](https://cloud.spring.io/spring-cloud-config/reference/html/#config-first-bootstrap)  Spring Cloud Config Client下的Config First Bootstrap和Discovery First Bootstrap
	* 1、使用spring.cloud.config.uri指定configserver
		* 添加bootstrap.yml文件配置，bootstrap.yml的优先级高于application.yml，即如果出现相同的配置，以application为准
		* spring.cloud.config.uri必须在bootstrap文件中

``` yaml
spring:
  cloud:
    config:
      label: master
      name: config
      profile: dev
      uri: http://localhost:3344  #config server的地址
```
- 	* 2、使用spring.cloud.config.discovery指定config配置中心的服务名
		* 同样需要bootstrap.yml文件配置，除了spring.cloud.config.discovery，eureka 服务中心地址：eureka.client.serviceUrl.defaultZone也必须在bootstrap.yml

``` yaml
spring:
  cloud:
    config:
      discovery:
        enabled: true
        service-id: CLOUD-CONFIG-CENTER
      name: config
      profile: dev
      label: master

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:7001/eureka
```
- 测试

``` java
@RestController
public class ConfigClientController {
    @Value("${config.info}")  //config server上的配置（包括git仓库中的配置）
    private String configInfo;   

    @GetMapping("/configInfo") 
    public String getConfigInfo(){
        return configInfo;
    }
}
```
注意点：Config Server一定要先开启，否则config客户端依赖的配置找不到，就会报错


## 二、动态刷新的分布式配置 ##
1、如果github出现更新，config server的配置可以随着更新，config客户端的配置也随之更新。但客户端中一些依赖配置的bean实例并没有得到更新
* 实现config客户端的bean动态刷新
	* 1、添加actuator依赖，接受监控
	* 2、修改配置，暴露监控端口

``` yaml
management:
  endpoints:
    web:
      exposure:
        include: "refresh"  #添加配置，暴露refresh端点
```
- 	* 3、需要github上的配置属性的controller类添加@RefreshScope，刷新功能，即刷新单例bean的属性
	* 4、给客户端发送post进行刷新：curl -X POST "http://localhost:80/actuator/refresh"

2、**为什么这样就可以让客户端的bean动态更新？(源码分析)**
* 1、首先actuator的端点refresh

``` java
@Endpoint(id = "refresh")
public class RefreshEndpoint {
    private ContextRefresher contextRefresher;

    public RefreshEndpoint(ContextRefresher contextRefresher) {
        this.contextRefresher = contextRefresher;
    }

    @WriteOperation
    public Collection<String> refresh() {
        Set<String> keys = this.contextRefresher.refresh();  //这里会刷新特定的bean
        return keys;
    }
}
```
- 	* contextRefresher.refresh

``` java
public synchronized Set<String> refresh() {
    Set<String> keys = this.refreshEnvironment();  //一、更新环境配置，返回更新的配置
    this.scope.refreshAll();  //二、调用refreshScope的refreshAll：1、先删除scope中的cache，再发布重新refresh事件
    return keys;
}
```
- 	* 一、调用refreshEnvironment

``` java
public synchronized Set<String> refreshEnvironment() {
    Map<String, Object> before = this.extract(this.context.getEnvironment().getPropertySources());  //获取当前的配置
    this.addConfigFilesToEnvironment();  //获取最新的配置
    Set<String> keys = this.changes(before, this.extract(this.context.getEnvironment().getPropertySources())).keySet();  //找出更新的部分
    this.context.publishEvent(new EnvironmentChangeEvent(this.context, keys));  //发布事件
    return keys;
}
```
-       * EnvironmentChangeEvent事件发布后，会被ConfigurationPropertiesRebinder监听到，它在onApplicationEvent方法中将配置了@ConfigurationProperties的bean全部重新装配属性（所以配置类一般用ConfigurationProperties来）
	* 二、调用refreshScope的refreshAll() 。那些加了@RefreshScope配置的bean都会刷新

``` java
public void refreshAll() {
    super.destroy();  //清除refreshScope中维护的cache中的所有bean
    this.context.publishEvent(new RefreshScopeRefreshedEvent());  //发布刷新事件
}
```
-		* RefreshScopeRefreshedEvent将会被EurekaClientConfigurationRefresher监听到，它会将refresh的scope的bean重新创建

3、另一种方式：通过**springcloud bus**，实现消息广播，将配置刷新通知消息通过广播形式将通知所有config客户端实现动态刷新
* 可以通过由一个config客户端广播给其他客户端，也可以由一个config server来广播所有config 客户端，一般使用第二种
* 演示由config server来广播所有config 客户端
	* config serve添加bus依赖，同时需要actuator依赖

``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```
* 配置

```yaml
#配置rabbitmq
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest


management:
  endpoints:  
    web:
      exposure:
        include: 'bus-refresh'  #暴露actuator的"bus-refresh"端点
```
* 刷新时，就不再是对config 客户端发送refresh请求了，而是向配置了bus的config server发送请求：
	* curl -X POST "http://localhost:3344/actuator/bus-refresh/service-id:port " 表示广播特定服务名的特定端口的服务
	* curl -X POST "http://localhost:3344/actuator/bus-refresh/service-id"  表示广播特定服务名的服务
	* curl -X POST "http://localhost:3344/actuator/bus-refresh" 表示广播所有注册的服务


4、总结：
* 动态刷新的原理机制大致为：
	* 通过actuator的refresh端点或bus-refresh端点，对config客户端的bean进行刷新
	* scope为“refersh”的bean会被刷新；
	* 配置了@ConfigurationProperties的bean全部重新装配属性

5、结论
* 以后配置类尽量用@ConfigurationProperties
* 以后需要动态刷新的bean需要加上@Compoent


注意：
* @RefreshScope+@Compoent会导致属性注入失败
	* 需要RefreshScope加上属性proxyMode=ScopedProxyMode.DEFAULT，，即@RefreshScope(proxyMode=ScopedProxyMode.DEFAULT)+@Compoent，属性才会注入




