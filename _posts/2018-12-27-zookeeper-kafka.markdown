---
layout:     post
title:      "doker zookeeper kafka单机搭建"
subtitle:   " \"集群搭建\""
date:       2018-12-12 21:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - docker
    - zookeeper
    - kafka
---

> “Never give up. ”


## 前言

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因工作需求,环境需要搭建zookeeper,kafka集群,也是刚接触从零开始搭建,此文章是用docker搭建的单机版,
搭建过程还是比较顺利的,此前用传统方式搭建了一遍,后面才知道用docker也可以,用了docker之后发现很方便

<p id = "build"></p>
---

## 正文

#### 首先拉取镜像
 docker pull zookeeper:latest
 
 docker pull wurstmeister/kafka:lastest
  
#### 启动容器
启动zookeeper
```
 docker run -d --name zookeeper --publish 2181:2181 \
 --volume /etc/localtime:/etc/localtime \
 zookeeper:latest
```
启动kafka
```
 docker run -d --name kafka --publish 9092:9092 \
 --link zookeeper \
 --env KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
 --env KAFKA_ADVERTISED_HOST_NAME=kafka所在宿主机的IP \
 --env KAFKA_ADVERTISED_PORT=9092 \
 --volume /etc/localtime:/etc/localtime \
 wurstmeister/kafka:latest
 ```
进入kafka发送消息
查看 docker kafka id
```
 docker exec -it {id} /bin/bash
```

进入kafka目录 /opt/kafaka_xx_xx 
创建topic  名为test
```
 bin/kafka-topics.sh --create --zookeeper zookeeper:2181 --replication-factor 1 --partitions 1 --topic test
```

查看topic列表
```
  bin/kafka-topics.sh --list --zookeeper zookeeper:2181
``` 
创建生产者输入测试信息
```
 bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
```
创建消费者接受信息
```
 bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```



 
## 后记

over

—— linyi 2018.12.27


