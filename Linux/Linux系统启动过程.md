## Linux系统启动过程

Linux系统的启动过程可以分为5个阶段：

- 内核引导
- 运行init
- 系统初始化
- 建立终端
- 用户登录系统

![Linux运行过程](https://tinyworker.github.io/images/Linux运行过程.png)

### 内核引导

计算机开启电源后，首先是BIOS自检，按照BIOS设置的启动设备来启动。当操作系统接管硬件后，会先读入/boot目录下的内核文件。
![内核引导步骤](https://tinyworker.github.io/images/内核引导步骤.png)

### 运行init

init进程是系统所有进程的起点，会优先读取配置文件，并根据其运行级别启动进程。
![init运行](https://tinyworker.github.io/images/init运行.png)

#### 运行级别

很多程序是需要开机启动的，在Linux中称为“守护进程”（daemon），init进程的一个任务，就是启动这些进程，但由于不同场景所需启动的程序也有所不同，Linux设计了运行级别（runlevel）：

- 运行级别0：系统停机状态，默认运行级别不能为0，否则不能正常启动。
- 运行级别1：单用户工作状态，root权限，用于系统维护，禁止远程登录。
- 运行级别2：多用户状态（没有NFS）
- 运行级别3：完全多用户状态（有NFS），登录后进入控制台命令行模式
- 运行级别4：系统未使用，保留
- 运行级别5：X11控制台，登录后进入图像GUI模式。
- 运行级别6：系统正常关闭并重启，默认不能为6，否则不能正常启动。

### 系统初始化

init配置中包含有一行：

	si::sysinit:/etc/rc.d/rc.sysinit

该配置调用执行脚本/etc/rc.d/rc.sysinit，该脚本会完成系统初始化的部分工作，rc.sysinit是每个运行级别都要首先运行的重要脚本。示例如下图（Ubuntu 18.04），每个运行级别均有一个脚本：
![rc.sysinit](https://tinyworker.github.io/images/rc_sysinit.png)

该脚本主要完成工作有：激活交换分区，检查磁盘，加载硬件模块以及其他一些需要优先执行的任务。

一个配置示例：

	l5:5:wait:/etc/rc.d/rc 5

表示以5位参数运行/etc/rc.d/rc，/etc/rc.d/rc是一个shell脚本，根据参数5，会执行/etc/rc.d/rc5.d/目录下的所有rc启动脚本，目录中的启动脚本实际上是一些链接文件，而不是真正的rc启动脚本，真正的rc脚本都放在/etc/rc.d/init.d目录下。
![rc.d0](https://tinyworker.github.io/images/rc_d.png)

![rc.d5](https://tinyworker.github.io/images/rc_d5.png)

这些脚本通常能接受start,stop,restart,status等参数。
以S开头的脚本，会以start参数来运行，若发现存在对应的脚本也有K字头，且已处于运行态（以/var/lock/subsys/下的文件作为标志），则以stop参数停止这些已经启动了的守护进程，然后在重新运行。

![系统初始化](https://tinyworker.github.io/images/系统初始化.png)

该行为是为了确保当init改变运行级别时，所有相关的守护进程都将重启。对于运行级别中的运行进程，可以通过chkconfig（Redhat）或setup中的System Services来设定（Ubuntu中是update-rc.d）

### 建立终端

rc执行完毕后，返回init，此时基本系统环境已设置，且守护进程已启动。接着会启动6个终端，用于用户登录系统。

	1:2345:respawn:/sbin/mingetty tty1
	2:2345:respawn:/sbin/mingetty tty2
	3:2345:respawn:/sbin/mingetty tty3
	4:2345:respawn:/sbin/mingetty tty4
	5:2345:respawn:/sbin/mingetty tty5
	6:2345:respawn:/sbin/mingetty tty6

可以看到，2,3,4,5都以respawn方式运行mingetty，minegetty程序能打开终端、设置模式。同时它会显示一个文本登录界面，这个就是我们经常看到的登录界面，会提示用户输入用户名，会作为参数到login程序来验证用户身份。

### 用户登录

用户通常有三种方式登录：命令行，图形界面，ssh。

![用户登录](https://tinyworker.github.io/images/用户登录.png)

对于运行级别为5的图形方式来说，登录是通过一个图形化的登录界面，成功后可以进入kde，Gnome等窗口管理器。

文本登录，通常仅root用户，不是root时，会存入/etc/nologin文件，输出内容后退出。这是用来防止非root用户登录，只有在/etc/securetty中登记的终端才允许root用户登录，若不存在，则root可以在任意终端上登录。

/etc/usertty文件用于对用户做出访问限制，若不存在该文件，则没有其他限制。

### Linux关机

通常来说，linux都是服务器运行环境，很少有关机的操作。
正确流程为：sync > shutdown > reboot > halt
指令为：shutdown，可以通过man shutdown来查看帮助文档。

示例如下：

	sync 将数据由内存同步到硬盘中。
	
	shutdown 关机指令，你可以man shutdown 来看一下帮助文档。例如你可以运行如下命令关机：
	
	shutdown –h 10 ‘This server will shutdown after 10 mins’ 这个命令告诉大家，计算机将在10分钟后关机，并且会显示在登陆用户的当前屏幕中。
	
	shutdown –h now 立马关机
	
	shutdown –h 20:25 系统会在今天20:25关机
	
	shutdown –h +10 十分钟后关机
	
	shutdown –r now 系统立马重启
	
	shutdown –r +10 系统十分钟后重启
	
	reboot 就是重启，等同于 shutdown –r now
	
	halt 关闭系统，等同于shutdown –h now 和 poweroff

其中，sync命令在重启或关闭时都需要首先运行，用于将内存数据写到磁盘中。