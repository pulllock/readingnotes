# 规划

一台master服务器、两台node服务器

| 虚拟机                  | IP           | 安装组件                                                       |     |
| -------------------- | ------------ | ---------------------------------------------------------- | --- |
| kubernetes-master-01 | 192.168.1.10 | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |     |
| kubernetes-node-01   | 192.168.1.20 | kubelet、kube-proxy、docker、etcd                             |     |
| kubernetes-node-02   | 192.168.1.21 | kubelet、kube-proxy、docker、etcd                             |     |

# 总体安装步骤

1. 安装多个虚拟机

2. 虚拟机系统进行初始配置

3. 部署master服务器

4. 部署多个node服务器

# 虚拟机安装

- 虚拟机：VirtualBox

- 系统：Ubuntu Server 22.04.1 LTS，下载地址：https://ubuntu.com/download/server

## 安装master服务器：kubernetes-master-01

VirtualBox设置：

- Name: kubernetes-master-01

- Type: Linux

- Version: Ubuntu (64-bit)

- Memory size: 1024MB

- Hard disk: Creat a virtual hard disk now

- File size: 20.00GB

- Hard disk file type: VDI (VirtualBox Disk Image)

- Storage on physical hard disk: Dynamically allocated

- Storage: 在这里选择要安装的系统的镜像文件

- Network: Adapter 1选择启用NAT，Adapter 2选择启用Bridged Adapter

设置好之后，选择启动虚拟机进行系统的安装，安装过程中的设置如下：

- Language: English

- Layout: English (US)

- Variant: English (US)

- Choose type of install: Ubuntu Server

- Network connections: 第一个NAT网卡保持默认，第二个Bridged Adapter网卡设置IP为：192.168.1.10，设置如下：
  
  - IPv4 Method: Manual
  
  - Subnet: 192.168.1.0/24
  
  - Address: 192.168.1.10
  
  - Gateway: 192.168.1.1
  
  - Name servers: 
  
  - Search domains: 

- Proxy address: 留空

- Mirror address: 默认

- Guided storage configuration: Use an entire disk

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

关闭防火墙：

- 停用防火墙: sudo systemctl stop ufw

- 禁用防火墙: sudo systemctl disable ufw

关闭swap：

- 编辑：`vim /etc/fstab`

- 注释掉最后一行：`/swap.img    none    swap    sw    0     0`

关闭selinux：

- Ubuntu Server 22.04.1没有开启selinux，无需进行关闭操作

同步服务器时间：

- 安装ntpdate：`sudo apt install ntpdate`

- 同步时间：`sudo ntpdate time.windows.com`

安装cfssl：

- `sudo apt install golang-cfssl`

## 安装node服务器：kubernetes-node-01

VirtualBox设置：

- Name: kubernetes-node-01

- Type: Linux

- Version: Ubuntu (64-bit)

- Memory size: 1024MB

- Hard disk: Creat a virtual hard disk now

- File size: 20.00GB

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
  
  - Gateway: 192.168.1.1
  
  - Name servers:
  
  - Search domains:

- Proxy address: 留空

- Mirror address: 默认

- Guided storage configuration: Use an entire disk

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

关闭防火墙：

- 停用防火墙: sudo systemctl stop ufw

- 禁用防火墙: sudo systemctl disable ufw

关闭swap：

- 编辑：`vim /etc/fstab`

- 注释掉最后一行：`/swap.img none swap sw 0 0`

关闭selinux：

- Ubuntu Server 22.04.1没有开启selinux，无需进行关闭操作

同步服务器时间：

- 安装ntpdate：`sudo apt install ntpdate`

- 同步时间：`sudo ntpdate time.windows.com`

安装cfssl：

- `sudo apt install golang-cfssl`

## 安装node服务器：kubernetes-node-02

VirtualBox设置：

- Name: kubernetes-node-02

- Type: Linux

- Version: Ubuntu (64-bit)

- Memory size: 1024MB

- Hard disk: Creat a virtual hard disk now

- File size: 20.00GB

- Hard disk file type: VDI (VirtualBox Disk Image)

- Storage on physical hard disk: Dynamically allocated

- Storage: 在这里选择要安装的系统的镜像文件

- Network: Adapter 1选择启用NAT，Adapter 2选择启用Bridged Adapter

设置好之后，选择启动虚拟机进行系统的安装，安装过程中的设置如下：

- Language: English

- Layout: English (US)

- Variant: English (US)

- Choose type of install: Ubuntu Server

- Network connections: 第一个NAT网卡保持默认，第二个Bridged Adapter网卡设置IP为：192.168.1.21，设置如下：
  
  - IPv4 Method: Manual
  
  - Subnet: 192.168.1.0/24
  
  - Address: 192.168.1.21
  
  - Gateway: 192.168.1.1
  
  - Name servers:
  
  - Search domains:

- Proxy address: 留空

- Mirror address: 默认

- Guided storage configuration: Use an entire disk

- Profile setup:
  
  - Your name: k8s
  
  - Your server's name: kubernetes-node-02
  
  - Pick a username: k8s
  
  - Choose a password: 12345678
  
  - Confirm your password: 12345678

- SSH Setup: Install OpenSSH server

安装完成后重启进入系统，登录：

- username: k8s

- password: 12345678

关闭防火墙：

- 停用防火墙: sudo systemctl stop ufw

- 禁用防火墙: sudo systemctl disable ufw

关闭swap：

- 编辑：`vim /etc/fstab`

- 注释掉最后一行：`/swap.img none swap sw 0 0`

关闭selinux：

- Ubuntu Server 22.04.1没有开启selinux，无需进行关闭操作

同步服务器时间：

- 安装ntpdate：`sudo apt install ntpdate`

- 同步时间：`sudo ntpdate time.windows.com`

安装cfssl：

- `sudo apt install golang-cfssl`

# 安装etcd

## kubernetes-master-01上安装etcd

安装etcd：`sudo apt install etcd`

## 生成etcd证书

- 创建生成etcd证书的目录：`mkdir -p ~/kubernetes/etcd-cert`

- 进入上面创建的目录中：`cd ~/kubernetes/etcd-cert`

- 创建ca-config.json文件（使用此文件来生成CA文件），文件内容如下：
  
  ```
  {
    "signing": {
      "default": {
        "expiry": "87600h"
      },
      "profiles": {
        "www": {
           "expiry": "87600h",
           "usages": [
              "signing",
              "key encipherment",
              "server auth",
              "client auth"
          ]
        }
      }
    }
  }
  ```

- 创建ca-csr.json文件（用来生成CA证书签名请求），文件内容如下：
  
  ```
  {
      "CN": "etcd CA",
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "Beijing",
              "ST": "Beijing"
          }
      ]
  } 
  ```

- 生成CA证书（ca.pem）和秘钥（ca-key.pem）：`cfssl gencert -initca ca-csr.json | cfssljson -bare ca`

- 创建server-csr.json文件（用来创建etcd证书签名），内容如下：
  
  ```
  {
      "CN": "etcd",
      "hosts": [
          "192.168.1.10",
          "192.168.1.20",
          "192.168.1.21"
          ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "BeiJing",
              "ST": "BeiJing"
          }
      ]
  }
  ```

- 生成etcd证书和私钥：`cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server`

- 创建存放etcd证书的目录：`sudo mkdir -p /opt/etcd/ssl`

- 将生成的证书拷贝到存放etcd证书的目录：`sudo cp ~/kubernetes/etcd-cert/{ca,server-key,server}.pem /opt/etcd/ssl`

## 配置etcd

编辑配置文件：`sudo vim /etc/default/etcd`，添加如下内容到配置文件中：

```
# 节点名称
ETCD_NAME="etcd-1"
# 数据目录
ETCD_DATA_DIR="/var/lib/etcd"
# 集群通信监听地址，当前服务器IP
ETCD_LISTEN_PEER_URLS="https://192.168.1.10:2380"
# 客户端访问监听地址，当前服务器IP
ETCD_LISTEN_CLIENT_URLS="https://192.168.1.10:2379"
# 集群通告地址，当前服务器IP
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.10:2380"
# 客户端通告地址
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.10:2379"
# 集群节点地址
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.1.10:2380,etcd-2=https://192.168.1.20:2380,etcd-3=https://192.168.1.21:2380"
# 集群Token
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
# 加入集群的当前状态，new-新集群 existing-加入已有集群
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_CERT_FILE="/opt/etcd/ssl/server.pem"
ETCD_KEY_FILE="/opt/etcd/ssl/server-key.pem"
ETCD_PEER_CERT_FILE="/opt/etcd/ssl/server.pem"
ETCD_PEER_KEY_FILE="/opt/etcd/ssl/server-key.pem"
ETCD_TRUSTED_CA_FILE="/opt/etcd/ssl/ca.pem"
ETCD_PEER_TRUSTED_CA_FILE="/opt/etcd/ssl/ca.pem"
```

开机启动etcd：`sudo systemctl enable etcd`

重启etcd：`sudo systemctl restart etcd`

## kubernetes-node-01上安装etcd

安装etcd：`sudo apt install etcd`

## 生成etcd证书

- 创建生成etcd证书的目录：`mkdir -p ~/kubernetes/etcd-cert`

- 进入上面创建的目录中：`cd ~/kubernetes/etcd-cert`

- 创建ca-config.json文件（使用此文件来生成CA文件），文件内容如下：
  
  ```
  {
    "signing": {
      "default": {
        "expiry": "87600h"
      },
      "profiles": {
        "www": {
           "expiry": "87600h",
           "usages": [
              "signing",
              "key encipherment",
              "server auth",
              "client auth"
          ]
        }
      }
    }
  }
  ```

- 创建ca-csr.json文件（用来生成CA证书签名请求），文件内容如下：
  
  ```
  {
      "CN": "etcd CA",
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "Beijing",
              "ST": "Beijing"
          }
      ]
  } 
  ```

- 生成CA证书（ca.pem）和秘钥（ca-key.pem）：`cfssl gencert -initca ca-csr.json | cfssljson -bare ca`

- 创建server-csr.json文件（用来创建etcd证书签名），内容如下：
  
  ```
  {
      "CN": "etcd",
      "hosts": [
          "192.168.1.10",
          "192.168.1.20",
          "192.168.1.21"
          ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "BeiJing",
              "ST": "BeiJing"
          }
      ]
  }
  ```

- 生成etcd证书和私钥：`cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server`

- 创建存放etcd证书的目录：`sudo mkdir -p /opt/etcd/ssl`

- 将生成的证书拷贝到存放etcd证书的目录：`sudo cp ~/kubernetes/etcd-cert/{ca,server-key,server}.pem /opt/etcd/ssl`

## 配置etcd

编辑配置文件：`sudo vim /etc/default/etcd`，添加如下内容到配置文件中：

```
# 节点名称
ETCD_NAME="etcd-2"
# 数据目录
ETCD_DATA_DIR="/var/lib/etcd"
# 集群通信监听地址，当前服务器IP
ETCD_LISTEN_PEER_URLS="https://192.168.1.20:2380"
# 客户端访问监听地址，当前服务器IP
ETCD_LISTEN_CLIENT_URLS="https://192.168.1.20:2379"
# 集群通告地址，当前服务器IP
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.20:2380"
# 客户端通告地址
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.20:2379"
# 集群节点地址
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.1.10:2380,etcd-2=https://192.168.1.20:2380,etcd-3=https://192.168.1.21:2380"
# 集群Token
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
# 加入集群的当前状态，new-新集群 existing-加入已有集群
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_CERT_FILE="/opt/etcd/ssl/server.pem"
ETCD_KEY_FILE="/opt/etcd/ssl/server-key.pem"
ETCD_PEER_CERT_FILE="/opt/etcd/ssl/server.pem"
ETCD_PEER_KEY_FILE="/opt/etcd/ssl/server-key.pem"
ETCD_TRUSTED_CA_FILE="/opt/etcd/ssl/ca.pem"
ETCD_PEER_TRUSTED_CA_FILE="/opt/etcd/ssl/ca.pem"
```

开机启动etcd：`sudo systemctl enable etcd`

重启etcd：`sudo systemctl restart etcd`

## kubernetes-node-02上安装etcd

安装etcd：`sudo apt install etcd`

## 生成etcd证书

- 创建生成etcd证书的目录：`mkdir -p ~/kubernetes/etcd-cert`

- 进入上面创建的目录中：`cd ~/kubernetes/etcd-cert`

- 创建ca-config.json文件（使用此文件来生成CA文件），文件内容如下：
  
  ```
  {
    "signing": {
      "default": {
        "expiry": "87600h"
      },
      "profiles": {
        "www": {
           "expiry": "87600h",
           "usages": [
              "signing",
              "key encipherment",
              "server auth",
              "client auth"
          ]
        }
      }
    }
  }
  ```

- 创建ca-csr.json文件（用来生成CA证书签名请求），文件内容如下：
  
  ```
  {
      "CN": "etcd CA",
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "Beijing",
              "ST": "Beijing"
          }
      ]
  } 
  ```

- 生成CA证书（ca.pem）和秘钥（ca-key.pem）：`cfssl gencert -initca ca-csr.json | cfssljson -bare ca`

- 创建server-csr.json文件（用来创建etcd证书签名），内容如下：
  
  ```
  {
      "CN": "etcd",
      "hosts": [
          "192.168.1.10",
          "192.168.1.20",
          "192.168.1.21"
          ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "BeiJing",
              "ST": "BeiJing"
          }
      ]
  }
  ```

- 生成etcd证书和私钥：`cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server`

- 创建存放etcd证书的目录：`sudo mkdir -p /opt/etcd/ssl`

- 将生成的证书拷贝到存放etcd证书的目录：`sudo cp ~/kubernetes/etcd-cert/{ca,server-key,server}.pem /opt/etcd/ssl`

## 配置etcd

编辑配置文件：`sudo vim /etc/default/etcd`，添加如下内容到配置文件中：

```
# 节点名称
ETCD_NAME="etcd-3"
# 数据目录
ETCD_DATA_DIR="/var/lib/etcd"
# 集群通信监听地址，当前服务器IP
ETCD_LISTEN_PEER_URLS="https://192.168.1.21:2380"
# 客户端访问监听地址，当前服务器IP
ETCD_LISTEN_CLIENT_URLS="https://192.168.1.21:2379"
# 集群通告地址，当前服务器IP
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.21:2380"
# 客户端通告地址
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.21:2379"
# 集群节点地址
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.1.10:2380,etcd-2=https://192.168.1.20:2380,etcd-3=https://192.168.1.21:2380"
# 集群Token
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
# 加入集群的当前状态，new-新集群 existing-加入已有集群
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_CERT_FILE="/opt/etcd/ssl/server.pem"
ETCD_KEY_FILE="/opt/etcd/ssl/server-key.pem"
ETCD_PEER_CERT_FILE="/opt/etcd/ssl/server.pem"
ETCD_PEER_KEY_FILE="/opt/etcd/ssl/server-key.pem"
ETCD_TRUSTED_CA_FILE="/opt/etcd/ssl/ca.pem"
ETCD_PEER_TRUSTED_CA_FILE="/opt/etcd/ssl/ca.pem"
```

开机启动etcd：`sudo systemctl enable etcd`

重启etcd：`sudo systemctl restart etcd`

# 安装docker

## kubernetes-node-01上安装docker

安装docker：

- `sudo apt update`

- 可选：`sudo apt install ca-certificates curl gnupg lsb-release`

- 可选：`sudo mkdir -p /etc/apt/keyrings`

- `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg`

- `echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

- `sudo apt update`

- `sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin`

开机启动docker：`sudo systemctl enable docker`

## kubernetes-node-02上安装docker

安装docker：

- `sudo apt update`

- 可选：`sudo apt install ca-certificates curl gnupg lsb-release`

- 可选：`sudo mkdir -p /etc/apt/keyrings`

- `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg`

- `echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

- `sudo apt update`

- `sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin`

开机启动docker：`sudo systemctl enable docker`

# 安装kube-apiserver

## kubernetes-master-01上安装kube-apiserver

### 生成证书

- 创建生成证书的目录：`mkdir -p ~/kubernetes/k8s-cert`

- 进入上面创建的目录中：`cd ~/kubernetes/k8s-cert`

- 创建ca-config.json文件（使用此文件来生成CA文件），文件内容如下：
  
  ```
  {
    "signing": {
      "default": {
        "expiry": "87600h"
      },
      "profiles": {
        "kubernetes": {
           "expiry": "87600h",
           "usages": [
              "signing",
              "key encipherment",
              "server auth",
              "client auth"
          ]
        }
      }
    }
  }
  ```

- 创建ca-csr.json文件（用来生成CA证书签名请求），文件内容如下：
  
  ```
  {
      "CN": "kubernetes",
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "Beijing",
              "ST": "Beijing",
              "O": "k8s",
              "OU": "System"
          }
      ]
  } 
  ```

- 生成CA证书（ca.pem）和秘钥（ca-key.pem）：`cfssl gencert -initca ca-csr.json | cfssljson -bare ca`

- 创建server-csr.json文件（用来创建kube-apiserver证书签名），内容如下：
  
  ```
  {
      "CN": "kubernetes",
      "hosts": [
          "192.168.1.10"
          ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "BeiJing",
              "ST": "BeiJing",
              "O": "k8s",
              "OU": "System"
          }
      ]
  }
  ```

- 生成kube-apiserver证书和私钥：`cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server`

- 创建存放kubernetes证书的目录：`sudo mkdir -p /opt/kubernetes/ssl`

- 将生成的证书拷贝到存放kubernetes证书的目录：`sudo cp ~/kubernetes/k8s-cert/{ca,ca-key,server-key,server}.pem /opt/kubernetes/ssl`

### 创建TLSBootstrapping Token

- 进入目录：`/opt/kubernetes/conf`

- 执行命令：`head -c 16 /dev/urandom | od -An -t x | tr -d ' '`，会生成一个token

- 编辑或者创建文件`token`，文件内容：`上面生成的token,kubelete-bootstrap,10001,"system:bootstrappers"`

### 安装和配置

- 下载地址：`https://www.downloadkubernetes.com/`，OS: `linux`，architectures: `amd64`，version: `v1.25.2`

- 将下载的文件：`kube-apiserver`加上执行权限：`sudo chmod 755 kube-apiserver`

- 创建kubernetes安装目录：`sudo mkdir /opt/kubernetes/{bin,conf,ssl,logs} -p`

- 将`kube-apiserver`文件拷贝到`/opt/kubernetes/bin/`目录下

- 在`/opt/kubernetes/conf/`目录下创建`kube-apiserver.conf`配置文件，配置文件内容如下：
  
  ```
  KUBE_APISERVER_OPTS="--logtostderr=false \
  --v=2 \
  --log-dir=/opt/kubernetes/logs \
  --etcd-servers=https://192.168.1.10:2379,https://192.168.1.20:2379,https://192.168.1.21:2379 \
  --bind-address=192.168.1.10 \
  --secure-port=6443 \
  --advertise-address=192.168.1.10 \
  --allow-privileged=true \
  --service-cluster-ip-range=192.168.1.0/24 \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
  --authorization-mode=RBAC,Node \
  --enable-bootstrap-token-auth=true \
  --token-auth-file=/opt/kubernetes/conf/token\
  --service-node-port-range=30000-32767 \
  --kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \
  --kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \
  --tls-cert-file=/opt/kubernetes/ssl/server.pem  \
  --tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \
  --client-ca-file=/opt/kubernetes/ssl/ca.pem \
  --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/opt/etcd/ssl/ca.pem \
  --etcd-certfile=/opt/etcd/ssl/server.pem \
  --etcd-keyfile=/opt/etcd/ssl/server-key.pem \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
  ```

- 使用systemd管理kube-apiserver，在`/usr/lib/systemd/system/`，目录下创建文件：`kube-apiserver.service`文件，文件内容如下：
  
  ```
  [Unit]
  Description=Kubernetes API Server
  Documentation=https://github.com/kubernetes/kubernetes
  
  [Service]
  EnvironmentFile=/opt/kubernetes/conf/kube-apiserver.conf
  ExecStart=/opt/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
  Restart=on-failure
  
  [Install]
  WantedBy=multi-user.target
  ```

- 重新加载服务：`systemctl daemon-reload`

- 自启动：`systemctl enable kube-apiserver`

# 安装kube-controller-manager

## kubernetes-master-01上安装kube-controller-manager

### 生成证书

证书可以直接使用kube-apiserver安装时生成的证书

## 安装和配置

- 下载地址：`https://www.downloadkubernetes.com/`，OS: `linux`，architectures: `amd64`，version: `v1.25.2`

- 将下载的文件：`kube-controller-manager`加上执行权限：`sudo chmod 755 kube-controller-manager`

- 创建kubernetes安装目录：`sudo mkdir /opt/kubernetes/{bin,conf,ssl,logs} -p`

- 将`kube-controller-manager`文件拷贝到`/opt/kubernetes/bin/`目录下

- 在`/opt/kubernetes/conf/`目录下创建`kube-controller-manager.conf`配置文件，配置文件内容如下：
  
  ```
  KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \
  --v=2 \
  --log-dir=/opt/kubernetes/logs \
  --leader-elect=true \
  --master=127.0.0.1:8080 \
  --address=127.0.0.1 \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.244.0.0/16 \
  --service-cluster-ip-range=192.168.1.0/24 \
  --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \
  --root-ca-file=/opt/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --experimental-cluster-signing-duration=87600h0m0s"
  ```

- 使用systemd管理kube-controller-manager，在`/usr/lib/systemd/system/`，目录下创建文件：`kube-controller-manager.service`文件，文件内容如下：
  
  ```
  [Unit]
  Description=Kubernetes Controller Manager
  Documentation=https://github.com/kubernetes/kubernetes
  
  [Service]
  EnvironmentFile=/opt/kubernetes/conf/kube-controller-manager.conf
  ExecStart=/opt/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
  Restart=on-failure
  
  [Install]
  WantedBy=multi-user.target
  ```

- 重新加载服务：`systemctl daemon-reload`

- 自启动：`systemctl enable kube-controller-manager`

# 安装kube-scheduler

## kubernetes-master-01上安装kube-scheduler

### 生成证书

证书可以直接使用kube-apiserver安装时生成的证书

## 安装和配置

- 下载地址：`https://www.downloadkubernetes.com/`，OS: `linux`，architectures: `amd64`，version: `v1.25.2`

- 将下载的文件：`kube-scheduler`加上执行权限：`sudo chmod 755 kube-scheduler`

- 创建kubernetes安装目录：`sudo mkdir /opt/kubernetes/{bin,conf,ssl,logs} -p`

- 将`kube-scheduler`文件拷贝到`/opt/kubernetes/bin/`目录下

- 在`/opt/kubernetes/conf/`目录下创建`kube-scheduler.conf`配置文件，配置文件内容如下：
  
  ```
  KUBE_SCHEDULER_OPTS="--logtostderr=false \
  --v=2 \
  --log-dir=/opt/kubernetes/logs \
  --leader-elect \
  --master=127.0.0.1:8080 \
  --address=127.0.0.1"
  ```

- 使用systemd管理kube-scheduler，在`/usr/lib/systemd/system/`，目录下创建文件：`kube-scheduler.service`文件，文件内容如下：
  
  ```
  [Unit]
  Description=Kubernetes Scheduler
  Documentation=https://github.com/kubernetes/kubernetes
  
  [Service]
  EnvironmentFile=/opt/kubernetes/conf/kube-scheduler.conf
  ExecStart=/opt/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
  Restart=on-failure
  
  [Install]
  WantedBy=multi-user.target
  ```

- 重新加载服务：`systemctl daemon-reload`

- 自启动：`systemctl enable kube-scheduler`

# # 安装kubectl

## kebernetes-master-01上安装kubectl

# 给kubelete-bootstrap授权

## kubernetes-master-01上进行授权

执行如下命令：

```bash
kubectl create clusterrolebinding kubelet-bootstrap  --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```
