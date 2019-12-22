# 快速上手K8s 部署

> 本期推送内容为 10.26 期 k8s workshop。由于 k8s 的环境安装稍微繁琐，内容拆分为两期推送，上期主题为 k8s 的环境搭建，使用了第三方工具 ansible 来进行快速搭建。
>
> GitHub传送门：[Kubeasz](https://github.com/easzlab/kubeasz)
>
> 本期主题为 k8s 快速上手，默认大家已经安装好 k8s 环境，那么现在就开始我们的探索吧。

## k8s 基本操作

- 查看组件信息

  ```
  kubectl get componentstatuses
  ```

  ~~~
  NAME                 STATUS    MESSAGE             ERROR
  controller-manager   Healthy   ok
  scheduler            Healthy   ok
  etcd-0               Healthy   {"health":"true"}
  ~~~

- 查看资源类型

  ~~~
  kubectl api-resources
  ~~~
  
- 查看 api 版本

  ~~~
  kubectl api-versions
  ~~~

- 解释资源特性

  ~~~
  kubectl explain <type>.<fieldName>[.<fieldName>]
  
  kubectl explain pods
  kubectl explain pods.spec.containers
  ~~~

## k8s 部署初体验

部署方式有以下 3 种：

- Using Generators (Run, Expose)
- Using Imperative way (Create)
- Using Declarative way (Apply)

### 1. 生成式部署

最简单的方式去部署一个容器就是使用 `kubectl run`， 类似于`docker run`

例如：`kubectl run`

```bash
kubectl run  --generator=run-pod/v1  nginx --image=nginx
kubectl run  --generator=run-pod/v1  nginx --image=nginx --image-pull-policy=IfNotPresent
```

执行后，可以用 `kubectl get pods `

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/1dw6j.png" alt="k8s-run-nginx" style="zoom:50%;" />

### 2. 命令式部署

对于 Imperative，需要用户告诉系统需要做什么。kubectl 转换命令行参数成为声明式的 kubernetes Deployment 对象。

Imperative 的 Commands 主要就是通过命令行的 Flag 直接进行 Imperative 的操作，如 Create 或者是 delete 之类。例子是 Kubectl run, Expose，或是 Create deployment，它就会把底层对于对象配置的操作对用户隐藏了起来。

Imperative 对象配置。Object configuration 的意思是一个对象可以被定义为 yaml 或者 json 的格式存储在文件中，我们叫它 Object Configuration。Imperative Object Configuration 是直接对 Object Configuration 进行 Imperative 的操作，比如说 Kubectl create, Replace,Delete

`kubectl create`

~~~bash
kubectl create deployment --image=nginx  nginx
~~~

<img src="https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/fqplo.png" alt="k8s-create-nginx" style="zoom:67%;" />

### 3. 声明式部署

Declarative 的意思是申明的、陈述的，与之相反的是 Imperative，Imperative 意思是命令式的。

Declarative 的定义是用户设定期望的状态，系统会知道它需要执行什么操作，来达到期望的状态。

`kubectl apply `

~~~bash
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
~~~
deployment.yaml
~~~bash
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

~~~

![k8s-apply](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/2lgk5.png)

以上，利用 3 种不同的方式去部署了 nginx 镜像

## k8s 访问

1. 使用 proxy 访问

   ~~~bash
   kubectl proxy
   Starting to serve on 127.0.0.1:8001
   ~~~

   测试

   ~~~bash
   curl http://localhost:8001/api/v1/namespaces/default/pods/nginx/proxy/
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   <style>
       body {
           width: 35em;
           margin: 0 auto;
           font-family: Tahoma, Verdana, Arial, sans-serif;
       }
   </style>
   </head>
   <body>
   <h1>Welcome to nginx!</h1>
   <p>If you see this page, the nginx web server is successfully installed and
   working. Further configuration is required.</p>
   
   <p>For online documentation and support please refer to
   <a href="http://nginx.org/">nginx.org</a>.<br/>
   Commercial support is available at
   <a href="http://nginx.com/">nginx.com</a>.</p>
   
   <p><em>Thank you for using nginx.</em></p>
   </body>
   </html>
   
   ~~~

2. 端口 forward 

   可以对 pods 端口与宿主机端口作映射，通过宿主机 IP:<Port>进行访问

   ~~~bash
   kubectl port-forward nginx-554b9c67f9-d9hs6 31080:80
   ~~~

   访问 http://127.0.0.1:31080

   ![k8s-port-forward](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/9aoyp.png)

3. 端口 expose

   将资源暴露为新的Kubernetes Service。

   指定deployment、service、replica set、replication controller或pod，并使用该资源的选择器作为指定端口上新服务的选择器。deployment 或 replica set只有当其选择器可转换为service支持的选择器时，即当选择器仅包含matchLabels组件时才会作为暴露新的Service。

   `kubectl expose deployment nginx --port=80 --type=NodePort `
   
   ![k8s-get-svc](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/fwgtz.png)



## Try it out:

部署 friendlyhello 到 k8s 集群

准备相关文件：

app.py 

~~~python
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

~~~

Dockerfile

~~~dockerfile
FROM python:3.7-slim

WORKDIR /app

COPY . /app

RUN pip install flask redis -i https://mirrors.aliyun.com/pypi/simple  --trusted-host mirrors.aliyun.com
EXPOSE 5000

CMD ["python", "app.py"]

~~~

构建镜像：

`docker build -t friendlyhello:v4 .` 

部署：

~~~bash
kubectl run friendlyhello --image=friendlyhello:v4 --port=8080 --image-pull-policy=Never
~~~

![k8s-deploy-run](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/96cwk.png)

尝试访问

~~~bash
kubectl port-forward friendlyhello-f7869687d-wm4t6 32080:5000

~~~

![k8s-test-hello](https://blog-1252790741.cos.ap-shanghai.myqcloud.com/imgs/y2xfv.png)

