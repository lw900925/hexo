---
title: Docker：容器编排利器Compose（起步篇）
date: 2017-03-17
desc:
keywords: Docker,Compose
categories: [docker]
---

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/banner/docker-logo.jpeg">

一个大型的Docker组成的微服务应用中，容器的数量是非常庞大的，如果依赖传统的人工配置方式进行维护，对于开发和运维来说简直就是噩梦。Compose的出现正是为了解决这个问题。

<!-- more -->

## Compose简介

Compose的前身是Fig，Fig被Docker收购之后正式更名为Compose，Compose向下兼容Fig。Compose是一个用于定义和运行多容器Docker应用的工具，只需要一个Compose的配置文件和一个简单的命令就可以创建并运行应用所需的所有容器。在配置文件中，所有容器通过`services`来定义，并使用`docker-compose`命令启动或停止容器以及所有依赖容器。

## 安装Compose

Compose的安装方式有多种，这里推荐使用`curl`命令安装，在安装之前，要确保你的机器上已经安装了Docker，可以运行`sudo docker version`命令来确认是否已安装了Docker。截至目前，Compose的最新发布版为`1.11.2`，下面演示在一台已经安装好Docker的Linux主机上安装Compose。

安装很简单，只需要执行下面的命令即可：

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.11.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

等待安装完毕后，执行下面的命令，为`docker-compose`添加可执行权限：

```bash
chmod +x /usr/local/bin/docker-compose
```

输入`docker-compose --version`命令可以查看安装结果。

除了这种安装方式之外，还可以通过Python的`pip`命令安装或将Compose安装成Docker容器，详情请参见[https://docs.docker.com/compose/install/#install-as-a-container][676d77a3]。

如果要卸载Compose，可以执行`sudo rm /usr/local/bin/docker-compose`命令。

## Compose入门

下面我们通过一个简单的例子演示Compose的使用步骤，使用Python构建一个Web应用，该应用使用Flask框架，并在Redis中维护一个命中计数（即使你不熟悉Python也没有关系，你甚至不需要安装Python和Redis，我们会从容器中获取这些依赖环境）。

### 创建工程

首先需要一个文件夹作为项目文件夹：

```bash
mkdir composetest
cd composetest
```

在项目文件夹下创建一个`app.py`的文件，并将下面的代码拷贝并粘贴到该文件中：

```python
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis(host='redis', port=6379)

@app.route('/')
def hello():
    count = redis.incr('hits')
    return 'Hello World! I have been seen {} times.\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

在项目文件夹下创建一个`requirements.txt`的文件，并将下面的代码拷贝并粘贴到该文件中：

```
flask
redis
```

到此，我们已经完成了新建项目，编码，添加依赖等工作。

### 创建Dockerfile

下面我们创建一个`Dockerfile`文件用于构建Docker镜像，该镜像包含了运行该Web应用的所有依赖，包括Python运行环境。

在项目文件夹下创建一个`Dockerfile`文件，并将下面的内容拷贝并粘贴到该文件中：

```
FROM python:3.4-alpine
ADD . /code
WORKDIR /code
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

大概解释一下这个配置文件：

- 使用`python-3.4-alpine`作为基础镜像
- 将当前目录添加到镜像中`/code`目录下
- 将`/code`设置为工作目录
- 安装Python依赖
- 设置默认执行命令

### 在Compose文件中定义services

在项目文件夹下创建一个`docker-compose.yml`文件，并将下面的内容拷贝并粘贴到该文件中：

```yml
version: '2'
services:
  web:
    build: .
    ports:
     - "5000:5000"
    volumes:
     - .:/code
  redis:
    image: "redis:alpine"
```

该配置文件中包含两个`services`，即`web`和`redis`。`web`会使用当前目录中的`Dockerfile`文件构建镜像，并将容器的`5000`端口暴露给主机，然后将项目文件夹挂载到容器中的`/code`目录下；`redis`使用官方发布的镜像构建。

### 构建并运行

执行下面的命令构建并运行容器：

```bash
sudo docker-compose up
```

容器构建完成并启动后，可以在浏览器中输入`http://localhost:5000`查看结果。页面会打印“Hello World! I have been seen 1 times.”，刷新页面后，计数会累加变成2。

### 更新应用

由于项目文件夹挂载到了容器中，所以我们可以直接修改项目文件夹的应用，修改的结果立即反应到容器中，而不用重新启动容器。将`app.py`文件中的`hello`方法中的返回值修改成如下：

```python
return 'Hello from Docker! I have been seen {} times.\n'.format(count)
```

保存后刷新浏览器，发现打印结果已经更新。

## Compose的其他命令

上面提到的Componse使用命令构建并启动容器，是以前台的方式启动的，如果希望以后台启动，可以添加参数`-d`，比如下面这样：

```bash
sudo docker-compose up -d
```

`docker-compose ps`命令可以查看正在运行的容器：

```bash
liuwei@liuwei-Ubuntu:~$ sudo docker-compose ps
Name                      Command               State           Ports
-------------------------------------------------------------------------------------
composetest_redis_1   docker-entrypoint.sh redis ...   Up      6379/tcp
composetest_web_1     python app.py                    Up      0.0.0.0:5000->5000/tcp
```

如果使用`sudo docker-compose up -d`命令以后台方式启动，可以用`docker-compose stop`命令停止。`docker-compose down --volumes`命令可以停止容器并将其删除，`--volumns`表示同时删除redis数据文件目录。

有关Compose的更多命令，可以通过`sudo docker-compose --help`查看。

以上就是Compose的一个基本使用过程，可以发现，Compose将`docker run`命令整合到了一个`docker-compose.yml`配置文件中，对于大型Docker集群的管理是很方便的，例可以将多个`service`组合成更复杂的`service`组，为每个`service`指定不同的`Dockerfile`，然后把它们`link`在一起。

[676d77a3]: https://docs.docker.com/compose/install/#install-as-a-container "https://docs.docker.com/compose/install/#install-as-a-container"
