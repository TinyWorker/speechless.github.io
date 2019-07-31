## 使用模板与UI

Jupyterhub的页面是通过Jinja模板来生成的。模板可以定义一次并整合到所有页面中，通过模板可以控制Jhub的外观。

### 自定义模板

Jhub首先在配置的路径（JupyterHub.template_paths）中寻找自定义模板，若没有模板找到则使用默认模板（0.9版本）。

### 扩展模板

Jinja具有扩展模板的机制。基础模板定义为一个block，子模板可以替换或补充block中的元素。

例如，子模板扩展page.html，示例：

	{ extends "templates/spawn_pending.html" }
	
	{ block message }
	{{ super() }}
	<p>Patience is a virtue.</p>
	{ endblock }

其中，extends代表扩展的父页面，添加了templates路径前缀则是指明自定义模板位置，block代表父页面的内容，子模板可以在其中替换元素，继承父页面元素使用super()。

声明：每个{}内都有%包裹，去掉是因为Jekyll不支持逻辑运算符的文本。

### 页面声明

在页面上添加声明，有两个选项：使用扩展页面，或者使用配置变量。

若设置了JupyterHub.template_vars = {'announcement':'xx'}，那么xx将会置于所有页面顶部。更详细的变量announcement_login，announcement_spawn，announcement_home，announcement_logout，将会在对应的页面上显示。这些变量的修改生效需要重启。

同时在扩展页面上也可以生效，并且不需要重启。但注意扩展页面优先。

	{ extends "templates/login.html" }
	{ set announcement = 'some message' }