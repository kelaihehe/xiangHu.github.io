---
title: springCloud Alibaba—sentinel
date: 2020-07-15 17:23:41
tags:
  - SpringCloud Alibaba
  - sentinel
summary: sentinel的简单介绍及使用
categories: SpringCloud Alibaba
---

sentinel：实现了服务熔断降级+服务限流+热点限流+前端UI控制台

### 1、sentinel服务端 ###
* 运行sentinel-dashboard.jar，一个springboot编写的sentinel服务端， 默认端口为8080
* 控制台
{% asset_img Image1.png  %}

### 2、sentinel客户端 ###
* 依赖
``` xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```
* 配置
``` yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080   #设置sentinel服务端地址 ，接入
```
* 测试（只有在第一次访问了api后，才会在sentinel控制台显示链路。首次调用的时候进行初始化，开始向控制台发送心跳包）
	* 流控规则
		* 资源名（默认为请求路径）
		* 阈值类型：
			* QPS：每秒到达请求数
			* 线程数：每秒执行的线程数
		* 流控模式：
			* 直接：资源超过阈值条件，直接限流
			* 关联：当关联的其他资源超过了该阈值条件，将自己限流
			* 链路：当入口资源流量到达阈值，整个调用链路都会限流
		* 流控效果（针对QPS，线程数方式没有）：
			* 快速失败：直接抛异常
			* warm up：根据codeFactor的值（默认为3，可配置），初始阈值为阈值/codeFactor，然后在设定的预热时长内，慢慢增加请求阈值，在预热时长结束时到达阈值。（效果是：刚开始能接受的请求数少，慢慢地增多）
			* 排队等待：当请求到达时，正在处理的请求数到达阈值，那么将进行排队等候，如果超过超时时间，那么将抛出异常。（效果是：对于服务器来说每秒只能最多只能处理阈值数大的请求）
{% asset_img Image2.png  %}
	* 降级规则   [官网](https://github.com/alibaba/Sentinel/wiki/%E7%86%94%E6%96%AD%E9%99%8D%E7%BA%A7)
		* 降级策略
			* RT：平均响应时间。
				* 触发熔断降级条件：1s收到超过5个请求&&1s中平均请求响应时间大于设定的阈值
				* 条件满足后：在未来的设定的时间窗口内进入熔断状态
			* 异常比例：
				* 触发熔断降级条件：1s收到超过5个请求&&1s中（异常请求数/总通过的请求数）大于设定的阈值
				* 条件满足后：在未来的设定的时间窗口内进入熔断状态
			* 异常数：
				* 触发熔断降级条件：在1s内的异常请求数大于阈值
				* 条件满足后：在未来的设定的时间窗口内进入熔断状态
{% asset_img Image3.png  %}
	* 热点规则
		* 对某些特定的api请求或api中某个参数请求进行限流
		* 参数索引：api的参数中的位置，从0开始
		* 参数例外项
			* 参数类型，该参数索引对应的参数的类型
			* 参数值：该参数的某个特定值
			* 限流阈值：对应的限流阈值
			* 意义：当访问该特例时，阈值也不同
{% asset_img Image4.png  %}
	* 系统规则（控制系统整体的流量）   [官网](https://github.com/alibaba/Sentinel/wiki/%E7%B3%BB%E7%BB%9F%E8%87%AA%E9%80%82%E5%BA%94%E9%99%90%E6%B5%81)
{% asset_img Image5.png  %}
{% asset_img Image6.png  %}
	* 授权规则
		* 直接设定禁止或允许某些服务对资源的访问
{% asset_img Image7.png  %} 

3、sentinel持久化   https://github.com/alibaba/Sentinel/wiki/动态规则扩展
* sentinel中限流、熔断等配置，可以通过配置数据源来实现对配置进行持久化
* 分为两种模式：拉模式和推模式
* 拉模式（主动式）是，客户端定期去管理中心拉取配置。目前支持的有：文件、consul
* 推模式（响应式）是，客户端通过监听机制获取配置更新。目前支持的有：Nacos、ZooKeeper, Redis,Apollo, etcd
* 这里演示nacos
	* 依赖
``` xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
    <version>x.y.z</version>
</dependency>
```
	* 配置
``` yaml
spring:
    sentinel:
      datasource:
        ds1:  #数据源1
          nacos:
            server-addr: localhost:8848 #nacos地址
            data-id: sentinel #nacos中的data-ID
            data-type: json #文件格式
            rule-type: flow #可选flow，degrade，authority，system, param-flow, gw-flow, gw-api-group
```
	* flow，degrade，authority，system, param-flow，分别表示获取流控规则、降级规则、授权规则、系统规则和热点规则
	* gw-flow, gw-api-group分别表示网关的限流和分组
* 配置表示，根据Data-ID从nacos获取配置信息
* nacos上需先配置上，例如：
``` 
[
    {
        "resource": "/hello",
        "limitApp": "default",  #default表示对所有应用来调用该接口都进行监控
        "grade": 1,
        "count": 5,
        "strategy": 0,
        "controlBehavior": 0,
        "clusterMode": false
    }
]
```
* 对于不同规则，json中对应key-value如何填？
	* 根据rule-type去找对应的AbstractRule的实现类，找key（包括AbstractRule的resource和limitApp）
	* 再根据RuleConstant去找key去找对应的可选项和意义

4、注解@SentinelResource
* 在接口方法上标注使用

``` java
@SentinelResource(value = "resourceName1",blockHandler = "myblockHandler",fallback = "myfallback",exceptionsToIgnore = {},blockHandlerClass=XXX.class,fallbackClass=XXX.class)
```
* value指定资源名
* blockHandler表示：当该api被调用时不符合相应的规则，会抛出BlockException。由该blockHandler处理该异常。若不指定，则有sentinel默认的blockHandler处理
* fallback：由fallback来处理BlockException之外的异常
* exceptionsToIgnore：表示忽略哪些异常
* blockHandlerClass：指定BlockException处理类。若未指定，默认是当前类
* fallbackClass：指定其他Exception处理类。若未指定，默认是当前类

``` java
public String myblockHandler(BlockException blockException){   
    return "custom blockHandler....";
}

public String myfallback(Throwable t){  #参数可以指定Throwable，也可以不指定
    return "myfallback....";
}
```

### 附加 ###
解析降级规则-DegradeRule：

``` java
private final AtomicBoolean cut = new AtomicBoolean(false);  //熔断器开关状态，开始状态为关闭

public boolean passCheck(Context context, DefaultNode node, int acquireCount, Object... args) {
    if (this.cut.get()) {  //先判断熔断器开关状态，若开启，直接返回不通过，若关闭，再进行下一步判断
        return false;
    } else {
        ClusterNode clusterNode = ClusterBuilderSlot.getClusterNode(this.getResource());  
        if (clusterNode == null) {
            return true;
        } else {
            double exception;
            if (this.grade == 0) {    //grade=0 表示根据RT进行熔断降级
                exception = clusterNode.avgRt();   //获取平均时间RT
                if (exception < this.count) {  //若小于阈值，直接放行
                    this.passCount.set(0L);
                    return true;
                }

                if (this.passCount.incrementAndGet() < 5L) { //若大于阈值，但请求数小于5，直接放行
                    return true;
                }
            } else if (this.grade == 1) {  //grade=1 表示根据异常率进行熔断降级
                exception = clusterNode.exceptionQps();
                double success = clusterNode.successQps();
                double total = clusterNode.totalQps(); 
                if (total < 5.0D) {  //若请求数小于5，直接放行
                    return true;
                }

                double realSuccess = success - exception;
                if (realSuccess <= 0.0D && exception < 5.0D) {  //失败数小于5且成功数小于0的情况则无视，直接放行
                    return true;
                }

                if (exception / success < this.count) { //若异常lv小于阈值，直接放行
                    return true;
                }
            } else if (this.grade == 2) {  //grade=2 表示根据异常数进行熔断降级
                exception = (double)clusterNode.totalException();
                if (exception < this.count) { //若异常数小于阈值，直接放行
                    return true;
                }
            }

            if (this.cut.compareAndSet(false, true)) {   //若上面规则未通过，则将熔断器状态变更为true，并设置定时时间
                DegradeRule.ResetTask resetTask = new DegradeRule.ResetTask(this);
                pool.schedule(resetTask, (long)this.timeWindow, TimeUnit.SECONDS);
            }

            return false;
        }
    }
}
```
