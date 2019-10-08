## Submitting Applications

使用spark-submit脚本，用于在集群中加载应用。可以通过统一格式的接口使用所有Spark支持的集群管理器，这样不必为每一个管理器单独配置。

### 绑定应用依赖
若代码依赖于其他项目，需要将其一起打包与应用放在一起，这样才能将代码分布到Spark集群中。实现其目的，创建一个包含代码和依赖的assembly jar。SBT和Maven都有assembly插件。当创建了assembly jar后，Spark和Hadoop的依赖设置作用域为provided，因为集群在运行时会提供，所以他们无需绑定。完成后可以通过spark-submit脚本提交，assembly jar也会传递。

### 通过spark-submit启动应用

脚本会根据Spark和其依赖设置类路径，支持不同的集群管理器和部署模式，通用范例如下：

	./bin/spark-submit \
	  --class <main-class> \
	  --master <master-url> \
	  --deploy-mode <deploy-mode> \
	  --conf <key>=<value> \
	  ... # other options
	  <application-jar> \
	  [application-arguments]

其中的常用选项含义：

- --class：应用的主程序入口
- --master：集群的master URL
- --deploy-mode：将driver部署到工作节点，或作为外部客户端本地运行。
- --conf：任意Spark配置属性，会被转化为“key=value”
- application-jar：绑定jar包路径，包含应用和所有依赖。URL必须是集群中全局可见的，比如hdfs://或file://，所有节点可访问。
- application-arguments:传递到应用主方法的参数，可选

一个常用的部署策略是从网关机器（物理上与工作机连接）提交应用，在这个场景下，client模式更合适，在client模式下，driver直接通过spark-submit进程启动，对集群表现为client。输入和输出都在控制台。因此，该模式十分适合涉及到交互指令的应用（Spark Shell）。

同样的，若你的应用从远程提交，通常使用cluster模式来最小化driver与executor间的网络延迟。当前，standalone模式不支持Python应用的cluster模式。

有一些选项可以用于指定集群管理器，例如，Spark Standalone cluster和cluster模式，可以指定--supervise选项来确保driver能在非0失败时自动重启。通过spark-submit --help来查看所有可选项。

### Master URLs

| Master URL | 含义 |
| ----  ---- |
| local | 以一个worker线程在本地运行（无并行） | 
| local[K] | 以K个worker线程在本地运行（K通常为物理机核心数量） |
| local[K,F] | 以K个worker线程在本地运行，F为最大失败数 |
| local[*] | 以机器中逻辑核心数量的worker线程运行 |
| local[*,F] | 同上，添加最大失败数 |
| spark://HOST:PORT | Spark Standalone cluster的访问路径，端口必须是master配置的，默认7077 |
| spark://HOST1:PORT1,HOST2:PORT2 | 基于Zookeeper的Spark Standalone cluster，列表必须有zookeeper上配置的所有的master地址，端口必须是master配置，默认7077|
| mesos://HOST:PORT | mesos集群，默认端口5050，或者使用Zookeeper，使用mesos://zk://，以cluster模式提交，HOST:PORT应该连接到MesosClusterDispatcher |
| yarn | YARN集群，以client或cluster模式。集群地址通过HADOOP_CONF_DIR或YARN_CONF_DIR参数决定 |
| k8s://HOST:PORT | Kubernetes集群，仅cluster模式。HOST和PORT对应Kubernetes API Server，默认TLS连接，强制使用非安全连接，使用k8s://http://HOST:PORT |

### 从文件中加载配置

submit脚本可从properties文件加载Spark默认配置，并传递到应用中。默认从Spark路径中的conf/spark-defaults.conf读取。

加载默认配置能避免使用spark-submit一些标志的需要，比如spark.master配置后，可以忽略--master的submit选项。通常，显式设置的配置具有最高优先级，然后是标志值，最后是配置文件的值。

### 先进的依赖管理

使用spark-submit，应用jar包和在--jars选项中的所有jar包会被自动传输到集群中，--jars后的URL必须通过逗号隔开，该列表会包含在driver和executor的类路径下。目录扩展不适用与--jars。

Spark使用如下格式进行不同方式的传输：

- file: - 绝对路径，file:/的URI通过driver的HTTP文件服务提供，每个executor从driver的HTTP服务中拉取文件。
- hdfs:,http:,https:,ftp: - 从提供的URI拉取文件。
- local: - 该URI意味着文件应该存在每个worker节点的本地文件，不需要产生网络IO，在大文件或jar包下工作很理想，文件/jar包会推送到每个worker节点，或通过NFS/GlusterFS等方式共享。

注意，jar和文件在executor节点上会为每个SparkContext复制到工作路径下。这会造成大量的空间占用需要及时清洗。在YARN中，清洗工作是自动处理的，Spark Standalone模式下，自动清理可通过spark.worker.cleanup.appDataTtl属性激活。

用户还可以通过提供带有-packages的以逗号分隔的Maven坐标列表来包括任何其他依赖项。使用此命令时，将处理所有传递依赖项。可以以逗号分隔的方式添加其他存储库（或SBT中的解析器），并带有--repositories标志。 （请注意，在某些情况下，可以在存储库URI中提供受密码保护的存储库的凭据，例如https：// user：password @ host /。...以这种方式提供凭据时要小心。）这些命令可以是 与pyspark，spark-shell和spark-submit一起使用，以包含Spark软件包。


