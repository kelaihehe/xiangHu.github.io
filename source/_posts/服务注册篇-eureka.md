---
title: 服务注册篇-eureka1:简单使用
date: 2019-03-4 23:33:52
summary: eureka的简单使用
tags:
  - SpringCloud
  - eureka
categories: SpringCloud
---
# 一、创建eureka注册中心 #
- 添加依赖
``` xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

- 配置application.yml
``` yaml
server:
  port: 7001
eureka:
  instance:
    hostname: eureka-server
  client: #以下两条表明自己是server
    register-with-eureka: false #表示不把自己注册进去
    fetch-registry: false       #表示不从注册中心获取服务
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/     #eureka单机配置，自己的地址  
      #defaultZone:  http://localhost:7002/eureka/,   http://localhost:7003/eureka/          #eureka集群配置，集群中除自己之外其他eureka的地址    
```
- 开启Eureka
{% asset_img Image1.png asd %}
- 运行，访问http://localhost:8761/，就可以看到以下页面
{% asset_img Image2.png asd %}

# 二、创建eureka客户端 #
- 添加依赖
``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
- 2、配置application.yml
``` yaml
server:
  port: 8001
spring:
  application:
    name: provider
#eureka配置1
eureka:
  instance: 
    prefer-ip-address: true  #注册时用ip进行注册
    ip-address: 183.223.218.27
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka/   #eureka注册中心地址
#eureka配置2
eureka:
  client:
    #表示是否将自己注册进EurekaServer默认为true
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true。单节点无所谓，集群必须设置为true才能配合ribbon使用负载均衡
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:7001/eureka        #eureka单机配置
      #defaultZone:  http://localhost:7001/eureka,http://localhost:7002/eureka/,http://localhost:7003/eureka/     #eureka集群配置，集群中所有eureka的地址
```
- 两种eureka配置都可行：配置1是通过显式将服务的ip进行注册，配置2是通过隐式将服务的ip进行注册
- 开启eurekaClient
{% asset_img Image3.png %}
- 运行项目，访问eureka注册中心地址，可以发现是否被注册
{% asset_img Image4.png %}
	- 在eureka上显示该项目名称，则表示成功了


# 三、eureka中注册的服务间的调用 #
- 使用RestTemplate进行服务调用
	- 1、注入RestTemplate，记得加上LoadBalanced
{% asset_img Image5.png %}
		- 需要加@LoadBalanced注解，会在RestTemplate执行之前进行拦截，通过服务名获取port:ip，再执行远程调用
	- 2、使用restTemplate实例访问服务
{% asset_img Image6.png %}
