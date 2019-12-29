# Docker 集群入门与实战

这篇文章将侧重介绍 docker 在集群方面的部署，在介绍完集群相关概念之后会进入大家喜闻乐见的实战环节~

首先让我们来看看与集群相关的对象：

- swarm
- node
- stack
- service

其中 swarm 在上篇推文已经有过介绍，回顾请看->// 链接

### NODE

node 的概念和 swarm 密切相关，swarm 是指一组包含一个 或多个 运行 docker engine 的主机（物理机或者虚拟机）组成的群组，节点（node）就自然是指在群组中的单体了。

![Swarm mode cluster](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/h1bt3.png)

运行`Docker`的主机可以主动初始化一个`Swarm`集群或者加入一个已存在的`Swarm`集群，这样这个运行`Docker`的主机就成为一个`Swarm`集群的节点 (`node`) 。

node 有两类： managers 和 works，在下面的图中很容易理解它们的关系。

managers 负责 swarm的任务调度和编排决策，works 负责任务的执行，

node 的类别可以互相转换，如果想要转换一个 worker 为 manager，可以使用 `docker node promte` , 反之则使用 `docker node demote`

```shell
$ docker node --help

Usage:    docker node COMMAND

Manage Swarm nodes

Commands:
  demote      Demote one or more nodes from manager in the swarm
  inspect     Display detailed information on one or more nodes
  ls          List nodes in the swarm
  promote     Promote one or more nodes to manager in the swarm
  ps          List tasks running on one or more nodes, defaults to current node
  rm          Remove one or more nodes from the swarm
  update      Update a node

Run 'docker node COMMAND --help' for more information on a command.
```

使用ls命令查看节点信息：

```shell
$ docker node ls

ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
v1oo73vt40hnjivnhnw78po4u *   dameng-00           Ready               Active              Leader              18.09.8-ce
yrz7drszt4hq5j8c8rbhoexqc     dameng-01           Ready               Active                                  18.09.8-ce
```

### SERVICE

> A [service](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/) is the definition of how you want to run your application containers in a swarm.

简单来说，service 定义了 worker node 上要执行的任务。swarm 的主要编排任务就是保证 service 处于期望的状态下。

创建service时，您可以指定要使用的容器镜像以及在运行容器中执行的命令。您还可以定义service的选项，包括：

- 集群外部提供服务的端口
- 服务的网络配置，以连接集群中的其他服务
- CPU和内存大小分配
- 滚动更新策略
- 要在群中运行的镜像的副本数

Service 的创建过程：

![docker-create-service](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/3zb34.png)

```shell
$ docker service --help

Usage:    docker service COMMAND

Manage services

Commands:
  create      Create a new service
  inspect     Display detailed information on one or more services
  logs        Fetch the logs of a service or task
  ls          List services
  ps          List the tasks of one or more services
  rm          Remove one or more services
  rollback    Revert changes to a service's configuration
  scale       Scale one or multiple replicated services
  update      Update a service

Run 'docker service COMMAND --help' for more information on a command.
```

查看：

```shell
$ docker service ls

ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
oyiesfc3biiq        myapp_myapp         replicated          1/1                 friendlyhello:v3    *:5000->5000/tcp
xrk7kska1z76        myapp_redis         replicated          1/1                 redis:latest
```

扩容：

```shell
$ docker service scale myapp_myapp=2

ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
oyiesfc3biiq        myapp_myapp         replicated          2/2                 friendlyhello:v3    *:5000->5000/tcp
xrk7kska1z76        myapp_redis         replicated          1/1                 redis:latest
```

### STACK

stack是一组相互关联的服务，它们共享依赖关系，并且可以一起 orchestrated (编排)和缩放。单个stack能够定义和协调整个应用程序的功能（尽管非常复杂的应用程序可能希望使用多个stack）。 

```shell
$ docker stack --help

Usage:    docker stack [OPTIONS] COMMAND

Manage Docker stacks

Options:
      --kubeconfig string     Kubernetes config file
      --orchestrator string   Orchestrator to use (swarm|kubernetes|all)

Commands:
  deploy      Deploy a new stack or update an existing stack
  ls          List stacks
  ps          List the tasks in the stack
  rm          Remove one or more stacks
  services    List the services in the stack
```

部署：

```shell
# compose:
$ docker-compose -f CONFIG-YAML up

# stack:
$ docker stack deploy -c CONFIG-YAML STACK-NAME
```

查看：

```shell
# compose:
$ docker-compose ps
$ docker ps

# stack:
$ docker node ls
$ docker stack ls
$ docker service ls
```

终止：

```shell
# compose:
$ docker-compose -f CONFIG-YAML down

# stack:
$ docker stack rm STACK-NAME
```

网络：

单机 vs 跨节点

> 注意: docker stack 默认使用的是swarm，但也是可以对接k8s的

## TRY IT OUT : 使用 Swarm 部署 

这次我们要用`Swarm`集群来部署前面做过的`friendlyhello`。

在前面，我们了解到可以用`Docker Machine `很快地创建一个虚拟的Docker主机，接下来我们来创建2个新的Docker主机，并加入到集群中。

### STEP 1: 创建Swarm集群——管理节点

首先是一个管理节点，创建并通过ssh连接：
```shell
$ docker-machine create -d virtualbox manager
$ docker-machine ssh manager
```

我们可以看到：

![docker-ssh-manager](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/bgrn9.png)

然后，我们用`docker swarm init`从这个节点初始化一个`Swarm`集群，如果这个Docker主机有多个IP（多个网卡），就要用`--advertise-addr`指定一个:

```shell
$ docker swarm init --advertise-addr 192.168.99.107
```

我们可以看到：

![docker-swarm-init](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/hi8v1.png)

现在我们的`Manager`节点就是刚刚创建的集群的管理节点了，

记得复制一下它输出的添加工作节点的那句命令。

### STEP 2: 添加项目文件

接下来我们`Manager`节点的`~/try-it-out-4`里添加几个文件:

- app.py
- Dockerfile
- docker-stack.yaml

```shell
$ mkdir try-it-out-4
$ cd try-it-out-4
$ vi app.py
$ vi Dockerfile
$ vi docker-stack.yaml

$ docker build -t friendlyhello .
```

![ls-try-4](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/o5ytb.png)

#### app.py

```python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)


@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)


if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000)
```

#### Dockerfile

```Dockerfile
FROM python:3.7-slim

WORKDIR /app

COPY . /app

RUN pip install flask redis -i https://mirrors.aliyun.com/pypi/simple  --trusted-host mirrors.aliyun.com 
EXPOSE 5000
ENV NAME World

CMD ["python", "app.py"]
```

#### docker-stack.yaml

```yaml
version: "3"

services:
  myapp:
    image: friendlyhello
    container_name: myapp
    ports:
      - 5000:5000
    environment:
      NAME: World

  redis:
    image: redis
    container_name: web
```


### STEP 3: 创建Swarm集群——工作节点

继续，我们来创建一个工作节点：

首先回到主机：

```shell
$ exit 
```

接着创建一个新的虚拟机`worker`，并通过上面复制的那句命令加入到集群里:

```shell
$ docker-machine create -d virtualbox worker
$ docker-machine ssh worker

$ docker swarm join --token SWMTKN-1-3wd0vdozskitmpw5vofkjc9ie6251wuno21dmbugqk56pd97iv-eu9w5gkkmy7chvgcwt7j71iu4 192.168.99.107:2377
```

我们可以看到：

![docker-swarm-join](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/akbwz.png)

### STEP 4: 使用Stack部署服务

我们先回到`manager`节点：

```shell
$ exit

$ docker-machine ssh manager
```

然后使用`docker stack deploy`部署服务，其中`-c`参数指定`docker-stack.yaml`文件：

```shell
$ docker stack deploy -c ~/try-it-out/docker-stack.yaml friendlyhello
```

![docker-stack-depoly](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/riw90.png)

部署完毕。

### STEP 5: 访问friendlyhello

现在就可以通过集群中的任意一个节点的IP访问到这个`flask`项目了：

![local-acces](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/t3yc1.png)

## TRY IT OUT: 使用 Docker stack 部署

我们继续用前一个Case的集群。

进入`manager`节点，在`try-it-out-5`里新建一个`docker-stack.yaml`：

```shell
$ docker-machine ssh manager

$ mkdir try-it-out-5
$ vi docker-stack.yaml
```

#### docker-stack.yaml

```yaml
version: '3.1'

services:

  db:
    image: postgres
    command: postgres -c 'shared_buffers=512MB' -c 'max_connections=2000'
    restart: always
    environment:
      POSTGRES_USER: dameng
      POSTGRES_PASSWORD: pythonic
    ports:
      - 5432:5432
    volumes:
      - pgdata:/var/lib/postgresql/data


  adminer:
    image: adminer
    restart: always
    ports:
      - 8998:8080

volumes:
  pgdata:
```

然后部署：

```shell
$ docker stack deploy -c docker-stack.yaml postgresql
```

接着就可以从`8998端口`访问GUI了：

![db-web-access](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/a42ud.png)
