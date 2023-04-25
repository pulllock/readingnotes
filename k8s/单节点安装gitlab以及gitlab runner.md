# 规划

单节点部署gitlab以及gitlab runner

| 虚拟机       | IP           | 安装组件                                | 资源               |
| --------- | ------------ | ----------------------------------- | ---------------- |
| devops-01 | 192.168.1.30 | docker、docker compose、harbor、gitlab | 2CPU、6G内存、100G硬盘 |
| devops-02 | 192.168.1.31 | gitlab runner、maven                 | 1CPU、2G内存、100G硬盘 |

# 安装服务器：devops-01

VirtualBox设置：

- Name: devops-01

- Type: Linux

- Version: Ubuntu (64-bit)

- Memory size: 6144MB

- CPU：2

- Hard disk: Create a virtual hard disk now

- File size: 100GB

- Hard disk file type: VDI (VirtualBox Disk Image)

- Storage on physical hard disk: Dynamically allocated

- Storage: 在这里选择要安装的系统的镜像文件

- Network: Adapter 1选择启用NAT，Adapter 2选择启用Bridged Adapter

设置好之后，选择启动虚拟机进行系统的安装，安装过程中的设置如下：

- Language: English

- Layout: English (US)

- Variant: English (US)

- Choose type of install: Ubuntu Server

- Network connections: 第一个NAT网卡保持默认，第二个Bridged Adapter网卡设置IP为：192.168.1.30，设置如下：

- IPv4 Method: Manual

- Subnet: 192.168.1.0/24

- Address: 192.168.1.30

- Gateway:

- Name servers:

- Search domains:

- Proxy address: 留空

- Mirror address: 默认

- Guided storage configuration: Use an entire disk，不勾选LVM

- Profile setup:

- Your name: devops

- Your server's name: devops-01

- Pick a username: devops

- Choose a password: 12345678

- Confirm your password: 12345678

- SSH Setup: Install OpenSSH server

安装完成后重启进入系统，登录：

- username: devops

- password: 12345678

设置root密码：

- `sudo passwd root`，密码：12345678

修改ssh配置：

- `sudo vim /etc/ssh/sshd_config`

- 将`PermitRootLogin prohibie-password`修改为：`PermitRootLogin yes`

- 将`PasswordAuthentication`修改为：yes

再次重启使用root登录系统：

- username: root

- password: 12345678

关闭防火墙：

- 停用防火墙: `systemctl stop ufw`

- 禁用防火墙: `systemctl disable ufw`

关闭swap：

- 编辑：`vim /etc/fstab`

- 注释掉最后一行：`/swap.img none swap sw 0 0`

关闭selinux：

- Ubuntu Server 22.04.2没有开启selinux，无需进行关闭操作

同步服务器时间：

- 安装ntpdate：`apt install ntpdate`

- 同步时间：`ntpdate time.windows.com`

# 安装服务器：devops-02

VirtualBox设置：

- Name: devops-02

- Type: Linux

- Version: Ubuntu (64-bit)

- Memory size: 2048MB

- CPU：1

- Hard disk: Create a virtual hard disk now

- File size: 100GB

- Hard disk file type: VDI (VirtualBox Disk Image)

- Storage on physical hard disk: Dynamically allocated

- Storage: 在这里选择要安装的系统的镜像文件

- Network: Adapter 1选择启用NAT，Adapter 2选择启用Bridged Adapter

设置好之后，选择启动虚拟机进行系统的安装，安装过程中的设置如下：

- Language: English

- Layout: English (US)

- Variant: English (US)

- Choose type of install: Ubuntu Server

- Network connections: 第一个NAT网卡保持默认，第二个Bridged Adapter网卡设置IP为：192.168.1.31，设置如下：

- IPv4 Method: Manual

- Subnet: 192.168.1.0/24

- Address: 192.168.1.31

- Gateway:

- Name servers:

- Search domains:

- Proxy address: 留空

- Mirror address: 默认

- Guided storage configuration: Use an entire disk，不勾选LVM

- Profile setup:

- Your name: devops

- Your server's name: devops-02

- Pick a username: devops

- Choose a password: 12345678

- Confirm your password: 12345678

- SSH Setup: Install OpenSSH server

安装完成后重启进入系统，登录：

- username: devops

- password: 12345678

设置root密码：

- `sudo passwd root`，密码：12345678

修改ssh配置：

- `sudo vim /etc/ssh/sshd_config`

- 将`PermitRootLogin prohibie-password`修改为：`PermitRootLogin yes`

- 将`PasswordAuthentication`修改为：yes

再次重启使用root登录系统：

- username: root

- password: 12345678

关闭防火墙：

- 停用防火墙: `systemctl stop ufw`

- 禁用防火墙: `systemctl disable ufw`

关闭swap：

- 编辑：`vim /etc/fstab`

- 注释掉最后一行：`/swap.img none swap sw 0 0`

关闭selinux：

- Ubuntu Server 22.04.2没有开启selinux，无需进行关闭操作

同步服务器时间：

- 安装ntpdate：`apt install ntpdate`

- 同步时间：`ntpdate time.windows.com`

# 在devops-01上安装gitlab ce

## 安装gitlab ce

- 执行命令：`curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash`

- 安装：`apt install gitlab-ce`，等待安装完成，完成后继续执行下面命令

- 修改配置：`vim /etc/gitlab/gitlab.rb`，修改：`external_url 'http://192.168.1.30:8060'`

- 执行命令：`gitlab-ctl reconfigure`

- 启动：`gitlab-ctl start`

- 查看状态：`gitlab-ctl status`

- 浏览器访问：`http://192.168.1.30:8060`

## 修改默认管理员账号

- 默认管理员账号：root，密码在`/etc/gitlab/initial_root_password`文件中，示例：`VbLYiv0NPIH4w0PTQT40uOTC3BY3334N+oqhmMue0cM=`

- 使用默认账号密码登录后，修改密码，修改后账号密码：root/Aa123456

# 在devops-02上安装gitlab runner

## 安装gitlab runner

- 执行命令：`curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash`
- 安装：`apt install gitlab-runner`

## 注册gitlab runner到gitlab上

- 浏览器打开gitlab，在gitlab上查询token，位置：`左上角主菜单 --> 管理员 --> CI/CD --> Runner --> Register an instance runner --> Registration token`，示例token：`NcvYCVH6kFtzEKZsnNUE`

- 在devops-02上注册服务，执行命令：`gitlab-runner register`，继续等待输入：
  
  - Enter the GitLab instance URL：`http://192.168.1.30:8060`
  
  - Enter the registration token：`NcvYCVH6kFtzEKZsnNUE`，这个上一步的token
  
  - Enter a description for the runner：`my-gitlab-runner`，这里是输入描述
  
  - Enter tags for the runner (comma-separated)：`first-runner,k8s-runner`，这里是输入标签，可输入多个，用逗号分割
  
  - Enter optional maintenance note for the runner：`这是我的第一个runner，用来测试部署k8s`，这里是输入备注
  
  - Enter an executor: parallels, shell, docker-autoscaler, instance, kubernetes, custom, docker-windows, docker-ssh, ssh, virtualbox, docker+machine, docker-ssh+machine, docker：`shell`，这里是输入runner的执行环境，当前使用的是shell

- 验证gitlab runner是否注册成功，浏览器中打开gitlab进行查询，位置：`左上角主菜单 --> 管理员 --> CI/CD --> Runner`，可以看到刚才注册Runner

# 在devops-02上安装maven

如果使用到maven，则可以安装maven：

- 先安装JDK：`apt install openjdk-17-jdk`

- 再安装maven：`apt install maven`

# 在devops-02上安装docker

如果使用到docker，则可以安装docker：

- 安装docker：`apt install docker.io`

- 修改文件：`vim /etc/docker/daemon.json`，添加`{"insecure-registries":["http://192.168.1.30:8070"]}`，保存修改

- 将gitlab-runner用户添加到docker组中：`usermod -aG docker gitlab-runner`

- 重启docker服务：`systemctl restart docker`

# 在devops-02上安装kubectl并配置

- 安装kubectl：`snap install kubectl --classic`

- 验证安装：`kubectl version --client`

- 进入目录：`cd /home/gitlab-runner`，创建目录`mkdir .kube`，进入刚创建的目录：`cd .kube`

- 在`.kube`目录下创建`config`文件，内容如下：
  
  ```yml
  apiVersion: v1
  clusters:
  - cluster:
      insecure-skip-tls-verify: true
      server: https://192.168.1.10:6443
    name: kubernetes-local-demo
  contexts:
  - context:
      cluster: kubernetes-local-demo
      namespace: default
      user: admin-user
    name: admin
  current-context: admin
  kind: Config
  preferences: {}
  users:
  - name: admin-user
    user:
      token: eyJhbGciOiJSUzI1NiIsImtpZCI6Imh1OGptMElzN1Jib25JbzVaNVIzY3ZyelZVMFJMY1g5Q21pVUktXzBENk0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2YzBjZDUzZS01ZDUyLTQ2YTQtODgyYy0wMDNmZTM0ZmE0OWYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.CH9mBMf2Irx_YRGl-Tkxdg-U3A_TLaSi_csSB6i_H_Za3FO555BX83_YP5wBgbvSAJeZDKUX08oKl6q5sGC_-EPkBtePY1n3CevVG4JGcyUsy0cEI6LLIdEY5P0gmTJ1He7LpGh0NKSCJ8RYdmuukACQBYjDMRU_9S1VZFy7pCjyHoH-V2Wt97gPPGzA5plpW0UlaOtMs-vqcXYO_uzMwZYyewi1lycktw2J_GDxJ3OKku8UH3SXrtsPjZzpWuZb1ddWrLNdWi6GZ4PwVCROZLu8xEgIF00snBsVrEM6jnBuN1IsPnj6wOngHyfW0OUV8TUp4hqCE2qvy4AFe5jSsw
  ```

- 回到`/home/gitlab-runner`目录验证，执行命令：`kubectl --kubeconfig .kube/config cluster-info`