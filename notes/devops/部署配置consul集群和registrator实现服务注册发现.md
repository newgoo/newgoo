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

    ps:由于简化，就没有搞那么多虚拟机,直接新建docker的网络`docker network create --subnet=172.10.0.0/16 mynetwork`
    
#### docker部署consul集群  

* consul-server1

    ```sh
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

     ```sh
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

     ```
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

     ```
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