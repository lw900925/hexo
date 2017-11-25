---
title: Docker：创建Redis集群
date: 2017-03-10
desc:
keywords: Docker
categories: [docker]
---

<img src="http://ohwsf74ph.bkt.clouddn.com/image/banner/docker-logo.jpeg">

虽然其他网站上有大量关于Docker创建Redis集群的文章，但大多数都比较片面（感觉还是无脑复制粘贴，错误百出），所以决定重新整理一下，把遇到的问题都记录下来。

<!-- more -->

## 1.获取Redis镜像

首先从Docker Hub或其他镜像仓库中获取Redis镜像，这里我使用了Docker官方提供的Redis镜像，运行下面命令获取Redis镜像：

```bash
sudo docker pull ubuntu
sudo docker pull redis
```

镜像下载完毕后，执行`sudo docker images`命令查看结果：

```bash
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
redis                        latest              e4a35914679d        9 days ago          183 MB
ubuntu                       latest              0ef2e08ed3fa        10 days ago         130 MB
```

可以看到`redis`和`ubuntu`两个镜像已经获取成功，下面开始配置。

## 2.配置Redis集群

在配置Redis集群之前，需要一个`redis.conf`的配置文件，该配置文件可以从Redis官方站点获取：

```bash
wget -c http://download.redis.io/redis-stable/redis.conf
```

等待下载完毕后，将`redis.conf`拷贝三份，并依次重命名为：`redis-master.conf`，`redis-slave1.conf`，`redis-slave2.conf`，将三个文件放在`/home/liuwei/docker/redis/`目录中，看起来像这样：

```bash
/home/liuwei/docker/redis/redis-master.conf
/home/liuwei/docker/redis/redis-slave1.conf
/home/liuwei/docker/redis/redis-slave2.conf
```

**注**：这三个文件存放的位置无特殊要求，可以指定任意位置。

将`redis-master.conf`文件中的配置项做如下修改：

```
daemonize yes
pidfile /var/run/redis.pid
```

将`redis-slave1.conf`和`redis-slave2.conf`文件中的配置项做如下修改：

```
daemonize yes
pidfile /var/run/redis.pid
slaveof master 6379
```

`slaveof`默认是注释掉的，只需要开启注释即可。需要注意的是`slaveof`的格式为`slaveof <masterip> <masterport>`，上面配置中`masterip`参数为`master`，实际上是一个别名，稍后会对它进行解释。配置完成后，下一步就是创建Docker容器并启动了。

## 3.创建Redis容器

创建Redis容器只需要使用第一步下载好的Redis镜像，调用Docker的`run`命令即可，不过要创建一个Redis集群，还需要处理容器与容器之间的通信问题（即Redis的主从复制需要容器之间能够相互连通），这里使用`docker run`命令的`--link`参数来建立容器之间的相互连通。

简单介绍一下`--link`这个参数，`--link`的使用格式为：`name:alias`，可以在`docker run`命令中重复使用该参数，例如：

```bash
sudo docker run -it --name ubuntu ubuntu /bin/bash
sudo docker run -it --name redis --link ubuntu:ubuntu redis /bin/bash
```

上述命令将使用镜像创建一个名为`ubuntu`容器，然后创建一个名为`redis`的容器，将该容器连接到`ubuntu`容器上。通过`--link`参数连接的两个容器，可以避免容器的IP和端口暴露到外网导致的安全问题，以及容器在重启后IP地址变化导致访问失效。当容器的IP发生变化时，Docker会自动维护容器中的hosts文件，如果你打开两个通过`--link`参数连接的容器中的某个容器的`/etc/hosts`文件，你将看到如下内容：

```bash
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
# 以下记录了容器的连接信息
172.17.0.2	master 385f6821c0db redis-master
172.17.0.3	54466fb83744
```

通过`--link`参数，我们可以创建包含一个`master`和两个`slave`的Redis集群，使用如下命令创建：

```bash
sudo docker run -it -v /home/liuwei/docker/redis/redis-master.conf:/usr/local/etc/redis/redis.conf --name redis-master redis /bin/bash
sudo docker run -it -v /home/liuwei/docker/redis/redis-slave1.conf:/usr/local/etc/redis/redis.conf --name redis-slave1 --link redis-master:master redis /bin/bash
sudo docker run -it -v /home/liuwei/docker/redis/redis-slave2.conf:/usr/local/etc/redis/redis.conf --name redis-slave2 --link redis-master:master redis /bin/bash
```

上述命令中使用`--link redis-master:master`参数，前面提到的`redis-slave.conf`配置文件中`slaveof`配置项，这里使用了一个`master`作为别名，其效果和使用IP一样（IP地址在`/etc/host`文件中）。

需要注意的是`-v`参数，`-v`参数用于将宿主机上的某个目录挂载到容器中。由于容器都是轻量化设计，只包含运行时的必须文件，所以在容器中使用`vim`之类的命令很不方便（可能需要自行安装vim编辑器），所以我们将之前配置好的`redis.conf`文件挂载到对应的容器中。因此，我们可以直接在宿主机上使用`vim`命令或其他文本编辑器编辑`redis.conf`文件。

`-v`的格式为：`-v /host/path:/container/path`，前部分表示主机的目录，后部分表示容器的目录（如果目录不存在，容器启动的时候会自行创建）。

容器启动完毕后，可以运行`sudo docker ps`查看容器启动状态：

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
d2b29622b98d        redis               "docker-entrypoint..."   3 hours ago         Up 3 hours          6379/tcp                 redis-slave2
54466fb83744        redis               "docker-entrypoint..."   3 hours ago         Up 3 hours          6379/tcp                 redis-slave1
385f6821c0db        redis               "docker-entrypoint..."   3 hours ago         Up 3 hours          6379/tcp                 redis-master
```

接下来是启动`redis`服务，这里推荐为每个容器分配一个`shell`窗口，方便直接在容器内执行命令。先启动`master`，然后启动`slaver`。分别在`master`、`slave1`和`slave2`中运行启动`redis`服务的命令：

```bash
redis-server /usr/local/etc/redis/redis.conf
```

启动后，在`master`容器中执行下面的命令查看`redis`服务的运行状态：

```bash
redis-cli
127.0.0.1:6379> info
```

结果为：

```bash
# 此处已省略

# Replication
role:master
connected_slaves:2
slave0:ip=172.17.0.4,port=6379,state=online,offset=29,lag=1
slave1:ip=172.17.0.3,port=6379,state=online,offset=29,lag=1
master_repl_offset:29
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:28

# 此处已省略
```

这里有个坑需要注意一下，我在配置完毕后启动容器，运行`info`命令查看到上述信息中`connected_slaves`的值为0，测试主从复制也没有成功，后来搜索了相关资料，需要将`redis-master.conf`文件中的`bind 127.0.0.1`修改为`bind 0.0.0.0`，修改完毕后重启`master`服务即可。


可以看到`master`中已经包含两个`slave`，接下来我们测试一下主从复制是否成功，在`master`容器中输入下面的命令：

```bash
127.0.0.1:6379> set master liuwei
OK
127.0.0.1:6379> get master
"liuwei"
```

然后分别在`slave1`和`slave2`容器中执行：

```bash
127.0.0.1:6379> get master
"liuwei"
```

如果能成功看到输出信息，说明主从复制成功。

相比使用虚拟机搭建Redis集群环境，Docker显得更简单轻量，如果需要配置更多`slave`，只需要使用命令创建一个容器，连接到`master`容器。
