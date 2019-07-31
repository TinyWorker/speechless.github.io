###linux普通用户给予root权限

首先让普通用户具有root权限，需要将用户添加到sudoer列表中。

使用指令：visudo

进入编辑页面后，找到root

root	ALL=(ALL)	ALL

在其下添加，如下命令，即可将admin用户授予root权限。

admin	ALL=(ALL)	ALL

若需要无密码登录，如下

admin	ALL=(ALL)	NOPASSWD:ALL