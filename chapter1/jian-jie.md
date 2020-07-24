## 1.背景

Zuul是Netflix开源的一个项目，Spring只是将Zuul集成在了Spring Cloud中。而Spring Cloud Gateway是Spring Cloud的一个子项目。

还有一个版本的说法是Zuul2的连续跳票和Zuul1的性能并不是很理想，从而催生了Spring Cloud Gateway。

## 2.**什么是Spring Cloud Gateway**

Spring Cloud Gateway 是 Spring Cloud 的一个全新项目，该项目是基于 Spring 5.0，Spring Boot 2.0 和 Project Reactor 等技术开发的网关，它旨在为微服务架构提供一种简单有效的统一的 API 路由管理方式。

Spring Cloud Gateway 作为 Spring Cloud 生态系统中的网关，目标是替代 Netflix Zuul，其不仅提供统一的路由方式，并且基于 Filter 链的方式提供了网关基本的功能，例如：安全，监控/指标，和限流。

## 3.**特点**

* 基于Spring Framework5， Project Reactor 和 Spring Boot 2.0
* 动态路由
* Predicates 和 Filter作用于特定路由
* 集成Hystrix熔断路由
* 集成Spring Cloud DisccoveryClient
* 易于编写的Predicates和Filters
* 限流
* 路劲重新



