---
title: Kubernetes环境搭建
date: 2022-10-02 19:37:03
type: tags
tags: [Kubernetes, Containerd]
categories: [DevOps, Kubernetes, 云原生]
---


## 搭建环境 

> 待补充：二进制安装，containerd替换容器运行时

### 使用`MiniKube`准备环境

- **安装**

进入[minikube官网](https://minikube.sigs.k8s.io/docs/start/)下载资源

```shell
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

- **安装`kubectl`**

```shel
minikube kubectl
```

- **使用**

所有原生的`kubectl`命令都需要用`minikube kubectl --`代替，因此使用别名

```shell
alias kubectl="minikube kubectl --"
```

- **启动Kubernetes环境**

```shell
minikube start --image-mirror-country='cn' --kubernetes-version=v1.23.3
# 如果docker启动在root用户, 需要用 minikube start --kubernetes-version=v1.23.3 --force
# 如果用别的用户, 需要将用户添加到docker组， sudo usermod -aG docker $USER && newgrp docker
```

初始化完毕之后可以查看集群状态

```shell

[root@VM-8-17-centos kubernetes]# minikube node list
minikube        192.168.49.2
[root@VM-8-17-centos kubernetes]# minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```



### 使用`kubeadm`准备环境

> 高可用集群部署可以参考 [kubeadm创建高可用集群](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

**控制平面节点**和**计算节点**都需要执行[环境初始化]()、[启动kubelet]()的步骤。

#### 环境初始化

所有主机都需要执行以下步骤

- **互通机器相互放开安全策略**
- **关闭SWAP**

```shell
# 临时关闭
sudo swapoff -a
# 永久关闭
cp -p /etc/fstab /etc/fstab.bak$(date '+%Y%m%d%H%M%S') # 备份
# Redhat
sed -i "s/\/dev\/mapper\/rhel-swap/\#\/dev\/mapper\/rhel-swap/g" /etc/fstab
# CentOS
sed -i "s/\/dev\/mapper\/centos-swap/\#\/dev\/mapper\/centos-swap/g" /etc/fstab
# 修改后重新挂载全部挂载点
mount -a

# 查看Swap
free -m
cat /proc/swaps
```



- **关闭`SELINUX`**

```shell
$ sudo setenforce 0
$ sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

- **允许iptables桥接流量**

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system
```

**检查br_netfilter模块**

如果`/proc/sys/net/bridge/bridge-nf-call-iptables`文件不存在，则执行

```shell
modprobe br_netfilter
```



- **用户**

```shell
$ useradd k8s
$ passwd k8s
$ sed -i '/^root.*ALL=(ALL).*ALL/a\k8s\tALL=(ALL) \tNOPASSWD: ALL' /etc/sudoers

```

- **配置环境变量**

```shell
$ su - k8s
$ mkdir -p ${HOME}/dev/env/k8s
$ tee ~/.bashrc << EOF

# .bashrc
 
# User specific aliases and functions
 
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
 
# Source global definitions
if [ -f /etc/bashrc ]; then
        . /etc/bashrc
fi
 
# User specific environment
# Basic envs
export LANG="en_US.UTF-8" # 设置系统语言为 en_US.UTF-8，避免终端出现中文乱码
export PS1='[\u@$(hostname) \W]\$ ' # 默认的 PS1 设置会展示全部的路径，为了防止过长，这里只展示："用户名@主机名 最后的目录名"
export WORKSPACE="${HOME}/dev/env/k8s" # 设置工作目录
export PATH=$HOME/bin:$PATH # 将 $HOME/bin 目录加入到 PATH 变量中
 
# Default entry folder
cd '$WORKSPACE' # 登录系统，默认进入 workspace 目录
EOF
```

- **kubernetes yum源**

```shell
$ cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```



- **容器运行时**

**Docker**

> kubernetes v1.24已经移除了dockershim，不支持docker作为cri，执行 ls -l /var/run|grep docker 可以看到

```shell
# 如果已经安装了docker就不用管
$ docker info
# 安装docker
$ sudo yum install -y yum-utils
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

$ sudo yum install docker-ce docker-ce-cli containerd.io -y
# 镜像地址
$ mkdir -p /etc/docker
$ sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://registry.docker-cn.com", "https://kd7d1vuv.mirror.aliyuncs.com"]
}
EOF
# 替换docker镜像存储位置
$ systemctl show --property=FragmentPath docker
$ sed -i 's#.*ExecStart.*#ExecStart=/usr/bin/dockerd -g /data/docker  -H fd:// --containerd=/var/run/containerd/containerd.sock#g' /usr/lib/systemd/system/docker.service

# 启动Docker
$ sudo systemctl start docker
$ sudo systemctl enable docker 

```



**Containerd**

- 安装Containerd

```shell
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# 安装containerd
sudo yum install -y containerd.io-1.6.6-3.1.el7


# 查看containerd的配置
cat /etc/containerd/config.toml|grep -v '#'
# 覆盖原有配置
sudo tee /etc/containerd/config.toml << EOF
[plugins]

  [plugins.cri]
    sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"
    [plugins.cri.registry]
      [plugins.cri.registry.mirrors]

        [plugins.cri.registry.mirrors."docker.io"]
          endpoint = ["https://peldeode.mirror.aliyuncs.com"]
EOF

# 启动服务，允许开机自启
sudo systemctl start containerd
sudo systemctl enable containerd

```

- 搭建`Containerd`代理

> 国内环境拉取k8s.gcr.io , registry.k8s.io镜像有墙的阻碍

搭建`http`代理

**使用v2ray**

```shell
vi /etc/v2ray/config.json
 "inbounds": [
 	 {
      "port": 41997,
      "protocol": "http",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "settings": {
        "auth": "noauth",
        "udp": false
      }
    }
 ]
```

**使用mosn**

```shell
# 待补充
```

开通网络白名单后`Containerd`设置代理

```shell
systemctl show --property=FragmentPath containerd
vim /usr/lib/systemd/system/containerd.service
[Service]
Environment="HTTP_PROXY=http://8.218.45.255:41997/"
Environment="HTTPS_PROXY=http://8.218.45.255:41997/"

systemctl daemon-reaload
systemctl restart containerd
```







#### 依赖准备

所有主机都需要执行

> `kubeadm`：用来初始化集群的指令。
>
> `kubelet`：在集群中的每个节点上用来启动 Pod 和容器等。
>
> `kubectl`：用来与集群通信的命令行工具。
>
> 需要注意三点：
>
> - `kubelet`在v1.24将cri-dockerd移除了
> - `kubectl`与集群版本差异必须在一个小版本号内，如：v1.24能与v1.23、v1.24和v1.25的控制平面通信
> - 可以直接通过yum install -y kubelet kubeadm kubectl  --disableexcludes=kubernetes 直接安装二进制

##### yum安装

> 在安装 kubeadm 的过程中，kubeadm 和 kubelet、kubectl、kubernetes-cni 这几个二进制文件都会被自动安装好

```shell
sudo  yum install -y kubelet-1.24.0-0 kubeadm-1.24.0-0 kubectl-1.24.0-0
```



##### 下载二进制文件安装

> kubelet直接二进制安装可能不够..待确定

- **安装kubectl**

> 如果要下载最新稳定版可以：curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
>
> 如果有socks5代理可以加上 --socks5 'user:pwd@host:port'如 :
>
> $  curl --socks5 'liang:liang1997!!@8.218.45.255:57231' -LO https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubectl

```shell
$ PROXY='--socks5 ''liang:liang1997!!@8.218.45.255:57231'''
$ curl $PROXY -LO https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubectl
```

验证`kubectl`正确性

```shell
$ curl $PROXY -LO "https://dl.k8s.io/v1.24.0/bin/linux/amd64/kubectl.sha256"
$ echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check # 输出kubectl: OK
```

安装`kubectl`二进制

```shell
$ sudo install -o root -g root -m 0755 kubectl /usr/bin/kubectl
```

查看所安装的版本

```shell
$ kubectl version --client
```

- **安装kubelet**

下载`kubelet`

```shell
$ PROXY='--socks5 ''liang:liang1997!!@8.218.45.255:57231'''
$ curl $PROXY -LO https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubelet
```

校验`kubelet`

```shell
 $ curl $PROXY -LO "https://dl.k8s.io/v1.24.0/bin/linux/amd64/kubelet.sha256"
 $ echo "$(cat kubelet.sha256)  kubelet" | sha256sum --check # kubelet: OK
```

安装`kubelet`二进制

```shell
$ sudo install -o root -g root -m 0755 kubelet /usr/bin/kubelet
```



查看版本

```shell
$ kubelet --version
Kubernetes v1.24.0
```



- **安装kubeadm**

下载`kubelet`

```shell
$ PROXY='--socks5 ''liang:liang1997!!@8.218.45.255:57231'''
$ curl $PROXY -LO https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubeadm
```

校验`kubelet`

```shell
 $ curl $PROXY -LO "https://dl.k8s.io/v1.24.0/bin/linux/amd64/kubeadm.sha256"
 $ echo "$(cat kubeadm.sha256)  kubeadm" | sha256sum --check # kubeadm: OK
```

安装`kubelet`二进制

```shell
$ sudo install -o root -g root -m 0755 kubeadm /usr/local/bin/kubeadm
```

查看版本

```shell
$ kubeadm version
```



- **其他依赖**

**crictl**

```shell
PROXY='--socks5 ''liang:liang1997!!@8.218.45.255:57231'''
curl $PROXY -LO https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.24.0/crictl-v1.24.0-linux-amd64.tar.gz
tar -zxvf crictl-v1.24.0-linux-amd64.tar.gz
sudo cp ./crictl /usr/bin/
```

**conntrack**

```shelll
sudo yum install -y conntrack
```



#### 启动`kubelet`

配置`kubelet.service`

```shell
$ cat <<EOF |sudo tee /usr/lib/systemd/system/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

$ systemctl show --property=FragmentPath kubelet
```

配置默认镜像

```shell
cat <<EOF| sudo tee /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.2"
EOF
```



启动

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl start kubelet
$ sudo systemctl enable --now kubelet
```



#### 初始化控制节点所需的VIP（可选）

> 这里使用haproxy + keepalived，云计算需要同账户下内网互通，或者使用云LB产品提供VIP

所有`master`节点都需要安装：
```shell
$ sudo yum -y install haproxy keepalived
```

所有`master`节点修改`haproxy`配置文件：

```shell
$ MASTER_IP_PORT1=159.75.25.84:6443
$ MASTER_IP_PORT2=43.138.155.95:6443
$ cat > /etc/haproxy/haproxy.cfg << EOF
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats

defaults
    mode                    tcp
    log                     global
    option                  tcplog
    option                  dontlognull
    option                  redispatch
    retries                 3
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout check           10s
    maxconn                 3000

frontend  k8s_https *:8443
    mode      tcp
    maxconn      2000
    default_backend     https_sri
    
backend https_sri
    balance      roundrobin
    server master1-api $MASTER_IP_PORT1  check inter 10000 fall 2 rise 2 weight 1
    server master2-api $MASTER_IP_PORT2  check inter 10000 fall 2 rise 2 weight 1
EOF
```

各个`MASTER`节点配置`KEEPALIVED`:

master1:

```shell
$ mkdir /etc/keepalived
$ MASTER_IP_PORT1=159.75.25.84
$ EXPOSE_VIP= # 公网VIP
$ cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
	script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
rise 1
}
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    mcast_src_ip $MASTER_IP_PORT1
    virtual_router_id 51
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        $EXPOSE_VIP
    }
    track_script {
       chk_apiserver
    }
}
EOF
```

master2:

```shell
$ mkdir /etc/keepalived
$ MASTER_IP_PORT2=43.138.155.95
$ EXPOSE_VIP= # 公网VIP
$ cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
	script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
rise 1
}
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    mcast_src_ip $MASTER_IP_PORT2
    virtual_router_id 51
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        $EXPOSE_VIP
    }
    track_script {
       chk_apiserver
    }
}
EOF
```

两个`master`节点配置健康检查脚本：

```shell
$ cat > /etc/keepalived/check_apiserver.sh  << EOF
#!/bin/bash

err=0
for k in $(seq 1 3)
do
    check_code=$(pgrep haproxy)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 1
        continue
    else
        err=0
        break
    fi
done

if [[ $err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi
EOF

$ chmod +x /etc/keepalived/check_apiserver.sh # 授权
```

启动服务，检查是否运行正常

```shell
systemctl daemon-reload
systemctl enable --now haproxy
systemctl enable --now keepalived

ping $EXPOSE_VIP

```



#### 初始化控制平面节点

##### (内网不通)

> [参考链接](https://blog.csdn.net/chen645800876/article/details/105833835/)

内网不通（多台云主机，跨租户，跨云平台）的情况比较麻烦，需要使用虚拟网卡挂载公网IP

###### 创建虚拟网卡

所有节点都需要执行：

```shell
# step0 ，公网IP变量
#EXPOSE_IP=43.138.155.95
#EXPOSE_IP=114.132.166.222
EXPOSE_IP=159.75.25.84
#EXPOSE_IP=119.91.157.11
# step1 ，注意替换你的公网IP进去
cat > /etc/sysconfig/network-scripts/ifcfg-eth0:1 <<EOF
BOOTPROTO=static
DEVICE=eth0:1
IPADDR=${EXPOSE_IP}
PREFIX=32
TYPE=Ethernet
USERCTL=no
ONBOOT=yes
EOF
# step2 如果是centos8，需要重启
systemctl restart network
# step3 查看新建的IP是否进去
ip addr
```

###### 修改kubelet启动参数

所有节点都需要执行：

```shell
vi /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --node-ip=43.138.155.95 # 加上-node-ip=43.138.155.95（公网IP）
```

###### 修改`kubeadm`配置

```shell
MASTER_EXPOSE_IP=43.138.155.95
MASTER_INNER_IP=10.0.8.17
cat >kubeadm-config.yaml <<EOF
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: rickye.yejialiangabcdef # 6个字符.16个字符
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: ${MASTER_EXPOSE_IP}
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  name: $(hostname)
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
  - ${MASTER_INNER_IP} # 内网
  - ${MASTER_EXPOSE_IP} # 公网
  - 10.96.0.1   #不要替换，此IP是API的集群地址，部分服务会用到
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: ${MASTER_EXPOSE_IP}:6443 # 公网
controllerManager:  # 新版本
  extraArgs: 
    # horizontal-pod-autoscaler-use-rest-clients: "true" 
    horizontal-pod-autoscaler-sync-period: "10s" 
    node-monitor-grace-period: "10s"
    cluster-cidr: "10.244.0.0/16" # 保持与flannel一样
dns:
  {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.24.0      # 改成对应的版本
networking:
  dnsDomain: cluster.local
  podSubnet: 172.168.0.0/12     # 如果跟公司ip冲突得改
  serviceSubnet: 10.96.0.0/12   # 如果跟公司ip冲突得改
scheduler: {}
EOF
```

初始化

```shell
kubeadm init --config=kubeadm-config.yaml 

kubeadm join 43.138.155.95:6443 --token rickye.yejialiangabcdef \
        --discovery-token-ca-cert-hash sha256:e89f72c62750d95dd726f2d1c3dbddc270df5776ac13bf484dace01c91c27cdb
```

###### 修改Master节点Apiserver配置

```shell
# 修改两个信息，添加--bind-address和修改--advertise-address
vim /etc/kubernetes/manifests/kube-apiserver.yaml

spec:
  containers:
  - command:
    - kube-apiserver
    - --bind-address=0.0.0.0 #添加此参数
    - --advertise-address=43.138.155.95 

```







##### (内网互通)

> kubeadm init <args>， 有几个参数需要特别注意：
>
> - --control-plane-endpoint：控制平面节点要做高可用，则需要为所有控制平面节点设置共享端点，可以是DNS名字或者负载均衡的IP
> - --config init.yaml：从指定的配置文件初始化
> - --cri-socket：指定容器运行时实现的socket路径
>
> 可以通过修改源码再编译的方式，修改CA和证书的有效期，源位置分别位于：
>
> [CA有效期定义](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/client-go/util/cert/cert.go#L39)
>
> [证书有效期定义](https://github.com/kubernetes/kubernetes/blob/master/cmd/kubeadm/app/constants/constants.go#L49)

###### **编写初始化控制节点的配置文件**

>  kubeadm config print init-defaults 可以打印默认的初始化配置

```shell
$ MASTER_IP=192.168.0.1 
```



```yaml
$ cat >kubeadm-config.yaml <<EOF
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: rickye.yejialiangabcdef # 6个字符.16个字符
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: ${MASTER_IP}# 内网IP不用变
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  name: $(hostname)
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
  - ${MASTER_IP}
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: ${MASTER_IP}:6443
controllerManager:  # 新版本
  extraArgs: 
    # horizontal-pod-autoscaler-use-rest-clients: "true" 
    horizontal-pod-autoscaler-sync-period: "10s" 
    node-monitor-grace-period: "10s"
    cluster-cidr: "10.244.0.0/16" # 保持与flannel一样
dns:
  {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.24.0      # 改成对应的版本
networking:
  dnsDomain: cluster.local
  podSubnet: 172.168.0.0/12     # 如果跟公司ip冲突得改
  serviceSubnet: 10.96.0.0/12   # 如果跟公司ip冲突得改
scheduler: {}
EOF
```

指定配置文件初始化控制节点：

```shell
$ kubeadm init --config ./kubeadm-config.yaml
```

控制台输出：

```shell

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 192.168.0.100:6443 --token rickye.yejialiangabcdef \
        --discovery-token-ca-cert-hash sha256:2145963ad5d86508f5eb355dedf2ba864db4c4326b65558e63bde123f05186d6 \
        --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.100:6443 --token rickye.yejialiangabcdef \
        --discovery-token-ca-cert-hash sha256:2145963ad5d86508f5eb355dedf2ba864db4c4326b65558e63bde123f05186d6
```

按照提示，在`master`节点执行以下操作配置环境信息：

```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  # root用户可以直接
  export KUBECONFIG=/etc/kubernetes/admin.conf
```



#### 安装网络插件

##### (内网不通)

###### 主节点修改`flannel`配置

```shell
# 共修改两个地方，一个是args下，添加
 args:
 - --public-ip=$(PUBLIC_IP) # 添加此参数，申明公网IP
 - --iface=eth0             # 添加此参数，绑定网卡
 
 
 # 然后是env下
 env:
 - name: PUBLIC_IP     #添加环境变量
   valueFrom:          
     fieldRef:          
       fieldPath: status.podIP
       
       
# 展示

      containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.19.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --public-ip=$(PUBLIC_IP) # 添加此参数，申明公网IP
        - --iface=eth0             # 添加此参数，绑定网卡
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: PUBLIC_IP     #添加环境变量
          valueFrom:
            fieldRef:
              fieldPath: status.podIP


```

```shell
kubectl apply -f kube-flannel.yml
```



##### (内网互通)

安装网络插件前，`CoreDNS`的`pod`是未启动的，因此需要安装网络插件，常见的网络插件可以参考[kubernetes官网-实现kubernetes网络模型的插件](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model)

这里使用[Flannel](https://github.com/flannel-io/flannel#flannel)

###### 下载配置文件

> 如果PING raw.githubusercontent.com返回的是localhost，可以将域名跟IP的映射关系写死到/etc/hosts
>
> 1. 安装nslookup dig的工具：yum install -y bind-utils
> 2. dig raw.githubusercontent.com 将IP（185.199.108.133）记录
> 3. cat /etc/hosts增加记录: 185.199.108.133 raw.githubusercontent.com

```shell
 PROXY='--socks5 ''liang:liang1997!!@8.218.45.255:57231'''
 curl $PROXY -LO  https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

如果进行`kubeadm init`时指定了`cluster-cidr `或者在配置文件中自定义了网段，那么需要修改配置文件的`Network`为自定义网段：

> 我这里直接kubectl apply -f kube-flannel.yml后，查看flannel pod日志发现 此处的`10.244.0.0/16`和默认初始化的`pod-cidr`不一致，因此修改kube-flannel.yml网段为默认初始化`172.160.0.0/12`问题解决。
>
> 默认的网段可以通过` cat /etc/kubernetes/manifests/kube-controller-manager.yaml`查看，这里需要注意的是，`--cluster-cidr=172.168.0.0/12`代表的是整个`kubernetes`集群的网段，每个`node`是这网段的一部分，比如`master`为`172.168.0.0/24`，`node1`为`172.168.1.0/24`，如果flannel的网段设置为`172.168.0.0/24`那么只能管理`master`节点的网络，不能管理`node1`的网络，所以这里的`flannel`一定要设置为集群的网段。

```yml
net-conf.json: | 
  { 
    "Network": "10.244.0.0/16", 
    "Backend": { 
      "Type": "vxlan" 
      } 
  }
```

###### 启动插件

```shell
# 直接执行
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# 下载yml后执行
kubectl apply -f kube-flannel.yml
```

查看POD运行状态均正常：

```shell
kubectl get pod -A
NAMESPACE      NAME                              READY   STATUS    RESTARTS      AGE
kube-flannel   kube-flannel-ds-5z25n             1/1     Running   3 (60s ago)   83s
kube-system    coredns-7f74c56694-8b87c          1/1     Running   0             70m
kube-system    coredns-7f74c56694-pqv62          1/1     Running   0             70m
kube-system    etcd-linux01                      1/1     Running   6 (32m ago)   71m
kube-system    kube-apiserver-linux01            1/1     Running   6 (32m ago)   71m
kube-system    kube-controller-manager-linux01   1/1     Running   1 (32m ago)   71m
kube-system    kube-proxy-4546z                  1/1     Running   1 (32m ago)   70m
kube-system    kube-scheduler-linux01            1/1     Running   6 (32m ago)   71m

```



#### 初始化计算节点

如果之前没有记录在`master`输出的`kubeadm join`指令，则可以到`master`节点执行以下命令：

```shell
# 查看token列表
kubeadm token list
# 获取CA证书的sha256哈希值
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
2145963ad5d86508f5eb355dedf2ba864db4c4326b65558e63bde123f05186d6

# 构建命令
kubeadm join 192.168.0.100:6443 --token rickye.yejialiangabcdef \
        --discovery-token-ca-cert-hash sha256:2145963ad5d86508f5eb355dedf2ba864db4c4326b65558e63bde123f05186d6
```

生成的`token`具有失效时间，可以手工生成无期限的token：

```shell
kubeadm token create --ttl 0
```

也可以很方便的，使用一个命令构建`kubeadm join`：

```shell
kubeadm token create --ttl 0 --print-join-command
kubeadm join 192.168.0.100:6443 --token ujp31z.dt5tsl0t5rcd0mui --discovery-token-ca-cert-hash sha256:2145963ad5d86508f5eb355dedf2ba864db4c4326b65558e63bde123f05186d6

```

`master`查看节点是否接入集群

> 如果`worker`节点的10250端口没有对`master`开放，网络插件是起不来的

```shell
kubectl get pod -A
NAMESPACE      NAME                              READY   STATUS    RESTARTS      AGE
kube-flannel   kube-flannel-ds-5z25n             1/1     Running   3 (34m ago)   34m
kube-flannel   kube-flannel-ds-dr2w6             1/1     Running   0             3m16s  # worker节点网络相关的POD
kube-system    coredns-7f74c56694-8b87c          1/1     Running   0             104m
kube-system    coredns-7f74c56694-pqv62          1/1     Running   0             104m
kube-system    etcd-linux01                      1/1     Running   6 (65m ago)   104m
kube-system    kube-apiserver-linux01            1/1     Running   6 (65m ago)   104m
kube-system    kube-controller-manager-linux01   1/1     Running   1 (65m ago)   104m
kube-system    kube-proxy-4546z                  1/1     Running   1 (65m ago)   104m
kube-system    kube-proxy-vp4fq                  1/1     Running   0             3m16s
kube-system    kube-scheduler-linux01            1/1     Running   6 (65m ago)   104m

 kubectl get node
NAME      STATUS   ROLES           AGE     VERSION
linux01   Ready    control-plane   105m    v1.24.0
linux02   Ready    <none>          4m17s   v1.24.0

```



#### 部署其他组件

##### Dashboard插件

```shell
PROXY='--socks5 ''liang:liang1997!!@8.218.45.255:57231'''
curl $PROXY -LO  https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml
kubectl apply -f recommended.yaml

# 查看状态
kubectl get pod -n kubernetes-dashboard
NAME                                        READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-8c47d4b5d-r6ktw   1/1     Running   0          118s
kubernetes-dashboard-5676d8b865-crv57       1/1     Running   0          118s
```





##### 容器存储插件

这里用[ROOK](https://rook.github.io/docs/rook/latest/Getting-Started/intro/)插件：

###### 部署Rook Operator

```shell
PROXY='--socks5 ''liang:liang1997!!@8.218.45.255:57231'''
curl $PROXY -LO https://raw.githubusercontent.com/rook/rook/master/deploy/examples/crds.yaml
curl $PROXY -LO https://raw.githubusercontent.com/rook/rook/master/deploy/examples/common.yaml
curl $PROXY -LO https://raw.githubusercontent.com/rook/rook/master/deploy/examples/operator.yaml
kubectl create -f crds.yaml -f common.yaml -f operator.yaml

# 查看POD状态
kubectl -n rook-ceph get pod
NAME                                 READY   STATUS    RESTARTS   AGE
rook-ceph-operator-b5c96c99b-d29gs   1/1     Running   0          2m45s

```



###### 部署Ceph Cluster

> 待研究，如何使用国内镜像拉取rook相关的镜像

- **拉取镜像方法一：提前拉取**

拉取需要的镜像，再通过`tag`将镜像改名：

```shell
touch ceph_cluster_image.sh
SOURCE_MIRROR="registry.cn-hangzhou.aliyuncs.com/google_containers/"
TARGET_MIRROR="k8s.gcr.io/sig-storage/"
IMAGE_LIST=('csi-provisioner:v3.0.0' 'csi-node-driver-registrar:v2.3.0' 'csi-attacher:v3.3.0' 'csi-snapshotter:v4.2.0' 'csi-resizer:v1.3.0' )
for(( i=0;i<${#IMAGE_LIST[@]};i++)) do
  name=${IMAGE_LIST[$i]}
  ctr -n k8s.io image pull $SOURCE_MIRROR""$name 
  ctr -n k8s.io image tag $SOURCE_MIRROR""$name  $TARGET_MIRROR""$name 
  ctr -n k8s.io image rm $SOURCE_MIRROR""$name 
done

sh -x ceph_cluster_image.sh
```

- **拉取镜像方法二：设置全局代理**

```shell
export https_proxy='socks5://liang:liang1997!!@8.218.45.255:57231'
export http_proxy='socks5://liang:liang1997!!@8.218.45.255:57231'
# 拉取镜像即可
ctr image pull k8s.gcr.io/sig-storage/csi-provisioner:v3.0.0
# 复原
unset http_proxy
unset https_proxy
```

- **设置Containerd全局代理**

搭建`http`代理

**使用v2ray**

```shell
vi /etc/v2ray/config.json
 "inbounds": [
 	 {
      "port": 41997,
      "protocol": "http",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "settings": {
        "auth": "noauth",
        "udp": false
      }
    }
 ]
```

**使用mosn**

```shell
# 待补充
```

开通网络白名单后`Containerd`设置代理

```shell
systemctl show --property=FragmentPath containerd
vim /usr/lib/systemd/system/containerd.service
[Service]
Environment="HTTP_PROXY=http://8.218.45.255:41997/"
Environment="HTTPS_PROXY=http://8.218.45.255:41997/"

systemctl daemon-reaload
systemctl restart containerd
```







```shell
PROXY='--socks5 ''liang:liang1997!!@8.218.45.255:57231'''
curl $PROXY -LO https://raw.githubusercontent.com/rook/rook/master/deploy/examples/cluster.yaml
kubectl create -f cluster.yaml

# 查看POD状态
kubectl -n rook-ceph get pod
```





### 使用二进制方式准备环境




