---
typora-root-url: ../../../typora
---

[TOC]

# 概述

**Feign 封装了Http 调用流程，更适合面向接口化的变成习惯**。

- Feign 宗旨：使编写 Java Http 客户端变得更容易。
- <font color=red>通过 Feign 只需要定义服务绑定接口且以声明式的方法</font>，优雅而简单的显示了服务调用。
- Feign 集成了 Ribbon，通过轮训实现了客户端的负载均衡。



Feign 是如何设计？

![](/images/spring_cloud/WX20230329-211031@2x.png)





# 快速上手

项目架构如下图：

- Client：从 Nacos Server 中获取需要调用的微服务的信息（IP 和 Port）
- Server：会将自己的信息注册进 Nacos Server
- Client：通过 OpenFeign 远程调用 Server 服务。

![](/images/spring_cloud/WX20230330-231136@2x.png)



## client 项目

主要 mvn 依赖

```xml
        <!--openfeign-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <!--nacos-discovery-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
```



yaml 配置信息

```yaml
server:
  port: 80

spring:
  application:
    name: cloud-client-feign80
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
```



主启动类：注意通过注解需要 @EnableFeignClients 启动 Feign

```java
@SpringBootApplication
@EnableFeignClients
public class FeignMain80 {
    public static void main(String[] args) {
        SpringApplication.run(FeignMain80.class, args);
    }
}
```



业务类

- <font color=red>@FeignClient 这个注解表示此类使用 FeignClient 进行远程调用。</font>
- @FeignClient 的 value 值是：需要调用的微服务在 Nacos 中注册的名称。
- FeignClient 通过 微服务名称 + GetMapping 的地址，定位到需要调用的是哪个服务。



```java
@Component
@FeignClient(value = "cloudalibaba-sentinel-service")
public interface FeignService {
    @GetMapping(value = "/user/{id}")
    User getUserById(@PathVariable("id") Long id);
}
```



Controller

- 后端程序员习惯面向接口编程，通过 OpenFeign 的封装，符合了程序的开发习惯。

```java
@RestController
public class FeignController {
    @Resource
    private FeignService feignService;

    @GetMapping(value = "/feign/user/get/{id}")
    public User getPaymentById(@PathVariable("id") Long id) {
        return feignService.getUserById(id);
    }
}
```



## Server 项目

主要 mvn 依赖

```xml
        <!--SpringCloud ailibaba nacos -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
```



yaml 配置信息

```yaml
server:
  port: 8041

spring:
  application:
    name: cloudalibaba-sentinel-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
```



主启动类

```java
@EnableDiscoveryClient
@SpringBootApplication
public class MainApp8402 {
    public static void main(String[] args) {
        SpringApplication.run(MainApp8402.class, args);
    }
}
```



Controller

```java
@RestController
public class UserController {
    @Value("${server.port}")
    private String serverPort;

    @GetMapping("/user/{id}")
    public User getUserById(@PathVariable("id") int id) {
        return User.builder()
                .id(id)
                .name("dyf" + id)
                .age(id + 10)
                .serverPort(serverPort)
                .build();
    }
}
```



<font color=red>注意：准备两个 Server，测试 Client 调用的负载均衡的策略（轮训）。</font>



## 测试

1. 启动 Nacos 服务
2. 启动 两个 Server 服务
3. 启动 Client 服务



如下图：一个Client 和两个 Server 都成功注册进 Nacos 服务。

![](/images/spring_cloud/WX20230330-221236@2x.png)



如下图：通过返回的端口数据，我们可以发现 OpenFeign 具有负载均衡的功能。

![](/images/spring_cloud/WX20230330-221341@2x.png)

![](/images/spring_cloud/WX20230330-221359@2x.png)



# 超时设置

OpenFeign 默认的超时时间是<font color=red> 1 秒</font>。



开发一个超时接口：

```java
    @GetMapping("/user/timeout")
    public String timeout() {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return serverPort;
    }
```

如下图：有超时错误。

![](/images/spring_cloud/WX20230330-233306@2x.png)



在 Client 项目自定义 OpenFeign 超时

```yaml
#设置feign客户端超时时间(OpenFeign默认支持ribbon)
ribbon:
#指的是建立连接所用的时间，适用于网络状况正常的情况下,两端连接所用的时间
  ReadTimeout: 5000
#指的是建立连接后从服务器读取到可用资源所用的时间
  ConnectTimeout: 5000
```

请求正常了

![](/images/spring_cloud/WX20230330-233150@2x.png)



## 配置多个超时时间

yaml 文件中配置多个超时时间

```yaml
feign:
  client:
    config:
      default:
        # 日志级别
        loggerLevel: full
        # 超时设置
        connectTimeout: 1500
        readTimeout: 1500
      payment-core:
        connectTimeout: 5000
        readTimeout: 5000
```



FeignClient 调用时通过 contextId 属性指定使用哪个超时时间。

```java
@FeignClient(name = "payment-service", contextId = "payment-core",  path = "/payment")
public interface PaymentFeign {
    @PostMapping("/create")
    PaymentVo create(@RequestBody @Validated PaymentDto paymentDto);
}
```



# 日志配置

日志级别：

- NONE：默认的，不显示任何日志
- BASIC：仅记录请求方法、URL、响应状态码、执行时间。
- HEADERS：处理 BASIC 中定义的信息外，还会打印请求和响应头信息。
- FULL：除了 HEADERS 中定义的信息外，还会打印请求和响应的正文及元数据。



通过配置类：开启 FULL 日志级别。

```java
@Configuration
public class FeignConfig {
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```



配置哪些类需要打印日志

```yaml
logging:
  level:
    # feign日志以什么级别监控哪个接口
    com.dyf.springcloud.feign.service.FeignService: debug
```



打印日志如下图

![](/images/spring_cloud/WX20230330-234517@2x.png)



# 引入 Sentinel

引入 sentinel mvn 依赖

```xml
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>
```



激活 Sentinel 对 Feign 的支持

```yaml
feign:
  sentinel:
    enabled: true 
```



新增熔断限量的兜底策略类。

```java
@Component
public class FeignFallbackService implements FeignService {
    @Override
    public User getUserById(Long id) {
        return User.builder()
                .id(-1)
                .name("限流熔断兜底策略")
                .build();
    }

    @Override
    public String timeout() {
        return "限流熔断兜底策略";
    }
}
```



在业务类的 @FeignClient 注解上，新增 fallback 的兜底策略。

```java
@Component
@FeignClient(value = "cloudalibaba-sentinel-service", fallback = FeignFallbackService.class)
public interface FeignService {
    @GetMapping(value = "/user/{id}")
    User getUserById(@PathVariable("id") Long id);

    @GetMapping("/user/timeout")
    String timeout();
}
```



主启动类新增 @EnableDiscoveryClient 开启 Sentinel

```java
@SpringBootApplication
@EnableFeignClients
@EnableDiscoveryClient
public class FeignMain80 {
    public static void main(String[] args) {
        SpringApplication.run(FeignMain80.class, args);
    }
}
```



## 测试

1. 启动 Client 服务
2. 关闭 Server 服务，模拟 Server 服务异常。



如下图：已经走了兜底策略。

![](/images/spring_cloud/WX20230331-000841@2x.png)