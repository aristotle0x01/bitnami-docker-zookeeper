# play with zookeeper cluster locally

# setup cluster

## docker source

为了简单，直接使用docker集群进行local部署

[Docker Official Images](https://docs.docker.com/docker-hub/official_repos/)

**docker-compose file:**

`version: '3.1'



services:

  zoo1:

​    image: zookeeper

​    restart: always

​    hostname: zoo1

​    ports:

​      \- 2181:2181

​      \- 9081:8080

​    environment:

​      ZOO_MY_ID: 1

​      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181



  zoo2:

​    image: zookeeper

​    restart: always

​    hostname: zoo2

​    ports:

​      \- 2182:2181

​      \- 9082:8080

​    environment:

​      ZOO_MY_ID: 2

​      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zoo3:2888:3888;2181



  zoo3:

​    image: zookeeper

​    restart: always

​    hostname: zoo3

​    ports:

​      \- 2183:2181

​      \- 9083:8080

​    environment:

​      ZOO_MY_ID: 3

​      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181`

## cluster control

`docker-compose -f stack.yml up`

`docker-compose -f stack.yml down`

## zkCli.sh 连接zookeeper

**1.命令方式**

`zkCli.sh -server 127.0.0.1:2182`

**2.docker方式**

1) 首先确认zk集群使用的network

`docker network ls`

2) 连接某台server

`docker run -it --network=bitnami-docker-zookeeper_default --rm --link bitnami-docker-zookeeper_zoo3_1:zookeeper zookeeper zkCli.sh -server bitnami-docker-zookeeper_zoo3_1
:2183`

**bitnami-docker-zookeeper_default**为network名称

**bitnami-docker-zookeeper_zoo3_1**为server名称



# tcpdump数据分析

## 创建tcpdump镜像

为什么不直接使用本地命令呢？是为了方便通过“--net=container:your id” attach到其它container上

**建立镜像**

`docker build -t tcpdump - <<EOF 
FROM ubuntu 
RUN apt-get update && apt-get install -y tcpdump 
VOLUME ["/tcpdump"]
CMD tcpdump -i eth0 
EOF`

## 启动命令

`docker run -v /your/local/folder:/tcpdump --tty --net=container:"your container id" tcpdump tcpdump -tttt -s0 -X -vv tcp port 2181 -w /tcpdump/captcha.cap`

增加volume是为了将文件dump到本地便于wireshark分析

**其它简单命令**

```
docker run --tty --net=container:6628d9a4e3e5 tcpdump
docker run --tty --net=container:6628d9a4e3e5 tcpdump tcpdump -N -A 'port 8080'
```

## ref

[How to TCPdump effectively in Docker](https://xxradar.medium.com/how-to-tcpdump-effectively-in-docker-2ed0a09b5406)

[Using tcpdump With Docker](https://rmoff.net/2019/11/29/using-tcpdump-with-docker/)

