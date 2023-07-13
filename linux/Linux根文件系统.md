# 根文件系统目录结构

| 目录          | 全称                                     | 作用                                                            |
| ----------- | -------------------------------------- | ------------------------------------------------------------- |
| /bin        | Binaries                               | 存放系统命令，所有用户可执行。是/usr/bin目录的软连接                                |
| /sbin       | Super User Binaries                    | 存放和系统环境设置相关的命令，只有超级用户可以使用，有些命令在有授权时普通用户也可以使用。是/usr/sbin目录的软连接 |
| /etc        | Editable Text Configuration Chest      | 存放配置文件                                                        |
| /home       | Home                                   | 用户主目录                                                         |
| /root       | Root                                   | root用户的主目录                                                    |
| /lib        | Libary                                 | 存放系统运行所需的共享库                                                  |
| /dev        | Devices                                | 存放设备文件                                                        |
| /tmp        | Temporary                              | 存放临时文件                                                        |
| /boot       | Boot                                   | 存放引导加载程序使用的文件                                                 |
| /mnt        | Mount                                  | 挂载目录，挂在临时文件系统                                                 |
| /opt        | Optional Application Software Packages | 可选应用安装包，第三方软件安装保存位置                                           |
| /usr        | Unix Shared Resources                  | Unix共享资源目录，存放命令、库、手册等                                         |
| /usr/bin    | Unix Shared Resources Binaries         | 存放系统命令，所有用户可以执行                                               |
| /usr/sbin   | Super User Binaries                    | 存放根文件系统不必要的系统管理命令，超级用户可执行                                     |
| /usr/local  | Local                                  | 安装第三方软件的目录，一般是通过编译源码的方式安装的软件                                  |
| /proc       | Processes                              | 表示内核的当前状态信息                                                   |
| /var        | Variable                               | 存储各种变化的文件，比如log等                                              |
| /lost+found | Lost And Found                         | 存放一些系统出错的检查结果                                                 |
| /srv        | Server                                 | 服务数据目录                                                        |
| /sys        | System                                 | 文件系统，把内核模块的设备信息和驱动信息导出到用户态，同时也可以通过用户态对设备进行设置                  |
| /media      | Media                                  | 挂载目录                                                          |
| /misc       | Miscellaneous Device                   | 挂载目录                                                          |
| /run        | Run                                    | 系统运行时需要，重启后会重新生成                                              |
