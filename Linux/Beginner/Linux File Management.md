## 文件及目录管理

文件管理不外乎文件或目录的创建、删除、查询、移动，有mkdir/rm/mv

文件查询是重点，用find来进行查询；find的参数丰富，也非常强大；

查看文件内容是个大的话题，文本的处理有太多的工具供我们使用，在本章中只是点到即止，后面会有专门的一章来介绍文本的处理工具；

有时候，需要给文件创建一个别名，我们需要用到ln，使用这个别名和使用原文件是相同的效果；

### 创建和删除

- 创建：mkdir
- 删除：rm
- 删除非空目录：rm -rf file/directory
- 删除日志：rm *log（== find ./-name "*log" -exec rm {};）
- 移动：mv
- 复制：cp [-r]（如果是目录）

查看当前目录文件个数：

	$find ./ | wc -l

复制目录：

	$cp -r source_dir dest_dir

### 目录切换

- 找到文件/目录位置：cd
- 切换到上一个工作目录： cd -
- 切换到home目录： cd or cd ~
- 显示当前路径: pwd
- 更改当前工作路径为path: $cd path

### 列出目录项

- 显示当前目录下的文件 ls
- 按时间排序，以列表的方式显示目录项 ls -lrt

通常该指令因为使用频率的问题，可以建立快捷命令，在.bashrc中设置别名：

	alias lsl='ls -lrt'
	alias lm='ls -al|more'

这样，使用lsl，就可以显示目录中的文件并按照修改时间排序。

给每项文件前面增加一个id编号(看上去更加整洁):

	>ls | cat -n
	1 a 2 a.out 3 app 4 b 5 bin 6 config

注：.bashrc 在/home/你的用户名/ 文件夹下，以隐藏文件的方式存储；可使用 ls -a 查看；

### 查找目录及文件find/locate

搜寻文件或目录：

	$find ./ -name "core*" | xargs file

查找目标文件夹是否有.o文件：

	$find ./ -name "*.o"

递归当前目录和子目录删除所有.o文件

	$find ./ -name "*.o" -exec rm {} \;

find是实时查找，若需要更快的查询，可以尝试locate，该指令会为文件系统建立索引数据库，若有文件更新，需要定期执行更新命令来更新索引库。

寻找包含有string的路径：

	$locate string

更新索引数据库：
	
	$updatedb

与find不同，locate不是实时查找。所以需要更新数据库来获取最新的文件索引信息。

### 查看文件内容

查看文件：cat vi head tail more

显示时显示行号：

	$cat -n

按页显示列表：

	$ls -al | more

只看前10行：

	$head - 10 **

显示文件第一行：

	$head -1 filename

显示文件倒数五行：

	$tail -5 filename

查看两个文件的差别：

	$diff file1 file2

动态显示文本最新信息：

	$tail -f filename

### 查找文件内容

使用egrep查询文件内容：

	egrep '03.1\/CO\/AE' TSF_STAT_111130.log.012
	egrep 'A_LMCA777:C' TSF_STAT_111130.log.035 > co.out2

### 文件与目录权限修改

- 改变文件的拥有者 chown
- 改变文件读、写、执行等属性 chmod
- 递归子目录修改： chown -R tuxapp source/
- 增加脚本可执行权限： chmod a+x myscript

### 文件增加别名

创建符号链接/硬链接:
	ln cc ccAgain :硬连接；删除一个，将仍能找到；
	ln -s cc ccTo :符号链接(软链接)；删除源，另一个无法使用；（后面一个ccTo 为新建的文件）


### 管道和重定向

- 批处理命令连接执行，使用 |
- 串联: 使用分号 ;
- 前面成功，则执行后面一条，否则，不执行:&&
- 前面失败，则后一条执行: ||

提示命名是否执行成功：

	ls /proc && echo suss! || echo failed.

等同于：

	if ls /proc; then echo suss; else echo fail; fi

重定向：

	ls proc/*.c > list 2> &1 将标准输出和标准错误重定向到同一文件。

等同于：

	ls proc/*.c &> list

清空文件：
	
	:> a.txt

重定向：

	echo aa >> a.txt

### 设置环境变量

启动帐号后自动执行的是文件为 .profile，然后通过这个文件可设置自己的环境变量；

安装的软件路径一般需要加入到path中:
	
	PATH=$APPDIR:/opt/app/soft/bin:$PATH:/usr/local/bin:$TUXDIR/bin:$ORACLE_HOME/bin;export PATH

### Bash快捷输入或删除

快捷键：

- Ctl-U   删除光标到行首的所有字符,在某些设置下,删除全行
- Ctl-W   删除当前光标到前边的最近一个空格之间的字符
- Ctl-H   backspace,删除光标前边的字符
- Ctl-R   匹配最相近的一个文件，然后输出

### 综合应用

查找record.log中包含AAA，但不包含BBB的记录总数：

	cat -v record.log | grep AAA | grep -v BBB | wc -l

### 总结

文件管理，目录的创建、删除、查询、管理: mkdir rm mv

文件的查询和检索: find locate

查看文件内容：cat vi tail more

管道和重定向: ; | && >

