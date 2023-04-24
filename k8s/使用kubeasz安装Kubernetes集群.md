# 规划

一台master服务器、一台node服务器

| 虚拟机                  | IP           | 安装组件                                                                            | 资源              |
| -------------------- | ------------ | ------------------------------------------------------------------------------- | --------------- |
| kubernetes-master-01 | 192.168.1.10 | kube-apiserver、kube-controller-manager、kube-scheduler、kubernetes-dashboard、etcd | 1CPU、1G内存、20G硬盘 |
| kubernetes-node-01   | 192.168.1.20 | kubelet、kube-proxy、docker                                                       | 1CPU、1G内存、20G硬盘 |

# 总体安装步骤

1. 安装多个虚拟机

2. 虚拟机系统进行初始配置

3. 部署master服务器

4. 部署多个node服务器

# 虚拟机安装

- 虚拟机：VirtualBox

- 系统：Ubuntu Server 22.04.2 LTS，下载地址：https://ubuntu.com/download/server

## 安装master服务器：kubernetes-master-01

VirtualBox设置：

- Name: kubernetes-master-01

- Type: Linux

- Version: Ubuntu (64-bit)

- Memory size: 1024MB

- CPU：1

- Hard disk: Creat a virtual hard disk now

- File size: 20GB

- Hard disk file type: VDI (VirtualBox Disk Image)

- Storage on physical hard disk: Dynamically allocated

- Storage: 在这里选择要安装的系统的镜像文件

- Network: Adapter 1选择启用NAT，Adapter 2选择启用Bridged Adapter

设置好之后，选择启动虚拟机进行系统的安装，安装过程中的设置如下：

- Language: English

- Layout: English (US)

- Variant: English (US)

- Choose type of install: Ubuntu Serve

- Network connections: 第一个NAT网卡保持默认，第二个Bridged Adapter网卡设置IP为：192.168.1.10，设置如下：
  
  - IPv4 Method: Manual
  
  - Subnet: 192.168.1.0/24
  
  - Address: 192.168.1.10
  
  - Gateway:
  
  - Name servers:
  
  - Search domains:

- Proxy address: 留空

- Mirror address: 默认

- Guided storage configuration: Use an entire disk，不勾选LVM

- Profile setup:
  
  - Your name: k8s
  
  - Your server's name: kubernetes-master-01
  
  - Pick a username: k8s
  
  - Choose a password: 12345678
  
  - Confirm your password: 12345678

- SSH Setup: Install OpenSSH server

安装完成后重启进入系统，登录：

- username: k8s

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

- 注释掉最后一行：`/swap.img    none    swap    sw    0     0`

关闭selinux：

- Ubuntu Server 22.04.2没有开启selinux，无需进行关闭操作

同步服务器时间：

- 安装ntpdate：`apt install ntpdate`

- 同步时间：`ntpdate time.windows.com`

## 安装node服务器：kubernetes-node-01

VirtualBox设置：

- Name: kubernetes-node-01

- Type: Linux

- Version: Ubuntu (64-bit)

- Memory size: 1024MB

- Hard disk: Creat a virtual hard disk now

- File size: 20GB

- Hard disk file type: VDI (VirtualBox Disk Image)

- Storage on physical hard disk: Dynamically allocated

- Storage: 在这里选择要安装的系统的镜像文件

- Network: Adapter 1选择启用NAT，Adapter 2选择启用Bridged Adapter

设置好之后，选择启动虚拟机进行系统的安装，安装过程中的设置如下：

- Language: English

- Layout: English (US)

- Variant: English (US)

- Choose type of install: Ubuntu Server

- Network connections: 第一个NAT网卡保持默认，第二个Bridged Adapter网卡设置IP为：192.168.1.20，设置如下：
  
  - IPv4 Method: Manual
  
  - Subnet: 192.168.1.0/24
  
  - Address: 192.168.1.20
  
  - Gateway:
  
  - Name servers:
  
  - Search domains:

- Proxy address: 留空

- Mirror address: 默认

- Guided storage configuration: Use an entire disk，不勾选LVM

- Profile setup:
  
  - Your name: k8s
  
  - Your server's name: kubernetes-node-01
  
  - Pick a username: k8s
  
  - Choose a password: 12345678
  
  - Confirm your password: 12345678

- SSH Setup: Install OpenSSH server

安装完成后重启进入系统，登录：

- username: k8s

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

# 部署节点说明

使用kubernetes-master-01作为部署节点，部署操作在kubernetes-master-01上完成。

# 配置免密登录

此步骤需要在kubernetes-master-01上进行操作。

配置kubernetes-master-01和kubernetes-node-01的免密登录：

- 在kubernetes-master-01上生成ssh秘钥对：`ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519`
- 拷贝公钥到kubernetes-master-01上：`ssh-copy-id root@192.168.1.10`，输入密码后确认，然后使用`ssh root@192.168.1.10`登录kubernetes-master-01进行验证，登录验证成功后exit退出kubernetes-master-01
- 拷贝公钥到kubernetes-node-01上：`ssh-copy-id root@192.168.1.20`，输入密码后确认，然后使用`ssh root@192.168.1.20`登录kubernetes-node-01进行验证，登录验证成功后exit退出kubernetes-node-01

# 安装ansible

此步骤需要在kubernetes-master-01上进行操作。

- 如果没有安装pip，可以先安装pip：`apt install python3-pip`
- 安装ansible：`pip install ansible`，如果安装太慢，可以用阿里源：`pip install ansible -i https://mirrors.aliyun.com/pypi/simple/`

# 下载安装脚本并通过脚本安装kubeasz

此步骤需要在kubernetes-master-01上进行操作。

- 下载安装脚本：`wget https://github.com/easzlab/kubeasz/releases/download/3.5.2/ezdown`
- 修改权限：`chomd +x ./ezdown`
- 使用脚本ezdown下载kubeasz代码、二进制、默认容器镜像：`./ezdown -D`
- 安装完成后，kubeasz安装在了`/etc/kubeasz`目录下
- 下载额外镜像：`./ezdown -X`

# 生成kubernetes集群配置文件

此步骤需要在kubernetes-master-01上进行操作。

- kubernetes集群名为：`kubernetes-local-demo`
- 生成集群配置文件，进入目录：`cd /etc/kubeasz/`， 执行命令：`./ezctl new kubernetes-local-demo`，生成的配置文件在：`/etc/kubeasz/clusters/kubernetes-local-demo`目录下，有两个文件：hosts和config.yml

# 修改集群配置文件

此步骤需要在kubernetes-master-01上进行操作。

集群配置文件在`/etc/kubeasz/clusters/kubernetes-local-demo`目录下，有两个文件：hosts和config.yml，先修改hosts文件`vim hosts`内容如下：

```
# 'etcd' cluster should have odd member(s) (1,3,5,...)
[etcd]
# 192.168.1.1
# 192.168.1.2
# 192.168.1.3
192.168.1.10

# master node(s), set unique 'k8s_nodename' for each node
# CAUTION: 'k8s_nodename' must consist of lower case alphanumeric characters, '-' or '.',
# and must start and end with an alphanumeric character
[kube_master]
# 192.168.1.1 k8s_nodename='master-01'
# 192.168.1.2 k8s_nodename='master-02'
# 192.168.1.3 k8s_nodename='master-03'
192.168.1.10 k8s_nodename='kubernetes-master-01'

# work node(s), set unique 'k8s_nodename' for each node
# CAUTION: 'k8s_nodename' must consist of lower case alphanumeric characters, '-' or '.',
# and must start and end with an alphanumeric character
[kube_node]
# 192.168.1.4 k8s_nodename='worker-01'
# 192.168.1.5 k8s_nodename='worker-02'
192.168.1.20 k8s_nodename='kubernetes-node-01'

# [optional] harbor server, a private docker registry
# 'NEW_INSTALL': 'true' to install a harbor server; 'false' to integrate with existed one
[harbor]
#192.168.1.8 NEW_INSTALL=false

# [optional] loadbalance for accessing k8s from outside
[ex_lb]
#192.168.1.6 LB_ROLE=backup EX_APISERVER_VIP=192.168.1.250 EX_APISERVER_PORT=8443
#192.168.1.7 LB_ROLE=master EX_APISERVER_VIP=192.168.1.250 EX_APISERVER_PORT=8443

# [optional] ntp server for the cluster
[chrony]
#192.168.1.1

[all:vars]
# --------- Main Variables ---------------
# Secure port for apiservers
SECURE_PORT="6443"

# Cluster container-runtime supported: docker, containerd
# if k8s version >= 1.24, docker is not supported
CONTAINER_RUNTIME="containerd"

# Network plugins supported: calico, flannel, kube-router, cilium, kube-ovn
# CLUSTER_NETWORK="calico"
CLUSTER_NETWORK="flannel"

# Service proxy mode of kube-proxy: 'iptables' or 'ipvs'
PROXY_MODE="ipvs"

# K8S Service CIDR, not overlap with node(host) networking
SERVICE_CIDR="10.68.0.0/16"

# Cluster CIDR (Pod CIDR), not overlap with node(host) networking
CLUSTER_CIDR="172.20.0.0/16"

# NodePort Range
NODE_PORT_RANGE="30000-32767"

# Cluster DNS Domain
CLUSTER_DNS_DOMAIN="cluster.local"

# -------- Additional Variables (don't change the default value right now) ---
# Binaries Directory
bin_dir="/opt/kube/bin"

# Deploy Directory (kubeasz workspace)
base_dir="/etc/kubeasz"

# Directory for a specific cluster
cluster_dir="{{ base_dir }}/clusters/kubernetes-local-demo"

# CA and other components cert/key Directory
ca_dir="/etc/kubernetes/ssl"

# Default 'k8s_nodename' is empty
k8s_nodename=''
```

继续修改config.yml文件：`vim config.yml`内容如下：

```yaml
############################
# prepare
############################
# 可选离线安装系统软件包 (offline|online)
INSTALL_SOURCE: "online"

# 可选进行系统安全加固 github.com/dev-sec/ansible-collection-hardening
OS_HARDEN: false


############################
# role:deploy
############################
# default: ca will expire in 100 years
# default: certs issued by the ca will expire in 50 years
CA_EXPIRY: "876000h"
CERT_EXPIRY: "438000h"

# force to recreate CA and other certs, not suggested to set 'true'
CHANGE_CA: false

# kubeconfig 配置参数
CLUSTER_NAME: "kubernetes-local-demo"
CONTEXT_NAME: "context-{{ CLUSTER_NAME }}"

# k8s version
K8S_VER: "1.26.1"

# set unique 'k8s_nodename' for each node, if not set(default:'') ip add will be used
# CAUTION: 'k8s_nodename' must consist of lower case alphanumeric characters, '-' or '.',
# and must start and end with an alphanumeric character (e.g. 'example.com'),
# regex used for validation is '[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*'
K8S_NODENAME: "{%- if k8s_nodename != '' -%} \
                    {{ k8s_nodename|replace('_', '-')|lower }} \
               {%- else -%} \
                    {{ inventory_hostname }} \
               {%- endif -%}"

############################
# role:etcd
############################
# 设置不同的wal目录，可以避免磁盘io竞争，提高性能
ETCD_DATA_DIR: "/var/lib/etcd"
ETCD_WAL_DIR: ""


############################
# role:runtime [containerd,docker]
############################
# ------------------------------------------- containerd
# [.]启用容器仓库镜像
ENABLE_MIRROR_REGISTRY: true

# [containerd]基础容器镜像
SANDBOX_IMAGE: "easzlab.io.local:5000/easzlab/pause:3.9"

# [containerd]容器持久化存储目录
CONTAINERD_STORAGE_DIR: "/var/lib/containerd"

# ------------------------------------------- docker
# [docker]容器存储目录
DOCKER_STORAGE_DIR: "/var/lib/docker"

# [docker]开启Restful API
ENABLE_REMOTE_API: false

# [docker]信任的HTTP仓库
INSECURE_REG: '["http://easzlab.io.local:5000"]'


############################
# role:kube-master
############################
# k8s 集群 master 节点证书配置，可以添加多个ip和域名（比如增加公网ip和域名）
MASTER_CERT_HOSTS:
  # - "10.1.1.1"
  # - "k8s.easzlab.io"
  # - "www.test.com"
  - 192.168.1.10

# node 节点上 pod 网段掩码长度（决定每个节点最多能分配的pod ip地址）
# 如果flannel 使用 --kube-subnet-mgr 参数，那么它将读取该设置为每个节点分配pod网段
# https://github.com/coreos/flannel/issues/847
NODE_CIDR_LEN: 24


############################
# role:kube-node
############################
# Kubelet 根目录
KUBELET_ROOT_DIR: "/var/lib/kubelet"

# node节点最大pod 数
MAX_PODS: 110

# 配置为kube组件（kubelet,kube-proxy,dockerd等）预留的资源量
# 数值设置详见templates/kubelet-config.yaml.j2
KUBE_RESERVED_ENABLED: "no"

# k8s 官方不建议草率开启 system-reserved, 除非你基于长期监控，了解系统的资源占用状况；
# 并且随着系统运行时间，需要适当增加资源预留，数值设置详见templates/kubelet-config.yaml.j2
# 系统预留设置基于 4c/8g 虚机，最小化安装系统服务，如果使用高性能物理机可以适当增加预留
# 另外，集群安装时候apiserver等资源占用会短时较大，建议至少预留1g内存
SYS_RESERVED_ENABLED: "no"


############################
# role:network [flannel,calico,cilium,kube-ovn,kube-router]
############################
# ------------------------------------------- flannel
# [flannel]设置flannel 后端"host-gw","vxlan"等
FLANNEL_BACKEND: "vxlan"
DIRECT_ROUTING: false

# [flannel]
flannel_ver: "v0.19.2"

# ------------------------------------------- calico
# [calico] IPIP隧道模式可选项有: [Always, CrossSubnet, Never],跨子网可以配置为Always与CrossSubnet(公有云建议使用always比较省事，其他的话需要修改各自公有云的网络配置，具体可以参考各个公有云说明)
# 其次CrossSubnet为隧道+BGP路由混合模式可以提升网络性能，同子网配置为Never即可.
CALICO_IPV4POOL_IPIP: "Always"

# [calico]设置 calico-node使用的host IP，bgp邻居通过该地址建立，可手工指定也可以自动发现
IP_AUTODETECTION_METHOD: "can-reach={{ groups['kube_master'][0] }}"

# [calico]设置calico 网络 backend: brid, vxlan, none
CALICO_NETWORKING_BACKEND: "brid"

# [calico]设置calico 是否使用route reflectors
# 如果集群规模超过50个节点，建议启用该特性
CALICO_RR_ENABLED: false

# CALICO_RR_NODES 配置route reflectors的节点，如果未设置默认使用集群master节点
# CALICO_RR_NODES: ["192.168.1.1", "192.168.1.2"]
CALICO_RR_NODES: []

# [calico]更新支持calico 版本: ["3.19", "3.23"]
calico_ver: "v3.24.5"

# [calico]calico 主版本
calico_ver_main: "{{ calico_ver.split('.')[0] }}.{{ calico_ver.split('.')[1] }}"

# ------------------------------------------- cilium
# [cilium]镜像版本
cilium_ver: "1.12.4"
cilium_connectivity_check: true
cilium_hubble_enabled: false
cilium_hubble_ui_enabled: false

# ------------------------------------------- kube-ovn
# [kube-ovn]选择 OVN DB and OVN Control Plane 节点，默认为第一个master节点
OVN_DB_NODE: "{{ groups['kube_master'][0] }}"

# [kube-ovn]离线镜像tar包
kube_ovn_ver: "v1.5.3"

# ------------------------------------------- kube-router
# [kube-router]公有云上存在限制，一般需要始终开启 ipinip；自有环境可以设置为 "subnet"
OVERLAY_TYPE: "full"

# [kube-router]NetworkPolicy 支持开关
FIREWALL_ENABLE: true

# [kube-router]kube-router 镜像版本
kube_router_ver: "v0.3.1"
busybox_ver: "1.28.4"


############################
# role:cluster-addon
############################
# coredns 自动安装
dns_install: "yes"
corednsVer: "1.9.3"
ENABLE_LOCAL_DNS_CACHE: true
dnsNodeCacheVer: "1.22.13"
# 设置 local dns cache 地址
LOCAL_DNS_CACHE: "169.254.20.10"

# metric server 自动安装
metricsserver_install: "yes"
metricsVer: "v0.5.2"

# dashboard 自动安装
dashboard_install: "yes"
dashboardVer: "v2.7.0"
dashboardMetricsScraperVer: "v1.0.8"

# prometheus 自动安装
prom_install: "no"
prom_namespace: "monitor"
prom_chart_ver: "39.11.0"

# nfs-provisioner 自动安装
nfs_provisioner_install: "no"
nfs_provisioner_namespace: "kube-system"
nfs_provisioner_ver: "v4.0.2"
nfs_storage_class: "managed-nfs-storage"
nfs_server: "192.168.1.10"
nfs_path: "/data/nfs"

# network-check 自动安装
network_check_enabled: false
network_check_schedule: "*/5 * * * *"

############################
# role:harbor
############################
# harbor version，完整版本号
HARBOR_VER: "v2.6.3"
HARBOR_DOMAIN: "harbor.easzlab.io.local"
HARBOR_PATH: /var/data
HARBOR_TLS_PORT: 8443
HARBOR_REGISTRY: "{{ HARBOR_DOMAIN }}:{{ HARBOR_TLS_PORT }}"

# if set 'false', you need to put certs named harbor.pem and harbor-key.pem in directory 'down'
HARBOR_SELF_SIGNED_CERT: true

# install extra component
HARBOR_WITH_NOTARY: false
HARBOR_WITH_TRIVY: false
HARBOR_WITH_CHARTMUSEUM: true
```

# 安装集群

此步骤需要在kubernetes-master-01上进行操作。

- 进入目录：`cd /etc/kubeasz/`，执行命令进行安装：`./ezctl setup kubernetes-local-demo all`

# 安装完成后查看集群状态

此步骤需要在kubernetes-master-01上进行操作。

- `kubectl get node`可以查看到各节点状态
- `kubectl get pod --all-namespaces`可以查看所有集群pod状态
- `kubectl get svc --all-namespaces`可以查看所有集群服务状态

# 使用kubernetes-dashboard

此步骤需要在kubernetes-master-01上进行操作。

- 查看kubernetes-dashboard端口，执行命令：`kubectl get svc --all-namespaces`，可以在结果中找到`kube-system   kubernetes-dashboard        NodePort    10.68.63.43     <none>        443:30971/TCP            58m`，其中30971是访问的端口号

- 在浏览器中使用地址访问dashboard：`https://192.168.1.20:30971`，注意这里IP是kubernetes-node-01的ip

- 登录方式使用Token方式，需要再kubernetes-master-01上查询token：`kubectl describe -n kube-system secrets admin-user`，在结果中找到`token:`后面的就是需要的Token，示例：`eyJhbGciOiJSUzI1NiIsImtpZCI6Imh1OGptMElzN1Jib25JbzVaNVIzY3ZyelZVMFJMY1g5Q21pVUktXzBENk0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2YzBjZDUzZS01ZDUyLTQ2YTQtODgyYy0wMDNmZTM0ZmE0OWYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.CH9mBMf2Irx_YRGl-Tkxdg-U3A_TLaSi_csSB6i_H_Za3FO555BX83_YP5wBgbvSAJeZDKUX08oKl6q5sGC_-EPkBtePY1n3CevVG4JGcyUsy0cEI6LLIdEY5P0gmTJ1He7LpGh0NKSCJ8RYdmuukACQBYjDMRU_9S1VZFy7pCjyHoH-V2Wt97gPPGzA5plpW0UlaOtMs-vqcXYO_uzMwZYyewi1lycktw2J_GDxJ3OKku8UH3SXrtsPjZzpWuZb1ddWrLNdWi6GZ4PwVCROZLu8xEgIF00snBsVrEM6jnBuN1IsPnj6wOngHyfW0OUV8TUp4hqCE2qvy4AFe5jSsw`，拷贝到浏览器的dashboard界面中进行登录即可

# 配置kubernetes-master-01的docker的仓库为自定义部署的harbor

自定义的harbor部署参考：单节点安装harbor.md。

此步骤需要在kubernetes-master-01上进行操作。

- 修改文件：`vim /etc/docker/daemon.json`，将`"insecure-registries": ["http://easzlab.io.local:5000"]`修改为`"insecure-registries":["http://192.168.1.30:8070"]`，保存修改

- 重启docker服务：`systemctl restart docker`

# 配置kubernetes-master-01的containerd的仓库为自定义部署的harbor

自定义的harbor部署参考：单节点安装harbor.md。

此步骤需要在kubernetes-master-01上进行操作。

- 修改文件：`vim /etc/containerd/config.toml`，找到`[plugins."io.containerd.grpc.v1.cri".registry]`，在`[plugins."io.containerd.grpc.v1.cri".registry.configs]`下面添加：
  
  ```toml
  [plugins."io.containerd.grpc.v1.cri".registry.configs."192.168.1.30:8070".tls]
            insecure_skip_verify = true
  ```

- 在`[plugins."io.containerd.grpc.v1.cri".registry.mirrors]`下面添加：
  
  ```toml
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."192.168.1.30:8070"]
            endpoint = ["http://192.168.1.30:8070"]
  ```

- 重启containerd服务：`systemctl restart containerd`