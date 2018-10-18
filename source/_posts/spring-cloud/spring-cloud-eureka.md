---
title: Spring Cloud 微服务：Spring Cloud Eureka
date: 2017-08-17
desc:
keywords: spring cloud
categories: [spring cloud]
---

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/banner/spring-cloud-logo.jpeg">

这篇主要介绍Spring Cloud Netflix套件中比较重要的模块——Spring Cloud Eureka，使用Spring Cloud Eureka搭建注册中心，并配置为高可用，以及将其他微服务注册到Eureka上。

<!-- more -->

## 服务注册与发现

想象一下我们在之前的开发模式中，如果某个服务需要调用另一个服务提供的接口，我们需要知道被调用服务的IP地址，该IP地址必须是固定的，通常会将该IP写在配置文件中维护。不过，在微服务架构模式中，所有微服务实例都是集群部署，如果还按照手动维护配置文件中的服务IP地址，很容易出错，而且维护工作量大，这就需要一个统一维护服务IP清单的注册中心。

服务注册通常都会依赖注册中心，注册中心是服务发现的关键部分，它是包含各个服务实例网络地址的数据库。注册中心要保证高可用和及时更新，客户端定期从注册中心获取最新的服务清单。注册中心还需要定期检测清单中的服务是否可用，将不可用的服务剔除。

服务的发现主要有两种：客户端发现和服务端发现。Netflix OSS套件中的Eureka就属于客户端发现，客户端从注册中心获取可用的服务清单，然后使用负载均衡算法向其中某个服务发起请求；另一种是服务端发现，客户端通过负载均衡向指定服务发起请求，负载均衡首先查询注册中心，将请求转发到可用的服务上。

## Spring Cloud Eureka简介

Spring Cloud Eureka是Spring对Netflix Eureka的封装，使用Spring Boot风格的配置，开发人员只需要引入相关依赖，做一些简单配置就可以实现Eureka的功能。Spring Cloud Eureka分为两部分，Eureka Server和Eureka Client。

Eureka Server也就是我们上面提到的注册中心，和大多数注册中心框架一样，Eureka支持集群方式部署，以此达到高可用。Eureka的高可用是通过各个副本（Replica）互相注册来实现的，我们随后会对这一特性展开讨论。

Eureka Client主要提供服务的注册和发现，通过引入依赖的方式嵌入到微服务项目中，当Eureka Client启动时，会将自己注册到注册中心，并定期向注册中心发送心跳（Heartbeat）请求来续约。同时，Eureka Client还可以定期从注册中心查询可用的服务清单，缓存到本地并定期刷新。

## 开始构建

### Eureka Server

新建一个Spring Boot工程`springcloud-eureka`，在`build.gradle`中引入依赖`spring-cloud-starter-eureka-server`依赖：

```groovy
dependencies {
    compile('org.springframework.cloud:spring-cloud-starter-eureka-server')
}
```

在Spring Boot的启动类上添加`@EnableEurekaServer`注解：

```java
package org.matrixstudio.springcloud.eureka;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }
}
```

在`application.properties`文件中添加Eureka的配置：

```
# Spring application
spring.application.name=springcloud-eureka

# Server port
server.port=8761

management.security.enabled=false

# Spring cloud eureka
eureka.instance.hostname=springcloud-eureka
eureka.instance.prefer-ip-address=true

eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.service-url.defaultZone=http://localhost:${server.port}/eureka/
```

然后启动`springcloud-eureka`项目，在浏览器中输入`http://localhost:8761/`就可以看到Eureka的Dashboard页面了。

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/_post/eureka-server-dashboard-01.png">

可以看到，在Instances currently registered with eureka下面的表格中没有数据，这是因为还没有服务注册到注册中心。

注册中心已经搭建完成，下面我们将微服务注册到Eureka Server上。

### Eureka Client

在上一片文章[Spring Cloud 微服务：Spring Cloud Config](https://lw900925.github.io/spring-cloud/spring-cloud-config.html)中我们创建了一个`springcloud-file-service`的项目作为Config Client，现在我们在该项目上添加Eureka Client的支持，使其注册到Eureka Server注册中心上。

首先需要引入依赖，`springcloud-file-service`项目中加入`spring-cloud-starter-eureka`起步依赖：

```groovy
dependencies {
    compile('org.springframework.cloud:spring-cloud-starter-eureka')
}
```

然后修改`springcloud-file-service`项目的Spring Boot启动类，添加`@EnableDiscoveryClient`注解，开启Eureka Client功能：

```java
package org.matrixstudio.springcloud.file;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class FileServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(FileServiceApplication.class, args);
    }
}
```

还需要在`application.properties`文件中添加Eureka Server配置：

```
# Spring application
spring.application.name=springcloud-file-service

# Server port
server.port=8000

management.security.enabled=false

# Spring cloud eureka discovery
eureka.instance.prefer-ip-address=true

eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

然后分别启动`springcloud-file-service`和`springcloud-eureka`项目，查看Eureka Server的Dashboard界面。

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/_post/eureka-server-dashboard-02.png">

可以看到Instances currently registered with eureka下方的表格内多了一个`SPRINGCLOUD-FILE-SERVICE`的服务，说明`springcloud-file-service`已经注册到Eureka Server上了。

也可以将`springcloud-file-service`部署多个实例，都注册到Eureka Server上，提高容错率（可以通过指定`profile`方式启动多个实例）。

```
java -jar springcloud-file-service-0.0.1-SNAPSHOT.jar --server.port=8000
java -jar springcloud-file-service-0.0.1-SNAPSHOT.jar --server.port=8001
```

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/_post/eureka-server-dashboard-06.png">

查看Eureka Dashboard，发现列表中有两个`springcloud-file-service`的实例，端口分别为`8080`和`8081`。

## 的高可用

前面提到了注册中心的高可用，Eureka也可以配置为高可用注册中心。前面我们构建Eureka Server的时候，`application.properties`里有两个配置：

```
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

- `eureka.client.register-with-eureka`：表示是否将注册到Eureka Server
- `eureka.client.fetch-registry`：表示是否获取注册中心的服务注册信息

这两个配置默认值都是`true`，在构建Eureka Server时，由于Eureka Server是单机部署，所以Eureka不需要将自己注册到注册中心，也不需要获取服务清单。如果是集群部署，需要将这两个配置修改为默认值（或不指定这两个配置，默认就是`true`），Eureka Server通过集群中的副本之间相互注册，相互同步服务注册信息来达到高可用。

我们需要准备两个配置文件`application-replica1.properties`和`application-replica2.properties`，两个文件内容分别如下：

- `application-replica1.properties`：

```
# Spring application
spring.application.name=springcloud-eureka

server.port=8761

# Spring cloud eureka instance
eureka.instance.hostname=replica1

# Spring cloud eureka client
eureka.client.service-url.defaultZone=http://replica2:8762/eureka/
```

- `application-replica2.properties`：

```
# Spring application
spring.application.name=springcloud-eureka

server.port=8762

# Spring cloud eureka instance
eureka.instance.hostname=replica2

# Spring cloud eureka client
eureka.client.service-url.defaultZone=http://replica1:8761/eureka/
```

*** 注：*** 需要在`hosts`文件中添加`replica1`和`replica2`的映射，让Eureka能通过`hosts`访问的到。

```
# Eureka Replica Start
127.0.0.1 replica1
127.0.0.1 replica2
# Eureka Replica End
```

分别将Eureka的两个副本启动起来：

```
java -jar springcloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=replica1
java -jar springcloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=replica2
```

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/_post/eureka-server-dashboard-03.png">

浏览器输入`http://localhost:8761/`查看`replica1`副本的Dashboard页面，注意页面中DS Replicas下方的列表，此时多出一个`replica2`的副本，然后查看另一个副本的Dashboard页面，发现DS Replicas下方多出了`replica2`的副本。

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/_post/eureka-server-dashboard-04.png">

此时可以将`springcloud-file-service`服务注册到Eureka Server集群上，修改`application.properties`配置文件如下：

```
# Spring application
spring.application.name=springcloud-file-service

# Server port
server.port=8000

management.security.enabled=false

# Spring cloud eureka discovery
eureka.instance.prefer-ip-address=true

eureka.client.service-url.defaultZone=http://replica1:8761/eureka/,http://replica2:8762/eureka/
```

## 将配置中心注册在Eureka上

在上一片文章[Spring Cloud 微服务：Spring Cloud Config](https://lw900925.github.io/spring-cloud/spring-cloud-config.html)中我们介绍了Spring Cloud Config的配置中心——Config Server，Config Server也可以看作一个微服务，既然是微服务，应该要支持高可用。

Config Server高可用有两种部署方式，一种是直接部署成多个实例，通过Nginx等反向代理工具做负载均衡；还有一种就是通过注册到Eureka Server上实现高可用，配置方法和Eureka Client配置方法一致。

需要注意的是，如果将Config Server注册到Eureka Server上，Config Client端获取配置的方式需要指定为从Eureka Server的服务清单中获取。在`springcloud-file-service`的`application.properties`文件中添加下面的配置：

```
spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.service-id=springcloud-config-server
```

最后，附上项目源码地址：

- spring-cloud：[https://github.com/lw900925/springcloud](https://github.com/lw900925/springcloud)
- spring-cloud-config-repository (github): [https://github.com/lw900925/springcloud-config-repository](https://github.com/lw900925/springcloud-config-repository)
- spring-cloud-config-repository (gitlab): [https://gitlab.com/lw900925/springcloud-config-repository](https://gitlab.com/lw900925/springcloud-config-repository)

如果有疑问，请在下方评论区参与讨论。
