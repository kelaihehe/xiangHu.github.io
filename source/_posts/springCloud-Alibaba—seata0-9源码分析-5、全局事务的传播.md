---
title: springCloud Alibaba—seata0.9源码分析 5、全局事务的传播
date: 2020-07-15 22:07:00
tags:
  - SpringCloud Alibaba
  - seata
summary: seata0.9版本源码解析：seata的全局事务传播
categories: SpringCloud Alibaba
---

## 同一线程中的传播 ##
* RootContext中维护了ThreadLocal变量，来保存XID

## 跨服务的传播 ##
* 调用端：
	* SeataFeignClientAutoConfiguration
		* SeataBeanPostProcessor会对每个FeignClient的bean包装成SeataFeignClient或SeataLoadBalancerFeignClient
		* 它们的execute方法都如下：将rootContext的Xid添加到请求头中

``` java
public Response execute(Request request, Options options) throws IOException {
    Request modifiedRequest = this.getModifyRequest(request);  //添加Xid到请求头中
    return this.delegate.execute(modifiedRequest, options);
}


private Request getModifyRequest(Request request) {
    String xid = RootContext.getXID();
    if (StringUtils.isEmpty(xid)) {
        return request;
    } else {
        Map<String, Collection<String>> headers = new HashMap(16);
        headers.putAll(request.headers());
        List<String> fescarXid = new ArrayList();
        fescarXid.add(xid);
        headers.put("TX_XID", fescarXid);
        return Request.create(request.method(), request.url(), headers, request.body(), request.charset());
    }
}
```
- - SeataRestTemplateAutoConfiguration
	* SeataRestTemplateInterceptor 

``` java
public class SeataRestTemplateInterceptor implements ClientHttpRequestInterceptor {
    public SeataRestTemplateInterceptor() {
    }

    public ClientHttpResponse intercept(HttpRequest httpRequest, byte[] bytes, ClientHttpRequestExecution clientHttpRequestExecution) throws IOException {
        HttpRequestWrapper requestWrapper = new HttpRequestWrapper(httpRequest);
        String xid = RootContext.getXID();
        if (!StringUtils.isEmpty(xid)) {
            requestWrapper.getHeaders().add("TX_XID", xid);
        }

        return clientHttpRequestExecution.execute(requestWrapper, bytes);
    }
}
```

* 服务端：
	* SeataHandlerInterceptorConfiguration
		* SeataHandlerInterceptor的preHandle对请求进行拦截，将获取的Xid绑定到RootContext中

``` java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    String xid = RootContext.getXID();
    String rpcXid = request.getHeader("TX_XID");
    if (log.isDebugEnabled()) {
        log.debug("xid in RootContext {} xid in RpcContext {}", xid, rpcXid);
    }


    if (xid == null && rpcXid != null) {
        RootContext.bind(rpcXid);
        if (log.isDebugEnabled()) {
            log.debug("bind {} to RootContext", rpcXid);
        }
    }
    return true;
}
```



