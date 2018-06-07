---
layout: post
title: "Structured Streaming教程(2) —— 常用输入与输出"
date: 2017-10-26 19:21:00
categories: 大数据 Spark 实时计算
tags: 大数据 Spark 实时计算
author: Sun-Ming
---

* content
{:toc}

## 前言

上篇了解了一些基本的Structured Streaming的概念，知道了Structured Streaming其实是一个无下界的无限递增的DataFrame。基于这个DataFrame，我们可以做一些基本的select、map、filter操作，也可以做一些复杂的join和统计。本篇就着重介绍下，Structured Streaming支持的输入输出，看看都提供了哪些方便的操作。




## 数据源

Structured Streaming 提供了几种数据源的类型，可以方便的构造Steaming的DataFrame。默认提供下面几种类型：

### File：文件数据源

file数据源提供了很多种内置的格式，如csv、parquet、orc、json等等，就以csv为例:
```
package xingoo.sstreaming

import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.types.StructType

object FileInputStructuredStreamingTest {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession
      .builder
      .master("local")
      .appName("StructuredNetworkWordCount")
      .getOrCreate()

    spark.sparkContext.setLogLevel("WARN")

    import spark.implicits._
    val userSchema = new StructType().add("name", "string").add("age", "integer")
    val lines = spark.readStream
      .option("sep", ";")
      .schema(userSchema)
      .csv("file:///Users/xingoo/IdeaProjects/spark-in-action/data/*")

    val query = lines.writeStream
      .outputMode("append")
      .format("console")
      .start()

    query.awaitTermination()
  }
}
```
这样，在对应的目录下新建文件时，就可以在控制台看到对应的数据了。
```
aaa;1
bbb;2
aaa;5
ddd;6
```
还有一些其他可以控制的参数：

* maxFilesPerTrigger 每个batch最多的文件数，默认是没有限制。比如我设置了这个值为1，那么同时增加了5个文件，这5个文件会每个文件作为一波数据，更新streaming dataframe。
* latestFirst 是否优先处理最新的文件，默认是false。如果设置为true，那么最近被更新的会优先处理。这种场景一般是在监听日志文件的时候使用。
* fileNameOnly 是否只监听固定名称的文件。

### socket网络数据源

在我们自己练习的时候，一般都是基于这个socket来做测试。首先开启一个socket服务器，`nc -lk 9999`，然后streaming这边连接进行处理。
```
  spark.readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()
```
### kafka数据源

这个是生产环境或者项目应用最多的数据源，通常架构都是：
```
应用数据输入-->kafka-->spark streaming -->其他的数据库
```
由于kafka涉及的内容还比较多，因此下一篇专门介绍kafka的集成。

## 输出

在配置完输入，并针对DataFrame或者DataSet做了一些操作后，想要把结果保存起来。就可以使用DataSet.writeStream()方法，配置输出需要配置下面的内容：

* format ： 配置输出的格式
* output mode：输出的格式
* query name：查询的名称，类似tempview的名字
* trigger interval：触发的间隔时间，如果前一个batch处理超时了，那么不会立即执行下一个batch，而是等下一个trigger时间在执行。
* checkpoint location：为保证数据的可靠性，可以设置检查点保存输出的结果。

### output Mode

详细的来看看这个输出模式的配置，它与普通的Spark的输出不同，只有三种类型：

* complete，把所有的DataFrame的内容输出，这种模式只能在做agg聚合操作的时候使用，比如ds.group.count，之后可以使用它
* append，普通的dataframe在做完map或者filter之后可以使用。这种模式会把新的batch的数据输出出来，
* update，把此次新增的数据输出，并更新整个dataframe。有点类似之前的streaming的state处理。

### 输出的类型

Structed Streaming提供了几种输出的类型：

* file，保存成csv或者parquet
```
noAggDF
  .writeStream
  .format("parquet")
  .option("checkpointLocation", "path/to/checkpoint/dir")
  .option("path", "path/to/destination/dir")
  .start()
```
* console，直接输出到控制台。一般做测试的时候用这个比较方便。
```
noAggDF
  .writeStream
  .format("console")
  .start()
```
* memory，可以保存在内容，供后面的代码使用
```
aggDF
  .writeStream
  .queryName("aggregates")
  .outputMode("complete")
  .format("memory")
  .start()
spark.sql("select * from aggregates").show()  
```
* foreach，参数是一个foreach的方法，用户可以实现这个方法实现一些自定义的功能。
```
writeStream
    .foreach(...)
    .start()
```
这个foreach的功能很强大，稍后也会详细的说明。
