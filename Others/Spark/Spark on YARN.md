# Spark on YARN

***

## 在YARN上运行Spark

确保HADOOP_CONF_DIR或YARN_CONF_DIR指向包含Hadoop集群的（客户端）配置文件的目录。 这些配置用于写入HDFS并连接到YARN ResourceManager。 此目录中包含的配置将分发到YARN群集，以便应用程序使用的所有容器都使用相同的配置。 如果配置引用不是由YARN管理的Java系统属性或环境变量，则还应在Spark应用程序的配置（在客户端模式下运行时的驱动程序，执行程序和AM）中进行设置。

有两种部署模式可用于在YARN上启动Spark应用程序。 在群集模式下，Spark驱动程序在由YARN在群集上管理的应用程序主进程中运行，并且客户端可以在启动应用程序后消失。 在客户端模式下，驱动程序在客户端进程中运行，而应用程序主控仅用于从YARN请求资源。

与Spark支持的其他集群管理器不同，在其中，master的地址是在--master参数中指定的，而在YARN模式下，ResourceManager的地址是从Hadoop配置中获取的。 因此，--master参数是yarn。

在cluster模式下启动Spark应用：

	./bin/spark-submit --class path.to.your.Class --master yarn --deploy-mode cluster [options] <app jar> [app options]


上面启动了一个YARN客户端程序，该程序启动了默认的Application Master。 然后，SparkPi将作为Application Master的子线程运行。 客户端将定期轮询Application Master以获取状态更新，并将其显示在控制台中。 应用程序完成运行后，客户端将退出。 请参阅下面的“调试应用程序”部分，以了解如何查看驱动程序和执行程序日志。


在client模式下启动Spark应用：

	./bin/spark-shell --master yarn --deploy-mode client

### 添加其他jar包

在cluster模式下，driver和客户端在不同机器上运行，因此，SparkContext.addJar不能与客户端本地文件一起使用。 要使客户端上的文件可用于SparkContext.addJar，请在启动命令中将它们与--jars选项一起使用

	./bin/spark-submit --class my.main.Class \
	    --master yarn \
	    --deploy-mode cluster \
	    --jars my-other-jar.jar,my-other-other-jar.jar \
	    my-main-jar.jar \
	    app_arg1 app_arg2


## 准备

在YARN上运行Spark需要使用YARN支持构建的Spark二进制分发版。 可以从项目网站的下载页面下载二进制发行版。 要自己构建Spark，请参阅构建Spark。

为了使YARN端可以访问Spark运行时jar，可以指定spark.yarn.archive或spark.yarn.jars。 有关详细信息，请参阅Spark属性。 如果未指定spark.yarn.archive和spark.yarn.jars，Spark将使用$ SPARK_HOME / jars下的所有jar创建一个zip文件，并将其上传到分布式缓存。


## 配置

YARN上的Spark的大多数配置与其他部署模式相同。 有关这些的更多信息，请参见配置页面。 这些是特定于YARN上Spark的配置。

## 验证应用

用YARN术语来说，执行者和应用程序主管在“容器”内部运行。 在应用程序完成之后，YARN有两种处理容器日志的模式。 如果启用了日志聚合（使用yarn.log-aggregation-enable配置），则将容器日志复制到HDFS并在本地计算机上删除。 可以使用yarn logs命令从群集中的任何位置查看这些日志。

	yarn logs -applicationId <app ID>

将从给定应用程序的所有容器中打印出所有日志文件的内容。 您还可以使用HDFS Shell或API在HDFS中直接查看容器日志文件。 可以通过查看您的YARN配置（yarn.nodemanager.remote-app-log-dir和yarn.nodemanager.remote-app-log-dir-suffix）找到它们所在的目录。 日志也可在Spark Web UI的“执行程序”选项卡下使用。 您需要同时运行Spark历史记录服务器和MapReduce历史记录服务器，并在yarn-site.xml中正确配置yarn.log.server.url。 Spark历史记录服务器UI上的日志URL将您重定向到MapReduce历史记录服务器以显示聚合的日志。

如果未启用日志聚合，则日志将在YARN_APP_LOGS_DIR下的每台计算机上本地保留，根据Hadoop版本和安装情况，该日志通常配置为/ tmp / logs或$ HADOOP_HOME / logs / userlogs。 要查看容器的日志，需要转到包含它们的主机并在此目录中查找。 子目录按应用程序ID和容器ID组织日志文件。 日志也可以在Spark Web UI的“执行程序”选项卡下使用，并且不需要运行MapReduce历史记录服务器。

要查看每个容器的启动环境，请将yarn.nodemanager.delete.debug-delay-sec增加到一个较大的值（例如36000），然后通过容器所在节点上的yarn.nodemanager.local-dirs访问应用程序缓存。 推出了。 该目录包含启动脚本，JAR和用于启动每个容器的所有环境变量。 该过程对于调试类路径问题特别有用。 （请注意，启用此功能需要对群集设置具有管理员权限，并需要重新启动所有节点管理器。因此，这不适用于托管群集）。

要对master或executor使用自定义的log4j配置，有如下方式：

- 通过使用spark-submit上传自定义log4j.properties，方法是将其添加到要与应用程序一起上传的文件的--files列表中

- 将-Dlog4j.configuration = <配置文件的位置>添加到spark.driver.extraJavaOptions（对于驱动程序）或spark.executor.extraJavaOptions（对于执行程序）。 请注意，如果使用文件，则应明确提供file：协议，并且文件必须在所有节点上本地存在。

- 更新$ SPARK_CONF_DIR / log4j.properties文件，它将与其他配置一起自动上载。 请注意，如果指定了多个选项，则其他2个选项的优先级高于此选项。

请注意，对于第一种选择，执行器和应用程序主控器将共享相同的log4j配置，这可能会在它们在同一节点上运行时引起问题（例如，尝试写入同一日志文件）。

如果需要引用适当的位置以将日志文件放入YARN中，以便YARN可以正确显示和聚合它们，请在log4j.properties中使用spark.yarn.app.container.log.dir。 例如，log4j.appender.file_appender.File = $ {spark.yarn.app.container.log.dir} /spark.log。 对于流应用程序，配置RollingFileAppender并将文件位置设置为YARN的日志目录将避免因大型日志文件而导致磁盘溢出，并且可以使用YARN的日志实用程序访问日志。

要为应用程序主程序和执行程序使用自定义的metrics.properties，请更新$ SPARK_CONF_DIR / metrics.properties文件。 它会自动以其他配置上传，因此您无需使用--files手动指定它。

## Spark属性

表格待补充；

| 属性名称 | 默认值 | 功能 |
|---|---|---|
|spark.yarn.am.memory | 512m | 客户端模式下用于YARN Application Master的内存量，格式与JVM内存字符串相同（例如512m，2g）。 在群集模式下，请改用spark.driver.memory。使用小写的后缀，例如 k，m，g，t和p分别表示kibi，mebi，gibi，tebi和pebibytes。 |
| spark.yarn.am.cores | 1 | 客户端模式下用于YARN Application Master的内核数。 在群集模式下，请改用spark.driver.cores。|
| spark.yarn.am.waitTime	 | 100s | 群集模式下，YARN Application Master等待SparkContext初始化的时间。 在客户端模式下，YARN Application Master等待驱动程序连接到它的时间。 |
| spark.yarn.submit.file.replication | The default HDFS replication (usually 3)	 | 上传到应用程序HDFS中的文件的HDFS复制级别。 其中包括Spark jar，应用程序jar和任何分布式缓存文件/归档之类的东西。 |
| spark.yarn.stagingDir | Current user's home directory in the filesystem | 提交申请时使用的登台目录 |
| spark.yarn.preserve.staging.files | false | 设置为true可以在作业结束时保留暂存的文件（Spark jar，app jar，分布式缓存文件），而不是删除它们 |
| spark.yarn.scheduler.heartbeat.interval-ms | 3000 | Spark应用程序主设备心跳到YARN ResourceManager中的时间间隔（以毫秒为单位）。 该值的上限为有效期间隔的YARN配置的一半，即yarn.am.liveness-monitor.expiry-interval-ms。 |
| spark.yarn.scheduler.initial-allocation.interval	 | 200ms | 当存在挂起的容器分配请求时，Spark应用程序主服务器急切地向YARN ResourceManager发出心跳的初始时间间隔。 它不应大于spark.yarn.scheduler.heartbeat.interval-ms。 如果仍然存在挂起的容器，则在连续的急切心跳中，分配间隔将加倍，直到达到spark.yarn.scheduler.heartbeat.interval-ms。 |
| spark.yarn.max.executor.failures | numExecutors * 2, with minimum of 3 | 应用程序失败之前执行程序失败的最大次数 |
| spark.yarn.historyServer.address	 | (none) | Spark历史记录服务器的地址，例如 host.com：18080。 该地址不应包含方案（http：//）。 由于历史记录服务器是一项可选服务，因此默认情况下未设置。 当Spark应用程序完成将应用程序从ResourceManager UI链接到Spark历史记录服务器UI时，此地址将提供给YARN ResourceManager。 对于此属性，可以将YARN属性用作变量，并且在运行时将其替换为Spark。 例如，如果Spark历史记录服务器与YARN ResourceManager在同一节点上运行，则可以将其设置为$ {hadoopconf-yarn.resourcemanager.hostname}：18080。 |
| spark.yarn.dist.archives	 | (none) | 以逗号分隔的归档列表，将其提取到每个执行程序的工作目录中。 |
| spark.yarn.dist.files	 | (none) | 以逗号分隔的文件列表，将其放置在每个执行程序的工作目录中。 |
| spark.yarn.dist.jars | (none) | 将以逗号分隔的jar列表放置在每个执行程序的工作目录中 |
| spark.yarn.dist.forceDownloadSchemes | (none) | 在将文件添加到YARN的分布式缓存之前，将文件下载到本地磁盘的方案的列表，以逗号分隔。 用于YARN服务不支持Spark支持的方案（例如http，https和ftp）的情况。 |
| spark.executor.instances | 2 | 静态分配的执行程序数。 启用spark.dynamicAllocation.enabled时，初始执行程序集将至少如此之大。 |
| spark.yarn.am.memoryOverhead | AM memory * 0.10, with minimum of 384	 | 与spark.driver.memoryOverhead相同，但用于客户端模式下的YARN Application Master。 |
| spark.yarn.queue	 | default | 提交应用程序的YARN队列的名称。 |
| spark.yarn.jars	 | (none) | 包含要分发到YARN容器的Spark代码的库列表。 默认情况下，YARN上的Spark将使用本地安装的Spark jar，但是Spark jar也可以位于HDFS上的世界可读位置。 这使YARN可以将其缓存在节点上，因此无需在每次运行应用程序时将其分发。 例如，要指向HDFS上的jar，请将此配置设置为hdfs：/// some / path。 允许使用小球。 |
| spark.yarn.archive	 | (none) | 包含所需Spark jar的归档文件，用于分发到YARN缓存。 如果设置，则此配置将替换spark.yarn.jars，并且存档将在所有应用程序容器中使用。 归档文件应在其根目录中包含jar文件。 与上一个选项一样，存档文件也可以托管在HDFS上，以加快文件分发速度。 |
| spark.yarn.access.hadoopFileSystems	 | (none) | 您的Spark应用程序将要访问的安全Hadoop文件系统的列表，以逗号分隔。 例如，spark.yarn.access.hadoopFileSystems = hdfs：//nn1.com：8032，hdfs：//nn2.com：8032，webhdfs：//nn3.com：50070。 Spark应用程序必须有权访问列出的文件系统，并且必须正确配置Kerberos才能访问它们（在同一领域或在受信任领域）。 Spark为每个文件系统获取安全令牌，以便Spark应用程序可以访问那些远程Hadoop文件系统。 spark.yarn.access.namenodes已过时，请改用它。 |
| spark.yarn.appMasterEnv.[EnvironmentVariableName]	 | (none) | 将EnvironmentVariableName指定的环境变量添加到YARN上启动的Application Master进程中。 用户可以指定多个，并设置多个环境变量。 在集群模式下，它控制Spark驱动程序的环境，而在客户端模式下，它仅控制执行程序启动器的环境。 |
| spark.yarn.containerLauncherMaxThreads | 25 | YARN Application Master中用于启动执行程序容器的最大线程数。 |
| spark.yarn.am.extraJavaOptions	 | (none) | 在客户端模式下传递给YARN Application Master的一串额外的JVM选项。 在集群模式下，请改用spark.driver.extraJavaOptions。 请注意，使用此选项设置最大堆大小（-Xmx）设置是非法的。 可以通过spark.yarn.am.memory设置最大堆大小设置 |
| spark.yarn.am.extraLibraryPath	 | (none) | 设置在客户端模式下启动YARN Application Master时要使用的特殊库路径。|
| spark.yarn.maxAppAttempts	 | yarn.resourcemanager.am.max-attempts in YARN	 | 提交申请的最大尝试次数。 它不应大于YARN配置中的最大最大尝试次数。 |
| spark.yarn.am.attemptFailuresValidityInterval | (none) | 定义AM故障跟踪的有效间隔。 如果AM已经运行了至少已定义的间隔，则AM故障计数将被重置。 如果未配置，则不会启用此功能。 |
| spark.yarn.executor.failuresValidityInterval | (none) | 定义执行程序失败跟踪的有效间隔。 超过有效间隔的执行器故障将被忽略。 |
| spark.yarn.submit.waitAppCompletion | true | 在YARN群集模式下，控制客户端是否等待退出直到应用程序完成。 如果设置为true，则客户端进程将保持活动状态，报告应用程序的状态。 否则，客户端进程将在提交后退出。 |
| spark.yarn.am.nodeLabelExpression | (none) | 将调度限制节点AM组的YARN节点标签表达式。 只有大于或等于2.6的YARN版本才支持节点标签表达式，因此在针对早期版本运行时，将忽略此属性。 |
| spark.yarn.executor.nodeLabelExpression | (none) | 将调度限制节点执行程序集的YARN节点标签表达式。 只有大于或等于2.6的YARN版本才支持节点标签表达式，因此在针对早期版本运行时，将忽略此属性。 |
| spark.yarn.tags	 | (none) | 将以逗号分隔的字符串列表作为YARN ApplicationReports中显示的YARN应用程序标签传递，可用于查询YARN应用程序时进行过滤。 |
| spark.yarn.keytab | (none) | 包含上面指定的主体的密钥表的文件的完整路径。 该密钥表将通过安全分布式缓存复制到运行YARN Application Master的节点上，以定期更新登录凭单和委托令牌。 （也可以与“本地”母版一起使用） |
| spark.yarn.principal | (none) | 在安全HDFS上运行时用于登录KDC的主体。 （也可以与“本地”母版一起使用） |
| spark.yarn.kerberos.relogin.period | 1m | 多久检查一次是否应更新kerberos TGT。 该值应设置为短于TGT更新周期（或如果未启用TGT更新，则为TGT生存期）。 对于大多数部署，默认值应该足够。
 |
| spark.yarn.config.gatewayPath | (none) | 在网关主机（启动Spark应用程序的主机）上有效的路径，但对于集群中其他节点中相同资源的路径可能会有所不同。 与spark.yarn.config.replacementPath结合使用，它用于支持具有异构配置的集群，以便Spark可以正确启动远程进程。
替换路径通常将包含对YARN导出的某些环境变量的引用（因此对Spark容器可见）。例如，如果网关节点在/ disk1 / hadoop上安装了Hadoop库，并且YARN将Hadoop安装的位置作为HADOOP_HOME环境变量导出，则将该值设置为/ disk1 / hadoop并将替换路径设置为$ HADOOP_HOME将 确保用于启动远程进程的路径正确引用了本地YARN配置。 |
| spark.yarn.config.replacementPath | (none) | 见上条说明 |
| spark.security.credentials.${service}.enabled | true | 控制启用安全性后是否获取服务的凭据。 默认情况下，在配置所有受支持服务的凭据时，将检索这些凭据，但是，如果某种行为与正在运行的应用程序发生冲突，则可以禁用该行为。 有关更多详细信息，请参阅[在安全群集中运行]（running-on-yarn.html＃running-in-a-secure-cluster） |
| spark.yarn.rolledLog.includePattern | (none) | Java Regex过滤与定义的包含模式匹配的日志文件，这些日志文件将以滚动方式聚合。 这将与YARN的滚动日志聚合一起使用，以在YARN端启用此功能。nodemanager.log-aggregation.roll-monitoring-interval-seconds应该在yarn-site.xml中配置。 此功能只能与Hadoop 2.6.4+一起使用。 需要将Spark log4j附加程序更改为使用FileAppender或另一个附加程序，该附加程序可以在运行时处理要删除的文件。 根据在log4j配置中配置的文件名（例如spark.log），用户应将正则表达式（spark *）设置为包括所有需要汇总的日志文件。 |
| spark.yarn.rolledLog.excludePattern | (none) | Java Regex可以过滤与定义的排除模式匹配的日志文件，并且这些日志文件不会以滚动方式聚合。 如果日志文件名与包含和排除模式均匹配，则该文件最终将被排除 |



## 关键要点

- 在调度决策中是否遵循核心请求取决于所使用的调度程序及其配置方式。

- 在集群模式下，Spark执行程序和Spark驱动程序使用的本地目录将是为YARN配置的本地目录（Hadoop YARN配置yarn.nodemanager.local-dirs）。 如果用户指定spark.local.dir，它将被忽略。 在客户端模式下，Spark执行程序将使用为YARN配置的本地目录，而Spark驱动程序将使用spark.local.dir中定义的目录。 这是因为Spark驱动程序不在客户端模式下在YARN群集上运行，只有Spark执行程序可以运行。

- --files和--archives选项支持使用＃与Hadoop类似来指定文件名。 例如，您可以指定：--files localtest.txt＃appSees.txt，这会将您本地命名为localtest.txt的文件上传到HDFS，但这将通过名称appSees.txt链接到该文件，并且您的应用程序应使用 在YARN上运行时，将其命名为appSees.txt以进行引用。

- 如果与本地文件一起使用并在群集模式下运行，则--jars选项允许SparkContext.addJar函数起作用。 如果将它与HDFS，HTTP，HTTPS或FTP文件一起使用，则无需使用它


## 在安全集群上运行

如安全性所述，Kerberos在安全的Hadoop群集中用于验证与服务和客户端关联的主体。 这使客户可以请求这些经过身份验证的服务； 授予身份验证的委托人权利的服务。

Hadoop服务发出hadoop令牌以授予对服务和数据的访问权限。 客户端必须首先获取将要访问的服务的令牌，并将其与在YARN群集中启动的应用程序一起传递。

为了使Spark应用程序与任何Hadoop文件系统（例如hdfs，webhdfs等），HBase和Hive进行交互，它必须使用启动该应用程序的用户的Kerberos凭据获取相关令牌，即其身份的主体 将成为已启动的Spark应用程序的版本。

这通常是在启动时完成的：在安全集群中，Spark将自动为集群的默认Hadoop文件系统以及HBase和Hive获取令牌。

如果HBase在类路径中，则HBase令牌将获得，HBase配置声明应用程序是安全的（即，hbase-site.xml将hbase.security.authentication设置为kerberos），而spark.security.credentials.hbase.enabled不 设置为false。

同样，如果Hive在类路径上，则将获得Hive令牌，其配置包括“ hive.metastore.uris”中元数据存储的URI，并且spark.security.credentials.hive.enabled未设置为false。

如果应用程序需要与其他安全的Hadoop文件系统进行交互，则必须在启动时明确请求访问这些集群所需的令牌。 通过在spark.yarn.access.hadoopFileSystems属性中列出它们来完成此操作。

	spark.yarn.access.hadoopFileSystems hdfs://ireland.example.org:8020/,webhdfs://frankfurt.example.org:50070/

Spark支持通过Java服务机制与其他可识别安全的服务集成（请参阅java.util.ServiceLoader）。 为此，Spark应该可以通过在jar的META-INF / services目录中相应文件中列出其名称来实现org.apache.spark.deploy.yarn.security.ServiceCredentialProvider的实现。 可以通过将spark.security.credentials。{service} .enabled设置为false来禁用这些插件，其中{service}是凭据提供程序的名称


## 配置外部随机播放服务

要在YARN群集中的每个NodeManager上启动Spark Shuffle服务，请遵循以下说明：

1. 使用YARN配置文件构建Spark。 如果您使用预打包的发行版，请跳过此步骤。

2. 找到spark- <version> -yarn-shuffle.jar。 如果自己构建Spark，则应位于$ SPARK_HOME / common / network-yarn / target / scala- <version>下，如果使用发行版，则应位于yarn下。

3. 将此jar添加到集群中所有NodeManager的类路径。
在每个节点上的yarn-site.xml中，将spark_shuffle添加到yarn.nodemanager.aux-services中，然后将yarn.nodemanager.aux-services.spark_shuffle.class设置为org.apache.spark.network.yarn.YarnShuffleService。

4. 通过在etc / hadoop / yarn-env.sh中设置YARN_HEAPSIZE（默认为1000）来增加NodeManager的堆大小，以避免在随机播放期间

5. 出现垃圾回收问题。

6. 重新启动集群中的所有NodeManager。

| 属性名称 | 默认值 | 功能 |
|---|---|---|
|spark.yarn.shuffle.stopOnFailure | false | Spark Shuffle Service初始化失败时是否停止NodeManager。 这样可以防止由于未运行Spark Shuffle服务的NodeManager上运行容器而导致应用程序故障。 |

## 通过Apache Oozie启动应用

Apache Oozie可以在工作流程中启动Spark应用程序。 在安全的群集中，启动的应用程序将需要相关的令牌才能访问群集的服务。 如果使用密钥表启动Spark，这是自动的。 但是，如果要在没有密钥表的情况下启动Spark，则必须将设置安全性的职责移交给Oozie。

可以在Oozie网站的特定版本文档的“身份验证”部分中找到有关为安全群集配置Oozie以及获取作业凭据的详细信息。

对于Spark应用程序，必须为Oozie设置Oozie工作流，以请求该应用程序需要的所有令牌，包括：

- YARN资源管理器。

- 本地Hadoop文件系统。

- 任何用作I / O源或目标的远程Hadoop文件系统。

- 蜂巢-如果使用的话。

- HBase-如果使用的话。

- YARN时间轴服务器（如果应用程序与此交互）。

为了避免Spark尝试（然后失败）获取Hive，HBase和远程HDFS令牌，必须将Spark配置设置为禁用服务的令牌收集。

Spark配置必须包括以下几行：

	spark.security.credentials.hive.enabled false
	spark.security.credentials.hbase.enabled false

配置选项spark.yarn.access.hadoopFileSystems必须未设置。


## Kerberos问题

调试Hadoop / Kerberos问题可能很“困难”。 一种有用的技术是通过设置HADOOP_JAAS_DEBUG环境变量来在Hadoop中启用Kerberos操作的额外日志记录。

	export HADOOP_JAAS_DEBUG=true

可以将JDK类配置为通过系统属性sun.security.krb5.debug和sun.security.spnego.debug = true启用其Kerberos和SPNEGO / REST身份验证的额外日志记录

	-Dsun.security.krb5.debug=true -Dsun.security.spnego.debug=true

所有这些选项都可以在Application Master中启用：

	spark.yarn.appMasterEnv.HADOOP_JAAS_DEBUG true
	spark.yarn.am.extraJavaOptions -Dsun.security.krb5.debug=true -Dsun.security.spnego.debug=true

最后，如果org.apache.spark.deploy.yarn.Client的日志级别设置为DEBUG，则该日志将包括所有获得的令牌的列表以及它们的到期详细信息。

## 使用Spark History Server来替代Spark Web UI

禁用应用程序UI时，可以将Spark History Server应用程序页面用作运行应用程序的跟踪URL。 在安全群集上或减少Spark驱动程序的内存使用量可能是理想的。 要通过Spark History Server设置跟踪，请执行以下操作：

- 在应用程序方面，在Spark的配置中设置spark.yarn.historyServer.allowTracking = true。 如果禁用了应用程序的UI，这将告诉Spark使用历史服务器的URL作为跟踪URL。

- 在Spark History Server上，将org.apache.spark.deploy.yarn.YarnProxyRedirectFilter添加到spark.ui.filters配置中的过滤器列表中。


请注意，历史记录服务器信息可能不是与应用程序状态有关的最新信息。
