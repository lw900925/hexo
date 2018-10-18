---
title: Docker：配置HTTP/HTTPS代理
date: 2018-05-14
desc:
keywords: Docker
categories: [docker]
---

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/banner/docker-logo.jpeg">

<!-- more -->

# 起因

我在使用Docker的`pull`命令拉取ELK官方提供的镜像时，会出现无法连接的情况，并且会出现`TLS handshake timeout`的错误。在搜索相关文章之后得出结论：国内的网络环境不好，导致连接`docker.elastic.co`失败或无法连接。于是我第一时间想到了代理的方式，好在Docker支持设置代理来访问其他Registry，下面记录整个配置过程。

# 准备工作

首先，你的机器上需要安装好Docker，当我写这篇文章时，Docker的版本为`18.03`，对于后续版本，本文章的配置方法可能会失效。

此外，还需要准备一个代理服务器，可以正常访问境外网站（如：Google，YouTuBe等）。我用的是VPS搭建的Shadowsocks代理，本机Shadowsocks客户端开启之后可以直接通过`http://127.0.0.1:1080/`访问境外网站。

假设你的环境也是Ubuntu（其他环境应该也是类似的）。

# 开始配置

1. 创建如下路径的目录

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
```

2. 进入到上一步创建的目录下，并在该目录下创建一个名为`http-proxy.conf`的文件（如：`/etc/systemd/system/docker.service.d/http-proxy.conf`），使用vim编辑文件内容如下

```bash
[Service]
Environment="HTTPS_PROXY=http://127.0.0.1:1080/" "NO_PROXY=localhost,127.0.0.1,registry.docker-cn.com,hub-mirror.c.163.com"
```

3. 刷新配置

```bash
sudo systemctl daemon-reload
```

4. 重启Docker服务

```bash
sudo systemctl restart docker
```

5. 查看配置

```bash
systemctl show --property=Environment docker
```

出现如下信息表示配置成功：

```bash
Environment=HTTPS_PROXY=http://127.0.0.1:1080/ NO_PROXY=localhost,127.0.0.1,registry.docker-cn.com,hub-mirror.c.163.com
```

6. 验证配置是否生效

重新从`docker.elastic.co`上拉取`elasticsearch`镜像，此时已经可以正常连接了，只是速度较慢。

```bash
liuwei@liuwei-Ubuntu:~$ sudo docker pull docker.elastic.co/elasticsearch/elasticsearch:6.2.4
6.2.4: Pulling from elasticsearch/elasticsearch
469cfcc7a4b3: Downloading [==========================>                        ]  38.87MB/73.17MB
8e27facfa9e0: Downloading [===================================>               ]  40.05MB/56.33MB
cdd15392adc7: Download complete 
ddcc70fbd933: Downloading [====================>                              ]  44.31MB/108.9MB
3d3fa0383994: Waiting 
15d1376ebd55: Waiting
```

这种方法适用于从一些第三方提供的Registry上拉取镜像时，由于网络原因无法连接。如果从Docker官方的镜像仓库中拉取镜像时，一种比较好的办法就是配置`registry-mirrors`实现加速，具体方法请自行搜索。

上述步骤是来自Docker官方文档中的部分，原文地址为[Control Docker with systemd](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)。

如果你在配置过程遇到问题，欢迎在下方评论区留言讨论。