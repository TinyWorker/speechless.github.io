## 验证器
验证器是用于授权用户使用hub和SUNS的机制。

### 默认的PAM验证器
PAM验证器是基于用户名和密码的一种本地账户登录机制。

### OAuth验证器
OAuto之类的一些登录机制并不映射到用户名密码，而是使用令牌。使用这些机制时可以重载登录处理器。

目前Jhub的OAuth可以直接如下的公共服务：Autho，Bitbucket，CILogon，GitHub，GitLab，Globus，Google，MediaWiki，Okpy，OpenShift。

### Dummy验证器
在进行测试的时候，使用Dummy可能会很有用。该验证允许任意的用户名或密码，除非设置了全局密码。当设置好后，任意的用户名+全局密码才能登陆。

其他的验证器在Jhub wiki上有列表。

### 验证器原理

#### 基础验证器的工作原理
使用的是简单的用户名+密码验证。核心方法为：

	Authenticator.authenticate(handler, data)

该方法会传递Tornado的RequestHandler及POST数据，如果成功则返回用户名，失败则返回None。

验证器具有规范化用户名的功能，可以在传递用户名给Hub之前进行，默认的规范化行为是转换为小写。

	c.Authenticator.username_map  = {
	  'service-name': 'localname'
	}

当使用PAM验证器时，提供了一种规范化方式，会将username转换为username-uid-username的形式，这在允许多个相同用户名的场景下很有用。

验证器也可以定义用户名的合法格式，使用pattern来做校验。

	c.Authenticator.username_pattern = r'w.*'

### 自定义验证器
可以通过自定义验证器来实现其他的验证机制。

但因为验证器会将用户名传递给生成器，所以在自定义时通常会二者一起实现。

### 通过entry_points配置来注册自定义验证器

若要注册自定义验证器，要先创建setup.py脚本，添加如下代码：

	setup(
	  ...
	  entry_points={
	    'jupyterhub.authenticators': [
	        'myservice = mypackage:MyAuthenticator',
	    ],
	  },
	)

然后将该脚本放入包中，并配置选项为myservice：

	c.JupyterHub.authenticator_class = 'myservice'

### 验证器状态

验证器的状态可以持久化，类似一个验证相关的令牌，但要使用该特性，需要满足配置激活且开启加密
	
	#配置的激活
	c.Authenticator.enable_auth_state = True
	#环境变量配置，使用加密
	export JUPYTERHUB_CRYPT_KEY=$(openssl rand -hex 32)

### pre_spawn_start和post_spawn_stop连接点

验证器会使用两个连接点，pre_spawn_start(user,spawner)和post_spawn_stop(user,spawner)，将通过的附加状态信息添加到验证器和生成器上。这些连接点通常用于与验证有关的启动。


### JupyterHub是一个OAuth Provider

在0.8版本开始，JupyterHub作为一个OAuth provider存在。