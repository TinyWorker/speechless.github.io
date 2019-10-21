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

### DStreams的输入和接收（Input DStreams and Receivers）

输入DStream是从流数据源接收的数据流的DStream对象。每个Input DStream与一个Receiver对象关联，该对象从源接收数据并将其存储在内存中。

Spark Streaming提供了两种内置的流数据源：

- 基础源：在StreamingContext API中直接可用，如文件系统，tcp连接。
- 高级源：如kafka，Flume，Kinesis，通过额外的工具类使用，需要对应的依赖。

如果需要在streaming应用中并行接收多个数据流，可以创建多个input DStream，这会创建多个receiver对象同时接收多个数据流。但是，Spark的worker/executor是一个长期运行的任务，所以它会占据一个核心分配给Streaming应用，确保Streaming应用分配到了足够的核心来接收并处理数据。

部分要点：

- 当本地运行Streaming程序时，不要使用local或local[1]，因为Receiver会占用一个线程，导致没有线程来处理接收的数据，因此，本地运行时使用local[n]作为master。
- 
- 当集群运行时，分配给Streaming的核心数必须大于receiver的数据。

#### Basic Source

示例中看过ssc.socketTextStream()，通过TCP连接接收文本数据。除socket意外，StreamingContext API提供了多种方法来创建DStream。


##### File Streams

从任意兼容HDFS API的文件系统中读取数据（HDFS,S3,NFS），通过StreamingContext.fileStream[KeyClass, ValueClass, InputFormatClass]。

文件系统不要求运行receiver，所以不必分配核心。

目录是如何监控的：

Spark Streaming会监控目录并处理任何在该目录下的文件。

- 简单的目录，比如"hdfs://namenode:8040/logs/"。直接属于该路径的文件在被发现时会被处理。
- POSIX glob pattern，比如"hdfs://namenode:8040/logs/2017*"。DStream由匹配的目录下所有文件组成，该方式用于匹配目录，而非目录下文件。
- 所有文件必须是统一的数据格式。
- 文件是基于其修改时间，而非创建时间。
- 当进程开始后，文件的修改不会导致文件重读，因为更新会被忽略。
- 目录下文件越多，扫描变更的时间就越久，即使没有任何修改。
- 若通配符用于匹配目录，重命名整个目录来匹配会将该目录添加到监控目录中，流中仅包含目录中修改时间在当前窗口内的文件。
- 调用FileSystem.setTimes（）修复时间戳是一种在以后的窗口中拾取文件的方法，即使其内容没有更改。

##### 使用对象存储数据源

HDFS之类的文件系统往往会在创建输出流后立即对其文件设置修改时间。 当打开文件时，甚至在数据完全写入之前，它也可能包含在DStream中-之后，将忽略同一窗口中对该文件的更新。 也就是说：更改可能会丢失，流中会省略数据。

为了确保在窗口中可以接收到更改，请将文件写入一个不受监视的目录，然后在关闭输出流后立即将其重命名为目标目录。 如果重命名的文件在创建窗口期间显示在扫描的目标目录中，则将提取新数据。

相反，由于实际复制了数据，因此诸如Amazon S3和Azure存储之类的对象存储通常具有较慢的重命名操作。 此外，重命名的对象可能具有named（）操作的时间作为其修改时间，因此可能不被视为原始创建时间暗示它们是该窗口的一部分。

需要对目标对象存储进行仔细的测试，以验证存储的时间戳行为与Spark Streaming期望的一致。 直接写入目标目录可能是通过所选对象存储流传输数据的适当策略。

##### 自定义Receiver

可以通过自定义Receiver来接收数据。

##### RDDs队列

可以基于RDD的队列创建DStream，每个推送到队列的RDD被认为是一个数据的批次，并按照流的方式处理。

#### 高级源

这类数据源会与非Spark依赖进行交互，有些有复杂的依赖包。因此，为了降低依赖的版本冲突，基于高级源的功能被移到了外部包。

在spark-shell中是不支持这些包的，若需要在shell中使用，需要下载对应jar包并添加到shell的classpath中。

#### 自定义源

仅需要实现自定义的receiver，然后从其中接收数据并推送到spark即可。


##### Receiver可靠性

根据其可靠性，可以有两种数据源。 源（例如Kafka和Flume）允许确认传输的数据。 如果从这些可靠来源接收数据的系统正确地确认了接收到的数据，则可以确保不会由于任何类型的故障而丢失任何数据。这指向两种receiver：

1. 可靠Receiver，当接收到数据并通过复制将其存储在Spark中时，可靠的接收器会正确地将确认发送到可靠的源。

2. 不可靠Receiver，不会将确认发送到源。 可以将其用于不支持确认的来源，甚至可以用于不希望或不需要进入确认复杂性的可靠来源。


### DStreams的转换操作

与RDD类似，转换操作运行DStream中的数据进行修改。DStream支持多种在RDD上的转换操作。


| 转换  | 功能 |
| ------------- | ------------- |
| map(func)  | 通过将源DStream的每个元素传递给函数func来返回新的DStream。|
| flatMap(func)  | 与map相似，但是每个输入项可以映射到0个或多个输出项。|
| filter(func)  | 通过仅选择func返回true的源DStream的记录来返回新的DStream。|
| repartition(numPartitions)	  | 通过创建更多或更少的分区来更改此DStream中的并行度。|
| union(otherStream)  | 返回一个新的DStream，其中包含源DStream和otherDStream中的元素的并集。|
| count()  | 通过计算源DStream的每个RDD中的元素数，返回一个新的单元素RDD DStream。|
| reduce(func)  | 通过使用函数func（带有两个参数并返回一个）来聚合源DStream的每个RDD中的元素，从而返回一个单元素RDD的新DStream。 该函数应具有关联性和可交换性，以便可以并行计算。|
| countByValue() | 在类型为K的元素的DStream上调用时，返回一个新的（K，Long）对的DStream，其中每个键的值是其在源DStream的每个RDD中的频率。|
| reduceByKey(func, [numTasks])	 | 在（K，V）对的DStream上调用时，返回一个新的（K，V）对的DStream，其中使用给定的reduce函数聚合每个键的值。 注意：默认情况下，这使用Spark的并行任务的默认数量（本地模式为2，在集群模式下，数量由config属性spark.default.parallelism确定）进行分组。 您可以传递一个可选的numTasks参数来设置不同数量的任务。|
| join(otherStream, [numTasks])	 | 当在（K，V）和（K，W）对的两个DStream上调用时，返回一个新的（K，（V，W））对的DStream，其中每个键都有所有元素对。|
| cogroup(otherStream, [numTasks])	 | 在（K，V）和（K，W）对的DStream上调用时，返回一个新的（K，Seq [V]，Seq [W]）元组的DStream。|
| transform(func) | 通过对源DStream的每个RDD应用RDD-to-RDD函数来返回新的DStream。 这可用于在DStream上执行任意RDD操作。|
| updateStateByKey(func) | 返回一个新的“状态” DStream，在该DStream中，通过在键的先前状态和键的新值上应用给定函数来更新每个键的状态。 这可用于维护每个键的任意状态数据。|


#### UpdateStateByKey

updateStateByKey操作使您可以保持任意状态，同时用新信息连续更新它，使用该操作有两步：

1. 定义状态，状态可以是任意的数据类型
2. 定义状态更新函数，定义如何更新状态的函数，使用之前的状态值和接收的新值。

在每个批次中，Spark都会对所有现有密钥应用状态更新功能，而不管它们是否在批次中具有新数据。 如果更新函数返回None，则将删除键值对。

	def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
	    val newCount = ...  // add the new values with the previous running count to get the new count
	    Some(newCount)
	}

这适用于包含单词的DStream

	val runningCounts = pairs.updateStateByKey[Int](updateFunction _)	

更新函数会在每个单词处调用。

注意updateStateByKey操作要求配置好检查点目录。

#### Transform

变换操作（以及它的诸如transformWith之类的变体）允许将任意RDD-to-RDD函数应用于DStream。 它可用于应用DStream API中未公开的任何RDD操作。 例如，将数据流中的每个批次与另一个数据集连接在一起的功能未直接在DStream API中公开。 但是可以使用transform来执行此操作。 这实现了非常强大的可能性。 例如，可以通过将输入数据流与预先计算的垃圾邮件信息（也可能由Spark生成）结合在一起，然后基于该信息进行过滤来进行实时数据清除。

	val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // RDD containing spam information
	
	val cleanedDStream = wordCounts.transform { rdd =>
	  rdd.join(spamInfoRDD).filter(...) // join data stream with spam information to do data cleaning
	  ...
	}

请注意，在每个批处理间隔中都会调用提供的函数。 这使您可以执行随时间变化的RDD操作，即可以在批之间更改RDD操作，分区数，广播变量等。

#### Window

Spark Streaming提供了窗口计算，允许对数据的滑动窗口应用转换。

![streaming-dstream-window](https://tinyworker.github.io/images/streaming-dstream-window.png)

如图所示，每当窗口在源DStream上滑动时，落入窗口内的源RDD就会合并并进行操作，以生成窗口DStream的RDD。 在这种特定情况下，该操作将应用于数据的最后3个时间单位，并以2个时间单位滑动。 这表明任何窗口操作都需要指定两个参数。

- window length，窗口的持续时间。
- sliding interval，窗口操作的执行间隔。

这两个参数必须是源DStream的批处理间隔的倍数。

	val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))

窗口操作的常用方法如下表：


| 转换  | 功能 |
| ------------- | ------------- |
| window(windowLength, slideInterval) | 返回基于源DStream的窗口批处理计算的新DStream。|
| countByWindow(windowLength, slideInterval) | 返回流中元素的滑动窗口计数。|
| reduceByWindow(func, windowLength, slideInterval)	 | 返回一个新的单元素流，该流是通过使用func在滑动间隔内聚合流中的元素而创建的。 该函数应该是关联的和可交换的，以便可以并行正确地计算它。|
| reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks])	 | 在（K，V）对的DStream上调用时，返回新的（K，V）对的DStream，其中使用给定的reduce函数func在滑动窗口中的批处理上聚合每个键的值。 注意：默认情况下，这使用Spark的并行任务的默认数量（本地模式为2，在集群模式下，数量由config属性spark.default.parallelism确定）进行分组。 您可以传递一个可选的numTasks参数来设置不同数量的任务。 |
| reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks])	| 上面的reduceByKeyAndWindow（）的更有效的版本，其中，使用前一个窗口的缩减值递增地计算每个窗口的缩减值。 这是通过减少进入滑动窗口的新数据并“逆向减少”离开窗口的旧数据来完成的。 一个示例是在窗口滑动时“增加”和“减少”键的计数。 但是，它仅适用于“可逆归约函数”，即具有相应“逆归约”功能（作为参数invFunc的归约）的归约函数。 像reduceByKeyAndWindow中一样，reduce任务的数量可以通过可选参数配置。 请注意，必须启用检查点才能使用此操作。|
| countByValueAndWindow(windowLength, slideInterval, [numTasks])	 | 在（K，V）对的DStream上调用时，返回新的（K，Long）对的DStream，其中每个键的值是其在滑动窗口内的频率。 像reduceByKeyAndWindow中一样，reduce任务的数量可以通过可选参数配置。|


#### Join

Spark Streaming中可以轻松执行不同的联结操作。

##### Stream-Stream joins

流之间的联结：

	val stream1: DStream[String, String] = ...
	val stream2: DStream[String, String] = ...
	val joinedStream = stream1.join(stream2)

##### Stream-dataset jions

流与数据集的联结

	val dataset: RDD[String, String] = ...
	val windowedStream = stream.window(Seconds(20))...
	val joinedStream = windowedStream.transform { rdd => rdd.join(dataset) }

实际上，您还可以动态更改要加入的数据集。 提供给转换的函数在每个批处理间隔中进行评估，因此将使用数据集引用所指向的当前数据集。


### DStream的输出（Output）

输出操作允许将DStream的数据推出到外部系统，例如数据库或文件系统。 由于输出操作实际上允许外部系统使用转换后的数据，因此它们会触发所有DStream转换的实际执行（类似于RDD的操作）。


| 输出  | 功能 |
| ------------- | ------------- |
| print()	 | 在运行流应用程序的驱动程序节点上，打印DStream中每批数据的前十个元素。 这对于开发和调试很有用。|
| saveAsTextFiles(prefix, [suffix]) | 将此DStream的内容另存为文本文件。 每个批处理间隔的文件名都是基于前缀和后缀“ prefix-TIME_IN_MS [.suffix]”生成的。|
| saveAsObjectFiles(prefix, [suffix]) | 将此DStream的内容另存为序列化Java对象的SequenceFiles。 每个批处理间隔的文件名都是基于前缀和后缀“ prefix-TIME_IN_MS [.suffix]”生成的。|
| saveAsHadoopFiles(prefix, [suffix]) | 将此DStream的内容另存为Hadoop文件。 每个批处理间隔的文件名都是基于前缀和后缀“ prefix-TIME_IN_MS [.suffix]”生成的。|
| foreachRDD(func)	| 最通用的输出运算符，将函数func应用于从流生成的每个RDD。 此功能应将每个RDD中的数据推送到外部系统，例如将RDD保存到文件或通过网络将其写入数据库。 请注意，函数func在运行流应用程序的驱动程序进程中执行，并且通常在其中具有RDD操作，这将强制计算流RDD。|

#### foreachRDD的设计模式

dstream.foreachRDD是功能强大的原语，它允许将数据发送到外部系统。 但是，重要的是要了解如何正确有效地使用此原语。 应避免的一些常见错误如下：

通常，将数据写入外部系统需要创建一个连接对象（例如，与远程服务器的TCP连接），然后使用该对象将数据发送到远程系统。 为此，开发人员可能会无意间尝试在Spark驱动程序中创建连接对象，然后尝试在Spark worker中使用该对象以将记录保存在RDD中。

	dstream.foreachRDD { rdd =>
	  val connection = createNewConnection()  // executed at the driver
	  rdd.foreach { record =>
	    connection.send(record) // executed at the worker
	  }
	}

这是不正确的，因为这要求将连接对象序列化并从驱动程序发送给工作程序。 这样的连接对象很少能在机器之间转移。 此错误可能表现为序列化错误（连接对象不可序列化），初始化错误（连接对象需要在工作程序中初始化）等。正确的解决方案是在工作程序中创建连接对象。

但是，这可能会导致另一个常见错误-为每个记录创建一个新的连接。 例如
	
	dstream.foreachRDD { rdd =>
	  rdd.foreach { record =>
	    val connection = createNewConnection()
	    connection.send(record)
	    connection.close()
	  }
	}

通常，创建连接对象会浪费时间和资源。 因此，为每个记录创建和销毁连接对象会导致不必要的高开销，并且会大大降低系统的整体吞吐量。 更好的解决方案是使用rdd.foreachPartition-创建一个连接对象，并使用该连接发送RDD分区中的所有记录。

	dstream.foreachRDD { rdd =>
	  rdd.foreachPartition { partitionOfRecords =>
	    val connection = createNewConnection()
	    partitionOfRecords.foreach(record => connection.send(record))
	    connection.close()
	  }
	}

这将分摊许多记录上的连接创建开销。

最后，可以通过在多个RDD /批次之间重用连接对象来进一步优化。 与将多个批次的RDD推送到外部系统时可以重用的连接对象相比，它可以维护一个静态的连接对象池，从而进一步减少了开销。

	dstream.foreachRDD { rdd =>
	  rdd.foreachPartition { partitionOfRecords =>
	    // ConnectionPool is a static, lazily initialized pool of connections
	    val connection = ConnectionPool.getConnection()
	    partitionOfRecords.foreach(record => connection.send(record))
	    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
	  }
	}

请注意，应按需延迟创建池中的连接，如果一段时间不使用，则超时。 这样可以最有效地将数据发送到外部系统。

一些注意要点：

- DStream由输出操作延迟执行，就像RDD由RDD操作延迟执行一样。 具体来说，DStream输出操作内部的RDD动作会强制处理接收到的数据。 因此，如果您的应用程序没有任何输出操作，或者内部没有任何RDD操作，则具有dstream.foreachRDD（）之类的输出操作，则将不会执行任何操作。 系统将仅接收数据并将其丢弃。

- 默认情况下，输出操作一次执行一次。 然后按照在应用程序中定义的顺序执行它们


### DataFrame和SQL

Streaming data中可以使用DataFrames和SQL操作，必须要基于StreamingContext所使用的SparkContext来创建SparkSession，此外，必须这样做，以便可以在驱动程序故障时重新启动它。 这是通过创建SparkSession的延迟实例化单例实例来完成的。

	val words: DStream[String] = ...
	
	words.foreachRDD { rdd =>
	
	  // Get the singleton instance of SparkSession
	  val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()
	  import spark.implicits._
	
	  // Convert RDD[String] to DataFrame
	  val wordsDataFrame = rdd.toDF("word")
	
	  // Create a temporary view
	  wordsDataFrame.createOrReplaceTempView("words")
	
	  // Do word count on DataFrame using SQL and print it
	  val wordCountsDataFrame = 
	    spark.sql("select word, count(*) as total from words group by word")
	  wordCountsDataFrame.show()
	}

您还可以在来自不同线程的流数据上定义的表上运行SQL查询（即与正在运行的StreamingContext异步）。 只要确保将StreamingContext设置为记住足够的流数据即可运行查询。 否则，不知道任何异步SQL查询的StreamingContext将在查询完成之前删除旧的流数据。 例如，如果您要查询最后一批，但是查询可能需要5分钟才能运行，请调用streamingContext.remember（Minutes（5））

## MLlib

您还可以轻松使用MLlib提供的机器学习算法。 首先，有流媒体机器学习算法（例如，流线性回归，流KMeans等），可以同时从流数据中学习以及将模型应用于流数据。 除此之外，对于更多种类的机器学习算法，您可以离线学习学习模型（即使用历史数据），然后在线将模型应用于流数据。 有关更多详细信息，请参见MLlib指南。

## Caching / Persistence

与RDD相似，DStreams还允许开发人员将流的数据持久存储在内存中。 也就是说，在DStream上使用persist（）方法将自动将该DStream的每个RDD持久保存在内存中。 如果DStream中的数据将被多次计算（例如，对同一数据进行多次操作），这将很有用。 对于基于窗口的操作（如reduceByWindow和reduceByKeyAndWindow）以及基于状态的操作（如updateStateByKey），这是隐式的。 因此，由基于窗口的操作生成的DStream会自动保存在内存中，而无需开发人员调用persist（）。

对于通过网络接收数据的输入流（例如，Kafka，Flume，套接字等），默认的持久性级别设置为将数据复制到两个节点以实现容错。

请注意，与RDD不同，DStream的默认持久性级别将数据序列化在内存中。


## 检查点（Checkpointing）

流式应用程序必须24/7全天候运行，因此必须对与应用程序逻辑无关的故障（例如，系统故障，JVM崩溃等）具有弹性。 为此，Spark Streaming需要将足够的信息检查点指向容错存储系统，以便可以从故障中恢复。 检查点有两种类型的数据：

- Metadata检查，元数据检查点-将定义流计算的信息保存到HDFS等容错存储中。 这用于从运行流应用程序的驱动程序的节点的故障中恢复。元数据包括：

	- 配置，用于创建流式应用的配置。
	- DStream操作，定义流式应用的DStream操作集合。
	- 未完成的批次，进入队列但未完成的批次任务。


- Data检查，将生成的RDD保存到可靠的存储中。 在某些状态转换中，这是必须的，这些转换将跨多个批次的数据进行合并。 在此类转换中，生成的RDD依赖于先前批次的RDD，这导致依赖项链的长度随时间不断增加。 为了避免恢复时间的无限增加（与依赖关系成比例），定期将有状态转换的中间RDD检查点指向可靠存储（例如HDFS），以切断依赖关系链。


总而言之，从驱动程序故障中恢复时，主要需要元数据检查点，而如果使用有状态转换，则即使是基本功能，也需要数据或RDD检查点。

### 什么时候使用检查点功能？

检查点在以下需求中必须启用：

- 状态化转换操作，无论使用updateStateByKey或reduceByKeyAndWindow，检查点目录必须提供以允许定期的RDD检查点。
- 从运行应用程序的驱动错误恢复，元数据检查点可用于恢复程序信息。

请注意，没有前述状态转换的简单流应用程序可以在不启用检查点的情况下运行。 在这种情况下，从驱动程序故障中恢复也将是部分的（某些丢失但未处理的数据可能会丢失）。 这通常是可以接受的，并且许多都以这种方式运行Spark Streaming应用程序。 预计将来会改善对非Hadoop环境的支持。

### 如何配置检查点

可以通过在容错，可靠的文件系统（例如HDFS，S3等）中设置目录来启用检查点，将检查点信息保存到该目录中。 这是通过使用streamingContext.checkpoint（checkpointDirectory）完成的。 这将允许您使用前面提到的有状态转换。 此外，如果要使应用程序从驱动程序故障中恢复，则应重写流应用程序以具有以下行为。

- 当程序第一次启动，会创建新的StreamingContext，设置所有的流并调用start()；
- 当程序从失败中重启，会从检查点数据中重新创建StreamingContext。


	def functionToCreateContext(): StreamingContext = {
	  val ssc = new StreamingContext(...)   // new context
	  val lines = ssc.socketTextStream(...) // create DStreams
	  ...
	  ssc.checkpoint(checkpointDirectory)   // set checkpoint directory
	  ssc
	}
	
	// Get StreamingContext from checkpoint data or create a new one
	val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)
	
	// Do additional setup on context that needs to be done,
	// irrespective of whether it is being started or restarted
	context. ...

	// Start the context
	context.start()
	context.awaitTermination()

如果存在checkpointDirectory，则将根据检查点数据重新创建上下文。 如果该目录不存在（即首次运行），则将调用函数functionToCreateContext来创建新上下文并设置DStreams。 请参阅Scala示例RecoverableNetworkWordCount。 本示例将网络数据的字数附加到文件中。

除了使用getOrCreate之外，还需要确保驱动程序进程在发生故障时自动重新启动。 这只能通过用于运行应用程序的部署基础结构来完成。 这将在“部署”部分中进一步讨论。

请注意，RDD的检查点会导致保存到可靠存储的成本。 这可能会导致RDD被检查点的那些批次的处理时间增加。 因此，需要仔细设置检查点的间隔。 在小批量（例如1秒）时，每批检查点可能会大大降低操作吞吐量。 相反，检查点太少会导致沿袭和任务规模增加，这可能会产生不利影响。 对于需要RDD检查点的有状态转换，默认间隔为批处理间隔的倍数，至少应为10秒。 可以使用dstream.checkpoint（checkpointInterval）进行设置。 通常，DStream的5-10个滑动间隔的检查点间隔是一个很好的尝试设置。

### 累加器，广播变量，检查点（Accumulators，Broadcast Variables，Checkpoints）

无法从Spark Streaming中的检查点恢复累加器和广播变量。 如果启用检查点并同时使用“累加器”或“广播”变量，则必须为“累加器”和“广播”变量创建延迟实例化的单例实例，以便在驱动程序发生故障重新启动后可以重新实例化它们。 在下面的示例中显示。

	object WordBlacklist {
	
	  @volatile private var instance: Broadcast[Seq[String]] = null
	
	  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
	    if (instance == null) {
	      synchronized {
	        if (instance == null) {
	          val wordBlacklist = Seq("a", "b", "c")
	          instance = sc.broadcast(wordBlacklist)
	        }
	      }
	    }
	    instance
	  }
	}
	
	object DroppedWordsCounter {
	
	  @volatile private var instance: LongAccumulator = null
	
	  def getInstance(sc: SparkContext): LongAccumulator = {
	    if (instance == null) {
	      synchronized {
	        if (instance == null) {
	          instance = sc.longAccumulator("WordsInBlacklistCounter")
	        }
	      }
	    }
	    instance
	  }
	}
	
	wordCounts.foreachRDD { (rdd: RDD[(String, Int)], time: Time) =>
	  // Get or register the blacklist Broadcast
	  val blacklist = WordBlacklist.getInstance(rdd.sparkContext)
	  // Get or register the droppedWordsCounter Accumulator
	  val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
	  // Use blacklist to drop words and use droppedWordsCounter to count them
	  val counts = rdd.filter { case (word, count) =>
	    if (blacklist.value.contains(word)) {
	      droppedWordsCounter.add(count)
	      false
	    } else {
	      true
	    }
	  }.collect().mkString("[", ", ", "]")
	  val output = "Counts at time " + time + " " + counts
	})


## 部署应用（Deploy Applications)

要运行Spark Streaming应用，需要有如下的条件：


- *集群需要有集群管理器*，这适用于任何Spark应用。

- *打包应用的jar*，需要编译应用为jar包。若使用哦spark-submit来启动应用，则jar包中不用提供Spark和Spark Streaming依赖，若使用高级源，需要在jar中包含对应的依赖。

- *为executor配置合适的内存*，由于接收的数据必须存储在内存中，executor必须配置足够的内存来持有数据，比如10分钟的窗口操作，意味着至少10分钟内数据必须保留。

- *配置检查点*，如果流应用程序需要它，则必须将Hadoop API兼容的容错存储中的目录（例如HDFS，S3等）配置为检查点目录，并且以这样的方式编写流应用程序，以便可以将检查点信息用于故障恢复。

- *配置Driver自动重启*，用于自动从driver失败中恢复，用于运行流应用程序的部署基础结构必须监视驱动程序进程，并在驱动程序失败时重新启动。 不同的集群管理器具有不同的工具来实现这一目标。

	- *Spark Standalone*，可以提交Spark应用程序驱动程序以在Spark Standalone集群中运行（请参阅集群部署模式），即，应用程序驱动程序本身在工作程序节点之一上运行。 此外，可以指示独立群集管理器监督驱动程序，并在驱动程序由于非零退出代码或由于运行驱动程序的节点故障而失败时重新启动它。 有关更多详细信息，请参见Spark Standalone指南中的集群模式和监督。

	- *YARN*，Yarn支持自动重启应用程序的类似机制。 请参阅YARN文档以获取更多详细信息。

- *配置预写日志*，我们引入了预写日志，以实现强大的容错保证。如果启用，则将从接收器接收的所有数据写入配置检查点目录中的预写日志。这样可以防止驱动程序恢复时丢失数据，从而确保零数据丢失（在“容错语义”部分中进行了详细讨论）。可以通过将配置参数spark.streaming.receiver.writeAheadLog.enable设置为true来启用此功能。但是，这些更强的语义可能会以单个接收器的接收吞吐量为代价。可以通过并行运行更多接收器以提高总吞吐量来纠正此问题。另外，由于启用了预写日志，因此建议禁用Spark中接收数据的复制，因为该日志已存储在复制的存储系统中。可以通过将输入流的存储级别设置为StorageLevel.MEMORY_AND_DISK_SER来完成。在使用S3（或任何不支持刷新的文件系统）作为预写日志时，请记住启用spark.streaming.driver.writeAheadLog.closeFileAfterWrite和spark.streaming.receiver.writeAheadLog.closeFileAfterWrite。有关更多详细信息，请参见Spark Streaming配置。请注意，启用I / O加密后，Spark不会加密写入预写日志的数据。如果需要对预写日志数据进行加密，则应将其存储在本地支持加密的文件系统中。

- *设置最大接收速率*，如果群集资源不够大，无法让流式传输应用程序尽快处理接收到的数据，则可以通过设置记录/秒的最大速率限制来限制接收器的速率。 请参阅配置参数spark.streaming.receiver.maxRate（用于接收器）和spark.streaming.kafka.maxRatePerPartition（用于直接Kafka方法）。 在Spark 1.5中，我们引入了一个称为背压的功能，该功能消除了设置此速率限制的需要，因为Spark Streaming会自动计算出速率限制，并在处理条件发生变化时动态调整它们。 可以通过将配置参数spark.streaming.backpressure.enabled设置为true来启用此背压。

### 更新应用代码
如果需要使用新的应用程序代码升级正在运行的Spark Streaming应用程序，则有两种可能的机制。

- 升级后的Spark Streaming应用程序将启动，并与现有应用程序并行运行。 一旦新的（接收的数据与旧的数据相同）已经预热并且准备好黄金时间，则可以将旧的数据降下来。 请注意，对于支持将数据发送到两个目标的数据源（即较早的和升级的应用程序），可以这样做。

- 正常关闭现有应用程序（有关正常关闭选项，请参阅StreamingContext.stop（...）或JavaStreamingContext.stop（...）），以确保在关闭之前可以完全处理已接收的数据。 然后可以启动升级的应用程序，它将从较早的应用程序停止的同一点开始处理。 请注意，只有使用支持源端缓冲的输入源（例如Kafka和Flume）才能完成此操作，因为在上一个应用程序关闭且升级的应用程序尚未启动时需要缓冲数据。 并且无法完成从升级前代码的较早检查点信息重新启动的操作。 检查点信息本质上包含序列化的Scala / Java / Python对象，尝试使用经过修改的新类反序列化对象可能会导致错误。 在这种情况下，请使用其他检查点目录启动升级的应用程序，或者删除先前的检查点目录。


## 监控应用（Monitoring Applications）
除了Spark的监视功能外，Spark Streaming还具有其他功能。 使用StreamingContext时，Spark Web UI会显示一个附加的Streaming选项卡，其中显示有关正在运行的接收器（接收器是否处于活动状态，接收到的记录数，接收器错误等）和已完成的批处理（批处理时间，排队延迟等）的统计信息。 ）。 这可用于监视流应用程序的进度。

Web UI中的以下两个指标特别重要：

- 处理时间-处理每批数据的时间。
- 调度延迟-批处理在队列中等待先前批处理完成的时间。

如果批处理时间始终大于批处理时间间隔和/或排队延迟不断增加，则表明系统无法像生成批处理一样快地处理批处理，并且落后于此。 在这种情况下，要考虑减少批处理时间。

还可以使用StreamingListener界面监视Spark Streaming程序的进度，该界面可让您获取接收器状态和处理时间。 请注意，这是开发人员API，将来可能会得到改进（即，报告了更多信息）


## 性能调试（Performance Tuning）
要在集群上的Spark Streaming应用程序中获得最佳性能，需要进行一些调整。 本节说明了可以调整以提高应用程序性能的许多参数和配置。 在较高级别上，需要考虑两件事：

1. 通过有效使用群集资源减少每批数据的处理时间。
2. 设置正确的批处理大小，以便可以在接收到批处理数据后尽快对其进行处理（也就是说，数据处理与数据摄取保持同步）。

### 减少批次处理时间
Spark可以进行许多优化，以最大程度地减少每批的处理时间。 这些已在《调优指南》中详细讨论。 本节重点介绍一些最重要的内容。

#### 数据接收的并行级别
通过网络（例如Kafka，Flume，套接字等）接收数据需要对数据进行反序列化并将其存储在Spark中。如果数据接收成为系统的瓶颈，请考虑并行化数据接收。请注意，每个输入DStream都会创建一个接收器（在工作计算机上运行），该接收器接收单个数据流。因此，可以通过创建多个输入DStream并将其配置为从源接收数据流的不同分区来实现接收多个数据流。例如，可以将接收两个主题数据的单个Kafka输入DStream拆分为两个Kafka输入流，每个输入流仅接收一个主题。这将运行两个接收器，从而允许并行接收数据，从而提高了总体吞吐量。这些多个DStream可以结合在一起以创建单个DStream。然后，可以将应用于单个输入DStream的转换应用于统一流。

	val numStreams = 5
	val kafkaStreams = (1 to numStreams).map { i => KafkaUtils.createStream(...) }
	val unifiedStream = streamingContext.union(kafkaStreams)
	unifiedStream.print()

应该考虑的另一个参数是接收器的块间隔，该间隔由配置参数spark.streaming.blockInterval确定。对于大多数接收器而言，接收到的数据在存储在Spark内存中之前会合并为数据块。每批中的块数确定了将在类似地图的转换中用于处理接收到的数据的任务数。每批接收器中每个接收器的任务数大约为（批处理间隔/块间隔）。例如，200 ms的块间隔将每2秒批处理创建10个任务。如果任务数太少（即少于每台计算机的核心数），那么它将效率低下，因为将不使用所有可用的核心来处理数据。要增加给定批处理间隔的任务数，请减小阻止间隔。但是，建议的块间隔最小值约为50毫秒，在此之下，任务启动开销可能是个问题。

使用多个输入流/接收器接收数据的一种替代方法是显式重新划分输入数据流（使用inputStream.repartition（<分区数>））。这将在进一步处理之前将接收到的数据批分布在集群中指定数量的计算机上

#### 数据处理的并行级别
如果在计算的任何阶段使用的并行任务数量不够高，则群集资源可能无法得到充分利用。 例如，对于诸如reduceByKey和reduceByKeyAndWindow之类的分布式归约操作，并行任务的默认数量由spark.default.parallelism配置属性控制。 您可以将并行性级别作为参数传递（请参见PairDStreamFunctions文档），或设置spark.default.parallelism配置属性以更改默认值。

#### 数据序列化

可以通过调整序列化格式来减少数据序列化的开销。 在流传输的情况下，有两种类型的数据正在序列化。

- 输入数据，默认情况下，通过Receivers接收的输入数据将通过StorageLevel.MEMORY_AND_DISK_SER_2存储在执行程序的内存中。 也就是说，数据被序列化为字节以减少GC开销，并被复制以容忍执行器故障。 同样，数据首先保存在内存中，并且仅在内存不足以容纳流计算所需的所有输入数据时才溢出到磁盘。 显然，这种序列化会产生开销-接收者必须对接收到的数据进行反序列化，然后使用Spark的序列化格式对其进行重新序列化。

- 流操作生成的持久性RDD，通过流计算生成的RDD可以保留在内存中。 例如，窗口操作会将数据保留在内存中，因为它们将被多次处理。 但是，与Spark Core默认的StorageLevel.MEMORY_ONLY不同，默认情况下，由流计算生成的持久性RDD会与StorageLevel.MEMORY_ONLY_SER（即序列化）保持一致，以最大程度地减少GC开销。

在这两种情况下，使用Kryo序列化都可以减少CPU和内存的开销。 有关更多详细信息，请参见《 Spark Tuning Guide》。 对于Kryo，请考虑注册自定义类，并禁用对象引用跟踪（请参阅《配置指南》中与Kryo相关的配置）。

在流应用程序需要保留的数据量不大的特定情况下，将数据（两种类型）持久化为反序列化对象是可行的，而不会产生过多的GC开销。 例如，如果您使用的是几秒钟的批处理间隔并且没有窗口操作，则可以尝试通过显式设置存储级别来禁用持久化数据中的序列化。 这将减少由于序列化导致的CPU开销，从而可能在没有太多GC开销的情况下提高性能。

#### 任务启动开销

如果每秒启动的任务数量很高（例如，每秒50个或更多），则向从服务器发送任务的开销可能会很大，并且将难以实现亚秒级的延迟。 可以通过以下更改来减少开销：

- 执行模式：在独立模式或粗粒度Mesos模式下运行Spark可以比细粒度Mesos模式更好地执行任务启动时间。 有关更多详细信息，请参阅“在Mesos上运行”指南。

这些更改可以将批处理时间减少100毫秒，从而使亚秒级的批处理大小可行。


## 设置正确的批次间隔（Setting the Right Batch Interval）

为了使在群集上运行的Spark Streaming应用程序稳定，系统应能够尽快处理接收到的数据。 换句话说，应尽快处理一批数据。 可以通过监视流式Web UI中的处理时间来找到对于应用程序是否正确的方法，其中批处理时间应小于批处理间隔。

根据流计算的性质，所使用的批处理间隔可能会对数据速率产生重大影响，该速率可以由应用程序在一组固定的群集资源上维持。 例如，让我们考虑前面的WordCountNetwork示例。 对于特定的数据速率，系统也许能够跟上每2秒（即2秒的批处理间隔）报告字数的要求，但不是每500毫秒。 因此，需要设置批次间隔，以便可以维持生产中的预期数据速率。

找出适合您的应用程序的正确批处理大小的一种好方法是使用保守的批处理间隔（例如5-10秒）和低数据速率进行测试。 要验证系统是否能够跟上数据速率，可以检查每个已处理批处理经历的端到端延迟的值（可以在Spark驱动程序log4j日志中查找“ Total delay”，也可以使用 StreamingListener接口）。 如果延迟保持与批次大小相当，则系统是稳定的。 否则，如果延迟持续增加，则意味着系统无法跟上，因此不稳定。 一旦有了稳定配置的想法，就可以尝试提高数据速率和/或减小批处理大小。 注意，由于暂时的数据速率增加而引起的延迟的瞬时增加可能是好的，只要该延迟减小回到低值（即，小于批量大小）即可。



## 内存调试（Memory Tuning）

《调整指南》中详细讨论了调整Spark应用程序的内存使用情况和GC行为。 强烈建议您阅读。 在本节中，我们将专门在Spark Streaming应用程序的上下文中讨论一些调整参数。

Spark Streaming应用程序所需的群集内存量在很大程度上取决于所使用的转换类型。 例如，如果要对最后10分钟的数据使用窗口操作，则群集应具有足够的内存以将10分钟的数据保存在内存中。 或者，如果要对大量键使用updateStateByKey，则所需的内存将很大。 相反，如果您想执行一个简单的map-filter-store操作，则所需的内存将很少。

通常，由于通过接收器接收的数据存储在StorageLevel.MEMORY_AND_DISK_SER_2中，因此无法容纳在内存中的数据将溢出到磁盘上。 这可能会降低流应用程序的性能，因此建议根据流应用程序的要求提供足够的内存。 最好尝试以小规模查看内存使用情况并据此进行估计。

内存调整的另一个方面是垃圾回收。 对于需要低延迟的流应用程序，不希望由于JVM垃圾收集而造成较大的停顿。

有一些参数可以调整内存使用和GC开销：

- DStream的持久性级别：如前面的“数据序列化”部分所述，默认情况下，输入数据和RDD被持久化为序列化字节。 与反序列化的持久性相比，这减少了内存使用和GC开销。 启用Kryo序列化可进一步减少序列化的大小和内存使用量。 通过压缩可以进一步减少内存使用（请参见Spark配置spark.rdd.compress），而这会占用CPU时间。
- 清除旧数据：默认情况下，将自动清除DStream转换生成的所有输入数据和持久性RDD。 Spark Streaming根据使用的转换来决定何时清除数据。 例如，如果您使用10分钟的窗口操作，那么Spark Streaming将保留最后10分钟的数据，并主动丢弃较旧的数据。 通过设置streamingContext.remember，可以将数据保留更长的时间（例如，以交互方式查询较旧的数据）。
- CMS垃圾收集器：强烈建议使用并发标记和清除GC，以使与GC相关的暂停时间始终保持较低。 尽管已知并发GC会降低系统的整体处理吞吐量，但仍建议使用并发GC以实现更一致的批处理时间。 确保在驱动程序（使用spark-submit中使用--driver-java-options）和执行程序（使用Spark配置spark.executor.extraJavaOptions）上都设置了CMS GC。
- 其他提示：为了进一步减少GC开销，请尝试以下更多提示。
	- 使用OFF_HEAP存储级别持久保留RDD。 请参阅《 Spark编程指南》中的更多详细信息。
	- 使用更多具有较小堆大小的执行程序。 这将减少每个JVM堆中的GC压力。


重要要点：

- DStream与单个接收器关联。 为了获得读取并行性，需要创建多个接收器，即多个DStream。 接收器在执行器中运行。 它占据了一个核心。 预订接收器插槽后，请确保有足够的内核可用于处理，即spark.cores.max应考虑接收器插槽。 接收者以循环方式分配给执行者。

- 当从流源接收数据时，接收器会创建数据块。 每blockInterval毫秒生成一个新的数据块。 在batchInterval期间创建了N个数据块，其中N = batchInterval / blockInterval。 这些块由当前执行器的BlockManager分发给其他执行器的块管理器。 之后，驱动程序上运行的网络输入跟踪器将被告知有关块的位置，以进行进一步处理。

- 在驱动程序上为在batchInterval期间创建的块创建了RDD。 在batchInterval期间生成的块是RDD的分区。 每个分区都是一个任务。 blockInterval == batchinterval意味着将创建一个分区，并且可能在本地对其进行处理。

- 块上的映射任务在执行器中执行（一个执行器接收该块，另一个执行器复制该块），该执行器具有与块间隔无关的块，除非执行非本地调度。除非间隔时间间隔越大，则块越大。 较高的spark.locality.wait值会增加在本地节点上处理块的机会。 需要在这两个参数之间找到平衡，以确保较大的块在本地处理。

- 可以通过调用inputDstream.repartition（n）来定义分区数，而不必依赖batchInterval和blockInterval。 这会随机重新随机排列RDD中的数据以创建n个分区。 是的，以获得更大的并行度。 虽然以洗牌为代价。 RDD的处理由驾驶员的工作计划者安排为工作。 在给定的时间点，只有一项作业处于活动状态。 因此，如果一个作业正在执行，则其他作业将排队。

- 如果您有两个dstream，将形成两个RDD，并且将创建两个作业，这些作业将一个接一个地调度。 为避免这种情况，可以合并两个dstream。 这将确保为dstream的两个RDD形成单个unionRDD。 然后将此unionRDD视为一项工作。 但是，RDD的分区不会受到影响。

- 如果批处理时间超过batchinterval，那么显然接收者的内存将开始填满，并最终引发异常（最有可能是BlockNotFoundException）。 当前无法暂停接收器。 使用SparkConf配置spark.streaming.receiver.maxRate，可以限制接收器的速率。


## 容错语义（Fault-tolerance Semantics）

### 背景（Background）

为了了解Spark Streaming提供的语义，让我们记住Spark RDD的基本容错语义。

- RDD是一个不变的，确定性可重新计算的分布式数据集。 每个RDD都会记住在容错输入数据集上用于创建它的确定性操作的沿袭。

- 如果由于工作节点故障而导致RDD的任何分区丢失，则可以使用操作沿袭从原始容错数据集中重新计算该分区。

- 假设所有RDD转换都是确定性的，则最终转换后的RDD中的数据将始终相同，而不管Spark集群中的故障如何。

Spark在容错文件系统（例如HDFS或S3）中的数据上运行。 因此，从容错数据生成的所有RDD也是容错的。 但是，Spark Streaming并非如此，因为大多数情况下是通过网络接收数据的（使用fileStream时除外）。 为了对所有生成的RDD实现相同的容错属性，将接收到的数据复制到集群中工作节点中的多个Spark执行程序中（默认复制因子为2）。 这导致系统中发生故障时需要恢复的两种数据：

1. 接收和复制的数据-由于该数据的副本存在于其他节点之一上，因此该数据在单个工作节点发生故障时仍可幸免。

2. 已接收但已缓冲数据以进行复制-由于不进行复制，因此恢复此数据的唯一方法是再次从源中获取数据。

此外，我们应该关注两种故障：

1. 工作节点的故障-运行执行程序的任何工作节点都可能发生故障，并且这些节点上的所有内存中数据都将丢失。 如果任何接收器在故障节点上运行，则其缓冲的数据将丢失。

2. 驱动程序节点发生故障-如果运行Spark Streaming应用程序的驱动程序节点发生故障，则显然SparkContext将丢失，并且所有执行程序及其内存中数据也会丢失。


### 定义（Definitions）

流系统的语义通常是根据系统可以处理每个记录多少次来捕获的。 系统在所有可能的操作条件下（尽管有故障等）可以提供三种保证。

1. 最多一次：每条记录将被处理一次或根本不处理。

2. 至少一次：每个记录将被处理一次或多次。 它比最多一次强，因为它确保不会丢失任何数据。 但是可能有重复项。

3. 恰好一次：每个记录将被恰好处理一次-不会丢失任何数据，也不会多次处理任何数据。 这显然是三者中最强有力的保证。


### 基本语义（Basic Semantics）

概括地说，在任何流处理系统中，处理数据都需要三个步骤。

1. 接收数据：使用接收器或其他方式从源接收数据。

2. 转换数据：使用DStream和RDD转换对接收到的数据进行转换。

3. 推送数据：将最终转换后的数据推送到外部系统，例如文件系统，数据库，仪表板等。

如果流应用程序必须实现端到端的精确一次保证，则每个步骤都必须提供精确一次保证。也就是说，每条记录必须被接收一次，被转换一次，并被推送到下游系统一次。让我们了解Spark Streaming上下文中这些步骤的语义。

1. 接收数据：不同的输入源提供不同的保证。下一部分将对此进行详细讨论。

2. 转换数据：由于RDD提供的保证，所有接收到的数据将只处理一次。即使出现故障，只要可以访问接收到的输入数据，最终转换后的RDD将始终具有相同的内容。

3. 推送数据：默认情况下，输出操作确保至少一次语义，因为它取决于输出操作的类型（是否为幂等）和下游系统的语义（是否支持事务）。但是用户可以实现自己的事务处理机制来实现一次语义。本节稍后将对此进行详细讨论。


### 接收数据的语义

不同的输入源提供不同的保证，范围从至少一次到恰好一次。

#### 基于文件
如果所有输入数据已经存在于诸如HDFS之类的容错文件系统中，则Spark Streaming始终可以从任何故障中恢复并处理所有数据。 这提供了一次精确的语义，这意味着无论发生什么故障，所有数据都会被精确处理一次。


#### 基于接收器源

对于基于接收方的输入源，容错语义取决于故障情况和接收方的类型。如前所述，接收器有两种类型：

1. 可靠的接收器-这些接收器仅在确保已复制接收到的数据后才确认可靠的来源。如果此类接收器发生故障，则源将不会收到对缓冲（未复制）数据的确认。因此，如果重新启动接收器，则源将重新发送数据，并且不会由于失败而丢失任何数据。

2. 不可靠的接收器-这种接收器不发送确认，因此当由于工作程序或驱动程序故障而失败时，可能会丢失数据。

根据所使用的接收器类型，我们可以实现以下语义。如果工作节点发生故障，那么可靠的接收器不会造成数据丢失。如果接收器不可靠，则接收到但未复制的数据可能会丢失。如果驱动程序节点发生故障，则除了这些丢失之外，所有已接收并复制到内存中的过去数据都将丢失。这将影响有状态转换的结果。

为了避免丢失过去收到的数据，Spark 1.2引入了预写日志，该日志将收到的数据保存到容错存储中。通过启用预写日志和可靠的接收器，数据丢失为零。就语义而言，它至少提供了一次保证。


#### 基于Kafka API

在Spark 1.3中，我们引入了新的Kafka Direct API，它可以确保Spark Streaming一次接收所有Kafka数据。 同时，如果实施一次精确的输出操作，则可以实现端到端的一次精确保证。 《 Kafka集成指南》中进一步讨论了这种方法。


### 输出操作的语义

输出操作（如foreachRDD）至少具有一次语义，也就是说，在工作程序失败的情况下，转换后的数据可能多次写入外部实体。 尽管使用saveAs *** Files操作将其保存到文件系统是可以接受的（因为文件只会被相同的数据覆盖），但可能需要付出额外的努力才能实现一次语义。 有两种方法：

- 等幂更新：多次尝试总是写入相同的数据。 例如，saveAs *** Files始终将相同的数据写入生成的文件。

- 事务性更新：所有更新都是事务性完成的，因此原子更新仅进行一次。 一种做到这一点的方法如下：

	- 使用批处理时间（在foreachRDD中可用）和RDD的分区索引来创建标识符。 该标识符唯一地标识流应用程序中的Blob数据。

	- 使用标识符以事务方式（即，原子地一次）更新与此Blob的外部系统。 也就是说，如果尚未提交标识符，则自动提交分区数据和标识符。 否则，如果已经提交，则跳过更新。

示例如下：

	dstream.foreachRDD { (rdd, time) =>
	  rdd.foreachPartition { partitionIterator =>
	    val partitionId = TaskContext.get.partitionId()
	    val uniqueId = generateUniqueId(time.milliseconds, partitionId)
	    // use this uniqueId to transactionally commit the data in partitionIterator
	  }
	}



