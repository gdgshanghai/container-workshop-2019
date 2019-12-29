# 通过Docker Swarm部署friendlyhello:v4

本篇教程会介绍如何使用`docker swarm`进行容器的编排与分发friendlyhello:v4，这一服务可以通过浏览器访问，获取当前节点的Hostname，最终效果如下：

![Screen Shot 2019-09-03 at 10.04.24 AM.png](https://i.loli.net/2019/09/03/QZmxpPXaqkTn4UC.png)

## STEP 1: 创建虚拟机

为了模拟集群环境，我们创建3个虚拟机，1个作为manager节点、另外2个则是worker节点，~~我自己的mac是8g内存，虽然有些捉急，但3个vm还是撑得住的~~：

```shell
$ docker-machine create manager
$ docker-machine create worker-1
$ docker-machine create worker-2
```

![Screen Shot 2019-09-03 at 9.20.51 AM.png](https://i.loli.net/2019/09/03/DKp7ykHSqAZ5PTd.png)

然后分别进入这三台虚拟机（请开三个terminal来操作）：

```shell
$ docker-machine ssh manager
```

```shell
$ docker-machine ssh worker-1
```

```shell
$ docker-machine ssh worker-2
```

![Screen Shot 2019-09-03 at 9.29.41 AM.png](https://i.loli.net/2019/09/03/8Zkw9AorCMXexNh.png)

## STEP 2: 初始化集群

首先确认一下每个vm的IP：

```shell
$ ifconfig
```

例如，我现在三个ip分别是：

- manager: 192.168.99.112
- worker-1: 192.168.99.113
- worker-2: 192.168.99.114

然后在`manager`里，执行：

```shell
$ docker swarm init --addvertise-addr <你的manager节点的ip>

# 我自己的命令：
$ docker swarm init --addvertise-addr 192.168.99.112
```

完成后，复制输出的`加入swarm集群的命令`，在两个worker里分别执行一下，最后效果如下就说明成功了：

![Screen Shot 2019-09-03 at 9.30.53 AM.png](https://i.loli.net/2019/09/03/avPZALgc2X86Rpn.png)

## STEP 3: 创建`friendlyhello:v4`服务

我们先在当前目录下直接创建两个文件，`app.py` `Dockerfile`，并加入如下代码：

- app.py

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
  
      html = "<b>HostName:</b> {host_name}<br/>" \
             "<b>Hostname:</b> {hostname}<br/>" \
             "<b>Visits:</b> {visits}"
      return html.format(host_name=os.getenv("HOSTNAME", "UNKNOWN"),
                         hostname=socket.gethostname(), visits=visits)
  
  if __name__ == "__main__":
      app.run(host='0.0.0.0', port=5000)
  ```

  

- Dockerfile

  ```dockerfile
  FROM python:3.7-slim
  
  WORKDIR /app
  
  COPY . /app
  
  RUN pip install flask redis -i https://mirrors.aliyun.com/pypi/simple  --trusted-host mirrors.aliyun.com 
  EXPOSE 5000
  
  CMD ["python", "app.py"]
  ```

  ![Screen Shot 2019-09-03 at 9.34.05 AM.png](https://i.loli.net/2019/09/03/pafHRqj2x3ZXSiC.png)

接着构建`image`：

```shell
$ docker build -t friendlyhello:v4 .
```

![Screen Shot 2019-09-03 at 9.35.14 AM.png](https://i.loli.net/2019/09/03/ECDa76mJ8t5THYq.png)

然后通过`docker service create`来创建服务，`--replicas 3`参数表示创建3个副本：

```shell
$ docker service create --replicas 3 -p 5000:5000 --name friendly friendlyhello:v4
```

![Screen Shot 2019-09-03 at 9.36.24 AM.png](https://i.loli.net/2019/09/03/GaklXuD3RLdrPCq.png)

成功以后，我们就可以看到这一服务的`task`们了：

```shell
$ docker service ps friendly
```

![Screen Shot 2019-09-03 at 9.39.09 AM.png](https://i.loli.net/2019/09/03/9Et2yqeDBz3dTFl.png)

但是，我们发现只有`manager`节点的`task`启动了，2个`worker`节点都表示没有镜像，所以接下来我们就需要去做**分发**。

## STEP 4: 快速分发镜像到所有节点

我们先缩个容：

```shell
$ docker service scale friendly=1
```

![Screen Shot 2019-09-03 at 9.40.35 AM.png](https://i.loli.net/2019/09/03/qhD1vaiwNsPo4jg.png)

为了高效分发，我们不用`docker hub`，而是自己部署一个`registry`服务：

```shell
$ docker service create --name registry --publish 5555:5000 registry:2
```

看看成功了没：

```shell
$ docker service ps registry
```

![Screen Shot 2019-09-03 at 9.42.54 AM.png](https://i.loli.net/2019/09/03/YFkC57tcTfLsEmy.png)

OK，接下来我们来把`friendlyhello:v4`镜像给推送到`registry`，在此之前，我们先创建一个配置文件，加入配置，否则可能会出现`https`相关问题：

```shell
$ vi /etc/docker/daemon.json
```

写入：

```json
{"insecure-registries": ["<你的manager的ip>:5555"]}

// 我的配置：
{"insecure-registries": ["192.168.99.112:555"]}
```

![Screen Shot 2019-09-03 at 9.45.41 AM.png](https://i.loli.net/2019/09/03/VWuKMhawe49NFqx.png)

接着重启一下docker：

```shell
$ sudo /etc/init.d/docker restart
```

重启后，我们就可以重新打`tag`然后`push`到`registry`了：

```shell
$ docker tag friendlyhello:v4 <你的manager的ip>:5555/friendlyhello:v4
$ docker push <你的manager的ip>:5555/friendlyhello:v4

# 我的命令：
$ docker tag friendlyhello:v4 192.168.99.112:5555/friendlyhello:v4
$ docker push 192.168.99.112:5555/friendlyhello:v4
```

![Screen Shot 2019-09-03 at 9.50.10 AM.png](https://i.loli.net/2019/09/03/a1D3F9rXRsLJIty.png)

> 如果这里出现了奇奇怪怪的问题，试试等个几秒重新执行一下

接着，分别到`worker-1`和`worker-2`里，加入一个一样的`daemon.json：`

```shell
$ vi /etc/docker/daemon.json
```

![Screen Shot 2019-09-03 at 9.51.58 AM.png](https://i.loli.net/2019/09/03/R6eqM3g1AhwPIlr.png)

**别忘了添加完配置后重启一下**

重启后，就可以`pull`下我们需要的那个镜像了：

```shell
$ docker pull 192.168.99.112:5555/friendlyhello:v4
$ docker tag 192.168.99.112:5555/friendlyhello:v4 friendlyhello:v4
```

![Screen Shot 2019-09-03 at 9.54.59 AM.png](https://i.loli.net/2019/09/03/eohCiryKT9Q6WwE.png)

> 如果又出现了奇奇怪怪的问题，可以试试在`manager`节点再重新`push`一下

接着在`manager`节点扩容：

```shell
$ docker service scale friendly=3
```

![Screen Shot 2019-09-03 at 9.56.02 AM.png](https://i.loli.net/2019/09/03/qEd5BCybPX7TnhZ.png)

这下3个节点的`task`就都成功跑起来了：

```shell
$ docker service ps friendly
```

![Screen Shot 2019-09-03 at 9.56.35 AM.png](https://i.loli.net/2019/09/03/OcbPlfA697h8eNg.png)

可以通过浏览器访问看看：

![Screen Shot 2019-09-03 at 9.59.57 AM.png](https://i.loli.net/2019/09/03/9kTsUNjDHKweX5A.png)

部署是没什么问题了，不过我们还需要让它显示所使用的node的名字，这需要加入环境变量。

我们先关了现在的服务：

```shell
$ docker service rm friendly
```

再重新创建，同时加入新的参数：

```shell
$ docker service create --replicas 3 -p 5000:5000 --name friendly3 -e HOSTNAME="{{.Node.Hostname}}" --hostname="{{.Node.Hostname}}-{{.Node.ID}}-{{.Service.Name}}" friendlyhello:v4
```

![Screen Shot 2019-09-03 at 10.03.46 AM.png](https://i.loli.net/2019/09/03/rew3x4lX8jOoZkB.png)

再来访问看看：

![Screen Shot 2019-09-03 at 10.04.24 AM.png](https://i.loli.net/2019/09/03/pVhrF4odb715x6f.png)


