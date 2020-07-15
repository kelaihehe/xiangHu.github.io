---
title: 服务熔断篇：hystrix
date: 2020-07-15 11:29:43
tags:
  - SpringCloud
  - hystrix
summary: hystrix的简单介绍及使用
categories: SpringCloud
---
熔断器Hystrix是一种容错框架，包括线程/信号量隔离，服务降级、服务熔断，服务限流，提供分布式系统的弹性

### 什么是服务降级？ ###
- 如果服务器出现超时或程序运行异常等，不能让客户端等太久或返回客户端一个不好的错误页面，而应该返回一个友好提示。
- 换句话说，如果主逻辑出现问题，应该换一个备用逻辑使用
- 一般在最外层的服务使用服务降级

### 什么是服务熔断？ ###
- 仍是服务器出现问题，可以用服务降级来处理。但仍会有一些请求进来，并走主逻辑，这样仍会占用服务资源。
- 如果干脆在未来一段时间内容，拦截所有请求，即打开熔断器，全部直接走备用逻辑，给服务器一个缓冲时间。当一段时间过后，再尝试接受部分请求，走主逻辑，如果服务正常，那么就关闭熔断器，恢复服务运行。

### 服务降级和服务熔断的关系 ###
- 类似于springboot和springcloud的关系）
- 服务熔断需要依赖服务降级。
- 服务熔断只是服务降级的条件之一。

### 什么是服务限流？ ###
- 待续

### 分布式框架出现的服务雪崩问题 ###
- 在分布式框架中，多个微服务之间进行调用。假如：微服务A调用微服务B，微服务B又调用了微服务C，微服务C又调用了其他的微服务，这称之为“扇出”。
- 当扇出的链路上的某个微服务的调用响应时间过长或不可用，那么对微服务A的调用就会占用越来越多的系统资源，从而引发系统雪崩

### 服务熔断和服务降级的同异 ###
- 同：对于用户而言，二者都使其体验到某些功能暂时不可用
- 异：
	- 触发条件不一样：服务熔断一般是在某个服务故障引起的，而服务降级一般是从整体负荷考虑
	- 管理目标的层次不一样：服务熔断几乎每个微服务都需要；但是降级一般可能只需要在某些业务微服务进行

### hystrix的参数配置 ###
- 官网： https://github.com/Netflix/Hystrix/wiki/Configuration
- 中文：https://www.cnblogs.com/520playboy/p/8074347.html
- 一些关键配置：
	- Execution前缀的属性配置：
``` 
execution.isolation.strategy 隔离策略，默认是Thread, 可选Thread｜Semaphore
	https://www.jianshu.com/p/df1525d58c20
	https://www.jianshu.com/p/af8dc67e5238
execution.timeout.enabled 执行是否启用超时，默认启用true
execution.isolation.thread.timeoutInMilliseconds 命令执行超时时间，默认1000ms
Circuit Breaker前缀的属性配置
circuitBreaker.enabled：默认true，用来跟踪circuit的健康性，如果未达标则让request短路。
circuitBreaker.requestVolumeThreshold：默认20，一个rolling window内最小的请求数。如果设为20，那么当一个rolling window的时间内（比如说1个rolling window是10秒）收到19个请求，即使19个请求都失败，也不会触发circuit break。
circuitBreaker.sleepWindowInMilliseconds：默认5000ms，触发短路的时间值，当该值设为5000时，则当触发circuit break后的5000毫秒内都会拒绝request，也就是5000毫秒后才会关闭circuit。
circuitBreaker.errorThresholdPercentage：默认50，错误比率阀值，如果错误率>=该值，circuit会被打开，并短路所有请求触发fallback。
circuitBreaker.forceOpen 强制打开熔断器
circuitBreaker.forceClosed 强制关闭熔断器
metrics前缀的属性配置
metrics.rollingPercentile.enabled 执行时是否enable指标的计算和跟踪，默认true
metrics.rollingPercentile.timeInMilliseconds 设置rolling percentile window的时间，默认10000
```
	* 默认配置下的服务熔断：
		* 在timeInMilliseconds（10s）时间内的错误率大于errorThresholdPercentage（50%） || 在timeInMilliseconds（10s）时间中，超过requestVolumeThreshold（20）个请求，将打开断路器，并沉默sleepWindowInMilliseconds（5s）时间，进入开启状态，不再接受请求，并进行服务降级。
		* 时间过后，进入半关闭状态，尝试接受一次请求，若成功，再恢复到关闭状态，否则继续开启，重新休眠计时
	* 默认配置下的服务降级：
		* 触发服务降级的条件：
			* 1、出现异常
			* 2、执行时间超时：超过timeoutInMilliseconds（1s）
			* 3、服务熔断开启
			* 4、线程池 / 信号量打满也会导致服务降级

## 服务降级 ##
* 依赖（如果有feign，则不需要，因为fegin集成了hystrix）
``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```
* 启动类上添加
	* @EnableCircuitBreaker
* 使用方式：
	* 1、在每个方法上添加@HystrixCommand注解，并添加属性fallbackMethod， 表示为该方法指定fallback方法。@HystrixCommand可以放在controller方法和service方法上。
		* 如果该方法触发了Hystrix条件，则会调用回调方法
``` java
//指定回调方法
@HystrixCommand(fallbackMethod="paymentInfo_TimeOutHandler")   
```
	* 2、在@DefaultProperties(defaultFallback = "globalFallBackMethod")，同时在需要进行降级的方法上添加@HystrixCommand，但不声明fallbackMethod属性，表示使用默认的defaultFallback。@DefaultProperties只能放在类上
{% asset_img Image1.png  %}

* 配置hystrix参数  (参数名从HystrixCommandProperties类中找)
	* 1、通过注解
		* @HystrixCommand和@DefaultProperties都有一个属性：commandProperties，通过commandProperties对服务降级条件和服务熔断条件设置
		* 示例：设置超时时间
``` java
@HystrixCommand(fallbackMethod = "paymentTimeOutFallBackMethod",commandProperties = {        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "1500")  //设置该方法的执行时间不能超过1.5s，否则执行fallbackMethod})
```
		* 2、通过properties文件配置

``` yaml
默认配置：
hystrix:
  command:
    default   
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 500

#为某个具体方法定制的的配置  
hystrix:
  command:
#commandKey名规则在Feign类的configKey方法中定义了  
#commandKey名规则：类名#方法名（参数类型，参数类型）作为commandKey  
    "Service#getProduct(int)":  
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 500

```
## hystrix结合feign[官网1.5节](https://cloud.spring.io/spring-cloud-static/spring-cloud-openfeign/2.2.0.RELEASE/reference/html/#spring-cloud-feign-hystrix-fallback)##    
* feign实现了WebService接口来调用服务。如果想对这个服务调用的接口方法进行服务降级，不需要另写一个方法来调用该接口方法，再对该新方法服务降级，可以直接在feignClient注解上的属性fallback设置回调方法
* properties文件配置：
``` yaml
feign:
  hystrix:
    enabled: true
```
	* 注意：

		* 如果加了这个配置，将默认所有被FeignClient注解的类的方法都会被包装成HystrixCommand。
		* 如果不想所有的FeignClient注解的类都需要服务降级，那么需要加入configuration属性配置,如下

``` java
@FeignClient(value = "CLOUD-PROVIDER-SERVICE",configuration =FeignDisableHystrixConfiguration.class)

public class FeignDisableHystrixConfiguration { 
    @Bean
    @Scope("prototype")
    public Feign.Builder feignBuilder() {
        return Feign.builder();   
    }
}
```
- 
		* feign.hystrix.enable=true之后，会默认注入HystrixFeign的bean，即实现了Hystrix的feign。而这里通过显式的方式，注入普通的Feign，表示不需要服务降级
* 在FeignClient的属性fallback设置回调实现类
{% asset_img Image2.png  %}
{% asset_img Image3.png  %}
* 对于FeignClient这种方式，如何配置hystrix参数?
	* 1、通过properties文件配置。与上面的一致
	* 2、通过feignClient注解的属性configuration
		* 该configuration类不需要用@Configuration注解。但如果用了，需要注意：如果放在ComponentScan下，则会被当成全局配置

``` java
@ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
protected static class HystrixFeignConfiguration {
    @Bean
    @Scope("prototype")
    @ConditionalOnMissingBean
    @ConditionalOnProperty(name = "feign.hystrix.enabled", matchIfMissing = true)
    public Feign.Builder feignHystrixBuilder() {
        return HystrixFeign.builder().setterFactory(new SetterFactory() {
            @Override
            public HystrixCommand.Setter create(Target<?> target, Method method) {
                return HystrixCommand.Setter
                        .withGroupKey(HystrixCommandGroupKey.Factory.asKey(MessageService.class.getSimpleName()))// MessageService下的所有方法的Hystrix Configuration
                        .andCommandPropertiesDefaults(
                                HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(10000) // 超时配置
                        );
            }
        });
    }
}
```
参考：https://segmentfault.com/a/11900000098499322、


## 服务熔断 ##
* 只有开启了circuitBreaker.enabled,才会监控断路器并判断是否打开。默认开启着
* 几个相关参数示例
``` java
@HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback",commandProperties = {
        @HystrixProperty(name="circuitBreaker.enabled",value = "true"),    //是否开启监控curcuit
        @HystrixProperty(name="circuitBreaker.requestVolumeThreshold",value = "2"),
        @HystrixProperty(name="circuitBreaker.sleepWindowInMilliseconds",value = "10000"),
        @HystrixProperty(name="circuitBreaker.errorThresholdPercentage",value = "60"),
        @HystrixProperty(name="metrics.rollingStats.timeInMilliseconds",value="10000")
})
public String paymentCircuitBreaker(Integer id){
    if(id<0){
        throw new RuntimeException("id 不能为负数");
    }
    return Thread.currentThread().getName()+"调用成功";
}
```




