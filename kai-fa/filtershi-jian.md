

## **2 Filter实践**

### **2.1 AddReqquestHeader GatewayFilter Factory**

#### **2.1.1导入依赖**

```
1.<dependency>  
2.    <groupId>org.springframework.cloud</groupId>  
3.    <artifactId>spring-cloud-starter-gateway</artifactId>  
4.</dependency>  
```

#### **2.1.2 修改配置文件，加入以下的配置信息**

```
1.server:  
2.  port: 8081  
3.spring:  
4.  profiles:  
5.    active: add_request_header_route  
6.  
7.---  
8.spring:  
9.  cloud:  
10.    gateway:  
11.      routes:  
12.      - id: add_request_header_route  
13.        uri: http://httpbin.org:80/get  
14.        filters:  
15.        - AddRequestHeader=X-Request-Foo, Bar  
16.        predicates:  
17.        - After=2017-01-20T17:42:47.789-07:00[America/Denver]  
18.  profiles: add_request_header_route  
```

#### **2.1.3 测试**

启动工程，通过curl命令来模拟请求：

```
curl localhost:8081
```

最终显示了从 http://httpbin.org:80/get得到了请求，响应如下：

```
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Connection": "close",
    "Forwarded": "proto=http;host=\"localhost:8081\";for=\"0:0:0:0:0:0:0:1:56248\"",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.58.0",
    "X-Forwarded-Host": "localhost:8081",
    "X-Request-Foo": "Bar"
  },
  "origin": "0:0:0:0:0:0:0:1, 210.22.21.66",
  "url": "http://localhost:8081/get"
}
```

可以上面的响应可知，确实在请求头中加入了X-Request-Foo这样的一个请求头，在配置文件中配置的AddRequestHeader过滤器工厂生效

### **2.2 RewritePath GatewayFilter Factory**

在Nginx服务启中有一个非常强大的功能就是重写路径，Spring Cloud Gateway默认也提供了这样的功能，这个功能是Zuul没有的。

#### **2.2.1在配置文件中加上以下的配置**

```

```

上面的配置中，所有的/foo/\*\*开始的路径都会命中配置的router，并执行过滤器的逻辑。

在本案例中配置了RewritePath过滤器工厂，此工厂将/foo/\(?.\*\)重写为{segment}，然后转发到https://blog.csdn.net。

比如在网页上请求localhost:8081/foo/forezp，此时会将请求转发到https://blog.csdn.net/forezp的页面，比如在网页上请求localhost:8081/foo/forezp/1，页面显示404，就是因为不存在https://blog.csdn.net/forezp/1这个页面。

### **2.3 自定义过滤器**

在spring Cloud Gateway中，过滤器需要实现GatewayFilter和Ordered2个接口。写一个RequestTimeFilter，代码如下：

1.publicclassRequestTimeFilterimplementsGatewayFilter,Ordered{

2.

3.privatestaticfinalLoglog=LogFactory.getLog\(GatewayFilter.class\);

4.privatestaticfinalStringREQUEST\_TIME\_BEGIN="requestTimeBegin";

5.

6.@Override

7.publicMono**&lt;Void&gt;**filter\(ServerWebExchangeexchange,GatewayFilterChainchain\){

8.

9.exchange.getAttributes\(\).put\(REQUEST\_TIME\_BEGIN,System.currentTimeMillis\(\)\);

10.returnchain.filter\(exchange\).then\(

11.Mono.fromRunnable\(\(\)-**&gt;**{

12.LongstartTime=exchange.getAttribute\(REQUEST\_TIME\_BEGIN\);

13.if\(startTime!=null\){

14.log.info\(exchange.getRequest\(\).getURI\(\).getRawPath\(\)+":"+\(System.currentTimeMillis\(\)-startTime\)+"ms"\);

15.}

16.}\)

17.\);

18.

19.}

20.

21.@Override

22.publicintgetOrder\(\){

23.return0;

24.}

25.}

l在上面的代码中，Ordered中的int getOrder\(\)方法是来给过滤器设定优先级别的，值越大则优先级越低。

lfilterI\(exchange,chain\)方法，在该方法中，先记录了请求的开始时间，并保存在ServerWebExchange中，此处是一个“pre”类型的过滤器

lchain.filter的内部类中的run\(\)方法中相当于”post”过滤器，在此处打印了请求所消耗的时间

将该过滤器注册到router中

1.@Bean

2.publicRouteLocatorcustomerRouteLocator\(RouteLocatorBuilderbuilder\){

3.//@formatter:off

4.returnbuilder.routes\(\)

5..route\(r-**&gt;**r.path\("/customer/\*\*"\)

6..filters\(f-**&gt;**f.filter\(newRequestTimeFilter\(\)\)

7..addResponseHeader\("X-Response-Default-Foo","Default-Bar"\)\)

8..uri\("http://httpbin.org:80/get"\)

9..order\(0\)

10..id\("customer\_filter\_router"\)

11.\)

12..build\(\);

13.//@formatter:on

14.}

重启程序，通过curl命令模拟请求：

 curl localhost:8081/customer/123

在程序的控制台输出一下的请求信息的日志：

2018-11-16 15:02:20.177  INFO 20488 --- \[ctor-http-nio-3\] o.s.cloud.gateway.filter.GatewayFilter   : /customer/123: 152ms

### **2.4 global filter**

Spring Cloud Gateway根据作用范围划分为GatewayFilter和GlobalFilter，二者区别如下：

lGatewayFilter : 需要通过spring.cloud.routes.filters 配置在具体路由下，只作用在当前路由上或通过spring.cloud.default-filters配置在全局，作用在所有路由上

lGlobalFilter : 全局过滤器，不需要在配置文件中配置，作用在所有的路由上，最终通过GatewayFilterAdapter包装成GatewayFilterChain可识别的过滤器，它为请求业务以及路由的URI转换为真实业务服务的请求地址的核心过滤器，不需要配置，系统初始化时加载，并作用在每个路由上。

在下面的案例中将讲述如何编写自己GlobalFilter，该GlobalFilter会校验请求中是否包含了请求参数“token”，如何不包含请求参数“token”则不转发路由，否则执行正常的逻辑。代码如下：

1.**publicclass**TokenFilter**implements**GlobalFilter,Ordered{

2.

3.Loggerlogger=LoggerFactory.getLogger\(TokenFilter.**class**\);

4.@Override

5.**public**Mono&lt;Void&gt;filter\(ServerWebExchangeexchange,GatewayFilterChainchain\){

6.Stringtoken=exchange.getRequest\(\).getQueryParams\(\).getFirst\("token"\);

7.**if**\(token==**null**\|\|token.isEmpty\(\)\){

8.logger.info\("tokenisempty..."\);

9.exchange.getResponse\(\).setStatusCode\(HttpStatus.UNAUTHORIZED\);

10.**return**exchange.getResponse\(\).setComplete\(\);

11.}

12.**return**chain.filter\(exchange\);

13.}

14.

15.@Override

16.**publicint**getOrder\(\){

17.**return**-100;

18.}

19.}



l在上面的TokenFilter需要实现GlobalFilter和Ordered接口，这和实现GatewayFilter很类似。

l然后根据ServerWebExchange获取ServerHttpRequest，然后根据ServerHttpRequest中是否含有参数token，如果没有则完成请求，终止转发，否则执行正常的逻辑。

然后需要将TokenFilter在工程的启动类中注入到Spring Ioc容器中，代码如下：



1.@Bean

2.**public**TokenFiltertokenFilter\(\){

3.**returnnew**TokenFilter\(\);

4.}



启动工程，使用curl命令请求：

 curl localhost:8081/customer/123

可以看到请没有被转发，请求被终止，并在控制台打印了如下日志：

2018-11-16 15:30:13.543  INFO 19372 --- \[ctor-http-nio-2\] gateway.TokenFilter

上面的日志显示了请求进入了没有传“token”的逻辑。

### **2.5源码下载**

[示例代码-GitHub](https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-gateway-predicate)

���?�$U�&gt;

