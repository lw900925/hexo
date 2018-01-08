---
title: Docker：Spring Boot应用发布到Docker
date: 2017-03-08
desc:
keywords: Docker
categories: [docker]
---

<img src="http://ohwsf74ph.bkt.clouddn.com/image/banner/docker-logo.jpeg">

Spring官网上有一篇[Getting Start][934425c5]，介绍了如何使用Docker发布Spring Boot应用，算是比较详细了，不过有些细节没有提及到，而且官网的入门手册是英文版。这里重新整理记录一下，算是给英文不好的小伙伴一个参考，也给自己留个备忘。

<!-- more -->

## 准备

需要的工具以及运行环境：

- JDK 1.8 or later
- Maven 3.0 +
- 你喜欢的IDE或其他文本编辑器

## 创建工程

首先，你需要创建一个Spring Boot工程，Spring Tool Suite和IntelliJ IDEA都自带插件可以创建，还有一种方式是从[http://start.spring.io/][1f5046f6]上创建，推荐使用这种方式。填好表单中的`Group Id`和`Artifact Id`之后，点击Generate Project按钮就可以生成了，将下载后的工程导入到你喜欢的IDE中。

修改`pom.xml`文件，添加`docker-maven-plugin`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.matrixstudio.springboot</groupId>
    <artifactId>docker</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>docker</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <docker.image.prefix>springio</docker.image.prefix>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

            <!-- Docker maven plugin -->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>0.4.3</version>
                <configuration>
                    <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
                    <dockerDirectory>src/main/docker</dockerDirectory>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

`docker-maven-plugin`插件用于将Spring Boot工程构建为Docker镜像：

- `imageName`表示Docker镜像名称，我们使用Docker的命名规范，命名为：`springio/docker`
- `dockerDirectory`表示Dockerfile的路径
- `resource`表示在构建时需要的资源文件，这些文件和Dockerfile放在一起，这里只需要Spring Boot生成的jar文件即可。

打开`DockerApplication.java`文件，修改成如下内容：

```java
package org.matrixstudio.springboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class DockerApplication {

    @RequestMapping("/")
    public String home() {
        return "Hello Docker World";
    }

    public static void main(String[] args) {
        SpringApplication.run(DockerApplication.class, args);
    }
}
```

## 编译并运行

执行如下命令运行Spring Boot工程：

```bash
mvn package && java -jar target/docker-0.0.1-SNAPSHOT.jar
```

打开浏览器，输入`http://localhost:8080`，如果出现“Hello Docker World”说明运行成功。

**注**：运行上面的命令时，需要从Maven官方仓库中下载很多依赖包，国内网络不太稳定，下载速度较慢，可考虑使用第三方提供的镜像站，比如阿里的Maven镜像仓库。在`pom.xml`中加入下面配置：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.matrixstudio.springboot</groupId>
    <artifactId>docker</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <!-- Dependencies -->
    ......

    <!-- Build -->
    ......

    <!-- Aliyun repository -->
    <repositories>
        <repository>
        <id>central</id>
            <name>aliyun</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
        </repository>
    </repositories>
</project>
```

## 容器化项目

首先要确保你的机器上安装了Docker，如果你的Docker安装在一台Linux服务器上，你需要将上面的Spring Boot工程上传到该服务器上，下面的步骤假设你是在Linux环境上操作。

### 创建Dockerfile

Docker使用一个名为`Dockerfile`的文件来指定image层，所以我们首先需要创建一个`Dockerfile`文件，执行下面的命令创建`Dockerfile`文件：

```bash
sudo tee src/main/docker/Dockerfile <<-'EOF'
FROM frolvlad/alpine-oraclejdk8:slim
VOLUME /tmp
ADD docker-0.0.1-SNAPSHOT.jar app.jar
RUN sh -c 'touch /app.jar'
ENV JAVA_OPTS=""
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]
EOF
```

大概解释一下上面的命令：

- `VOLUME`指向了一个`/tmp`的目录，由于Spring Boot使用内置的Tomcat容器，Tomcat默认使用`/tmp`作为工作目录。效果就是在主机的`/var/lib/docker`目录下创建了一个临时文件，并连接到容器的`/tmp`。
- 将项目的jar文件作为`app.jar`添加到容器
- `RUN`表示在新创建的镜像中执行一些命令，然后把执行的结果提交到当前镜像。这里使用`touch`命令来改变文件的**修改时间**，Docker创建的所有容器文件默认状态都是“未修改”。这对于简单应用来说不需要，不过对于一些静态内容（比如：`index.html`）的文件就需要一个“修改时间”。
- 为了缩短Tomcat的启动时间，我们添加一个`java.security.egd`的系统属性指向`/dev/urandom`作为`ENTRYPOINT`。

### 构建Docker镜像

运行下面的命令构建Docker镜像：

```bash
mvn package docker:build
```

构建完成后，运行下面的命令查看：

```bash
sudo docker images
```

结果为：

```
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
springio/docker              latest              7e2ba2f7e81e        2 minutes ago       195 MB
frolvlad/alpine-oraclejdk8   slim                00d8610f052e        4 days ago          167 MB
```

可以看到我们构建的镜像已经出现了，下一步就是运行该镜像。

### 运行Docker镜像

执行下面的命令来运行上一步构建的Docker镜像：

```bash
sudo docker run -p 8080:8080 -t springio/docker
```

如果不出意外，你将看到下面的输出内容：

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.5.2.RELEASE)

2017-03-08 03:34:59.434  INFO 6 --- [           main] o.m.springboot.DockerApplication         : Starting DockerApplication v0.0.1-SNAPSHOT on 00eed53e6928 with PID 6 (/app.jar started by root in /)
2017-03-08 03:34:59.445  INFO 6 --- [           main] o.m.springboot.DockerApplication         : No active profile set, falling back to default profiles: default
2017-03-08 03:34:59.752  INFO 6 --- [           main] ationConfigEmbeddedWebApplicationContext : Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@4b9af9a9: startup date [Wed Mar 08 03:34:59 GMT 2017]; root of context hierarchy
2017-03-08 03:35:03.755  INFO 6 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat initialized with port(s): 8080 (http)
2017-03-08 03:35:03.807  INFO 6 --- [           main] o.apache.catalina.core.StandardService   : Starting service Tomcat
2017-03-08 03:35:03.821  INFO 6 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/8.5.11
2017-03-08 03:35:04.042  INFO 6 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2017-03-08 03:35:04.043  INFO 6 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 4303 ms
2017-03-08 03:35:04.441  INFO 6 --- [ost-startStop-1] o.s.b.w.servlet.ServletRegistrationBean  : Mapping servlet: 'dispatcherServlet' to [/]
2017-03-08 03:35:04.455  INFO 6 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'characterEncodingFilter' to: [/*]
2017-03-08 03:35:04.457  INFO 6 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
2017-03-08 03:35:04.468  INFO 6 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'httpPutFormContentFilter' to: [/*]
2017-03-08 03:35:04.468  INFO 6 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'requestContextFilter' to: [/*]
2017-03-08 03:35:05.110  INFO 6 --- [           main] s.w.s.m.m.a.RequestMappingHandlerAdapter : Looking for @ControllerAdvice: org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@4b9af9a9: startup date [Wed Mar 08 03:34:59 GMT 2017]; root of context hierarchy
2017-03-08 03:35:05.390  INFO 6 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/]}" onto public java.lang.String org.matrixstudio.springboot.DockerApplication.home()
2017-03-08 03:35:05.402  INFO 6 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error],produces=[text/html]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.boot.autoconfigure.web.BasicErrorController.errorHtml(javax.servlet.http.HttpServletRequest,javax.servlet.http.HttpServletResponse)
2017-03-08 03:35:05.404  INFO 6 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error]}" onto public org.springframework.http.ResponseEntity<java.util.Map<java.lang.String, java.lang.Object>> org.springframework.boot.autoconfigure.web.BasicErrorController.error(javax.servlet.http.HttpServletRequest)
2017-03-08 03:35:05.512  INFO 6 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/webjars/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-03-08 03:35:05.512  INFO 6 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-03-08 03:35:05.639  INFO 6 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**/favicon.ico] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-03-08 03:35:06.019  INFO 6 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
2017-03-08 03:35:06.168  INFO 6 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8080 (http)
2017-03-08 03:35:06.183  INFO 6 --- [           main] o.m.springboot.DockerApplication         : Started DockerApplication in 7.893 seconds (JVM running for 8.743)
2017-03-08 03:35:56.728  INFO 6 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring FrameworkServlet 'dispatcherServlet'
2017-03-08 03:35:56.728  INFO 6 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : FrameworkServlet 'dispatcherServlet': initialization started
2017-03-08 03:35:56.774  INFO 6 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : FrameworkServlet 'dispatcherServlet': initialization completed in 43 ms
```

执行以下命令，查看正在运行的Docker容器：

```bash
sudo docker ps
```

可以看到已经有一个Docker容器在运行了：

```
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                    NAMES
00eed53e6928        springio/docker     "sh -c 'java $JAVA..."   About a minute ago   Up About a minute   0.0.0.0:8080->8080/tcp   fervent_leavitt
```

现在输入`http://localhost:8080`可以查看到“Hello Docker World”结果。

如果要停止容器，可以执行下面的命令：

```bash
sudo docker stop 00e
```

  [1f5046f6]: http://start.spring.io/ "http://start.spring.io/"
  [934425c5]: http://spring.io/guides/gs/spring-boot-docker/ "Getting Start"
