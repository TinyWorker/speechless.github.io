## JupyterHub子系统概览

###JupyterHub的子系统：Hub，Proxy，Single-user Notebook server

JupyterHub是一组进程，用于提供组内成员的单用户Jupyter Notebook服务（后面用SUJ代替），有三个主要的子系统通过命令启动：

- Hub（集束）：管理账户，验证信息，并使用Spawner协调单用户服务。
- Proxy（代理）：用动态代理来向Hub和SUNS发送HTTP请求。http代理是可配置的，默认是node-http-proxy。
- Single-User Notebook Server（SUNS）：每个用户在登录系统时，会启动一个单独用户专用的Jupyter Notebook服务，是通过Spawner启动的。

![JupyterHub subsystems](https://tinyworker.github.io/images/jhub-parts.png)

### 子系统间的交互

用户对Jhub的访问是通过浏览器输入IP或域名的。对于用户访问操作，有如下基本原则：

- Hub来产生代理（在默认的Jhub配置中）
- 代理默认将所有请求转发到Hub。
- Hub要处理登录，并产生SUNS。
- Hub配置代理将URL前缀的请求转发到SUNS中。

要注意代理是唯一的公共接口监听进程。Hub位于代理/hub之后，SUNS位于代理/user/[username]之后。

不同的验证器控制Jhub的访问。默认的PAM使用用户账户作为校验，使用PAM就需要给组内的每个成员创建一个账户。使用其他的验证器可以允许用任意组织持有的单点登录系统账户登录。

接着，spawner控制着Jhub如何为每个用户启动独立的SUNS。默认是在其系统用户名下启动一个SUNS（同一个机器）。需要在不同的容器间启动服务，通常会使用到Docker。

### Jhub接入用户登录
当用户接入Jhub时，有以下事件发生：

- 登录数据会传递给验证器实例。
- 验证器在确认信息是合法时会返回用户名
- SUNS为已登录用户启动
- 当SUNS启动后，代理会被告知转发请求到/user/[username]/*的SUNS中。
- cookie是保存在/hub/，包含一个加密的token。
- 浏览器会被重定向到/user/[username]，请求会被SUNS处理。

当SUNS使用OAuth识别用户时：

- 有请求时，SUNS检查cookie。
- 若没有cookie，重定向到Hub通过OAuth进行验证。
- 经过Hub验证，浏览器重定向到SUNS
- token被验证并存储在cookie中
- 如果没有用户被确认，浏览器会返回登录页。

### 默认行为

代理会默认监听在端口8000上的所有公共接口。所以通常我们访问Jhub的方式：

- http://localhost:8000。
- 任意指向系统的IP或域名。

启动Jhub会在当前工作目录中写入两个文件：

- **jupyter.sqlite**，是SQLite数据库包含Hub的所有状态。这个文件允许Hub记录当前运行用户和运行位置，包含其他信息能分开重启Jhub的部件。该数据库内不会记录比Hub的用户名更敏感的信息。
- **jupyter\_cookie\_secret**是安全cookie的加密key，这个文件需要持久化，以便于Hub重启时可以避开非法cookie。反过来说，删除这个文件并重启Jhub将会直接否定所有登录cookie。

文件的路径可以通过配置指定。但建议将文件存放在标准的UNIX文件系统路径下，比如/etc/jupyterhub存放配置文件，而/srv/jupyterhub存放安全和运行时文件。

### 自定义Jhub
有两个基本的扩展点：

- 验证器如何验证用户的。
- Spawner如何启动SUNS的。

以上每个都可以由自定义的类管理，Jhub仅提供基本的默认属性。如果要使用自定义的验证或Spawner，要继承Authenticator/Spawner，并覆盖相关的方法。