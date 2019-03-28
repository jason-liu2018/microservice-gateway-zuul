# Zuul 过滤器类型
Zuul大部分功能都是通过过滤器来实现的。
Zuul中定义了四种标准过滤器类型，这些过滤器类型对应于请求的典型生命周期。
- PRE：
这种过滤器在请求被路由之前调用。我们可利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等。

- ROUTING：
这种过滤器将请求路由到微服务。这种过滤器用于构建发送给微服务的请求，并使用Apache HttpClient或Netfilx Ribbon请求微服务。

- POST：
这种过滤器在路由到微服务以后执行。这种过滤器可用来为响应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等。

- ERROR：
在其他阶段发生错误时执行该过滤器。

除了默认的过滤器类型，Zuul还允许我们创建自定义的过滤器类型 extends ZuulFilter。

# 禁用Zuul过滤器
Spring Cloud默认为Zuul编写并启用了一些过滤器，例如DebugFilter、FormBodyWrapperFilter、PreDecorationFilter等。
这些过滤器都存放在spring-cloud-netflix-core这个Jar包的org.springframework.cloud.netflix.zuul.filters包中。
  
一些场景下，我们想要禁用掉部分过滤器，此时该怎么办呢？
答案非常简单，只需设置zuul.<SimpleClassName>.<filterType>.disable=true ，即可禁用SimpleClassName所对应的过滤器。
以过滤器org.springframework.cloud.netflix.zuul.filters.post.SendResponseFilter为例，只需设置zuul.SendResponseFilter.post.disable=true ，即可禁用该过滤器。

# Spring Cloud内置的Zuul过滤器

可将@EnableZuulProxy简单理解为@EnableZuulServer的增强版

## @EnableZuulServer过滤器

### RequestContext
用于在过滤器之间传递消息。它的数据保存在每个请求的ThreadLocal中。
它用于存储请求路由到哪里、错误、HttpServletRequest、HttpServletResponse都存储在RequestContext中。
RequestContext扩展了ConcurrentHashMap，所以，任何数据都可以存储在上下文中.


### pre类型过滤器
- ServletDetectionFilter：
该过滤器用于检查请求是否通过Spring Dispatcher。检查后，通过isDispatcherServletRequest设置布尔值。

- FormBodyWrapperFilter：
解析表单数据，并为请求重新编码。

- DebugFilter：
顾名思义，调试用的过滤器，可以通过zuul.debug.request=true ，或在请求时，加上debug=true的参数，例如$ZUUL_HOST:ZUUL_PORT/path?debug=true 开启该过滤器。
这样，该过滤器就会把RequestContext.setDebugRouting() 、RequestContext.setDebugRequest() 设为true。

### route类型过滤器
- SendForwardFilter：
该过滤器使用Servlet RequestDispatcher转发请求，转发位置存储在RequestContext.getCurrentContext().get("forward.to") 中。
可以将路由设置成：
```aidl
zuul:
  routes:
    abc: 
      path: /abc/**
      url: forward:/abc
```
然后访问$ZUUL_HOST:ZUUL_PORT/abc ，观察该过滤器的执行过程。

### post类型过滤器
- SendResponseFilter：
将Zuul所代理的微服务的的响应写入当前响应。

### error类型过滤器
- SendErrorFilter：
如果RequestContext.getThrowable() 不为null，那么默认就会转发到/error，也可以设置error.path属性修改默认的转发路径。

## @EnableZuulProxy过滤器
如果使用注解@EnableZuulProxy，那么除上述过滤器之外，Spring Cloud还会安装以下过滤器

### pre类型过滤器
- PreDecorationFilter：
该过滤器根据提供的RouteLocator确定路由到的地址，以及怎样去路由。该路由器也可为后端请求设置各种代理相关的header。

### route类型过滤器
- RibbonRoutingFilter：
该过滤器使用Ribbon，Hystrix和可插拔的HTTP客户端发送请求。serviceId在RequestContext.getCurrentContext().get("serviceId") 中。
该过滤器可使用不同的HTTP客户端，例如
   -- Apache HttpClient：默认的HTTP客户端
   -- Squareup OkHttpClient v3：如需使用该客户端，需保证com.squareup.okhttp3的依赖在classpath中，并设置ribbon.okhttp.enabled = true 。
   -- Netflix Ribbon HTTP client：设置ribbon.restclient.enabled = true 即可启用该HTTP客户端。
   需要注意的是，该客户端有一定限制，例如不支持PATCH方法，另外，它有内置的重试机制。
   
- SimpleHostRoutingFilter：
该过滤器通过Apache HttpClient向指定的URL发送请求。URL在RequestContext.getRouteHost() 中。

# Zuul的高可用
Zuul的高可用非常关键，因为外部请求到后端微服务的流量都会经过Zuul。
故而在生产环境中，我们一般都需要部署高可用的Zuul以避免单点故障。

## Zuul客户端也注册到了Eureka Server上
这种情况下，Zuul的高可用非常简单，只需将多个Zuul节点注册到Eureka Server上，就可实现Zuul的高可用。
此时，Zuul的高可用与其他微服务的高可用没什么区别。
Zuul客户端会自动从Eureka Server中查询Zuul Server的列表，并使用Ribbon负载均衡地请求Zuul集群。
这种场景一般用于Sidecar。

## Zuul客户端未注册到Eureka Server上
现实中，这种场景往往更常见，例如，Zuul客户端是一个手机APP——我们不可能让所有的手机终端都注册到Eureka Server上。
这种情况下，我们可借助一个额外的负载均衡器来实现Zuul的高可用，例如Nginx、HAProxy、F5等

# 快速定位Zuul的性能瓶颈
一次请求，会经过若干过滤器，如何查看每个过滤器执行的耗时呢？只需开启Zuul的Debug能力即可。
```aidl
zuul:
  include-debug-header: true
management:
  endpoints:
    web:
      exposure:
        include: '*'
```
只需在访问Zuul时，添加 ?debug=true 即可对Zuul进行Debug.
例如监控路径ZUUL_HOST:ZUUL_PORT/SOME_PATH 经过了哪些过滤器，性能瓶颈出现在哪个过滤器，只需构造 ZUUL_HOST:ZUUL_PORT/SOME_PATH?debug=true 即可。

请求后，访问 ZUUL_HOST:ZUUL_PORT/actuator/httptrace ，即可看到类似如下的结果：
```aidl
"X-Zuul-Debug-Header": ["[[[Filter pre 5 PreDecorationFilter]]][[[Filter {PreDecorationFilter TYPE:pre ORDER:5} Execution time = 1ms]]][[[{PreDecorationFilter} added retryable=false]]][[[{PreDecorationFilter} added ignoredHeaders=[authorization, set-cookie, cookie]]]][[[{PreDecorationFilter} added originResponseHeaders=[com.netflix.util.Pair@d68cf7e9]]]][[[{PreDecorationFilter} added zuulRequestHeaders={x-forwarded-host=localhost:8040, x-forwarded-proto=http, x-forwarded-prefix=/microservice-provider-user, x-forwarded-port=8040, x-forwarded-for=0:0:0:0:0:0:0:1}]]][[[{PreDecorationFilter} added requestURI=/users/1]]][[[{PreDecorationFilter} added proxy=microservice-provider-user]]][[[{PreDecorationFilter} changed executedFilters=ServletDetectionFilter[SUCCESS][0ms], Servlet30WrapperFilter[SUCCESS][0ms], DebugFilter[SUCCESS][0ms], PreDecorationFilter[SUCCESS][1ms]]]][[[{PreDecorationFilter} added serviceId=microservice-provider-user]]][[[Invoking {route} type filters]]][[[Filter route 10 RibbonRoutingFilter]]][[[Filter {RibbonRoutingFilter TYPE:route ORDER:10} Execution time = 9ms]]][[[{RibbonRoutingFilter} changed originResponseHeaders=[com.netflix.util.Pair@d68cf7e9, com.netflix.util.Pair@694b84a6, com.netflix.util.Pair@a4baea16, com.netflix.util.Pair@99438774]]]][[[{RibbonRoutingFilter} added responseDataStream=org.apache.http.conn.EofSensorInputStream@1145027a]]][[[{RibbonRoutingFilter} added zuulResponseHeaders=[com.netflix.util.Pair@694b84a6, com.netflix.util.Pair@99438774]]]][[[{RibbonRoutingFilter} added responseStatusCode=200]]][[[{RibbonRoutingFilter} added responseGZipped=false]]][[[{RibbonRoutingFilter} added ribbonResponse=org.springframework.cloud.netflix.ribbon.apache.RibbonApacheHttpResponse@5e2ce130]]][[[{RibbonRoutingFilter} changed executedFilters=ServletDetectionFilter[SUCCESS][0ms], Servlet30WrapperFilter[SUCCESS][0ms], DebugFilter[SUCCESS][0ms], PreDecorationFilter[SUCCESS][1ms], RibbonRoutingFilter[SUCCESS][9ms]]]][[[{RibbonRoutingFilter} added zuulResponse=org.springframework.cloud.netflix.ribbon.RibbonHttpResponse@1e0eabde]]][[[Filter route 100 SimpleHostRoutingFilter]]][[[Filter route 500 SendForwardFilter]]][[[Invoking {post} type filters]]][[[Filter post 1000 SendResponseFilter]]]"],
```

由结果可知，该端点依次打印了请求经过了哪些过滤器、每个过滤器的耗时。简单分析一下，就能了解Zuul的性能瓶颈了。


## 开启默认Debug
经过上面的配置，已实现对Zuul的Debug，但每次都要添加一个debug=true 的小尾巴，也是挺烦的，
如果不想添加，而想让Zuul默认就对请求开启Debug，添加如下配置即可：
```aidl
zuul:
  debug:
    request: true
```

这样，即使不添加debug=true ，Zuul也会Debug。

> 相关源码其实比较简单，就一个类： org.springframework.cloud.netflix.zuul.filters.pre.DebugFilter