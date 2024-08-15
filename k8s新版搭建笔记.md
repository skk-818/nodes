# k8s 部署文档新版

https://www.kancloud.cn/king_om/kubernetes/3154676

# 环境准备

### 网络配置

==注意：ip 配置需要和 vm中 net 网络的配置保持同步==

vim /etc/sysconfig/network-scripts/ifcfg-ens33

```sh
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="12e875a7-5564-4b3d-9ab6-0d2a4c3a33e3"
DEVICE="ens33"
ONBOOT="yes"
IPV6_PRIVACY="no"
IPADDR="192.168.5.40" # 根据 vm 配置来
GATEWAY="192.168.5.2"
NETMASK="255.255.255.0"
DNS1="192.168.5.2"

systemctl restart network
```

### 节点环境

master 192.168.5.41

node1 192.168.5.42

node2 192.168.5.43

node3 192.168.5.44

### cnetos7 配置阿里云镜像源

```sh
//下载阿里yum源
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo     
//扩展包
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo  

yum makecache        //更新yum仓库
yum clean all       //清理缓存

yum repolist  // 查看仓库列表
```

### 更新工具，下载相关工具

```sh
yum update -y
yum install -y yum-utils
```

### 节点基本设置,所有节点上

```sh
# 设置主机名
hostnamectl set-hostname master

# ip 
192.168.5.41 master 
192.168.5.42 node1 
192.168.5.43 node2
192.168.5.44 node3 


# 配置网桥过滤和地址转发
cat > /etc/sysctl.d/kubernetes.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
# 使用以下命令验证 net.ipv4.ip_forward 是否设置为 1
sysctl net.ipv4.ip_forward


# 配置 ipvs 功能
yum install ipset ipvsadm -y
# 添加需要加载的模块写入脚本文件
cat <<EOF> /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
# 为脚本添加执行权限 
chmod +x /etc/sysconfig/modules/ipvs.modules
# 执行脚本文件
/bin/bash /etc/sysconfig/modules/ipvs.modules
# 查看模块是否加载成功
lsmod | grep -e ip_vs -e nf_conntrack_ipv4

ip_vs_sh               12688  0 
ip_vs_wrr              12697  0 
ip_vs_rr               12600  0 
ip_vs                 145458  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack_ipv4      19149  7 
nf_defrag_ipv4         12729  1 nf_conntrack_ipv4
nf_conntrack          143411  9 ip_vs,nf_nat,nf_nat_ipv4,nf_nat_ipv6,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4,nf_conntrack_ipv6
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack



# 时间同步
systemctl start chronyd
systemctl enable chronyd
date



# 禁用SELinux和Firewalld服务：
systemctl stop firewalld
systemctl disable firewalld
sed -i 's/enforcing/disabled/' /etc/selinux/config # 重启后生效


# 禁用swap分区
$ swapoff -a # 临时关闭swap分区
$ vi /etc/fstab # 永久关闭swap分区，注释掉fstab中包含swap的这一行即可
# /dev/mapper/centos-swap swap                    swap    defaults        0 0

# 需要重启服务器生效
reboot

# 确认环境
查看防火墙状态. 查看防火墙状态systemctl status firewalld
查看swap分区情况 free 
查看SELinux状态：  sestatus
```



# 安装 Docker ，每个节点上

```sh
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum makecache fast(timer)

# 查看可用版本
yum list docker-ce --showduplicates | sort -r

# 安装 
yum -y install docker-ce-20.10.10-3.el8 
yum -y install docker-ce # 安装最新本

systemctl start docker 

# 配置
# 4、添加一个配置文件
#Docker 在默认情况下使用Vgroup Driver为cgroupfs，而Kubernetes推荐使用systemd来替代cgroupfs
[root@master ~]# mkdir /etc/docker
[root@master ~]# vim /etc/docker/daemon.json
{
	"exec-opts": ["native.cgroupdriver=systemd"],
     "registry-mirrors": [
        "https://dockerhub.icu",
        "https://docker.chenby.cn",
        "https://docker.1panel.live",
        "https://docker.awsl9527.cn",
        "https://docker.anyhub.us.kg",
        "https://dhub.kubesre.xyz"
      ]
}

systemctl restart  docker 
systemctl enable  docker # 开机自动启动
```



# k8s 安装

### 设置国内镜像源，旧版本，k8s版本停留在1.28

```sh
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### k8s 新版阿里云镜像,根据要下载的版本，更换 v1.28

```sh
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/repodata/repomd.xml.key
EOF
setenforce 0 # 关闭selinux 防火墙 关闭了 selinux 就不用了
# 会提示没有软件包，更新一下源
yum clean all
yum makecache
yum update
# yum list kubelet kubeadm kubectl 看一下可安装的包
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet
```



### cri-docker

==每个节点都要安装==

```sh
# 下载地址
https://github.com/Mirantis/cri-dockerd/releases

# 解压并复制过去
tar -zxvf ./cri-dockerd-0.3.14.amd64.tgz
cp -r ./cri-dockerd/cri-dockerd  /usr/local/bin/
chmod +x /usr/local/bin/cri-dockerd 

# 配置启动
cat >  /etc/systemd/system/cri-dockerd.service << EOF
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
 
[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9 --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --container-runtime-endpoint=unix:///var/run/cri-dockerd.sock --cri-dockerd-root-directory=/var/lib/dockershim --docker-endpoint=unix:///var/run/docker.sock --cri-dockerd-root-directory=/var/lib/docker
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
EOF

# 配置
cat > /etc/systemd/system/cri-dockerd.socket <<EOF
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service
[Socket]
ListenStream=/var/run/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker
[Install]
WantedBy=sockets.target
EOF


# 
systemctl daemon-reload # 重新加载服务
systemctl start cri-dockerd.service # 启动服务
systemctl enable cri-dockerd.service # 开机自启
```

### 安装组件

```sh
# 安装组件并记录好版本
yum install -y kubelet kubeadm kubectl
```

### 初始化集群

```sh

 kubeadm init \
--apiserver-advertise-address=192.168.5.90 \
--image-repository=registry.aliyuncs.com/google_containers \
--kubernetes-version v1.30.3 \
--service-cidr=10.9.0.0/16 \
--pod-network-cidr=10.8.0.0/16 \
--cri-socket unix:///var/run/cri-dockerd.sock

  
 # 如果初始化失败
 kubeadm reset 
 rm -r $HOME/.kube/config
 
        
kubeadm join 192.168.5.90:6443 --token 4nfqzw.hy3c3lntkzwult2m \
        --discovery-token-ca-cert-hash sha256:8ae584615bc703ef06567fc0e0cc5987ad43b5975d851c89bc26b8153d0d6610 \
        --cri-socket unix:///var/run/cri-dockerd.sock
```

### 部署网络插件 calio

```sh
# 下载
wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml

# 修改网络地址
vi calio.yaml

- name: CALICO_IPV4POOL_CIDR
   value: "10.8.0.0/16" # 对应初始化命令的 ip

# 经常出现 镜像拉不下来的情况，先把镜像在每个节点上都拉下来再安装
docker.io/calico/kube-controllers:v3.25.0
docker.io/calico/cni:v3.25.0
docker.io/calico/node:v3.25.0

# 执行安装
kubectl apply -f ./calio.yaml
```

### 部署 ingerss

全节点部署 ingress https://www.kancloud.cn/king_om/kubernetes/3154676 



### 卸载 k8s 

https://www.jb51.net/server/313713crh.htm



# 使用



















