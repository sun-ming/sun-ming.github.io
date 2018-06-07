---
layout: post
title: "Structured Streaming教程(3) —— 与Kafka的集成"
date: 2017-10-27 19:30:00
categories: 大数据 Spark 实时计算
tags: 大数据 Spark 实时计算
author: Sun-Ming
---

* content
{:toc}

## 前言

Structured Streaming最主要的生产环境应用场景就是配合kafka做实时处理，不过在Strucured Streaming中kafka的版本要求相对搞一些，只支持0.10及以上的版本。就在前一个月，我们才从0.9升级到0.10，终于可以尝试structured streaming的很多用法，很开心~




## 引入

如果是maven工程，直接添加对应的kafka的jar包即可:
```
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql-kafka-0-10_2.11</artifactId>
    <version>2.2.0</version>
</dependency>
```
## 读取kafka的数据
### 以流的形式查询

读取的时候，可以读取某个topic，也可以读取多个topic，还可以指定topic的通配符形式：

* 读取一个topic
```
val df = spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("subscribe", "topic1")
  .load()
df.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
  .as[(String, String)]
```

* 读取多个topic
```
val df = spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("subscribe", "topic1,topic2")
  .load()
df.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
  .as[(String, String)]
```

* 读取通配符形式的topic组
```
val df = spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("subscribePattern", "topic.*")
  .load()
df.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
  .as[(String, String)]
```

### 以批的形式查询

关于Kafka的offset，structured streaming默认提供了几种方式：

* 设置每个分区的起始和结束值
```
val df = spark
  .read
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("subscribe", "topic1,topic2")
  .option("startingOffsets", """{"topic1":{"0":23,"1":-2},"topic2":{"0":-2}}""")
  .option("endingOffsets", """{"topic1":{"0":50,"1":-1},"topic2":{"0":-1}}""")
  .load()
df.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
  .as[(String, String)]
```

* 配置起始和结束的offset值（默认）
```
val df = spark
  .read
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("subscribePattern", "topic.*")
  .option("startingOffsets", "earliest")
  .option("endingOffsets", "latest")
  .load()
df.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
  .as[(String, String)]
```

### Schema信息

读取后的数据的Schema是固定的，包含的列如下：

|Column   		|	Type   	|说明	|
| :-----: 		| :----:	|:----: |
|key 			|	binary 	|	信息的key|
|value			| 	binary 	|	信息的value(我们自己的数据)|
|topic 			|	string 	|	主题|
|partition 		|	int 	|	分区|
|offset			|	long 	|	偏移值|
|timestamp 		|	long 	|	时间戳|
|timestampType	| 	int 	|	类型|

### source相关的配置

无论是流的形式，还是批的形式，都需要一些必要的参数：

* kafka.bootstrap.servers kafka的服务器配置，host:post形式，用逗号进行分割，如host1:9000,host2:9000
* assign，以json的形式指定topic信息
* subscribe，通过逗号分隔，指定topic信息
* subscribePattern，通过java的正则指定多个topic

assign、subscribe、subscribePattern同时之中能使用一个。

其他比较重要的参数有：

* startingOffsets, offset开始的值，如果是earliest，则从最早的数据开始读；如果是latest，则从最新的数据开始读。默认流是latest，批是earliest
* endingOffsets，最大的offset，只在批处理的时候设置，如果是latest则为最新的数据
* failOnDataLoss，在流处理时，当数据丢失时（比如topic被删除了，offset在指定的范围之外），查询是否报错，默认为true。这个功能可以当做是一种告警机制，如果对丢失数据不感兴趣，可以设置为false。在批处理时，这个值总是为true。
* kafkaConsumer.pollTimeoutMs，excutor连接kafka的超时时间，默认是512ms
* fetchOffset.numRetries，获取kafka的offset信息时，尝试的次数；默认是3次
* fetchOffset.retryIntervalMs，尝试重新读取kafka offset信息时等待的时间，默认是10ms
* maxOffsetsPerTrigger，trigger暂时不会用，不太明白什么意思。Rate limit on maximum number of offsets processed per trigger interval. The specified total number of offsets will be proportionally split across topicPartitions of different volume.

## 写入数据到Kafka

Apache kafka仅支持“至少一次”的语义，因此，无论是流处理还是批处理，数据都有可能重复。比如，当出现失败的时候，structured streaming会尝试重试，但是不会确定broker那端是否已经处理以及持久化该数据。但是如果query成功，那么可以断定的是，数据至少写入了一次。比较常见的做法是，在后续处理kafka数据时，再进行额外的去重，关于这点，其实structured streaming有专门的解决方案。

保存数据时的schema：

* key，可选。如果没有填，那么key会当做null，kafka针对null会有专门的处理（待查）。
* value，必须有
* topic，可选。（如果配置option里面有topic会覆盖这个字段）

下面是sink输出必须要有的参数：

* kafka.bootstrap.servers，kafka的集群地址，host:port格式用逗号分隔。

### 流处理的数据写入
```
// 基于配置指定topic
val ds = df
  .selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
  .writeStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("topic", "topic1")
  .start()

// 在字段中包含topic
val ds = df
  .selectExpr("topic", "CAST(key AS STRING)", "CAST(value AS STRING)")
  .writeStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .start()
```
### 批处理的数据写入

跟流处理其实一样
```
df.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
  .write
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("topic", "topic1")
  .save()

df.selectExpr("topic", "CAST(key AS STRING)", "CAST(value AS STRING)")
  .write
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .save()
```
## kafka的特殊配置

针对Kafka的特殊处理，可以通过DataStreamReader.option进行设置。

关于详细的kafka配置可以参考[consumer的官方文档](http://kafka.apache.org/documentation.html#newconsumerconfigs)

以及[kafka producer的配置](http://kafka.apache.org/documentation/#producerconfigs)

注意下面的参数是不能被设置的，否则kafka会抛出异常：

* group.id kafka的source会在每次query的时候自定创建唯一的group id
* auto.offset.reset 为了避免每次手动设置startingoffsets的值，structured streaming在内部消费时会自动管理offset。这样就能保证订阅动态的topic时不会丢失数据。startingOffsets在流处理时，只会作用于第一次启动时，之后的处理都会自定的读取保存的offset。
* key.deserializer，value.deserializer，key.serializer，value.serializer 序列化与反序列化，都是ByteArraySerializer
* enable.auto.commit kafka的source不会提交任何的offset
* interceptor.classes 由于kafka source读取数据都是二进制的数组，因此不能使用任何拦截器进行处理。

## 参考

* [官方文档](http://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)

