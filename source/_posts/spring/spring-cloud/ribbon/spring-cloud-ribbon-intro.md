---
title: Spring Cloud Ribbon 简介
date: 2017-11-08 19:20:37
categories: "Spring Cloud"
tags:
    - Spring Cloud
    - Spring Boot
    - Java
    - Ribbon
keywords: Spring Cloud, Spring Boot, Java, Ribbon
---

- 基于 HTTP 和 TCP 的 **客户端** 负载均衡工具
- 基于 Netflix Ribbon 实现，通过 Spring Cloud 封装

[Demo Address](https://github.com/singoasher/spring-cloud-ribbon-demo)

<!-- more -->

## 依赖

Eureka + Ribbon:

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

`spring-cloud-starter-eureka` 已包含必要依赖，其中包括 Ribbon 相关

Ribbon:

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```


## 使用

### 虚拟主机名

使用 `@LoadBalanced` 的 `RestTemplate` 对象时，虚拟主机名可以 **自动映射** 为对应的服务地址。

如下面会用到的 `http://stores/stores` 中，`stores` 即为虚拟主机名。

通过 Eureka + Ribbon 访问服务时，默认情况下，虚拟主机名与服务名一致，如果需要设定，在服务端修改如下配置

```
eureka:
  instance:
    virtual-host-name: foobar
```

仅通过 Ribbon 访问服务，虚拟主机名则为配置文件中设定的客户端名称，如下例中的 `foo`

```
foo:
  ribbon:
    listOfServers: http://example.com:10088,http://example.com:20088
```

详见下文 [脱离注册中心使用](#脱离注册中心使用)


### 创建负载均衡的 RestTemplate 对象

```java
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    public String doOtherStuff() {
        return restTemplate.getForObject("http://stores/stores", String.class);
    }
}
```

### 创建不同需求的 RestTemplate 对象

If you want a RestTemplate that is not load balanced, create a RestTemplate bean and inject it as normal. 
To access the load balanced RestTemplate use the @LoadBalanced qualifier when you create your @Bean.

```java
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate loadBalanced() {
        return new RestTemplate();
    }

    @Primary
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    @LoadBalanced
    private RestTemplate loadBalanced;
    
    @Autowired
    private RestTemplate restTemplate;

    public String doOtherStuff() {
        return loadBalanced.getForObject("http://stores/stores", String.class);
    }

    public String doStuff() {
        return restTemplate.getForObject("http://example.com", String.class);
    }
}
```

### 直接使用 Ribbon API

```java
public class MyClass {
    @Autowired
    private LoadBalancerClient loadBalancer;

    public void doStuff() {
        ServiceInstance instance = loadBalancer.choose("stores");
        URI storesUri = URI.create(String.format("http://%s:%s", instance.getHost(), instance.getPort()));
        // ... do something with the URI
    }
}
```


## 脱离注册中心使用

注册中心并非 Ribbon 使用的必要条件，在无注册中心的情况下，可通过下面配置，设置 Ribbon 的服务访问列表

```
foo:
  ribbon:
    listOfServers: http://example.com:10088,http://example.com:20088
```

适用场景：
- 无注册中心
- 未在注册中心注册
- Ribbon 禁用了注册中心

### 配置 Ribbon 禁用 Eureka

```
ribbon:
  eureka:
    enabled: false
```


## 自定义配置

### 配置类

```java
@Configuration
public class FooConfiguration {
    @Bean
    public IPing ribbonPing(IClientConfig config) {
        return new PingUrl();
    }
}
```

**WARNING** 
- The FooConfiguration has to be **@Configuration** 
- but take care that it is **NOT** in a **@ComponentScan** for the main application context, 
- **otherwise** it will be shared by all the @RibbonClients. 

> If you use @ComponentScan (or @SpringBootApplication) you need to take steps to avoid it being included 
> (for instance put it in a separate, non-overlapping package, or specify the packages to scan explicitly in the @ComponentScan).


### 配置使能 @RibbonClient

```java
@Configuration
@RibbonClient(name = "foo", configuration = FooConfiguration.class)
public class TestConfiguration {
}
```

### 配置使能 @RibbonClients

```java
@Configuration
@RibbonClients(value = {
        @RibbonClient(name = "foo", configuration = FooConfiguration.class),
        @RibbonClient(name = "bar", configuration = BarConfiguration.class)
})
public class TestConfiguration {
}
```

### 配置介绍

默认配置 `org.springframework.cloud.netflix.ribbon.RibbonClientConfiguration`

- Rule 
    - 逻辑组件，用于决定本次选择服务列表中的哪一个
- Ping 
    - 后台运行，确认各个服务的存活状态
- ServerList 
    - 可以被静态或动态指定，动态指定（via `DynamicServerListLoadBalancer`）时，后台线程会以特定时间间隔，刷新并过滤服务列表

Spring Cloud Netflix provides the following beans by default for ribbon (BeanType beanName: ClassName):
- IClientConfig ribbonClientConfig: DefaultClientConfigImpl
- IRule ribbonRule: ZoneAvoidanceRule
- IPing ribbonPing: NoOpPing
- ServerList<Server> ribbonServerList: ConfigurationBasedServerList
- ServerListFilter<Server> ribbonServerListFilter: ZonePreferenceServerListFilter
- ILoadBalancer ribbonLoadBalancer: ZoneAwareLoadBalancer
- ServerListUpdater ribbonServerListUpdater: PollingServerListUpdater


### 通过属性自定义 Ribbon 客户端

```
foo:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

The supported properties are listed below and should be prefixed by <clientName>.ribbon.:
- NFLoadBalancerClassName: should implement ILoadBalancer
- NFLoadBalancerRuleClassName: should implement IRule
- NFLoadBalancerPingClassName: should implement IPing
- NIWSServerListClassName: should implement ServerList
- NIWSServerListFilterClassName should implement ServerListFilter

> Classes defined in these properties have precedence over 
> beans defined using @RibbonClient(configuration=MyRibbonConfig.class) and the defaults provided by Spring Cloud Netflix.

**说明** 以上优先级说明是官方文档的备注，但是实测效果，优先级如下
- beans defined using @RibbonClient(configuration=MyRibbonConfig.class)
- classes defined in these properties
- the defaults provided by Spring Cloud Netflix

**注意** 虚拟主机名、@RibbonClient 配置的名称、通过属性配置的名称，同一种服务名称应保持 **大小写一致**


## 立即加载

Each Ribbon named client has a corresponding child Application Context that Spring Cloud maintains, 
this application context is lazily loaded up on the first request to the named client. 

This lazy loading behavior can be changed to instead eagerly load up these child Application contexts 
at startup by specifying the names of the Ribbon clients.

```yaml
ribbon:
  eager-load:
    enabled: true
    clients: client1, client2, client3
```


## 参考链接

- [Spring Cloud](http://cloud.spring.io/spring-cloud-static/Dalston.SR3/)

