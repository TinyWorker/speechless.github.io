## 使用JHub的REST API

使用Jhub的REST API，你能够使用如下特性：

- 检查用户是否活跃
- 添加或删除用户
- 停止或启动单用户notebook服务
- 验证服务

REST API提供了用户从hub发送和接收信息的的标准方法。


### 创建API令牌

在使用Jupyterhub API发送请求时，需要在请求中传递API令牌。

令牌的生成方式通常是：

	openssl rand -hex 32

openssl的命令生成备用令牌，可以添加到JHub的配置项.api_tokens中。

为用户生成令牌使用指令如下，会产生一个随机字符串并将其注册到Hub的数据库中。

	jupyterhub token <username>

在0.8版本后，令牌请求页面对用户开放，可以不通过指令，直接在页面上请求。

### 在配置文件添加API令牌

可以直接在Jhub配置文件中添加令牌与用户名的字典，格式如下：

	c.JupyterHub.api_tokens = {
	    'secret-token': 'username',
	}


### API请求

要使用API需要添加令牌在请求的验证头（Authorization header)中。示例代码如下：

	import requests
	
	api_url = 'http://127.0.0.1:8081/hub/api'
	
	r = requests.get(api_url + '/users',
	    headers={
	             'Authorization': 'token %s' % token,
	            }
	    )
	
	r.raise_for_status()
	users = r.json()

要成功访问API，满足以下任意条件即可：

1. 令牌是属于notebook的所有者
2. 令牌属于管理员用户/服务，且admin_access设置为true


### 通过API生成多个命名服务

0.8版本后，每个用户可以启动多个命名服务。在之前，用户仅能启动单个服务，命令如下：

	curl -X POST -H "Authorization: token <token>" "http://127.0.0.1:8081/hub/api/users/<user>/server"

要使用多命名服务启动，首先修改配置已激活：

	c.JupyterHub.allow_named_servers = True

设置完成后，启动服务命令如下：

	curl -X POST -H "Authorization: token <token>" "http://127.0.0.1:8081/hub/api/users/<user>/servers/<serverA>"
	curl -X POST -H "Authorization: token <token>" "http://127.0.0.1:8081/hub/api/users/<user>/servers/<serverB>"

若要停止服务，将上述命令中的POST替换为DELETE即可。

注意：命名服务若要通过API工作，生成器就需要有为每个用户处理多服务的能力并保证名称的唯一性。