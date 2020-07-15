---
title: 服务网关篇——Spring Cloud Gateway：简单使用
date: 2020-07-15 12:32:02
tags:
  - SpringCloud
  - Gateway
summary: SpringCloud Gate的简单介绍及使用
categories: SpringCloud
---

### Gateway工作流程 ###
{% asset_img Image1.png  %}

## 创建GateWay客户端（同样需要注册到注册中心） ##
* 1、依赖
``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```	
* 2、配置

``` yaml
spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      routes:
        - id: payment_routh #payment_routh    #路由的ID，没有固定规则但要求唯一，简易配合服务名
          uri: lb://CLOUD-PROVIDER-SERVICE         #匹配后提供服务的路由地址
          predicates:
            - Path=/provider/**          #断言，路径相匹配的进行路由

#        - id: payment_routh2 #payment_routh   #路由的ID，没有固定规则但要求唯一，简易配合服务名
#          uri: http://localhost:8001          #指定固定的ip地址端口
#          predicates:
#            - Path=/payment/lb/**             #断言，路径相匹配的进行路由
```

## predicate ##
- 默认的predicate[官网第4节](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.1.RELEASE/reference/html/#gateway-request-predicates-factories)
{% asset_img Image2.png  %}
* 例如：The After Route Predicate Factory
``` yaml
spring:
  cloud:
    gateway:
      routes:
        - id: payment_routh #payment_routh    #路由的ID，没有固定规则但要求唯一，简易配合服务名
          uri: lb://CLOUD-PROVIDER-SERVICE         #匹配后提供服务的路由地址
          predicates:
            - Path=/provider/**          #使用了Path断言，路径相匹配的进行路由
            - After=2017-01-20T17:42:47.789-07:00[America/Denver]    #表示必须在这之后请求才放行
```

## Filter ##
* Spring Cloud Gateway的filter分为pre类型和post类型：pre用来过滤request请求，post用来过滤response响应
* Gateway中又有全局filter和特定filter，即GlobalFilter和GatewayFilter。GlobalFilter作用于全局，GatewayFilter必须指明某个router
- GlobalFilter：[官网第6节](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.1.RELEASE/reference/html/#global-filters)
* 默认的GlobalFilter有
{% asset_img Image3.png  %}
	* 比如LoadBalancerClient Filter，会对uri: lb://SERVICE的router使用LoadBalancerClient解析出port和ip，即实现负载均衡
* 自定义GlobalFilter：实现记录每次的请求时间

``` java
@Slf4j
@Component 
public class MyGlobalFilter implements GlobalFilter, Ordered {

    public static final String request_begin_time = "request_begin_time";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        exchange.getAttributes().put(request_begin_time,System.currentTimeMillis());  //pre 过滤 ，将请求开始时间加入到请求参数中

        
        return chain.filter(exchange).then(  // 在then方法里的，相当于aop中的后置通知，即post过滤
                Mono.fromRunnable(()->{
                    Long beginTime = exchange.getAttribute("request_begin_time");
                    log.info(exchange.getRequest().getURI().getRawPath() + ": " + (System.currentTimeMillis() - beginTime) + "ms"); //打印请求时间
                })
        );
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```
### GatewayFilter ###
* 默认GatewayFilter [官网第5节](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.1.RELEASE/reference/html/#global-filters)
{% asset_img Image4.png  %}
	* 例如：AddRequestHeader GatewayFilter，添加请求头
		* 配置规则如下

``` yaml
spring:
  cloud:
    gateway:
      routes:
        - id: add_request_header_route
          uri: https://example.org
          filters: 
            - AddRequestHeader=X-Request-red, blue  #表示添加请求头：X-Request-red，值为blue
```
* 自定义GatewayFilter（两种方式）
	* 1、自定义GatewayFilter，并通过java代码将该filter配置到router中
		* 例子：同样是实现记录请求时间。可以看到代码基本和上面的globalFilter一致，只是实现的接口不同

```java
//第一步
@Slf4j
public class MyGateWayFilter implements GatewayFilter, Ordered {   

    public static final String request_begin_time = "request_begin_time";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        exchange.getAttributes().put(request_begin_time,System.currentTimeMillis());  //pre 过滤
        return chain.filter(exchange).then(
                Mono.fromRunnable(()->{
                    Long beginTime = exchange.getAttribute("request_begin_time");
                    log.info(exchange.getRequest().getURI().getRawPath() + ": " + (System.currentTimeMillis() - beginTime) + "ms");
                })
        );
    }

    @Override
    public int getOrder() {
        return 0;
    }
}

//第二步 注入RouteLocator的bean
@Configuration
public class GateFilterConfiguration {

    @Bean
    public RouteLocator routeLocator(RouteLocatorBuilder builder){  
        return builder.routes()
                .route(r->r.path("/provider/**")    //创建router
                        .filters(f->f.filter(new MyGateWayFilter()))  //添加自定义的filter
                        .uri("lb://CLOUD-PROVIDER-SERVICE")).build(); //添加uri
    }
}
```
- 	* 2、自定义MyGatewayFilterFactory，再添加到配置文件

``` java
//第一步
@Component
@Slf4j
public class MyGatewayFilterFactory extends AbstractGatewayFilterFactory<MyGatewayFilterFactory.Config> {

    /**
     * 定义可以再yaml中声明的属性变量
     */
    private static final String args1 = "args1";
    private static final String args2 = "args2";


    public MyGatewayFilterFactory(){
        super(Config.class);  // 将自定义的config传给父类
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList(args1, args2);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return ((exchange, chain) -> {
            exchange.getAttributes().put("request_begin_time",System.currentTimeMillis());  //pre 过滤
            log.info("config-type: "+config.getArgs1());
            // 在then方法里的，相当于aop中的后置通知
            return chain.filter(exchange).then(
                    Mono.fromRunnable(()->{
                        Long beginTime = exchange.getAttribute("request_begin_time");
                        //exchange.getResponse().getHeaders().get
                        log.info(exchange.getRequest().getURI().getRawPath() + ": " + (System.currentTimeMillis() - beginTime) + "ms");
                    })
            );


        });
        //return new MyGateWayFilter();
    }

    /**
     * 自定义的config类，用来设置传入的参数
     */
    public static class Config {
        private String args1;
        private String args2;
        public String getArgs1() {
            return args1;
        }
        public void setArgs1(String args1) {
            this.args1 = args1;
        }
        public String getArgs2() {
            return args2;
        }
        public void setArgs2(String args2) {
            this.args2 = args2;
        }
    }
}

//第二步
spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      routes:
        - id: payment_routh #payment_routh    #路由的ID，没有固定规则但要求唯一，简易配合服务名
          uri: lb://CLOUD-PROVIDER-SERVICE         #匹配后提供服务的路由地址
          predicates:
            - Path=/provider/**          #断言，路径相匹配的进行路由
          filters:         #第一种方式：
            - name: My     #filter的名字默认是MyGatewayFilterFactory的前缀，否则显示找不到
              args:
                op: Myop
                type: Mytype
#         filters:             
#              - My=Myop,Mytype   #第二种方式
```

部分参考： https://blog.csdn.net/forezp/article/details/85057268
