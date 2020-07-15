---
title: 负载均衡篇——springCloud Ribbon：简单使用
date: 2020-07-15 10:09:06
tags:
  - SpringCloud
  - Ribbon
summary: springcloud Ribbon的简单使用
categories: SpringCloud
---
Ribbon是工作在消费端的，消费端在调用服务时，通过ribbon的负载均衡机制来选择调用服务中其中一个实例

## Ribbon的使用 ##
- Ribbon在服务注册篇已经使用过了。不管是eureka、zookeeper还是consul，它们的依赖中已经包含了Ribbon
- ribbon当然也可以脱离这些框架使用：[官网第7.6节](https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.2.0.RELEASE/reference/html/#spring-cloud-ribbon-without-eureka)  

## 配置负载均衡算法（两种方式） [见官网第7节](https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.2.0.RELEASE/reference/html/#customizing-the-ribbon-client) 
- 1、配置properties文件
``` yaml
serviceName:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule   #指定哪个服务使用哪个负载均衡算法
    NFLoadBalancerPingClassName: com.netflix.loadbalancer.DummyPing  #指定哪个服务使用哪种服务存活检测
```
- 2、使用@RibbonClient
	- 局部配置方式

``` java
/*********************单个配置方式**************/
@Configuration
@RibbonClient(value = "CLOUD-PROVIDER-SERVICE",configuration = MyRule.class)
public class Balance4ServiceConfig {
}

@Configuration
public class MyRule {   //该配置类必须要有@Configuration注解，且不能在@componentScan注解下。否则就变成全局Client的配置了
    @Bean
    public IRule iRule(){
        return new RoundRobinRule();
    }

    @Bean
    public IPing iPing(){
        return new PingUrl(false,"/provider");
    }
}


/*********************多个配置方式**************/
@Configuration
@RibbonClients({@RibbonClient(value = "service1",configuration = MyRule1.class),
                @RibbonClient(value = "service2",configuration = MyRule2.class)})
public class Balance4ServiceConfig {
}
```
- 
	- 全局配置方式
		- 在@componentScan注解下，注入IRule的bean和IPing的Bean等

### 各默认的配置类 ###
- IRule：默认为**ZoneAvoidanceRule**，里面的实现就是轮询
{% asset_img Image1.png  %}
- IPing：默认为**DummyPing**，直接返回true，不进行ping


### 默认的轮询算法+默认ping规则可能会出现的问题 ###
- 如果一个服务的某个实例宕机了，由于默认不进行ping，直接返回true，所以可能会轮询到已经宕机的服务实例，导致访问失败
- 解决方案：
	- 修改IPing实现类，使用PingUrl类（存在的问题：这样每次都要去ping，增加时延）
		- new PingUrl(false,"/provider");  //第一个参数表示http访问ping，true表示http。第二个参数表示实例下的路径。表示访问服务实例之前，去ping该实例下的/provider。如果正常访问，则返回true
