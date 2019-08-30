## JupyterHub的Docker环境

### Docker安装

安装的是Docker Engine社区版，在CentOS 7下。需要激活centos-extras仓库，默认是激活的。

推荐使用overlay2存储驱动。

### 删除旧版本

若安装了旧版本，先删除

	$ sudo yum remove docker \
	                  docker-client \
	                  docker-client-latest \
	                  docker-common \
	                  docker-latest \
	                  docker-latest-logrotate \
	                  docker-logrotate \
	                  docker-engine

在默认目录/var/lib/docker下，包括images，containers，volumes和networks这些内容被保留，Docker Engine现在叫docker-ce。

### 安装社区版

