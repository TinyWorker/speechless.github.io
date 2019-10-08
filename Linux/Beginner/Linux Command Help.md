## Linux的指令帮助

###概述

使用linux时，面临命令的拼写和参数时，可以利用linux内置的帮助文档来查看。

只记得部分命令关键字，使用man -k搜索。
需要知道某个指令的简要说明，使用whatis，更详细的介绍，使用info。
查看命令在哪个位置，使用which。
查看命令的具体参数和使用方法，使用man。


###命令使用

####查看命令的简要说明
简要说明：

	$whatis command

正则匹配：

	$whatis -w 'loca*'

详细说明：
	
	$info command

#### 使用man

查询命令：

	$man command

通过page up和page down来上下翻页。

在man的帮助手册中，将帮助文档分为了9个类别，对于有的关键字可能存在多个类别中， 我们就需要指定特定的类别来查看；（一般我们查询bash命令，归类在1类中）；

man页面所属的分类标识(常用的是分类1和分类3)

- (1)、用户可以操作的命令或者是可执行文件
- (2)、系统核心可调用的函数与工具等
- (3)、一些常用的函数与数据库
- (4)、设备文件的说明
- (5)、设置文件或者某些文件的格式
- (6)、游戏
- (7)、惯例与协议等。例如Linux标准文件系统、网络协议、ASCⅡ，码等说明内容
- (8)、系统管理员可用的管理条令
- (9)、与内核有关的文件

前面说到使用whatis会显示命令所在的具体的文档类别，我们学习如何使用它

	eg:
	$whatis printf
	printf               (1)  - format and print data
	printf               (1p)  - write formatted output
	printf               (3)  - formatted output conversion
	printf               (3p)  - print formatted output
	printf [builtins]    (1)  - bash built-in commands, see bash(1)

我们看到printf在分类1和分类3中都有；分类1中的页面是命令操作及可执行文件的帮助；而3是常用函数库说明；如果我们想看的是C语言中printf的用法，可以指定查看分类3的帮助：

	$man 3 printf
	$man -k keyword

查询关键字 根据命令中部分关键字来查询命令，适用于只记住部分命令的场合；

eg：查找GNOME的config配置工具命令:

	$man -k GNOME config| grep 1

对于某个单词搜索，可直接使用/word来使用: /-a; 多关注下SEE ALSO 可看到更多精彩内容


####查看路径

查看程序的binary文件所在路径:

	$which command

eg:查找make程序安装路径:

	$which make
	/opt/app/openav/soft/bin/make install

查看程序的搜索路径:

	$whereis command

当系统中安装了同一软件的多个版本时，不确定使用的是哪个版本时，这个命令就能派上用场；

### 总结

whatis info man which whereis