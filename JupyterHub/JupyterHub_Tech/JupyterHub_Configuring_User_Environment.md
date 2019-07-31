## 配置用户环境

部署Jhub意味着你要为多用户提供Jupyter notebook环境。那么就需要某些方法来配置用户环境。

jupyterhub-singleuser是扩展的标准jupyter notebook，大部分的配置和文档都适用于singleuser环境。用户环境的配置通常不通过Jupyterhub，而是通过Jupyter的系统范围配置发生。


### 安装包

为了让外部包对用户可用，通常要在系统范围或共享环境中安装包。

包的安装路径应该与singleuser的安装路径处于同一环境，且用户必须有读/执行权限。若要用户可以安装额外的包，需要有写的权限。


### 配置Jupyter和IPython

二者皆有自己的配置系统。

作为JHub的管理用，你通常希望为所有用户安装和配置环境。Jupyter和IPython支持系统范围的配置(逻辑路径下的全局配置）。

典型的配置文件路径有：

- 系统范围的/etc/{jupyter|ipython}

- 环境范围的有{sys.prefix}/etc/{jupyter|ipython}

例如，系统范围内激活扩展应用，在/etc/ipython/ipython_config.py

	c.InteractiveShellApp.extensions.append("cython")

例如，为所有用户激活Jupyter notebook的配置，在/etc/jupyter/jupyter_notebook_config.py

	# shutdown the server after no activity for an hour
	c.NotebookApp.shutdown_no_activity_timeout = 60 * 60
	# shutdown kernels after no activity for 20 minutes
	c.MappingKernelManager.cull_idle_timeout = 20 * 60
	# check for idle kernels every two minutes
	c.MappingKernelManager.cull_interval = 2 * 60

### 安装kernelspecs

你可能有多个Jupyter kernel，而且能让所有用户使用。那么就意味着kernelspecs安装在系统范围或JHub的sys.prefix。

默认kernelspecs是安装在系统范围的，但某些kernel可能默认装在home路径下。这时需要移动到系统范围目录下。

例如，安装系统范围的核心

	/path/to/python3 -m IPython kernel install --prefix=/usr/local
	/path/to/python2 -m IPython kernel install --prefix=/usr/local

### 多用户主机与容器

有两种类型的环境取决于你选择的生成器：

- 多用户主机（共享系统）

- 容器

第一类是共享系统，用户有JHub的账户和home路径，这时共享配置和安装必须在系统范围路径下，比如/etc/或/usr/local或自定义路径如/opt/conda。

第二类是容器，系统范围的环境实际上是容器镜像。

无论哪种场景，你都要避免将配置放入用户的home路径下，因为用户可以修改这些配置。同时，home路径一旦创建就一直存在，管理员很难进行更新。


### 命名服务

默认情况下，每个用户有一个服务。

但JHub可以允许每个用户有多个服务，这能允许用户配置决定哪个服务将要启动，因此用户可以有多个配置同时允许，而不需要停止并重启其中一个。

可以配置用户允许的最大服务数量：

	c.JupyterHub.named_server_limit_per_user = 5