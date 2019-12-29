# Docker 高级特性

本次分享给大家介绍Docker 的高级特性与相应的工具。 它们就是Docker 三剑客，Compose、Machine和Swarm

 

## Compose

### 介绍

Docker Compose 是Docker 官方编排（Orchestration）项目之一，负责快速的部署分布式应用。

Compose 定位是 「定义和运行多个Docker 容器的应用（Defining and running multi-container Docker applications）」

其前身是开源项目Fig。其代码目前在https://github.com/docker/compose 上开源。 

### 安装

~~~bash
pip install -U docker-compose
~~~

或

~~~bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
~~~

### 使用

`Dockerfile`

~~~dockerfile
FROM python:3.7-slim

WORKDIR /app

COPY . /app

RUN pip install flask -i https://mirrors.aliyun.com/pypi/simple  --trusted-host mirrors.aliyun.com 

EXPOSE 80

ENV NAME World

CMD ["python", "app.py"]

~~~

`app.py`

~~~python
from flask import Flask
import os
import socket

app = Flask(__name__)

@app.route("/")
def hello():
    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}"
    return html.format(name=os.getenv("NAME", "world"),
               hostname=socket.gethostname())

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)

~~~

`docker-compose.yml`

~~~yaml
version: "3"
services:
  myapp:
    # build: .
    image: friendlyhello:v2
    container_name: myapp
    ports:
      - "5000:80"
    environment:
      NAME: World

  redis:
    image: redis
    container_name: web
~~~

执行 `docker-compose build` 可生成镜像

执行 `docker-compose up` 启动容器运行

浏览器访问

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/073759.png" alt="local-access" style="zoom:50%;" />

###命令说明

 <img src="./imgs/docker-compose-commands.png" alt="docker-compose-commands" style="zoom:50%;" />



## Machine

### 介绍

Docker Machine 是`Docker` 官方编排（Orchestration）项目之一，负责在多种平台上快速安装`Docker `环境。

 ![docker-machine-architecture](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/073839.png)



### 使用

使用 `virtualbox` 类型的驱动，创建一台`Docker` 主机，命名为 `manager`。

~~~bash
docker-machine create -d virtualbox manager 
~~~



![create-manager](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/073845.png)

可以在创建时加上如下参数，来配置主机或者主机上的`Docker`。

 ~~~bash
--engine-opt dns=114.114.114.114 配置Docker 的默认DNS

--engine-registry-mirror https://registry.docker-cn.com 配置Docker 的仓库镜像

--virtualbox-memory 2048 配置主机内存

--virtualbox-cpu-count 2 配置主机CPU
 ~~~

更多参数请使用 `docker-machine create —help` 命令查看。

`docker-machine ls` 查看主机

![docker-machine-ls](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/073854.png)

`docker-machine env manager` 查看环境变量

![docker-machine-env-manager](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/0bct6.png)

切换 `docker` 主机 `manager` 为操作对象 

~~~bash
eval $(docker-machine env manager)
~~~

或者可以 `ssh` 登录到 `docker` 主机

~~~bash
docker-machine ssh manager
~~~

![docker-machine-ssh-manager](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/gbw2f.png)



### 命令说明

![docker-machine-commands](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/dgvi9.png)

 


## Swarm

`Swarm` 是使用`SwarmKit` 构建的`Docker` 引擎内置（原生）的集群管理和编排工具。

![docker-swarm-architecture](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/73kns.png)

### 使用

初始化集群

在上节介绍 `docker-machine` 的时候，我们创建了`manager`节点，而初始化集群需要在管理节点内执行

`docker swarm init --advertise-addr=IP_ADDR` ![docker-swarm-init](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/u74f8.png)

现在来创建两个工作节点`worker1`, `worker2`并加入集群

~~~bash
docker-machine create -d virtualbox worker1

eval $(docker-machine env worker1)

docker swarm join --token SWMTKN-1-59qol34ustn06wtqs6bnsgar4j170k5aj24weu5yegq8qp66cb-26aroyxll4zh9pl8cdwuo7vm4 192.168.99.101:2377
~~~

![docker-swarm-join](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/jv19u.png)

同理`worker2` 节点

进入`manager` 节点执行 

`docker node ls`

![docker-node-ls](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/w64b8.png)

 由此，我们就得到了一个最小化的集群。 



### 命令说明 ![docker-swarm-commands](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/wycws.png)



## 疑难解答

- 在`docker stack deploy –c docker-compose.yml` 后，在`docker ps` 中无法看到端口映射？

![Q1](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/odmm7.png) 

关于docker swarm mode 部署后端口的问题，可以使用`docker service ls `来查看端口是否正确暴露，因为此时是通过service来暴露的，并不是直接在container上暴露，所以此时用`docker ps`是看不到的，但暴露的端口依旧可以访问，这样实现和k8s里的service实现是有些相似的。

- 执行`docker-compose -f docker-compose.yml up -d`,返回

~~~
Pulling myapp (friendlyhello:v2)...

ERROR: Get https://registry-1.docker.i... net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
~~~

​	compose文件中如果已经build过，就用image直接指定这个image，注释掉build的指令。如果没有build过，就放开build指令，执行`docker-compose`的build它，当然也可以使用`docker build`来构建它。因为这一块在上一章节已经提到过，所以对于部分这次直接切入的同学可能会有疑惑。而到了docker stack时，已经不支持`docker stack`来build它了，需要统一使用docker build来构建镜像。
