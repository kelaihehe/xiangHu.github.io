---
title: springCloud Alibaba—seata0.9源码分析 1、自动配置
date: 2020-07-15 20:09:21
tags:
  - SpringCloud Alibaba
  - seata
summary: seata0.9版本源码解析：seata的自动配置
categories: SpringCloud Alibaba
---

seata的自动配置类有：
* GlobalTransactionAutoConfiguration
* SeataFeignClientAutoConfiguration：在服务调用前添加请求头Xid   
* SeataRestTemplateAutoConfiguration：在服务调用前添加请求头Xid  
* SeataHystrixAutoConfiguration
* SeataHandlerInterceptorConfiguration  ：对请求进行拦截    



1、GlobalTransactionAutoConfiguration

``` java
@Configuration
@EnableConfigurationProperties({SeataProperties.class})
public class GlobalTransactionAutoConfiguration {
    private final ApplicationContext applicationContext;
    private final SeataProperties seataProperties;

    public GlobalTransactionAutoConfiguration(ApplicationContext applicationContext, SeataProperties seataProperties) {
        this.applicationContext = applicationContext;
        this.seataProperties = seataProperties;
    }

    @Bean 
    public GlobalTransactionScanner globalTransactionScanner() {  //注入GlobalTransactionScanner
        String applicationName = this.applicationContext.getEnvironment().getProperty("spring.application.name");
        String txServiceGroup = this.seataProperties.getTxServiceGroup();
        if (StringUtils.isEmpty(txServiceGroup)) {
            txServiceGroup = applicationName + "-fescar-service-group";
            this.seataProperties.setTxServiceGroup(txServiceGroup);
        }

        return new GlobalTransactionScanner(applicationName, txServiceGroup);
    }
}
```
* GlobalTransactionScanner 
	* 1、wrapIfNecessary：创建出拦截器对@GlobalTransactional注解的方法进行aop

``` java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {  
    if (this.disableGlobalTransaction) {
        return bean;
    } else {
        try {
            synchronized(PROXYED_SET) {
                if (PROXYED_SET.contains(beanName)) {
                    return bean;
                } else {
                    this.interceptor = null;
                    if (TCCBeanParserUtils.isTccAutoProxy(bean, beanName, this.applicationContext)) {
                        this.interceptor = new TccActionInterceptor(TCCBeanParserUtils.getRemotingDesc(beanName));
                    } else {
                        Class<?> serviceInterface = SpringProxyUtils.findTargetClass(bean);
                        Class<?>[] interfacesIfJdk = SpringProxyUtils.findInterfaces(bean);
                        if (!this.existsAnnotation(new Class[]{serviceInterface}) && !this.existsAnnotation(interfacesIfJdk)) {
                            return bean;
                        }
                        if (this.interceptor == null) {
                            this.interceptor = new GlobalTransactionalInterceptor(this.failureHandlerHook);  //创建一个拦截器
                        }
                    }
                //...
                }
            }
        } catch (Exception var13) {
            throw new RuntimeException(var13);
        }
    }
}
```
-	- 2、afterPropertiesSet  ：对TM和RM进行初始化

``` java
public void afterPropertiesSet() {    //bean属性注入后调用
    if (this.disableGlobalTransaction) {
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Global transaction is disabled.");
        }
    } else {
        this.initClient();  //如果支持全局事务，则初始化seataClient
    }
}

private void initClient() {
    if (!StringUtils.isNullOrEmpty(this.applicationId) && !StringUtils.isNullOrEmpty(this.txServiceGroup)) {
        TMClient.init(this.applicationId, this.txServiceGroup);  //初始化TM
        RMClient.init(this.applicationId, this.txServiceGroup);  //初始化RM
        this.registerSpringShutdownHook();
    } else {
        throw new IllegalArgumentException("applicationId: " + this.applicationId + ", txServiceGroup: " + this.txServiceGroup);
    }
}
```
-   * RMClient

``` java
public static void init(String applicationId, String transactionServiceGroup) {
    RmRpcClient rmRpcClient = RmRpcClient.getInstance(applicationId, transactionServiceGroup);  //获取单例
    rmRpcClient.setResourceManager(DefaultResourceManager.get());  //获取RM单例 ，并设置RM属性
    rmRpcClient.setClientMessageListener(new RmMessageListener(DefaultRMHandler.get()));  //设置Client监听器
    rmRpcClient.init();
}
```
-	- DefaultRMHandler负责处理TC发送的请求

部分参考：
- 1、https://www.cnblogs.com/lay2017/p/12485081.html
- 2、https://seata.io/zh-cn/docs/dev/mode/at-mode.html