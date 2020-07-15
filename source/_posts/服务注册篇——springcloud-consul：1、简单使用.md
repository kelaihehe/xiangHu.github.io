---
title: 服务注册篇——springcloud consul：简单使用
date: 2020-07-15 09:44:31
tags:
  - SpringCloud
  - consul
summary: springcloud consul的简单使用
categories: SpringCloud
---

# 1、服务器上consul的安装 #
- 官网下载安装包，解压
- 直接运行启动命令：	
	- 简单命令：./consul agent -ui -client=0.0.0.0  
	- 复杂命令：./consul agent -data-dir /tmp/node0 -node=node0 -bind=0.0.0.0 -datacenter=dc1 -ui -client=0.0.0.0 -server -bootstrap-expect 1
		- data-dir 指定数据目录，其他的节点对于这个目录必须有读的权限
		- node 指定节点的名称
		- bind 为该节点绑定一个地址
		- datacenter 数据中心名称
		- ui 使用自带的ui
		- client 指定web ui的监听地址，默认127.0.0.1只能本机访问，改为0.0.0.0可外网访问
		- server 运行在server模式
		- bootstrap-expect 集群期望的节点数，只有节点数量达到这个值才会选举leader
		- config-dir 指定配置文件，定义服务的，默认所有一.json结尾的文件都会读
		- enable-script-checks=true 设置检查服务为可用
		- join 加入到已有的集群中
	- windows的简单启动命令：
		- consul agent -dev

	- 通过web ui访问，默认端口为8500
{% asset_img Image1.png  %}

# 2、springcloud consul的服务注册 #
- 依赖
``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```
- 配置
``` yaml
spring:
  application:
    name: consul-consumer-order
  ###consul注册中心地址
  cloud:
    consul:
      host: 39.99.209.200
      port: 8500
      discovery:
        service-name: ${spring.application.name}
```
- 启动类
	- 添加@EnableDiscoveryClient



# 3、服务间的调用 #
- 使用RestTemplate进行服务调用



