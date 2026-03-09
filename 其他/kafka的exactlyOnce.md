# Kafka的exactly once策略

## 背景
在之前的一个项目，为了保证数据质量，需要使用kafka的Kafka的exactly once

## 幂等生产者

Kafka 给每个 Producer 分配：PID，同时会给每个消息分配一个sequence number，Broker 会检查：(PID, partition, sequence)，当存在重复的消息就会丢弃。幂等生产者解决 Producer 重试导致的重复。

## kafka事务
Kafka 把 生产消息 + offset提交 放进 一个事务。
``` 
    beginTransaction()

    读取topicA

    处理数据

    发送topicB

    提交offset

    commitTransaction()
```
这样就可以保证
写入topicB + offset提交
要么一起成功
要么一起失败

## 业务层保证
1 幂等写入

2 去重表

3 业务幂等