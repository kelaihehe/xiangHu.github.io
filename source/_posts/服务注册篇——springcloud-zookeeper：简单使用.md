---
title: 服务注册篇——springcloud zookeeper：简单使用
date: 2020-07-15 09:55:17
tags:
  - SpringCloud
  - zookeeper
summary: springcloud zookeeper的简单使用
categories: SpringCloud
---


## springcloud zookeeper进行服务注册 ##

1. 依赖
``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
</dependency>
``` 
2. 配置
``` yaml
spring:
  application:
    name: cloud-provider-payment
  cloud:
    zookeeper:
      connect-string: 127.0.0.1:2181   #zookeeper的ip：port
```
3. 启动类激活服务发现
{% asset_img Image1.png  %}

## 服务间的调用（和eureka中的服务调用一样） ##
- 使用RestTemplate进行服务调用
	1. 注入RestTemplate，记得加上LoadBalanced
{% asset_img Image2.png  %}
		- 需要加@LoadBalanced注解，会在RestTemplate执行之前进行拦截，通过服务名获取port:ip，再执行远程调用
	2. 使用restTemplate实例访问服务
{% asset_img Image3.png  %}
