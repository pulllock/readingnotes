昨晚半夜网上一个朋友找到我，说是使用gogs搭建git服务器，使用ssh操作要免密啥啥啥的～也没描述清楚。就是要ssh的方式，提交时候不要账号密码，心想这不就三下的事情吗？结果折腾到晚上一点，没好～敢肯定的是他按照网上的毒教程，被坑了！还是自己本地虚拟机配置一下吧～

# 环境说明

- 本机Ubuntu16.10
- virtualbox上运行的是Centos7
- 虚拟机中mysql已经安装好
- 虚拟机中firewall已禁用，安装了iptables
- 虚拟机中已经安装git

# 步骤

- 去gogs网站下载，这里下载的是0.10.18版本，文件名是`linux_amd64.zip`
- mysql建立gogs数据库
- 新建用户名字为git的用户（用户目录/home/git）
- 解压下载的文件，然后运行程序
- 配置，安装
- 现在已经可以访问了，也可以使用http方式进行clone和提交了
- 配置ssh方式

## 下载gogs
去gogs网站下载，[https://dl.gogs.io/](https://dl.gogs.io/) ，我下载的是0.10.18，linux 64位版本。

## 建立gogs数据库
在mysql中建立gogs数据库。

## 新建git用户
在虚拟机Centos中新建一个git用户。

- 创建git组：`sudo groupadd git`。
- 创建git用户，分到git组中：`sudo useradd -g git git`
- 设置git用户的密码：`sudo passwd git`

接下来切换到刚才新建的git用户，一定要切换到这个git用户！！！！

切换用户：`su git`

## 解压文件，运行
现在已经切换到git这个用户了，切记一定要切换到git这个用户才能执行以下步骤。

首先进入`/home/git`目录下，将下载的文件解压到`/home/git`目录下并重新命名，我这里是命名为gogs。然后进入gogs文件夹下，运行`./gogs web`，应该没啥错。

## 配置，安装
上面运行完成之后，打开浏览器输入：[http://localhost:3000/install](http://localhost:3000/install) ，就可以看到安装配置页面了，里面配置根据自己需要配置（请先阅读文档了解清楚了，再自定义配置。）我这里填了mysql的密码，其他基本都是默认值。点击保存，有可能会提示git的path问题，请安装git！

## 测试http方式
现在已经可以访问了，访问：[http://localhost:3000](http://localhost:3000) 不出意外，可以看到页面了。接下来需要注册一个用户，然后登录，添加一个仓库，在局域网中使用http的方式clone，我猜应该没啥意外情况。我这里是`http://192.168.1.104:3000/dachengxi/gogs-test.git`，你的根据情况来。

## 使用ssh方式
首先需要在你的机器上生成ssh公钥：`ssh-keygen -t rsa -C "your_email@example.com"`，各种回车之后完成，生成的文件在你的用户主目录下的.ssh文件夹下，其中id_rsa.pub文件中的内容是我们需要的。打开此文件，复制所有内容。

然后打开gogs页面，点击右上角头像，找到用户设置，然后选择管理SSH密钥，在这里添加一个密钥，名字随便输，下面内容是你刚才复制的那个id_rsa.pub文件中的内容，添加进去保存，就好了。（其实这一步就是在你git用户主目录下的.ssh文件夹下生成一个叫做authorized_keys的文件，里面内容就是上面你添加的内容）。

## 测试ssh方式
上面的步骤没出啥错，现在已经可以使用，我这里是`git@192.168.1.104:dachengxi/gogs-test.git`，你的根据自己情况来定。

# 其他
其他各种高级功能不做讨论，请自己找文档找文章找自己！

请确认虚拟机防火墙开放了3000端口，22端口。

请确认git已经安装。

请确认你运行gogs的时候，是你新建的git用户。