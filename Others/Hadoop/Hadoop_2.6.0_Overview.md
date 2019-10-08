## Apache Hadoop 2.6.0

2.6.0是一个小幅更新版本，基于2.4.1稳定发行版。

主要特性和改进如下。

通用组件：

- 使用HTTP代理服务的验证提升，在通过代理服务访问WebHDFS时很有用。
- 新的Hadoop指标池允许直接写入Graphite。
- 与Hadoop兼容文件系统（HCFS）有关的工作。

HDFS:

- 支持POSIX类型文件系统的扩展属性。更多细节查看用户文档。
- 使用OfflineImageViewer，客户端可以通过WebHDFS的api浏览fsimage。
- NFS网关获得了许多可支持性改进和错误修复。 不再需要Hadoop portmapper运行网关，并且网关现在可以拒绝来自非特权端口的连接。
- SecondaryNameNode，JournalNode和DataNode Web UI已使用HTML5和Javascript进行了现代化。

YARN：

- YARN REST API现在支持写入和修改擦欧洲哦。用户可以通过API提交和杀死应用。
- 时间线存储在YARN中，为应用存储通用信息和应用专属信息，支持通过Kerberos验证。
- Fair Scheduler支持动态的分层用户队列，用户队列是运行时在任意指定的父队列下动态创建。