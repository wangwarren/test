## **2 Filter实践**

### **2.1 AddReqquestHeader GatewayFilter Factory**

#### **2.1.1导入依赖**

```
1.<dependency>  
2.    <groupId>org.springframework.cloud</groupId>  
3.    <artifactId>spring-cloud-starter-gateway</artifactId>  
4.</dependency>
```

#### **2.1.2 修改配置文件，加入以下的配置信息**

```
1.server:  
2.  port: 8081  
3.spring:  
4.  profiles:  
5.    active: add_request_header_route  
6.  
7.---  
8.spring:  
9.  cloud:  
10.    gateway:  
11.      routes:  
12.      - id: add_request_header_route  
13.        uri: http://httpbin.org:80/get  
14.        filters:  
15.        - AddRequestHeader=X-Request-Foo, Bar  
16.        predicates:  
17.        - After=2017-01-20T17:42:47.789-07:00[America/Denver]  
18.  profiles: add_request_header_route
```

#### **2.1.3 测试**

启动工程，通过curl命令来模拟请求：

```
curl localhost:8081
```

最终显示了从 [http://httpbin.org:80/get得到了请求，响应如下：](http://httpbin.org:80/get得到了请求，响应如下：)

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
1.spring:  
2.  profiles:  
3.    active: rewritepath_route  
4.---  
5.spring:  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.      - id: rewritepath_route  
10.        uri: https://blog.csdn.net  
11.        predicates:  
12.        - Path=/foo/**  
13.        filters:  
14.        - RewritePath=/foo/(?<segment>.*), /$\{segment}  
15.  profiles: rewritepath_route
```

上面的配置中，所有的/foo/\*\*开始的路径都会命中配置的router，并执行过滤器的逻辑。

在本案例中配置了RewritePath过滤器工厂，此工厂将/foo/\(?.\*\)重写为{segment}，然后转发到[https://blog.csdn.net。](https://blog.csdn.net。)

比如在网页上请求localhost:8081/foo/forezp，此时会将请求转发到[https://blog.csdn.net/forezp的页面，比如在网页上请求localhost:8081/foo/forezp/1，页面显示404，就是因为不存在https://blog.csdn.net/forezp/1这个页面。](https://blog.csdn.net/forezp的页面，比如在网页上请求localhost:8081/foo/forezp/1，页面显示404，就是因为不存在https://blog.csdn.net/forezp/1这个页面。)

### **2.3 自定义过滤器**

在spring Cloud Gateway中，过滤器需要实现GatewayFilter和Ordered2个接口。写一个RequestTimeFilter，代码如下：

```
1.public class RequestTimeFilter implements GatewayFilter, Ordered {  
2.  
3.    private static final Log log = LogFactory.getLog(GatewayFilter.class);  
4.    private static final String REQUEST_TIME_BEGIN = "requestTimeBegin";  
5.  
6.    @Override  
7.    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {  
8.  
9.        exchange.getAttributes().put(REQUEST_TIME_BEGIN, System.currentTimeMillis());  
10.        return chain.filter(exchange).then(  
11.                Mono.fromRunnable(() -> {  
12.                    Long startTime = exchange.getAttribute(REQUEST_TIME_BEGIN);  
13.                    if (startTime != null) {  
14.                        log.info(exchange.getRequest().getURI().getRawPath() + ": " + (System.currentTimeMillis() - startTime) + "ms");  
15.                    }  
16.                })  
17.        );  
18.  
19.    }  
20.  
21.    @Override  
22.    public int getOrder() {  
23.        return 0;  
24.    }  
25.}
```

* 在上面的代码中，Ordered中的int getOrder\(\)方法是来给过滤器设定优先级别的，值越大则优先级越低。

* filterI\(exchange,chain\)方法，在该方法中，先记录了请求的开始时间，并保存在ServerWebExchange中，此处是一个“pre”类型的过滤器

* chain.filter的内部类中的run\(\)方法中相当于”post”过滤器，在此处打印了请求所消耗的时间

将该过滤器注册到router中

```
1.@Bean  
2.    public RouteLocator customerRouteLocator(RouteLocatorBuilder builder) {  
3.        // @formatter:off  
4.        return builder.routes()  
5.                .route(r -> r.path("/customer/**")  
6.                        .filters(f -> f.filter(new RequestTimeFilter())  
7.                                .addResponseHeader("X-Response-Default-Foo", "Default-Bar"))  
8.                        .uri("http://httpbin.org:80/get")  
9.                        .order(0)  
10.                        .id("customer_filter_router")  
11.                )  
12.                .build();  
13.        // @formatter:on  
14.    }
```

重启程序，通过curl命令模拟请求：

```
curl localhost:8081/customer/123
```

在程序的控制台输出一下的请求信息的日志：

```
2018-11-16 15:02:20.177  INFO 20488 --- [ctor-http-nio-3] o.s.cloud.gateway.filter.GatewayFilter   : /customer/123: 152ms
```

### **2.4 global filter**

Spring Cloud Gateway根据作用范围划分为GatewayFilter和GlobalFilter，二者区别如下：

* GatewayFilter : 需要通过spring.cloud.routes.filters 配置在具体路由下，只作用在当前路由上或通过spring.cloud.default-filters配置在全局，作用在所有路由上

* GlobalFilter : 全局过滤器，不需要在配置文件中配置，作用在所有的路由上，最终通过GatewayFilterAdapter包装成GatewayFilterChain可识别的过滤器，它为请求业务以及路由的URI转换为真实业务服务的请求地址的核心过滤器，不需要配置，系统初始化时加载，并作用在每个路由上。

在下面的案例中将讲述如何编写自己GlobalFilter，该GlobalFilter会校验请求中是否包含了请求参数“token”，如何不包含请求参数“token”则不转发路由，否则执行正常的逻辑。代码如下：

```
1.public class TokenFilter implements GlobalFilter, Ordered {  
2.  
3.    Logger logger=LoggerFactory.getLogger( TokenFilter.class );  
4.    @Override  
5.    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {  
6.        String token = exchange.getRequest().getQueryParams().getFirst("token");  
7.        if (token == null || token.isEmpty()) {  
8.            logger.info( "token is empty..." );  
9.            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);  
10.            return exchange.getResponse().setComplete();  
11.        }  
12.        return chain.filter(exchange);  
13.    }  
14.  
15.    @Override  
16.    public int getOrder() {  
17.        return -100;  
18.    }  
19.}
```

* 在上面的TokenFilter需要实现GlobalFilter和Ordered接口，这和实现GatewayFilter很类似。

* 然后根据ServerWebExchange获取ServerHttpRequest，然后根据ServerHttpRequest中是否含有参数token，如果没有则完成请求，终止转发，否则执行正常的逻辑。

然后需要将TokenFilter在工程的启动类中注入到Spring Ioc容器中，代码如下：

```
1.@Bean  
2.public TokenFilter tokenFilter(){  
3.        return new TokenFilter();  
4.}
```

启动工程，使用curl命令请求：

```
curl localhost:8081/customer/123
```

可以看到请没有被转发，请求被终止，并在控制台打印了如下日志：

```
2018-11-16 15:30:13.543  INFO 19372 --- [ctor-http-nio-2] gateway.TokenFilter
```

上面的日志显示了请求进入了没有传“token”的逻辑。

### **2.5源码下载**

[示例代码-GitHub](https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-gateway-predicate)

## **3.高级应用**

### **3.1 熔断**

1.Spring Cloud Gateway 也可以利用 Hystrix 的熔断特性，在流量过大时进行服务降级，同样我们还是首先给项目添加上依赖。

1.&lt;dependency&gt;

2.&lt;groupId&gt;org.springframework.cloud&lt;/groupId&gt;

3.&lt;artifactId&gt;spring-cloud-starter-netflix-hystrix&lt;/artifactId&gt;

4.&lt;/dependency&gt;

配置示例

1.spring:

2.cloud:

3.gateway:

4.routes:

5.-id:hystrix\_route

6.uri:http://example.org

7.filters:

8.-Hystrix=myCommandName

配置后，gateway 将使用 myCommandName 作为名称生成 HystrixCommand 对象来进行熔断管理。

2.如果想添加熔断后的回调内容，需要在添加一些配置

1.spring:

2.cloud:

3.gateway:

4.routes:

5.-id:hystrix\_route

6.uri:lb://spring-cloud-producer

7.predicates:

8.-Path=/consumingserviceendpoint

9.filters:

10.-name:Hystrix

11.args:

12.name:fallbackcmd

13.fallbackUri:forward:/incaseoffailureusethis

fallbackUri: forward:/incaseoffailureusethis配置了 fallback 时要会调的路径，当调用 Hystrix 的 fallback 被调用时，请求将转发到/incaseoffailureuset这个 URI。

#### **源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter14)





### **3.2 限流**

在Spring Cloud Gateway中，有Filter过滤器，因此可以在“pre”类型的Filter中自行实现上述三种过滤器。但是限流作为网关最基本的功能，Spring Cloud Gateway官方就提供了RequestRateLimiterGatewayFilterFactory这个类，适用Redis和lua脚本实现了令牌桶的方式。

#### **3.2.1 引入依赖和redis的reactive依赖**

1.&lt;dependency&gt;

2.&lt;groupId&gt;org.springframework.cloud&lt;/groupId&gt;

3.&lt;artifactId&gt;spring-cloud-starter-gateway&lt;/artifactId&gt;

4.&lt;/dependency&gt;

5.

6.&lt;dependency&gt;

7.&lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;

8.&lt;artifatId&gt;spring-boot-starter-data-redis-reactive&lt;/artifactId&gt;

9.&lt;/dependency&gt;

#### **3.2.2 修改配置文件**

1.server:

2.port:8081

3.spring:

4.cloud:

5.gateway:

6.routes:

7.-id:limit\_route

8.uri:http://httpbin.org:80/get

9.predicates:

10.-After=2017-01-20T17:42:47.789-07:00\[America/Denver\]

11.filters:

12.-name:RequestRateLimiter

13.args:

14.key-resolver:'\#{@hostAddrKeyResolver}'

15.redis-rate-limiter.replenishRate:1

16.redis-rate-limiter.burstCapacity:3

17.application:

18.name:gateway-limiter

19.redis:

20.host:localhost

21.port:6379

22.database:0

在上面的配置文件，指定程序的端口为8081，配置了 redis的信息，并配置了RequestRateLimiter的限流过滤器，该过滤器需要配置三个参数：

lburstCapacity，令牌桶总容量。

lreplenishRate，令牌桶每秒填充平均速率。

lkey-resolver，用于限流的键的解析器的 Bean 对象的名字。它使用 SpEL 表达式根据\#{@beanName}从 Spring 容器中获取 Bean 对象。

#### **3.2.3 实现KeyResolver**

根据Hostname进行限流，则需要用hostAddress去判断

1.**publicclass**HostAddrKeyResolver**implements**KeyResolver{

2.

3.@Override

4.**public**Mono&lt;String&gt;resolve\(ServerWebExchangeexchange\){

5.**return**Mono.just\(exchange.getRequest\(\).getRemoteAddress\(\).getAddress\(\).getHostAddress\(\)\);

6.}

7.}

8.@Bean

9.**public**HostAddrKeyResolverhostAddrKeyResolver\(\){

10.**returnnew**HostAddrKeyResolver\(\);

11.}

可以根据uri去限流，这时KeyResolver代码如下

1.**publicclass**UriKeyResolver**implements**KeyResolver{

2.

3.@Override

4.**public**Mono&lt;String&gt;resolve\(ServerWebExchangeexchange\){

5.**return**Mono.just\(exchange.getRequest\(\).getURI\(\).getPath\(\)\);

6.}

7.

8.}

9.

10.@Bean

11.**public**UriKeyResolveruriKeyResolver\(\){

12.**returnnew**UriKeyResolver\(\);

13.}

以用户的维度去限流：

1.@Bean

2.KeyResolveruserKeyResolver\(\){

3.**return**exchange-&gt;Mono.just\(exchange.getRequest\(\).getQueryParams\(\).getFirst\("user"\)\);

4.}

#### **3.2.4 源码下载**

[示例源码-GitHub](https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-gateway-limiter)

### **3.3 重试**

RetryGatewayFilter 是 Spring Cloud Gateway 对请求重试提供的一个 GatewayFilter Factory。

配置示例

1.spring:

2.cloud:

3.gateway:

4.routes:

5.-id:retry\_test

6.uri:lb://spring-cloud-producer

7.predicates:

8.-Path=/retry

9.filters:

10.-name:Retry

11.args:

12.retries:3

13.statuses:BAD\_GATEWAY

Retry GatewayFilter 通过这四个参数来控制重试机制： retries, statuses, methods, 和 series。

lretries：重试次数，默认值是 3 次

lstatuses：HTTP 的状态返回码，取值请参考：org.springframework.http.HttpStatus

lmethods：指定哪些方法的请求需要进行重试逻辑，默认值是 GET 方法，取值参考：org.springframework.http.HttpMethod

lseries：一些列的状态码配置，取值参考：org.springframework.http.HttpStatus.Series。符合的某段状态码才会进行重试逻辑，默认值是 SERVER\_ERROR，值是 5，也就是 5XX\(5 开头的状态码\)，共有5 个值。

#### **源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter14)



@font-face{

font-family:"Times New Roman";

}



@font-face{

font-family:"宋体";

}



@font-face{

font-family:"Wingdings";

}



@font-face{

font-family:"Arial";

}



@font-face{

font-family:"黑体";

}



@font-face{

font-family:"Calibri";

}



@font-face{

font-family:"Consolas";

}



@font-face{

font-family:"Segoe UI";

}



@font-face{

font-family:"微软雅黑";

}



@list l0:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l2:level1{

mso-level-number-format:bullet;

mso-level-suffix:tab;

mso-level-text:"";

mso-level-tab-stop:none;

mso-level-number-position:left;

margin-left:21.0000pt;text-indent:-21.0000pt;font-family:Wingdings;}



@list l3:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l5:level1{

mso-level-number-format:bullet;

mso-level-suffix:tab;

mso-level-text:"";

mso-level-tab-stop:none;

mso-level-number-position:left;

margin-left:21.0000pt;text-indent:-21.0000pt;font-family:Wingdings;}



@list l6:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



p.MsoNormal{

mso-style-name:正文;

mso-style-parent:"";

margin:0pt;

margin-bottom:.0001pt;

mso-pagination:none;

text-align:justify;

text-justify:inter-ideograph;

font-family:Calibri;

mso-fareast-font-family:宋体;

mso-bidi-font-family:'Times New Roman';

font-size:10.5000pt;

mso-font-kerning:1.0000pt;

}



h2{

mso-style-name:"标题 2";

mso-style-noshow:yes;

mso-style-next:正文;

margin-top:13.0000pt;

margin-bottom:13.0000pt;

mso-para-margin-top:0.0000gd;

mso-para-margin-bottom:0.0000gd;

page-break-after:avoid;

mso-pagination:lines-together;

text-align:justify;

text-justify:inter-ideograph;

mso-outline-level:2;

line-height:172%;

font-family:Arial;

mso-fareast-font-family:黑体;

mso-bidi-font-family:'Times New Roman';

font-weight:bold;

font-size:16.0000pt;

mso-font-kerning:1.0000pt;

}



h3{

mso-style-name:"标题 3";

mso-style-noshow:yes;

mso-style-next:正文;

margin-top:13.0000pt;

margin-bottom:13.0000pt;

mso-para-margin-top:0.0000gd;

mso-para-margin-bottom:0.0000gd;

page-break-after:avoid;

mso-pagination:lines-together;

text-align:justify;

text-justify:inter-ideograph;

mso-outline-level:3;

line-height:172%;

font-family:Calibri;

mso-fareast-font-family:宋体;

mso-bidi-font-family:'Times New Roman';

font-weight:bold;

font-size:16.0000pt;

mso-font-kerning:1.0000pt;

}



h4{

mso-style-name:"标题 4";

mso-style-noshow:yes;

mso-style-next:正文;

margin-top:14.0000pt;

margin-bottom:14.5000pt;

mso-para-margin-top:0.0000gd;

mso-para-margin-bottom:0.0000gd;

page-break-after:avoid;

mso-pagination:lines-together;

text-align:justify;

text-justify:inter-ideograph;

mso-outline-level:4;

line-height:155%;

font-family:Arial;

mso-fareast-font-family:黑体;

mso-bidi-font-family:'Times New Roman';

font-weight:bold;

font-size:14.0000pt;

mso-font-kerning:1.0000pt;

}



span.10{

font-family:'Times New Roman';

}



span.15{

font-family:'Times New Roman';

color:rgb\(0,0,255\);

text-decoration:underline;

text-underline:single;

}



span.16{

font-family:'Times New Roman';

color:rgb\(128,0,128\);

text-decoration:underline;

text-underline:single;

}



p.p{

mso-style-name:"普通\\(网站\\)";

margin-top:5.0000pt;

margin-right:0.0000pt;

margin-bottom:5.0000pt;

margin-left:0.0000pt;

mso-margin-top-alt:auto;

mso-margin-bottom-alt:auto;

mso-pagination:none;

text-align:left;

font-family:Calibri;

mso-fareast-font-family:宋体;

mso-bidi-font-family:'Times New Roman';

font-size:12.0000pt;

}



span.msoIns{

mso-style-type:export-only;

mso-style-name:"";

text-decoration:underline;

text-underline:single;

color:blue;

}



span.msoDel{

mso-style-type:export-only;

mso-style-name:"";

text-decoration:line-through;

color:red;

}

@page{mso-page-border-surround-header:no;

	mso-page-border-surround-footer:no;}@page Section0{

}

div.Section0{page:Section0;}

## **3.高级应用**

### **3.1 熔断**

1.Spring Cloud Gateway 也可以利用 Hystrix 的熔断特性，在流量过大时进行服务降级，同样我们还是首先给项目添加上依赖。

1.&lt;dependency&gt;

2.&lt;groupId&gt;org.springframework.cloud&lt;/groupId&gt;

3.&lt;artifactId&gt;spring-cloud-starter-netflix-hystrix&lt;/artifactId&gt;

4.&lt;/dependency&gt;

配置示例

1.spring:

2.cloud:

3.gateway:

4.routes:

5.-id:hystrix\_route

6.uri:http://example.org

7.filters:

8.-Hystrix=myCommandName

配置后，gateway 将使用 myCommandName 作为名称生成 HystrixCommand 对象来进行熔断管理。

2.如果想添加熔断后的回调内容，需要在添加一些配置

1.spring:

2.cloud:

3.gateway:

4.routes:

5.-id:hystrix\_route

6.uri:lb://spring-cloud-producer

7.predicates:

8.-Path=/consumingserviceendpoint

9.filters:

10.-name:Hystrix

11.args:

12.name:fallbackcmd

13.fallbackUri:forward:/incaseoffailureusethis

fallbackUri: forward:/incaseoffailureusethis配置了 fallback 时要会调的路径，当调用 Hystrix 的 fallback 被调用时，请求将转发到/incaseoffailureuset这个 URI。

#### **源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter14)





### **3.2 限流**

在Spring Cloud Gateway中，有Filter过滤器，因此可以在“pre”类型的Filter中自行实现上述三种过滤器。但是限流作为网关最基本的功能，Spring Cloud Gateway官方就提供了RequestRateLimiterGatewayFilterFactory这个类，适用Redis和lua脚本实现了令牌桶的方式。

#### **3.2.1 引入依赖和redis的reactive依赖**

1.&lt;dependency&gt;

2.&lt;groupId&gt;org.springframework.cloud&lt;/groupId&gt;

3.&lt;artifactId&gt;spring-cloud-starter-gateway&lt;/artifactId&gt;

4.&lt;/dependency&gt;

5.

6.&lt;dependency&gt;

7.&lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;

8.&lt;artifatId&gt;spring-boot-starter-data-redis-reactive&lt;/artifactId&gt;

9.&lt;/dependency&gt;

#### **3.2.2 修改配置文件**

1.server:

2.port:8081

3.spring:

4.cloud:

5.gateway:

6.routes:

7.-id:limit\_route

8.uri:http://httpbin.org:80/get

9.predicates:

10.-After=2017-01-20T17:42:47.789-07:00\[America/Denver\]

11.filters:

12.-name:RequestRateLimiter

13.args:

14.key-resolver:'\#{@hostAddrKeyResolver}'

15.redis-rate-limiter.replenishRate:1

16.redis-rate-limiter.burstCapacity:3

17.application:

18.name:gateway-limiter

19.redis:

20.host:localhost

21.port:6379

22.database:0

在上面的配置文件，指定程序的端口为8081，配置了 redis的信息，并配置了RequestRateLimiter的限流过滤器，该过滤器需要配置三个参数：

lburstCapacity，令牌桶总容量。

lreplenishRate，令牌桶每秒填充平均速率。

lkey-resolver，用于限流的键的解析器的 Bean 对象的名字。它使用 SpEL 表达式根据\#{@beanName}从 Spring 容器中获取 Bean 对象。

#### **3.2.3 实现KeyResolver**

根据Hostname进行限流，则需要用hostAddress去判断

1.**publicclass**HostAddrKeyResolver**implements**KeyResolver{

2.

3.@Override

4.**public**Mono&lt;String&gt;resolve\(ServerWebExchangeexchange\){

5.**return**Mono.just\(exchange.getRequest\(\).getRemoteAddress\(\).getAddress\(\).getHostAddress\(\)\);

6.}

7.}

8.@Bean

9.**public**HostAddrKeyResolverhostAddrKeyResolver\(\){

10.**returnnew**HostAddrKeyResolver\(\);

11.}

可以根据uri去限流，这时KeyResolver代码如下

1.**publicclass**UriKeyResolver**implements**KeyResolver{

2.

3.@Override

4.**public**Mono&lt;String&gt;resolve\(ServerWebExchangeexchange\){

5.**return**Mono.just\(exchange.getRequest\(\).getURI\(\).getPath\(\)\);

6.}

7.

8.}

9.

10.@Bean

11.**public**UriKeyResolveruriKeyResolver\(\){

12.**returnnew**UriKeyResolver\(\);

13.}

以用户的维度去限流：

1.@Bean

2.KeyResolveruserKeyResolver\(\){

3.**return**exchange-&gt;Mono.just\(exchange.getRequest\(\).getQueryParams\(\).getFirst\("user"\)\);

4.}

#### **3.2.4 源码下载**

[示例源码-GitHub](https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-gateway-limiter)

### **3.3 重试**

RetryGatewayFilter 是 Spring Cloud Gateway 对请求重试提供的一个 GatewayFilter Factory。

配置示例

1.spring:

2.cloud:

3.gateway:

4.routes:

5.-id:retry\_test

6.uri:lb://spring-cloud-producer

7.predicates:

8.-Path=/retry

9.filters:

10.-name:Retry

11.args:

12.retries:3

13.statuses:BAD\_GATEWAY

Retry GatewayFilter 通过这四个参数来控制重试机制： retries, statuses, methods, 和 series。

lretries：重试次数，默认值是 3 次

lstatuses：HTTP 的状态返回码，取值请参考：org.springframework.http.HttpStatus

lmethods：指定哪些方法的请求需要进行重试逻辑，默认值是 GET 方法，取值参考：org.springframework.http.HttpMethod

lseries：一些列的状态码配置，取值参考：org.springframework.http.HttpStatus.Series。符合的某段状态码才会进行重试逻辑，默认值是 SERVER\_ERROR，值是 5，也就是 5XX\(5 开头的状态码\)，共有5 个值。

#### **源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter14)



@font-face{

font-family:"Times New Roman";

}



@font-face{

font-family:"宋体";

}



@font-face{

font-family:"Wingdings";

}



@font-face{

font-family:"Arial";

}



@font-face{

font-family:"黑体";

}



@font-face{

font-family:"Calibri";

}



@font-face{

font-family:"Consolas";

}



@font-face{

font-family:"Segoe UI";

}



@font-face{

font-family:"微软雅黑";

}



@list l0:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l0:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l1:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l2:level1{

mso-level-number-format:bullet;

mso-level-suffix:tab;

mso-level-text:"";

mso-level-tab-stop:none;

mso-level-number-position:left;

margin-left:21.0000pt;text-indent:-21.0000pt;font-family:Wingdings;}



@list l3:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l3:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l4:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l5:level1{

mso-level-number-format:bullet;

mso-level-suffix:tab;

mso-level-text:"";

mso-level-tab-stop:none;

mso-level-number-position:left;

margin-left:21.0000pt;text-indent:-21.0000pt;font-family:Wingdings;}



@list l6:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l6:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l7:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l8:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l9:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level1{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%1.";

mso-level-tab-stop:36.0000pt;

mso-level-number-position:left;

margin-left:36.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level2{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%2.";

mso-level-tab-stop:72.0000pt;

mso-level-number-position:left;

margin-left:72.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level3{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%3.";

mso-level-tab-stop:108.0000pt;

mso-level-number-position:left;

margin-left:108.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level4{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%4.";

mso-level-tab-stop:125.8500pt;

mso-level-number-position:left;

margin-left:144.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level5{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%5.";

mso-level-tab-stop:161.9000pt;

mso-level-number-position:left;

margin-left:180.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level6{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%6.";

mso-level-tab-stop:197.9000pt;

mso-level-number-position:left;

margin-left:216.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level7{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%7.";

mso-level-tab-stop:233.9000pt;

mso-level-number-position:left;

margin-left:252.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level8{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%8.";

mso-level-tab-stop:269.9000pt;

mso-level-number-position:left;

margin-left:288.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



@list l10:level9{

mso-level-number-format:decimal;

mso-level-suffix:tab;

mso-level-text:"%9.";

mso-level-tab-stop:305.9000pt;

mso-level-number-position:left;

margin-left:324.0000pt;text-indent:-18.0000pt;font-family:'Times New Roman';font-size:12.0000pt;}



p.MsoNormal{

mso-style-name:正文;

mso-style-parent:"";

margin:0pt;

margin-bottom:.0001pt;

mso-pagination:none;

text-align:justify;

text-justify:inter-ideograph;

font-family:Calibri;

mso-fareast-font-family:宋体;

mso-bidi-font-family:'Times New Roman';

font-size:10.5000pt;

mso-font-kerning:1.0000pt;

}



h2{

mso-style-name:"标题 2";

mso-style-noshow:yes;

mso-style-next:正文;

margin-top:13.0000pt;

margin-bottom:13.0000pt;

mso-para-margin-top:0.0000gd;

mso-para-margin-bottom:0.0000gd;

page-break-after:avoid;

mso-pagination:lines-together;

text-align:justify;

text-justify:inter-ideograph;

mso-outline-level:2;

line-height:172%;

font-family:Arial;

mso-fareast-font-family:黑体;

mso-bidi-font-family:'Times New Roman';

font-weight:bold;

font-size:16.0000pt;

mso-font-kerning:1.0000pt;

}



h3{

mso-style-name:"标题 3";

mso-style-noshow:yes;

mso-style-next:正文;

margin-top:13.0000pt;

margin-bottom:13.0000pt;

mso-para-margin-top:0.0000gd;

mso-para-margin-bottom:0.0000gd;

page-break-after:avoid;

mso-pagination:lines-together;

text-align:justify;

text-justify:inter-ideograph;

mso-outline-level:3;

line-height:172%;

font-family:Calibri;

mso-fareast-font-family:宋体;

mso-bidi-font-family:'Times New Roman';

font-weight:bold;

font-size:16.0000pt;

mso-font-kerning:1.0000pt;

}



h4{

mso-style-name:"标题 4";

mso-style-noshow:yes;

mso-style-next:正文;

margin-top:14.0000pt;

margin-bottom:14.5000pt;

mso-para-margin-top:0.0000gd;

mso-para-margin-bottom:0.0000gd;

page-break-after:avoid;

mso-pagination:lines-together;

text-align:justify;

text-justify:inter-ideograph;

mso-outline-level:4;

line-height:155%;

font-family:Arial;

mso-fareast-font-family:黑体;

mso-bidi-font-family:'Times New Roman';

font-weight:bold;

font-size:14.0000pt;

mso-font-kerning:1.0000pt;

}



span.10{

font-family:'Times New Roman';

}



span.15{

font-family:'Times New Roman';

color:rgb\(0,0,255\);

text-decoration:underline;

text-underline:single;

}



span.16{

font-family:'Times New Roman';

color:rgb\(128,0,128\);

text-decoration:underline;

text-underline:single;

}



p.p{

mso-style-name:"普通\\(网站\\)";

margin-top:5.0000pt;

margin-right:0.0000pt;

margin-bottom:5.0000pt;

margin-left:0.0000pt;

mso-margin-top-alt:auto;

mso-margin-bottom-alt:auto;

mso-pagination:none;

text-align:left;

font-family:Calibri;

mso-fareast-font-family:宋体;

mso-bidi-font-family:'Times New Roman';

font-size:12.0000pt;

}



span.msoIns{

mso-style-type:export-only;

mso-style-name:"";

text-decoration:underline;

text-underline:single;

color:blue;

}



span.msoDel{

mso-style-type:export-only;

mso-style-name:"";

text-decoration:line-through;

color:red;

}

@page{mso-page-border-surround-header:no;

	mso-page-border-surround-footer:no;}@page Section0{

}

div.Section0{page:Section0;}

## **3.高级应用**

### **3.1 熔断**

1.Spring Cloud Gateway 也可以利用 Hystrix 的熔断特性，在流量过大时进行服务降级，同样我们还是首先给项目添加上依赖。

1.&lt;dependency&gt;

2.&lt;groupId&gt;org.springframework.cloud&lt;/groupId&gt;

3.&lt;artifactId&gt;spring-cloud-starter-netflix-hystrix&lt;/artifactId&gt;

4.&lt;/dependency&gt;

配置示例

1.spring:

2.cloud:

3.gateway:

4.routes:

5.-id:hystrix\_route

6.uri:http://example.org

7.filters:

8.-Hystrix=myCommandName

配置后，gateway 将使用 myCommandName 作为名称生成 HystrixCommand 对象来进行熔断管理。

2.如果想添加熔断后的回调内容，需要在添加一些配置

1.spring:

2.cloud:

3.gateway:

4.routes:

5.-id:hystrix\_route

6.uri:lb://spring-cloud-producer

7.predicates:

8.-Path=/consumingserviceendpoint

9.filters:

10.-name:Hystrix

11.args:

12.name:fallbackcmd

13.fallbackUri:forward:/incaseoffailureusethis

fallbackUri: forward:/incaseoffailureusethis配置了 fallback 时要会调的路径，当调用 Hystrix 的 fallback 被调用时，请求将转发到/incaseoffailureuset这个 URI。

#### **源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter14)





### **3.2 限流**

在Spring Cloud Gateway中，有Filter过滤器，因此可以在“pre”类型的Filter中自行实现上述三种过滤器。但是限流作为网关最基本的功能，Spring Cloud Gateway官方就提供了RequestRateLimiterGatewayFilterFactory这个类，适用Redis和lua脚本实现了令牌桶的方式。

#### **3.2.1 引入依赖和redis的reactive依赖**

1.&lt;dependency&gt;

2.&lt;groupId&gt;org.springframework.cloud&lt;/groupId&gt;

3.&lt;artifactId&gt;spring-cloud-starter-gateway&lt;/artifactId&gt;

4.&lt;/dependency&gt;

5.

6.&lt;dependency&gt;

7.&lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;

8.&lt;artifatId&gt;spring-boot-starter-data-redis-reactive&lt;/artifactId&gt;

9.&lt;/dependency&gt;

#### **3.2.2 修改配置文件**

1.server:

2.port:8081

3.spring:

4.cloud:

5.gateway:

6.routes:

7.-id:limit\_route

8.uri:http://httpbin.org:80/get

9.predicates:

10.-After=2017-01-20T17:42:47.789-07:00\[America/Denver\]

11.filters:

12.-name:RequestRateLimiter

13.args:

14.key-resolver:'\#{@hostAddrKeyResolver}'

15.redis-rate-limiter.replenishRate:1

16.redis-rate-limiter.burstCapacity:3

17.application:

18.name:gateway-limiter

19.redis:

20.host:localhost

21.port:6379

22.database:0

在上面的配置文件，指定程序的端口为8081，配置了 redis的信息，并配置了RequestRateLimiter的限流过滤器，该过滤器需要配置三个参数：

lburstCapacity，令牌桶总容量。

lreplenishRate，令牌桶每秒填充平均速率。

lkey-resolver，用于限流的键的解析器的 Bean 对象的名字。它使用 SpEL 表达式根据\#{@beanName}从 Spring 容器中获取 Bean 对象。

#### **3.2.3 实现KeyResolver**

根据Hostname进行限流，则需要用hostAddress去判断

1.**publicclass**HostAddrKeyResolver**implements**KeyResolver{

2.

3.@Override

4.**public**Mono&lt;String&gt;resolve\(ServerWebExchangeexchange\){

5.**return**Mono.just\(exchange.getRequest\(\).getRemoteAddress\(\).getAddress\(\).getHostAddress\(\)\);

6.}

7.}

8.@Bean

9.**public**HostAddrKeyResolverhostAddrKeyResolver\(\){

10.**returnnew**HostAddrKeyResolver\(\);

11.}

可以根据uri去限流，这时KeyResolver代码如下

1.**publicclass**UriKeyResolver**implements**KeyResolver{

2.

3.@Override

4.**public**Mono&lt;String&gt;resolve\(ServerWebExchangeexchange\){

5.**return**Mono.just\(exchange.getRequest\(\).getURI\(\).getPath\(\)\);

6.}

7.

8.}

9.

10.@Bean

11.**public**UriKeyResolveruriKeyResolver\(\){

12.**returnnew**UriKeyResolver\(\);

13.}

以用户的维度去限流：

1.@Bean

2.KeyResolveruserKeyResolver\(\){

3.**return**exchange-&gt;Mono.just\(exchange.getRequest\(\).getQueryParams\(\).getFirst\("user"\)\);

4.}

#### **3.2.4 源码下载**

[示例源码-GitHub](https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-gateway-limiter)

### **3.3 重试**

RetryGatewayFilter 是 Spring Cloud Gateway 对请求重试提供的一个 GatewayFilter Factory。

配置示例

1.spring:

2.cloud:

3.gateway:

4.routes:

5.-id:retry\_test

6.uri:lb://spring-cloud-producer

7.predicates:

8.-Path=/retry

9.filters:

10.-name:Retry

11.args:

12.retries:3

13.statuses:BAD\_GATEWAY

Retry GatewayFilter 通过这四个参数来控制重试机制： retries, statuses, methods, 和 series。

lretries：重试次数，默认值是 3 次

lstatuses：HTTP 的状态返回码，取值请参考：org.springframework.http.HttpStatus

lmethods：指定哪些方法的请求需要进行重试逻辑，默认值是 GET 方法，取值参考：org.springframework.http.HttpMethod

lseries：一些列的状态码配置，取值参考：org.springframework.http.HttpStatus.Series。符合的某段状态码才会进行重试逻辑，默认值是 SERVER\_ERROR，值是 5，也就是 5XX\(5 开头的状态码\)，共有5 个值。

#### **源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter14)

## **3.高级应用**

### **3.1 熔断**

1.Spring Cloud Gateway 也可以利用 Hystrix 的熔断特性，在流量过大时进行服务降级，同样我们还是首先给项目添加上依赖。

```
1.<dependency>  
2.  <groupId>org.springframework.cloud</groupId>  
3.  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>  
4.</dependency>  
```

配置示例

```
1.spring:  
2.  cloud:  
3.    gateway:  
4.      routes:  
5.      - id: hystrix_route  
6.        uri: http://example.org  
7.        filters:  
8.        - Hystrix=myCommandName  
```

配置后，gateway 将使用 myCommandName 作为名称生成 HystrixCommand 对象来进行熔断管理。

2.如果想添加熔断后的回调内容，需要在添加一些配置

```
1.spring:  
2.  cloud:  
3.    gateway:  
4.      routes:  
5.      - id: hystrix_route  
6.        uri: lb://spring-cloud-producer  
7.        predicates:  
8.        - Path=/consumingserviceendpoint  
9.        filters:  
10.        - name: Hystrix  
11.          args:  
12.            name: fallbackcmd  
13.            fallbackUri: forward:/incaseoffailureusethis  
```

fallbackUri: forward:/incaseoffailureusethis配置了 fallback 时要会调的路径，当调用 Hystrix 的 fallback 被调用时，请求将转发到/incaseoffailureuset这个 URI。

#### **源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter14)





### **3.2 限流**

在Spring Cloud Gateway中，有Filter过滤器，因此可以在“pre”类型的Filter中自行实现上述三种过滤器。但是限流作为网关最基本的功能，Spring Cloud Gateway官方就提供了RequestRateLimiterGatewayFilterFactory这个类，适用Redis和lua脚本实现了令牌桶的方式。

#### **3.2.1 引入依赖和redis的reactive依赖**

```
1.<dependency>  
2.    <groupId>org.springframework.cloud</groupId>  
3.    <artifactId>spring-cloud-starter-gateway</artifactId>  
4.</dependency>  
5.  
6.<dependency>  
7.    <groupId>org.springframework.boot</groupId>  
8.    <artifatId>spring-boot-starter-data-redis-reactive</artifactId>  
9.</dependency>  
```

#### **3.2.2 修改配置文件**

```
1.server:  
2.  port: 8081  
3.spring:  
4.  cloud:  
5.    gateway:  
6.      routes:  
7.      - id: limit_route  
8.        uri: http://httpbin.org:80/get  
9.        predicates:  
10.        - After=2017-01-20T17:42:47.789-07:00[America/Denver]  
11.        filters:  
12.        - name: RequestRateLimiter  
13.          args:  
14.            key-resolver: '#{@hostAddrKeyResolver}'  
```

在上面的配置文件，指定程序的端口为8081，配置了 redis的信息，并配置了RequestRateLimiter的限流过滤器，该过滤器需要配置三个参数：

* burstCapacity，令牌桶总容量。

* replenishRate，令牌桶每秒填充平均速率。

* key-resolver，用于限流的键的解析器的 Bean 对象的名字。它使用 SpEL 表达式根据\#{@beanName}从 Spring 容器中获取 Bean 对象。

#### **3.2.3 实现KeyResolver**

根据Hostname进行限流，则需要用hostAddress去判断

```
1.public class HostAddrKeyResolver implements KeyResolver {  
2.  
3.    @Override  
4.    public Mono<String> resolve(ServerWebExchange exchange) {  
5.        return Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());  
6.    }   
7.}    
8. @Bean  
9.    public HostAddrKeyResolver hostAddrKeyResolver() {  
10.        return new HostAddrKeyResolver();  
11.    }  
```

可以根据uri去限流，这时KeyResolver代码如下

```
1.public class UriKeyResolver  implements KeyResolver {  
2.  
3.    @Override  
4.    public Mono<String> resolve(ServerWebExchange exchange) {  
5.        return Mono.just(exchange.getRequest().getURI().getPath());  
6.    }  
7.  
8.}  
9.  
10. @Bean  
11. public UriKeyResolver uriKeyResolver() {  
12.     return new UriKeyResolver();  
13. }  
```

以用户的维度去限流：

```
1.@Bean  
2.KeyResolver userKeyResolver() {  
3.   return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));  
} 
```

#### **3.2.4 源码下载**

[示例源码-GitHub](https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-gateway-limiter)

### **3.3 重试**

RetryGatewayFilter 是 Spring Cloud Gateway 对请求重试提供的一个 GatewayFilter Factory。

配置示例

```
1.spring:  
2.  cloud:  
3.    gateway:  
4.      routes:  
5.      - id: retry_test  
6.        uri: lb://spring-cloud-producer  
7.        predicates:  
8.        - Path=/retry  
9.        filters:  
10.        - name: Retry  
11.          args:  
12.            retries: 3  
13.            statuses: BAD_GATEWAY  
```

* retries：重试次数，默认值是 3 次

* statuses：HTTP 的状态返回码，取值请参考：org.springframework.http.HttpStatus

* methods：指定哪些方法的请求需要进行重试逻辑，默认值是 GET 方法，取值参考：org.springframework.http.HttpMethod

* series：一些列的状态码配置，取值参考：org.springframework.http.HttpStatus.Series。符合的某段状态码才会进行重试逻辑，默认值是 SERVER\_ERROR，值是 5，也就是 5XX\(5 开头的状态码\)，共有5 个值。

#### **源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter14)

