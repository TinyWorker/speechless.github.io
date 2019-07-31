## JupyterHub的URL策略

这里不包含REST API的url。

通用来说，所有的URL都可以用Jhub的前缀来执行。所有的验证处理器都会重定向到/hub/login，在用户重定向到原始页面之前来登录用户。返回的请求应该保留所有参数。

### /
最顶层的请求，总是重定向到/hub/，用默认的Jhub处理器处理。

### /hub/
验证地址，该处理器将用户重定向到应用的默认路径即用户的默认服务。若用户服务未启动定向到/hub/spawn/，若已启动则定向到/user/:name。

该默认行为有两种自定义的方式：

第一种，将其重定向改为Jhub首页（/hub/home）而不是自己的服务，设置为：

	c.JupyterHub.redirect_to_server = False；

如果你需要多用户服务配置管理且不希望自动生成，那这个可能很有用。

第二种，直接指定任意的页面：
	
	c.JupyterHub.default_url = '/services/my-landing-service'


### /hub/home

Hub的首页，可以手动启动或停止用户服务，如果命名服务启用了，还有额外的一些管理工具。

### /hub/login
Jhub的登录页，当你有基于表单的用户名+密码登录时，会被渲染成登录表单样式。若登录由外部服务处理，那该页会有按钮进行外部服务登录。

如果想跳过按钮的交互式登录，见如下配置，在已登录的场景是很实用的，但Jhub的验证还是需要一次握手来完成的。

	c.Authenticator.auto_login = True

### /hub/logout
清空当前浏览器的cookie，但默认不会停止用户服务。若要同时停止用户服务，见如下配置。

	c.JupyterHub.shutdown_on_logout = True

### /user/:username[/:servername]
当用户服务运行时，该地址用来处理请求而非Hub。当用户服务未运行时，将重定向到/hub/user/:username/。

### /hub/user/:username[/:servername]
用户服务未运行时的处理地址，这个地址在Jhub中最为复杂，因为存在多种状态：

- 服务没有激活（用户匹配/用户不匹配）
- 服务已准备就绪
- 服务在等待，没有准备完成。

如果服务在等待生成，那么会重定向到/hub/spawn-pending/:username/:servername来观察当前进度。

如果服务没有激活，那么会链接到/hub/spawn/:username/:servername。继续会启动要请求的服务。在此时会返回HTTP503，因为为未运行的服务发出了请求。

如果服务准备好了，那假设代理还没有注册路径。 会执行一些检查并且在重定向到/user/:username/:servername前添加延迟。


### /user-redirect/...

该路径用于分享将用户重定向到他们默认服务的地址。当用户有同样的文件在各自服务的同样地址上时，你想要一个单独的连接就能访问到任意的用户文件，这个地址将很有用。

	e.g. a link to /user-redirect/notebooks/Index.ipynb will send user hortense to /user/hortense/notebooks/Index.ipynb

不要将该连接发给其他用户，除非你授权其他用户能访问你的服务。


### /hub/spawn[/:username[/:servername]]
/hub/spawn将会为当前用户生成默认服务。当username和servername被声明时，会生成指定服务给指定用户。一旦生成请求发出，浏览器跳转到/hub/spawn-pending/

当Spawner.options_form使用时，将渲染为一个表单，同时POST请求触发生成请求并跳转。

### /hub/spawn-pending[/:username[/:servername]]
该页面用于渲染进度，一旦服务启动完成，就会跳转到启动的服务/user/:username/:servername

如果该页面在服务已启动的情况下访问，会直接跳转到服务。

访问该页面不会引起任何副作用，当服务停止时，会显示生成失败的消息并返回/hub/spawn/...页面。


### /hub/token
在该页面，用户可以管理他们的Jhub API token，他们可以撤销并请求新令牌，来针对Jhub REST API编写脚本。

### /hub/admin
该页面是管理员页面，能做出以下管理行为：

- 添加、删除用户
- 授权管理权限
- 启动、停止用户服务
- 关闭Jhub