  
Spring Cloud Gateway 网关路由有两种配置方式：

1\)在配置文件 yml 中配置

2\)通过@Bean自定义 RouteLocator，在启动主类 Application 中配置

这两种方式是等价的，建议使用 yml 方式进配置。

## **4.1项目依赖**

```
1.<dependency>  
2.     <groupId>org.springframework.cloud</groupId>  
3.     <artifactId>spring-cloud-starter-gateway</artifactId>  
</dependency>  
```

Spring Cloud Gateway 是使用 netty+webflux 实现因此不需要再引入 web 模块。

## **4.2 配置文件**

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://blog.csdn.net  
11.          predicates:  
12.            - Path=/meteor_93  
```

各字段含义如下：

* id：我们自定义的路由 ID，保持唯一
* uri：目标服务地址
* predicates：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。

上面这段配置的意思是，配置了一个 id 为 gateway-service 的路由规则，当访问地址http://localhost:8080/meteor\_93时会自动转发到地址：

http://localhost:8080/meteor\_93。

## **4.3 另一种路由配置方式**

转发功能同样可以通过代码来实现，我们可以在启动类 GateWayApplication 中添加方法 customRouteLocator\(\) 来定制转发规则。

```
1.@SpringBootApplication  
2.public class GatewayApplication {   
3.    public static void main(String[] args) {  
4.        SpringApplication.run(GatewayApplication.class, args);  
5.    }  
6.  
7.    @Bean  
8.    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {  
9.        return builder.routes()  
10.                .route("path_route", r -> r.path("/meteor_93")  
11.                        .uri("https://blog.csdn.net"))  
12.                .build();  
13.    }  
14.}  
```

## **4.4 服务注册与发现**

实际的工作中，服务的相互调用都是依赖于服务中心提供的入口来使用，服务中心往往注册了很多服务，如果每个服务都需要单独配置的话，这将是一份很枯燥的工作。Spring Cloud Gateway 提供了一种默认转发的能力，只要将 Spring Cloud Gateway 注册到服务中心，Spring Cloud Gateway 默认就会代理服务中心的所有服务。

### **4.4.1 服务网关注册到注册中心**

添加对eureka的依赖，在启动文件加入注解@EnableEurekaClient。

```
1.<dependency>  
2.  <groupId>org.springframework.cloud</groupId>  
3.  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>  
4.</dependency> 
```

修改配置文件application.yml：

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      discovery:  
9.        locator:  
10.          enabled: true  
11.eureka:  
12.  client:  
13.    service-url:  
14.      defaultZone: http://localhost:8761/eureka/  
15.logging:  
16.  level:  
17.    org.springframework.cloud.gateway: debug  
```

配置说明：

lspring.cloud.gateway.discovery.locator.enabled：是否与服务注册于发现组件进行结合，通过 serviceId 转发到具体的服务实例。默认为 false，设为 true 便开启通过服务中心的自动根据 serviceId 创建路由的功能。

leureka.client.service-url.defaultZone指定注册中心的地址，以便使用服务发现功能

llogging.level.org.springframework.cloud.gateway 调整相 gateway 包的 log 级别，以便排查问题

修改完成后启动 gateway 项目，访问注册中心地址http://localhost:8761/即可看到名为 API-GATEWAY的服务。

### **4.4.2 测试**

将 gateway 注册到服务中心之后，网关会自动代理所有的在注册中心的服务，访问这些服务的语法为：

http://网关地址：端口/服务中心注册serviceId/具体的url



