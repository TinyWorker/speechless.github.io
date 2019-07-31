## 生成器（Spawners）

生成器会启动每个单用户notebook，其代表着一个连接进程的抽象接口，一个自定义的生成器需要实现三个动作：

- 启动进程
- 轮询进程是否依然运行
- 停止进程

### 生成器的控制方法

Spawner.start，为单个用户启动服务。通过self.user获得用户信息，会返回用户名，验证器和服务信息。

start方法的返回值为(ip,port)。

Note：在写协同程序时，不要在数据库改变和提交中使用yield

一个常见范例：

	def start(self):
	    self.ip = '127.0.0.1'
	    self.port = random_port()
	    # get environment variables,
	    # several of which are required for configuring the single-user server
	    env = self.get_env()
	    cmd = []
	    # get jupyterhub command to run,
	    # typically ['jupyterhub-singleuser']
	    cmd.extend(self.cmd)
	    cmd.extend(self.get_args())
	
	    yield self._actually_start_server_somehow(cmd, env)
	    return (self.ip, self.port)


Spawner.poll，用于检测生成器是否运行。依然运行则返回None，若退出则返回整型；对于本地进程，该方法使用os.kill(PID，0)来检测。

Spawner.stop，停止进程。必须是一个tornado协同程序，当进程停止时返回。

### 生成器状态

JupyterHub的启动和重启应该不影响单用户notebook服务，所以生成器需要持久化一些信息。使用可以转化为JSON的字典来存储信息。

与start，stop，poll不同，状态方法一定不能是协同的。同时，在单进程下，状态只会包含服务的进程ID：

	def get_state(self):
	    """get the current state"""
	    state = super().get_state()
	    if self.pid:
	        state['pid'] = self.pid
	    return state
	
	def load_state(self, state):
	    """load state from the database"""
	    super().load_state(state)
	    if 'pid' in state:
	        self.pid = state['pid']
	
	def clear_state(self):
	    """clear any state (called after shutdown)"""
	    super().clear_state()
	    self.pid = 0


### 生成器选项表单


### 自定义生成器


### 生成器，资源限制与保障
部分单用户notebook允许设置CPU或内存的限制或保证。为了让限制与保证可以使用，生成器必须支持其实现，一个支持的生成器为systemdspawner。

#### 内存限制和保证
c.Spawner.mem_limit，指定可用的最大内存，但不保证能达到最大的内存。可以使用环境变量MEM_LIMIT，指定的是绝对字节数。

c.Spawner.mem_guarantee，保证最小内存，这个内存是一定可用的。可用环境变量MEM_GUARANTEE配置。

#### CPU限制于保证
c.Spawner.cpu_limit，配置单用户服务可以使用的CPU核心数量。但该配置不意味着你可以用到限制的上限数量，因为有可能会被其他高优先级的应用抢占CPU。

c.Spawner.cpu_guarantee，配置对CPU的使用率。

#### 加密
在proxy，hub和notebook之间的通信是可以加密的，修改配置internal\_ssl即可激活。对于生成器使用这些证书，有两个相关方法：.create\_certs和.move\_certs。

create\_certs将使用内部可信的权威注册一个密钥证书对。在这个进程中，将ip和dns信息加入证书，用于证书验证。没有匹配的验证，notebook无法与hub通信（ssl激活时）。正常来说，这个方法不需要改变，默认的IP/DNS信息就足够了。

方法运行时，会根据配置生成文件。当生成器需要在其他主机上访问证书时，使用move\_certs方法来适当的移动证书。通常会移动到生成器所在的容器。

	c.JupyterHub.internal_certs_location

