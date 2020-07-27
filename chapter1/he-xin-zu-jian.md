<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [](#)
- [1.**流程**](#1%E6%B5%81%E7%A8%8B)
- [2.**核心组件介绍**](#2%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6%E4%BB%8B%E7%BB%8D)
  - [2**.1 Predicate介绍**](#21-predicate%E4%BB%8B%E7%BB%8D)
  - [2**.2 Filter介绍**](#22-filter%E4%BB%8B%E7%BB%8D)
    - [2**.2.1 作用**](#221-%E4%BD%9C%E7%94%A8)
    - [2**.2.2 生命周期**](#222-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## 

## 1.**流程**

![](/assets/图片1.png)

客户端向  Spring Cloud Gateway 发出请求。然后在 Gateway Handler Mapping 中找到与请求相匹配的路由，将其发送到 Gateway Web Handler。Handler 再通过指定的过滤器链来将请求发送到我们实际的服务执行业务逻辑，然后返回。过滤器之间用虚线分开是因为过滤器可能会在发送代理请求之前（“pre”）或之后（“post”）执行业务逻辑。

## 2.**核心组件介绍**

**Route（路由）**：这是网关的基本构建块。它由一个 ID，一个目标 URI，一组断言和一组过滤器定义。如果断言为真，则路由匹配。

**Predicate（断言）**：这是一个 Java 8 的 Predicate。输入类型是一个 ServerWebExchange。我们可以使用它来匹配来自 HTTP 请求的任何内容，例如 headers 或参数。

**Filter（过滤器）**：这是org.springframework.cloud.gateway.filter.GatewayFilter的实例，我们可以使用它修改请求和响应。

### 2**.1 Predicate介绍**

Predicate 来源于 Java 8，是 Java 8 中引入的一个函数，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。可以用于接口请求参数校验、判断新老数据是否有变化需要进行更新操作。

在 Spring Cloud Gateway 中 Spring 利用 Predicate 的特性实现了各种路由匹配规则，有通过 Header、请求参数等不同的条件来进行作为条件匹配到对应的路由。 Spring Cloud 内置的几种 Predicate 的实现。

1\)请求时间校验条件谓语-datetime

①AfteRoutePredicateFactory：请求时间满足在配置时间之后

②BeforeRoutePredicateFactory：请求时间满足在配置时间之前

③BetweenRoutePredicateFactory：请求时间满足在配合时间之间

2\)请求Cookie校验条件谓语-Cookie

①CookieRoutePredicateFactory：请求指定Cookie正则匹配指定值

3\)请求Header校验条件谓语-Header

①HeaderRoutePredicateFactory：请求指定Header正则匹配指定值

②CloudFoundRouteServiceRoutePredicateFactory：请求Header是否包含指定名称

4\)请求Host校验条件谓语-Host

①HostRoutePredicateFactory：请求Host匹配指定值

5\)请求Method校验条件谓语-Method

①MethodRoutePredicateFactory：请求Method匹配配置的method

6\)请求Path校验条件谓语-path

①PathRoutePredicateFactory：请求路径正则匹配指定值

7\)请求查询参数校验条件谓语-Queryparam

①QueryRoutePredicateFactory：请求查询参数正则匹配指定值

8\)请求远程地址校验条件谓语-RemoteAddr

①RemoteAddrRoutePredicateFactory：请求远程地址匹配配置指定值

### 2**.2 Filter介绍**

Predict决定了请求由哪一个路由处理，而filter在路由处理之前，需要经过“pre”类型的过滤器处理，处理返回响应之后，可以由“post”类型的过滤器处理。Spring Cloud Gateway 内置了9种 GlobalFilter，比如 Netty Routing Filter、LoadBalancerClient Filter、Websocket Routing Filter 等，根据名字即可猜测出这些 Filter 的作者，具体大家可以参考官网内容：[Global Filters](#_global_filters)

#### 2**.2.1 作用**

![](/assets/图片2.png)

Spring Cloud Gateway 的 Filter 的生命周期不像 Zuul 的那么丰富，它只有两个：“pre” 和 “post”。

lPRE： 这种过滤器在请求被路由之前调用。我们可利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等。

lPOST：这种过滤器在路由到微服务以后执行。这种过滤器可用来为响应添加标准的 HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等。

利用 GatewayFilter 可以修改请求的 Http 的请求或者响应，或者根据请求或者响应做一些特殊的限制，更多时候我们会利用 GatewayFilter 做一些具体的路由配置。

![](/assets/图片3.png)

#### 2**.2.2 生命周期**

Spring Cloud Gateway同zuul类似，有“pre”和“post”两种方式的filter。客户端的请求先经过“pre”类型的filter，然后将请求转发到具体的业务服务，比如上图中的user-service，收到业务服务的响应之后，再经过“post”类型的filter处理，最后返回响应到客户端。

![](/assets/图片4.png)

Spring Cloud Gateway 的 Filter 分为两种：GatewayFilter 与 GlobalFilter。GlobalFilter 会应用到所有的路由上，而 GatewayFilter 将应用到单个路由或者一个分组的路由上。

