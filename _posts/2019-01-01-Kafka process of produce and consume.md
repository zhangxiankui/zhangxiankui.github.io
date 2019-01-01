---
layout: post
title:  "Kafka消息生产和消费的过程"
categories: kafka
tags:  produce consume
author: zhangxiankui
---

* content
{:toc}


## 一.消息生产
#### 1.生产者数据发送
- ①生产者发送数据时，需要制定kafka的消息的集群地址和端口号以及要发送到的哪个topic中，当然每一条消息中需要包含发送到对应的patition中（基于socket协议发送数据）
- ②当消费者接收到发送的数据时，根据topic和patition信息，对应的patition的leader节点会处理这条消息
- ③会在patition的末尾追加这条消息，顺序写入磁盘中，具体存储详见Kafka数据存储总结


## 二.消息消费
#### 1.消费者节点注册
##### ①当消费者消费kafka数据时，kafka会在zookeeper上为该消费者注册节点/consumers/groupid，每个消费者是属于某个消费者组的，就是上面说的groupid
##### ②groupid节点下面有三个子节点，分别是ids，用于存储消费者信息，offsets，用于存储每个topic中partition的offset值。owners用于存储该消费者组里面的topic;offsets下面目录结构是/topic/partition/partitionid/offset，这样消费者就能知道从哪里去消费
#### 2.消费者消费模式
##### ①kafka有两种消费模式，分别是队列模式和发布订阅模式，大部分情况都是使用发布订阅模式
##### ②对于队列模式，指的是当消费者组只有一个消费者并且只有一个分区的情况，因为队列要求消息是FIFO的
##### ③对于发布订阅模式，指的是多消费者多分区情况

#### 3.消费者消费消息
- ①消费者在消费某个topic时，kafka会根据zookeeper节点中partition的offset值去定位消费，消费者会去消费topic下的所有partition消息，采用某种策略（如轮询），
每个partition里面的消息能够保证顺序性;但是需要注意的是，每一个确定的时刻，同一个消费者组中只能有一个消费者消费同一个partition
- ②kafka拿到对应分区的offset值时，通过segment的文件名进行二分法查找offset对应值落在哪个分区中，然后在通过index文件，查找对应的物理偏移量并定位到对应的数据消息，并开始消费

## 四、kafka之zookeeper的消费者节点图
![](https://zhangxiankui.github.io/imgs/kafka/kafka-consume-zokeeper.png)
                  
