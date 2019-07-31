## 外部服务（Services）

0.7版本后，JupyterHub支持外部的服务。

### 服务定义

外部服务是与Hub的REST API交互的一个进程。通常表现为一个具体的行为或任务。比如关闭空闲的notebook服务，注册附加web服务等。

服务有两个关键特性：

- 服务是否被Jupyterhub管理

- 服务是否有web服务且需要加入代理维护表中的

因此，区分了两种类型的服务：

- 由Hub管理的服务（Hub-Managed Service)

- 有自己的web服务并通过Hub的API进行通信操作（Externally-Managed Service)

### 服务属性

服务可以具有如下属性：

- name:str，服务名称；

- admin:bool，默认为false，服务是否具有管理员权限

- url:str，默认为None，服务的地址，当url是外部服务的地址时，需要将其加入代理中（/services/:name)。

- api_token:str，默认None，外部服务需要声明API令牌来执行API请求

如果被Hub管理，还可以具有如下属性：

- command:(str/Popen list)，jhub生成服务的指令，仅在服务为子进程时使用。若为声明指令，服务会做外部管理；若声明了指令，服务会被启动并被hub管理。

- environment:dict，服务的附加环境变量

- user:str，管理服务的系统用户名，若未声明，以Hub的相同用户运行。


### Hub维护的服务
被Hub管理的服务首先通过Hub启动，且Hub对其行为负责，该服务只能是Hub的本地子进程。

尽管该类型服务于生成器很类似，但该类服务不能支持与生成器相同的行为。

若想在Docker或其他部署环境中允许service，可以将其注册为外部服务。

#### 启动Hub管理的服务

该类服务是通过特定的命令来启动的。比如配置范例：

	c.JupyterHub.services = [
	    {
	        'name': 'cull-idle',
	        'admin': True,
	        'command': [sys.executable, '/path/to/cull-idle.py', '--timeout']
	    }
	]

其包含了服务名称，管理员权限，以及执行命令。

服务可能配置额外参数来描述服务所需的环境要求：

- environment:dict，额外环境变量

- user:str，不同于Hub用户的执行用户，需要Hub具有root权限

- cwd:path，不同于Hub目录的执行目录

启动服务所传递的环境变量：

	JUPYTERHUB_SERVICE_NAME:   The name of the service
	JUPYTERHUB_API_TOKEN:      API token assigned to the service
	JUPYTERHUB_API_URL:        URL for the JupyterHub API (default, http://127.0.0.1:8080/hub/api)
	JUPYTERHUB_BASE_URL:       Base URL of the Hub (https://mydomain[:port]/)
	JUPYTERHUB_SERVICE_PREFIX: URL path prefix of this service (/services/:service-name/)
	JUPYTERHUB_SERVICE_URL:    Local URL where the service is expected to be listening.
	                           Only for proxied web services.


### 外部服务

外部服务不属于Hub的子进程，因此必须要有API令牌来告诉Jhub是哪个外部服务来访问。

一个通常的外部服务配置如下：

	c.JupyterHub.services = [
	    {
	        'name': 'my-web-service',
	        'url': 'https://10.0.1.1:1984',
	        'api_token': 'super-secret',
	    }
	]

该配置中的url会作为JUPYTERHUB_SERVICE_URL传递。

### 实现自己的服务
对于自定义服务，有几个决策点需要关注：

- 是否需要公网url

- 是否需要Jhub负责启停

- 是否需要验证用户

当服务本身具有url时，服务能通过/services/前缀进行访问，比如https://myhub.com/services/my-service/。若要正确的寻址代理请求，必须考虑JUPYTERHUB_SERVICE_PREFIX。

### 服务与Hub验证（待补充）

Jhub在0.7后添加了授权其他服务使用的Hub验证机制。在登录hub后，Hub会保存cookie，服务可以使用该cookie进行验证请求。关于hub的验证客户端和服务可以自主实现的。

JHub附带有通过服务使用的Hub身份验证的参考实现。可以基于此并创建自定义身份验证客户端和服务。

基本参考是实现了HubAuth类，该类实现了指向Hub的请求。若要使用HubAuth，需要设置.api_token，该参数通过构造对象赋值，或者通过JUPYTERHUB_API_TOKEN环境变量赋值。

大多数的身份验证实现逻辑可以在HubAuth.user_for_cookie以及HubAuth.user_for_token方法中找到，返回参数为：

	{
	  "name": "username",
	  "groups": ["list", "of", "groups"],
	  "admin": False, # or True
	}  # 若没有用户通过验证则返回None

HubAuth同时会缓存Hub的响应时长，通过cookie_cache_max_age配置（默认5分钟）。

#### Flask实例

假设我们有个Flask服务返回用户信息，JHub的HubAuth类能提供给Flask服务进行请求验证。

	from functools import wraps
	import json
	import os
	from urllib.parse import quote
	
	from flask import Flask, redirect, request, Response
	
	from jupyterhub.services.auth import HubAuth
	
	#获取服务路由前缀
	prefix = os.environ.get('JUPYTERHUB_SERVICE_PREFIX', '/')
	
	#设置验证令牌及响应缓存时间60s
	auth = HubAuth(
	    api_token=os.environ['JUPYTERHUB_API_TOKEN'],
	    cookie_cache_max_age=60,
	)
	
	#初始化构造函数，创建Flask对象
	app = Flask(__name__)
	
	#身份验证的实现方法，内部调用的是auth的user_for_cookie和user_for_token。
	def authenticated(f):
	    """Decorator for authenticating with the Hub"""
	    @wraps(f)
	    def decorated(*args, **kwargs):
	        cookie = request.cookies.get(auth.cookie_name)
	        token = request.headers.get(auth.auth_header_name)
	        if cookie:
	            user = auth.user_for_cookie(cookie)
	        elif token:
	            user = auth.user_for_token(token)
	        else:
	            user = None
	        if user:
	            return f(user, *args, **kwargs)
	        else:
	            # redirect to login url on failed auth
	            return redirect(auth.login_url + '?next=%s' % quote(request.path))
	    return decorated
	
	
	@app.route(prefix)
	@authenticated
	def whoami(user):
	    return Response(
	        json.dumps(user, indent=1, sort_keys=True),
	        mimetype='application/json',
	        )


#### JupyterHub中使用tornado验证服务

大部分的Jupyter服务是基于tornado编写的，我们加入了一个混合类，HubAuthenticated，在JupyterHub中快速验证自定义tornado服务。

tornado的@web.authenticated调用了处理器的.get_current_user方法来验证用户。混合了后该方法可以使用HubAuth类。

	class MyHandler(HubAuthenticated, web.RequestHandler):
	    hub_users = {'inara', 'mal'}
	
	    def initialize(self, hub_auth):
	        self.hub_auth = hub_auth
	
	    @web.authenticated
	    def get(self):
	        ...

HubAuth会自动从服务环境变量中加载期望的配置。

若要限制用户接入，可以通过.hub_users或.hub_groups使用白名单，用于检查用户名和用户组列表。不在名单中不可接入，若二者皆未定义，则任意用户可接入。

#### JupyterHub中实现自己的身份验证

若不想使用参考实现，可通过Hub实现自己的身份验证。我们推荐阅读HubAuth类的实现作为参考：

1. 从请求中接收jupyterhub-services cookie。
2. 发送一个API请求(GET /hub/api/authorizations/cookie/jupyterhub-services/cookie-value)，cookie-value是经过编码的jupyterhub-services cookie，该请求必须通过Hub API令牌验证。

		r = requests.get(
			    '/'.join((["http://127.0.0.1:8081/hub/api",
			               "authorizations/cookie/jupyterhub-services",
			               quote(encrypted_cookie, safe=''),
			    ]),
			    headers = {
			        'Authorization' : 'token %s' % api_token,
			    },
			)
			r.raise_for_status()
			user = r.json()	
3. 成功后，返回的是JSON用户模型：
		
		{
		  "name": "inara",
		  "groups": ["serenity", "guild"],
		
		}