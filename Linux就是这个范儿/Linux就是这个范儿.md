### 目录和文件处理

`mkdir` 建立目录 `mkdir [option] directory`

`ls` 列出目录下内容 `ls [option] [file]`

`cd`更换工作目录 `cd [directory]`

`pwd` 显示当前工作目录

`cp` 拷贝文件以及目录 `cp [option] source dest`

`mv` 移动或者重命名文件 `mv [option source dest]`

`rm` 删除文件或目录 `rm [option] file`

### 文本处理

`cat` 连接文件打印到标准输出 `cat [option] [file]`

`more` 查看文件内容 当哈umianzai显示满一页的时候暂停 按空格继续或按q结束`more [option] file`

`less` 与more类似，less允许利用光标上下卷动文本内容进行浏览 `less [option] file`

`head` 查看文件头部内容 `head [option] [file]`

`tail` 查看文件尾部内容 `tail [option] [file]` 常用的一个选项是`-f` 可以用于跟随文件增长，显示文件最新的内容，对于在线监控软件日志有帮助。

`echo` 显示一行文本 `echo [option] [string]`

### 系统管理

`ps` 进程查看命令 `ps [option]`

`kill` 删除执行中的程序或工作 `kill [option]`

`jobs` 查到后台正在执行的命令的序号，非进程号pid 

`bg` 指定号码（非进程号）的命令进程放到后台运行 `ctrl + z` 然后 `bg <job id>`

`fg` 指定号码（非进程号）的命令进程放到前台运行 `fg <job id>`

### 文件系统

`du` 查看目录或文件所占用磁盘空间的大小 `du [option] [file]`

`df` 检查文件系统的磁盘空间占用情况 `df [option] [file]`

### /etc/passwd文件查看用户
`/etc/shadow`保存密码

`/etc/passwd`保存用户基本信息

结构如下：

`用户名:密码:UID:GID:用户全名:home目录:shell`

都为0的UID和GID分配给root用户和root用户组。

1~499是属于系统用户的

500~4294967295是分配给普通用户的。

### /etc/group文件 查看组
`/etc/group`用户组文件 是`/etc/passwd`中GID的来源

结构如下：

`组名:用户组密码:GID:用户组内的用户名`

用户组密码存储在`/etc/gshadow`文件中

### 管理用户和组

`adduser` 或`useradd` 添加新用户

`useradd fsocity` 添加一个fsocity用户

`passwd` 修改密码

`usermod`修改用户信息

`userdel`删除用户 `-r`选项 吧用户的home目录一同删除

用户组管理类似：

`groupadd`

`groupmod`

`groupdel`

`gpasswd`

更改`/etc/sudoers`文件可以让普通用户具备sudo特权。

`root	ALL=(ALL)	ALL`

`fsocity	ALL=(ALL)	ALL` 让fsocity用户有root的特权

`%wheel	ALL=(ALL)	ALL` 让wheel用户组的所有用户默认拥有sudo特权。

`%wheel	ALL=(ALL) 	NOPASSED：ALL` 不需要输入密码就拥有root权限

`su` 切换roor用户，不会更改当前所在目录

`su -` 切换root用户，改变当前目录到目标用户的目录下，也就是	`/root`目录下

`su fsocity -` 切换fsocity用户，并切换到`/home/fsocity`目录下

`whoami`

`who am i`

`who`

实际用户（UID）是指用户登录时所使用的用户。

有效用户（EUID）是指当前执行操作的用户，这个是能够利用su或sudo命令进行切换的。


