# Docker Basic 

## 安装

[官方安装文档](https://docs.docker.com/install/)

### Windows

[安装](https://www.runoob.com/docker/windows-docker-install.html)

### Debian/Ubuntu

```sh
# 官方
curl -sSL https://get.docker.com/ | sh

# 阿里云
curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -

# Daocloud
curl -sSL https://get.daocloud.io/docker | sh
```

### MacOS

```sh
brew cask install docker
```





## 简单指令

- 查看 Docker 版本

  版本信息：`docker --version`

  配置信息： `docker info`

  help 信息： `docker --help`

- 运行第一个 Docker 镜像

  `docker run hello-world`

## Docker 命令行工具

Docker CLI 指令分为两类，一类是 Management Commands，一类是镜像与容器 Commands。

你可以通过 `docker –help` 进行查看。

Docker 的命令格式为：

docker [OPTIONS] COMMAND

这里我们选取常用的几个命令进行示例:

`docker pull` :  从镜像仓库中拉取镜像

```sh
# 从镜像仓库中拉取 Ubuntu 镜像
docker pull Ubuntu

# 运行 Ubuntu 镜像，分配 tty 进入交互模式
# -i : interactive mode 
# -t: 分配 tty 
docker run –it ubuntu:latest
```

Docker-CLI 与 Linux 语法具有相似性，例如：

```sh
# 列出所有镜像
docker image ls 

# 列出所有容器
docker container ls

# 查看容器状态
docker container ps 
```

如果你有 Linux 基础，那么相信对于 Docker-CLI 上手还是比较容易的。



## TRY IT OUT #1

`docker run -d -P daocloud.io/daocloud/dao-2048`

> -d 表示容器启动后在后台运行

用 docker ps 查看容器运行状态如图：

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/071731.png" alt="image-20191219151718661" style="zoom:50%;" />

> 看到端口映射关系 0.0.0.0:32768->80。指宿主机的 32768 端口映射到容器的 80 端口

用浏览器打开 localhost:32768

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/071802.png" alt="image-20191219151800494" style="zoom:50%;" />



## Dockerfile 简介

Docker 可以从 Dockerfile 文件中构建镜像.

Dockerfile 语法请参考：https://docs.docker.com/engine/reference/builder/

下面列出一些最常用的语法：

- FROM : 这会从 Docker Hub 中拉取镜像，目的镜像基于所拉取的镜像进行搭建

- WORKDIR: RUN, CMD, ENTRYPOINT, COPY, ADD以此为工作路径

- COPY： 拷贝文件或文件夹到指定路径

- RUN：镜像的最上层执行命令，执行后的结果会被提交，作为后续操作基于的镜像。

- EXPOSE：暴露端口号

- ENV： 设置环境变量 

- CMD ["executable","param1","param2"]：一个 Dockerfile 应该只有一处 CMD 命令，如果有多处，则最后一处有效。

#TRy it out #2

首先准备一个 Dockerfile 文件 与一个 app.py 文件

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/071920.png" alt="image-20191219151919172" style="zoom:50%;" />

分别执行

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/071936.png" alt="image-20191219151933441" style="zoom:50%;" />

中间打印输出

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/072002.png" alt="image-20191219152000816" style="zoom:50%;" />

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/072016.png" alt="image-20191219152012608" style="zoom:50%;" />

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/072024.png" alt="image-20191219152022398" style="zoom:50%;" />

Docker 生成 container时会生成一个唯一的 container-id，在上图中 stop 命令用到了 container-id。当然，你可以使用 docker tag 命令对 container 进行重命名。

>  -p 4000:80 : 指的是从宿主机端口 4000 映射到容器端口 80  

现在打开浏览器访问 `localhost:4000`:

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/072039.png" alt="image-20191219152037355" style="zoom:50%;" />

## Reference

[docker 官方文档](https://docs.docker.com/)