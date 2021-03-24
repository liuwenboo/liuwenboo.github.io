## 卸载

1. 控制面板删除mysql后删除Program Files(x86)和ProgrameData中的MySQL文件夹

2. 清理注册表：（一般仅进行第一步，重新安装失败的话再清理注册表即可）

   HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\Eventlog\Application\MySQL 目录

   HKEY_LOCAL_MACHINE\SYSTEM\ControlSet002\Services\Eventlog\Application\MySQL 目录

   HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Eventlog\Application\MySQL 目录

   HKEY_LOCAL_MACHINE\SYSTEM\CurrentControl001\Services\MySQL 目录

   HKEY_LOCAL_MACHINE\SYSTEM\CurrentControl002\Services\MySQL 目录

   HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\MySQL 目录

## 我的配置

root密码：123456

service name：MySQL80