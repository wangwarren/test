## 3.高级应用

### **3.1 熔断**

1.Spring Cloud Gateway 也可以利用 Hystrix 的熔断特性，在流量过大时进行服务降级，同样我们还是首先给项目添加上依赖。

```
1.<dependency>  
2.  <groupId>org.springframework.cloud</groupId>  
3.  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>  
4.</dependency>
```

配置示例

```
1.spring:  
2.  cloud:  
3.    gateway:  
4.      routes:  
5.      - id: hystrix_route  
6.        uri: http://example.org  
7.        filters:  
        - Hystrix=myCommandName
```

配置后，gateway 将使用 myCommandName 作为名称生成 HystrixCommand 对象来进行熔断管理。

2.如果想添加熔断后的回调内容，需要在添加一些配置

```
1.spring:  
2.  cloud:  
3.    gateway:  
4.      routes:  
5.      - id: hystrix_route  
6.        uri: lb://spring-cloud-producer  
7.        predicates:  
8.        - Path=/consumingserviceendpoint  
9.        filters:  
10.        - name: Hystrix  
11.          args:  
12.            name: fallbackcmd  
            fallbackUri: forward:/incaseoffailureusethis
```

fallbackUri: forward:/incaseoffailureusethis配置了 fallback 时要会调的路径，当调用 Hystrix 的 fallback 被调用时，请求将转发到/incaseoffailureuset这个 URI。

#### **源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter14)

### **3.2 限流**

在Spring Cloud Gateway中，有Filter过滤器，因此可以在“pre”类型的Filter中自行实现上述三种过滤器。但是限流作为网关最基本的功能，Spring Cloud Gateway官方就提供了RequestRateLimiterGatewayFilterFactory这个类，适用Redis和lua脚本实现了令牌桶的方式。

#### **3.2.1 引入依赖和redis的reactive依赖**

```
1.<dependency>  
2.    <groupId>org.springframework.cloud</groupId>  
3.    <artifactId>spring-cloud-starter-gateway</artifactId>  
4.</dependency>  
5.  
6.<dependency>  
7.    <groupId>org.springframework.boot</groupId>  
8.    <artifatId>spring-boot-starter-data-redis-reactive</artifactId>  
9.</dependency>
```

#### **3.2.2 修改配置文件**

```
1.server:  
2.  port: 8081  
3.spring:  
4.  cloud:  
5.    gateway:  
6.      routes:  
7.      - id: limit_route  
8.        uri: http://httpbin.org:80/get  
9.        predicates:  
10.        - After=2017-01-20T17:42:47.789-07:00[America/Denver]  
11.        filters:  
12.        - name: RequestRateLimiter  
13.          args:  
14.            key-resolver: '#{@hostAddrKeyResolver}'
15.            redis-rate-limiter.replenishRate: 1  
16.            redis-rate-limiter.burstCapacity: 3  
17.  application:  
18.    name: gateway-limiter  
19.  redis:  
20.    host: localhost  
21.    port: 6379  
    database: 0
```

在上面的配置文件，指定程序的端口为8081，配置了 redis的信息，并配置了RequestRateLimiter的限流过滤器，该过滤器需要配置三个参数：

* burstCapacity，令牌桶总容量。

* replenishRate，令牌桶每秒填充平均速率。

* key-resolver，用于限流的键的解析器的 Bean 对象的名字。它使用 SpEL 表达式根据\#{@beanName}从 Spring 容器中获取 Bean 对象。

#### **3.2.3 实现KeyResolver**

根据Hostname进行限流，则需要用hostAddress去判断

```
1.public class HostAddrKeyResolver implements KeyResolver {  
2.  
3.    @Override  
4.    public Mono<String> resolve(ServerWebExchange exchange) {  
5.        return Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());  
6.    }   
7.}    
8. @Bean  
9.    public HostAddrKeyResolver hostAddrKeyResolver() {  
10.        return new HostAddrKeyResolver();  
11.    }
```

可以根据uri去限流，这时KeyResolver代码如下

```
1.public class UriKeyResolver  implements KeyResolver {  
2.  
3.    @Override  
4.    public Mono<String> resolve(ServerWebExchange exchange) {  
5.        return Mono.just(exchange.getRequest().getURI().getPath());  
6.    }  
7.  
8.}  
9.  
10. @Bean  
11. public UriKeyResolver uriKeyResolver() {  
12.     return new UriKeyResolver();
```

以用户的维度去限流：

```
1.@Bean  
2.KeyResolver userKeyResolver() {  
3.   return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));  
}
```

#### **3.2.4 源码下载**

[示例源码-GitHub](https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-gateway-limiter)

### **3.3 重试**

RetryGatewayFilter 是 Spring Cloud Gateway 对请求重试提供的一个 GatewayFilter Factory。

配置示例

```
1.spring:  
2.  cloud:  
3.    gateway:  
4.      routes:  
5.      - id: retry_test  
6.        uri: lb://spring-cloud-producer  
7.        predicates:  
8.        - Path=/retry  
9.        filters:  
10.        - name: Retry  
11.          args:  
12.            retries: 3  
13.            statuses: BAD_GATEWAY
```

* retries：重试次数，默认值是 3 次

* statuses：HTTP 的状态返回码，取值请参考：org.springframework.http.HttpStatus

* methods：指定哪些方法的请求需要进行重试逻辑，默认值是 GET 方法，取值参考：org.springframework.http.HttpMethod

* series：一些列的状态码配置，取值参考：org.springframework.http.HttpStatus.Series。符合的某段状态码才会进行重试逻辑，默认值是 SERVER\_ERROR，值是 5，也就是 5XX\(5 开头的状态码\)，共有5 个值。

#### **源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter14)

