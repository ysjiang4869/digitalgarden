---
{"dg-publish":true,"dg-permalink":"kafka01","permalink":"/kafka01/","title":"Kafka入门教程 (一) 基本概念","tags":["Kafka"],"created":"2021-06-15 22:00","updated":"2024-08-19 09:45"}
---


# Kafka入门教程 (一) 基本概念

## Topic

一个Topic代表了一类资源，一种事件。比如用户上传的数据可以是一个topic，系统产生的事件也可以是一个topic

## Broker

一个broker代表一个kafka实例，通常建议一台物理机配置一个kafka实例，因为配置多个磁盘的IO限制也注定了性能不会提升太多

## Partition

一个Topic可以创建多个Partition，一个partition就是一个存储kafka数据的文件（称为log），每个partition内部，消息是顺序排列的。

由于Partition是顺序写磁盘，不需要关心锁的问题，保证了Kafka的高吞吐量；每个partition内的数据是有序的。多个partition可以解决磁盘IO的性能限制，同时，也可以通过指定数据发送给kafka时的key，key被用来决定数据放到哪个partition，这样就消费数据时，某个partition的数据就是按照某个规则有序的了。

Partition和broker没有必然联系（可以参考[Partition分配到broker的策略](https://www.jianshu.com/p/5c4a915843a4)），但是一个partition只能位于一个broker上。

## 4. Consumer

消费者，消费Kafka中存储的数据，一个consumer可以消费多个partition（同topic/不同topic）

## 5. Consumer Group

消费者组。一个消费者组可以有多个消费者，当使用高级消费者(high level)时，只需要指定订阅(subscribe)的topic，属于同一消费者组的消费者会根据一定规则分配消费该topic的某几个partition；使用低级消费时，则需要指定consumer分配(assign)的topic和partition，指定哪个partition就消费哪个，没有限制。

_**一个patition只能被同一个消费者组的一个订阅的consumer(高级)消费**_，但是可以被不同消费者组的多个consumer消费(订阅/分配均可)，或者被_**同一个组的一个订阅的consumer和任意个分配的consumer(低级)消费**_；这个限制是由client端实现的，详情参见本系列第二篇文章。

## 6. Offset

offset记录了某个consumer group在一个partition中消费数据的位置。由partition和group唯一确定。

## 7. Producer

生产者，生产数据到Kafka

## 系统示意图

![kafka01.png](https://s3.665210.xyz/pictures/Notion/programmer/kafka01.png)

- topic a：
    
    - p1 被groupMconsumerA 消费
    - p2 被groupMconsumerB 消费
    

    实现了一个topic被两个同group独立的consumer消费，提升消费速度。

    
- topicb:
    
    - p1 被groupMconsumerB和groupNconsumerC同时消费
    - p2 被groupMconsumerB和groupNconsumerC同时消费
    

    实现了多播的效果，所有到topicb的数据，consumerB和C同时接收到

    

## 参考链接

1. [Kafka背景及架构介绍](http://www.infoq.com/cn/articles/kafka-analysis-part-1)

2. [KafkaPartition与Broker的映射关系](https://www.jianshu.com/p/5c4a915843a4)
