---
title: Spring Cloud 微服务：Spring Cloud Config
date: 2017-08-15
desc:
keywords: spring cloud
categories: [spring cloud]
---

<img src="http://ohwsf74ph.bkt.clouddn.com/image/banner/spring-cloud-logo.jpeg">

前段时间工作太忙，博客好长时间暂停更新，正好现在空闲时间，把最近工作中学到的技术总结一下，分享给大家。

这篇文章是Spring Cloud系列的第一篇，首先从Spring Cloud Config这个项目说起吧，它是一个分布式配置中心服务，用于为分布式系统中各个微服务提供外部配置支持。

<!-- more -->

## Spring Cloud Config简介

Spring Cloud Config项目分为两部分，客户端和服务器（简单起见，我们统一称为Config Client和Config Server）。

Config Server是一个微服务应用，用于提供集中的外部配置支持，也就是我们前面提到的分布式配置中心。Config Server支持连接外部的配置仓库来管理各个微服务的配置文件，配置仓库可以是Git、Svn或本地文件系统。此外，Config Server还支持配置文件加密/解密、多仓库支持、动态刷新配置、自动推送配置等，这篇文章将会逐一介绍这些特性。

Config Client是用于连接到Config Server的客户端，它也是一个微服务应用，Config Client在启动的时候从Config Server中获取对于的配置信息，然后加载到Spring容器中。

下面我们来创建并启动一个Config Server，看看它是如何运行的。

## 快速开始

### 构建Config Server

可以在[http://start.spring.io/](http://start.spring.io/)上生成一个Spring Boot项目，也可以使用Spring Tool Suite或IntelliJ IDEA等IDE创建，支持Maven和Gradle两种构建工具，这里我选择Gradle作为构建工具，采用Gradle多模块方式构建。

创建一个`springcloud-config-server`的子项目，并在`build.gradle`文件中引入`spring-cloud-config-server`依赖：

```groovy
dependencies {
    compile('org.springframework.cloud:spring-cloud-config-server')
}
```

然后在Spring Boot的启动主类中添加`@EnableConfigServer`注解，表示开启Config Server功能：

```java
package org.matrixstudio.springcloud.config;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

在`application.properties`配置文件中添加仓库配置信息：

```
spring.cloud.config.server.git.uri=https://github.com/lw900925/springcloud-config-repository.git
spring.cloud.config.server.git.search-paths=base-service
#spring.cloud.config.server.git.username=username
#spring.cloud.config.server.git.password=password
spring.cloud.config.server.git.timeout=60
```

** 配置详解 **
- `spring.cloud.config.server.git.uri`：Config Server需要连接的Git仓库地址
- `spring.cloud.config.server.git.search-paths`：Config Server所连接仓库的搜索路径（你可以将不同类型服务的配置文件分别放在不同的目录中，方便管理，然后通过指定该属性让Config Server找到它们）
- `spring.cloud.config.server.git.username`：连接到Git仓库的用户名（这里使用的仓库未public类型，所以不需要用户名和密码）
- `spring.cloud.config.server.git.password`：连接到Git仓库的密码
- `spring.cloud.config.server.git.timeout`：连接到Git的超时时间，单位为秒（second），默认值为5

至此，Config Server服务搭建完毕了，现在可以在`springcloud-config-server`目录下通过执行`gradle bootRun`命令运行项目。

### 准备配置文件

在上面的`application.properties`配置文件中指定了一个Github仓库地址`https://github.com/lw900925/springcloud-config-repository.git`作为配置仓库，我们需要在该仓库中添加一些配置文件以便Config Server启动后可以访问到它们。

首先在仓库中创建一个`base-service`的目录，并在该目录下添加两个配置文件：

```
springcloud-file-service-dev.properties
springcloud-file-service-test.properties
```

上面两个配置文件分别表示名称为`springcloud-file-service`的微服务应用的开发环境和测试环境的配置文件（`springcloud-file-service`微服务将在后面构建），配置文件的内容为：

```
spring.application.name=springcloud-file-service

spring.profiles.active=dev
```

*** 注 ***：`springcloud-file-service-test.properties`文件中的`spring.profiles.active`为`test`。

文件添加完毕后，推送到远程仓库，就可以启动Config Server来测试了，这里使用了`8888`作为Config Server的端口，我们可以通过`curl`命令或浏览器地址栏来测试Config Server从远程Git仓库获取配置文件：

```
curl http://localhost:8888/springcloud-file-service/dev | python -mjson.tool
```

上面的命令在Windows环境下执行需要安装Python运行环境和curl工具，如果不想安装也可以通过浏览器地址访问。上述命令执行结果为：

```json
{
    "name": "springcloud-file-service",
    "profiles": [
        "dev"
    ],
    "label": null,
    "version": null,
    "state": null,
    "propertySources": [
        {
            "name": "https://github.com/lw900925/springcloud-config-repository.git/base-service/springcloud-file-service-dev.properties",
            "source": {
                "spring.profiles.active": "dev",
                "spring.application.name": "springcloud-file-service",
            }
        }
    ]
}
```

也许你发现在请求的URL里包含了配置文件的名称和环境信息，Config Server正是通过这两个参数定位具体的配置文件的，Config Server会将配置文件的URL映射出来（可以在Config Server的启动日志中找到映射信息），它们的对应关系如下：

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

`application`表示微服务的名称，也就是上面请求URL中的`springcloud-file-service`；`profile`表示对于的环境，即`dev`；`label`是可选参数，表示Git分支名称，因为Config Server默认从`master`分支作获取配置文件，所以该参数可以省略，如果配置文件放在其他分支，就需要指定该参数值。

### 构建Config Client

通过Config Server我们已经可以手动获取对应微服务应用的配置文件了，接下来通过在微服务中获取配置文件。

首先需要创建一个项目，命名为`springcloud-file-service`，并在`build.gradle`文件中添加`spring-cloud-starter-config`依赖：

```groovy
dependencies {
    compile('org.springframework.cloud:spring-cloud-starter-config')

    compile('org.springframework.boot:spring-boot-starter-actuator')
}
```

因为项目需要在启动的时候获取配置信息，所以需要添加`bootstrap.properties`配置文件：

```
spring.application.name=springcloud-file-service

spring.profiles.active=dev

# Spring cloud config client
spring.cloud.config.label=master
spring.cloud.config.uri=http://localhost:8888
```

启动`springcloud-file-service`应用，通过访问`/env.json`路径，可以看到配置信息已经获取到：

```
{
    "profiles": [
        "dev"
    ],
    "server.ports": {
        "local.server.port": "8000"
    },
    "configService:https://github.com/lw900925/springcloud-config-repository.git/base-service/springcloud-file-service-dev.properties": {
        "spring.profiles.active": "dev",
        "spring.application.name": "springcloud-file-service"
    },

    // 以下部分省略
}
```

## Config Server多仓库支持

由于一些复杂的需求，配置文件往往分布在不同的Git仓库中，Config Server支持多仓库的配置，我们可以将生产、开发、测试环境的配置文件分别放在不同的Git仓库，需要在`springcloud-config-server`应用中的`application.properties`配置文件中添加如下配置：

```
# Spring cloud config server
spring.cloud.config.server.git.uri=https://github.com/lw900925/springcloud-config-repository.git
spring.cloud.config.server.git.search-paths=base-service
#spring.cloud.config.server.git.username=username
#spring.cloud.config.server.git.password=password
spring.cloud.config.server.git.timeout=60

spring.cloud.config.server.git.repos.prod.uri=https://gitlab.com/lw900925/springcloud-config-repository.git
spring.cloud.config.server.git.repos.prod.pattern=*prod*
spring.cloud.config.server.git.repos.prod.searchPaths=base-service
spring.cloud.config.server.git.repos.prod.timeout=60
```

通过`spring.cloud.config.server.git.repos.*`配置不同的仓库，上述配置文件中添加了一个`prod`的库作为生产环境的Github库，当获取`prod`环境的配置文件时，Config Server首先会在该库中获取配置文件，如果没有获取到，就会进入默认的库中获取。

## 安全配置

由于Config Server中的配置文件信息比较敏感，如果没有做限制，任何人可以通过Config Server获取配置信息，会有很大隐患。配置Config Server以安全方式访问有很多种方法，如OAuth2.0认证，防火墙白名单等，这里推荐使用Spring Security项目，他可以和Spring Boot项目无缝整合。

### Config Server配置

在`springcloud-config-server`项目中的`build.gradle`文件添加`spring-boot-starter-security`依赖：

```groovy
dependencies {
    compile('org.springframework.boot:spring-boot-starter-security')
}
```

Spring Security默认使用HTTP Basic方式认证，当启动Config Server时，可以在控制台看到如下日志信息：

```
2017-08-15 16:19:27.319  INFO 13720 --- [           main] b.a.s.AuthenticationManagerConfiguration : 

Using default security password: 5cd88fb5-0f77-4560-a3fe-7815f742862b
```

此时通过浏览器访问Config Server会提示输入用户名和密码，用户名默认为`user`，密码即上面日志中输出的UUID，每次重新启动Config Server时会重新生成，如果不希望每次启动Config Server都重新生成密码，可以在`application.properties`中修改：

```
security.user.name=spring
security.user.password=password
```

### Config Client配置

在`springcloud-file-service`项目的`bootstrap.properties`文件中添加Config Server的认证信息：

```
spring.cloud.config.username=user
spring.cloud.config.password=password
```

## 加密/解密

一些敏感的配置信息（如数据库连接信息）如果以明文方式存放在配置文件中，一旦泄露，将会造成不可挽回的损失。Config Server支持配置信息加密，加密后以密文存储在配置文件中，保证配置信息的安全。

使用加密/解密特性需要JCE（Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files）的支持，默认的JCE在Sun的jre版本中自带，不过有长度限制，使用时可能会抛出`java.security.InvalidKeyException: Illegal key size`异常信息，所以我们需要安装Oracle提供的不限长度的JCE，下载传送门：

- [Java 6 JCE](http://www.oracle.com/technetwork/java/javase/downloads/jce-6-download-429243.html)
- [Java 7 JCE](http://www.oracle.com/technetwork/java/javase/downloads/jce-7-download-432124.html)
- [Java 8 JCE](http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)

请根据所使用的JDK版本对号入座，下载后是一个zip包，解压到JDK安装目录下`/jre/lib/security`目录中覆盖原来的即可。

安装好JCE后，可以开启Config Server，在控制台中查看启动日志，可以发现加密/解密的相关端点（Endpoint）：

- `[/encrypt],methods=[POST]`
- `[/encrypt/{name}/{profiles}],methods=[POST]`
- `[/decrypt/{name}/{profiles}],methods=[POST]`
- `[/decrypt],methods=[POST]`
- `[/encrypt/status],methods=[GET]`
- `[/key],methods=[GET]`
- `[/key/{name}/{profiles}],methods=[GET]`

此时如果访问`/encrypt/status`端点会返回如下信息：

```json
{
    "description": "No key was installed for encryption service",
    "status": "NO_KEY"
}
```

这是因为没有配置密钥，在`application.properties`配置文件中添加加密的密钥：

```
encrypt.key=config_server_key
```

添加完成后再次访问`/encrypt/status`端点，显示加密功能可用：

```json
{
    "status": "OK"
}
```

现在访问加密解密端点查看能否正常加解密：

```
curl http://user:password@localhost:8888/encrypt -d password
7b113712cc107539d41c4e4343e13da38411f2c66d916841a273eb58a6585818

curl http://user:password@localhost:8888/decrypt -d 7b113712cc107539d41c4e4343e13da38411f2c66d916841a273eb58a6585818
password
```

除了使用密钥加密外，Config Server还支持证书加密，这种加密方式使用起来有点复杂，但比使用密钥的方式更安全。首先，需要用JDK自带的`keytool`命令生成证书：

```
keytool -genkeypair -alias spring-cloud-config-server -keyalg RSA \
  -dname "CN=zh-CN, OU=spring.io, O=spring, L=Shanghai, S=Shanghai, C=CN" \
  -keypass 123456 -keystore .keystore -storepass 123456
```

默认会在当前用户目录下生成`.keystore`证书文件，将该文件拷贝到`springcloud-config-server`项目的`/src/main/resources`目录下，然后在`application.properties`中添加以下配置：

```
encrypt.key-store.location=classpath:.keystore
encrypt.key-store.alias=spring-cloud-config-server
encrypt.key-store.password=123456
encrypt.key-store.secret=123456
```

重新启动Config Server，访问加密解密端点：

```
curl http://user:password@localhost:8888/encrypt -d password
AQBp7YRFbAyDkoVtrxymRrmhzfLfxuVIfvhLHcRT/uRbfJn87TRkwds509uq7BbMxGn2xqCTeLNguN0uBHxVHETlHsh8PabJuVwZAsRFw0Q9nXbU+KdLiaUroLIbtqpQgKltzq91lsKSciXauX5JGzOqgkJhy81UfCgJFeR4m3EmWm8b/EV8Bs0KVekkHCMyyxZwja2wABDyd9BRX4mMCzNuFQ5bHT3RKhgbFWuwewSJC3dFhmLhUYczuT2xqTm/0rp6+zq22tugr9ils9814p2JGpz8o+fQc86i6nOzWKl2+ab3E3aLdG/7tMWHhv6mEmO5/dvcqFz4BHdeN6slsNaWedrxO9hkcWlK7nxwIKLVahutN42n+8a1hg3A+lsp8PY=

curl http://user:password@localhost:8888/decrypt -d AQBp7YRFbAyDkoVtrxymRrmhzfLfxuVIfvhLHcRT/uRbfJn87TRkwds509uq7BbMxGn2xqCTeLNguN0uBHxVHETlHsh8PabJuVwZAsRFw0Q9nXbU+KdLiaUroLIbtqpQgKltzq91lsKSciXauX5JGzOqgkJhy81UfCgJFeR4m3EmWm8b/EV8Bs0KVekkHCMyyxZwja2wABDyd9BRX4mMCzNuFQ5bHT3RKhgbFWuwewSJC3dFhmLhUYczuT2xqTm/0rp6+zq22tugr9ils9814p2JGpz8o+fQc86i6nOzWKl2+ab3E3aLdG/7tMWHhv6mEmO5/dvcqFz4BHdeN6slsNaWedrxO9hkcWlK7nxwIKLVahutN42n+8a1hg3A+lsp8PY=
password
```

现在可以将加密后的密文存储在配置文件中，密文在配置文件中需要以`{cipher}`标记开头，Config Server在获取到配置文件时检查变量值如果以`{cipher}`开头，就认为是一段密文，会对其进行解密。这里可以为`springcloud-file-service`配置数据库的密码：

```
# Spring datasource
spring.datasource.url=jdbc:mysql://localhost:3306/file-service
spring.datasource.username=root
spring.datasource.password={cipher}AQBrf7mTx037Xt6r6gKV0LO63RdZJSMHyVDgE9hQqb9ZUtij9XtW0ZSFaD1oiCmSVBTNtU8/0yXnahwuOS2sWptOAdPRLkXwG3BjZFuOt50cV/wiU6OkdShPshsOEHgI7OSWXjXqpv57JyptgId91iFx7eRyQARaYZKlJenh5RCdDGGiBD+cf41d3EtuuEp3IiWnw9zA9QplTjnO3+zgWgRvAPPtcqt0sII9Fk62241w+TPrPxHeJK+HKnoqFG+DJGvTC9PxNqWD1i/v5dFEpse4TSWUMpqXJsi3y+JUTcL+rBwR+I9NoOS81EA4R1sFBrocpFi43rEltG4wppES5ZBREnz+SdHWYa8RzFDajXP38tjFu6lmZj5sCjLxsY7+o18=
```

将配置文件推送到远程Git仓库，启动Config Server即可。

## 动态刷新配置

有时候我们修改某个微服务的配置文件，推送到远程Git仓库，然后将该微服务重新启动，以便让它从Config Server获取最新的配置文件，但如果改为服务是集群部署（比如有10个实例的集群），逐个重启的做法效率太低。好在Spring Cloud Config为我们提供了动态刷新某个微服务配置的支持，只要在该微服务上访问刷新配置的端点即可。

动态刷新需要添加`spring-boot-starter-actuator`依赖，在`springcloud-file-service`项目中`build.gradle`添加如下依赖：

```groovy
dependencies {
    compile('org.springframework.boot:spring-boot-starter-actuator')
}
```

启动`springcloud-file-service`项目后在控制台查看启动日志可以发现多了一个`/refresh`的端点，该端点就是刷新配置，首先我们访问`/env.json`端点查看远程仓库的配置信息：

```json
{
    "profiles": [
        "dev"
    ],
    "server.ports": {
        "local.server.port": "8000"
    },
    "configService:https://github.com/lw900925/springcloud-config-repository.git/base-service/springcloud-file-service-dev.properties": {
        "spring.datasource.username": "root",
        "spring.profiles.active": "dev",
        "spring.datasource.url": "jdbc:mysql://localhost:3306/file-service",
        "spring.application.name": "springcloud-file-service",
        "spring.datasource.password": "******"
    },

    // 以下信息省略
}
```

修改`spring.datasource.url`为`jdbc:mysql://127.0.0.1:3306/file-service`，并推送到远程Git仓库，然后访问`springcloud-file-service`的`/refresh`端点：

```json
["spring.datasource.url"]
```

输出信息表示`spring.datasource.url`的值已刷新，再次请求`/env.json`端点：

```json
{
    "profiles": [
        "dev"
    ],
    "server.ports": {
        "local.server.port": "8000"
    },
    "configService:https://github.com/lw900925/springcloud-config-repository.git/base-service/springcloud-file-service-dev.properties": {
        "spring.datasource.username": "root",
        "spring.profiles.active": "dev",
        "spring.datasource.url": "jdbc:mysql://127.0.0.1:3306/file-service",
        "spring.application.name": "springcloud-file-service",
        "spring.datasource.password": "******"
    },

    // 以下信息省略
}
```

可以看到`spring.datasource.url`值已经变成修改后的值，这期间`springcloud-file-service`没有重启，即实现了配置热加载。

## 自动推送配置

前面介绍了在不重启微服务的情况下刷新配置文件，不过这种方式依然比较繁琐，即使不重启微服务，也是需要手动请求每个微服务的`/refresh`端点，如果集群中运行的实例较多，这种操作枯燥而又费时，也许可以使用脚本批量操作，不过Spring Cloud Config为我们提供了更自动化的方式。

自动推送配置需要配合Spring Cloud Bus项目实现，Spring Cloud Bus为微服务提供消息总线功能，关于Spring Cloud Bus项目将在后面的文章中做详细介绍，这里我们可以先用起来。整个流程大致如下：

Spring Cloud Bus有两种实现，RabbitMQ和Kafka，此处选用RabbitMQ作为实现，需要分别在`springcloud-config-server`和`springcloud-file-service`项目中添加依赖：

```groovy
dependencies {
    compile('org.springframework.cloud:spring-cloud-starter-bus-amqp')
}
```

然后分别修改两个项目的`application.properties`配置文件，添加RabbitMQ配置（需要安装RabbitMQ，具体请参考RabbitMQ官方文档）：

```
# Spring RabbitMQ
spring.rabbitmq.host=10.145.4.171
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.virtual-host=/
```

最后重启两个项目，发现启动日志中新增了`/bus/refresh`端点，该端点接受一个`destination`作为参数，`destination`是配置文件中`spring.application.name`的值，请求格式为：

```
POST http://user:password@localhost:8888/bus/refresh?destination=springcloud-file-service:**
POST http://user:password@localhost:8888/bus/refresh?destination=springcloud-file-service:8000
```

`destination`指定某个微服务的名称，如果只刷新某几个实例的配置，直接指定端口号即可，`**`表示刷新所有微服务实例的配置。

一些代码仓库服务（如Gitlab、Githun、Bitbucket等）都提供了发送通知消息的功能（通过webhook实现），当本地仓库的变更推送到远程仓库时，远程仓库会发送请求到指定的URL中，该URL通常是Config Server的地址。当Config Server接收到远程仓库发送的请求，会对其进行解析，然后更新微服务的配置。

要让自动推送的功能正常运行，需要在`springcloud-config-server`项目添加`spring-cloud-config-monitor`依赖：

```groovy
dependencies {
    compile('org.springframework.cloud:spring-cloud-config-monitor')
}
```

添加完成后重启Config Server，可以在启动日志中看到新增`/monitor`端点，该端点就是接收Git远程仓库通知的端点。由于Config Server配置了Spring Security的安全认证，还需要在`application.properties`中添加配置让Spring Security排除掉`/monitor`端点：

```
security.ignored=/monitor
```

然后在Git仓库中配置webhook的Payload URL为`http://112.65.18.61:8000/monitor`（如何配置wenhook请自行Google or Baidu），并确保勾选Push events。

如果不想排除`/monitor`端点，也可以将验证信息配置在Payload URL中，例如`http://user:password@112.65.18.61:8000/monitor`（不太推荐这种做法）。

> ** 注意 **：Playload URL指向Config Server的服务，请将该服务的IP和端口映射到外网，确保Github或Gitlab可以访问到。如果使用自建的Gitlab，也请确保该URL能被Gitlab访问到。

最后，附上项目源码地址：

- spring-cloud：[https://github.com/lw900925/springcloud](https://github.com/lw900925/springcloud)
- spring-cloud-config-repository (github): [https://github.com/lw900925/springcloud-config-repository](https://github.com/lw900925/springcloud-config-repository)
- spring-cloud-config-repository (gitlab): [https://gitlab.com/lw900925/springcloud-config-repository](https://gitlab.com/lw900925/springcloud-config-repository)

如果有疑问，请在下方评论区参与讨论。