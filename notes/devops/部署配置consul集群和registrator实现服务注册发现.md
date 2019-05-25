部署配置consul集群和registrator实现服务注册发现
---
### consul集群部署
#### 主机需求
    
名字|IP
---|---
consul-server1|172.10.0.2
consul-server2|172.10.0.3
consul-server3|172.10.0.4
consul-client1|172.10.0.5

ps:由于简化，就没有搞那么多虚拟机,直接新建docker的网络  
`docker network create --subnet=172.10.0.0/16 mynetwork`
    
#### docker部署consul集群  

* consul-server1

    ```bash
    docker run -d \
        --name=consul-server1 \
        --net=mynetwork \
        --restart=always \
        -h consul-server1 \
        consul agent \
        -server \
        -bind=172.10.0.2 \
        -bootstrap-expect=2 \
        -node=consul-server1 \
        -data-dir=/tmp/data-dir \
        -client 0.0.0.0 \
        -ui
     ```
* consul-server2

     ```bash
     docker run -d \
         --name=consul-server2 \
         --net=mynetwork \
         --restart=always \
         -h consul-server2 \
         consul agent \
         -server \
         -bind=172.10.0.3 \
         -join=172.10.0.2 \
         -bootstrap-expect=2 \
         -node=consul-server2 \
         -data-dir=/tmp/data-dir \
         -client 0.0.0.0 \
         -ui
     ```
* consul-server3

     ```bash
     docker run -d \
         --name=consul-server3 \
         --net=mynetwork \
         --restart=always \
         -h consul-server3 \
         consul agent \
         -server \
         -bind=172.10.0.4 \
         -join=172.10.0.2 \
         -bootstrap-expect=2 \
         -node=consul-server3 \
         -data-dir=/tmp/data-dir \
         -client 0.0.0.0 \
         -ui
     ```
* consul-client1 

     ```bash
     docker run -d \
        --name=consul-client1 \
        -p 8300-8302:8300-8302/tcp \
        -p 8500:8500/tcp \
        -p 8301-8302:8301-8302/udp \
        -p 8600:8600/tcp \
        -p 53:8600/udp \
        --net=mynetwork \
        --restart=always \
        -h consul-client1 \
        consul agent \
        -bind=172.10.0.5 \
        -retry-join=172.10.0.2 \
        -node=consul-client1 \
        -client 0.0.0.0 \
        -ui
     ```
 ps: 由于我们使用的是docker网络，我们本地是无法直接使用容器的端口，所以在client1 中将容器的端口暴露出来，以供外部使用

#### docker部署gliderlabs/registrator集群
* [官方文档](https://gliderlabs.github.io/registrator/latest/)
* 启动命令
```bash
docker run -d \
    --name=registrator \
    --net=mynetwork \
    --volume=/var/run/docker.sock:/tmp/docker.sock \
    gliderlabs/registrator:latest \
    consul://172.10.0.2:8500
```
ps: 
1. 这里我们可以随便使用一个consul作为注册都行
2. registrator 是通过监听docker的api来实现注册的
3. 再启动容器中注入环境变量就可以实现使用服务的注册名字

#### 启动测试服务
* 启动一个nginx服务测试注册
 ```bash
 docker run \
     -e 'SERVICE_NAME=nginx' \
     -e 'SERVICE_TAGS=a' \
     --net=mynetwork \
     --ip 172.10.0.124  \
     --name=nginx-consul \
     --rm\
     -d \
     -p 80  \
     nginx
 ```
* 查看启动的服务
`docker run --net=mynetwork --rm -it appropriate/curl curl 172.10.0.124:80`
输出
```html
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
```
说明我们服务是已经正常启动了

* 查看consul是否已经成功注册
`http://127.0.0.1:8500/v1/catalog/service/nginx`
输出
```json
[
    {
        "ID":"1d6b4e83-afae-a09c-968b-c23b2fd454b9",
        "Node":"consul-server1",
        "Address":"172.10.0.2",
        "Datacenter":"dc1",
        "TaggedAddresses":{
            "lan":"172.10.0.2",
            "wan":"172.10.0.2"
        },
        "NodeMeta":{
            "consul-network-segment":""
        },
        "ServiceKind":"",
        "ServiceID":"4f31ea13ce0d:nginx-consul:80",
        "ServiceName":"nginx",
        "ServiceTags":[
            "a"
        ],
        "ServiceAddress":"172.10.0.124",
        "ServiceWeights":{
            "Passing":1,
            "Warning":1
        },
        "ServiceMeta":{

        },
        "ServicePort":32768,
        "ServiceEnableTagOverride":false,
        "ServiceProxyDestination":"",
        "ServiceProxy":{

        },
        "ServiceConnect":{

        },
        "CreateIndex":12132,
        "ModifyIndex":12132
    }
]
```
ps: 这里能访问，是在启动consul-client的时候后将端口暴露出来，在本机可以访问，也可以通过浏览器的方式查看consul服务的状态`http://127.0.0.1:8500/ui/dc1/services`

* 测试consul是否能正常服务发现(dns方式)    
    ps: 先前已经将53/udp端口暴露出来，所以只需要将dns地址填写我的本机地址(192.168.0.251)就行了  
  测试命令  
  `docker run --net=mynetwork --dns=192.168.0.251 --rm -it appropriate/curl curl nginx.service.consul`  
  可以看到返回的结果和先前的是一样
```html
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
```

* consul会给我们做负载均衡  
我们都知道dns可以做负载均衡，那么问题来了，我们使用consul做dns服务发现，那是否就可以做负载均能呢？   
答案是肯定的，这里就不多赘述的，感兴趣的朋友可以去尝试一下

#### 总结  
* consul负载均衡的微服务应用有很多策略和技术，这里只是简单地说明了其中一种比较常用的
* consul+registrator可以做到不侵入代码的情况就可以做到服务的注册发现