---
layout: post
title:  "Kafka数据存储"
categories: kafka
tags:  kafka 数据存储
author: zhangxiankui
---

* content
{:toc}


## 一、kafka的数据存储
#### 1.和其他MQ消息中间件不同，kafka中的topic只是一个逻辑概念，真正消费的最小单位是partition;而kafka存储的最小单元是segment;
#### 2.当生产者不断向kafka发送数据时，先去定位到消息存储的指定分区，然后会在分区下创建segment文件，每个segment会存储两个文件，分别为filename.index和filename.log文件;
- ①其中filename是由32位数字组成，为当前segment内存储offset的最小值（高位补零）
- ②.log文件存储的为生产者发送的消息数据，按照offset从小到大依次排列，当有新消息发送至kafka时，数据会被顺序写到该segment的最尾端，也就是.log文件的最后
```
0000123.log文件结构类似于
data123
data124
data125
`````
- ③.index文件存储的是相对offset值和物理偏移地址的对应关系，也就是一种键值对
```
（1）0000123.index的文件结构类似于
id    偏移量
0          0
1        134
2        456
（2）其中id表示该segment中第i个元素，后面的偏移量表示第i个元素和第一个元素的内存偏移量
```

## 二、kafka中segment存储结构图
![](https://zhangxiankui.github.io/imgs/kafka/kafka-segment.png)

## 三.kafka数据存储的优势
#### 1.kafka基于磁盘的顺序读写能够达到600M/S的速度，完全不亚于内存读写的速度，并且由于它是基于磁盘的，所以对数据丢失有很好的处理方案
#### 2.kafka采用多个partition来进行数据的分布式存储，避免单点带来的性能瓶颈，这样就提高了kafka整体的消息处理性能，所以当kafka存在性能瓶颈时，可以考虑使用增加partition的方式解决
#### 3.kafka在partition中使用了分段的概念
- ①这种设计使得消息的过期删除变得更加方便（直接清楚某一分段的文件即可），kafka消息是可以设置失效时间的
- ②分段维护index和log文件能够提升整体的存储效率
- ③分段之后，当数据消费时可以通过二分法查找对应的segment，然后过去offset所在位置并进行消费，效率很高

## 四、kafka存储架构图
![](https://zhangxiankui.github.io/imgs/kafka/kafka-storage.png)
                  
