---
title: Docker：自定义镜像仓库
date: 2017-03-07
desc:
keywords: Docker
categories: [docker]
---

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/banner/docker-logo.jpeg">

从阿里云提供的镜像仓库获取Docker镜像。

<!-- more -->

由于国内网络问题，使用官方的Docker Hub上`pull`一个镜像非常慢，并且时不时会连接超时，所以有必要使用国内提供的镜像仓库。

## 国内镜像仓库

国内的镜像仓库提供者有：

- [DaoCloud - Docker加速器][3f9d0ca8]
- [阿里云 - 开发者平台][f5c1e843]
- [希云cSphere - 微镜像][e359b71d]
- [时速云 - 镜像广场][419a8165]
- [网易蜂巢 - 镜像中心][9ae21bcf]

我使用的是阿里云的镜像仓库，下面是具体配置方法。

## 配置Docker镜像仓库

首先需要到[开发者平台][f5c1e843]登录阿里云账号（没有的话可以注册一个），登录后点击页面右上角的[管理中心][57695810]按钮进入管理控制台，然后点击左侧[加速器][34c3e09d]菜单按钮，可以看到为你自己分配的专属加速地址以及不同平台的配置文档。

**注意**：第一次进入控制台需要设置一个密码，该密码用于登录Docker镜像仓库，请牢记，后面会用到。

下面内容我直接从阿里云提供的操作文档复制过来了，是Ubuntu版本的详细配置步骤，其他平台可以参考[此处][34c3e09d]。

### 安装／升级你的Docker客户端

推荐安装`1.6.0`以上版本的Docker客户端。

您可以通过阿里云的镜像仓库下载：[mirrors.aliyun.com/help/docker-engine][216d6e6a]

或执行以下命令：

```bash
curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -
```

### 如何使用Docker加速器

** 针对Docker客户端版本大于1.10的用户 **

您可以通过修改daemon配置文件`/etc/docker/daemon.json`来使用加速器：

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://15laodr8.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

** 针对Docker客户的版本小于等于1.10的用户 **

或者想配置启动参数，可以使用下面的命令将配置添加到docker daemon的启动参数中。

Ubuntu 12.04 14.04的用户：

```bash
echo "DOCKER_OPTS=\"\$DOCKER_OPTS --registry-mirror=https://15laodr8.mirror.aliyuncs.com\"" | sudo tee -a /etc/default/docker
sudo service docker restart
```

Ubuntu 15.04 16.04的用户：

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/mirror.conf <<-'EOF'
[Service]
ExecStart=/usr/bin/docker daemon -H fd:// --registry-mirror=https://15laodr8.mirror.aliyuncs.com
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 下载Docker镜像

上面步骤配置完成后，就可以使用阿里云镜像仓库下载镜像了，首先我们需要登录到阿里云的Registry，运行下面命令：

```bash
sudo docker login -u [username] -p [password] -e [email] registry.aliyuncs.com
```

`username`是登录的用户名，一般为注册时填写的邮箱，`password`就是首次进入控制台让你设置的密码，`email`为注册时填写的邮箱。运行命令后如果出现`Login Succeeded`表示登录成功。

登录成功后就可以下载镜像了，运行下面的命令试试看速度如何：

```
sudo docker pull ubuntu
```

具体下载的速度还是要取决于你的网络带宽，总体来说比Docker官方镜像仓库要快很多。

这里只说明了阿里云平台的Docker镜像仓库的配置方法，其他Docker仓库供应商的配置大体上比较类似，需要的话可以去查看相关的操作手册。


  [3f9d0ca8]: https://dashboard.daocloud.io/ "DaoCloud - Docker加速器"
  [f5c1e843]: https://dev.aliyun.com/ "阿里云 - 开发者平台"
  [e359b71d]: https://csphere.cn/hub "希云cSphere - 微镜像"
  [419a8165]: https://hub.tenxcloud.com/ "时速云 - 镜像广场"
  [9ae21bcf]: https://c.163.com/hub#/m/home/ "网易蜂巢 - 镜像中心"
  [57695810]: https://cr.console.aliyun.com "管理中心"
  [34c3e09d]: https://cr.console.aliyun.com/#/accelerator "加速器"
  [216d6e6a]: mirrors.aliyun.com/help/docker-engine "mirrors.aliyun.com/help/docker-engine"
