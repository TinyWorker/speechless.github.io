# Spark Streaming 编程指南 #

***

## 概述
Spark Streaming是Spark API的扩展部分，允许可扩展，高吞吐量，容错的流处理机制。数据可以通过多种来源获取，如kafka，Flume，Kinesis或TCP socket，可以使用复杂算法和高阶方法如map，reduce，join和window来表现数据处理。最终，处理的数据能推送到文件系统，数据库和活跃面板。

Spark的机器学习和图像处理也可以作为数据流输入。
![streaming-arch](https://tinyworker.github.io/images/streaming-arch.png)

在内部，工作机制如下图所示。
![streaming-flow](https://tinyworker.github.io/images/streaming-flow.png)

Spark Streaming接收活跃输入数据流并将数据拆分成批次，批次会由Spark Engine来生成结果的批次流。

Spark Streaming提供高阶的抽象，称为discretized stream或DStream，代表的连续流数据。DStream能从输入流或kafka等来源生成，或者在其他DStream上进行高阶操作。DStream是RDD的一个序列。


## 快速实例

我们首先来看一个简单的Spark Streaming的编程实例。该实例从一个监听TCP端口的数据服务获取text，然后统计单词的数量。


第一步，导入Spark Streaming的相关类和StreamingContext的一些隐式转换，这能够添加我们所需的方法（如DStream）。StreamingContext是所有Streaming方法的主要入口点。

	import org.apache.spark._
	import org.apache.spark.streaming._
	import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3
	
	// Create a local StreamingContext with two working thread and batch interval of 1 second.
	// The master requires 2 cores to prevent a starvation scenario.
	
	val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
	val ssc = new StreamingContext(conf, Seconds(1))
	
使用该context，创建一个DStream来代表从TCP来源的数据流，需要声明hostname和port

	// Create a DStream that will connect to hostname:port, like localhost:9999
	val lines = ssc.socketTextStream("localhost", 9999)
	
lines是DStream，DStream中的每条记录是一行文本。我们将文本分开成单词。

	// Split each line into words
	val words = lines.flatMap(_.split(" "))
	
flatMap方法是一对多的DStream操作，会通过从每条DStream的记录生成多条新记录来创建一个新的DStream，在本例中，行被拆分成了多个单词，words为新的DStream。

	import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3
	// Count each word in each batch
	val pairs = words.map(word => (word, 1))
	val wordCounts = pairs.reduceByKey(_ + _)
	
	// Print the first ten elements of each RDD generated in this DStream to the console
	wordCounts.print()
	
words这个DStream被用于做一对一的映射转换为DStream (word,1)Pair，会被减少来获取每批次的单词频率。最后，会将生成的统计打印出来。


但是，Spark Streaming只是将其过程设置好，不会有真正的处理，要让其开始这个处理流程，需要调用启动方法。

	ssc.start()             // Start the computation
	ssc.awaitTermination()  // Wait for the computation to terminate


## 基本概念


###依赖
Spark Streaming与Spark类似，可以通过Maven获取依赖。

	<dependency>
	    <groupId>org.apache.spark</groupId>
	    <artifactId>spark-streaming_2.11</artifactId>
	    <version>2.3.0</version>
	</dependency>

要从其他来源如Kafka，Flume，Kinesis获取数据，需要添加依赖，当前这些源不包含在Spark Streaming的核心API中。

| Source  | Artifact |
| ------------- | ------------- |
| Kafka  | spark-streaming-kafka-0-10_2.11 |
| Flume  | spark-streaming-flume_2.11  |
| Kinesis  | spark-streaming-kinesis-asl_2.11  |


### 初始化StreamingContext

初始化Spark Streaming程序，StreamingContext对象是必要条件。

可以从SparkConf对象中创建

	val conf = new SparkConf().setAppName(appName).setMaster(master)
	val ssc = new StreamingContext(conf, Seconds(1))

APPName是设置应用的名称，能在集群UI上看到，master是Spark，Mesos，YARN集群的地址，或者本地模式“local[*]”。在集群运行时，最好不要硬编码master，而是用spark-submit来启动。

必须根据您的应用程序的延迟要求和可用的群集资源来设置批处理间隔。

StreamingContext也可以从一个已存在的SparkContext中创建。

当一个context定义后，需要做如下事宜：

- 创建输入DStream来定义输入源。
- 转换并输出到DStream来定义Streaming的计算操作。
- 使用StreamingContext.start()开始接收数据并处理。
- 使用StreamingContext.awaitTermination()，等待处理结束（手动或遭遇错误）。
- 使用StreamingContext.stop()来手动停止处理流程。
 
需要记住的几个要点：

- 当context启动后，没有新的Streaming操作可以添加或设置。
- 当context停止后，不能重新启动。
- JVM中同时只允许一个StreamingContext活跃。
- stop()会同时关闭SparkContext，若仅停止StreamingContext，在方法中设置为false（stop(false))。
- SparkContext能重复使用来创建多个StreamingContext，只要之前的StreamingContext停止。

### Discretized Streams（DStreams）

Discretized Stream或DStream是Spark Streaming提供的基本抽象。代表连续的数据流，或输入流数据，或从输入流转换生成的处理数据流。在内部，DStream代表连续的RDD，这是Spark对不变的分布式数据集的抽象，DStream中的每个RDD都包含特定间隔的数据。

![streaming-dstream](https://tinyworker.github.io/images/streaming-dstream.png)

任意DStream接收的操作会转换成RDD的操作，例如，在将行流转换为单词的较早示例中，flatMap操作应用于行DStream中的每个RDD，以生成单词DStream的RDD。 如下图所示。

![streaming-dstream-ops](https://tinyworker.github.io/images/streaming-dstream-ops.png)

这些基础的RDD转换由Spark引擎计算。 DStream操作隐藏了大多数这些细节，并为开发人员提供了更高级别的API，以方便使用。 这些操作将在后面的部分中详细讨论。

### Input DStreams and Receivers
