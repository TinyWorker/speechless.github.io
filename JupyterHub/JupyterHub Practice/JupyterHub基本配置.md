## Jupyterhub的基础配置

在完成了Jupyterhub的安装后，我们可以进行基础的一些配置。

要查看适用于配置的指令选项，使用指令：

	jupyterhub --help-all

### Jupyterhub的配置方式

Jupyterhub默认不会生成配置文件，全部按照默认设置启动，若想要对其中的部分参数进行调整，可以使用指令生成配置文件：

	jupyterhub --generate-config

配置文件中包含了所有的配置项及说明，基本可以按照此指引来配置所需项。

在得到了配置文件后，我们可以指定生成的配置文件路径，用于程序读取：

	jupyterhub -f /path/to/jupyterhub_config.py

此时jupyterhub启动时按照指定的配置文件参数启动。

jupyterhub也可以通过指令来配置，启动时在后续脚本中添加参数项即可生效配置，比如在指定ip上使用https协议：

	jupyterhub --ip 10.0.1.2 --port 443 --ssl-key my_ssl.key --ssl-cert my_cert.cert

脚本设置有一定的不方便，比如在设置类参数时，我们使用--Class.trait的格式，比如：

	jupyterhub --Spawner.notebook_dir='~/assignments'

此外，在配置中可以指定Authenticator和Spawner，以及是否要单独的运行代理。

### 网络配置

上述提到可以用命令来配置ip，或可以设置两个参数如下：

	c.JupyterHub.ip='10.0.1.2'
	c.JupyterHub.port=443

443是默认的HTTPS/SSL端口。

由于Hub服务默认仅监听本地8081，因为Hub需要与代理和所有的生成器连接。当生成本地服务时，localhost设置一个IP地址是很好的方式。

无论代理或生成器是要远程调用还是在容器内隔离，都需要一个可接入的ip：

	c.JupyterHub.hub_ip='0.0.0.0' #可以监听所有接口
	c.JupyterHub.hub_port=54321
	c.JupyterHub.hub_connect_ip='10.0.1.4' #暴露的地址或域名

其中hub_connect_ip是其他服务用来访问hub的地址。

如果Hub暴露的是完整URL（https://proxy.example.org/jupyter/)，需要告诉JupyterHub服务的基本地址，设置：

	c.JupyterHub.base_url='/jupyter/'

### 安全性设置

如果JupyterHub在公网运行，请一定要使用SSL加密服务。

Jupyter的配置中，安全性是一个最重要的方面。在安全配置中，主要有三个主要配置：
1. SSL加密（启用HTTPS)
2. cookie的密钥（用于加密浏览器cookie的钥匙）
3. 代理验证令牌，为hub和其他服务用于代理验证的。

Hub会在存储前哈希化所有的密钥。失去对数据库的读访问应该将部署影响降至最小，若数据库被盗用，撤销现有令牌即可。

##### 启用SSL加密
启用SSL加密，首先要使用证书，要求一个官方可信的SSL证书或自己创建一个。使用配置：
	
	c.JupyterHub.ssl_key='/path/to/my.key'
	c.JupyterHub.ssl_cert='/path/to/my.cert'

若证书中包含了key，就不需要key文件了。

##### 使用letsencrypt

暂不考虑。

##### SSL终端在Hub外部
如果hub在代理后，且SSL终端是由NGINX提供的，那么hub不应该启用SSL。这时仅需注释掉上述配置。

#### cookie密钥

密钥是一个加密key，用于加密验证的cookie，有三种方式来生成和配置cookie密钥：

##### 生成并存储为密钥文件
长度应为32随机字节，用16进制编码，并存储在jupyterhub_cookie_secret文件中，示例指令：

	openssl rand -hex 32 > /srv/jupyterhub/jupyterhub_cookie_secret

在JupyterHub的大部分部署中，都要指定其到一个安全路径。

	c.JupyterHub.cookie_secret_file='/srv/jupyterhub/jupyterhub_cookie_secret'

文件若不存在则会生成新的问题，该文件必须要有被其他组或服务读取的权限。推荐授权为600。

##### 生成并存储为环境变量
如果要避免文件需要，可以存储在Hub进程中，指令如下：

	export JPY_COOKIE_SECRET=$(openssl rand -hex 32)

但该环境变量仅能被Hub所见，若动态设置，每次Hub启动的时候回将所有用户登出。

##### 生成并存储为二进制字符串
在配置文件jupyterhub_config.py中，设置：、
	
	c.JupyterHub.cookie_secret=bytes.fromhex('64 CHAR HEX STRING')

如果密钥值发生改变，那所有的单用户notebook都要重启。

#### 代理验证令牌
Hub将验证交给代理，代理使用Hub与代理共识的密钥进行验证。其令牌应该是一个随机的字符串。

##### 令牌的配置项

使用jupyterhub_config.py配置项：
	
	c.JupyterHub.proxy_auth_token='0bc02bede919e99a26de1e2a7a5aadfaf6228de836ec39a05a6c6942831d8fe5'

##### 令牌作为系统变量

	export CONFIGPROXY_AUTH_TOKEN=$(openssl rand -hex 32)

##### 不设置令牌

若不设置代理令牌，那Hub会生成一个随机key，hub只要重启，代理也要一起重启。默认情况下，代理是Hub的子进程，这点会自动发生。

### 验证与用户

默认使用PAM验证器，是传统的用户名密码验证。

#### 创建白名单
可以限制用户是否允许登录，使用白名单配置：

	c.Authenticator.whitelist = {'one','two','three'}

在Hub启动后，白名单中的用户会被加入Hub数据库里。

#### 配置管理员用户
jhub的管理用户，admin_users，可以操作白名单，同时也可以操作其他用户的服务，比如停止重启。

	c.Authenticator.admin_users = {'one'}

在管理员列表中的用户自动进入白名单。而每个验证器对于判定用户是否是管理员的方式有所不同，比如默认的PAM验证器时使用admin_group配置来决定：

	c.PAMAuthenticator.admin_groups={'one'}

#### 管理员访问其他用户的notebook
默认管理员访问是禁止的，若允许访问，那管理员可以登录其他用户的机器来调试。此权限应该告知用户该功能是否开启。

#### 从Hub添加/删除用户
用户的添加删除可以通过Hub的管理员面板或REST API。添加时会自动加入白名单和数据库，重启hub不需要手动更新白名单，因为会从数据库自动加载。

删除用户可以通过管理员面板或者清空jupyterhub.sqlite并重启。

#### 使用LocalAuthenticator创建系统用户
LocalAuthenticator是一个特别的验证器，它可以管理本地系统的用户。进行如下配置时，可以允许添加系统用户。

	c.LocalAuthenticator.create_system_users = True

但此特性在Jupyterhub直接映射了系统用户时不推荐使用。

#### DummyAuthenticator
DummyAuthenticator是用于测试的傻瓜验证器，允许任意的用户名密码登录，除非在配置了全局密码的情况下，匹配密码的任意用户名均可以登录。用如下设置配置全局密码：

	c.DummyAuthenticator.password = "password"

### 生成器和单用户notebook服务
单用户服务实际是一个jupyter notebook的实例，一个完整的，分割的多进程应用，有多种方式配置该服务。

在JupyterHub层面，能在生成器上设置参数。最简单的是Spawner.notebook_dir，用于设置服务的根目录。根目录是用户能访问的最高级别目录。

	c.Spawner.notebook_dir = '~/notebooks'

同时可以添加额外的命令行参数：

	c.Spawner.args = ['--debug','--profile=PHYS131']

也可以设置用户的默认页面：

	c.Spawner.args = ['--NotebookApp.default_url=/notebooks/Welcome.ipynb']

虽然单用户服务是继承的notebook应用，但依然从jupyter_notebook_config.py文件中加载配置。每个用户都在$HOME/.jupyter/中有这个文件。Jupyter也支持读取系统范围的配置文件，这是会影响所有用户的配置。

### 外部服务

服务是与Hub的REST API交互的进程。可以执行特定或动作或任务。比如关闭用户的服务可以是一个由服务自动化的任务。

#### API令牌基础

##### 创建API令牌
如果要运行外部服务，API令牌是必须提供给服务的。

在0.6.0，偏向于如下指令创建API令牌：

	openssl rand -hex 32

在0.8.0，有令牌请求页面来生成令牌。

##### 通过环境变量传递

通过名称为JUPYTERHUB_API_TOKEN的变量传递。

##### 需要外部访问的服务和任务使用API令牌

通常API令牌是和特定用户关联的，但API令牌可由需要外部访问的活动使用，这些活动可能与特定人员不对应。可以使用配置将服务与令牌关联起来：

	c.JupyterHub.services = [
		{'name' : 'adding-users', 'api_token':'super-secret-token'}	
	]

##### 重启JupyterHub
做完了上述的事情再重启hub，会看到如下消息：
	
	Adding API token for <username>

### 使用API令牌对单用户服务验证
此项功能在0.8中添加，在之前仅支持cookie的验证，0.8后支持使用API令牌进行验证。


#### 一个Hub管理的外部服务样例

首先在jupyterhub_config.py，添加服务:

	c.JupyterHub.services = [
		{
			'name' : 'cull-idle',
			'admin': True,
			'command' : [sys.executable, 'cull_idle_servers.py', '--timeout=3600'],
		}
	]


其中，admin指定服务具有admin权限；command指明服务是以子进程启动，受Hub管理。

这将手动运行cull-idle，cull-idle可以作为独立脚本运行，可以访问Hub，并定期检查空闲服务器并通过Hub的REST API关闭它们。 为了关闭服务器，给予cull-idle的令牌必须具有管理员权限。

生成令牌并放入环境变量JUPYTERHUB_API_TOKEN中，并手动启动：

	export JUPYTERHUB_API_TOKEN='token'
    python3 cull_idle_servers.py [--timeout=900] [--url=http://127.0.0.1:8081/hub/api]