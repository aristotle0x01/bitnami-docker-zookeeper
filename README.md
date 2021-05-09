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



